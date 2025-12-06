# Lab - Utilizando Auto Scaling na AWS (Linux)

Neste laboratório, utilizei a AWS CLI para criar uma instância EC2 base e gerar uma AMI personalizada a partir dela. Em seguida, utilizei essa imagem para configurar uma arquitetura de alta disponibilidade com Elastic Load Balancer (ELB) e Auto Scaling, garantindo que a aplicação escale automaticamente conforme a demanda de CPU.

A arquitetura final consiste em um Load Balancer distribuindo tráfego para instâncias em sub-redes privadas, que escalam horizontalmente.

<img width="1196" height="690" alt="image" src="https://github.com/user-attachments/assets/8d391e40-e4e8-4ee4-bf9c-a19eb3e1d917" />

<img width="1704" height="1038" alt="image" src="https://github.com/user-attachments/assets/f4aa2335-bbc5-49f5-91f2-670e3a56a18c" />

## Tarefa 1: Criar uma nova AMI para o Auto Scaling

Para começar, precisei criar a "imagem base" (AMI) que o Auto Scaling usará. Fiz todo esse processo via linha de comando (CLI) a partir de uma instância de gerenciamento chamada ```Command Host```.

### 1.1 Conexão e Configuração

Conectei-me ao ```Command Host``` via EC2 Instance Connect e configurei a CLI:

```bash
aws configure
```

* **Region**: ```us-west-2```

* **Format**: ```json```

<img width="1920" height="962" alt="image" src="https://github.com/user-attachments/assets/fdcc7b07-20a9-4bc8-b8a5-6c48c39ad787" />

<img width="1920" height="962" alt="image" src="https://github.com/user-attachments/assets/cebaef91-0bba-4b14-9999-73a08eef5922" />

### 1.2 Criar a Instância Base via CLI

Utilizei um script de ```user-data``` (já existente na máquina) para instalar a aplicação web. Executei o comando abaixo substituindo as variáveis (```KEYNAME```, ```AMIID```, etc) pelos valores fornecidos no laboratório:

```bash
aws ec2 run-instances --key-name KEYNAME --instance-type t3.micro --image-id AMIID --user-data file:///home/ec2-user/UserData.txt --security-group-ids HTTPACCESS --subnet-id SUBNETID --associate-public-ip-address --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=WebServer}]' --output text --query 'Instances[*].InstanceId'
```

Monitorei o status até que a instância estivesse rodando:

```bash
aws ec2 wait instance-running --instance-ids [ID-DA-NOVA-INSTANCIA]
```

<img width="1920" height="962" alt="image" src="https://github.com/user-attachments/assets/3f8080ce-8384-49eb-a3b9-af6d97ede1b8" />

Verifiquei se o servidor web estava funcional acessando o DNS público gerado.

<img width="1920" height="1009" alt="image" src="https://github.com/user-attachments/assets/cef6a37d-a9ad-4b09-84f2-322a8ca498b1" />

### 1.3 Criar a AMI Personalizada

Com a instância configurada, gerei a imagem que servirá de modelo:

```bash
aws ec2 create-image --name WebServerAMI --instance-id [ID-DA-NOVA-INSTANCIA]
```

<img width="1920" height="965" alt="image" src="https://github.com/user-attachments/assets/307add35-d99f-4783-9506-6472ddd1fc8b" />

## Tarefa 2: Criar um ambiente de Auto Scaling

Nesta etapa, fui para o Console da AWS para configurar a infraestrutura escalável.

### 2.1 Application Load Balancer (ALB)

Criei um Load Balancer chamado ```WebServerELB```:

* **VPC**: Lab VPC.

* **Subnets**: Selecionei as duas sub-redes Públicas.

* **Security Group**: ```HTTPAccess```.

* **Target Group**: Criei um novo grupo (```webserver-app```) apontando para a porta 80 e configurando o health check para ```/index.php```.

<img width="1920" height="965" alt="image" src="https://github.com/user-attachments/assets/221eca41-c964-4441-832c-9d5acd80c9e4" />

### 2.2 Launch Template

Criei o modelo de lançamento ```web-app-launch-template```:

* **AMI**: Selecionei a ```WebServerAMI``` criada na Tarefa 1.

* **Instance Type**: ```t3.micro```.

* **Security Group**: ```HTTPAccess```.

* **Key pair**: Não incluí chave (acesso não necessário para o teste).

<img width="1920" height="965" alt="image" src="https://github.com/user-attachments/assets/1b2f23a5-fefb-4d5e-ae91-d2cbc760a0a1" />

### 2.3 Auto Scaling Group (ASG)

Configurei o grupo ```Web App Auto Scaling Group```:

* **Launch Template**: ```web-app-launch-template```.

* **Rede**: Selecionei as duas sub-redes Privadas.

* **Load Balancer**: Associei ao Target Group ```webserver-app```.

* **Capacidade**: Min: 2, Desired: 2, Max: 4.

* **Scaling Policy**: Configurei o **Target Tracking** para manter a utilização média de **CPU em 50%**.

<img width="1920" height="965" alt="image" src="https://github.com/user-attachments/assets/8d4fe98a-18f8-4f43-ad67-753bbf9f6907" />

### Tarefa 3: Verificar a configuração

Aguardei o Auto Scaling provisionar as duas instâncias iniciais.
Fui até a seção **Target Groups** e verifiquei se as duas novas instâncias (```WebApp```) estavam com o status **Healthy**.

<img width="1920" height="965" alt="image" src="https://github.com/user-attachments/assets/4c31510a-3271-4d9c-9979-fa7426982707" />

### Tarefa 4: Testar o Auto Scaling

Para validar a elasticidade da arquitetura:

1. Acessei a URL DNS do Load Balancer.

2. Na aplicação web, cliquei em **Start Stress**. Isso gera uma carga artificial de 100% de CPU na instância que recebeu a requisição.

3. Voltei ao console do **Auto Scaling Group** e observei a aba **Activity**.

4. Após alguns minutos (devido ao alarme do CloudWatch disparar acima de 50% de CPU média), o Auto Scaling iniciou o lançamento de uma nova instância para dividir a carga.

Isso confirmou que a política de escalabilidade funcionou corretamente, adicionando recursos quando a demanda aumentou.

<img width="1920" height="1009" alt="image" src="https://github.com/user-attachments/assets/2ea560d6-b399-4cb6-92c1-11f063a7e87c" />

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/2aad4b90-42e0-4759-a4cd-8e71ce91ebac" />
