# Oracle Database 19c — Guia Completo de Instalação Automatizada em Oracle Linux 8.10 (VMware)

> **Ambiente:** Máquina virtual VMware com Oracle Linux 8.10 x86-64 — Servidor Ribas  
> **Data de execução:** 16 de maio de 2026  
> **Autor:** Documentado automaticamente pelo Claude Code  
> **Status:** Instalação concluída e validada com sucesso

---

## Sumário

1. [Descrição do Ambiente](#1-descrição-do-ambiente)
2. [Pré-requisitos de Hardware e Software](#2-pré-requisitos-de-hardware-e-software)
3. [Downloads Necessários](#3-downloads-necessários)
4. [Criação e Configuração da VM no VMware](#4-criação-e-configuração-da-vm-no-vmware)
5. [Instalação do Oracle Linux 8.10](#5-instalação-do-oracle-linux-810)
6. [Preparação do Linux para o Oracle Database](#6-preparação-do-linux-para-o-oracle-database)
7. [Transferência e Descompactação do Instalador](#7-transferência-e-descompactação-do-instalador)
8. [Instalação Silenciosa do Oracle Database 19c](#8-instalação-silenciosa-do-oracle-database-19c)
9. [Criação do Banco de Dados via DBCA](#9-criação-do-banco-de-dados-via-dbca)
10. [Configuração Pós-Instalação](#10-configuração-pós-instalação)
11. [Validação da Instalação](#11-validação-da-instalação)
12. [Comparativo Antes × Depois da Instalação](#12-comparativo-antes--depois-da-instalação)
13. [Resumo da Instalação](#13-resumo-da-instalação)
14. [Troubleshooting](#14-troubleshooting)
15. [Referências e Links Úteis](#15-referências-e-links-úteis)

---

## 1. Descrição do Ambiente

| Componente         | Detalhe                                                      |
|--------------------|--------------------------------------------------------------|
| Hypervisor         | VMware (full virtualization)                                 |
| Sistema Operacional| Oracle Linux Server release 8.10                             |
| Kernel             | 5.15.0-320.202.8.3.el8uek.x86_64 (UEK)                       |
| Arquitetura        | x86_64                                                       |
| IP da VM           | x.x.x.x                                                      |
| Hostname           | ol8-orcl19-ribas.localdomain                                 |
| Interface de rede  | ens33 (configuração estática manual)                         |
| Produto Oracle     | Oracle Database 19c Enterprise Edition (19.3.0.0.0)          |
| ORACLE_BASE        | /u01/app/oracle                                              |
| ORACLE_HOME        | /u01/app/oracle/product/19.0.0/dbhome_1                      |
| Global DB Name     | orcl.localdomain                                             |
| SID                | orcl                                                         |
| PDB                | orclpdb                                                      |
| Character Set      | AL32UTF8                                                     |
| Datafiles          | /u02/oradata                                                 |

> **Nota sobre a interface de rede:** O prompt original especificava a interface `ens160`, porém neste
> servidor VMware a interface foi criada como `ens33`. O endereço IP estático (x.x.x.x/21) já
> estava configurado corretamente via `nmcli` com `ipv4.method: manual`. Não foi necessária
> reconfiguração da rede.

---

## 2. Pré-requisitos de Hardware e Software

### Hardware da VM

| Recurso         | Valor                                     |
|-----------------|-------------------------------------------|
| CPU             | AMD Opteron(tm) Processor 6344            |
| Núcleos lógicos | 8 vCPU (1 socket × 8 cores)              |
| Frequência      | 2600 MHz                                  |
| RAM             | 62 GB (62Gi disponível ao OS)             |
| Swap            | 15 GB                                     |
| Disco /         | 62 GB (disk1-vg1-root)                    |
| Disco /u01      | 499 GB (disk2-vg1-u01)                    |
| Disco /u02      | 500 GB (disk3-vg1-u02)                    |
| Virtualização   | VMware (full virtualization)              |

### Requisitos Mínimos do Oracle Database 19c

| Recurso              | Mínimo Recomendado | Esta VM         |
|----------------------|--------------------|-----------------|
| RAM                  | 2 GB               | 62 GB ✓         |
| Swap (RAM < 2 GB)    | 2× RAM             | 15 GB ✓         |
| Disco /tmp           | 1 GB               | 32 GB tmpfs ✓   |
| Disco ORACLE_HOME    | 6.4 GB             | 499 GB (/u01) ✓ |
| Espaço em /u02       | Variável           | 500 GB ✓        |
| CPUs                 | 1 (mín.)           | 8 ✓             |

### Software do Sistema Operacional

| Pacote                          | Versão instalada                        |
|---------------------------------|-----------------------------------------|
| oracle-database-preinstall-19c  | 1.0-2.el8.x86_64                        |
| make                            | 4.2.1-11.el8.x86_64                     |
| net-tools                       | 2.0-0.52.20160912git.el8.x86_64         |
| nfs-utils                       | 2.3.3-68.0.1.el8_10.x86_64              |
| sysstat                         | 11.7.3-13.0.2.el8_10.x86_64             |
| unzip                           | 6.0-48.0.1.el8_10.x86_64               |
| ksh                             | 20120801-271.0.1.el8_10.x86_64          |
| binutils                        | 2.30-128.0.1.el8_10.x86_64             |
| glibc-devel                     | 2.28-251.0.4.el8_10.34.x86_64          |
| libaio-devel                    | 0.3.112-1.el8.x86_64                   |

---

## 3. Downloads Necessários

| Arquivo           | Descrição                                      | Tamanho | Origem                  |
|-------------------|------------------------------------------------|---------|-------------------------|
| V982063-01.zip    | Oracle Database 19c (19.3.0.0) Linux x86-64    | ~2.9 GB | My Oracle Support (MOS) |
| Oracle Linux 8.10 | OracleLinux-R8-U10-x86_64-dvd.iso              | ~10 GB  | edelivery.oracle.com    |

**Nota:** O arquivo `V982063-01.zip` corresponde ao `LINUX.X64_193000_db_home.zip` — mesmo
conteúdo, nome de entrega diferente conforme o canal de download (MOS vs. edelivery).

**URL MOS:** https://support.oracle.com → Downloads → Oracle Database 19c (19.3.0.0) for Linux x86-64  
**URL edelivery:** https://edelivery.oracle.com → Oracle Database 19c

---

## 4. Criação e Configuração da VM no VMware

### Especificações da VM

**Passo 1 — Criar nova máquina virtual:**
- Abra o VMware Workstation/Fusion
- Selecione: `File → New Virtual Machine → Typical (recommended)`
- Guest OS: `Linux` → `Oracle Linux 8 64-bit`
- Nome da VM: `OL8-ORCL19-RIBAS`

**Passo 2 — Configurar hardware:**
```
Processadores: 8 vCPUs (1 socket × 8 cores)
Memória: 62 GB (mínimo recomendado: 16 GB)
Disco principal (/): 70 GB (sistema operacional)
Disco adicional (/u01): 500 GB (ORACLE_HOME e inventário)
Disco adicional (/u02): 500 GB (datafiles, redo logs, control files)
Rede: Bridge ou VLAN específica (IP estático x.x.x.x/21)
```

**Passo 3 — Configurar BIOS/UEFI:**
- Habilitar virtualização de hardware (AMD-V)
- UEFI firmware recomendado para OL 8

**Passo 4 — Montar ISO:**
- CD/DVD → Use ISO image file → selecionar `OracleLinux-R8-U10-x86_64-dvd.iso`

**Passo 5 — Configuração de rede:**
```
Interface: ens33 (VMware E1000/VMXNET3)
IP: x.x.x.x/21
Gateway: x.x.x.x
DNS: x.x.x.x
Método: manual (static)
```

---

## 5. Instalação do Oracle Linux 8.10

### Sequência de instalação do OL 8.10

**Passo 1 — Boot da mídia:**
- Na tela de boot, selecione: `Install Oracle Linux 8.10`

**Passo 2 — Language Support:**
- Idioma: `English (United States)`

**Passo 3 — Installation Summary (principais configurações):**

| Seção                   | Configuração                                      |
|-------------------------|---------------------------------------------------|
| Software Selection      | Server (minimal ou Server with GUI)               |
| Installation Destination| Discos virtuais criados, LVM manual               |
| Network & Hostname      | IP estático x.x.x.x/21 — hostname: ol8-orcl19-ribas |
| Time & Date             | Americas/São Paulo (ou UTC)                       |
| Root Password           | Definir senha segura do root                      |
| Create User             | Não necessário (oracle criado pelo preinstall)    |

**Passo 4 — Particionamento recomendado (LVM):**
```
/boot/efi    1022 MB  EFI System (sda1)
/boot         988 MB  ext4 (sdb1)
/              62 GB  xfs   (disk1-vg1-root)
swap           15 GB  swap  (disk1-vg1-swap)
/home          60 GB  xfs   (disk1-vg1-home)
/var           60 GB  xfs   (disk1-vg1-var)
/u01          499 GB  xfs   (disk2-vg1-u01)
/u02          500 GB  xfs   (disk3-vg1-u02)
```

**Passo 5 — Após instalação:**
```bash
# Verificar versão
cat /etc/oracle-release
# Oracle Linux Server release 8.10

# Verificar kernel
uname -r
# 5.15.0-320.202.8.3.el8uek.x86_64
```

---

## 6. Preparação do Linux para o Oracle Database

Todos os comandos abaixo foram executados como **root** via SSH.

### 6.1 — Configurar /etc/hosts e hostname

```bash
# Verificar hostname atual
hostname
# ol8-orcl19-ribas.localdomain

# Se necessário, definir hostname permanentemente
hostnamectl set-hostname ol8-orcl19-ribas.localdomain

# Adicionar entrada no /etc/hosts
echo "x.x.x.x   ol8-orcl19-ribas.localdomain   ol8-orcl19-ribas" >> /etc/hosts

# Verificar resolução
ping -c 1 ol8-orcl19-ribas.localdomain
# RESOLUCAO OK
cat /etc/hosts
# 127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
# ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
# x.x.x.x     ol8-orcl19-ribas.localdomain ol8-orcl19-ribas
```

**Por quê:** O Oracle exige que o hostname resolva para um IP válido. O `/etc/hosts` garante
resolução mesmo sem DNS.

### 6.2 — Instalar pacote oracle-database-preinstall-19c

```bash
yum install -y oracle-database-preinstall-19c
```

**O que este pacote faz automaticamente:**
- Cria o usuário `oracle` (uid=54321) com grupos `oinstall`, `dba`, `oper`, `backupdba`,
  `dgdba`, `kmdba`, `racdba`
- Ajusta parâmetros do kernel em `/etc/sysctl.d/` (shmall, shmmax, semaphores, etc.)
- Configura limites de recursos em `/etc/security/limits.d/` (open files, stack, nproc)
- Instala dependências necessárias para o Oracle (63 pacotes nesta instalação)

**Verificação:**
```bash
rpm -q oracle-database-preinstall-19c
# oracle-database-preinstall-19c-1.0-2.el8.x86_64

id oracle
# uid=54321(oracle) gid=54321(oinstall) groups=54321(oinstall),54322(dba),
# 54323(oper),54324(backupdba),54325(dgdba),54326(kmdba),54330(racdba)
```

### 6.3 — Definir senha do usuário oracle

```bash
echo 'oracle:Oracle19c_Ribas' | chpasswd && echo "Senha oracle definida OK"
```

**Nota de segurança:** Em ambiente de produção, use `passwd oracle` de forma interativa.

### 6.4 — Desabilitar SELinux

```bash
# Desabilitar permanentemente (requer reboot para efeito total)
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

# Definir modo permissivo imediatamente (sem precisar de reboot)
setenforce 0

# Verificar
getenforce
# Permissive
grep "^SELINUX=" /etc/selinux/config
# SELINUX=disabled
```

**Por quê:** O SELinux em modo `enforcing` pode bloquear operações legítimas do Oracle. Em
produção, é possível configurar políticas SELinux adequadas, mas isso está fora do escopo desta
instalação de laboratório.

### 6.5 — Desabilitar Firewalld

```bash
systemctl stop firewalld
systemctl disable firewalld

# Verificar
systemctl is-active firewalld   # inactive
systemctl is-enabled firewalld  # disabled
```

**Por quê:** Para ambiente de laboratório, desabilitar simplifica a conectividade. Em produção,
configure regras específicas para as portas Oracle (1521, 5500).

### 6.6 — Verificar configuração de IP estático

```bash
# Verificar método de configuração da interface ens33
nmcli con show ens33 | grep ipv4.method
# ipv4.method: manual  (já configurado como estático)

# Verificar IP
ip addr show ens33 | grep "inet "
# inet x.x.x.x/21 brd 10.11.7.255 scope global noprefixroute ens33
```

**Nota:** Neste ambiente, a interface `ens33` já possuía configuração IP estática (`ipv4.method: manual`)
com IP x.x.x.x/21, gateway x.x.x.x e DNS x.x.x.x. Nenhuma alteração foi necessária.

Se precisar configurar IP estático em uma nova interface:
```bash
nmcli con mod ens33 ipv4.method manual ipv4.addresses x.x.x.x/21 \
  ipv4.gateway x.x.x.x ipv4.dns x.x.x.x
nmcli con up ens33
```

### 6.7 — Criar estrutura de diretórios Oracle

```bash
# Criar diretórios principais
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1
mkdir -p /u01/app/oraInventory
mkdir -p /u02/oradata

# Definir proprietário e permissões
chown -R oracle:oinstall /u01 /u02
chmod -R 775 /u01 /u02

# Verificar
ls -ld /u01/app/oracle/product/19.0.0/dbhome_1
# drwxrwxr-x. 2 oracle oinstall 6 May 16 11:43 /u01/app/oracle/product/19.0.0/dbhome_1
ls -ld /u02/oradata
# drwxrwxr-x. 2 oracle oinstall 6 May 16 11:43 /u02/oradata
```

**Estrutura de diretórios Oracle:**

| Diretório                                   | Finalidade                                              |
|---------------------------------------------|---------------------------------------------------------|
| `/u01/app/oracle`                           | ORACLE_BASE — base de todos os produtos Oracle          |
| `/u01/app/oracle/product/19.0.0/dbhome_1`   | ORACLE_HOME — binários e bibliotecas do Oracle 19c      |
| `/u01/app/oraInventory`                     | Inventário central do Oracle (criado pelo instalador)   |
| `/u01/app/oracle/admin/orcl`                | Arquivos administrativos do banco                       |
| `/u01/app/oracle/diag`                      | Automatic Diagnostic Repository (ADR)                   |
| `/u02/oradata`                              | Datafiles, redo logs e control files do banco           |
| `/u02/oradata/ORCL`                         | Datafiles do CDB (ORCL)                                 |
| `/u02/oradata/ORCL/orclpdb`                 | Datafiles do PDB (ORCLPDB)                              |

---

## 7. Transferência e Descompactação do Instalador

### 7.1 — Verificar instalador disponível

```bash
# O arquivo já estava disponível em /tmp
ls -lh /tmp/V982063-01.zip
# -rw-r--r--. 1 root root 2.9G May 16 09:53 /tmp/V982063-01.zip
```

Se precisar transferir o arquivo do Windows:
```powershell
# Na máquina LOCAL (Windows), copiar via SCP para /tmp do servidor
scp C:\Downloads\V982063-01.zip root@x.x.x.x:/tmp/V982063-01.zip
```

### 7.2 — Descompactar no ORACLE_HOME

```bash
# Descompactar dentro do ORACLE_HOME (iniciando em 11:44:14)
cd /u01/app/oracle/product/19.0.0/dbhome_1
unzip -q /tmp/V982063-01.zip
# Concluído em 11:45:34 (80 segundos)

# Corrigir proprietário após descompactação
chown -R oracle:oinstall /u01/app/oracle/product/19.0.0/dbhome_1

# Verificar tamanho
du -sh /u01/app/oracle/product/19.0.0/dbhome_1/
# 6.5G    /u01/app/oracle/product/19.0.0/dbhome_1/
```

**Tamanho após descompactação:** ~6.5 GB no ORACLE_HOME.

---

## 8. Instalação Silenciosa do Oracle Database 19c

### 8.1 — Arquivo de resposta (db_install.rsp)

```bash
# Criar arquivo de resposta em /tmp/db_install.rsp
cat > /tmp/db_install.rsp << 'RSPEOF'
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v19.0.0
oracle.install.option=INSTALL_DB_SWONLY
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/u01/app/oraInventory
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
ORACLE_BASE=/u01/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.OSDBA_GROUP=dba
oracle.install.db.OSOPER_GROUP=oper
oracle.install.db.OSBACKUPDBA_GROUP=backupdba
oracle.install.db.OSDGDBA_GROUP=dgdba
oracle.install.db.OSKMDBA_GROUP=kmdba
oracle.install.db.OSRACDBA_GROUP=racdba
oracle.install.db.rootconfig.configMethod=ROOT
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
DECLINE_SECURITY_UPDATES=true
RSPEOF
chown oracle:oinstall /tmp/db_install.rsp
```

**Explicação dos parâmetros principais:**

| Parâmetro                           | Valor                  | Explicação                                             |
|-------------------------------------|------------------------|--------------------------------------------------------|
| `oracle.install.option`             | INSTALL_DB_SWONLY      | Instala apenas o software, sem criar banco ainda       |
| `oracle.install.db.InstallEdition`  | EE                     | Enterprise Edition                                     |
| `INVENTORY_LOCATION`                | /u01/app/oraInventory  | Diretório do inventário central Oracle                 |
| `oracle.install.db.OSDBA_GROUP`     | dba                    | Grupo OS com privilégio SYSDBA                         |
| `oracle.install.db.rootconfig.configMethod` | ROOT          | Scripts root.sh serão executados manualmente           |
| `DECLINE_SECURITY_UPDATES`          | true                   | Não configurar atualizações automáticas de segurança   |

### 8.2 — Executar instalação silenciosa como oracle

```bash
# Criar script de instalação
cat > /tmp/run_install.sh << 'SCRIPT'
#!/bin/bash
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export CV_ASSUME_DISTID=OL8
export PATH=$ORACLE_HOME/bin:$PATH
echo "Executando runInstaller como oracle em $(date)"
$ORACLE_HOME/runInstaller -silent -responseFile /tmp/db_install.rsp \
  -ignorePrereqFailure -waitforcompletion
echo "EXIT_CODE=$?"
SCRIPT
chmod 755 /tmp/run_install.sh
chown oracle:oinstall /tmp/run_install.sh

# Executar como oracle (iniciando em 11:46:54)
su - oracle -c "/tmp/run_install.sh"
# Concluído em 11:49:02 (2 minutos e 8 segundos)
```

**Variável CV_ASSUME_DISTID explicada:**

| Variável          | Valor | Motivo                                                    |
|-------------------|-------|-----------------------------------------------------------|
| `CV_ASSUME_DISTID`| OL8   | Informa ao instalador que o SO é Oracle Linux 8, evitando |
|                   |       | o erro [INS-13001] "not supported on this OS"             |

**Flags do runInstaller:**

| Flag                  | Descrição                                               |
|-----------------------|---------------------------------------------------------|
| `-silent`             | Execução sem interface gráfica                          |
| `-responseFile`       | Arquivo com parâmetros de instalação                    |
| `-ignorePrereqFailure`| Ignorar falhas em pré-requisitos opcionais              |
| `-waitforcompletion`  | Aguardar conclusão antes de retornar                    |

**Resultado obtido:**
```
Successfully Setup Software with warning(s).
```

### 8.3 — Executar scripts root pós-instalação

```bash
# Script 1: Configura permissões do inventário Oracle
/u01/app/oraInventory/orainstRoot.sh
# Changing permissions of /u01/app/oraInventory.
# Adding read,write permissions for group.
# Removing read,write,execute permissions for world.
# Changing groupname of /u01/app/oraInventory to oinstall.
# The execution of the script is complete.

# Script 2: Configura links simbólicos, cria /etc/oratab e TFA
/u01/app/oracle/product/19.0.0/dbhome_1/root.sh
```

**O root.sh cria:**
- Links em `/usr/local/bin/`: `dbhome`, `oraenv`, `coraenv`
- Entrada inicial no `/etc/oratab`

---

## 9. Criação do Banco de Dados via DBCA

### 9.1 — Script de criação (run_dbca.sh)

```bash
cat > /tmp/run_dbca.sh << 'DBCAEOF'
#!/bin/bash
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=orcl
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib

dbca -silent -createDatabase \
  -templateName General_Purpose.dbc \
  -gdbname orcl.localdomain \
  -sid orcl \
  -responseFile NO_VALUE \
  -characterSet AL32UTF8 \
  -sysPassword Oracle19c_Ribas \
  -systemPassword Oracle19c_Ribas \
  -createAsContainerDatabase true \
  -numberOfPDBs 1 \
  -pdbName orclpdb \
  -pdbAdminPassword Oracle19c_Ribas \
  -databaseType MULTIPURPOSE \
  -totalMemory 16384 \
  -memoryMgmtType AUTO_SGA \
  -storageType FS \
  -datafileDestination /u02/oradata \
  -redoLogFileSize 50 \
  -emConfiguration DBEXPRESS \
  -emExpressPort 5500 \
  -ignorePreReqs
DBCAEOF
chmod 755 /tmp/run_dbca.sh
chown oracle:oinstall /tmp/run_dbca.sh

# Executar como oracle (iniciando em 11:51:15)
su - oracle -c "/tmp/run_dbca.sh"
# Concluído em ~12:17 (aproximadamente 26 minutos)
```

### 9.2 — Parâmetros do DBCA explicados

| Parâmetro                    | Valor              | Explicação                                                  |
|------------------------------|--------------------|-------------------------------------------------------------|
| `-gdbname`                   | orcl.localdomain   | Nome global do banco (db_name.db_domain)                    |
| `-sid`                       | orcl               | System Identifier da instância                              |
| `-characterSet`              | AL32UTF8           | Character set Unicode, suporta todos os idiomas             |
| `-createAsContainerDatabase` | true               | Cria CDB (Container Database) para suportar PDBs            |
| `-numberOfPDBs`              | 1                  | Um PDB (Pluggable Database) a ser criado                    |
| `-pdbName`                   | orclpdb            | Nome do Pluggable Database                                  |
| `-totalMemory`               | 16384              | 16 GB alocados para Oracle (SGA + PGA), ~26% de 62 GB RAM  |
| `-memoryMgmtType`            | AUTO_SGA           | ASMM — gerenciamento automático de SGA (para RAM > 4 GB)    |
| `-storageType`               | FS                 | Sistema de arquivos (não ASM)                               |
| `-datafileDestination`       | /u02/oradata       | Diretório para arquivos de dados                            |
| `-redoLogFileSize`           | 50                 | Tamanho dos redo log files em MB                            |
| `-emConfiguration`           | DBEXPRESS          | Habilita Oracle EM Express (acesso via browser)             |
| `-emExpressPort`             | 5500               | Porta HTTPS do EM Express                                   |

> **Importante sobre `-memoryMgmtType AUTO_SGA`:** Com RAM total > 4 GB, o Oracle não suporta
> AMM (Automatic Memory Management com MEMORY_TARGET). Deve-se usar ASMM (Automatic Shared
> Memory Management com SGA_TARGET), que é o valor `AUTO_SGA`.

### 9.3 — Progresso do DBCA (output real)

```
Prepare for db operation           8% complete
Copying database files            31% complete
Creating and starting Oracle instance
                                  32%, 36%, 40%, 43%, 46% complete
Completing Database Creation      51%, 54% complete
Creating Pluggable Databases      58%, 77% complete
Executing Post Configuration Actions
                                 100% complete
Database creation complete.
Global Database Name: orcl.localdomain
System Identifier(SID): orcl
```

**Duração total:** ~26 minutos (11:51 a ~12:17)

> **Nota técnica:** O DBCA executou as seguintes fases nesta instalação:
> 1. Cópia dos datafiles do template (31%)
> 2. Inicialização da instância Oracle (32-46%)
> 3. Execução do catálogo do CDB (catalog.sql, catproc.sql) com UTLRP — fase mais longa (~20 min)
> 4. Aplicação do Release Update (190410122720) no PDB$SEED
> 5. Reinicialização da instância e criação do ORCLPDB por clone do PDB$SEED
> 6. UTLRP no ORCLPDB + pós-scripts
>
> A frequente mensagem "Checkpoint not complete" no alert log indica que os redo logs (50 MB) são
> pequenos para o volume de redo gerado pelos scripts do catálogo. Em produção, recomenda-se usar
> redo logs de 200 MB ou mais.

---

## 10. Configuração Pós-Instalação

### 10.1 — Variáveis de ambiente do usuário oracle

Arquivo: `/home/oracle/.bash_profile`

```bash
# .bash_profile - Oracle Database 19c Environment
if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=orcl
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export TNS_ADMIN=$ORACLE_HOME/network/admin
export NLS_LANG=AMERICAN_AMERICA.AL32UTF8
umask 022
```

Arquivo: `/home/oracle/.bashrc`

```bash
# .bashrc - Oracle Environment
if [ -f /etc/bashrc ]; then
    . /etc/bashrc
fi
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=orcl
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export TNS_ADMIN=$ORACLE_HOME/network/admin
export NLS_LANG=AMERICAN_AMERICA.AL32UTF8
```

### 10.2 — /etc/oratab — habilitar autostart

```bash
# Alterar N para Y na entrada do banco
sed -i 's|orcl:/u01/app/oracle/product/19.0.0/dbhome_1:N|orcl:/u01/app/oracle/product/19.0.0/dbhome_1:Y|' /etc/oratab

# Verificar
grep "^orcl" /etc/oratab
# orcl:/u01/app/oracle/product/19.0.0/dbhome_1:Y
```

**Formato do /etc/oratab:**  
`<SID>:<ORACLE_HOME>:<Y|N>`  
- `Y` = banco iniciado automaticamente pelo `dbstart`  
- `N` = banco NÃO iniciado automaticamente

### 10.3 — Listener Oracle (listener.ora e tnsnames.ora)

Arquivo: `$ORACLE_HOME/network/admin/listener.ora`

```
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = ol8-orcl19-ribas.localdomain)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = orcl.localdomain)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = orcl)
    )
  )
```

Arquivo: `$ORACLE_HOME/network/admin/tnsnames.ora`

```
ORCL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = ol8-orcl19-ribas.localdomain)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl.localdomain)
    )
  )

ORCLPDB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = ol8-orcl19-ribas.localdomain)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orclpdb)
    )
  )
```

```bash
# Iniciar o listener (como oracle)
lsnrctl start

# Registrar serviços no listener
sqlplus "/ as sysdba" <<< "ALTER SYSTEM REGISTER;"

# Verificar status
lsnrctl status
```

### 10.4 — Configurar ORCLPDB para abrir automaticamente

```bash
# Conectar como SYSDBA e salvar estado de abertura do PDB
sqlplus -s / as sysdba << SQLEOF
ALTER PLUGGABLE DATABASE orclpdb OPEN;
ALTER PLUGGABLE DATABASE orclpdb SAVE STATE;
SQLEOF
```

**Por quê:** Após reinicialização do CDB, por padrão os PDBs ficam em MOUNT. O comando
`SAVE STATE` garante que o ORCLPDB seja aberto automaticamente quando o CDB reiniciar.

### 10.5 — Serviços systemd Oracle

Arquivo: `/etc/systemd/system/oracle-db.service`

```ini
[Unit]
Description=Oracle Database 19c (orcl)
After=network.target

[Service]
Type=forking
User=oracle
Group=oinstall
Environment=ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
Environment=ORACLE_SID=orcl
Environment=ORACLE_BASE=/u01/app/oracle
ExecStart=/u01/app/oracle/product/19.0.0/dbhome_1/bin/dbstart /u01/app/oracle/product/19.0.0/dbhome_1
ExecStop=/u01/app/oracle/product/19.0.0/dbhome_1/bin/dbshut /u01/app/oracle/product/19.0.0/dbhome_1
TimeoutStartSec=600
TimeoutStopSec=600

[Install]
WantedBy=multi-user.target
```

Arquivo: `/etc/systemd/system/oracle-listener.service`

```ini
[Unit]
Description=Oracle Listener (LISTENER)
After=network.target oracle-db.service
Requires=oracle-db.service

[Service]
Type=forking
User=oracle
Group=oinstall
Environment=ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
Environment=ORACLE_SID=orcl
ExecStart=/u01/app/oracle/product/19.0.0/dbhome_1/bin/lsnrctl start
ExecStop=/u01/app/oracle/product/19.0.0/dbhome_1/bin/lsnrctl stop
TimeoutStartSec=300
TimeoutStopSec=300

[Install]
WantedBy=multi-user.target
```

```bash
# Habilitar inicialização automática
systemctl daemon-reload
systemctl enable oracle-db
systemctl enable oracle-listener

# Verificar
systemctl is-enabled oracle-db      # enabled
systemctl is-enabled oracle-listener # enabled
```

**Para gerenciar os serviços:**
```bash
systemctl start oracle-db       # Iniciar banco
systemctl stop oracle-db        # Parar banco
systemctl status oracle-db      # Status do banco
systemctl start oracle-listener # Iniciar listener
systemctl status oracle-listener # Status do listener
```

---

## 11. Validação da Instalação

### 11.1 — Conectar via SQL*Plus

```bash
# Como usuário oracle (com .bash_profile carregado)
su - oracle
sqlplus / as sysdba
```

### 11.2 — Consultar v$instance

```sql
SELECT instance_name, status, version, host_name, startup_time FROM v$instance;
```

**Saída obtida:**
```
INSTANCE_NAME    STATUS       VERSION           HOST_NAME
---------------- ------------ ----------------- --------------------------------
STARTUP_TIME
------------------
orcl             OPEN         19.0.0.0.0        ol8-orcl19-ribas.localdomain
16-MAY-26
```

### 11.3 — Consultar v$database

```sql
SELECT name, open_mode, cdb, db_unique_name FROM v$database;
```

**Saída obtida:**
```
NAME      OPEN_MODE            CDB DB_UNIQUE_NAME
--------- -------------------- --- ------------------------------
ORCL      READ WRITE           YES orcl
```

### 11.4 — Consultar v$pdbs (Pluggable Databases)

```sql
SELECT name, open_mode, restricted, con_id FROM v$pdbs ORDER BY con_id;
```

**Saída obtida:**
```
NAME             OPEN_MODE  RES     CON_ID
---------------- ---------- --- ----------
PDB$SEED         READ ONLY  NO           2
ORCLPDB          READ WRITE NO           3
```

### 11.5 — Verificar versão completa

```sql
SELECT banner FROM v$version WHERE rownum=1;
```

**Saída obtida:**
```
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
```

### 11.6 — Parâmetros do banco

```sql
SELECT name, value FROM v$parameter 
WHERE name IN ('db_name','sga_target','pga_aggregate_target','db_block_size','processes','sessions')
ORDER BY name;
```

**Saída obtida:**
```
NAME                            VALUE
------------------------------- ----------------------------------------
db_block_size                   8192
db_name                         orcl
pga_aggregate_target            4294967296    (4 GB)
processes                       640
sessions                        984
sga_target                      12884901888   (12 GB)
```

### 11.7 — Character set e Tablespaces padrão

```sql
SELECT property_name, property_value FROM database_properties 
WHERE property_name IN ('NLS_CHARACTERSET','NLS_NCHAR_CHARACTERSET',
                        'DEFAULT_TEMP_TABLESPACE','DEFAULT_PERMANENT_TABLESPACE');
```

**Saída obtida:**
```
PROPERTY_NAME                   PROPERTY_VALUE
------------------------------- -------------------------
NLS_CHARACTERSET                AL32UTF8
NLS_NCHAR_CHARACTERSET          AL16UTF16
DEFAULT_PERMANENT_TABLESPACE    USERS
DEFAULT_TEMP_TABLESPACE         TEMP
```

### 11.8 — Status do listener

```bash
lsnrctl status
```

**Saída obtida:**
```
Services Summary...
Service "orcl.localdomain" has 2 instance(s).
  Instance "orcl", status UNKNOWN, has 1 handler(s)...
  Instance "orcl", status READY, has 1 handler(s)...
Service "orclpdb.localdomain" has 1 instance(s).
  Instance "orcl", status READY, has 1 handler(s)...
Service "orclXDB.localdomain" has 1 instance(s).
  Instance "orcl", status READY, has 1 handler(s)...
The command completed successfully
```

### 11.9 — Datafiles criados

```sql
SELECT file#, name, bytes/1024/1024 MB FROM v$datafile ORDER BY file#;
```

**Saída obtida:**
```
FILE# NAME                                              MB
----- ------------------------------------------------- ----
    1 /u02/oradata/ORCL/system01.dbf                    890
    3 /u02/oradata/ORCL/sysaux01.dbf                    520
    4 /u02/oradata/ORCL/undotbs01.dbf                   325
    5 /u02/oradata/ORCL/pdbseed/system01.dbf            270
    6 /u02/oradata/ORCL/pdbseed/sysaux01.dbf            330
    7 /u02/oradata/ORCL/users01.dbf                       5
    8 /u02/oradata/ORCL/pdbseed/undotbs01.dbf           100
    9 /u02/oradata/ORCL/orclpdb/system01.dbf            270
   10 /u02/oradata/ORCL/orclpdb/sysaux01.dbf            330
   11 /u02/oradata/ORCL/orclpdb/undotbs01.dbf           100
   12 /u02/oradata/ORCL/orclpdb/users01.dbf               5
```

### 11.10 — Processos Oracle em execução

```bash
ps -ef | grep ora_ | grep -v grep | wc -l
# 69 processos Oracle rodando
```

---

## 12. Comparativo Antes × Depois da Instalação

### Uso de Disco

| Ponto de Montagem | Antes       | Depois       | Delta          |
|-------------------|-------------|--------------|----------------|
| / (root)          | 6.3G (11%)  | 6.4G (11%)   | +0.1 GB        |
| /u01 (Oracle Home)| 0           | 11G (3%)     | +11 GB         |
| /u02 (Dados)      | 0           | 6.8G (2%)    | +6.8 GB        |
| /home             | 461 MB      | 461 MB       | sem alteração  |
| /var              | 1.3G        | 1.3G         | sem alteração  |

### Uso de Memória

| Recurso     | Antes      | Depois      | Observação                           |
|-------------|------------|-------------|--------------------------------------|
| RAM usada   | 397 MB     | 998 MB      | +600 MB (processos background)       |
| SGA Oracle  | 0          | 12 GB       | Configurado no DBCA (totalMemory 16G)|
| PGA Oracle  | 0          | 4 GB        | Agregado para todos os processos     |
| Swap usado  | 0 MB       | 0 MB        | Nenhum swap utilizado                |

### Processos do Sistema

| Métrica             | Antes | Depois   |
|---------------------|-------|----------|
| Processos Oracle    | 0     | 69       |
| Porta 1521 (TNS)    | Fechada | Aberta |
| Porta 5500 (EM Exp) | Fechada | Aberta |

### Estado do OS

| Componente      | Antes      | Depois                                |
|-----------------|------------|---------------------------------------|
| SELinux         | Enforcing  | Disabled (Permissive imediato)        |
| Firewalld       | Enabled    | Disabled                              |
| Oracle Services | Nenhum     | oracle-db e oracle-listener (enabled) |
| /etc/oratab     | Vazio      | orcl:...:Y                            |

### Sumário de Hardware (sem alteração após instalação)

| Recurso    | Valor                              |
|------------|------------------------------------|
| CPU        | AMD Opteron 6344 — 8 vCPUs         |
| RAM Total  | 62 GB                              |
| Swap       | 15 GB                              |
| Kernel     | 5.15.0-320.202.8.3.el8uek.x86_64   |
| Hypervisor | VMware (full virtualization)       |

---

## 13. Resumo da Instalação

| Item                     | Detalhe                                              |
|--------------------------|------------------------------------------------------|
| Produto                  | Oracle Database 19c Enterprise Edition               |
| Versão completa          | 19.0.0.0.0 (Release 19.3.0.0.0)                      |
| Nome da instância        | orcl                                                 |
| Status                   | OPEN (READ WRITE)                                    |
| Tipo                     | CDB (Container Database)                             |
| PDB                      | ORCLPDB (READ WRITE, CON_ID=3)                       |
| Character Set            | AL32UTF8 (Unicode)                                   |
| Hostname                 | ol8-orcl19-ribas.localdomain (x.x.x.x)               |
| Interface de rede        | ens33 (IP estático manual)                           |
| Listener                 | Porta 1521 (TCP) — READY                             |
| EM Express URL           | https://x.x.x.x:5500/em                              |
| Serviços systemd         | oracle-db e oracle-listener (enabled)                |
| Tempo: descompactação    | ~80 segundos (ZIP 2.9GB → 6.5GB)                     |
| Tempo: instalação SW     | ~2 minutos (runInstaller)                            |
| Tempo: criação do banco  | ~26 minutos (DBCA)                                   |
| Tempo total              | ~35 minutos                                          |

**URL do Oracle EM Express:** `https://x.x.x.x:5500/em`  
(Aceite o certificado auto-assinado no browser; faça login com `SYS/Oracle19c_Ribas AS SYSDBA`)

---

## 14. Troubleshooting

### 14.1 — Erro: CV_ASSUME_DISTID na instalação

**Sintoma:**
```
[FATAL] [INS-13001] Oracle Database is not supported on this operating system
```

**Causa:** O instalador não reconhece o Oracle Linux 8 como sistema suportado sem a variável de
identificação.

**Solução:**
```bash
export CV_ASSUME_DISTID=OL8
$ORACLE_HOME/runInstaller -silent ...
```

### 14.2 — Erro: Automatic Memory Management not allowed

**Sintoma:**
```
[FATAL] [DBT-11211] The Automatic Memory Management option is not allowed 
when the total physical memory is greater than 4GB.
```

**Causa:** Com RAM > 4 GB, Oracle AMM (MEMORY_TARGET) não é suportado.

**Solução:** Usar ASMM (Automatic Shared Memory Management):
```bash
-memoryMgmtType AUTO_SGA
```

### 14.3 — Erro: ORA-00845: MEMORY_TARGET not supported

**Causa:** MEMORY_TARGET requer que /dev/shm tenha espaço suficiente.

**Solução:**
```bash
df -h /dev/shm  # verificar tamanho

# Se necessário, aumentar:
mount -o remount,size=16G /dev/shm
```

### 14.4 — Interface de rede diferente do esperado

**Sintoma:** A interface `ens160` especificada não existe no servidor.

**Causa:** O VMware pode criar a interface com nomes diferentes (ens33, ens160, eth0, etc.),
dependendo do tipo de adaptador de rede configurado na VM.

**Diagnóstico:**
```bash
ip addr  # listar todas as interfaces
nmcli con show  # listar conexões NetworkManager
```

**Solução:** Usar o nome real da interface encontrado com os comandos acima.

### 14.5 — Listener sem serviços registrados / status UNKNOWN

**Sintoma:**
```
Service "orcl.localdomain" has 1 instance(s).
  Instance "orcl", status UNKNOWN, ...
```

**Causa:** O banco não registrou automaticamente no listener ainda. O status UNKNOWN ocorre quando
a instância foi configurada estaticamente no listener.ora mas ainda não se registrou dinamicamente.

**Solução:**
```sql
ALTER SYSTEM REGISTER;
```
Aguarde 30-60 segundos e verifique com `lsnrctl status`. O status mudará para READY.

### 14.6 — PDB ORCLPDB em MOUNT após reinicialização

**Causa:** Por padrão, PDBs não abrem automaticamente após restart do CDB, a menos que `SAVE STATE`
tenha sido executado.

**Solução:**
```sql
ALTER PLUGGABLE DATABASE orclpdb OPEN;
ALTER PLUGGABLE DATABASE orclpdb SAVE STATE;
```

### 14.7 — NLS_DATE_FORMAT com erro "not a valid identifier"

**Sintoma:**
```
bash: export: `HH24:MI:SS': not a valid identifier
```

**Causa:** O valor `"DD-MON-YYYY HH24:MI:SS"` contém espaço e precisa estar entre aspas no
arquivo `.bash_profile`. Se o arquivo for escrito via heredoc por ferramentas que interpretam
aspas, as aspas podem ser removidas.

**Solução:**
```bash
# Verificar o .bash_profile
cat /home/oracle/.bash_profile | grep NLS_DATE

# Remover a linha problemática
sed -i '/NLS_DATE_FORMAT/d' /home/oracle/.bash_profile

# Adicionar corretamente (o NLS_LANG já inclui formato padrão)
# Ou adicionar com aspas simples:
echo "export NLS_DATE_FORMAT='DD-MON-YYYY HH24:MI:SS'" >> /home/oracle/.bash_profile
```

### 14.8 — Erro de su com múltiplas linhas (plink)

**Sintoma:**
```
su: option requires an argument -- 'c'
```

**Causa:** Ao executar comandos multi-linha via `su -c "..."` com plink (PuTTY command line), o
argumento `-c` não recebe a string corretamente.

**Solução:** Usar um script intermediário:
```bash
cat > /tmp/meu_script.sh << 'EOF'
#!/bin/bash
export ORACLE_HOME=...
# comandos aqui
EOF
chmod 755 /tmp/meu_script.sh
chown oracle:oinstall /tmp/meu_script.sh
su - oracle -c "/tmp/meu_script.sh"
```

### 14.9 — DBCA lento / Checkpoint not complete

**Sintoma:** O DBCA demora mais de 30 minutos e o alert log mostra frequentemente:
```
Thread 1 cannot allocate new log, sequence N
Checkpoint not complete
```

**Causa:** Os redo log files de 50 MB são muito pequenos para o volume de redo gerado pelos scripts
do catálogo Oracle durante o DBCA. Isso causa espera de checkpoint.

**Solução (após a criação do banco):**
```sql
-- Aumentar redo log files para 200 MB
ALTER DATABASE ADD LOGFILE '/u02/oradata/ORCL/redo04.log' SIZE 200M;
-- (adicionar pelo menos 3 grupos novos)
-- Depois fazer switch e dropar os grupos pequenos
ALTER SYSTEM SWITCH LOGFILE;
ALTER DATABASE DROP LOGFILE GROUP 1;
```

**Prevenção (próxima instalação):** Use `-redoLogFileSize 200` no DBCA.

### 14.10 — Listener não inicia / TNS-12545

**Sintoma:**
```
TNS-12545: Connect failed because target host or object does not exist
```

**Causa:** Hostname no `listener.ora` não pode ser resolvido.

**Solução:**
```bash
# Verificar /etc/hosts
grep ol8-orcl19-ribas /etc/hosts

# Verificar resolução
ping -c 1 ol8-orcl19-ribas.localdomain

# Se necessário, usar IP no listener.ora
sed -i 's/HOST = ol8-orcl19-ribas.localdomain/HOST = x.x.x.x/' \
    $ORACLE_HOME/network/admin/listener.ora
lsnrctl start
```

### 14.11 — Espaço insuficiente para DBCA

**Sintoma:**
```
[FATAL] Insufficient disk space
```

**Solução:**
```bash
df -h /u02/oradata  # verificar espaço disponível

# Se o volume for LVM, expandir:
lvextend -L +50G /dev/disk3-vg1/u02
xfs_growfs /u02
```

---

## 15. Referências e Links Úteis

### Documentação Oficial Oracle

| Recurso                                | URL                                                                  |
|----------------------------------------|----------------------------------------------------------------------|
| Oracle DB 19c Install Guide Linux      | https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/ |
| Oracle DB 19c Concepts                 | https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/ |
| Oracle DB 19c Administrator's Guide    | https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/ |
| Oracle DB 19c SQL Language Reference   | https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/ |
| Oracle DB 19c Performance Tuning Guide | https://docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/ |
| Oracle Support (MOS)                   | https://support.oracle.com                                           |
| Multitenant Administrator's Guide      | https://docs.oracle.com/en/database/oracle/oracle-database/19/multi/ |

### Downloads

| Recurso                    | URL                                  |
|----------------------------|--------------------------------------|
| Oracle Technology Network  | https://www.oracle.com/technetwork   |
| Oracle eDelivery           | https://edelivery.oracle.com         |
| Oracle Linux               | https://yum.oracle.com               |
| Oracle Database 19c Patch  | https://support.oracle.com → Patches |

### Comandos de Referência Rápida

```bash
# =====================
# COMO USUÁRIO ORACLE
# =====================

# Status do banco
sqlplus -s "/ as sysdba" <<< "SELECT instance_name, status FROM v\$instance;"

# Iniciar/parar banco manualmente
dbstart $ORACLE_HOME
dbshut $ORACLE_HOME

# Iniciar/parar listener
lsnrctl start
lsnrctl stop
lsnrctl status

# Verificar alert log
tail -f $ORACLE_BASE/diag/rdbms/orcl/orcl/trace/alert_orcl.log

# Conectar ao CDB
sqlplus / as sysdba

# Conectar ao PDB
sqlplus sys/Oracle19c_Ribas@orclpdb as sysdba

# Verificar processos Oracle
ps -ef | grep ora_ | grep -v grep

# Oracle EM Express
# https://x.x.x.x:5500/em

# =====================
# COMO ROOT (systemd)
# =====================

# Iniciar banco via systemd
systemctl start oracle-db

# Parar banco via systemd
systemctl stop oracle-db

# Status dos serviços
systemctl status oracle-db
systemctl status oracle-listener

# Ver logs do serviço
journalctl -u oracle-db -n 50

# =====================
# CONSULTAS UTEIS
# =====================

# Verificar PDBs
SELECT name, open_mode, con_id FROM v$pdbs ORDER BY con_id;

# Abrir PDB manualmente
ALTER PLUGGABLE DATABASE orclpdb OPEN;

# Verificar parâmetros SGA/PGA
SHOW PARAMETER sga_target;
SHOW PARAMETER pga_aggregate_target;

# Verificar datafiles
SELECT file#, name, bytes/1024/1024 MB, status FROM v$datafile ORDER BY file#;

# Verificar redo logs
SELECT group#, sequence#, bytes/1024/1024 MB, status FROM v$log;

# Verificar tablespaces
SELECT tablespace_name, status, contents FROM dba_tablespaces;
```

---

*Documento gerado automaticamente pelo Claude Code em 16/05/2026.*  
*Instalação executada e validada com sucesso no ambiente descrito.*  
*Servidor: ol8-orcl19-ribas.localdomain (x.x.x.x)*
