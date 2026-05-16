# Oracle 19c — Habilitação do Modo ARCHIVELOG

> **Ambiente:** Máquina virtual VMware com Oracle Linux 8.10 x86-64  
> **Data de execução:** 16 de maio de 2026  
> **Autor:** Documentado automaticamente pelo Claude Code  
> **Status:** Modo ARCHIVELOG habilitado e validado com sucesso

---

## Sumário

1. [Descrição do Ambiente](#1-descrição-do-ambiente)
2. [O que é o Modo ARCHIVELOG](#2-o-que-é-o-modo-archivelog)
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
| FRA Path            | /u01/app/oracle/fast_recovery_area                   |
| FRA Size            | 50 GB                                                |
| **Archive Dest**    | USE_DB_RECOVERY_FILE_DEST (FRA)                      |

---

## 2. O que é o Modo ARCHIVELOG

Quando o banco está em modo **ARCHIVELOG**, o Oracle grava uma cópia de cada online redo log antes de reutilizá-lo. Esses arquivos — os **archived redo logs** — são a base de todo o recovery avançado do Oracle.

### O que o ARCHIVELOG habilita

| Recurso                          | Sem ARCHIVELOG | Com ARCHIVELOG |
|----------------------------------|:--------------:|:--------------:|
| Backup online (hot backup) RMAN  | ✗              | ✓             |
| Recovery point-in-time (PITR)    | ✗              | ✓             |
| Flashback Database               | ✗              | ✓             |
| Data Guard / Active Data Guard   | ✗              | ✓             |
| Backup com banco em produção     | ✗              | ✓             |
| Recovery sem perda de transações | Parcial        | ✓              |
 
### Fluxo dos archived redo logs

```
Transações → Online Redo Logs (grupo 1, 2, 3...)
                     ↓  (ao fazer log switch)
               ARCn archiva o log
                     ↓
               FRA: /u01/app/oracle/fast_recovery_area/ORCL/archivelog/
```

### Parâmetros envolvidos

| Parâmetro                     | Valor configurado                | Descrição                              |
|-------------------------------|----------------------------------|----------------------------------------|
| `log_mode`                    | ARCHIVELOG                       | Modo do banco (alterado com restart)   |
| `log_archive_dest_1`          | USE_DB_RECOVERY_FILE_DEST        | Destino herdado da FRA automaticamente |
| `log_archive_format`          | (padrão Oracle com FRA)          | Formato de nome dos archive logs       |

---

## 3. Estado Inicial — Antes da Configuração

```sql
SQL> SELECT log_mode FROM v$database;

LOG_MODE
------------
NOARCHIVELOG

SQL> ARCHIVE LOG LIST;

Database log mode              No Archive Mode
Automatic archival             Disabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     29
Current log sequence           31
```

**Conclusões do estado inicial:**
- Banco em modo `NOARCHIVELOG` — backups online impossíveis
- Archiving automático desabilitado
- FRA já configurada (Desafio 1) — destino herdado automaticamente, sem necessidade de configurar `log_archive_dest_1` manualmente
- Log sequence atual: 31

---

## 4. Passo a Passo da Configuração

> **Importante:** A alteração para ARCHIVELOG requer que o banco seja iniciado em modo **MOUNT**. É necessária uma janela de manutenção com breve indisponibilidade (~49 segundos neste ambiente).

### Passo 1 — Shutdown limpo

```sql
SHUTDOWN IMMEDIATE;
```

```
Database closed.
Database dismounted.
ORACLE instance shut down.
```

### Passo 2 — Startup em modo MOUNT

```sql
STARTUP MOUNT;
```

```
ORACLE instance started.

Total System Global Area  12.885.000.000 bytes
Fixed Size                    12.688.816 bytes
Variable Size              1.879.048.192 bytes
Database Buffers          10.972.000.000 bytes
Redo Buffers                  20.865.024 bytes
Database mounted.
```

> O banco fica em MOUNT: instância iniciada, controlfile lido, datafiles ainda fechados — nenhuma sessão de usuário é possível neste estado.

### Passo 3 — Alterar para modo ARCHIVELOG

```sql
ALTER DATABASE ARCHIVELOG;
```

```
Database altered.
```

### Passo 4 — Abrir o banco

```sql
ALTER DATABASE OPEN;
```

```
Database altered.
```

### Passo 5 — Forçar switch de redo log para gerar o primeiro archive

```sql
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM ARCHIVE LOG CURRENT;
```

```
System altered.
System altered.
```

> O switch força o processo **ARCn** a arquivar o log atual, confirmando que o archiving automático está funcionando.

---

## 5. Validação

### 5.1 Confirmar modo ARCHIVELOG

```sql
SQL> SELECT log_mode FROM v$database;

LOG_MODE
------------
ARCHIVELOG
```

### 5.2 ARCHIVE LOG LIST

```sql
SQL> ARCHIVE LOG LIST;

Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     31
Next log sequence to archive   33
Current log sequence           33
```

### 5.3 Destinos de archive válidos (v$archive_dest)

```sql
SELECT dest_id, status, destination
FROM   v$archive_dest
WHERE  status = 'VALID';
```

```
DEST_ID  STATUS    DESTINATION
-------  --------  --------------------------
      1  VALID     USE_DB_RECOVERY_FILE_DEST
```

### 5.4 Archived logs gerados (v$archived_log)

```sql
SELECT name, sequence#, first_time, completion_time,
       blocks*block_size/1024/1024 AS size_mb
FROM   v$archived_log
ORDER  BY completion_time DESC
FETCH  FIRST 5 ROWS ONLY;
```

```
NAME                                                                          SEQUENCE#  SIZE_MB
----------------------------------------------------------------------------- ---------  -------
.../fast_recovery_area/ORCL/archivelog/2026_05_16/o1_mf_1_32_o0kh2dfq_.arc        32    0.000
.../fast_recovery_area/ORCL/archivelog/2026_05_16/o1_mf_1_31_o0kh2d8o_.arc        31   14.409

2 rows selected.
```

### 5.5 Uso da FRA após primeiro archive (v$recovery_file_dest)

```sql
SELECT name,
       space_limit/1024/1024/1024 AS limit_gb,
       space_used/1024/1024       AS used_mb,
       number_of_files
FROM   v$recovery_file_dest;
```

```
NAME                                        LIMIT_GB    USED_MB  NUMBER_OF_FILES
----------------------------------------- ---------  ---------  ---------------
/u01/app/oracle/fast_recovery_area                50     14.410                2
```

### 5.6 Arquivos físicos na FRA

```bash
find /u01/app/oracle/fast_recovery_area -type f
```

```
/u01/app/oracle/fast_recovery_area/ORCL/archivelog/2026_05_16/o1_mf_1_31_o0kh2d8o_.arc
/u01/app/oracle/fast_recovery_area/ORCL/archivelog/2026_05_16/o1_mf_1_32_o0kh2dfq_.arc
```

**Interpretação dos resultados:**
- 2 archived logs criados automaticamente pelo processo ARCn
- Sequence 31: 14.41 MB (redo acumulado desde a criação do banco)
- Sequence 32: ~0 MB (log switch vazio após ALTER SYSTEM ARCHIVE LOG CURRENT)
- FRA consumindo 14.41 MB dos 50 GB disponíveis (< 0.03%)

---

## 6. Comparativo Antes × Depois

| Parâmetro / Condição             | Antes              | Depois                           |
|----------------------------------|--------------------|----------------------------------|
| `log_mode`                       | NOARCHIVELOG       | **ARCHIVELOG**                   |
| Automatic archival               | Disabled           | **Enabled**                      |
| Archive destination              | (inativo)          | USE_DB_RECOVERY_FILE_DEST (FRA)  |
| Oldest online log sequence       | 29                 | 31                               |
| Current log sequence             | 31                 | 33                               |
| Archived logs na FRA             | 0 arquivos / 0 MB  | 2 arquivos / 14.41 MB            |
| Hot backup RMAN possível         | Não                | **Sim**                          |
| Recovery point-in-time possível  | Não                | **Sim**                          |
| Flashback Database possível      | Não                | **Sim** (após configuração)      |
| Downtime para habilitar          | —                  | ~49 segundos                     |

---

## 7. Resumo da Configuração

| Item                     | Valor                                                 |
|--------------------------|-------------------------------------------------------|
| Início                   | 16/05/2026 15:49:00 BRT                               |
| Fim                      | 16/05/2026 15:49:49 BRT                               |
| Duração total            | **49 segundos**                                       |
| Restart necessário       | Sim (MOUNT → OPEN)                                    |
| Destino de archive       | FRA — USE_DB_RECOVERY_FILE_DEST                       |
| log_archive_dest_1 manual| Não necessário (FRA já configurada no Desafio 1)      |
| Primeiro archive gerado  | Sequence 31 — 14.41 MB                                |
| FRA após configuração    | 14.41 MB usados / 50 GB disponíveis                   |
| Próximo passo sugerido   | Configurar Flashback Database (`configure-flashback`) |

---

## 8. Troubleshooting

| Erro | Causa provável | Solução |
|------|----------------|---------|
| `ORA-01126: database must be mounted` | Tentativa de `ALTER DATABASE ARCHIVELOG` com banco OPEN | `SHUTDOWN IMMEDIATE` → `STARTUP MOUNT` → repetir |
| `ORA-19809: limit exceeded for recovery files` | FRA cheia | Aumentar `db_recovery_file_dest_size` ou `RMAN> DELETE OBSOLETE` |
| `ORA-16019: cannot use LOG_ARCHIVE_DEST_1 with LOG_ARCHIVE_DEST` | Conflito entre parâmetros legados | `ALTER SYSTEM RESET log_archive_dest SCOPE=BOTH` |
| Archive não é gerado após switch | Processo ARCn travado | Verificar `v$archive_processes`; executar `ALTER SYSTEM ARCHIVE LOG START` |
| Banco não sobe após STARTUP MOUNT | Controlfile corrompido ou ausente | Restore do controlfile via RMAN |

### Como monitorar archives em tempo real

```sql
-- Archives gerados na última hora
SELECT sequence#, name, blocks*block_size/1024/1024 AS size_mb,
       to_char(completion_time,'DD/MM HH24:MI') AS completed
FROM   v$archived_log
WHERE  completion_time > sysdate - 1/24
ORDER  BY sequence#;

-- Processos ARCn ativos
SELECT process, status, sequence#
FROM   v$archive_processes
WHERE  status != 'STOPPED';
```

### Rollback (voltar para NOARCHIVELOG)

```sql
-- Executar apenas em ambiente de testes — invalida backups online existentes
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE NOARCHIVELOG;
ALTER DATABASE OPEN;
```

---

## 9. Referências

| Recurso                                        | Descrição                                                  |
|------------------------------------------------|------------------------------------------------------------|
| Oracle Docs — Backup and Recovery User's Guide | Capítulo "Enabling Archiving"                              |
| `V$DATABASE`                                   | Coluna `LOG_MODE` — modo atual do banco                    |
| `V$ARCHIVED_LOG`                               | Histórico de archived logs gerados                         |
| `V$ARCHIVE_DEST`                               | Destinos de archive e seus status                          |
| `V$ARCHIVE_PROCESSES`                          | Processos ARCn e status de archiving                       |
| Desafio anterior: `habilitar-fra`              | Fast Recovery Area — destino usado pelo archiving          |
| Próximo desafio: `configure-flashback`         | Flashback Database — requer ARCHIVELOG habilitado          |

---

*Documentado automaticamente pelo Claude Code — Imersão Oracle Backup & Recovery com IA*
