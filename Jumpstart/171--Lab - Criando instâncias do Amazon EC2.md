# Lab - Criando Instâncias Amazon EC2

Nesse laboratório, irei explorar diferentes formas de lançar instâncias Amazon EC2. Primeiro, utilizarei o Console de Gerenciamento da AWS para criar um **Bastion Host**. Em seguida, conectarei a esse Bastion Host e utilizarei a AWS CLI (Linha de Comando) para provisionar automatizadamente uma segunda instância que servirá como um servidor web.

A arquitetura final será composta por um Bastion Host em uma sub-rede pública, gerenciando o lançamento de outra instância web.

<img width="928" height="702" alt="image" src="https://github.com/user-attachments/assets/94fb50e8-f8df-4212-a6a7-d44ea7219704" />

## Tarefa 1: Lançar uma instância EC2 usando o Console AWS

Nesta etapa, criei a instância que servirá como Bastion Host.

Passos realizados no Console EC2:

1. Cliquei em Launch instance.

2. Name: Defini como ```Bastion host```.

3. AMI: Selecionei Amazon Linux 2.

4. Instance Type: Escolhi ```t3.micro```.

5. Key pair: Selecionei "Proceed without key pair" (visto que usarei o EC2 Instance Connect).

6. Network settings:

    * VPC: ```Lab VPC```

    * Subnet: ```Public Subnet```

    * Auto-assign public IP: ```Enable```

    * Security Group: Criei um novo chamado ```Bastion security group``` com permissão para SSH.

7. Advanced details: Em "IAM instance profile", selecionei ```Bastion-Role``` (Isso é crucial para permitir que essa instância execute comandos da AWS CLI posteriormente).

Após revisar, lancei a instância.

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/d6b5a60a-942d-477e-bcf4-cdef7df28bfd" />

## Tarefa 2: Fazer login no Bastion Host

Com a instância ```Bastion host``` em execução ("Running"), selecionei a instância e cliquei em **Connect**.
Utilizei a opção **EC2 Instance Connect** para abrir um terminal diretamente no navegador.

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/de7e9848-ab83-4dcc-bb0c-b22cfc3da7e9" />

## Tarefa 3: Lançar uma instância EC2 usando a AWS CLI

Agora, dentro do terminal do Bastion Host, utilizei a AWS CLI para lançar o servidor web. Para isso, precisei coletar dinamicamente os IDs da AMI, da Subnet e do Security Group.

### Passo 1: Obter a AMI mais recente

Executei o script abaixo para configurar a região e buscar o ID da imagem do Amazon Linux 2 no Parameter Store:

```bash
# Definir a Região baseada na AZ atual
AZ=`curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone`
export AWS_DEFAULT_REGION=${AZ::-1}

# Recuperar a AMI mais recente do Linux 2 via SSM Parameter Store
AMI=$(aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 --query 'Parameters[0].[Value]' --output text)
echo $AMI
```

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/005df412-1821-4455-931c-1cd3b0cf7a7d" />

### Passo 2: Obter o ID da Subnet Pública

Busquei o ID da ```Public Subnet``` usando filtros de tag:

```bash
SUBNET=$(aws ec2 describe-subnets --filters 'Name=tag:Name,Values=Public Subnet' --query Subnets[].SubnetId --output text)
echo $SUBNET
```

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/8ded7053-3b0b-4cd7-9771-62716c24a344" />

### Passo 3: Obter o Security Group Web

O laboratório já possuía um grupo de segurança pré-configurado chamado ```WebSecurityGroup```. Recuperei seu ID:

```bash
SG=$(aws ec2 describe-security-groups --filters Name=group-name,Values=WebSecurityGroup --query SecurityGroups[].GroupId --output text)
echo $SG
```

<img width="1917" height="960" alt="Captura de tela de 2025-12-05 10-15-04" src="https://github.com/user-attachments/assets/524094de-8364-46dc-9f34-dac3691888f2" />

### Passo 4: Baixar o script de User Data

Para que o servidor web instale o Apache e a aplicação automaticamente ao iniciar, baixei um script de User Data:

```bash
wget [https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-100-RSJAWS-1-23732/171-lab-JAWS-create-ec2/s3/UserData.txt](https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-100-RSJAWS-1-23732/171-lab-JAWS-create-ec2/s3/UserData.txt)
cat UserData.txt
```

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/9a04e6c1-25b2-4bae-884e-50c141de3ddd" />


### Passo 5: Lançar a instância

Com todas as variáveis definidas (```$AMI```, ```$SUBNET```, ```$SG```), executei o comando ```run-instances``` para criar o servidor:

```bash
INSTANCE=$(\
aws ec2 run-instances \
--image-id $AMI \
--subnet-id $SUBNET \
--security-group-ids $SG \
--user-data file:///home/ec2-user/UserData.txt \
--instance-type t3.micro \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Web Server}]' \
--query 'Instances[*].InstanceId' \
--output text \
)
echo $INSTANCE
```

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/b299c3a7-fdb3-4408-88ba-40e0613f1d31" />

### Passo 6: Verificar e Testar

Vi todas as informções da instância:

```bash
aws ec2 describe-instances --instance-ids $INSTANCE
```
<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/d0ec4751-d6bf-4b2f-8ea9-7f4d7cad9b38" />

Monitorei o status da criação até que mudasse para "running":

```bash
aws ec2 describe-instances --instance-ids $INSTANCE --query 'Reservations[].Instances[].State.Name' --output text
```

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/27bd5c9f-c7dc-468c-837b-64800e486a10" />

### Passo 7: Testar o servidor web

Com a instância rodando, precisei validar se o servidor web foi instalado e configurado corretamente pelo script de User Data.

Recuperei o DNS público da nova instância:

```bash
aws ec2 describe-instances --instance-ids $INSTANCE --query Reservations[].Instances[].PublicDnsName --output text
```

Copiei o endereço DNS gerado (ex: ```ec2-xx-xx-xx.us-west-2.compute.amazonaws.com```), colei no navegador e confirmei que a página da aplicação web foi carregada com sucesso.

<img width="1917" height="1008" alt="image" src="https://github.com/user-attachments/assets/e0cb3f3d-daaa-4383-9c99-15ffdf0e265c" />

## Desafios Opcionais

Aproveitei o tempo restante para solucionar problemas em uma instância chamada Misconfigured Web Server que já existia no laboratório.

### Desafio 1: Conectar a uma instância EC2

**Problema**: Tentei conectar na instância "Misconfigured Web Server" usando o EC2 Instance Connect, mas a conexão falhava.

<img width="1917" height="961" alt="image" src="https://github.com/user-attachments/assets/5be4e3c8-b09b-4da4-beb5-f9a2b560891d" />

**Diagnóstico e Solução**:

1. Selecionei a instância e verifiquei a aba Security.

2. Percebi que o Security Group associado não possuía uma regra de entrada (Inbound Rule) permitindo tráfego na porta 22 (SSH).

3. Editei as regras de entrada do Security Group, adicionando permissão para SSH (Porta 22).

4. Tentei conectar novamente e obtive sucesso.

<img width="1917" height="961" alt="image" src="https://github.com/user-attachments/assets/11377ed8-54fd-49a1-87e2-887ccb647445" />

<img width="1917" height="961" alt="image" src="https://github.com/user-attachments/assets/3f2ffc32-8595-48a9-a377-b74682fdbdc5" />

### Desafio 2: Corrigir a instalação do servidor web

**Problema**: Ao acessar o DNS público da instância "Misconfigured Web Server" no navegador, a página não carregava.

**Diagnóstico e Solução**:

1. Após conectar na instância (resolvido no desafio anterior), verifiquei se o serviço web estava rodando.

2. Notei que o servidor Apache não estava iniciado.

<img width="1917" height="961" alt="image" src="https://github.com/user-attachments/assets/3dd99e60-e311-440c-9f07-2466e2c371c5" />

3. No terminal da instância, iniciei o servidor web manualmente:

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
```

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/a5a46c58-ab94-41b7-ab0b-8605933dbbc5" />

4. Atualizei o navegador e a página padrão do Apache carregou, confirmando a correção.

<img width="1917" height="1007" alt="image" src="https://github.com/user-attachments/assets/54400ae5-efe9-4d8e-8503-49a28d17b8d0" />
