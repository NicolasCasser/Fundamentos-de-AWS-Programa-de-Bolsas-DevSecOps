# Projeto: Implantação de WordPress em Alta Disponibilidade na AWS

Este repositório documenta a implementação de uma arquitetura de alta disponibilidade para a plataforma WordPress na nuvem AWS. O projeto foi desenvolvido como parte do programa de estudos "Fundamentos de AWS - DevSecOps" e visa demonstrar competências práticas em infraestrutura como código, provisionamento de recursos na nuvem e arquiteturas resilientes.

O objetivo é implantar o WordPress de forma escalável e tolerante a falhas, utilizando os principais serviços gerenciados da AWS para garantir desempenho e disponibilidade, simulando um ambiente de produção real.

## Arquitetura Proposta

A arquitetura distribui a aplicação em múltiplas instâncias EC2, gerenciadas por um Auto Scaling Group e com o tráfego balanceado por um Application Load Balancer. O armazenamento de arquivos é centralizado no Amazon EFS, e os dados são gerenciados por um banco de dados relacional Amazon RDS.

*(Dica: Tire um print screen do diagrama do seu PDF e adicione aqui. Para isso, faça o upload da imagem para o seu repositório GitHub e use o seguinte link:)*
`![Diagrama da Arquitetura](caminho/para/sua/imagem.png)`

## Componentes da Arquitetura na AWS

* **VPC (Virtual Private Cloud):** Rede virtual personalizada e isolada para hospedar todos os recursos do projeto.
    * **2 Sub-redes Públicas:** Utilizadas para os recursos que precisam de acesso direto à internet, como o Application Load Balancer.
    * **2 Sub-redes Privadas:** Utilizadas para os recursos de back-end, como as instâncias EC2 e o banco de dados RDS, para maior segurança.
    * **Internet Gateway:** Permite a comunicação entre a VPC e a internet.
    * **NAT Gateway:** Permite que as instâncias nas sub-redes privadas iniciem conexões com a internet (para atualizações), mas não sejam acessadas externamente.
* **Amazon EC2 (Elastic Compute Cloud):**
    * **Auto Scaling Group:** Automatiza o provisionamento e o escalonamento das instâncias EC2, garantindo que a aplicação se adapte à demanda de tráfego com base no uso de CPU.
    * **Launch Template:** Define o modelo de configuração para cada instância EC2, incluindo a AMI, o tipo de instância e o script de inicialização (`user-data`).
    * **Aplicação:** O WordPress é executado via Docker em cada instância.
* **Application Load Balancer (ALB):** Distribui o tráfego de entrada de forma automática entre as múltiplas instâncias EC2 ativas.
* **Amazon RDS (Relational Database Service):** Fornece um banco de dados MySQL/MariaDB gerenciado, altamente disponível e seguro, localizado nas sub-redes privadas.
* **Amazon EFS (Elastic File System):** Oferece um sistema de arquivos de rede compartilhado e escalável, utilizado para armazenar o conteúdo do WordPress (plugins, temas, uploads) de forma centralizada.

## Etapas de Configuração

O provisionamento da infraestrutura seguiu as seguintes etapas manuais via Console da AWS:

1.  **Configuração da Rede (VPC):**
    * Criação da VPC personalizada com 4 sub-redes em 2 Zonas de Disponibilidade.
    * Configuração do Internet Gateway, NAT Gateway e das Tabelas de Rotas para segmentar o tráfego público e privado.
2.  **Configuração dos Grupos de Segurança:**
    * Criação de grupos de segurança específicos para as instâncias EC2, o banco de dados RDS e o sistema de arquivos EFS, garantindo a comunicação segura e restrita entre eles.
3.  **Provisionamento do Banco de Dados:**
    * Criação da instância Amazon RDS (MySQL/MariaDB) nas sub-redes privadas.
4.  **Provisionamento do Sistema de Arquivos:**
    * Criação do Amazon EFS com pontos de montagem (mount targets) nas sub-redes privadas.
5.  **Criação do Molde da Aplicação:**
    * Desenvolvimento de um script de inicialização (`user-data`) para automatizar a instalação do Docker, a montagem do EFS e a configuração do WordPress.
    * Criação de um Launch Template contendo a AMI, as configurações de rede e o script `user-data`.
6.  **Configuração do Auto Scaling e Load Balancer (Próximos Passos):**
    * Criação do Auto Scaling Group associado ao Launch Template e às sub-redes privadas.
    * Criação do Application Load Balancer nas sub-redes públicas para distribuir o tráfego para o Auto Scaling Group.

## Status do Projeto

🚧 **Em desenvolvimento** 🚧

## Possíveis Melhorias Futuras

Para uma arquitetura de produção ainda mais robusta, o projeto poderia ser estendido para incluir:

* Uso do Amazon Route 53 para gerenciamento de DNS.
* Ativação da funcionalidade Multi-AZ no Amazon RDS para maior tolerância a falhas do banco de dados.
* Implementação de uma camada de cache com Amazon ElastiCache (Memcached/Redis).
* Provisionamento da infraestrutura como código usando Terraform ou AWS CloudFormation.
* Monitoramento avançado e alarmes com Amazon CloudWatch.
