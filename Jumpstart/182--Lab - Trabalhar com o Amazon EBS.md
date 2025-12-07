# Lab - Trabalhando com Amazon EBS

Neste laboratório, aprendi a gerenciar volumes do Amazon Elastic Block Store (EBS). O objetivo foi criar um volume adicional, anexá-lo a uma instância EC2, formatá-lo com um sistema de arquivos e montá-lo para uso. Além disso, realizei operações de backup criando um snapshot e restaurando dados a partir dele.

A arquitetura envolve uma instância EC2 Linux e a manipulação de volumes de bloco persistentes.

<img width="1626" height="470" alt="image" src="https://github.com/user-attachments/assets/0a447728-7484-4990-b168-33ccfe2c9638" />

## Tarefa 1: Criar um novo volume EBS

Para começar, criei um volume de armazenamento independente da instância.

1. No console EC2, identifiquei a Zona de Disponibilidade (AZ) da instância `Lab` (ex: `us-west-2a`).

2. Fui na seção **Volumes** e cliquei em **Create volume**.

3. Configurei:

    * **Volume type**: General Purpose SSD (gp2).

    * **Size**: 1 GiB.

    * **Availability Zone**: A mesma da instância (`us-west-2a`).

    * **Tags**: Key=`Name`, Value=`My Volume`.

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/023d8017-cf8f-4ef2-89ec-046bc50c1b1f" />

## Tarefa 2: Anexar o volume a uma instância EC2

Com o volume criado (status `Available`), precisei conectá-lo à instância.

1. Selecionei `My Volume` e fui em **Actions > Attach volume**.

2. Escolhi a instância `Lab`.

3. Defini o **Device name** como `/dev/sdb`.

4. Confirmei a operação. O status mudou para `In-use`.

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/d0c66807-2f03-4e27-8e1b-dce083d3192b" />

## Tarefa 3: Conectar à instância Lab EC2

Utilizei o **EC2 Instance Connect** para acessar o terminal da instância Linux e configurar o novo disco.

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/b3e01e53-57ef-4ea6-a400-bd521f4572c2" />

## Tarefa 4: Criar e configurar o sistema de arquivos

Dentro do terminal, realizei os seguintes passos para tornar o disco utilizável:

1. Verifiquei os discos atuais:

```bash
df -h
```

2. Formatei o novo volume (`/dev/sdb`) com o sistema de arquivos ext3:

```bash
sudo mkfs -t ext3 /dev/sdb
```

3. Criei um ponto de montagem e montei o volume:

```bash
sudo mkdir /mnt/data-store
sudo mount /dev/sdb /mnt/data-store
```

4. Configurei o `/etc/fstab` para garantir que o volume seja montado automaticamente após reinicializações:

```bash
echo "/dev/sdb   /mnt/data-store ext3 defaults,noatime 1 2" | sudo tee -a /etc/fstab
```

5. Criei um arquivo de texto para testar a persistência dos dados:

```bash
sudo sh -c "echo some text has been written > /mnt/data-store/file.txt"
cat /mnt/data-store/file.txt
```

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/9c6e95e3-d1af-40ef-b600-6af5cb58309d" />

## Tarefa 5: Criar um Snapshot do Amazon EBS

Para simular um backup, criei um snapshot do volume contendo o arquivo de texto.

1. No console, selecionei `My Volume`.

2. Fui em **Actions > Create snapshot**.

3. Defini a Tag Name como `My Snapshot`.

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/07de2af3-26c1-48c8-b4c9-3bede40c4da6" />

Enquanto o snapshot era criado, voltei ao terminal da instância e deletei o arquivo original para simular uma perda de dados:

```bash
sudo rm /mnt/data-store/file.txt
ls /mnt/data-store/file.txt
# Resultado: No such file or directory
```

## Tarefa 6: Restaurar o Snapshot do Amazon EBS

Recuperei os dados perdidos criando um novo volume a partir do snapshot.

### 6.1 Criar volume a partir do snapshot

1. Fui na seção **Snapshots** e selecionei `My Snapshot`.

2. Cliquei em **Actions > Create volume from snapshot**.

3. Mantive a mesma AZ e defini a Tag Name como `Restored Volume`.

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/0a23eb91-278b-4da5-acad-ffb627a94ad0" />

### 6.2 Anexar o volume restaurado

1. Selecionei o `Restored Volume` e o anexei à instância `Lab`.

2. Defini o dispositivo como `/dev/sdc`.

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/3bdbd77c-b253-49c7-9edc-f2284e22c813" />

### 6.3 Montar o volume restaurado

De volta ao terminal, montei este novo disco em um diretório diferente para verificar os dados:

```bash
sudo mkdir /mnt/data-store2
sudo mount /dev/sdc /mnt/data-store2
```

Verifiquei se o arquivo deletado estava presente neste volume de backup:

```bash
ls /mnt/data-store2/file.txt
```

O arquivo foi listado com sucesso, confirmando que o snapshot capturou os dados corretamente antes da exclusão.

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/22b3ac53-2afb-47c9-a2d9-1ea11d662196" />

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/0946cc20-a53d-4774-b8fd-6d43c811345c" />
