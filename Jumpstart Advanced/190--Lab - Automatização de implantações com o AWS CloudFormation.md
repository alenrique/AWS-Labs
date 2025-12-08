# Lab - Automação com CloudFormation

Neste laboratório, explorei o conceito de Infraestrutura como Código (IaC) utilizando o **AWS CloudFormation**. O objetivo foi implantar, atualizar e encerrar uma infraestrutura completa de forma automatizada e consistente, evitando configurações manuais propensas a erros.

Comecei com um template básico de rede e fui evoluindo a infraestrutura incrementalmente, adicionando armazenamento (S3) e computação (EC2).

<img width="2391" height="1142" alt="image" src="https://github.com/user-attachments/assets/bb71ca90-6ae4-4614-b8ae-9dc442e67697" />

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/9aed2a73-552f-4c33-ac4e-5318d2dcf508" />

## Tarefa 1: Implantar uma Stack CloudFormation

Comecei implantando a base da infraestrutura de rede.

1. Baixei o template `task1.yaml` e analisei sua estrutura (Parâmetros, Recursos e Outputs). Ele definia uma VPC e um Security Group.

2. No console do **CloudFormation**, criei uma nova Stack chamada `Lab`.

3. Fiz o upload do arquivo `task1.yaml`.

4. Mantive os parâmetros de CIDR padrão e iniciei a criação.

5. Acompanhei a aba Events até que o status mudasse para `CREATE_COMPLETE`.

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/f5e657f9-f590-4b56-a465-ce3458b4f095" />

## Tarefa 2: Adicionar um Bucket Amazon S3 à Stack

Nesta etapa, editei o template YAML localmente para adicionar um novo recurso.

1. Abri o arquivo `task1.yaml` no editor de texto.

2. Adicionei o seguinte bloco na seção `Resources`:

```yaml
  MyBucket:
    Type: AWS::S3::Bucket
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/06dfa652-0968-4705-a84a-78df3d1fb4cc" />

3. No console do CloudFormation, selecionei a stack `Lab` e cliquei em Update.

4. Selecionei "Replace current template" e fiz upload do arquivo modificado.

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/2be45b6d-0996-4bf5-b2d0-95c5c4ac768d" />

5. O CloudFormation detectou a mudança e mostrou que um recurso seria adicionado. Confirmei a atualização.

6. Após o status `UPDATE_COMPLETE`, verifiquei na aba Resources que o bucket S3 foi criado com sucesso.

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/f993b314-c486-47c7-a21f-2ce0d1444efc" />

## Tarefa 3: Adicionar uma Instância Amazon EC2 à Stack

Para finalizar a infraestrutura, adicionei um servidor web. Esta etapa foi mais complexa pois exigiu o uso de parâmetros dinâmicos e referências cruzadas (`!Ref`).

1. **Parâmetro de AMI**: Adicionei um parâmetro para buscar automaticamente a AMI mais recente do Amazon Linux 2 usando o Systems Manager Parameter Store:

```yaml
  AmazonLinuxAMIID:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2
```

2. **Recurso EC2**: Adicionei a definição da instância, referenciando o Security Group e a Subnet que já existiam no template:

```yaml
  AppInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref AmazonLinuxAMIID
      InstanceType: t3.micro
      SecurityGroupIds:
        - !Ref AppSecurityGroup
      SubnetId: !Ref PublicSubnet
      Tags:
        - Key: Name
          Value: App Server
```

3. Atualizei a stack novamente com o novo template.

4. Após a conclusão, fui ao console EC2 e confirmei que a instância `App Server` estava rodando, conectada à VPC e ao Security Group corretos.

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/e5663794-4de4-4b3f-9c3e-24d457d593f2" />

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/993c60d2-0511-4038-a5c3-e7d1b2fc5a40" />

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/389fc06d-7b80-4e89-8c6c-fe3db8ad8d0f" />

## Tarefa 4: Deletar a Stack

Para limpar o laboratório, utilizei o poder do CloudFormation para destruir tudo de uma vez.

1. Selecionei a stack `Lab` e cliquei em Delete.

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/dec88d6d-ca3e-4562-9611-90e29dfe0bd5" />

2. O status mudou para `DELETE_IN_PROGRESS`.

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/af964822-6b9a-4fc6-bc69-befd3bfee95a" />

3. O CloudFormation removeu automaticamente a Instância EC2, o Bucket S3, o Security Group e a VPC, garantindo que não restassem recursos órfãos gerando custos.

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/5c448ced-5517-44d9-8085-b9549d834c5d" />
