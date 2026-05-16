# Oracle 19c — Backup Full com RMAN

> **Ambiente:** Máquina virtual VMware com Oracle Linux 8.10 x86-64  
> **Data de execução:** 16 de maio de 2026  
> **Autor:** Documentado automaticamente pelo Claude Code  
> **Status:** Backup full concluído e validado com sucesso

---

## Sumário

1. [Descrição do Ambiente](#1-descrição-do-ambiente)
2. [O que é o Backup Full com RMAN](#2-o-que-é-o-backup-full-com-rman)
3. [Estado Inicial — Antes do Backup](#3-estado-inicial--antes-do-backup)
4. [Configuração dos Parâmetros RMAN](#4-configuração-dos-parâmetros-rman)
5. [Execução do Backup](#5-execução-do-backup)
6. [Validação](#6-validação)
7. [Schema do Banco — Arquivos Cobertos](#7-schema-do-banco--arquivos-cobertos)
8. [Resumo do Backup](#8-resumo-do-backup)
9. [Troubleshooting](#9-troubleshooting)
10. [Manutenção e Monitoramento](#10-manutenção-e-monitoramento)
11. [Referências](#11-referências)

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
| SID / PDB           | orcl / orclpdb                                       |
| Character Set       | AL32UTF8                                             |
| Datafiles           | /u02/oradata                                         |
| FRA Path            | /u01/app/oracle/fast_recovery_area                   |
| FRA Size            | 50 GB                                                |
| Modo do banco       | ARCHIVELOG / READ WRITE                              |
| **Tag do backup**   | FULL_20260516                                        |
| **Destino**         | FRA (USE_DB_RECOVERY_FILE_DEST)                      |

### Espaço em disco

| Filesystem                      | Tamanho | Usado  | Disponível | Uso% | Ponto de montagem |
|---------------------------------|---------|--------|------------|------|-------------------|
| /dev/mapper/disk2--vg1-u01      | 499 GB  | 11 GB  | 489 GB     | 3%   | /u01 (FRA)        |
| /dev/mapper/disk3--vg1-u02      | 500 GB  | 6.8 GB | 493 GB     | 2%   | /u02 (datafiles)  |

---

## 2. O que é o Backup Full com RMAN

O **RMAN (Recovery Manager)** é a ferramenta nativa do Oracle para backup e recovery. Um **backup full** cobre:

| Componente              | Descrição                                                         |
|-------------------------|-------------------------------------------------------------------|
| Datafiles               | Todos os blocos de dados de todos os tablespaces                  |
| Controlfile             | Estrutura de controle do banco (autobackup automático)            |
| SPFILE                  | Parâmetros de inicialização (incluso no autobackup do controlfile)|
| Archived Redo Logs      | Logs arquivados necessários para recovery point-in-time           |

### Por que usar `BACKUP DATABASE PLUS ARCHIVELOG`?

```
BACKUP DATABASE          →  cobre todos os datafiles (CDB + PDB + PDB$SEED)
PLUS ARCHIVELOG          →  inclui archived logs antes e depois dos datafiles
                            garantindo um backup consistente e recuperável
```

O Oracle automaticamente:
1. Arquiva o log atual **antes** do backup dos datafiles
2. Faz backup dos archived logs existentes
3. Faz backup dos datafiles
4. Arquiva o log atual **depois** do backup dos datafiles
5. Faz backup dos novos archived logs
6. Executa o **autobackup do controlfile e SPFILE**

### Parâmetros RMAN configurados

| Parâmetro                                        | Valor configurado                | Finalidade                                      |
|--------------------------------------------------|----------------------------------|-------------------------------------------------|
| `RETENTION POLICY`                               | RECOVERY WINDOW OF 7 DAYS        | Manter backups suficientes para 7 dias de PITR  |
| `CONTROLFILE AUTOBACKUP`                         | ON                               | Autobackup após cada backup e ALTER DATABASE     |
| `DEVICE TYPE DISK PARALLELISM`                   | 2 BACKUP TYPE TO BACKUPSET       | 2 canais paralelos (ORA_DISK_1 e ORA_DISK_2)    |

---

## 3. Estado Inicial — Antes do Backup

```sql
SQL> SELECT log_mode, open_mode, db_unique_name FROM v$database;

LOG_MODE     OPEN_MODE            DB_UNIQUE_NAME
------------ -------------------- ------------------------------
ARCHIVELOG   READ WRITE           orcl
```

```sql
SQL> SELECT name, space_limit/1024/1024/1024 AS limit_gb,
            space_used/1024/1024 AS used_mb
     FROM v$recovery_file_dest;

NAME                                        LIMIT_GB   USED_MB
----------------------------------------- ---------  --------
/u01/app/oracle/fast_recovery_area                50    14.410
```

```sql
SQL> SELECT sum(bytes)/1024/1024/1024 AS datafiles_gb FROM v$datafile;

DATAFILES_GB
------------
    3.101
```

**Conclusões do estado inicial:**
- Banco em ARCHIVELOG / READ WRITE — backup online possível sem interrupção de serviço
- FRA configurada com 50 GB, apenas 14.41 MB usados (archivelogs do Desafio 2)
- 3.1 GB de datafiles para backup — estimativa de saída: ~2-3 GB

---

## 4. Configuração dos Parâmetros RMAN

```rman
RMAN> CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
new RMAN configuration parameters are successfully stored

RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;
new RMAN configuration parameters are successfully stored

RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM 2 BACKUP TYPE TO BACKUPSET;
new RMAN configuration parameters are successfully stored
```

> **Retenção:** `RECOVERY WINDOW OF 7 DAYS` garante que o RMAN mantenha backups suficientes para permitir o restore de qualquer ponto dos últimos 7 dias. O `DELETE OBSOLETE` remove apenas o que ultrapassa essa janela.

> **Autobackup do controlfile:** Com `CONTROLFILE AUTOBACKUP ON`, o RMAN grava automaticamente controlfile + SPFILE ao final de cada sessão de backup — essencial para recovery em caso de perda do controlfile.

> **Paralelismo:** Dois canais (ORA_DISK_1 e ORA_DISK_2) permitem que diferentes datafiles sejam gravados simultaneamente, reduzindo o tempo total de backup.

---

## 5. Execução do Backup

### Comando executado

```rman
BACKUP TAG 'FULL_20260516' DATABASE PLUS ARCHIVELOG;
```

### Sequência de operações executadas pelo RMAN

**Fase 1 — Archived logs pré-backup (sequences 31–33)**
```
channel ORA_DISK_1: starting archived log backup set → seq 33
channel ORA_DISK_2: starting archived log backup set → seq 31, 32
→ 2 pieces gerados em paralelo
```

**Fase 2 — Datafiles (banco aberto, hot backup)**
```
channel ORA_DISK_1: input datafile → system01.dbf (890 MB) + users01.dbf (5 MB)   [CDB]
channel ORA_DISK_2: input datafile → sysaux01.dbf (540 MB) + undotbs01.dbf (325 MB) [CDB]
channel ORA_DISK_2: input datafile → orclpdb/sysaux01.dbf + orclpdb/undotbs01.dbf
channel ORA_DISK_1: input datafile → pdbseed/system01.dbf + pdbseed/undotbs01.dbf
channel ORA_DISK_2: input datafile → pdbseed/sysaux01.dbf
channel ORA_DISK_1: input datafile → orclpdb/system01.dbf + orclpdb/users01.dbf
→ 6 pieces de datafiles em paralelo
```

**Fase 3 — Archived log pós-backup (sequence 34)**
```
channel ORA_DISK_1: starting archived log backup set → seq 34
→ 1 piece de archivelog
```

**Fase 4 — Autobackup do controlfile e SPFILE**
```
piece handle=…/autobackup/2026_05_16/o1_mf_s_1233420698_o0klpc04_.bkp
→ controlfile + SPFILE gravados automaticamente
```

### Output do backup

```
Starting backup at 16-MAY-26
...
Finished backup at 16-MAY-26

Starting Control File and SPFILE Autobackup at 16-MAY-26
piece handle=/u01/app/oracle/fast_recovery_area/ORCL/autobackup/2026_05_16/o1_mf_s_1233420698_o0klpc04_.bkp
Finished Control File and SPFILE Autobackup at 16-MAY-26
```

---

## 6. Validação

### 6.1 LIST BACKUP SUMMARY

```
List of Backups
===============
Key   TY LV S Device Type  Completion Time  #Pieces  #Copies  Compressed  Tag
----- -- -- - -----------  ---------------  -------  -------  ----------  ---
1     B  A  A DISK         16-MAY-26        1        1        NO          FULL_20260516
2     B  A  A DISK         16-MAY-26        1        1        NO          FULL_20260516
3     B  F  A DISK         16-MAY-26        1        1        NO          FULL_20260516
4     B  F  A DISK         16-MAY-26        1        1        NO          FULL_20260516
5     B  F  A DISK         16-MAY-26        1        1        NO          FULL_20260516
6     B  F  A DISK         16-MAY-26        1        1        NO          FULL_20260516
7     B  F  A DISK         16-MAY-26        1        1        NO          FULL_20260516
8     B  F  A DISK         16-MAY-26        1        1        NO          FULL_20260516
9     B  A  A DISK         16-MAY-26        1        1        NO          FULL_20260516
10    B  F  A DISK         16-MAY-26        1        1        NO          TAG20260516T165138
```

> **Legenda:** TY=B (Backupset), LV=F (Full) / A (Archived Log), S=**A (Available)** — todos os pieces disponíveis

### 6.2 LIST BACKUP BY FILE — Datafiles

| File | Key | LV | S | Ckp SCN   | Tag              |
|------|-----|----|---|-----------|------------------|
| 1    | 4   | F  | A | 2188615   | FULL_20260516    |
| 3    | 3   | F  | A | 2188616   | FULL_20260516    |
| 4    | 3   | F  | A | 2188616   | FULL_20260516    |
| 5    | 6   | F  | A | 2165009   | FULL_20260516    |
| 6    | 7   | F  | A | 2165009   | FULL_20260516    |
| 7    | 4   | F  | A | 2188615   | FULL_20260516    |
| 8    | 6   | F  | A | 2165009   | FULL_20260516    |
| 9    | 8   | F  | A | 2188622   | FULL_20260516    |
| 10   | 5   | F  | A | 2188619   | FULL_20260516    |
| 11   | 5   | F  | A | 2188619   | FULL_20260516    |
| 12   | 8   | F  | A | 2188622   | FULL_20260516    |

### 6.3 Archived Logs cobertos

| Thread | Seq# | Low SCN   | Tag              |
|--------|------|-----------|------------------|
| 1      | 31   | 2173913   | FULL_20260516    |
| 1      | 32   | 2182904   | FULL_20260516    |
| 1      | 33   | 2182908   | FULL_20260516    |
| 1      | 34   | 2188593   | FULL_20260516    |

### 6.4 REPORT NEED BACKUP

```
Report of files that must be backed up to satisfy 7 days recovery window
File  Days  Name
----  -----  ----
(nenhum arquivo — política de retenção 100% satisfeita)
```

### 6.5 Uso da FRA após backup

```sql
SELECT name, space_limit/1024/1024/1024 AS limit_gb,
       space_used/1024/1024 AS used_mb, number_of_files
FROM   v$recovery_file_dest;
```

```
NAME                                         LIMIT_GB   USED_MB  NUMBER_OF_FILES
----------------------------------------- ----------  --------  ---------------
/u01/app/oracle/fast_recovery_area                50  2290.982               14
```

```sql
SELECT file_type, percent_space_used, number_of_files
FROM   v$flash_recovery_area_usage
WHERE  number_of_files > 0;
```

```
FILE_TYPE               PERCENT_SPACE_USED  NUMBER_OF_FILES
----------------------- ------------------  ---------------
ARCHIVED LOG                          0.07                4
BACKUP PIECE                           4.4               10
```

### 6.6 v$rman_backup_job_details

```
SESSION_RECID  STATUS     START_TIME   END_TIME   INPUT_BYTES  OUTPUT_BYTES  DEVICE
-------------  ---------  -----------  ---------  -----------  ------------  ------
4              COMPLETED  16-MAY-26    16-MAY-26  2.91 GB      2.18 GB       DISK
```

> **Taxa de compressão implícita:** 2.91 GB → 2.18 GB = ~25% de redução mesmo sem compressão habilitada (blocos Oracle vazios não são gravados)

### 6.7 Arquivos físicos na FRA

```
fast_recovery_area/ORCL/
├── archivelog/2026_05_16/
│   ├── o1_mf_1_31_o0kh2d8o_.arc          (14.4 MB)
│   ├── o1_mf_1_32_o0kh2dfq_.arc          (~0 KB)
│   ├── o1_mf_1_33_o0klokyw_.arc
│   └── o1_mf_1_34_o0klp8p4_.arc
├── autobackup/2026_05_16/
│   └── o1_mf_s_1233420698_o0klpc04_.bkp  (controlfile + SPFILE)
├── backupset/2026_05_16/
│   ├── o1_mf_annnn_FULL_20260516_o0klonss_.bkp   (archivelog seq 33)
│   ├── o1_mf_annnn_FULL_20260516_o0klonv2_.bkp   (archivelog seq 31-32)
│   ├── o1_mf_annnn_FULL_20260516_o0klp985_.bkp   (archivelog seq 34)
│   ├── o1_mf_nnndf_FULL_20260516_o0kloptf_.bkp   (sysaux01+undotbs01 CDB)
│   └── o1_mf_nnndf_FULL_20260516_o0klopvx_.bkp   (system01+users01 CDB)
├── 51F19508DE8E531DE063A0000B0AE59A/backupset/2026_05_16/   [ORCLPDB GUID]
│   ├── o1_mf_nnndf_FULL_20260516_o0klozol_.bkp   (orclpdb/sysaux01+undotbs01)
│   └── o1_mf_nnndf_FULL_20260516_o0klp3kd_.bkp   (orclpdb/system01+users01)
└── 51F1C78CF3925CF8E063A0000B0A8763/backupset/2026_05_16/   [PDB$SEED GUID]
    ├── o1_mf_nnndf_FULL_20260516_o0klozcw_.bkp   (pdbseed/system01+undotbs01)
    └── o1_mf_nnndf_FULL_20260516_o0klp5db_.bkp   (pdbseed/sysaux01)
```

Total: **14 arquivos**, **2.29 GB** ocupados na FRA (4.58% dos 50 GB)

---

## 7. Schema do Banco — Arquivos Cobertos

### Datafiles permanentes (REPORT SCHEMA)

| File# | Tamanho | Tablespace         | Localização                              |
|-------|---------|--------------------|------------------------------------------|
| 1     | 890 MB  | SYSTEM             | /u02/oradata/ORCL/system01.dbf           |
| 3     | 540 MB  | SYSAUX             | /u02/oradata/ORCL/sysaux01.dbf           |
| 4     | 325 MB  | UNDOTBS1           | /u02/oradata/ORCL/undotbs01.dbf          |
| 5     | 270 MB  | PDB$SEED:SYSTEM    | /u02/oradata/ORCL/pdbseed/system01.dbf   |
| 6     | 330 MB  | PDB$SEED:SYSAUX    | /u02/oradata/ORCL/pdbseed/sysaux01.dbf   |
| 7     | 5 MB    | USERS              | /u02/oradata/ORCL/users01.dbf            |
| 8     | 100 MB  | PDB$SEED:UNDOTBS1  | /u02/oradata/ORCL/pdbseed/undotbs01.dbf  |
| 9     | 270 MB  | ORCLPDB:SYSTEM     | /u02/oradata/ORCL/orclpdb/system01.dbf   |
| 10    | 340 MB  | ORCLPDB:SYSAUX     | /u02/oradata/ORCL/orclpdb/sysaux01.dbf   |
| 11    | 100 MB  | ORCLPDB:UNDOTBS1   | /u02/oradata/ORCL/orclpdb/undotbs01.dbf  |
| 12    | 5 MB    | ORCLPDB:USERS      | /u02/oradata/ORCL/orclpdb/users01.dbf    |

**Total de datafiles:** 11 arquivos · **3.175 GB**

### Tempfiles (não cobertos pelo backup — recriados automaticamente)

| File# | Tamanho | Tablespace         |
|-------|---------|--------------------|
| 1     | 32 MB   | TEMP               |
| 2     | 36 MB   | PDB$SEED:TEMP      |
| 3     | 36 MB   | ORCLPDB:TEMP       |

---

## 8. Resumo do Backup

| Item                         | Valor                                              |
|------------------------------|----------------------------------------------------|
| Início                       | 16/05/2026 16:51:08 BRT                            |
| Fim                          | 16/05/2026 16:51:41 BRT                            |
| **Duração total**            | **33 segundos**                                    |
| Tag                          | FULL_20260516                                      |
| Tipo                         | Full (DATABASE PLUS ARCHIVELOG)                    |
| Destino                      | FRA — USE_DB_RECOVERY_FILE_DEST                    |
| Canais utilizados            | 2 (ORA_DISK_1, ORA_DISK_2) em paralelo             |
| Input (datafiles + arclogs)  | **2.91 GB**                                        |
| Output (backupsets)          | **2.18 GB**                                        |
| Redução                      | ~25% (blocos vazios não gravados)                  |
| Pieces gerados               | 10 backupsets + 1 autobackup = 11 pieces           |
| Archived logs cobertos       | Sequences 31, 32, 33, 34                           |
| Controlfile autobackup       | Sim (TAG20260516T165138)                           |
| FRA usada após backup        | 2.29 GB / 50 GB (4.58%)                            |
| Status                       | **COMPLETED / AVAILABLE**                          |
| REPORT NEED BACKUP           | Nenhum arquivo — política de 7 dias satisfeita     |
| Interrupção de serviço       | **Nenhuma** (banco permaneceu OPEN/READ WRITE)     |

---

## 9. Troubleshooting

| Erro | Causa provável | Solução |
|------|----------------|---------|
| `RMAN-06059: expected archived log not found` | Archive perdido entre dois backups | `CROSSCHECK ARCHIVELOG ALL;` + `DELETE EXPIRED ARCHIVELOG ALL;` |
| `ORA-19809: limit exceeded for recovery files` | FRA cheia | `ALTER SYSTEM SET db_recovery_file_dest_size = <novo_valor> SCOPE=BOTH;` ou `RMAN> DELETE OBSOLETE;` |
| `RMAN-03009: failure of backup command` + `ORA-19502` | Disco cheio ou sem permissão no path | Liberar espaço ou ajustar permissões do diretório |
| Backup muito lento | Falta de paralelismo | `CONFIGURE DEVICE TYPE DISK PARALLELISM 4;` (ajustar ao nº de CPUs) |
| `ORA-01031: insufficient privileges` | Usuário sem SYSDBA/SYSBACKUP | Conectar com `rman target /` (OS authentication) |

---

## 10. Manutenção e Monitoramento

### Comandos úteis de manutenção

```rman
-- Validar consistência entre catálogo RMAN e arquivos físicos
CROSSCHECK BACKUP;
CROSSCHECK ARCHIVELOG ALL;

-- Remover backups obsoletos (fora da janela de 7 dias)
DELETE OBSOLETE;

-- Remover registros de backups cujos arquivos não existem mais
DELETE EXPIRED BACKUP;

-- Validar integridade dos blocks sem restaurar
BACKUP VALIDATE DATABASE;

-- Simular restore completo (não altera arquivos)
RESTORE DATABASE VALIDATE;
```

### Monitoramento em SQL*Plus

```sql
-- Histórico de jobs de backup
SELECT session_recid, status, start_time, end_time,
       input_bytes_display, output_bytes_display
FROM   v$rman_backup_job_details
ORDER  BY start_time DESC;

-- Ocupação da FRA por tipo
SELECT file_type, percent_space_used, number_of_files
FROM   v$flash_recovery_area_usage
WHERE  number_of_files > 0;

-- Arquivos que precisam de backup (violam retenção)
-- (via RMAN) REPORT NEED BACKUP;
```

---

## 11. Referências

| Recurso                                        | Descrição                                         |
|------------------------------------------------|---------------------------------------------------|
| Oracle Docs — Backup and Recovery User's Guide | Capítulo "Backing Up the Database"                |
| `V$RMAN_BACKUP_JOB_DETAILS`                    | Histórico e status de jobs de backup RMAN         |
| `V$RECOVERY_FILE_DEST`                         | Uso atual da FRA                                  |
| `V$FLASH_RECOVERY_AREA_USAGE`                  | Ocupação por tipo de arquivo na FRA               |
| Desafio 1: `habilitar-fra`                     | FRA — destino usado pelo backup                   |
| Desafio 2: `habilitar-archivelog`              | ARCHIVELOG — pré-requisito do backup online       |
| Próximo: `backup-compress`                     | Backup comprimido — redução adicional de espaço   |

---

*Documentado automaticamente pelo Claude Code — Imersão Oracle Backup & Recovery com IA*
