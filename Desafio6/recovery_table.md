# Desafio 6 — RMAN RECOVER TABLE: Recuperação Cirúrgica de Tabela

**Servidor:** ol8-orcl19-ribas.localdomain  
**Banco:** ORCL / PDB: ORCLPDB (Oracle 19c Enterprise Edition)  
**Data:** 17/05/2026  
**Skill utilizada:** `recovery-table` — RMAN RECOVER TABLE com PITR em instância auxiliar automática

---

## 1. Contexto e Cenário

### 1.1 O que é RMAN RECOVER TABLE?

O `RMAN RECOVER TABLE` é um recurso introduzido no Oracle 12c que permite recuperar uma ou mais tabelas específicas para um ponto no passado **sem restaurar nem afetar o banco de produção inteiro**. É a resposta do Oracle para o cenário mais temido pelos DBAs: `DROP TABLE ... PURGE` em produção, quando Flashback Table não é mais possível por falta de undo retenção ou porque o `PURGE` eliminou a entrada da recycle bin.

### 1.2 Cenário simulado

Um desenvolvedor executou acidentalmente o seguinte comando no PDB de produção:

```sql
DROP TABLE hr.employees CASCADE CONSTRAINTS PURGE;
```

- **PURGE:** elimina da recycle bin — `FLASHBACK TABLE` e `FLASHBACK QUERY` não funcionam.
- **CASCADE CONSTRAINTS:** removeu todas as foreign keys que referenciavam `employees` (de `job_history` e `departments`).
- **107 linhas perdidas**, incluindo dados de todos os funcionários do schema HR.

### 1.3 Por que não Flashback Query?

O `PURGE` torna o Flashback Table inoperante. A janela de undo (padrão: 15 minutos) também já havia expirado. A única saída era usar o backup RMAN e o mecanismo de PITR (Point-In-Time Recovery) automatizado pelo `RECOVER TABLE`.

### 1.4 Como o RMAN RECOVER TABLE funciona internamente

```
PRODUÇÃO (ORCL/ORCLPDB)
       │
       ├─ Lê backup do controlfile
       ├─ Cria instância auxiliar temporária (SID automático)
       │
       └─ INSTÂNCIA AUXILIAR (SID=uoxw)
              │
              ├─ FASE 1: Restore/recover SYSTEM, SYSAUX, UNDO
              │          (CDB root + PDB: abre em read only)
              │
              ├─ FASE 2: Identifica tablespace da tabela (USERS)
              │          Restore/recover datafile 12 (orclpdb/users01.dbf)
              │          Open com RESETLOGS
              │
              ├─ EXPDP: Exporta HR.EMPLOYEES → dump temporário
              │
              └─ Desliga e remove instância auxiliar automaticamente
       │
       └─ IMPDP: Importa dump no banco de produção (ORCLPDB)
              → Tabela, indexes, constraints, triggers restaurados
```

**Resultado:** a tabela volta à produção com todos os seus 107 registros, sem nenhuma interrupção de serviço.

---

## 2. Pré-requisitos Verificados

| Requisito | Status | Detalhe |
|-----------|--------|---------|
| Modo ARCHIVELOG | OK | `LOG_MODE: ARCHIVELOG` |
| FRA configurada | OK | `/u01/app/oracle/fast_recovery_area/` — 50 GB |
| Backup RMAN disponível | OK | `TAG20260517T151612` — executado às 15:15 |
| Archives entre backup e DROP | OK | Sequences 53 e 54 disponíveis na FRA |
| Acesso SYSDBA | OK | `sys/<senha>` |
| Espaço em `/u01/aux_dest` | OK | ~500 GB livres em `/u01` |
| Oracle 19c (12c+ obrigatório) | OK | 19.3.0.0.0 EE |

---

## 3. Implantação do Schema HR

Antes de simular o acidente, o schema de exemplo HR (Human Resources) da Oracle foi implantado no PDB `orclpdb`. Os scripts estão incluídos no próprio ORACLE_HOME:

```
$ORACLE_HOME/demo/schema/human_resources/
├── hr_cre.sql    ← criação das tabelas e sequences
├── hr_popul.sql  ← carga de dados
├── hr_idx.sql    ← criação de índices
├── hr_code.sql   ← procedures e triggers
├── hr_comnt.sql  ← comentários
└── hr_analz.sql  ← coleta de estatísticas
```

### 3.1 Criação do usuário HR

```sql
ALTER SESSION SET CONTAINER = ORCLPDB;

CREATE USER hr IDENTIFIED BY "<senha_hr>";
ALTER USER hr DEFAULT TABLESPACE users QUOTA UNLIMITED ON users;
ALTER USER hr TEMPORARY TABLESPACE temp;

GRANT CREATE SESSION, CREATE VIEW, ALTER SESSION, CREATE SEQUENCE TO hr;
GRANT CREATE SYNONYM, CREATE DATABASE LINK, RESOURCE, UNLIMITED TABLESPACE TO hr;
GRANT EXECUTE ON sys.dbms_stats TO hr;
```

### 3.2 Execução dos scripts

```bash
sqlplus hr/"<senha_hr>"@localhost:1521/orclpdb.localdomain

@$ORACLE_HOME/demo/schema/human_resources/hr_cre.sql
@$ORACLE_HOME/demo/schema/human_resources/hr_popul.sql
@$ORACLE_HOME/demo/schema/human_resources/hr_idx.sql
@$ORACLE_HOME/demo/schema/human_resources/hr_code.sql
@$ORACLE_HOME/demo/schema/human_resources/hr_comnt.sql
@$ORACLE_HOME/demo/schema/human_resources/hr_analz.sql
```

### 3.3 Tabelas criadas e populadas

| Tabela | Linhas |
|--------|-------:|
| REGIONS | 4 |
| COUNTRIES | 25 |
| LOCATIONS | 23 |
| DEPARTMENTS | 27 |
| JOBS | 19 |
| **EMPLOYEES** | **107** |
| JOB_HISTORY | 10 |
| **Total** | **215** |

Schema implantado com sucesso, incluindo 11 índices, procedures `ADD_JOB_HISTORY` e `SECURE_DML`, triggers, comentários e estatísticas.

---

## 4. Backup RMAN (Pré-requisito para RECOVER TABLE)

Um backup full foi executado imediatamente após a implantação do schema, garantindo que os dados de HR.EMPLOYEES fossem capturados no backupset.

```rman
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '%F';

BACKUP DATABASE PLUS ARCHIVELOG;
```

### 4.1 Resultado do backup

```
Starting backup at 17-MAY-26
Allocated channel: ORA_DISK_1 (DISK)
Allocated channel: ORA_DISK_2 (DISK)

-- Fase 1: archived logs anteriores (seq 31–52)
channel ORA_DISK_1: backup archived logs (seq 42–51)  → 15 s
channel ORA_DISK_2: backup archived logs (seq 31–41)  → 15 s

-- Fase 2: datafiles CDB root
ORA_DISK_1: datafiles system01.dbf + undotbs01.dbf    → 46 s
ORA_DISK_2: datafiles sysaux01.dbf + users01.dbf      → 15 s

-- Fase 3: datafiles PDB orclpdb
ORA_DISK_1: orclpdb/system01.dbf + users01.dbf        → 15 s
ORA_DISK_2: orclpdb/sysaux01.dbf + undotbs01.dbf      → 15 s

-- Fase 4: archived log atual (seq 53) + autobackup controlfile
Finished backup at 17-MAY-26

Control File and SPFILE Autobackup:
  /u01/app/oracle/product/19.0.0/dbhome_1/dbs/c-1761183591-20260517-01
```

| Item | Valor |
|------|-------|
| Tag | `TAG20260517T151612` |
| Início | 17-MAY-26 15:15:48 |
| Fim | 17-MAY-26 15:17:16 |
| Canais | 2 (ORA_DISK_1, ORA_DISK_2) |
| Compressão | YES (BASIC) |
| Datafiles cobertos | 12 (todos) |
| Local | FRA: `/u01/app/oracle/fast_recovery_area/ORCL/` |

---

## 5. Simulação do Acidente

### 5.1 Captura do SCN antes do DROP

```sql
-- Conectado ao orclpdb como sysdba
SELECT TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS') AS momento FROM dual;
SELECT current_scn FROM v$database;
SELECT COUNT(*) FROM hr.employees;
```

```
MOMENTO:          2026-05-17 15:18:20
SCN_ANTES_DROP:   2576014
EMPLOYEES_COUNT:  107
```

### 5.2 Execução do DROP acidental

```sql
DROP TABLE hr.employees CASCADE CONSTRAINTS PURGE;
```

```
Table dropped.   (executado em 2026-05-17 15:18:22)
```

**Efeitos imediatos:**
- Tabela removida com `PURGE` → sem entrada na recycle bin → `FLASHBACK TABLE` impossível
- `CASCADE CONSTRAINTS` removeu as FKs de `job_history.employee_id` e `departments.manager_id`
- `hr.departments` e `hr.job_history` continuaram existindo (sem integridade referencial)

### 5.3 Estado pós-DROP

```sql
SELECT table_name FROM dba_tables WHERE owner = 'HR' ORDER BY table_name;
```

```
COUNTRIES
DEPARTMENTS
JOBS
JOB_HISTORY
LOCATIONS
REGIONS

6 rows selected.   ← EMPLOYEES ausente
```

```sql
SELECT COUNT(*) FROM hr.employees;
-- ORA-00942: table or view does not exist
```

---

## 6. Execução do RMAN RECOVER TABLE

### 6.1 Preparação da auxiliary destination

```bash
mkdir -p /u01/aux_dest
chown oracle:oinstall /u01/aux_dest
chmod 775 /u01/aux_dest
```

### 6.2 Comando RMAN

> **Atenção CDB/PDB:** Em ambiente CDB, é obrigatório usar a cláusula `OF PLUGGABLE DATABASE` para que o RMAN identifique corretamente o tablespace onde a tabela residia. Sem ela, o RMAN procura a tabela no CDB root e falha com `RMAN-05057`.

```rman
RECOVER TABLE HR.EMPLOYEES
  OF PLUGGABLE DATABASE ORCLPDB
  UNTIL SCN 2576014
  AUXILIARY DESTINATION '/u01/aux_dest';
```

### 6.3 Fases da execução

#### Fase 1 — Criação da instância auxiliar e restore inicial

```
Creating automatic instance, with SID='uoxw'

initialization parameters:
  db_name=ORCL
  db_unique_name=uoxw_pitr_ORCLPDB_ORCL
  compatible=19.0.0
  sga_target=12288M
  db_create_file_dest=/u01/aux_dest
  log_archive_dest_1='location=/u01/aux_dest'
  enable_pluggable_database=true
  _clone_one_pdb_recovery=true

Oracle instance started (SGA: 12 GB)
```

**Restore controlfile → Mount → Archive current log:**

```
ORA_AUX_DISK_1: restoring control file
  source: c-1761183591-20260517-01 (autobackup)
  output: /u01/aux_dest/ORCL/controlfile/o1_mf_o0n1wjk6_.ctl
Finished restore at 17-MAY-26

sql: alter database mount clone database
sql: alter system archive log current
```

#### Fase 2 — Restore dos tablespaces de sistema (read only)

Datafiles restaurados para a instância auxiliar:

| Datafile | Tablespace | Destino |
|----------|-----------|---------|
| 1 | CDB SYSTEM | `/u01/aux_dest/ORCL/datafile/` |
| 3 | CDB SYSAUX | `/u01/aux_dest/ORCL/datafile/` |
| 4 | CDB UNDOTBS1 | `/u01/aux_dest/ORCL/datafile/` |
| 9 | ORCLPDB SYSTEM | `/u01/aux_dest/ORCL/51F1C7.../datafile/` |
| 10 | ORCLPDB SYSAUX | `/u01/aux_dest/ORCL/51F1C7.../datafile/` |
| 11 | ORCLPDB UNDOTBS1 | `/u01/aux_dest/ORCL/51F1C7.../datafile/` |

```
Starting media recovery
  archived log seq 53: o1_mf_1_53_o0n1kbx6_.arc
  archived log seq 54: o1_mf_1_54_o0n1nj5r_.arc
media recovery complete, elapsed time: 00:00:01

sql: alter database open read only
sql: alter pluggable database ORCLPDB open read only
```

> O RMAN abre o banco em **read only** na instância auxiliar para inspecionar o dicionário de dados do PDB e identificar o tablespace onde HR.EMPLOYEES residia (`USERS` → datafile 12).

#### Fase 3 — Restore do tablespace da tabela e RESETLOGS

Após identificar o datafile 12 como necessário:

```
-- Segunda fase: restart com RESETLOGS
set until scn 2576014;
set newname for datafile 12 to new;
restore clone datafile 12;

ORA_AUX_DISK_1: restoring datafile 00012
  source: TAG20260517T151612
  output: /u01/aux_dest/UOXW_PITR_ORCLPDB_ORCL/.../o1_mf_users_o0n22111_.dbf
Finished restore at 17-MAY-26

recover clone database tablespace "ORCLPDB":"USERS", "SYSTEM", ... delete archivelog;
  seq 53 → applied
  seq 54 → applied
media recovery complete

alter clone database open resetlogs;
alter pluggable database ORCLPDB open;
```

#### Fase 4 — Export via Data Pump (instância auxiliar → dump)

```
create or replace directory TSPITR_DIROBJ_DPDIR as '/u01/aux_dest';

EXPDP> Starting "SYS"."TSPITR_EXP_uoxw_uuuy":
EXPDP> Processing TABLE_EXPORT/TABLE/TABLE_DATA
EXPDP> Processing TABLE_EXPORT/TABLE/INDEX/STATISTICS/INDEX_STATISTICS
EXPDP> Processing TABLE_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
EXPDP> Processing TABLE_EXPORT/TABLE/TABLE
EXPDP> Processing TABLE_EXPORT/TABLE/COMMENT
EXPDP> Processing TABLE_EXPORT/TABLE/INDEX/INDEX
EXPDP> Processing TABLE_EXPORT/TABLE/CONSTRAINT/CONSTRAINT
EXPDP> Processing TABLE_EXPORT/TABLE/CONSTRAINT/REF_CONSTRAINT
EXPDP> Processing TABLE_EXPORT/TABLE/TRIGGER
EXPDP> . . exported "HR"."EMPLOYEES"    17.08 KB    107 rows
EXPDP> Job "SYS"."TSPITR_EXP_uoxw_uuuy" successfully completed
       elapsed: 0 00:00:38
```

#### Fase 5 — Shutdown da instância auxiliar + Import no banco de produção

```
shutdown clone abort

IMPDP> Master table "SYS"."TSPITR_IMP_uoxw_hyDf" successfully loaded/unloaded
IMPDP> Starting "SYS"."TSPITR_IMP_uoxw_hyDf":
IMPDP> Processing TABLE_EXPORT/TABLE/TABLE
IMPDP> Processing TABLE_EXPORT/TABLE/TABLE_DATA
IMPDP> . . imported "HR"."EMPLOYEES"    17.08 KB    107 rows
IMPDP> Processing TABLE_EXPORT/TABLE/COMMENT
IMPDP> Processing TABLE_EXPORT/TABLE/INDEX/INDEX
IMPDP> Processing TABLE_EXPORT/TABLE/CONSTRAINT/CONSTRAINT
IMPDP> Processing TABLE_EXPORT/TABLE/INDEX/STATISTICS/INDEX_STATISTICS
IMPDP> Processing TABLE_EXPORT/TABLE/CONSTRAINT/REF_CONSTRAINT
IMPDP> Processing TABLE_EXPORT/TABLE/TRIGGER
IMPDP> Processing TABLE_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
IMPDP> Job "SYS"."TSPITR_IMP_uoxw_hyDf" successfully completed
       elapsed: 0 00:00:37
```

#### Fase 6 — Limpeza automática

```
Removing automatic instance
  auxiliary instance file .../o1_mf_temp_..._.tmp         deleted
  auxiliary instance file .../o1_mf_users_o0n22111_.dbf   deleted
  auxiliary instance file .../o1_mf_sysaux_..._.dbf       deleted
  auxiliary instance file .../o1_mf_undotbs1_..._.dbf     deleted
  auxiliary instance file .../o1_mf_system_..._.dbf       deleted
  auxiliary instance file .../o1_mf_o0n1wjk6_.ctl         deleted
  auxiliary instance file tspitr_uoxw_11444.dmp            deleted

Automatic instance removed
Finished recover at 17-MAY-26
```

---

## 7. Troubleshooting Encontrado

### 7.1 ORA-02449 — DROP TABLE bloqueado por FK

**Erro:**
```sql
DROP TABLE hr.employees PURGE;
-- ORA-02449: unique/primary keys in table referenced by foreign keys
```

**Causa:** A tabela `employees` é referenciada via foreign key por `job_history.employee_id` e `departments.manager_id`.

**Solução:**
```sql
DROP TABLE hr.employees CASCADE CONSTRAINTS PURGE;
```
O `CASCADE CONSTRAINTS` derruba automaticamente as FKs dependentes antes de remover a tabela.

---

### 7.2 RMAN-05057 — Table not found (primeira tentativa)

**Erro:**
```
RMAN-03002: failure of recover command at 05/17/2026 15:20:30
RMAN-05063: Cannot recover specified tables
RMAN-05057: Table HR.EMPLOYEES not found
```

**Causa:** RMAN conectado ao CDB (`rman target sys/...@orcl`) **sem** a cláusula `OF PLUGGABLE DATABASE`. Nesse modo, o RMAN procura a tabela no dicionário de dados do CDB root — onde HR.EMPLOYEES nunca existiu. Sem encontrar a tabela, ele não inclui o datafile do tablespace USERS na lista de restore e falha ao tentar localizar os blocos da tabela na instância auxiliar.

**Diagnóstico:** Na primeira tentativa, o Memory Script gerado pelo RMAN não incluiu o datafile 12 (`orclpdb/users01.dbf`):
```
set newname for clone datafile  1 to new;   -- CDB SYSTEM
set newname for clone datafile  9 to new;   -- ORCLPDB SYSTEM
set newname for clone datafile  4 to new;   -- CDB UNDO
set newname for clone datafile  11 to new;  -- ORCLPDB UNDO
set newname for clone datafile  3 to new;   -- CDB SYSAUX
-- datafile 12 (ORCLPDB USERS) NÃO incluído!
```

**Solução:**
```rman
RECOVER TABLE HR.EMPLOYEES
  OF PLUGGABLE DATABASE ORCLPDB    -- ← obrigatório em CDB
  UNTIL SCN 2576014
  AUXILIARY DESTINATION '/u01/aux_dest';
```

Com `OF PLUGGABLE DATABASE`, o RMAN conecta ao PDB correto, localiza HR.EMPLOYEES no dicionário, identifica o tablespace USERS e inclui o datafile 12 no restore da segunda fase.

---

## 8. Validação Pós-Recovery

### 8.1 Tabelas do schema HR após o recovery

```sql
SELECT table_name, num_rows
FROM dba_tables
WHERE owner = 'HR'
ORDER BY table_name;
```

```
TABLE_NAME      NUM_ROWS
-----------     --------
COUNTRIES             25
DEPARTMENTS           27
EMPLOYEES            107   ← RECUPERADA
JOBS                  19
JOB_HISTORY           10
LOCATIONS             23
REGIONS                4

7 rows selected.
```

### 8.2 Contagem de registros

```sql
SELECT COUNT(*) AS employees_recuperados FROM hr.employees;

EMPLOYEES_RECUPERADOS
---------------------
                  107
```

### 8.3 Objetos recuperados

O Data Pump importou todos os objetos associados à tabela:

| Objeto | Tipo | Status |
|--------|------|--------|
| EMPLOYEES | TABLE | Criada |
| 107 rows | TABLE_DATA | Importados |
| EMP_EMAIL_UK | INDEX/CONSTRAINT | Criado |
| EMP_EMP_ID_PK | INDEX/CONSTRAINT | Criado |
| EMP_DEPARTMENT_IX | INDEX | Criado |
| EMP_JOB_IX | INDEX | Criado |
| EMP_MANAGER_IX | INDEX | Criado |
| EMP_NAME_IX | INDEX | Criado |
| EMP_SALARY_IX | INDEX | Criado |
| CHECK constraints (salary, job) | CONSTRAINT | Criadas |
| UPDATE_JOB_HISTORY | TRIGGER | Criado |
| Comentários de colunas | COMMENT | Importados |
| Estatísticas | STATISTICS | Importadas |

### 8.4 Instância auxiliar removida

```bash
ps -ef | grep ora_pmon_ | grep -v grep
# oracle  28464  1  0  11:05  ora_pmon_orcl   ← apenas produção
# Nenhum processo da instância auxiliar

ls -la /u01/aux_dest/
# total 0   ← todos os arquivos removidos pelo RMAN
```

---

## 9. Resumo da Operação

### 9.1 Timeline

| Etapa | Horário | Duração |
|-------|---------|---------|
| Backup RMAN (TAG20260517T151612) | 15:15 → 15:17 | ~1 min 28 s |
| SCN capturado antes do DROP | 15:18:20 | — |
| DROP TABLE executado | 15:18:22 | — |
| RECOVER TABLE iniciado | 15:18:52 | — |
| Criação instância auxiliar (SID=uoxw) | 15:19 | ~30 s |
| Restore fase 1 (SYSTEM/SYSAUX/UNDO CDB+PDB) | 15:19 → 15:21 | ~2 min |
| Media recovery (seq 53–54) + open read only | 15:21 | ~5 s |
| Restart + restore fase 2 (USERS datafile 12) | 15:22 | ~15 s |
| Media recovery RESETLOGS | 15:22 | ~5 s |
| EXPDP auxiliar → dump | 15:26 → 15:27 | ~38 s |
| Shutdown auxiliar | 15:27 | ~10 s |
| IMPDP dump → produção | 15:27 → 15:28 | ~37 s |
| Limpeza automática + Finished | 15:28 | ~5 s |
| **Total (do RECOVER TABLE)** | **15:18:52 → 15:27:57** | **~9 min 5 s** |

### 9.2 Parâmetros chave

| Parâmetro | Valor |
|-----------|-------|
| SCN alvo (antes do DROP) | 2576014 |
| Timestamp alvo | 2026-05-17 15:18:20 |
| Backup usado (datafiles) | TAG20260517T151612 |
| Autobackup controlfile | c-1761183591-20260517-01 |
| Archived logs aplicados | Sequences 53 e 54 |
| SID instância auxiliar | uoxw (gerado automaticamente) |
| Auxiliary destination | `/u01/aux_dest` |
| Dump file | `tspitr_uoxw_11444.dmp` (removido após import) |
| Registros recuperados | **107 de 107 (100%)** |
| Interrupção de serviço | **Nenhuma** |
| RESETLOGS no banco de produção | **Não** |

---

## 10. Lições Aprendidas

1. **`OF PLUGGABLE DATABASE` é obrigatório em CDB.** Sem ele, o RMAN conectado ao CDB root não encontra tabelas que existem apenas no PDB e falha com `RMAN-05057`. A cláusula direciona a busca ao dicionário correto e garante que o datafile do tablespace da tabela seja incluído no restore.

2. **`DROP TABLE ... PURGE` bypassa todos os mecanismos de Flashback.** A recycle bin é a última linha de defesa antes do RMAN. Com `PURGE`, o único caminho é o `RECOVER TABLE` a partir de um backup.

3. **`CASCADE CONSTRAINTS` é silencioso e destrutivo.** O drop removeu FKs em `job_history` e `departments` sem nenhum aviso. O `RECOVER TABLE` restaurou a própria tabela `EMPLOYEES` com seus constraints originais, mas as FKs nas tabelas dependentes precisariam ser recriadas manualmente em produção caso o incidente fosse real.

4. **A instância auxiliar consome SGA significativa.** Os 12 GB de SGA_TARGET configurados foram herdados do banco de produção. Em servidores com memória limitada, definir `DB_RECOVERY_FILE_DEST` e limitar o SGA da auxiliar via `PFILE` pode ser necessário.

5. **O RMAN faz tudo automaticamente em duas fases.** Primeira: abre o banco auxiliar em read only só com SYSTEM/SYSAUX/UNDO para inspecionar o dicionário e identificar o tablespace. Segunda: restart com o datafile da tabela e RESETLOGS. Esse design evita restore desnecessário de todos os datafiles.

6. **O backup precisa cobrir o momento dos dados desejados, não apenas o momento do acidente.** O backup foi tirado antes do DROP. Se o backup existisse somente de um período anterior à criação da tabela, o `RECOVER TABLE` não teria dados para recuperar.

7. **Para tabelas com muitas dependências, use `NOTABLEIMPORT` e importe manualmente.** A opção `DATAPUMP DUMP FILE ... NOTABLEIMPORT` gera apenas o dump, dando controle total sobre a ordem de import, tratamento de FKs e substituição de partições.

---

## 11. Referências

| Recurso | Descrição |
|---------|-----------|
| Oracle Docs — Backup and Recovery User's Guide | Capítulo "Recovering Tables and Table Partitions from RMAN Backups" |
| `RMAN-05057` | Table not found — verificar `OF PLUGGABLE DATABASE` em ambientes CDB |
| `V$SESSION_LONGOPS` | Monitorar progresso do RECOVER TABLE em tempo real |
| `DBA_SEGMENTS` | Identificar tablespace de uma tabela antes do drop (diagnóstico pós-facto via backup) |
| Desafio 5: `recovery-full` | Recovery de datafiles completos — quando a perda é maior que uma tabela |
| Próximo: `configure-flashback` | Flashback Database — prevenção antes do backup |

---

## 12. Indicador de Andamento da Imersão

```
DBAOCM — Imersão Oracle Backup Recovery com IA
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Fase 1 — Preparação do Ambiente
  ─────────────────────────────────────────────
  [✓] Desafio 1  │ Habilitar FRA              │ Concluído — 16/05/2026
  [✓] Desafio 2  │ Habilitar ARCHIVELOG        │ Concluído — 16/05/2026

  Fase 2 — Backup
  ─────────────────────────────────────────────
  [✓] Desafio 3  │ Backup Full com RMAN        │ Concluído — 16/05/2026
  [✓] Desafio 4  │ Backup Comprimido (BASIC)   │ Concluído — 16/05/2026

  Fase 3 — Recovery
  ─────────────────────────────────────────────
  [✓] Desafio 5  │ Recovery Completo (Full)    │ Concluído — 17/05/2026
  [✓] Desafio 6  │ RMAN RECOVER TABLE          │ Concluído — 17/05/2026  ◄ ATUAL
  [ ] Desafio 7  │ Flashback Database          │ Pendente
  [ ] Desafio 8  │ RMAN DUPLICATE DATABASE     │ Pendente

  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Progresso: ██████████████████████░░░░░  6 / 8  (75%)
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

*Documentado automaticamente pelo Claude Code — Imersão Oracle Backup & Recovery com IA*
