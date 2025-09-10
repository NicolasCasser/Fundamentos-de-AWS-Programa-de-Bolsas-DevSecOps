# Projeto: WordPress em Alta Disponibilidade na AWS

Este repositório documenta a implementação de uma arquitetura de alta disponibilidade para a plataforma WordPress na nuvem AWS. O objetivo é implantar o WordPress de forma escalável e tolerante a falhas, utilizando os principais serviços gerenciados da AWS para garantir desempenho e disponibilidade, simulando um ambiente de produção real.

## Arquitetura Proposta

A arquitetura final distribui a aplicação em múltiplas instâncias EC2, gerenciadas por um Auto Scaling Group e com o tráfego balanceado por um Application Load Balancer. O armazenamento de arquivos é centralizado no Amazon EFS, e os dados são gerenciados por um banco de dados relacional Amazon RDS.

![Diagrama da Arquitetura](imagens/Diagrama-WordPress.png)

---

## Etapas de Configuração

Esta seção descreve em detalhes o processo de construção do ambiente.

### Fase 1: Configuração da Rede (VPC)

A rede foi configurada com uma VPC personalizada contendo sub-redes públicas e privadas em duas Zonas de Disponibilidade para garantir a resiliência.

1.  **VPC:** Foi criada uma VPC com o nome `wordpress-vpc` e CIDR `10.0.0.0/16`.
2.  **Sub-redes:**
    * Foram criadas duas sub-redes públicas (`10.0.1.0/24`, `10.0.2.0/24`) para recursos com acesso à internet.
    * Foram criadas duas sub-redes privadas (`10.0.3.0/24`, `10.0.4.0/24`) para recursos protegidos.
3.  **Gateways e Rotas:**
    * Um **Internet Gateway** (`wordpress-igw`) foi criado e atrelado a uma tabela de rotas (`wordpress-public-rt`) para as sub-redes públicas.
    * Um **NAT Gateway** (`wordpress-ngw`) foi posicionado em uma sub-rede pública, com sua rota configurada na tabela de rotas privada (`wordpress-private-rt`).

### Fase 2: Configuração da Segurança (Security Groups)

Foram criados grupos de segurança específicos para controlar o tráfego entre os componentes:

1.  **`wordpress-db-sg` (RDS):** Permite tráfego na porta `3306` apenas a partir do grupo `wordpress-ec2-sg`.
2.  **`wordpress-efs-sg` (EFS):** Permite tráfego na porta `2049` (NFS) apenas a partir do grupo `wordpress-ec2-sg`.
3.  **`wordpress-alb-sg` (Load Balancer):** Permite tráfego público na porta `80` (HTTP) de qualquer lugar (`0.0.0.0/0`).
4.  **`wordpress-ec2-sg` (Instâncias EC2):** Permite tráfego na porta `80` apenas a partir do grupo `wordpress-alb-sg`, e na porta `22` (SSH) para acesso de manutenção.

### Fase 3: Provisionamento dos Serviços de Dados (RDS & EFS)

1.  **Amazon EFS:**
    * Foi criado um sistema de arquivos com o nome `wordpress-efs` na `wordpress-vpc`.
    * Para garantir a alta disponibilidade, foram criados **Mount Targets** em cada uma das **sub-redes privadas**, ambos associados ao Security Group `wordpress-efs-sg`.
    * Um ajuste crítico realizado na `wordpress-vpc` foi a habilitação das opções **"Enable DNS resolution"** e **"Enable DNS hostnames"**, essenciais para a resolução de nomes do EFS.

2.  **Amazon RDS:**
    * Primeiramente, foi criado um **Grupo de Sub-redes de Banco de Dados** (`wordpress-db-subnet-group`), que agrupa as duas **sub-redes privadas**, isolando o banco de dados da internet.
    * Em seguida, a **instância do banco de dados** foi criada com as seguintes especificações:
        * **Mecanismo:** `MySQL`, utilizando o modelo de `Nível gratuito`.
        * **Identificador:** `wordpress-db`, com o usuário mestre `admin`.
        * **Classe da instância:** `db.t3.micro`, conforme os requisitos do projeto.
        * **Conectividade:** A instância foi configurada na `wordpress-vpc`, utilizando o `wordpress-db-subnet-group` e o Security Group `wordpress-db-sg`, com o **Acesso público** definido como `Não`.
        * **Configuração adicional:** O **"Nome do banco de dados inicial"** foi definido como `wordpress` para simplificar a configuração da aplicação.

### Fase 4: Implementação da Alta Disponibilidade (EC2, ALB, ASG)

1.  **Launch Template (Modelo de Execução):**
    * Foi criado um modelo com o nome `wordpress-lt`, servindo como "molde" para todas as instâncias da aplicação.
    * **AMI:** Foi utilizada a **Amazon Linux 2023**.
    * **Tipo de instância:** `t2.micro`.
    * **Grupo de segurança:** `wordpress-ec2-sg`.
    * **Dados do usuário (User Data):** O modelo foi configurado com script `user-data`, responsável por toda a configuração da instância no momento da inicialização.

2.  **Application Load Balancer (ALB):**
    * Para distribuir o tráfego, primeiramente foi criado um **Grupo de Destino** (`wordpress-tg`) para `Instâncias`, operando na `wordpress-vpc` com o protocolo `HTTP:80`.
    * Em seguida, foi provisionado um **Application Load Balancer** (`wordpress-alb`), configurado como `Voltado para a Internet`. Ele foi mapeado para operar nas duas **sub-redes públicas** e protegido pelo Security Group `wordpress-alb-sg`. A regra principal do seu *listener* na porta 80 foi configurada para encaminhar todo o tráfego para o grupo de destino `wordpress-tg`.

3.  **Auto Scaling Group (ASG):**
    * Por fim, foi criado um ASG com o nome `wordpress-asg` para automatizar o ciclo de vida das instâncias.
    * **Configuração:** O ASG foi associado ao `wordpress-lt` e configurado para lançar instâncias nas duas **sub-redes PRIVADAS**, garantindo a segurança.
    * **Integração:** Foi anexado ao grupo de destino `wordpress-tg` do ALB, com as verificações de integridade do ELB habilitadas.
    * **Escalabilidade:** Foi definida uma **"Política de dimensionamento com monitoramento do objetivo"** para manter a `Utilização média da CPU` em `50%`, com a capacidade do grupo variando entre 2 e 3 instâncias, começando com 2.

---
### Script `user-data`

Este é o script validado que foi utilizado no Launch Template com a AMI **Amazon Linux 2023**.

```bash
#!/bin/bash
EFS_ID="id-do-efs"
RDS_ENDPOINT="endpoint-do-rds"
DB_NAME="wordpress"
DB_USER="admin"
DB_PASSWORD="SENHA-DO-RDS"

#INSTALAÇÃO E CONFIGURAÇÃO
yum update -y
yum install -y docker amazon-efs-utils

systemctl start docker
systemctl enable docker
usermod -a -G docker ec2-user

curl -L "[https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname](https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname) -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# MONTAGEM DO EFS
EFS_MOUNT_POINT="/mnt/efs-wordpress"
mkdir -p ${EFS_MOUNT_POINT}
echo "${EFS_ID}:/ ${EFS_MOUNT_POINT} efs _netdev,tls 0 0" >> /etc/fstab
mount -a -t efs

# PREPARAÇÃO E EXECUÇÃO DO DOCKER COMPOSE
COMPOSE_DIR="/home/ec2-user/wordpress"
mkdir -p ${COMPOSE_DIR}

cat <<EOF > ${COMPOSE_DIR}/docker-compose.yml
version: '3.8'
services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: "${RDS_ENDPOINT}"
      WORDPRESS_DB_USER: "${DB_USER}"
      WORDPRESS_DB_PASSWORD: "${DB_PASSWORD}"
      WORDPRESS_DB_NAME: "${DB_NAME}"
    volumes:
      - ${EFS_MOUNT_POINT}:/var/www/html
EOF

# PERMISSÕES E EXECUÇÃO
chmod 777 ${EFS_MOUNT_POINT}
chown -R ec2-user:ec2-user ${COMPOSE_DIR}

cd ${COMPOSE_DIR}
/usr/local/bin/docker-compose up -d
```
