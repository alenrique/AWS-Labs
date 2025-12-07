# Challenge Lab - Amazon S3

Neste desafio, o objetivo foi praticar operações fundamentais no Amazon S3 utilizando tanto a AWS CLI quanto o Console de Gerenciamento. As tarefas incluíram criar um bucket, fazer upload de objetos, configurar permissões de acesso público e listar o conteúdo via linha de comando.

## Tarefa 1 e 2: Conexão e Configuração da CLI

Para interagir com os serviços da AWS via linha de comando, conectei-me à instância **CLI Host** utilizando o **EC2 Instance Connect**.

No terminal, configurei minhas credenciais:

```bash
aws configure
```

* **Region**: `us-west-2`

* **Format**: `json`

## Tarefa 3: Executando o Desafio

### 1. Criar um Bucket S3

Utilizei a CLI para criar um bucket com um nome único globalmente.

```bash
aws s3api create-bucket --bucket desafio-s3-henrique-alencar --region us-west-2 --create-bucket-configuration LocationConstraint=us-west-2
```

<img width="1919" height="956" alt="image" src="https://github.com/user-attachments/assets/155faf25-bb55-4527-9435-eb82993a3202" />

### 2. Fazer Upload de um Objeto

Criei um arquivo de teste simples localmente e fiz o upload para o bucket.

```bash
echo "Este é um arquivo de teste para o desafio S3" > teste.txt
aws s3 cp teste.txt s3://desafio-s3-henrique-alencar/
```

<img width="1919" height="956" alt="image" src="https://github.com/user-attachments/assets/3004f435-3f81-4070-aabf-716e26c18b2c" />

### 3. Tentar acessar via Navegador (Falha Esperada)

No Console AWS, naveguei até o bucket, cliquei no arquivo `teste.txt` e copiei a **Object URL**.
Ao tentar acessar essa URL no navegador, recebi um erro **403 Forbidden** (Access Denied). Isso ocorreu porque, por padrão, todos os buckets e objetos são privados.

<img width="1919" height="1005" alt="image" src="https://github.com/user-attachments/assets/ddd212bb-79de-44be-8f6f-54e245547bb1" />

### 4. Tornar o Objeto Público

Para permitir o acesso, precisei realizar duas etapas:

1. **Desbloquear Acesso Público no Bucket**: No Console (aba Permissions do bucket), editei as configurações de "Block public access" e desmarquei a opção **Block all public access**, salvando a alteração.

<img width="1919" height="964" alt="image" src="https://github.com/user-attachments/assets/8a175ce9-ba93-4f86-89f7-13fe94694d29" />

2. **Definir a ACL do Objeto**: Via CLI, alterei a permissão do objeto específico para ser legível publicamente:

```bash
aws s3api put-object-acl --bucket desafio-s3-henrique-alencar --key teste.txt --acl public-read
```

<img width="1919" height="964" alt="image" src="https://github.com/user-attachments/assets/ef16bd76-bd5b-4dd3-aadb-2cd4363cc215" />

### 5. Acessar via Navegador (Sucesso)

Atualizei a página com a Object URL no navegador. Desta vez, o arquivo abriu corretamente, exibindo o texto que criei.

<img width="1919" height="1005" alt="image" src="https://github.com/user-attachments/assets/967e9296-c231-4b1d-a674-988b0c01afcc" />

### 6. Listar conteúdo via CLI

Para finalizar, listei o conteúdo do bucket via terminal para confirmar que o arquivo estava lá.

```bash
aws s3 ls s3://desafio-s3-henrique-alencar
```

O comando retornou o nome do arquivo `teste.txt`, concluindo o desafio.

<img width="1919" height="960" alt="image" src="https://github.com/user-attachments/assets/c80c6387-be54-47b4-a360-a769f8b0c347" />
