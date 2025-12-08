# Challenge Lab - CloudFormation: Criando VPC e EC2

Neste desafio, tive a tarefa de criar uma infraestrutura completa utilizando apenas um template do **AWS CloudFormation**. O objetivo era definir e provisionar uma VPC, um Internet Gateway, um Security Group, uma Subnet Privada e uma instância EC2, tudo via código (Infrastructure as Code - IaC).

A arquitetura solicitada consistia nos seguintes componentes:

* Uma Amazon VPC.

* Um Internet Gateway anexado à VPC.

* Um Security Group permitindo SSH de qualquer lugar.

* Uma Subnet Privada.

* Uma instância EC2 (t3.micro) dentro dessa subnet privada.

## Passo 1: Construindo o Template CloudFormation

Para resolver o desafio, escrevi um arquivo YAML definindo todos os recursos necessários. Utilizei parâmetros do **Systems Manager (SSM)** para buscar automaticamente a AMI mais recente do Amazon Linux 2, garantindo que o template funcione em qualquer região sem precisar "hardcodar" o ID da imagem.

Aqui está o código da solução que desenvolvi (`solution.yaml`):

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Challenge Lab - VPC and EC2

Parameters:
  # Parâmetro para pegar a AMI mais recente do Amazon Linux 2 automaticamente
  LatestAmiId:
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'

Resources:
  # 1. Criação da VPC
  MyLabVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: LabVPC

  # 2. Internet Gateway (IGW)
  MyIGW:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: LabIGW

  # Anexar o IGW à VPC
  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyLabVPC
      InternetGatewayId: !Ref MyIGW

  # 3. Subnet Privada
  MyPrivateSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyLabVPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [ 0, !GetAZs '' ] # Pega a primeira AZ da região
      Tags:
        - Key: Name
          Value: PrivateSubnet

  # 4. Security Group (Permitir SSH de qualquer lugar)
  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH traffic
      VpcId: !Ref MyLabVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: SSHSecurityGroup

  # 5. Instância EC2
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: !Ref LatestAmiId
      SubnetId: !Ref MyPrivateSubnet
      SecurityGroupIds:
        - !Ref MySecurityGroup
      Tags:
        - Key: Name
          Value: LabInstance
```

<img width="929" height="540" alt="image" src="https://github.com/user-attachments/assets/0d8d8244-cbba-4edf-ae8d-ba503c00059e" />

<img width="929" height="540" alt="image" src="https://github.com/user-attachments/assets/7a458476-fe75-487b-9232-2d2810bf686d" />

## Passo 2: Implantando a Stack via Terminal

Com o arquivo YAML pronto, utilizei a AWS CLI para realizar o deploy diretamente pelo terminal, agilizando o processo e evitando o uso da interface gráfica.

Executei o seguinte comando para criar a stack:

```bash
aws cloudformation create-stack --stack-name ChallengeStack --template-body file://solution.yaml
```

O comando retornou o StackId, confirmando que a solicitação foi enviada com sucesso para a AWS.

<img width="929" height="540" alt="image" src="https://github.com/user-attachments/assets/8bfe0b2f-2ea2-4805-bed6-0876052d46d0" />

## Passo 3: Verificação e Validação

Para monitorar o progresso da criação sem precisar atualizar a página do console, utilizei o comando `wait`:

```bash
aws cloudformation wait stack-create-complete --stack-name ChallengeStack
```

Após o comando liberar o terminal (indicando que o status chegou a `CREATE_COMPLETE`), validei se a instância foi criada corretamente consultando o EC2 via CLI:

```bash
aws ec2 describe-instances --filters "Name=tag:Name,Values=LabInstance" --query "Reservations[*].Instances[*].[InstanceId,State.Name,VpcId,SubnetId]" --output table
```

O retorno confirmou que a instância `LabInstance` estava com o status **running**, associada à VPC e Subnet corretas criadas pelo template.

<img width="929" height="540" alt="image" src="https://github.com/user-attachments/assets/1af7646a-7032-4373-8df0-cdec1a06c71d" />

Vendo através do console confirmei que tudo foi criado como deveria.

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/e7de41b8-6fd4-4449-9ed3-8ca9d7a54168" />
