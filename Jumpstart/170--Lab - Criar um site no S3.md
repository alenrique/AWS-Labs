# Lab - Criando um Website no S3

Nesse laboratório irei praticar o uso da AWS Command Line Interface (AWS CLI) a partir de uma instância Amazon EC2 para criar um bucket S3, configurar um usuário IAM, hospedar um site estático simples e criar um script de automação para atualizações.

O objetivo final é ter um site acessível publicamente através de uma URL do S3.

<img width="1654" height="787" alt="image" src="https://github.com/user-attachments/assets/424ada51-c806-4c5d-8f58-29f0ec134411" />

## Tarefa 1: Conectar-se a uma instância Amazon Linux EC2 usando SSM

Acessei a instância EC2 através do Session Manager (SSM). No terminal do navegador, executei os comandos para mudar para o usuário *ec2-user* e verificar o diretório atual.

```bash
sudo su -l ec2-user
pwd
```

<img width="1917" height="962" alt="image" src="https://github.com/user-attachments/assets/e2e1ff22-2d9b-4d64-9567-fc19ef37c9a4" />

## Tarefa 2: Configurar a AWS CLI

Como a instância Amazon Linux já vem com a CLI instalada, precisei apenas configurá-la com as credenciais do laboratório.

Executei o comando:

```bash
aws configure
```

Inseri as credenciais (AWS Access Key ID e AWS Secret Access Key) fornecidas no laboratório e configurei a região e o formato de saída:

* Default region name: ```us-west-2```

* Default output format: ```json```

<img width="1917" height="962" alt="image" src="https://github.com/user-attachments/assets/d32c44d7-2464-4957-837f-11647598205f" />

## Tarefa 3: Criar um bucket S3 usando a AWS CLI

Utilizei o comando ```aws s3api``` para criar um novo bucket. Escolhi um nome único (ex: iniciais-sobrenome-numeros) para o bucket.

Comando utilizado:

```bash
aws s3api create-bucket --bucket [nome-do-bucket] --region us-west-2 --create-bucket-configuration LocationConstraint=us-west-2
```

O retorno foi um JSON confirmando a localização do bucket criado.

<img width="1917" height="962" alt="image" src="https://github.com/user-attachments/assets/77c2aead-dd4a-4af7-b614-15fa88b111bd" />

## Tarefa 4: Criar um novo usuário IAM com acesso total ao Amazon S3

Criei um novo usuário no IAM chamado *awsS3user* e configurei seu perfil de login.

* Para criar o usuário:

```bash
aws iam create-user --user-name awsS3user
```

* Para criar a senha de login:

```bash
aws iam create-login-profile --user-name awsS3user --password Training123!
```

Em seguida, precisei dar permissões para esse usuário. Busquei a política de acesso total ao S3 (AmazonS3FullAccess) usando o comando de busca:

```bash
aws iam list-policies --query "Policies[?contains(PolicyName,'S3')]"
```

Copiei o ARN da política encontrada e anexei ao usuário:

```bash
aws iam attach-user-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --user-name awsS3user
```

Validei o acesso fazendo login no Console da AWS com esse novo usuário em uma aba anônima.

<img width="1917" height="962" alt="image" src="https://github.com/user-attachments/assets/f11fa0ec-1476-482c-a866-6e2e4de868c1" />

## Tarefa 5: Ajustar as permissões do bucket S3

No Console da AWS (logado com o usuário principal ou o novo usuário, caso já tenha permissão), fui até o bucket criado para ajustar as configurações de acesso público, permitindo que o site seja visível.

1. Na aba **Permissions**, em **Block public access (bucket settings)**, cliquei em **Edit** e desmarquei a opção **Block all public access**. Salvei as alterações.

2. Ainda em **Permissions**, em **Object Ownership**, habilitei as **ACLs (ACLs enabled)** e confirmei a ciência da alteração.

<img width="1917" height="962" alt="Captura de tela de 2025-12-05 09-06-16" src="https://github.com/user-attachments/assets/58427e1d-a982-4e59-9955-494109af6397" />

<img width="1917" height="962" alt="Captura de tela de 2025-12-05 09-06-50" src="https://github.com/user-attachments/assets/83007fe3-4faf-4fd1-af03-bec8ca0669ee" />

## Tarefa 6: Extrair os arquivos necessários para este laboratório

De volta ao terminal SSH, extraí os arquivos do site estático que estavam compactados.

```bash
cd ~/sysops-activity-files
tar xvzf static-website-v2.tar.gz
cd static-website
ls
```

Isso gerou o arquivo index.html e as pastas css e images.

<img width="1917" height="962" alt="image" src="https://github.com/user-attachments/assets/26ec6da8-c178-49fa-8df6-628361d9854e" />

## Tarefa 7: Carregar arquivos para o Amazon S3 usando a AWS CLI

Configurei o bucket para funcionar como um site estático:

```bash
aws s3 website s3://[nome-do-bucket]/ --index-document index.html
```

Em seguida, fiz o upload dos arquivos locais para o bucket, definindo a permissão como pública (--acl public-read) e usando recursividade para pegar as subpastas:

```bash
aws s3 cp /home/ec2-user/sysops-activity-files/static-website/ s3://[nome-do-bucket]/ --recursive --acl public-read
```

Para validar, listei os arquivos no bucket:

```bash
aws s3 ls [nome-do-bucket]
```

<img width="1917" height="962" alt="image" src="https://github.com/user-attachments/assets/7f8da380-a933-4566-97f2-698ffebe6b73" />

Acessei a URL do endpoint do site (disponível na aba Properties do bucket no console) e confirmei que o site estava no ar.

<img width="1917" height="1008" alt="image" src="https://github.com/user-attachments/assets/cbebb23e-ff62-4c39-8fac-bcfe14fb88b4" />

## Tarefa 8: Criar um arquivo em lote (batch) para tornar a atualização do site repetível

Para facilitar futuras atualizações, criei um script shell.

1. Criei o arquivo update-website.sh:

```bash
cd ~
touch update-website.sh
```

2. Editei o arquivo com o vi e adicionei o comando de upload que usei anteriormente:

```bash
#!/bin/bash
aws s3 cp /home/ec2-user/sysops-activity-files/static-website/ s3://[nome-do-bucket]/ --recursive --acl public-read
```

3. Dei permissão de execução ao script:

```bash
chmod +x update-website.sh
```

4. Fiz uma alteração no arquivo ```index.html``` localmente (mudando as cores de fundo) para testar.

5. Executei o script para atualizar o site automaticamente:

```bash
./update-website.sh
```

<img width="1917" height="965" alt="Captura de tela de 2025-12-05 09-18-28" src="https://github.com/user-attachments/assets/ba1ca5e1-4fc0-4fbb-b6b5-258f4c1df320" />

<img width="1917" height="965" alt="Captura de tela de 2025-12-05 09-23-28" src="https://github.com/user-attachments/assets/95a713d6-2758-49b8-8135-112694a85712" />

<img width="1917" height="965" alt="Captura de tela de 2025-12-05 09-25-26" src="https://github.com/user-attachments/assets/937a83a2-7612-45cc-8c06-5b53a9af0c5d" />

<img width="1917" height="965" alt="Captura de tela de 2025-12-05 09-25-57" src="https://github.com/user-attachments/assets/2845e2a9-9a81-4617-8a09-7fb36f67aa0d" />


Ao recarregar a página no navegador, as cores haviam mudado, confirmando que o script funcionou.

<img width="1917" height="1009" alt="Captura de tela de 2025-12-05 09-26-12" src="https://github.com/user-attachments/assets/2eb54183-8712-475f-ac0c-d741b8190df7" />

## Desafio Opcional: Otimização com Sync

Notei que o comando cp copia todos os arquivos novamente, mesmo os que não mudaram. Para otimizar, testei o comando ```sync```, que atualiza apenas arquivos modificados:

```bash
aws s3 sync /home/ec2-user/sysops-activity-files/static-website/ s3://[nome-do-bucket]/ --acl public-read
```

<img width="1917" height="962" alt="Captura de tela de 2025-12-05 09-31-49" src="https://github.com/user-attachments/assets/ca8f5ca7-2de0-4817-8fad-a1c0928a0325" />

<img width="1917" height="962" alt="Captura de tela de 2025-12-05 09-31-40" src="https://github.com/user-attachments/assets/a9bcb0bf-2f9b-4f80-b766-bd3ee11e5149" />

<img width="1917" height="1007" alt="Captura de tela de 2025-12-05 09-32-00" src="https://github.com/user-attachments/assets/fd5152e7-8203-4fe9-85b8-9dda5896f71b" />
