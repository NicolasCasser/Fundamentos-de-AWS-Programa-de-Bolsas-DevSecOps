# Projeto: WordPress em Alta Disponibilidade na AWS

Este repositório documenta a implementação de uma arquitetura de alta disponibilidade para a plataforma WordPress na nuvem AWS. O objetivo é implantar o WordPress de forma escalável e tolerante a falhas, utilizando os principais serviços gerenciados da AWS para garantir desempenho e disponibilidade, simulando um ambiente de produção real.

## Arquitetura Proposta

A arquitetura final distribui a aplicação em múltiplas instâncias EC2, gerenciadas por um Auto Scaling Group e com o tráfego balanceado por um Application Load Balancer. O armazenamento de arquivos é centralizado no Amazon EFS, e os dados são gerenciados por um banco de dados relacional Amazon RDS.

![Diagrama da Arquitetatura](imagens/Diagrama-WordPress.png)

---

## Guia de Implementação Passo a Passo

Esta seção descreve em detalhes o processo de construção do ambiente.

### Fase 1: Configuração da Fundação de Rede (VPC)

A rede foi configurada com uma VPC personalizada contendo sub-redes públicas e privadas em duas Zonas de Disponibilidade para garantir a resiliência.

1.  **VPC:** Foi criada uma VPC com o nome `wordpress-vpc` e CIDR `10.0.0.0/16`.
2.  **Sub-redes:**
    * Foram criadas duas sub-redes públicas (`10.0.1.0/24`, `10.0.2.0/24`) para recursos com acesso à internet.
    * Foram criadas duas sub-redes privadas (`10.0.3.0/24`, `10.0.4.0/24`) para recursos protegidos.
3.  **Gateways e Rotas:**
    * Um **Internet Gateway** (`wordpress-igw`) foi criado e atrelado a uma tabela de rotas (`wordpress-public-rt`) para as sub-redes públicas.
    * Um **NAT Gateway** (`wordpress-nat-gateway`) foi posicionado em uma sub-rede pública, com sua rota configurada na tabela de rotas privada (`wordpress-private-rt`).

### Fase 2: Configuração da Segurança (Security Groups)

Foram criados grupos de segurança específicos para controlar o tráfego entre os componentes:

1.  **`wordpress-db-sg` (RDS):** Permite tráfego na porta `3306` apenas a partir do grupo `wordpress-ec2-sg`.
2.  **`wordpress-efs-sg` (EFS):** Permite tráfego na porta `2049` (NFS) apenas a partir do grupo `wordpress-ec2-sg`.
3.  **`wordpress-alb-sg` (Load Balancer):** Permite tráfego público na porta `80` (HTTP) de qualquer lugar (`0.0.0.0/0`).
4.  **`wordpress-ec2-sg` (Instâncias EC2):** Permite tráfego na porta `80` apenas a partir do grupo `wordpress-alb-sg`, e na porta `22` (SSH) para acesso de manutenção.

### Fase 3: Provisionamento dos Serviços de Dados (RDS & EFS)

1.  **Amazon EFS:**
    * Foi criado um sistema de arquivos com o nome `wordpress-efs` na `wordpress-vpc`.
    * Foram criados **Mount Targets** em cada uma das **sub-redes privadas**, ambos associados ao Security Group `wordpress-efs-sg`.
    * Um ajuste crítico na `wordpress-vpc` foi a habilitação das opções **"Enable DNS resolution"** e **"Enable DNS hostnames"**.

2.  **Amazon RDS:**
    * Foi criado um **Grupo de Sub-redes de Banco de Dados** (`wordpress-db-subnet-group`) para agrupar as duas **sub-redes privadas**.
    * A **instância do banco de dados** (`wordpress-db`) foi criada com o mecanismo `MySQL` (classe `db.t3.micro`), configurada no `wordpress-db-subnet-group` e protegida pelo Security Group `wordpress-db-sg`, sem acesso público.
    * O **"Nome do banco de dados inicial"** foi definido como `wordpress`.

### Fase 4: Implementação da Alta Disponibilidade (EC2, ALB, ASG)

1.  **Launch Template (Modelo de Execução):**
    * Foi criado um modelo com o nome `wordpress-lt`, servindo como "molde" para todas as instâncias da aplicação.
    * **AMI:** Foi utilizada a **Amazon Linux 2023**.
    * **Tipo de instância:** `t2.micro`.
    * **Grupo de segurança:** `wordpress-ec2-sg`.
    * **Dados do usuário (User Data):** O modelo foi configurado com o script `user-data` final, que prepara a instância e clona o repositório Git com a configuração da aplicação.

2.  **Application Load Balancer (ALB):**
    * Foi criado um **Grupo de Destino** (`wordpress-targets`) para `Instâncias` na `wordpress-vpc` (protocolo `HTTP:80`).
    * Foi provisionado um **Application Load Balancer** (`wordpress-alb`), `Voltado para a Internet`, mapeado para as **sub-redes públicas** e usando o Security Group `wordpress-alb-sg`. A regra principal do seu *listener* na porta 80 foi configurada para encaminhar todo o tráfego para o grupo de destino `wordpress-targets`.

3.  **Auto Scaling Group (ASG):**
    * Foi criado um grupo com o nome `wordpress-asg`, associado ao `wordpress-lt`.
    * O ASG foi configurado para lançar instâncias nas duas **sub-redes PRIVADAS**, garantindo a segurança.
    * O grupo foi anexado ao `wordpress-targets` do ALB, com as verificações de integridade do ELB habilitadas.
    * Foi definida uma **"Política de dimensionamento com monitoramento do objetivo"** para manter a `Utilização média da CPU` em `50%`, com a capacidade do grupo variando entre 1 e 4 instâncias.

---
### Script `user-data` Final (Método com Git)

Este é o script final utilizado no Launch Template. Ele prepara a infraestrutura da instância e busca a configuração da aplicação (`docker-compose.yml`) do repositório Git.

```bash
#!/bin/bash
# --- CONFIGURAÇÃO DE VARIÁVEIS ---
EFS_ID="seu-id-do-efs"
RDS_ENDPOINT="seu-endpoint-do-rds"
DB_NAME="wordpress"
DB_USER="admin"
DB_PASSWORD="SUA-SENHA-DO-RDS"
GIT_REPO_URL="[https://github.com/seu-usuario/seu-repositorio.git](https://github.com/seu-usuario/seu-repositorio.git)"

# --- PREPARAÇÃO DA INSTÂNCIA ---
yum update -y
yum install -y docker amazon-efs-utils git

systemctl start docker
systemctl enable docker
usermod -a -G docker ec2-user

# --- MONTAGEM DO EFS ---
EFS_MOUNT_POINT="/mnt/efs-wordpress"
mkdir -p ${EFS_MOUNT_POINT}
echo "${EFS_ID}:/ ${EFS_MOUNT_POINT} efs _netdev,tls 0 0" >> /etc/fstab
mount -a -t efs
chmod 777 ${EFS_MOUNT_POINT}

# --- DEPLOY DA APLICAÇÃO VIA GIT E DOCKER COMPOSE ---
cd /home/ec2-user
git clone ${GIT_REPO_URL}
REPO_DIR=$(basename ${GIT_REPO_URL} .git)
cd ${REPO_DIR}

# Exporta as variáveis de ambiente para o Docker Compose
export RDS_ENDPOINT
export DB_USER
export DB_PASSWORD
export DB_NAME
export EFS_MOUNT_POINT

# Instala o Docker Compose e executa a aplicação
curl -L "[https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname](https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname) -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
/usr/local/bin/docker-compose up -d
```
---

## Análise e Descobertas do Processo

O processo de implementação revelou desafios técnicos importantes, principalmente relacionados às limitações de recursos da instância `t2.micro`.

1.  **Desafio de Recursos:** A combinação do sistema operacional, Docker e a aplicação WordPress (com servidor Apache) consome uma quantidade de memória e CPU que leva a instância `t2.micro` ao seu limite.
2.  **Conflito de Permissões com EFS:** Foi identificado um complexo problema de permissões entre o contêiner Docker e o volume de rede EFS (NFS), que impedia o WordPress de salvar arquivos de mídia. A solução robusta foi aplicar permissões abertas (`chmod 777`) no ponto de montagem do EFS.
3.  **Falha de Configuração do `user-data`:** Os erros de "502 Bad Gateway" na arquitetura final foram rastreados a uma falha na interpretação do script `user-data`. A solução foi garantir a integridade do script e armazená-lo de forma consistente em um Launch Template.

## Status do Projeto

* **CONCLUÍDO E FUNCIONAL:** A arquitetura de alta disponibilidade (VPC, ALB, ASG, RDS, EFS) foi totalmente implantada e validada.
* **VALIDADO:** O site (front-end) é servido corretamente através do Load Balancer e as instâncias são resilientes a falhas.
* **DESAFIO CONHECIDO:** O painel de administração (`/wp-admin`) apresenta instabilidade devido às limitações de recursos da instância `t2.micro`.
