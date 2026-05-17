# Desafio 8 — Flashback Database

**Data:** 17/05/2026  
**Ambiente:** ol8-orcl19-ribas.localdomain (Oracle 19c EE, CDB)  
**Objetivo:** Habilitar Flashback Database, criar Guaranteed Restore Point e demonstrar reversão completa do banco após operação destrutiva commitada

---

## O que é Flashback Database

O Flashback Database reverte o **banco inteiro** para um ponto no passado usando **flashback logs** armazenados na FRA — sem restore de backup, sem aplicar archived logs manualmente. É a tecnologia Oracle mais rápida para reverter operações destrutivas em massa.

| Técnica | Usa backup? | Escopo | Velocidade | Caso de uso |
|---------|-------------|--------|-----------|-------------|
| Flashback Query | Não | Linha/tabela | Imediato | Ver dado como estava |
| Flashback Table | Não | Tabela | Segundos | Reverter 1 tabela |
| **Flashback Database** | **Não** | **Banco inteiro** | **Minutos** | **Reverter tudo** |
| Restore + Recovery | Sim | Banco inteiro | Horas | Casos graves, sem flashback |

---

## Estado inicial verificado

| Parâmetro | Valor antes |
|-----------|-------------|
| FLASHBACK_ON | NO |
| LOG_MODE | ARCHIVELOG |
| db_flashback_retention_target | 1440 min (24h) — já configurado |
| FRA size | 50 GB |
| FRA livre | ~91,3% (45,6 GB) |
| FLASHBACK LOG uso | 0% |

---

## Passo 1 — Habilitar Flashback Database

No Oracle 19c (12c+), é possível habilitar com o banco **OPEN** — sem downtime:

```sql
ALTER DATABASE FLASHBACK ON;
```

**Saída:** `Database altered.`  
**Tempo:** 1 segundo (16:57:54 → 16:57:55)

> **Caminho clássico** (versões antigas ou em caso de erro):
> ```sql
> SHUTDOWN IMMEDIATE;
> STARTUP MOUNT;
> ALTER DATABASE FLASHBACK ON;
> ALTER DATABASE OPEN;
> ```

**Validação imediata:**
```sql
SELECT flashback_on FROM v$database;
-- YES
```

---

## Passo 2 — Criar Guaranteed Restore Point

Um **Guaranteed Restore Point** garante que o banco pode voltar exatamente àquele ponto, **independente** do `db_flashback_retention_target`. Ideal para janelas de teste, deploys e treinamentos.

```sql
CREATE RESTORE POINT ANTES_FLASHBACK_TEST GUARANTEE FLASHBACK DATABASE;
```

**Resultado:**
```
Name:    ANTES_FLASHBACK_TEST
SCN:     2585375
Criado:  17-MAY-26 16:58:14
Garantido: YES
Tamanho: 50 MB
```

> **Atenção:** Restore points garantidos consomem espaço na FRA enquanto existirem. Removê-los quando não mais necessários: `DROP RESTORE POINT ANTES_FLASHBACK_TEST;`

---

## Passo 3 — Cenário de desastre simulado

### Estado capturado antes do restore point

```sql
ALTER SESSION SET CONTAINER = orclpdb;
SELECT COUNT(*) FROM hr.employees;        -- 107 funcionários
SELECT employee_id, first_name, salary FROM hr.employees WHERE employee_id = 100;
-- 100 | Steven | King | 24000
```

### Operação destrutiva commitada

```sql
-- UPDATE em massa: salários multiplicados por 10 em toda a tabela
UPDATE hr.employees SET salary = salary * 10;
COMMIT;

-- Resultado pós-desastre
SELECT employee_id, first_name, salary FROM hr.employees WHERE employee_id = 100;
-- 100 | Steven | King | 240000  ← dado corrompido
```

107 registros corrompidos. Operação commitada — nem ROLLBACK nem Flashback Table resolvem (FK constraint impediria outros workarounds parciais).

---

## Passo 4 — Executar Flashback Database

```sql
-- Banco precisa estar em MOUNT
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;

-- Flashback para o restore point garantido
FLASHBACK DATABASE TO RESTORE POINT ANTES_FLASHBACK_TEST;

-- Abrir com RESETLOGS (nova encarnação)
ALTER DATABASE OPEN RESETLOGS;

-- PDB abre automaticamente com RESETLOGS no 19c
-- (ORA-65019 "already open" é esperado se tentar abrir manualmente)
```

**Tempo total do flashback:** ~1 minuto (16:59:08 → 17:00:09)

| Sub-fase | Duração |
|----------|---------|
| SHUTDOWN IMMEDIATE | ~5 s |
| STARTUP MOUNT | ~20 s |
| FLASHBACK DATABASE | ~5 s |
| ALTER DATABASE OPEN RESETLOGS | ~30 s |

---

## Validação

```sql
-- Banco voltou ao estado original
SELECT flashback_on, open_mode, log_mode FROM v$database;
-- YES | READ WRITE | ARCHIVELOG

-- Dados revertidos na PDB
ALTER SESSION SET CONTAINER = orclpdb;
SELECT COUNT(*) FROM hr.employees;             -- 107 (idêntico ao antes)
SELECT employee_id, salary FROM hr.employees WHERE employee_id = 100;
-- 100 | 24000  ← salário original restaurado

-- Uso da FRA por flashback logs
SELECT file_type, percent_space_used FROM v$flash_recovery_area_usage
WHERE file_type = 'FLASHBACK LOG';
-- FLASHBACK LOG | 0.20% (100 MB de 50 GB)

-- Restore point ainda existe (garantido — persiste após RESETLOGS)
SELECT name, guarantee_flashback_database FROM v$restore_point;
-- ANTES_FLASHBACK_TEST | YES
```

---

## Pontos-chave aprendidos

1. **Habilitação sem downtime no Oracle 19c** — `ALTER DATABASE FLASHBACK ON` com banco OPEN. Sem impacto para usuários conectados.

2. **Guaranteed Restore Point é o diferencial para deploys** — garante o ponto de retorno independente da janela de retenção. Sem ele, o flashback só é possível dentro da janela definida pelo `db_flashback_retention_target`.

3. **Flashback Database não usa backup** — usa apenas os flashback logs da FRA. Neste lab: apenas 100 MB (0,2% da FRA de 50 GB) foram consumidos.

4. **OPEN RESETLOGS gera nova encarnação** — assim como no Duplicate (Desafio 7), um novo DBID não é gerado, mas a encarnação muda. Backups anteriores ao RESETLOGS ficam inacessíveis sem configuração de catálogo.

5. **PDB abre automaticamente no 19c** — o `ALTER DATABASE OPEN RESETLOGS` já abre as PDBs em modo READ WRITE, dispensando o `ALTER PLUGGABLE DATABASE ALL OPEN` manual.

6. **Flashback logs são leves e rápidos** — diferente de archived logs, os flashback logs registram apenas blocos alterados em intervalos, tornando a operação muito mais rápida que restore + recovery.

7. **Restore points garantidos persistem após RESETLOGS** — o restore point `ANTES_FLASHBACK_TEST` ainda existe e poderia ser usado para novo flashback se necessário.
