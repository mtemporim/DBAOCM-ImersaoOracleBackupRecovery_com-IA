# Oracle 19c — Habilitação da Fast Recovery Area (FRA)

> **Ambiente:** Máquina virtual VMware com Oracle Linux 8.10 x86-64  
> **Data de execução:** 16 de maio de 2026  
> **Autor:** Documentado automaticamente pelo Claude Code  
> **Status:** FRA configurada e validada com sucesso

---

## Sumário

1. [Descrição do Ambiente](#1-descrição-do-ambiente)
2. [O que é a Fast Recovery Area](#2-o-que-é-a-fast-recovery-area)
3. [Estado Inicial — Antes da Configuração](#3-estado-inicial--antes-da-configuração)
4. [Passo a Passo da Configuração](#4-passo-a-passo-da-configuração)
5. [Validação](#5-validação)
6. [Comparativo Antes × Depois](#6-comparativo-antes--depois)
7. [Resumo da Configuração](#7-resumo-da-configuração)
8. [Troubleshooting](#8-troubleshooting)
9. [Referências](#9-referências)

---

## 1. Descrição do Ambiente

| Componente          | Detalhe                                              |
|---------------------|------------------------------------------------------|
| Hypervisor          | VMware (full virtualization)                         |
| Sistema Operacional | Oracle Linux Server release 8.10                     |
| Kernel              | 5.15.0-320.202.8.3.el8uek.x86_64 (UEK)               |
| Arquitetura         | x86_64                                               |
| IP da VM            | x.x.x.x                                              |
| Hostname            | ol8-orcl19-ribas.localdomain                         |
| Produto Oracle      | Oracle Database 19c Enterprise Edition (19.3.0.0.0)  |
| ORACLE_BASE         | /u01/app/oracle                                      |
| ORACLE_HOME         | /u01/app/oracle/product/19.0.0/dbhome_1              |
| Global DB Name      | orcl.localdomain                                     |
| SID                 | orcl                                                 |
| PDB                 | orclpdb                                              |
| Character Set       | AL32UTF8                                             |
| Datafiles           | /u02/oradata                                         |
| **FRA Path**        | /u01/app/oracle/fast_recovery_area                   |
| **FRA Size**        | 50 GB                                                |

### Espaço em disco

| Filesystem                      | Tamanho | Usado | Disponível | Uso% | Ponto de montagem |
|---------------------------------|---------|-------|------------|------|-------------------|
| /dev/mapper/disk2--vg1-u01      | 499 GB  | 11 GB | 489 GB     | 3%   | /u01              |
| /dev/mapper/disk3--vg1-u02      | 500 GB  | 6.8 GB| 493 GB     | 2%   | /u02              |

> A FRA foi criada em `/u01` (filesystem separado dos datafiles em `/u02`), seguindo a boa prática de isolar os arquivos de recuperação dos arquivos de dados.

---

## 2. O que é a Fast Recovery Area

A **Fast Recovery Area** (FRA) é uma área de armazenamento centralizada no Oracle Database para todos os arquivos relacionados à recuperação:

| Tipo de Arquivo               | Descrição                                               |
|-------------------------------|---------------------------------------------------------|
| Backupsets RMAN               | Backups completos e incrementais                        |
| Archived Redo Logs            | Logs arquivados necessários para recovery point-in-time |
| Flashback Logs                | Usados pelo Flashback Database                          |
| Autobackup do Controlfile     | Cópia automática do controlfile após cada backup        |
| Cópias do Online Redo Log     | Multiplexação opcional dos redo logs                    |
| Image Copies                  | Cópias bit a bit dos datafiles                          |

### Por que configurar a FRA?

- Gerenciamento automático de espaço: o Oracle exclui arquivos obsoletos automaticamente quando o espaço está acabando
- Pré-requisito para **modo ARCHIVELOG**, **Flashback Database** e **RMAN backup online**
- Simplifica o gerenciamento de retenção de backups
- Facilita operações de restore e recovery no RMAN

### Parâmetros de inicialização envolvidos

| Parâmetro                      | Finalidade                                                  |
|-------------------------------|--------------------------------------------------------------|
| `db_recovery_file_dest`        | Caminho do sistema de arquivos onde a FRA será criada       |
| `db_recovery_file_dest_size`   | Tamanho máximo em bytes que a FRA pode ocupar               |

---

## 3. Estado Inicial — Antes da Configuração

```sql
SQL> SHOW PARAMETER db_recovery_file_dest

NAME                           TYPE        VALUE
------------------------------ ----------- -----
db_recovery_file_dest          string
db_recovery_file_dest_size     big integer 0
```

```sql
SQL> SHOW PARAMETER log_archive_dest_1

NAME                           TYPE        VALUE
------------------------------ ----------- -----
log_archive_dest_1             string
```

**Conclusões do estado inicial:**
- FRA não configurada (`db_recovery_file_dest` vazio, `db_recovery_file_dest_size = 0`)
- Banco em modo `NOARCHIVELOG`
- Nenhum destino de archive log configurado
- RMAN sem destino padrão para backups

---

## 4. Passo a Passo da Configuração

### Passo 1 — Criar o diretório no sistema operacional

Executado como `root`:

```bash
mkdir -p /u01/app/oracle/fast_recovery_area
chown -R oracle:oinstall /u01/app/oracle/fast_recovery_area
chmod 750 /u01/app/oracle/fast_recovery_area
```

**Resultado:**
```
drwxr-x---. 2 oracle oinstall 6 May 16 14:39 /u01/app/oracle/fast_recovery_area
```

> **Importante:** O usuário `oracle` deve ter permissão de escrita. O `chmod 750` garante que apenas o dono (`oracle`) e o grupo (`oinstall`) tenham acesso — nenhuma permissão para outros usuários.

### Passo 2 — Configurar o tamanho da FRA

```sql
ALTER SYSTEM SET db_recovery_file_dest_size = 50G SCOPE=BOTH;
```

```
System altered.
```

> **Ordem importa:** O parâmetro `db_recovery_file_dest_size` **deve ser definido antes** de `db_recovery_file_dest`. O Oracle valida o caminho contra o tamanho já configurado. Inverter a ordem gera `ORA-19802`.

### Passo 3 — Definir o destino da FRA

```sql
ALTER SYSTEM SET db_recovery_file_dest = '/u01/app/oracle/fast_recovery_area' SCOPE=BOTH;
```

```
System altered.
```

> `SCOPE=BOTH` aplica a alteração imediatamente na instância em execução **e** persiste no SPFILE para sobreviver a restarts.

---

## 5. Validação

### 5.1 Confirmar parâmetros de inicialização

```sql
SQL> SHOW PARAMETER db_recovery_file_dest

NAME                           TYPE        VALUE
------------------------------ ----------- --------------------------------
db_recovery_file_dest          string      /u01/app/oracle/fast_recovery_area
db_recovery_file_dest_size     big integer 50G
```

### 5.2 Verificar uso da FRA — v$recovery_file_dest

```sql
SELECT name,
       space_limit/1024/1024/1024     AS limit_gb,
       space_used/1024/1024/1024      AS used_gb,
       number_of_files
FROM   v$recovery_file_dest;
```

```
NAME                                        LIMIT_GB   USED_GB NUMBER_OF_FILES
---------------------------------------- ---------- --------- ---------------
/u01/app/oracle/fast_recovery_area               50         0               0
```

### 5.3 Detalhamento por tipo de arquivo — v$flash_recovery_area_usage

```sql
SELECT file_type,
       percent_space_used,
       percent_space_reclaimable,
       number_of_files
FROM   v$flash_recovery_area_usage;
```

```
FILE_TYPE               PERCENT_SPACE_USED PERCENT_SPACE_RECLAIMABLE NUMBER_OF_FILES
----------------------- ------------------ ------------------------- ---------------
CONTROL FILE                             0                         0               0
REDO LOG                                 0                         0               0
ARCHIVED LOG                             0                         0               0
BACKUP PIECE                             0                         0               0
IMAGE COPY                               0                         0               0
FLASHBACK LOG                            0                         0               0
FOREIGN ARCHIVED LOG                     0                         0               0
AUXILIARY DATAFILE COPY                  0                         0               0

8 rows selected.
```

### 5.4 Estado do banco de dados

```sql
SQL> SELECT db_unique_name, log_mode, open_mode FROM v$database;

DB_UNIQUE_NAME                 LOG_MODE     OPEN_MODE
------------------------------ ------------ --------------------
orcl                           NOARCHIVELOG READ WRITE

SQL> SELECT host_name, version, instance_name FROM v$instance;

HOST_NAME                        VERSION           INSTANCE_NAME
-------------------------------- ----------------- ----------------
ol8-orcl19-ribas.localdomain     19.0.0.0.0        orcl
```

**Interpretação dos resultados:**
- FRA configurada com **50 GB** em `/u01/app/oracle/fast_recovery_area`
- Espaço usado: **0 GB** (nenhum arquivo criado ainda — esperado antes de habilitar ARCHIVELOG e executar backups)
- 8 tipos de arquivo reconhecidos pela FRA, todos com ocupação zero
- Banco **OPEN** em modo **READ WRITE** — sem interrupção de serviço durante a configuração

---

## 6. Comparativo Antes × Depois

| Parâmetro / Condição              | Antes                  | Depois                               |
|-----------------------------------|------------------------|--------------------------------------|
| `db_recovery_file_dest`           | (vazio)                | `/u01/app/oracle/fast_recovery_area` |
| `db_recovery_file_dest_size`      | `0`                    | `50G`                                |
| Diretório da FRA no SO            | Não existia            | Criado com `oracle:oinstall 750`     |
| Espaço alocado para FRA           | 0 GB                   | 50 GB                                |
| Restart necessário                | —                      | **Não** (`SCOPE=BOTH`)               |
| Interrupção de serviço            | —                      | **Nenhuma** (banco permaneceu OPEN)  |
| RMAN tem destino padrão           | Não                    | Sim                                  |
| Pré-requisito para ARCHIVELOG     | Não atendido           | **Atendido**                         |
| Pré-requisito para Flashback DB   | Não atendido           | **Atendido**                         |

---

## 7. Resumo da Configuração

| Item                     | Valor                                              |
|--------------------------|----------------------------------------------------|
| Data/hora de execução    | 16/05/2026 às 14:39–14:41 (BRT)                    |
| Diretório FRA            | `/u01/app/oracle/fast_recovery_area`               |
| Filesystem               | `/u01` — 499 GB total, 489 GB disponíveis          |
| Tamanho configurado      | 50 GB                                              |
| Espaço efetivamente usado| 0 GB (FRA recém-criada)                            |
| Parâmetro persistido     | SPFILE (SCOPE=BOTH)                                |
| Restart realizado        | Não                                                |
| Banco após configuração  | OPEN / READ WRITE                                  |
| Próximo passo sugerido   | Habilitar modo ARCHIVELOG (`habilitar-archivelog`) |

---

## 8. Troubleshooting

| Erro                                                                 | Causa                                                    | Solução                                                                                            |
|----------------------------------------------------------------------|----------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| `ORA-19802: cannot use DB_RECOVERY_FILE_DEST without DB_RECOVERY_FILE_DEST_SIZE` | `db_recovery_file_dest` definido antes do size | Executar `ALTER SYSTEM SET db_recovery_file_dest_size = <valor>` primeiro                      |
| `ORA-19816: WARNING: Files may exist in new recovery area`           | Diretório já contém arquivos de outra instância          | Limpar o diretório ou escolher outro caminho                                                        |
| `ORA-19504: failed to create file`                                   | Permissão insuficiente no SO                             | Executar `chown -R oracle:oinstall <dir>` e `chmod 750 <dir>` como root                            |
| `ORA-19809: limit exceeded for recovery files`                       | FRA lotou (space_used = space_limit)                     | Aumentar `db_recovery_file_dest_size` ou limpar backups obsoletos com `RMAN> DELETE OBSOLETE`      |
| FRA não persiste após restart                                        | Usado `SCOPE=MEMORY` ao invés de `SCOPE=BOTH`            | Redigitar os comandos com `SCOPE=BOTH` ou `SCOPE=SPFILE`                                           |
| `ORA-01031: insufficient privileges` ao executar ALTER SYSTEM        | Usuário sem privilégio SYSDBA                            | Conectar com `sqlplus / as sysdba` (OS authentication) ou com `sys` e `AS SYSDBA`                 |

### Como monitorar o uso da FRA

```sql
-- Visão geral do espaço
SELECT name,
       ROUND(space_limit/1073741824, 1) AS limit_gb,
       ROUND(space_used/1073741824, 3)  AS used_gb,
       ROUND(space_reclaimable/1073741824, 3) AS reclaimable_gb,
       number_of_files
FROM   v$recovery_file_dest;

-- Detalhamento por tipo
SELECT file_type, percent_space_used, number_of_files
FROM   v$flash_recovery_area_usage
WHERE  percent_space_used > 0;
```

### Rollback (remover FRA)

```sql
-- Executar apenas se necessário — NÃO executar se há archivelogs/backups na FRA
ALTER SYSTEM SET db_recovery_file_dest = '' SCOPE=BOTH;
ALTER SYSTEM SET db_recovery_file_dest_size = 0 SCOPE=BOTH;
```

```bash
# Remover diretório (apenas após garantir que não há arquivos referenciados)
rm -rf /u01/app/oracle/fast_recovery_area
```

---

## 9. Referências

| Recurso                                         | Descrição                                              |
|-------------------------------------------------|--------------------------------------------------------|
| Oracle Docs — Backup and Recovery User's Guide  | Capítulo "Configuring the Fast Recovery Area"          |
| `V$RECOVERY_FILE_DEST`                          | View com uso atual da FRA                              |
| `V$FLASH_RECOVERY_AREA_USAGE`                   | Detalhamento por tipo de arquivo                       |
| `V$SPPARAMETER`                                 | Confirmar que os parâmetros foram gravados no SPFILE   |
| Próximo desafio: `habilitar-archivelog`         | Habilitar ARCHIVELOG para permitir backups online e recovery point-in-time |

---

*Documentado automaticamente pelo Claude Code — Imersão Oracle Backup & Recovery com IA*
