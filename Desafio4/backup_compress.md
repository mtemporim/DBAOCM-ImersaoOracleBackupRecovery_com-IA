# Oracle 19c — Backup Comprimido com RMAN

> **Ambiente:** Máquina virtual VMware com Oracle Linux 8.10 x86-64  
> **Data de execução:** 16 de maio de 2026  
> **Autor:** Documentado automaticamente pelo Claude Code  
> **Status:** Backup comprimido concluído e validado com sucesso

---

## Sumário

1. [Descrição do Ambiente](#1-descrição-do-ambiente)
2. [O que é o Backup Comprimido com RMAN](#2-o-que-é-o-backup-comprimido-com-rman)
3. [Estado Inicial — Configuração RMAN existente](#3-estado-inicial--configuração-rman-existente)
4. [Configuração da Compressão](#4-configuração-da-compressão)
5. [Execução do Backup Comprimido](#5-execução-do-backup-comprimido)
6. [Validação e Comparativo](#6-validação-e-comparativo)
7. [Análise do Trade-off CPU × Espaço](#7-análise-do-trade-off-cpu--espaço)
8. [Restauração da Configuração RMAN](#8-restauração-da-configuração-rman)
9. [Resumo do Backup](#9-resumo-do-backup)
10. [Troubleshooting](#10-troubleshooting)
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
| SID / PDB           | orcl / orclpdb                                       |
| FRA Path            | /u01/app/oracle/fast_recovery_area (50 GB)           |
| Modo do banco       | ARCHIVELOG / READ WRITE                              |
| **Algoritmo usado** | BASIC (incluído na Enterprise Edition)               |
| **Tag do backup**   | COMPRESS_BASIC_20260516                              |

---

## 2. O que é o Backup Comprimido com RMAN

O RMAN suporta compressão nativa de backupsets, reduzindo o espaço em disco sem necessidade de ferramentas externas. A compressão atua em nível de **bloco Oracle** — blocos vazios não são gravados mesmo sem compressão; com compressão, os blocos com dados são adicionalmente comprimidos.

### Algoritmos disponíveis

| Algoritmo | Velocidade  | Taxa de compressão | CPU    | Licença                    |
|-----------|-------------|--------------------|--------|----------------------------|
| `BASIC`   | Moderada    | Alta               | Alto   | Enterprise Edition (grátis)|
| `LOW`     | Muito rápida| Baixa              | Baixo  | Advanced Compression Option|
| `MEDIUM`  | Equilibrada | Média              | Médio  | Advanced Compression Option|
| `HIGH`    | Lenta       | Muito alta         | Muito  | Advanced Compression Option|

> **Regra prática:** `BASIC` é a escolha padrão para EE sem licença adicional. `MEDIUM` é o equilíbrio ideal quando a licença ACO está disponível.

### Como funciona o BASIC

O algoritmo `BASIC` usa **zlib** (deflate) — o mesmo algoritmo do gzip. Ele opera em streaming sobre os blocos Oracle antes de gravá-los no backupset:

```
Datafile → leitura de blocos Oracle → filtragem de blocos vazios
         → compressão zlib por canal → gravação no backupset (.bkp)
```

---

## 3. Estado Inicial — Configuração RMAN existente

Configuração herdada do Desafio 3 (backup-full):

```rman
RMAN> SHOW ALL;

CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE DEVICE TYPE DISK PARALLELISM 2 BACKUP TYPE TO BACKUPSET;  ← sem compressão
CONFIGURE COMPRESSION ALGORITHM 'BASIC' AS OF RELEASE 'DEFAULT' OPTIMIZE FOR LOAD TRUE;
```

Backup anterior para referência de comparação:

```
Key  TY  LV  S  Compressed  Tag               Input    Output  Ratio   Duração
---  --  --  -  ----------  ----------------  -------  ------  ------  -------
1-9  B   F   A  NO          FULL_20260516     2.91 GB  2.18 GB  1.34x   22 s
```

FRA antes do backup comprimido:
```
/u01/app/oracle/fast_recovery_area → 2290.98 MB usados / 50 GB (4.58%) — 14 arquivos
```

---

## 4. Configuração da Compressão

### Passo 1 — Definir algoritmo BASIC

```rman
CONFIGURE COMPRESSION ALGORITHM 'BASIC';
```

```
new RMAN configuration parameters:
CONFIGURE COMPRESSION ALGORITHM 'BASIC' AS OF RELEASE 'DEFAULT' OPTIMIZE FOR LOAD TRUE;
new RMAN configuration parameters are successfully stored
```

> `OPTIMIZE FOR LOAD TRUE` instrui o RMAN a priorizar velocidade de compressão sobre taxa — comportamento padrão e recomendado para janelas de backup curtas.

### Passo 2 — Ativar compressão persistente no tipo de backup

```rman
CONFIGURE DEVICE TYPE DISK PARALLELISM 2 BACKUP TYPE TO COMPRESSED BACKUPSET;
```

```
old RMAN configuration parameters:
CONFIGURE DEVICE TYPE DISK PARALLELISM 2 BACKUP TYPE TO BACKUPSET;
new RMAN configuration parameters:
CONFIGURE DEVICE TYPE DISK PARALLELISM 2 BACKUP TYPE TO COMPRESSED BACKUPSET;
new RMAN configuration parameters are successfully stored
```

> Com `COMPRESSED BACKUPSET` como padrão, **todos os backups futuros** nesta sessão e nas próximas usarão compressão automaticamente, sem precisar especificar `AS COMPRESSED BACKUPSET` no comando.

---

## 5. Execução do Backup Comprimido

### Comando executado

```rman
BACKUP TAG 'COMPRESS_BASIC_20260516' DATABASE PLUS ARCHIVELOG;
```

### Sequência de operações

**Fase 1 — Archived logs pré-backup (sequences 31–35)**
```
ORA_DISK_1: compressed archived log backup → seq 31, 32, 33
ORA_DISK_2: compressed archived log backup → seq 34, 35
→ 2 pieces em paralelo
```

**Fase 2 — Datafiles comprimidos (banco OPEN — hot backup)**
```
ORA_DISK_1: system01.dbf (890 MB) + users01.dbf (5 MB)   [CDB]
ORA_DISK_2: sysaux01.dbf (540 MB) + undotbs01.dbf (325 MB) [CDB]
ORA_DISK_2: orclpdb/sysaux01.dbf + orclpdb/undotbs01.dbf
ORA_DISK_2: pdbseed/system01.dbf + pdbseed/undotbs01.dbf
ORA_DISK_1: pdbseed/sysaux01.dbf
ORA_DISK_2: orclpdb/system01.dbf + orclpdb/users01.dbf
→ 6 pieces de datafiles em paralelo
```

**Fase 3 — Archived log pós-backup (sequence 36)**
```
ORA_DISK_1: compressed archived log backup → seq 36
→ 1 piece
```

**Fase 4 — Autobackup do Controlfile + SPFILE**
```
→ autobackup/2026_05_16/o1_mf_s_1233423369_o0ko9scv_.bkp
```

### LIST BACKUP SUMMARY — resultado final

```
Key   TY  LV  S  Compressed  Tag
----  --  --  -  ----------  ----------------------
1     B   A   A  NO          FULL_20260516
...
9     B   A   A  NO          FULL_20260516
10    B   F   A  NO          TAG20260516T165138
11    B   A   A  YES         COMPRESS_BASIC_20260516   ← comprimido
12    B   A   A  YES         COMPRESS_BASIC_20260516   ← comprimido
13    B   F   A  YES         COMPRESS_BASIC_20260516   ← comprimido
14    B   F   A  YES         COMPRESS_BASIC_20260516   ← comprimido
15    B   F   A  YES         COMPRESS_BASIC_20260516   ← comprimido
16    B   F   A  YES         COMPRESS_BASIC_20260516   ← comprimido
17    B   F   A  YES         COMPRESS_BASIC_20260516   ← comprimido
18    B   F   A  YES         COMPRESS_BASIC_20260516   ← comprimido
19    B   A   A  YES         COMPRESS_BASIC_20260516   ← comprimido
20    B   F   A  YES         TAG20260516T173609        ← autobackup
```

---

## 6. Validação e Comparativo

### 6.1 v$rman_backup_job_details — Comparativo direto

```sql
SELECT session_recid, input_type, compression_ratio,
       time_taken_display, input_bytes_display, output_bytes_display, status
FROM   v$rman_backup_job_details
ORDER  BY start_time;
```

| Session | Tipo    | Razão de compressão | Duração  | Input   | Output     | Status    |
|---------|---------|--------------------:|----------|---------|------------|-----------|
| 4       | DB FULL | **1.34×**           | 00:00:22 | 2.91 GB | 2.18 GB    | COMPLETED |
| 16      | DB FULL | **5.75×**           | 00:01:17 | 2.89 GB | **515 MB** | COMPLETED |

### 6.2 Análise da compressão

| Métrica                  | Backup Full (sem compressão) | Backup BASIC         | Diferença               |
|--------------------------|-----------------------------:|---------------------:|------------------------:|
| Input                    | 2.91 GB                      | 2.89 GB              | —                       |
| Output                   | 2.18 GB                      | **515 MB**           | **−76.4%**              |
| Razão de compressão      | 1.34×                        | **5.75×**            | 4.3× melhor             |
| Duração                  | 22 segundos                  | 77 segundos          | +55 s (+2.5× mais lento)|
| Espaço economizado       | —                            | **1.68 GB**          | vs backup sem compressão|
| FRA ocupada após backup  | 2.29 GB (4.58%)              | +526 MB adicionais   | Total: 2.75 GB (5.5%)   |

> **Por que 5.75× de compressão?**  
> O banco foi criado hoje e contém pouquíssimos dados reais. Os datafiles têm **blocos Oracle majoritariamente zerados** — o algoritmo BASIC (zlib) comprime zeros com altíssima eficiência. Em bancos de produção com dados reais, espere razões de **1.5× a 3×** com BASIC.

### 6.3 FRA após ambos os backups

```sql
SELECT name, space_limit/1024/1024/1024 AS limit_gb,
       space_used/1024/1024 AS used_mb, number_of_files
FROM   v$recovery_file_dest;
```

```
NAME                                         LIMIT_GB  USED_MB  NUMBER_OF_FILES
----------------------------------------- ---------- --------  ---------------
/u01/app/oracle/fast_recovery_area                50  2816.51               26
```

```sql
SELECT file_type, percent_space_used, number_of_files
FROM   v$flash_recovery_area_usage WHERE number_of_files > 0;
```

```
FILE_TYPE               PERCENT_SPACE_USED  NUMBER_OF_FILES
----------------------- ------------------  ---------------
ARCHIVED LOG                          0.09                6
BACKUP PIECE                          5.41               20
```

---

## 7. Análise do Trade-off CPU × Espaço

### Comparativo teórico entre algoritmos (referência Oracle)

| Algoritmo      | Velocidade relativa | Taxa típica  | CPU    | Licença | Cenário ideal                           |
|----------------|:-------------------:|:------------:|:------:|:-------:|-----------------------------------------|
| Sem compressão | ★★★★★             | 1.0× – 1.5×  | Nenhum | EE      | Backup mais rápido, disco barato       |
| `BASIC`        | ★★★☆☆             | 2× – 6×      | Alto   | EE      | Sem ACO — maior redução possível       |
| `LOW`          | ★★★★★             | 1.5× – 2.5×  | Baixo  | ACO     | I/O é o gargalo, CPU é abundante       |
| `MEDIUM`       | ★★★★☆             | 2× – 4×      | Médio  | ACO     | Equilíbrio geral — recomendado com ACO |
| `HIGH`         | ★★☆☆☆             | 3× – 7×      | Muito  | ACO     | Armazenamento caro, CPU abundante      |

### Resultado obtido neste ambiente

```
Sem compressão: 2.18 GB  /  22 s  /  razão 1.34×
BASIC:           515 MB  /  77 s  /  razão 5.75×

Economia de espaço:  76.4% a menos que o backup sem compressão
Custo de tempo:      +55 segundos (+2.5× mais lento)
```

**Conclusão:** Para este banco de laboratório, o algoritmo BASIC oferece excelente economia de espaço. Em produção, avalie o impacto de CPU em horários de pico e a janela de backup disponível.

---

## 8. Restauração da Configuração RMAN

Após o exercício, o RMAN foi reconfigurado para **não usar compressão por padrão** (mantendo o comportamento do Desafio 3):

```rman
CONFIGURE DEVICE TYPE DISK PARALLELISM 2 BACKUP TYPE TO BACKUPSET;
```

```
old RMAN configuration parameters:
CONFIGURE DEVICE TYPE DISK PARALLELISM 2 BACKUP TYPE TO COMPRESSED BACKUPSET;
new RMAN configuration parameters:
CONFIGURE DEVICE TYPE DISK PARALLELISM 2 BACKUP TYPE TO BACKUPSET;
new RMAN configuration parameters are successfully stored
```

> Para usar compressão pontualmente sem alterar o padrão, use:  
> `BACKUP AS COMPRESSED BACKUPSET TAG 'MEU_TAG' DATABASE PLUS ARCHIVELOG;`

---

## 9. Resumo do Backup

| Item                              | Valor                                            |
|-----------------------------------|--------------------------------------------------|
| Início                            | 16/05/2026 17:34:45 BRT                          |
| Fim                               | 16/05/2026 17:36:11 BRT                          |
| **Duração total**                 | **77 segundos**                                  |
| Tag                               | COMPRESS_BASIC_20260516                          |
| Algoritmo de compressão           | BASIC (zlib — incluído na EE)                    |
| Canais utilizados                 | 2 (ORA_DISK_1, ORA_DISK_2) em paralelo           |
| Input                             | 2.89 GB                                          |
| **Output (comprimido)**           | **515 MB**                                       |
| **Razão de compressão**           | **5.75× (82% de redução)**                       |
| Economia vs backup sem compressão | 1.68 GB a menos na FRA                           |
| Pieces gerados (Compressed=YES)   | 9 backupsets + 1 autobackup controlfile/SPFILE   |
| FRA total após ambos os backups   | 2.82 GB / 50 GB (5.64%)                          |
| Status                            | **COMPLETED / AVAILABLE**                        |
| Interrupção de serviço            | **Nenhuma** (banco permaneceu OPEN)              |
| Config RMAN após exercício        | Restaurada para BACKUP TYPE TO BACKUPSET         |

---

## 10. Troubleshooting

| Erro                                              | Causa provável                              | Solução                                                           |
|---------------------------------------------------|---------------------------------------------|-------------------------------------------------------------------|
| `ORA-19918: Advanced Compression Option required` | Tentou usar LOW/MEDIUM/HIGH sem licença ACO | Usar `CONFIGURE COMPRESSION ALGORITHM 'BASIC';`                   |
| Backup mais lento que o esperado                  | CPU saturada pela compressão BASIC          | Reduzir paralelismo ou trocar para `LOW` (com ACO)                |
| Taxa de compressão baixa (< 1.5×)                 | Dados já comprimidos (LOBs, JPEG, ZIP)      | Normal — dados comprimidos não comprimem mais                     |
| `ORA-19809: limit exceeded for recovery files`    | FRA cheia (ambos backups + archivelogs)     | `RMAN> DELETE OBSOLETE;` ou aumentar `db_recovery_file_dest_size` |

### Verificar razão de compressão por backup

```sql
SELECT session_recid, input_type,
       ROUND(compression_ratio, 2) AS ratio,
       time_taken_display,
       input_bytes_display,
       output_bytes_display
FROM   v$rman_backup_job_details
ORDER  BY start_time DESC;
```

### Reverter compressão para não-padrão

```rman
CONFIGURE COMPRESSION ALGORITHM CLEAR;
CONFIGURE DEVICE TYPE DISK BACKUP TYPE TO BACKUPSET;
```

---

## 11. Referências

| Recurso                                        | Descrição                                                |
|------------------------------------------------|----------------------------------------------------------|
| Oracle Docs — Backup and Recovery User's Guide | Capítulo "Compressing RMAN Backups"                      |
| `V$RMAN_BACKUP_JOB_DETAILS`                    | Coluna `COMPRESSION_RATIO` — razão de compressão por job |
| `V$BACKUP_PIECE`                               | Tamanho de cada piece individual                         |
| Desafio 3: `backup-full`                       | Backup full de referência (sem compressão)               |
| Próximo: `configure-flashback`                 | Flashback Database                                       |

---

*Documentado automaticamente pelo Claude Code — Imersão Oracle Backup & Recovery com IA*
