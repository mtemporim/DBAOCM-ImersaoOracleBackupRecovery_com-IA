# Desafio 7 — Duplicate Database com RMAN

**Data:** 17/05/2026  
**Ambiente:** ol8-orcl19-ribas.localdomain (Oracle 19c EE, CDB)  
**Objetivo:** Criar um clone (DUPDB) do banco ORCL no mesmo servidor usando RMAN DUPLICATE backup-based

---

## Cenário

| Item | Source | Clone |
|------|--------|-------|
| DB_NAME | ORCL | DUPDB |
| DB_UNIQUE_NAME | ORCL | DUPDB |
| DBID | 1761183591 | 958719387 |
| SID | orcl | dupdb |
| Datafiles | /u02/oradata/ORCL/ | /u02/oradata/DUPDB/ |
| Redo logs | /u02/oradata/ORCL/ | /u02/oradata/DUPDB/ |
| SGA | 12 GB | 4 GB |
| PGA | 4 GB | 1 GB |
| PDB | orclpdb (READ WRITE) | orclpdb (READ WRITE) |

**Modalidade:** Backup-based duplicate (mesmo servidor, mesmo ORACLE_HOME)  
**Backup utilizado:** TAG20260517T151612 (backup comprimido do dia, COMPRESS_POSRECOVERY_20260517)  
**Archived logs aplicados:** seq 53, 54, 55, 56, 57

---

## Pré-requisitos verificados

- [x] Oracle 19c EE instalado e compartilhado (mesmo ORACLE_HOME)
- [x] Banco source ORCL em ARCHIVELOG e OPEN READ WRITE
- [x] Backup RMAN recente disponível na FRA
- [x] Espaço em disco suficiente: /u02 com 493 GB livres
- [x] RAM suficiente: 62 GB (30 GB livres após ORCL com 12 GB SGA)

---

## Passo 1 — Criar diretórios para DUPDB

```bash
mkdir -p /u01/app/oracle/admin/DUPDB/adump
mkdir -p /u01/app/oracle/admin/DUPDB/dpdump
mkdir -p /u02/oradata/DUPDB
mkdir -p /u01/app/oracle/fast_recovery_area/DUPDB
```

---

## Passo 2 — Criar pfile mínimo (initdupdb.ora)

`$ORACLE_HOME/dbs/initdupdb.ora`:

```ini
DB_NAME=DUPDB
DB_UNIQUE_NAME=DUPDB
ENABLE_PLUGGABLE_DATABASE=TRUE
DB_BLOCK_SIZE=8192
SGA_TARGET=4G
PGA_AGGREGATE_TARGET=1G
CONTROL_FILES='/u02/oradata/DUPDB/control01.ctl','/u02/oradata/DUPDB/control02.ctl'
DB_RECOVERY_FILE_DEST='/u01/app/oracle/fast_recovery_area'
DB_RECOVERY_FILE_DEST_SIZE=20G
DIAGNOSTIC_DEST='/u01/app/oracle'
AUDIT_FILE_DEST='/u01/app/oracle/admin/DUPDB/adump'
REMOTE_LOGIN_PASSWORDFILE=EXCLUSIVE
```

> `ENABLE_PLUGGABLE_DATABASE=TRUE` é obrigatório para duplicar um CDB.

---

## Passo 3 — Criar password file

```bash
orapwd file=$ORACLE_HOME/dbs/orapwdupdb password=<senha_sys> format=12
```

> A senha deve ser **idêntica** à do SYS do banco source, pois o RMAN vai se autenticar no target e no auxiliary com a mesma senha.

---

## Passo 4 — Atualizar listener.ora (entrada estática obrigatória)

O DUPDB não existe ainda, então não consegue registrar-se dinamicamente. É necessária uma entrada **estática** no `SID_LIST_LISTENER`:

```
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = orcl.localdomain)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = orcl)
    )
    (SID_DESC =
      (GLOBAL_DBNAME = dupdb.localdomain)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = dupdb)
    )
  )
```

```bash
lsnrctl reload
```

---

## Passo 5 — Atualizar tnsnames.ora

Adicionar entrada para DUPDB com `(UR=A)` — permite conectar mesmo em NOMOUNT:

```
DUPDB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = ol8-orcl19-ribas.localdomain)(PORT = 1521))
    (CONNECT_DATA =
      (SID = dupdb)
      (UR=A)
    )
  )
```

> Usar `SID =` (não `SERVICE_NAME =`) porque o banco ainda não registrou serviços dinâmicos em NOMOUNT.

---

## Passo 6 — Iniciar DUPDB em NOMOUNT

```bash
export ORACLE_SID=dupdb
sqlplus / as sysdba
```

```sql
STARTUP NOMOUNT PFILE='/u01/app/oracle/product/19.0.0/dbhome_1/dbs/initdupdb.ora';
```

**Saída:**
```
ORACLE instance started.

Total System Global Area 4294964720 bytes
Fixed Size                  9143792 bytes
Variable Size             855638016 bytes
Database Buffers         3422552064 bytes
Redo Buffers               7630848 bytes
```

**Verificação do listener:**
```
Service "dupdb.localdomain" has 1 instance(s).
  Instance "dupdb", status UNKNOWN, has 1 handler(s) for this service...
```

> `status UNKNOWN` é o comportamento esperado para instância em NOMOUNT (sem registro dinâmico).

---

## Passo 7 — Executar RMAN DUPLICATE

Conectar com `ORACLE_SID=dupdb` (OS auth vai para o auxiliary):

```bash
export ORACLE_SID=dupdb
rman
```

```rman
CONNECT TARGET 'sys/<senha_sys>@orcl';
CONNECT AUXILIARY /;

DUPLICATE DATABASE TO DUPDB
  SPFILE
    SET DB_NAME='DUPDB'
    SET DB_UNIQUE_NAME='DUPDB'
    SET DB_FILE_NAME_CONVERT='/u02/oradata/ORCL','/u02/oradata/DUPDB'
    SET LOG_FILE_NAME_CONVERT='/u02/oradata/ORCL','/u02/oradata/DUPDB'
    SET CONTROL_FILES='/u02/oradata/DUPDB/control01.ctl','/u02/oradata/DUPDB/control02.ctl'
    SET DB_RECOVERY_FILE_DEST='/u01/app/oracle/fast_recovery_area'
    SET DB_RECOVERY_FILE_DEST_SIZE='20G'
    SET SGA_TARGET='4G'
    SET PGA_AGGREGATE_TARGET='1G'
    SET AUDIT_FILE_DEST='/u01/app/oracle/admin/DUPDB/adump'
    SET DIAGNOSTIC_DEST='/u01/app/oracle'
  NOFILENAMECHECK;
```

### O que o RMAN fez internamente (Memory Scripts)

1. **Restore SPFILE** → `/u01/app/oracle/product/19.0.0/dbhome_1/dbs/spfiledupdb.ora`
2. **Aplicar parâmetros SPFILE** via `ALTER SYSTEM SET` (db_name, file_name_convert, control_files, SGA, etc.)
3. **Shutdown/Startup NOMOUNT** com novo SPFILE
4. **Restore controlfile** do backup → `/u02/oradata/DUPDB/control01.ctl` e `control02.ctl`
5. **ALTER DATABASE MOUNT**
6. **Restore datafiles** (2 canais paralelos, 11 datafiles — CDB root + pdbseed + orclpdb)
   - Backup: TAG20260517T151612
7. **RECOVER database** — archived logs seq 53–57 aplicados
8. **ALTER DATABASE OPEN RESETLOGS** — nova encarnação, novo DBID gerado
9. **ALTER PLUGGABLE DATABASE ALL OPEN** — orclpdb aberto automaticamente

### Tempo de execução

| Fase | Início | Fim | Duração |
|------|--------|-----|---------|
| RMAN START | 16:18:18 | — | — |
| Restore SPFILE + controlfile | 16:18:20 | 16:18:30 | ~10 s |
| Restore datafiles (11 files, 2 canais) | 16:18:30 | 16:21:30 | ~3 min |
| Recovery (5 archived logs) | 16:21:30 | 16:21:35 | ~5 s |
| OPEN RESETLOGS + PDB OPEN | 16:21:35 | 16:22:05 | ~30 s |
| **Total** | **16:18:18** | **16:22:05** | **~3 min 47 s** |

---

## Validação

```sql
-- No DUPDB (ORACLE_SID=dupdb)

-- 1. Identidade do banco
SELECT name, db_unique_name, dbid, open_mode, log_mode, cdb FROM v$database;
-- DUPDB | DUPDB | 958719387 | READ WRITE | ARCHIVELOG | YES
-- DBID diferente do source (1761183591) — correto: RESETLOGS gera novo DBID

-- 2. Instância
SELECT instance_name, status FROM v$instance;
-- dupdb | OPEN

-- 3. Datafiles — todos em /u02/oradata/DUPDB/
SELECT name FROM v$datafile ORDER BY 1;
-- 11 datafiles confirmados

-- 4. PDBs
SELECT con_id, name, open_mode FROM v$pdbs ORDER BY con_id;
-- CON_ID=2: PDB$SEED READ ONLY
-- CON_ID=3: ORCLPDB  READ WRITE

-- 5. Objetos válidos
SELECT COUNT(*) FROM dba_objects WHERE status='VALID';
-- 72.552

-- 6. Teste funcional — switch logfile
ALTER SYSTEM SWITCH LOGFILE;
-- System altered.
```

---

## Pontos-chave aprendidos

1. **Entrada estática no listener é obrigatória** para duplicate no mesmo servidor — sem ela, o RMAN não consegue conectar no auxiliary em NOMOUNT (ORA-12528).

2. **`(UR=A)` no tnsnames** permite conexão mesmo com instância em estado não-aberto.

3. **`CONNECT AUXILIARY /`** com `ORACLE_SID=dupdb` usa OS auth local — sem precisar de senha no comando.

4. **`DB_FILE_NAME_CONVERT`** e **`LOG_FILE_NAME_CONVERT`** são suficientes quando source e destino estão no mesmo servidor — o prefixo `/u02/oradata/ORCL` cobre todos os subdirs (root, pdbseed, orclpdb).

5. **`NOFILENAMECHECK`** é necessário no mesmo servidor para o RMAN não reclamar que os nomes de datafile do source e do clone são diferentes do spfile original.

6. **RESETLOGS gera novo DBID** — o clone não é mais o ORCL. Isso é esperado e necessário para que os dois bancos coexistam na mesma FRA.

7. **PDBs são clonados automaticamente** — o RMAN executa `ALTER PLUGGABLE DATABASE ALL OPEN` ao final sem intervenção.

8. **Backup comprimido funciona normalmente** para duplicate — o RMAN descomprime durante o restore transparentemente.
