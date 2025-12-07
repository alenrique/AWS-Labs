# Lab - Gerenciando Armazenamento (EBS e S3)

Neste laboratório, utilizei a AWS CLI para gerenciar volumes do Amazon EBS e buckets do Amazon S3. O objetivo foi automatizar a criação de snapshots de volumes EBS, criar scripts de limpeza para snapshots antigos e sincronizar arquivos de uma instância EC2 para um bucket S3 utilizando versionamento para recuperação de dados.

O ambiente consiste em uma VPC com duas instâncias: "Command Host" (para administração) e "Processor" (onde os dados residem).

<img width="1538" height="970" alt="image" src="https://github.com/user-attachments/assets/77440cb1-1e15-464a-b505-930e941f0f0f" />

## Tarefa 1: Criação e configuração de recursos

Iniciei criando a infraestrutura básica necessária para o armazenamento.

* **Bucket S3**: Criei um bucket S3 com nome único no console da AWS para armazenar os arquivos sincronizados posteriormente.

<img width="1920" height="957" alt="image" src="https://github.com/user-attachments/assets/303ba64f-901d-4962-a89c-0c6c31cc4e5a" />

* **IAM Role**: Modifiquei a instância EC2 chamada "Processor", anexando a role `S3BucketAccess`. Isso é crucial para permitir que a instância interaja com o S3 e o EBS sem a necessidade de chaves de acesso hardcoded.

<img width="1920" height="957" alt="image" src="https://github.com/user-attachments/assets/a4c1bbff-0119-45f0-9706-533845f80409" />

## Tarefa 2: Gerenciando Snapshots de Instância

Nesta etapa, conectei-me à instância **Command Host** via EC2 Instance Connect para gerenciar remotamente os volumes da instância "Processor".

### 2.1 Criando um Snapshot Inicial

Primeiro, precisei identificar o ID do volume e da instância "Processor".

```bash
# Obter o Volume ID
aws ec2 describe-instances --filter 'Name=tag:Name,Values=Processor' --query 'Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.{VolumeId:VolumeId}'

# Obter o Instance ID
aws ec2 describe-instances --filters 'Name=tag:Name,Values=Processor' --query 'Reservations[0].Instances[0].InstanceId'
```

Para garantir a consistência dos dados, parei a instância antes de criar o snapshot:

```bash
aws ec2 stop-instances --instance-ids [INSTANCE-ID]
aws ec2 wait instance-stopped --instance-id [INSTANCE-ID]
```

<img width="1920" height="957" alt="image" src="https://github.com/user-attachments/assets/b7ae0b98-94a7-48ae-a516-ffb98c14cca0" />

Criei o snapshot e reiniciei a instância:

```bash
aws ec2 create-snapshot --volume-id [VOLUME-ID]
aws ec2 start-instances --instance-ids [INSTANCE-ID]
```

<img width="1920" height="957" alt="image" src="https://github.com/user-attachments/assets/fc47b2b3-3497-4094-bd46-b17cb74e29f2" />

### 2.2 Agendando Snapshots Automáticos (Cron)

Para automatizar o backup, criei uma tarefa no Cron do Linux para tirar um snapshot a cada minuto:

```bash
echo "* * * * * aws ec2 create-snapshot --volume-id [VOLUME-ID] 2>&1 >> /tmp/cronlog" > cronjob
crontab cronjob
```

Após alguns minutos, verifiquei que vários snapshots foram criados automaticamente:

```bash
aws ec2 describe-snapshots --filters "Name=volume-id,Values=[VOLUME-ID]"
```

<img width="1920" height="957" alt="image" src="https://github.com/user-attachments/assets/7aa68aec-90a1-492d-95c0-d919447a1632" />

### 2.3 Retendo apenas os últimos snapshots

Para economizar custos e armazenamento, utilizei um script Python (`snapshotter_v2.py`) que identifica todos os snapshots do volume e deleta os mais antigos, mantendo apenas os dois mais recentes.

Parei o cron job (`crontab -r`) e executei o script:

```bash
python3.8 snapshotter_v2.py
```

O output confirmou a deleção dos snapshots excedentes.

<img width="1920" height="957" alt="image" src="https://github.com/user-attachments/assets/168e54c8-a743-4fe5-a115-8a1d13e558ab" />

## Tarefa 3 (Desafio): Sincronizar arquivos com Amazon S3

Neste desafio, conectei-me à instância Processor para configurar a sincronização de arquivos e testar a recuperação de desastres usando versionamento do S3.

### 3.1 Preparação e Versionamento

Baixei os arquivos de exemplo e habilitei o versionamento no bucket S3. O versionamento é essencial para recuperar arquivos deletados ou sobrescritos.

```bash
wget [https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-100-RSJAWS-3-124627/183-lab-JAWS-managing-storage/s3/files.zip](https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-100-RSJAWS-3-124627/183-lab-JAWS-managing-storage/s3/files.zip)
unzip files.zip

# Habilitar Versionamento
aws s3api put-bucket-versioning --bucket [NOME-DO-BUCKET] --versioning-configuration Status=Enabled
```

### 3.2 Sync e Recuperação de Arquivos

Sincronizei a pasta local com o bucket S3:

```bash
aws s3 sync files s3://[NOME-DO-BUCKET]/files/
```

<img width="1920" height="957" alt="image" src="https://github.com/user-attachments/assets/494a3bc6-0969-49b1-b052-bf8660fcc496" />

Simulei um erro humano deletando um arquivo localmente e propagando a deleção para o S3:

```bash
rm files/file1.txt
aws s3 sync files s3://[NOME-DO-BUCKET]/files/ --delete
```

Para recuperar o arquivo, listei as versões disponíveis do objeto deletado:

```bash
aws s3api list-object-versions --bucket [NOME-DO-BUCKET] --prefix files/file1.txt
```

<img width="1920" height="957" alt="image" src="https://github.com/user-attachments/assets/4e93fe60-7568-4026-afd4-c2551e28378f" />

Copiei o `VersionId` da versão anterior à deleção e restaurei o arquivo:

```bash
aws s3api get-object --bucket [NOME-DO-BUCKET] --key files/file1.txt --version-id [VERSION-ID] files/file1.txt
```

<img width="1920" height="957" alt="image" src="https://github.com/user-attachments/assets/2ff15cb0-47a3-465b-bfef-7d54cad152f6" />

Validei que o arquivo `file1.txt` foi restaurado com sucesso no diretório local.

<img width="1920" height="957" alt="image" src="https://github.com/user-attachments/assets/39f72885-ad90-4d42-b49d-9f71d713b80b" />
