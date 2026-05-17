# Desafio 5 — Recovery Full: Restauração após Perda de Datafiles

**Servidor:** ol8-orcl19-ribas.localdomain  
**Banco:** ORCL (Oracle 19c Enterprise Edition)  
**Data:** 17/05/2026  
**Skill utilizada:** `recovery-full` — Cenário 2 (Perda completa de datafiles, controlfile OK)

---

## 1. Simulação do Desastre

Para simular a perda de datafiles críticos do CDB root, os seguintes arquivos foram removidos manualmente do disco:

```bash
rm -rf /u02/oradata/ORCL/sysaux01.dbf
rm -rf /u02/oradata/ORCL/system01.dbf
rm -rf /u02/oradata/ORCL/undotbs01.dbf
rm -rf /u02/oradata/ORCL/users01.dbf
```

**Observação:** A instância permaneceu `OPEN` imediatamente após a exclusão, pois os blocos dos datafiles estavam em cache no SGA (buffer cache). A falha só se manifestou no momento do restart.

---

## 2. Diagnóstico Inicial

### 2.1 Arquivos Presentes no Disco Após a Exclusão

```
/u02/oradata/ORCL/
├── control01.ctl          (18M) — OK
├── control02.ctl          (18M) — OK
├── redo01.log             (51M) — OK
├── redo02.log             (51M) — OK
├── redo03.log             (51M) — OK
├── temp01.dbf             (33M) — OK
├── orclpdb/               — OK
└── pdbseed/               — OK

AUSENTES:
  system01.dbf    ← TABLESPACE SYSTEM (FILE# 1)
  sysaux01.dbf    ← TABLESPACE SYSAUX  (FILE# 3)
  undotbs01.dbf   ← TABLESPACE UNDOTBS (FILE# 4)
  users01.dbf     ← TABLESPACE USERS   (FILE# 7)
```

### 2.2 Estado Após Restart (SHUTDOWN ABORT + STARTUP)

O banco montou com sucesso (controlfile íntegro) mas **falhou ao abrir** os datafiles:

```
ORACLE instance started.
Database mounted.
ORA-01157: cannot identify/lock data file 1 - see DBWR trace file
ORA-01110: data file 1: '/u02/oradata/ORCL/system01.dbf'
```

### 2.3 Erros no Alert Log

```
ORA-01157: cannot identify/lock data file 1
ORA-01110: data file 1: '/u02/oradata/ORCL/system01.dbf'
ORA-27037: unable to obtain file status
Linux-x86_64 Error: 2: No such file or directory

ORA-01157: cannot identify/lock data file 3
ORA-01110: data file 3: '/u02/oradata/ORCL/sysaux01.dbf'
ORA-27037: unable to obtain file status

ORA-01157: cannot identify/lock data file 4
ORA-01110: data file 4: '/u02/oradata/ORCL/undotbs01.dbf'
ORA-27037: unable to obtain file status

ORA-01157: cannot identify/lock data file 7
ORA-01110: data file 7: '/u02/oradata/ORCL/users01.dbf'
ORA-27037: unable to obtain file status

Checker run found 4 new persistent data failures
```

### 2.4 Estado da Instância (pré-recovery)

| Atributo | Valor |
|----------|-------|
| instance_name | orcl |
| status | MOUNTED |
| database_status | ACTIVE |
| open_mode | MOUNTED |
| log_mode | ARCHIVELOG |
| controlfile | OK |
| v$recover_file | 4 datafiles pendentes |

### 2.5 Determinação do Cenário

| Controlfile | Datafiles CDB Root | Cenário |
|-------------|-------------------|---------|
| **OK** | **4 ausentes** | **Cenário 2 — Restore completo (controlfile OK)** |

---

## 3. Plano de Recovery

O banco está em `MOUNT`, controlfile íntegro, sem necessidade de `RESETLOGS`. O procedimento é:

```
RESTORE DATABASE  →  RECOVER DATABASE  →  ALTER DATABASE OPEN
```

**Backup selecionado pelo RMAN:** `COMPRESS_BASIC_20260516` (backup comprimido, o mais recente disponível)

---

## 4. Execução do Recovery

### 4.1 Backups Disponíveis

```
List of Backups
===============
Key  TY LV S Device  Completion      #Pieces Compressed Tag
---  -- -- - ------  --------------  ------- ---------- ---
1    B  A  A DISK    16-MAY-26       1       NO         FULL_20260516
3    B  F  A DISK    16-MAY-26       1       NO         FULL_20260516
11   B  A  A DISK    16-MAY-26       1       YES        COMPRESS_BASIC_20260516
13   B  F  A DISK    16-MAY-26       1       YES        COMPRESS_BASIC_20260516  ← usado
```

### 4.2 RESTORE DATABASE

```sql
RESTORE DATABASE;
```

**Execução:**
```
Starting restore at 17-MAY-26
allocated channel: ORA_DISK_1 (device type=DISK)
allocated channel: ORA_DISK_2 (device type=DISK)

skipping datafile 5  — pdbseed/system01.dbf    (já presente)
skipping datafile 6  — pdbseed/sysaux01.dbf     (já presente)
skipping datafile 8  — pdbseed/undotbs01.dbf    (já presente)

ORA_DISK_1: restoring datafile 3 → sysaux01.dbf
ORA_DISK_1: restoring datafile 4 → undotbs01.dbf
  source: COMPRESS_BASIC_20260516
  restore complete, elapsed time: 00:00:22

ORA_DISK_2: restoring datafile 1 → system01.dbf
ORA_DISK_2: restoring datafile 7 → users01.dbf
  source: COMPRESS_BASIC_20260516
  restore complete, elapsed time: 00:00:52

ORA_DISK_1: restoring datafile 9 → orclpdb/system01.dbf
ORA_DISK_1: restoring datafile 12 → orclpdb/users01.dbf
  restore complete, elapsed time: 00:00:15

ORA_DISK_2: restoring datafile 10 → orclpdb/sysaux01.dbf
ORA_DISK_2: restoring datafile 11 → orclpdb/undotbs01.dbf
  restore complete, elapsed time: 00:00:15

Finished restore at 17-MAY-26
```

### 4.3 RECOVER DATABASE

```sql
RECOVER DATABASE;
```

**Archived logs aplicados (sequences 36 a 46):**
```
Starting recover at 17-MAY-26
starting media recovery

seq 36  — archivelog 2026-05-16  o1_mf_1_36_o0ko9q6s_.arc
seq 37  — archivelog 2026-05-16  o1_mf_1_37_o0ky0r41_.arc
seq 38  — archivelog 2026-05-17  o1_mf_1_38_o0lhbhwl_.arc
seq 39  — archivelog 2026-05-17  o1_mf_1_39_o0m0xjlm_.arc
seq 40  — archivelog 2026-05-17  o1_mf_1_40_o0m0ytmc_.arc
seq 41  — archivelog 2026-05-17  o1_mf_1_41_o0m11cs5_.arc
seq 42  — archivelog 2026-05-17  o1_mf_1_42_o0m13mw2_.arc
seq 43  — archivelog 2026-05-17  o1_mf_1_43_o0md2603_.arc
seq 44  — archivelog 2026-05-17  o1_mf_1_44_o0mgzhgy_.arc
seq 45  — archivelog 2026-05-17  o1_mf_1_45_o0mh0bjo_.arc
seq 46  — archivelog 2026-05-17  o1_mf_1_46_o0mh2okt_.arc

media recovery complete, elapsed time: 00:00:20
Finished recover at 17-MAY-26
```

### 4.4 ALTER DATABASE OPEN

```sql
ALTER DATABASE OPEN;
```

**Resultado:** `Database altered.`

Sem necessidade de `RESETLOGS` (controlfile estava íntegro).

---

## 5. Validação Pós-Recovery

### 5.1 Estado do Banco

| Atributo | Valor |
|----------|-------|
| instance_name | orcl |
| status | **OPEN** |
| database_status | ACTIVE |
| open_mode | **READ WRITE** |
| log_mode | ARCHIVELOG |

### 5.2 Datafiles

| FILE# | Nome | Status |
|-------|------|--------|
| 1 | /u02/oradata/ORCL/system01.dbf | SYSTEM |
| 3 | /u02/oradata/ORCL/sysaux01.dbf | ONLINE |
| 4 | /u02/oradata/ORCL/undotbs01.dbf | ONLINE |
| 5 | /u02/oradata/ORCL/pdbseed/system01.dbf | SYSTEM |
| 6 | /u02/oradata/ORCL/pdbseed/sysaux01.dbf | ONLINE |
| 7 | /u02/oradata/ORCL/users01.dbf | ONLINE |
| 8 | /u02/oradata/ORCL/pdbseed/undotbs01.dbf | ONLINE |
| 9 | /u02/oradata/ORCL/orclpdb/system01.dbf | SYSTEM |
| 10 | /u02/oradata/ORCL/orclpdb/sysaux01.dbf | ONLINE |
| 11 | /u02/oradata/ORCL/orclpdb/undotbs01.dbf | ONLINE |
| 12 | /u02/oradata/ORCL/orclpdb/users01.dbf | ONLINE |

### 5.3 Arquivos Pendentes de Recovery

```sql
SELECT * FROM v$recover_file;
-- no rows selected   ← OK
```

### 5.4 Teste Funcional

```sql
ALTER SYSTEM SWITCH LOGFILE;     -- System altered.
SELECT count(*) FROM dba_objects; -- 72.387 objetos
```

---

## 6. Resumo da Operação

| Etapa | Início | Fim | Duração |
|-------|--------|-----|---------|
| Diagnóstico + Restart | 11:05 | 11:06 | ~1 min |
| RESTORE DATABASE | 11:06:12 | 11:07:30 | ~1 min 18 s |
| RECOVER DATABASE | 11:07:30 | 11:07:50 | ~20 s |
| ALTER DATABASE OPEN | 11:07:50 | 11:08:04 | ~14 s |
| **Total** | **11:06:11** | **11:08:04** | **~2 min** |

**Backup utilizado:** `COMPRESS_BASIC_20260516` (backup comprimido de 16/05/2026)  
**Archived logs aplicados:** sequences 36 a 46  
**Paralelismo:** 2 canais (ORA_DISK_1 + ORA_DISK_2)  
**RESETLOGS:** Não necessário (controlfile preservado)

---

## 7. Lições Aprendidas

1. **O banco permanece OPEN após perda de datafiles** enquanto os blocos estiverem em buffer cache — a falha só se manifesta no próximo startup.
2. **Controlfile íntegro = recovery mais simples**: sem necessidade de SET DBID, RESTORE SPFILE, RESTORE CONTROLFILE ou RESETLOGS.
3. **Modo ARCHIVELOG é obrigatório**: os 11 archived logs aplicados trouxeram o banco ao ponto exato antes da falha, sem perda de dados.
4. **FRA como local de backup**: todos os archived logs e backupsets estavam disponíveis localmente, eliminando a necessidade de busca em fita ou repositório externo.
5. **Paralelismo 2 reduz o tempo de restore**: os canais ORA_DISK_1 e ORA_DISK_2 restauraram datafiles simultaneamente.
