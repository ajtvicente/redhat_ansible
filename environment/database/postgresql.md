# PostgreSQL

![postgresql](/redhat_ansible/images/postgresql.png)

## Sumário

- [Sobre o PostgreSQL](#sobre-o-postgresql)
- [Requisitos](#requisitos)
- [Arquitetura](#arquitetura)
- [Preparando Sistema Operacional AWS-EC2](#preparando-sistema-operacional-aws-ec2)
- [Preparando Sistema Operacional On-Premises](#preparando-sistema-operacional-on-premises)
- [Instalando o PostgreSQL](#instalando-o-postgresql)
- [Tuning](#tuning)
- [Referências](#referências)

## Sobre o PostgreSQL

O *PostgreSQL* é um sistema de gerenciamento de banco de dados relacional (SGBDR) de código aberto, robusto e altamente escalável. O *Red Hat Enterprise Linux* (RHEL) oferece uma plataforma estável e segura para a implantação de instâncias PostgreSQL em ambientes corporativos e produtivos.

O *PostgreSQL* oferece:

- **Conformidade ACID**: Garante a integridade de dados com transações ACID completas
- **Extensibilidade**: Suporte a procedimentos armazenados, funçÕes customizadas e tipos de dados personalizados
- **Escalabilidade**: Capacidade de gerenciar grandes volumes de dados com performance otimizada
- **Segurança**: Autenticação robusta, controle de acesso granular e criptografado
- **Código aberto**: Comunidade ativa e desenvolvimento contínuo

## Requisitos

Abaixo estão relacionados os Requisitos de Hardware, Requisitos de Rede, Requisitos de Software, Compatibilidade e Ciclo de Vida.

- **Requisitos de Hardware**

Tipo                 | Mínimo (Dev/Stage) | Recomendado (Prod) | Performance (Enterprise)
---------------------|--------------------|--------------------|-------------------------
Processamento - CPU  | 2-4 Cores          | 8-16 Core          | 16+ Cores - threads
Memória - RAM        | 8-16 GB            | 32-64 GB           | 64-256 GB - cache
Armazenamento - DISK | HDD                | SSD - SATA/SAS     | NVMe - low-latency
Arquitetura - ARQ    | x86_64 / ARM64     | x86_64 / ARM64     | x86_64 / ARM64

- **Requisitos de Rede**

Tipo                     | Desenvolvimento (Dev) | Homologação (Stage) | Produção (Prod)
-------------------------|-----------------------|---------------------|----------------
Porta - port             | 5432 (postgres)       | 5432 (postgres)     | 5432 (postgres)
Latência - latency       | 50 ms                 | 10 ms               | 5 ms
Segurança - ssl/tls      | Opcional              | Recomendado         | Mandatório
Replicação - replication | Opcional              | Recomendado         | Mandatório

- **Requisitos de Software**

Sistema Operacional         | Versão Padrão (Default) | Versões Suportadas
----------------------------|-------------------------|--------------------------
Red Hat Enterprise Linux 10 | PostgreSQL 16           | PostgreSQL 16-18
Red Hat Enterprise Linux 9  | PostgreSQL 15           | PostgreSQL 13-15-16-18
Red Hat Enterprise Linux 8  | PostgreSQL 13           | PostgreSQL 10-12-13-15-16 

- **Compatibilidade**

Versão Red Hat Ansible | Arquitetura RPM | Arquitetura Containerized | Arquitetura Operator                             | Database
-----------------------|-----------------|---------------------------|--------------------------------------------------|-----------------------
AAP-2.7                | Não Suportado   | RHEL-9, RHEL-10           | Red Hat OpenShift Container Platform 4.14 - 4.22 | PostgreSQL 15, 16, 17
AAP-2.6                | RHEL-9          | RHEL-9, RHEL-10           | Red Hat OpenShift Container Platform 4.14 - 4.20 | PostgreSQL 15, 16, 17
AAP-2.5                | RHEL-9, RHEL-8  | RHEL-9, RHEL-10           | Red Hat OpenShift Container Platform 4.12 - 4.20 | PostgreSQL 15
AAP-2.4                | RHEL-9, RHEL-8  | RHEL-9 (Tech-Preview)     | Red Hat OpenShift Container Platform 4.10 - 4.20 | PostgreSQL 15 (≥ 2.4-13), 13 (≤ 2.4-12)

- **Ciclo de Vida PostgreSQL**

Versão        | Release Inicial | Release Final
--------------|-----------------|--------------
PostgreSQL 18 | 25/09/2025      | 14/11/2030
PostgreSQL 17 | 26/09/2024      | 08/11/2029
PostgreSQL 16 | 14/09/2023      | 09/11/2028
PostgreSQL 15 | 13/10/2022      | 11/11/2027
PostgreSQL 14 | 30/09/2021      | 12/11/2026
PostgreSQL 13 | 24/09/2020      | 13/11/2025

## Arquitetura


1. **Standalone**: Um único servidor PostgreSQL funcionando independentemente, sem replicação ou failover automático.

```Code
┌─────────────────────────────┐
│   PostgreSQL Server         │
│   (Single Instance)         │
├─────────────────────────────┤
│  • Database Files           │
│  • WAL (Write-Ahead Log)    │
│  • Backups (Local)          │
└──────────────┬──────────────┘
               │
        ┌──────┴───────┐
        ↓              ↓
   ┌────────┐    ┌──────────┐
   │ Storage│    │ Backup   │
   │ Local  │    │ Storage  │
   └────────┘    └──────────┘
```

2. **Replication**: Um servidor primário (master), com um ou mais servidores secundários (replicas) que recebem atualizações em tempo real via WAL (Write-Ahead Log) streaming.

```Code
┌──────────────────────────────────┐
│      PRIMARY (Master)            │
│   Produz transações (WAL)        │
│   Aceita Reads e Writes          │
└──────────────┬───────────────────┘
               │
     WAL Streaming Replication
     (Síncrona ou Assíncrona)
     │         │         │
     ↓         ↓         ↓
┌─────────┐ ┌─────────┐ ┌─────────┐
│Replica 1│ │Replica 2│ │Replica 3│
│Standby  │ │Standby  │ │Standby  │
│(Read)   │ │(Read)   │ │(Read)   │
└─────────┘ └─────────┘ └─────────┘
```

3. **Failover**: Replicação Streaming + automação de failover, usando ferramentas como Patroni, pg_auto_failover ou pgPool-II, que oferecem recursos de failover.

```Code
┌──────────────────────────────────────────┐
│     Patroni HA Cluster                   │
├──────────────────────────────────────────┤
│                                          │
│  ┌──────────────┐  ┌──────────────┐      │
│  │  Primary     │  │  Standby 1   │      │
│  │  PostgreSQL  │  │  PostgreSQL  │      │
│  │  (Active)    │  │  (Ready)     │      │
│  └──────┬───────┘  └──────┬───────┘      │
│         │                 │              │
│  ┌──────┴─────────────────┴───────┐      │
│  │  Consensus Layer (etcd)        │      │
│  │  • Detecta falhas              │      │
│  │  • Elege novo primary          │      │
│  │  • Promove standby             │      │
│  └────────────────────────────────┘      │
│                                          │
│  ┌──────────────┐                        │
│  │  Standby 2   │                        │
│  │  PostgreSQL  │                        │
│  │  (Watch)     │                        │
│  └──────────────┘                        │
└──────────────────────────────────────────┘
```

4. **Cloud**: Usar serviço gerenciado de um Cloud-Provider (AWS-RDS, Google Cloud SQL, Azure Database), o provedor gerencia a infraestrutura, backups, patches e alta disponibilidade (High-Availability).

```Code
┌─────────────────────────────────┐
│    AWS Management Console       │
│  • Backups automáticos          │
│  • Replicação Multi-AZ          │
│  • Patches automáticos          │
│  • Monitoring CloudWatch        │
│  • Auto-scaling (RDS Proxy)     │
└────────┬─────────────────────────┘
         │
    ┌────┴───────────┐
    ↓                ↓
┌────────────┐  ┌────────────┐
│Primary     │  │Standby     │
│(AZ-1)      │  │(AZ-2)      │
└────────────┘  └────────────┘
     ↓              ↓
   ┌──────────────────┐
   │ Automatic Backup │
   │ (S3)             │
   └──────────────────┘
```

5. **Container**: PostgreSQL em container Docker/Podman, orquestrado por Docker Compose (desenvolvimento) ou Kubernetes/OpenShift (produção).

```Code
            ┌─────────────────────┐
            │   3 POD CLUSTER     │
            ├─────────────────────┤
            │ Pod 0: PRIMARY      │
            │ Pod 1: STANDBY      │
            │ Pod 2: STANDBY      │
            └─────────────────────┘
                       ↓
       ┌───────────────┼──────────────┐
       │               │              │
    ┌──┴──┐         ┌──┴──┐        ┌──┴──┐
    │ RW  │         │ RO  │        │ RO  │
    │Write│         │Read │        │Read │
    └─────┘         └─────┘        └─────┘
       ↓               ↓              ↓
    App1.1        Analytics       Reports
   (Primary)      Services        Services
```

## Preparando Sistema Operacional AWS-EC2

Remova as configurações de subscrição da AWS para EC2:

```bash
# Comandos executados via ROOT
subscription-manager unregister
subscription-manager remove --all
subscription-manager clean
dnf clean all
rm -rf /var/cache/yum/*

# Comandos executados via SUDO
sudo subscription-manager unregister
sudo subscription-manager remove --all
sudo subscription-manager clean
sudo dnf clean all
sudo rm -rf /var/cache/yum/*
```

Edite o arquivo *rhsm.conf* permitindo a gestão de repositórios pela Red Hat:

```bash
# Comandos executados via ROOT
vi /etc/rhsm/rhsm.conf

# Comandos executados via SUDO
sudo vi /etc/rhsm/rhsm.conf

# Altere
manage_repos = 1
auto_registration_interval = 60
```

Remova todos os arquivos de subscrição da AWS:

```bash
# Comandos executados via ROOT
cd /etc/yum.repos.d/
rm -rf redhat-rhui*

# Comandos executados via SUDO
cd /etc/yum.repos.d/
sudo rm -rf redhat-rhui*
```

Registre seu Sistema Operacional no Red Hat Content Delivery Network (CDN):

```bash
# Comandos executados via ROOT
subscription-manager register
subscription-manager attach --auto

# Comandos executados via SUDO
sudo subscription-manager register
sudo subscription-manager attach --auto
```

Atualize seu Sistema Operacional:

```bash
# Comandos executados via ROOT
dnf update -y

# Comandos executados via SUDO
sudo dnf update -y
```

Instale os pacotes básicos em seu Sistema Operacional:

```bash
# Comandos executados via ROOT
dnf install -y git tree sysstat vim curl wget net-tools bind-utils nmap git-core rsync bash-completion ansible-core insights-client

# Comandos executados via SUDO
sudo dnf install -y git tree sysstat vim curl wget net-tools bind-utils nmap git-core rsync bash-completion ansible-core insights-client
```

Registre seu Sistema Operacional no Red Hat Lightspeed (Insights):

```bash
# Comandos executados via ROOT
rhc connect

# Comandos executados dia SUDO
sudo rhc connect
```

Verifique se a porta padrão do PostgreSQL está liberada no AWS-VPC:

```bash
# Comandos executados via ROOT
nmap -sT -p 5432 localhost

# Comandos executados via SUDO
sudo nmap -sT -p 5432 localhost
```

Reinicie seu Sistema Operacional:

```bash
# Comandos executado via ROOT e SUDO
reboot
```

## Preparando Sistema Operacional On-Premises

Registre seu Sistema Operacional no Red Hat Content Delivery Network (CDN):

```bash
# Comandos executados via ROOT
subscription-manager register
subscription-manager attach --auto

# Comandos executados via SUDO
sudo subscription-manager register
sudo subscription-manager attach --auto
```

Atualize seu Sistema Operacional:

```bash
# Comandos executados via ROOT
dnf update -y

# Comandos executados via SUDO
sudo dnf update -y
```

Instale os pacotes básicos em seu Sistema Operacional:

```bash
# Comandos executados via ROOT
dnf install -y git tree sysstat vim curl wget net-tools bind-utils nmap git-core rsync bash-completion ansible-core insights-client firewalld

# Comandos executados via SUDO
sudo dnf install -y git tree sysstat vim curl wget net-tools bind-utils nmap git-core rsync bash-completion ansible-core insights-client firewalld
```

Registre seu Sistema Operacional no Red Hat Lightspeed (Insights):

```bash
# Comandos executados via ROOT
rhc connect

# Comandos executados dia SUDO
sudo rhc connect
```

Habilite, inicie e verifique o status do serviço do Firewall:

```bash
# Comandos executados via ROOT
systemctl enable --now firewalld
systemctl start firewalld
systemctl status firewalld

# Comandos executados via SUDO
sudo systemctl enable --now firewalld
sudo systemctl start firewalld
sudo systemctl status firewalld
```

Configure e aplique as regras de Firewall voltadas para o PostgreSQL:

```bash
# Comandos executados via ROOT
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<IP>" port port="5432" protocol="tcp" accept'
firewall-cmd --permanent --add-service=postgresql
firewall-cmd --permanent --add-port=5432/tcp
firewall-cmd --reload
firewall-cmd --list-all

# Comandos executados dia SUDO
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<IP>" port port="5432" protocol="tcp" accept'
sudo firewall-cmd --permanent --add-service=postgresql
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

Verifique se a porta padrão do PostgreSQL está liberada:

```bash
# Comandos executados via ROOT
nmap -sT -p 5432 localhost

# Comandos executados via SUDO
sudo nmap -sT -p 5432 localhost
```

Reinicie seu Sistema Operacional:

```bash
# Comandos executado via ROOT e SUDO
reboot
```

## Instalando o PostgreSQL

Instale o PostgreSQL no Red Hat Enterprise Linux 8:

```bash
# Comandos executados via ROOT
yum module install -y postgresql:13/server
yum module install -y postgresql:15/server
yum module install -y postgresql:16/server

# Comandos executados via SUDO
sudo yum module install -y postgresql:13/server
sudo yum module install -y postgresql:15/server
sudo yum module install -y postgresql:16/server
```

Instale o PostgreSQL no Red Hat Enterprise Linux 9:

```bash
# Comandos executados via ROOT
dnf module install -y postgresql:13/server
dnf module install -y postgresql:15/server
dnf module install -y postgresql:16/server

# Comandos executados via SUDO
sudo dnf module install -y postgresql:13/server
sudo dnf module install -y postgresql:15/server
sudo dnf module install -y postgresql:16/server
```

Instale o PostgreSQL no Red Hat Enterprise Linux 10:

```bash
# Comandos executados via ROOT
dnf install -y postgresql16-server

# Comandos executados via SUDO
sudo dnf install -y postgresql16-server
```

Verifique a versão instalado do PostgreSQL:

```bash
# Comandos executado via ROOT e SUDO
postgres -V
```

Inicie a database e habilite, inicie e verifique o status do serviço do PostgreSQL:

```bash
# Comandos executados via ROOT
postgresql-setup --initdb
systemctl start postgresql.service
systemctl enable postgresql.service
systemctl status postgresql.service

# Comandos executados via SUDO
sudo postgresql-setup --initdb
sudo systemctl start postgresql.service
sudo systemctl enable postgresql.service
sudo systemctl status postgresql.service
```

Atribua uma senha para o usuário *postgres* (Sistema Operacional):

```bash
# Comandos executados via ROOT
passwd postgres

# Comandos executados via SUDO
sudo passwd postgres
```

Acesse o Cliente-SQL:

```bash
# Comandos executados via ROOT e SUDO
sudo -u postgres psql
```

Atribua uma senha para o usuário *postgres* (Database):

```sql
ALTER USER postgres WITH PASSWORD 'redhat.123';

-- Saia do Client-SQL
\q
```

Edite o arquivo de configurações do PostgreSQL:

```bash
# Comandos executados via ROOT
vim /var/lib/pgsql/data/postgresql.conf

# Comandos executados via SUDO
sudo vim /var/lib/pgsql/data/postgresql.conf

# Altere
listen_addresses = '<IP-or-FQDN>'
port = 5432
```

Edite o arquivo de conexões do PostgreSQL (IPV4):

```bash
# Comandos executados via ROOT
vim /var/lib/pgsql/data/pg_hba.conf

# Comandos executados via SUDO
sudo vim /var/lib/pgsql/data/pg_hba.conf

# Altere
host    all             all             127.0.0.1/32            md5
host    all             all             <ip-pg-primary>/32      md5
host    all             all             <ip-pg-replica>/32      md5
host    all             all             <ip-gateway>/32         md5
host    all             all             <ip-hub>/32             md5
host    all             all             <ip-eda>/32             md5
host    all             all             <ip-controller>/32      md5
host    all             all             <ip-metrics>/32         md5
host    all             all             <ip-lightspeed>/32      md5
host    all             all             <ip-portal>/32          md5
```

Reinicie e verifique o status do serviço do PostgreSQL:

```bash
# Comandos executados via ROOT
systemctl restart postgresql.service
systemctl status postgresql.service

# Comandos executados via SUDO
sudo systemctl restart postgresql.service
sudo systemctl status postgresql.service
```

Acesse o Cliente-SQL:

```bash
# Comandos executados via ROOT e SUDO
sudo -u postgres psql
```

Crie as databases para o Red Hat Ansible Automation Platform 2.x:

```sql
-- Red Hat Ansible Automation Gateway
CREATE DATABASE automation_gateway;
CREATE USER automation_gateway WITH PASSWORD 'redhat.123';
GRANT ALL PRIVILEGES ON DATABASE automation_gateway TO automation_gateway;
ALTER DATABASE automation_gateway OWNER TO automation_gateway;

-- Red Hat Ansible Automation Controller
CREATE DATABASE automation_controller;
CREATE USER automation_controller WITH PASSWORD 'redhat.123';
GRANT ALL PRIVILEGES ON DATABASE automation_controller TO automation_controller;
ALTER DATABASE automation_controller OWNER TO automation_controller;

-- Red Hat Ansible Automation Private Hub
CREATE DATABASE automation_hub;
CREATE USER automation_hub WITH PASSWORD 'redhat.123';
GRANT ALL PRIVILEGES ON DATABASE automation_hub TO automation_hub;
ALTER DATABASE automation_hub OWNER TO automation_hub;

-- Red Hat Ansible Automation Event Driven
CREATE DATABASE automation_eda;
CREATE USER automation_eda WITH PASSWORD 'redhat.123';
GRANT ALL PRIVILEGES ON DATABASE automation_eda TO automation_eda;
ALTER DATABASE automation_eda OWNER TO automation_eda;

-- Red Hat Ansible Automation Metrics Service
CREATE DATABASE automation_metrics;
CREATE USER automation_metrics WITH PASSWORD 'redhat.123';
GRANT ALL PRIVILEGES ON DATABASE automation_metrics TO automation_metrics;
ALTER DATABASE automation_metrics OWNER TO automation_metrics;

-- Red Hat Ansible Automation Lightspeed
CREATE DATABASE automation_lightspeed;
CREATE USER automation_lightspeed WITH PASSWORD 'redhat.123';
GRANT ALL PRIVILEGES ON DATABASE automation_lightspeed TO automation_lightspeed;
ALTER DATABASE automation_lightspeed OWNER TO automation_lightspeed;

-- Red Hat Ansible Automation Portal
CREATE DATABASE automation_portal;
CREATE USER automation_portal WITH PASSWORD 'redhat.123';
GRANT ALL PRIVILEGES ON DATABASE automation_portal TO automation_portal;
ALTER DATABASE automation_portal OWNER TO automation_portal;
```

Verifique se as databases foram criadas:

```sql
\l
                                                                           List of databases
         Name          |         Owner         | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |                Access privileges                
-----------------------+-----------------------+----------+-------------+-------------+------------+-----------------+-------------------------------------------------
 automation_controller | automation_controller | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =Tc/automation_controller                      +
                       |                       |          |             |             |            |                 | automation_controller=CTc/automation_controller
 automation_eda        | automation_eda        | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =Tc/automation_eda                             +
                       |                       |          |             |             |            |                 | automation_eda=CTc/automation_eda
 automation_gateway    | automation_gateway    | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =Tc/automation_gateway                         +
                       |                       |          |             |             |            |                 | automation_gateway=CTc/automation_gateway
 automation_hub        | automation_hub        | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =Tc/automation_hub                             +
                       |                       |          |             |             |            |                 | automation_hub=CTc/automation_hub
 automation_lightspeed | automation_lightspeed | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =Tc/automation_lightspeed                      +
                       |                       |          |             |             |            |                 | automation_lightspeed=CTc/automation_lightspeed
 automation_metrics    | automation_metrics    | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =Tc/automation_metrics                         +
                       |                       |          |             |             |            |                 | automation_metrics=CTc/automation_metrics
 automation_portal     | automation_portal     | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =Tc/automation_portal                          +
                       |                       |          |             |             |            |                 | automation_portal=CTc/automation_portal
 postgres              | postgres              | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | 
 template0             | postgres              | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres                                    +
                       |                       |          |             |             |            |                 | postgres=CTc/postgres
 template1             | postgres              | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres                                    +
                       |                       |          |             |             |            |                 | postgres=CTc/postgres
(10 rows)

-- Saia do Client-SQL
\q
```

Instale o pacote adicional ao PostgreSQL:

```bash
# Comandos executados via ROOT
dnf install -y postgresql-contrib

# Comandos executados via SUDO
sudo dnf install -y postgresql-contrib
```

Acesse o Cliente-SQL:

```bash
# Comandos executados via ROOT e SUDO
sudo -u postgres psql
```

Habilite a extensão *hstore* na database *automation_hub*:

```sql
-- Acesse a database automation_hub
\c automation_hub

-- habilite a extensão hstore
CREATE EXTENSION hstore;

-- verifique se a extensão foi habilitada
\dx
                           List of installed extensions
  Name   | Version |   Schema   |                   Description                    
---------+---------+------------+--------------------------------------------------
 hstore  | 1.8     | public     | data type for storing sets of (key, value) pairs
 plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(2 rows)

-- Saia do Client-SQL
\q
```

Caso seja necessário recriar uma databese, use o comando DROP para excluir a database existente (cuidado: este processo elimina todos os dados):

```sql
-- Exclusão da database
DROP DATABASE nome_do_banco;

-- Forçar a exclusão da database
DROP DATABASE nome_do_banco WITH (FORCE);
```

## Tuning

Segue abaixo o tuning voltado para melhoria de desempenho do PostgreSQL em um servidor Red Hat Enterprise Linux:

1. **Sistema Operacional**

Configurações de kernel recomendadas:

```bash
# Comandos executados via ROOT
vim /etc/sysctl.conf

# Comandos executados via SUDO
sudo vim /etc/sysctl.conf

# Adicione o contaúdo
vm.swappiness = 1 
vm.dirty_expire_centisecs = 500 
vm.dirty_writeback_centisecs = 100
vm.dirty_background_ratio = 5
vm.dirty_ratio = 10
fs.aio-max-nr = 1048576
vm.nr_hugepages = 2304 # Otimiza o comportamento de HugePages (16GB- 2304 | 32 4608 | 64 8704)
net.core.rmem_max = 2097152
net.core.wmem_max = 2097152
net.core.netdev_budget = 600
net.core.netdev_budget_usecs = 4000
net.core.somaxconn = 65535
net.ipv4.tcp_rmem = 4096 87380 2097152
net.ipv4.tcp_wmem = 4096 16384 2097152
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_window_scaling = 1
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_buckets = 262144
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
```

Configurações de limits recomendadas:

```bash
# Comandos executados via ROOT
vim /etc/security/limits.conf

# Comandos executados via SUDO
sudo vim /etc/security/limits.conf

# Adicione o contaúdo
postgres    soft    nofile     1048576
postgres    hard    nofile     1048576
postgres    soft    nproc      16384
postgres    hard    nproc      16384
postgres    soft    memlock    unlimited
postgres    hard    memlock    unlimited
```

Para desabilitar IPV6 em redes que não utilizam o protocolo:

```bash
echo "net.ipv6.conf.all.disable_ipv6=1" >> /etc/sysctl.d/ipv6.conf
echo "net.ipv6.conf.lo.disable_ipv6=1" >> /etc/sysctl.d/ipv6.conf
echo "net.ipv6.conf.<interface>.disable_ipv6=1" >> /etc/sysctl.d/ipv6.conf

nmcli connection show

nmcli connection modify lo ipv6.method "ignore"
nmcli connection modify lo ipv6.method "disabled"
nmcli con down lo && nmcli con up lo

nmcli connection modify eth0 ipv6.method "ignore"
nmcli connection modify eth0 ipv6.method "disabled"
nmcli con down eth0 && nmcli con up eth0
```

Reinicie seu Sistema Operacional:

```bash
# Comandos executado via ROOT e SUDO
reboot
```

2. **PostgreSQL**

Edite o arquivo de configurações do PostgreSQL:

```bash
# Comandos executados via ROOT
vim /var/lib/pgsql/data/postgresql.conf

# Comandos executados via SUDO
sudo vim /var/lib/pgsql/data/postgresql.conf
```

Valores para os parâmetros do PostgreSQL abaixo, devem ser gerados no [pgTune](https://pgtune.leopard.in.ua/), conforme o sizing do seu ambiente:

```text
max_connections = <pgtune>
shared_buffers = <pgtune>
effective_cache_size = <pgtune>
maintenance_work_mem = <pgtune>
checkpoint_completion_target = <pgtune>
wal_buffers = <pgtune>
default_statistics_target = <pgtune>
random_page_cost = <pgtune>
effective_io_concurrency = <pgtune>
work_mem = <pgtune>
huge_pages = <pgtune>
jit = <pgtune>
wal_compression = <pgtune>
min_wal_size = <pgtune>
max_wal_size = <pgtune>
max_worker_processes = <pgtune>
max_parallel_workers_per_gather = <pgtune>
max_parallel_workers = <pgtune>
max_parallel_maintenance_workers = <pgtune>
```

Recomendação de valores para os parâmetros do PostgreSQL:

```text
password_encryption = 'scram-sha-256' # md5 or scram-sha-256
dynamic_shared_memory_type = posix    # the default is the first option
wal_level = logical                   # minimal, replica, or logical
wal_log_hints = on                    # also do full page writes of non-critical updates
max_wal_senders = 10                  # max number of walsender processes
hot_standby = on                      # "off" disallows queries during recovery
log_destination = 'stderr'            # Valid values are combinations of
logging_collector = on                # Enable capturing of stderr and csvlog
log_directory = 'log'                 # directory where log files are written,
log_filename = 'postgresql-%a.log'    # log file name pattern,
log_truncate_on_rotation = on         # If on, an existing log file with the
log_rotation_age = 1d                 # Automatic rotation of logfiles will
log_rotation_size = 0                 # Automatic rotation of logfiles will
log_line_prefix = '< %m %a >'         # special values:
log_timezone = 'America/Sao_Paulo'
datestyle = 'iso, mdy'
timezone = 'America/Sao_Paulo'
lc_messages = 'en_US.UTF-8'           # locale for system error message
lc_monetary = 'en_US.UTF-8'           # locale for monetary formatting
lc_numeric = 'en_US.UTF-8'            # locale for number formatting
lc_time = 'en_US.UTF-8'               # locale for time formatting
default_text_search_config = 'pg_catalog.english'
```

Reinicie e verifique o status do serviço do PostgreSQL:

```bash
# Comandos executados via ROOT
systemctl restart postgresql.service
systemctl status postgresql.service

# Comandos executados via SUDO
sudo systemctl restart postgresql.service
sudo systemctl status postgresql.service
```

## Referências

- [PostgreSQL Versioning Policy](https://www.postgresql.org/support/versioning/)
- [Top Support Policies for Red Hat Ansible Automation](https://access.redhat.com/articles/ansible-top-support-policies)
- [Red Hat Ansible Automation Platform Life Cycle](https://access.redhat.com/support/policy/updates/ansible-automation-platform)
- [Red Hat Ansible Automation Execution Environment Life Cycle](https://access.redhat.com/support/policy/updates/ansible-automation-platform-execution-environments)
- [Requirements Hardware PostgreSQL-13](https://www.postgresql.org/docs/13/install-requirements.html)
- [Requirements Hardware PostgreSQL-14](https://www.postgresql.org/docs/14/install-requirements.html)
- [Requirements Hardware PostgreSQL-15](https://www.postgresql.org/docs/15/install-requirements.html)
- [Requirements Hardware PostgreSQL-16](https://www.postgresql.org/docs/16/install-requirements.html)
- [Requirements Hardware PostgreSQL-17](https://www.postgresql.org/docs/17/install-requirements.html)
- [Installing and configuring a PostgreSQL database server by using RHEL-8 system roles](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/deploying_different_types_of_servers/using-databases#installing-postgresql_using-postgresql)
- [Installing and configuring a PostgreSQL database server by using RHEL-9 system roles](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_and_using_database_servers/using-postgresql_configuring-and-using-database-servers#installing-postgresql_using-postgresql)
- [Installing and configuring a PostgreSQL database server by using RHEL-10 system roles](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_and_using_database_servers/using-postgresql#installing-postgresql)
- [pgTune](https://pgtune.leopard.in.ua/)
- [PostgreSQL](https://www.postgresql.org/)
- [What is PostgreSQL?](https://www.postgresql.org/about/)
- [Architectural Fundamentals](https://www.postgresql.org/docs/current/tutorial-arch.html)
- [Requirements](https://www.postgresql.org/docs/current/install-requirements.html)
- [Managing Kernel Resources](https://www.postgresql.org/docs/current/kernel-resources.html)
- [How do I setup a logon banner for my ssh clients to see when they login?](https://access.redhat.com/solutions/3816)
- [Create ASCII text banners online](https://manytools.org/hacker-tools/ascii-banner/)
- [How to register and subscribe a RHEL system to the Red Hat Customer Portal using Red Hat Subscription-Manager?](https://access.redhat.com/solutions/253273)
- [What is the maximum limit for "net.core.somaxconn" parameter?](https://access.redhat.com/solutions/6439141)
- [Getting error: Invalid argument after setting "net.core.somaxconn" value greater than 65535 in RHEL 7?](https://access.redhat.com/solutions/6443181)
- [kernel: Possible SYN flooding on port #. Sending cookies.](https://access.redhat.com/solutions/30453)
- [What is the purpose of /proc/sys/vm/drop_caches in Red Hat Enterprise Linux?](https://access.redhat.com/solutions/5409)
- [Should the cache be dropped on a GFS2 file-system when a performance issue is occurring?](https://access.redhat.com/solutions/393963)
- [How do I configure TCP maximum slot table entries to 128 for RHEL 7/8 NFS clients?](https://access.redhat.com/solutions/6535871)
- [On RHEL servers, changing the semaphore value fails with a message "setting key "kernel.sem": Numerical result out of range".](https://access.redhat.com/solutions/4968021)
- [9.2. Automatic NUMA Balancing](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_tuning_and_optimization_guide/sect-virtualization_tuning_optimization_guide-numa-auto_numa_balancing)
- [How to set ulimit values](https://access.redhat.com/solutions/61334)
- [How to set or change the default soft or hard limit for the number of user's processes?](https://access.redhat.com/solutions/406663)
- [How to increase number of open files limit in Red Hat Enterprise Linux ?](https://access.redhat.com/solutions/60746)
- [What is the maximum value and default value for fs.nr_open in Red Hat Enterprise Linux?](https://access.redhat.com/solutions/1479623)
- [How to increase the "nr_open" value to increase the "nofile" limits ?](https://access.redhat.com/solutions/3485471)
- [Slow SSH, SCP or SFTP transfer performance](https://access.redhat.com/solutions/393343)