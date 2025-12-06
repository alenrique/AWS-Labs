# Lab - Migrando para o Amazon RDS

Neste laboratório, realizei a migração do banco de dados da aplicação web "Café". Originalmente, o banco de dados rodava localmente na mesma instância EC2 da aplicação. O objetivo foi migrar esses dados para uma instância gerenciada Amazon RDS (MariaDB), garantindo maior escalabilidade e facilidade de gerenciamento.

A arquitetura final separa a camada de aplicação da camada de dados, colocando o banco de dados em sub-redes privadas.

<img width="816" height="904" alt="image" src="https://github.com/user-attachments/assets/c6cc399a-0377-40ab-b95b-41652e525c24" />

<img width="1516" height="878" alt="image" src="https://github.com/user-attachments/assets/4aa1eb74-7232-4d19-9e10-0c9f6e11140f" />

## Tarefa 1: Gerar dados de pedidos no site do Café

Antes de migrar, acessei a aplicação atual (rodando com banco local) e fiz alguns pedidos para gerar dados.
Anotei o número de pedidos no histórico para comparar após a migração, garantindo que nenhum dado foi perdido.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/fb3548f8-b31e-40c1-b9f3-46959005cd28" />

## Tarefa 2: Criar uma instância Amazon RDS usando a AWS CLI

Para criar a infraestrutura do banco de dados, utilizei a linha de comando a partir da instância CLI Host.

### 2.1 Configuração da CLI

Conectei ao CLI Host e configurei minhas credenciais:

```bash
aws configure
```

### 2.2 Criar componentes de pré-requisito

Para garantir a segurança e a rede correta para o RDS, criei os seguintes recursos:

1. **Security Group do Banco de Dados (`CafeDatabaseSG`)**:

```bash
aws ec2 create-security-group --group-name CafeDatabaseSG --description "Security group for Cafe database" --vpc-id <CafeInstance VPC ID>
```

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/6b63e496-94fe-495c-96a8-3f1deffc6009" />

2. **Regra de Entrada**: Permiti tráfego na porta 3306 (MySQL) vindo apenas do Security Group da aplicação:

```bash
aws ec2 authorize-security-group-ingress --group-id <CafeDatabaseSG Group ID> --protocol tcp --port 3306 --source-group <CafeSecurityGroup Group ID>
```

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/dad3886c-22b0-463a-bdc7-670c473d355a" />

3. **Sub-redes Privadas**: Criei duas sub-redes privadas em zonas de disponibilidade diferentes para o grupo de sub-redes do banco:

```bash
# Subnet 1
aws ec2 create-subnet --vpc-id <CafeInstance VPC ID> --cidr-block 10.200.2.0/23 --availability-zone <CafeInstance Availability Zone>

# Subnet 2
aws ec2 create-subnet --vpc-id <CafeInstance VPC ID> --cidr-block 10.200.10.0/23 --availability-zone <Outra AZ>
```

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/64c38474-85e6-431b-a628-d95ea22b381d" />

4. **DB Subnet Group**: Agrupei as duas sub-redes criadas:

```bash
aws rds create-db-subnet-group --db-subnet-group-name "CafeDB Subnet Group" --db-subnet-group-description "DB subnet group for Cafe" --subnet-ids <ID Subnet 1> <ID Subnet 2> --tags "Key=Name,Value= CafeDatabaseSubnetGroup"
```

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/3822a3f2-37da-43e2-838d-70035aefb540" />

### 2.3 Criar a Instância RDS MariaDB

Finalmente, executei o comando para criar o banco de dados gerenciado:

```bash
aws rds create-db-instance \
--db-instance-identifier CafeDBInstance \
--engine mariadb \
--engine-version 10.11.8 \
--db-instance-class db.t3.micro \
--allocated-storage 20 \
--availability-zone <CafeInstance Availability Zone> \
--db-subnet-group-name "CafeDB Subnet Group" \
--vpc-security-group-ids <CafeDatabaseSG Group ID> \
--no-publicly-accessible \
--master-username root --master-user-password 'Re:Start!9'
```

Monitorei a criação até que o status mudasse para `available` e copiei o **Endpoint Address** do RDS.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/d18d1b0e-b08d-4708-9168-cd2af614420d" />

## Tarefa 3: Migrar dados da aplicação para o Amazon RDS

Com o RDS pronto, iniciei o processo de migração dos dados.

1. **Backup do Banco Local**: Na instância da aplicação (`CafeInstance`), usei o `mysqldump` para extrair os dados atuais:

```bash
mysqldump --user=root --password='Re:Start!9' --databases cafe_db --add-drop-database > cafedb-backup.sql
```

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/24a31f68-d9fb-46df-a5a0-8bc0e4692789" />

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/3b99cbda-ccb2-4707-8009-89d4b1906d2d" />

2. **Restauração no RDS**: Enviei os dados do arquivo `.sql` diretamente para o novo endpoint do RDS:

```bash
mysql --user=root --password='Re:Start!9' --host=<Endpoint do RDS> < cafedb-backup.sql
```

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/ffa20139-94af-4405-89db-fa14aea3c352" />

3. **Verificação**: Conectei interativamente no RDS e fiz um `select` na tabela de produtos para confirmar que os dados estavam lá.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/18a5bee1-3a09-4d38-a8c0-4da4ca6d5af4" />

## Tarefa 4: Configurar o site para usar o Amazon RDS

Agora precisei "apontar" a aplicação para o novo banco. Como a aplicação utiliza o **AWS Systems Manager Parameter Store** para gerenciar configurações:

1. Acessei o console do **Systems Manager > Parameter Store**.

2. Editei o parâmetro `/cafe/dbUrl`.

3. Substituí o valor antigo pelo **Endpoint Address** do novo RDS.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/70e14017-9348-4816-8289-38e8543ccfeb" />

Ao recarregar o site e acessar o "Order History", confirmei que os pedidos feitos na Tarefa 1 estavam lá, provando que a aplicação agora lê do RDS migrado com sucesso.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/22d72004-b55f-4b83-b67b-cb2c09a08a44" />

## Tarefa 5: Monitorar o banco de dados Amazon RDS

Para finalizar, explorei as capacidades de monitoramento do RDS.

1. No console RDS, selecionei a instância `cafedbinstance` e fui na aba **Monitoring**.

2. Observei métricas como `CPUUtilization` e `FreeableMemory`.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/61b7cd06-05a4-4c08-bc90-4d7c9ed8f94e" />

3. Testei a métrica `DatabaseConnections` abrindo uma conexão manual via terminal e observando o gráfico atualizar no CloudWatch, confirmando a conexão ativa.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/c82e84d0-7766-424e-a626-47d1e9f90090" />
