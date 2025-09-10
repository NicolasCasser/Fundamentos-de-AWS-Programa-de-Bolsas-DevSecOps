# Projeto: WordPress em Alta Disponibilidade 

Este repositório documenta a implementação de uma arquitetura de alta disponibilidade para a plataforma WordPress na nuvem AWS. 

O objetivo é implantar o WordPress de forma escalável e tolerante a falhas, utilizando os principais serviços gerenciados da AWS para garantir desempenho e disponibilidade, simulando um ambiente de produção real.

## Arquitetura Proposta

A arquitetura final distribui a aplicação em múltiplas instâncias EC2, gerenciadas por um Auto Scaling Group e com o tráfego balanceado por um Application Load Balancer. O armazenamento de arquivos é centralizado no Amazon EFS, e os dados são gerenciados por um banco de dados relacional Amazon RDS.

![Diagrama da Arquitetura](imagens/Diagrama-WordPress.png)

---

## Etapas de Configuração

Esta seção serve como um guia passo a passo para reconstruir o ambiente completo.

### Fase 1: Configuração da Rede (VPC)
A rede foi configurada com uma VPC personalizada contendo sub-redes públicas e privadas em duas Zonas de Disponibilidade para garantir a resiliência.

1.  **VPC:** Criada com o nome `wordpress-vpc` e CIDR `10.0.0.0/16`.
2.  **Sub-redes:**
    * Duas sub-redes públicas (`10.0.1.0/24`, `10.0.2.0/24`) para recursos com acesso à internet, como o Load Balancer.
    * Duas sub-redes privadas (`10.0.3.0/24`, `10.0.4.0/24`) para recursos protegidos, como as instâncias EC2 e o RDS.
3.  **Gateways e Rotas:**
    * Um **Internet Gateway** foi criado e atrelado a uma tabela de rotas para as sub-redes públicas.
    * Um **NAT Gateway** foi posicionado em uma sub-rede pública para permitir que as instâncias privadas acessem a internet para atualizações.

### Fase 2: Configuração da Segurança (Security Groups)
Foram criados grupos de segurança específicos para controlar o tráfego entre os componentes:

1.  **`wordpress-db-sg` (RDS):** Permite tráfego na porta `3306` apenas a partir do grupo `wordpress-ec2-sg`.
2.  **`wordpress-efs-sg` (EFS):** Permite tráfego na porta `2049` (NFS) apenas a partir do grupo `wordpress-ec2-sg`.
3.  **`wordpress-alb-sg` (Load Balancer):** Permite tráfego público na porta `80` (HTTP) de qualquer lugar (`0.0.0.0/0`).
4.  **`wordpress-ec2-sg` (Instâncias EC2):** Permite tráfego na porta `80` apenas a partir do grupo `wordpress-alb-sg`, e na porta `22` (SSH) para acesso de manutenção.

### Fase 3: Provisionamento dos Serviços de Dados (RDS & EFS)

1.  **Amazon EFS:** Um sistema de arquivos elástico foi criado para armazenar de forma centralizada os arquivos do WordPress, como temas, plugins e uploads de mídia (fotos, vídeos).
2.  **Amazon RDS:** Uma instância de banco de dados (MySQL/MariaDB) foi provisionada nas sub-redes privadas para armazenar os dados da aplicação de forma segura e gerenciada.

### Fase 4: Implementação da Alta Disponibilidade (EC2, ALB, ASG)

1.  **Launch Template:** Foi criado um "molde" chamado `wordpress-lt` baseado na AMI **Amazon Linux 2023** e no tipo de instância **t2.micro**. Este molde inclui o script `user-data` abaixo, que provou ser funcional.

    ```bash
    #!/bin/bash
    # Script user-data validado para Amazon Linux 2
    EFS_ID="seu-id-do-efs"
    RDS_ENDPOINT="seu-endpoint-do-rds"
    DB_NAME="wordpress"
    DB_USER="seu-usuario-db"
    DB_PASSWORD="SUA-SENHA-DO-RDS"

    yum update -y
    yum install -y docker amazon-efs-utils

    systemctl start docker
    systemctl enable docker
    usermod -a -G docker ec2-user

    curl -L "[https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname](https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname) -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    chmod +x /usr/local/bin/docker-compose

    EFS_MOUNT_POINT="/mnt/efs-wordpress"
    mkdir -p ${EFS_MOUNT_POINT}
    echo "${EFS_ID}:/ ${EFS_MOUNT_POINT} efs _netdev,tls 0 0" >> /etc/fstab
    mount -a -t efs

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

    chmod 777 ${EFS_MOUNT_POINT}
    chown -R ec2-user:ec2-user ${COMPOSE_DIR}

    cd ${COMPOSE_DIR}
    /usr/local/bin/docker-compose up -d
    ```

2.  **Application Load Balancer (ALB):** Um ALB foi criado nas sub-redes públicas para distribuir o tráfego de entrada entre as instâncias EC2.
3.  **Auto Scaling Group (ASG):** Um ASG foi configurado para manter um mínimo de 2 instâncias rodando nas sub-redes privadas. Foi definida uma **"Política de dimensionamento com monitoramento do objetivo"** para escalar com base na utilização média da CPU, garantindo a escalabilidade da aplicação.

---

## Análise e Descobertas do Processo

O processo de implementação revelou desafios técnicos importantes, principalmente relacionados às limitações de recursos da instância `t2.micro`.

1.  **Desafio de Recursos:** A combinação do sistema operacional, Docker e a aplicação WordPress (com servidor Apache) consome uma quantidade de memória e CPU que leva a instância `t2.micro` ao seu limite.
2.  **Conflito de Permissões com EFS:** Foi identificado um complexo problema de permissões entre o contêiner Docker e o volume de rede EFS (NFS), que impedia o WordPress de salvar arquivos de mídia. A solução foi aplicar permissões abertas (`chmod 777`) no ponto de montagem do EFS.
3.  **Conclusão sobre o Painel de Admin:** A arquitetura se provou funcional para servir o front-end do site. No entanto, o acesso ao painel de administração (`/wp-admin`), que é uma aplicação muito mais pesada, resulta em timeouts.

## Status do Projeto (Ponto de Parada)

* **IMPLANTADO:** Arquitetura de alta disponibilidade (VPC, ALB, ASG, RDS, EFS) está no ar e funcional.
* **VALIDADO:** O site (front-end) é servido corretamente através do Load Balancer e as instâncias são gerenciadas pelo Auto Scaling.
* **PROBLEMA ATUAL:** O acesso ao painel de administração (`/wp-admin`) resulta em **timeout** devido ao esgotamento de recursos das instâncias `t2.micro`.
