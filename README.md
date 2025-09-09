# Projeto: WordPress em Alta Disponibilidade 

Este repositório documenta a implementação de uma arquitetura de alta disponibilidade para a plataforma WordPress na nuvem AWS. O projeto foi desenvolvido como parte do programa de estudos "Fundamentos de AWS - DevSecOps".

O objetivo é implantar o WordPress de forma escalável e tolerante a falhas, utilizando os principais serviços gerenciados da AWS para garantir desempenho e disponibilidade.

## Arquitetura Proposta

A arquitetura final distribuirá a aplicação em múltiplas instâncias EC2, gerenciadas por um Auto Scaling Group e com o tráfego balanceado por um Application Load Balancer. O armazenamento de arquivos será centralizado no Amazon EFS, e os dados serão gerenciados por um banco de dados relacional Amazon RDS.

*(Adicione aqui o print screen do diagrama da arquitetura)*
`![Diagrama da Arquitetura](caminho/para/sua/imagem.png)`

---

## Etapas de Configuração Detalhadas (Até o Momento)

Esta seção serve como um guia passo a passo para reconstruir o ambiente de base validado.

### Fase 1: Configuração da Fundação de Rede (VPC)

1.  **Criar a VPC:**
    * **Nome:** `wordpress-vpc`
    * **IPv4 CIDR block:** `10.0.0.0/16`

2.  **Criar Sub-redes (Subnets):**
    * `wordpress-public-subnet-a` (em uma AZ, ex: us-east-1a) com CIDR `10.0.1.0/24`.
    * `wordpress-public-subnet-b` (em outra AZ, ex: us-east-1b) com CIDR `10.0.2.0/24`.
    * `wordpress-private-subnet-a` (na mesma AZ da public-a) com CIDR `10.0.3.0/24`.
    * `wordpress-private-subnet-b` (na mesma AZ da public-b) com CIDR `10.0.4.0/24`.

3.  **Criar Gateways:**
    * **Internet Gateway:** Criar um com o nome `wordpress-igw` e anexá-lo à `wordpress-vpc`.
    * **NAT Gateway:** Criar um com o nome `wordpress-nat-gateway`, alocando um novo Elastic IP e posicionando-o em uma das sub-redes públicas (ex: `wordpress-public-subnet-a`).

4.  **Configurar Tabelas de Rotas (Route Tables):**
    * **Tabela Pública (`wordpress-public-rt`):**
        * Criar uma nova tabela de rotas.
        * Adicionar uma rota: `Destination: 0.0.0.0/0` -> `Target: Internet Gateway (wordpress-igw)`.
        * Associar esta tabela às duas sub-redes públicas.
    * **Tabela Privada (`wordpress-private-rt`):**
        * Usar a tabela de rotas padrão da VPC.
        * Adicionar uma rota: `Destination: 0.0.0.0/0` -> `Target: NAT Gateway (wordpress-nat-gateway)`.
        * Garantir que esta tabela esteja associada às duas sub-redes privadas.

### Fase 2: Configuração da Segurança (Security Groups)

1.  **`wordpress-db-sg` (Para o RDS):**
    * **Descrição:** `Permite acesso ao banco de dados apenas das instancias EC2`.
    * **Regra de Entrada:** `Type: MYSQL/Aurora` (Porta 3306), `Source: wordpress-ec2-sg` (referência ao grupo das instâncias).

2.  **`wordpress-efs-sg` (Para o EFS):**
    * **Descrição:** `Permite acesso NFS das instancias EC2`.
    * **Regra de Entrada:** `Type: NFS` (Porta 2049), `Source: wordpress-ec2-sg`.

3.  **`wordpress-ec2-sg` (Para as Instâncias EC2):**
    * **Descrição:** `Permite trafego web e SSH para instancias de teste`.
    * **Regra de Entrada (para o teste):** `Type: HTTP` (Porta 80), `Source: Anywhere-IPv4 (0.0.0.0/0)`.
    * **Regra de Entrada (para acesso):** `Type: SSH` (Porta 22), `Source: My IP`.

### Fase 3: Provisionamento dos Serviços de Dados (RDS & EFS)

1.  **Amazon EFS:**
    * **Nome:** `wordpress-efs`.
    * **VPC:** `wordpress-vpc`.
    * **Mount Targets:** Criar um em cada sub-rede privada (`wordpress-private-subnet-a` e `wordpress-private-subnet-b`), ambos usando o security group `wordpress-efs-sg`.
    * **Ajuste Crítico na VPC:** Habilitar as opções **"Enable DNS resolution"** e **"Enable DNS hostnames"** nas configurações da `wordpress-vpc`. Sem isso, o `mount` do EFS falhará.

2.  **Amazon RDS:**
    * **DB Subnet Group:** Criar um grupo de sub-redes (`wordpress-db-subnet-group`) contendo as duas sub-redes privadas.
    * **Criação da Instância:**
        * **Engine:** MySQL ou MariaDB.
        * **Template:** Free tier.
        * **Identifier:** `wordpress-db`.
        * **Instance type:** `db.t3.micro` ou similar.
        * **VPC:** `wordpress-vpc`, usando o `wordpress-db-subnet-group` e o security group `wordpress-db-sg`.
        * **Public Access:** `No`.
    * **Criação Manual do Banco:** Após a instância estar "Available", usar uma EC2 temporária para conectar-se via cliente `mysql` e executar os comandos: `DROP DATABASE IF EXISTS wordpress;` e `CREATE DATABASE wordpress;`.

### Fase 4: Validação da Instância Base (Teste bem-sucedido)

O passo final foi validar uma configuração de instância única que funciona de forma estável.

1.  **Lançar a Instância EC2:**
    * **AMI:** Amazon Linux 2023 (Padrão).
    * **Instance Type:** `t2.micro`.
    * **Network:** `wordpress-vpc`, em uma **sub-rede pública**.
    * **Auto-assign public IP:** `Enable`.
    * **Security Group:** `wordpress-ec2-sg`.
    * **User Data:** Usar o script validado abaixo.

2.  **Script `user-data` Validado:**

    ```bash
    #!/bin/bash
    # Variaveis
    EFS_ID="seu-id-do-efs"
    RDS_ENDPOINT="seu-endpoint-do-rds"
    DB_USER="seu-usuario-db"
    DB_PASSWORD="SUA-SENHA-DO-RDS"

    AWS_REGION=$(curl -s [http://169.254.169.254/latest/meta-data/placement/region](http://169.254.169.254/latest/meta-data/placement/region))

    # Otimização: Criação de Swap File para estabilidade na t2.micro
    fallocate -l 2G /swapfile
    chmod 600 /swapfile
    mkswap /swapfile
    swapon /swapfile
    echo '/swapfile swap swap defaults 0 0' >> /etc/fstab

    # Instalação de pacotes
    dnf update -y
    dnf install -y docker nfs-utils mariadb105

    # Configuração do Docker
    systemctl start docker
    systemctl enable docker
    usermod -a -G docker ec2-user

    # Montagem e permissões do EFS
    mkdir -p /mnt/efs/wordpress
    mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport ${EFS_ID}.efs.${AWS_REGION}.amazonaws.com:/ /mnt/efs/wordpress
    chown -R 33:33 /mnt/efs/wordpress
    echo "${EFS_ID}.efs.${AWS_REGION}.amazonaws.com:/ /mnt/efs/wordpress nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev 0 0" >> /etc/fstab

    # Criação do banco de dados (garantia)
    mysql -h ${RDS_ENDPOINT} -u ${DB_USER} -p${DB_PASSWORD} -e "CREATE DATABASE IF NOT EXISTS wordpress;"

    # Executa o contêiner do WordPress com a montagem de volume corrigida
    docker run -d --name wordpress --restart always \
      -p 80:80 \
      -v /mnt/efs/wordpress:/var/www/html/wp-content \
      -e WORDPRESS_DB_HOST=${RDS_ENDPOINT} \
      -e WORDPRESS_DB_USER=${DB_USER} \
      -e WORDPRESS_DB_PASSWORD=${DB_PASSWORD} \
      -e WORDPRESS_DB_NAME=wordpress \
      wordpress
    ```

---

## Status do Projeto (Ponto de Parada)

* **CONCLUÍDO:** Configuração e validação de toda a infraestrutura base (VPC, Subnets, Gateways, SGs, RDS, EFS).
* **CONCLUÍDO:** Desenvolvimento e validação de um script `user-data` funcional para uma instância EC2 `t2.micro`.
* **PRÓXIMO PASSO:** Realizar o teste de persistência ("Prova de Fogo") na configuração bem-sucedida para confirmar que os dados sobrevivem à recriação da instância.
* **PENDENTE:** Fazer a transição para a configuração final com `docker-compose.yml` a partir do repositório Git.
* **PENDENTE:** Criar o Launch Template, Application Load Balancer e Auto Scaling Group.
