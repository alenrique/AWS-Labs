# Challenge Lab - AWS Lambda Word Count

Neste desafio, criei uma solução serverless utilizando AWS Lambda para processar arquivos de texto automaticamente. O objetivo foi configurar uma função que, ao detectar o upload de um arquivo de texto (```.txt```) em um bucket S3, conta o número de palavras contidas nele e envia o resultado por e-mail através do Amazon SNS.

A arquitetura utiliza o S3 como gatilho (trigger) para invocar a função Lambda.

## Passo 1: Criar o Tópico SNS

Para receber as notificações por e-mail, primeiro configurei o serviço de mensageria.

1. No console do **Amazon SNS**, criei um Tópico (Topic) do tipo **Standard** chamado `WordCountTopic`.

2. Criei uma **Assinatura (Subscription)** para este tópico:

    * **Protocol**: Email

    * **Endpoint**: Meu endereço de e-mail pessoal.

3. Acessei minha caixa de entrada e cliquei no link de confirmação da AWS para validar a assinatura.

4. Copiei o **ARN** do tópico para usar no código da Lambda.

<img width="1920" height="965" alt="image" src="https://github.com/user-attachments/assets/9f6871c2-f237-42ce-bdd9-46f2933ae609" />

## Passo 2: Criar o Bucket S3

1. Criei o bucket que armazenará os arquivos de texto.

2, No console do **S3**, criei um bucket com nome único (ex: `lab-lambda-wordcount-seu-nome`).

Mantive as configurações padrão de região e bloqueio de acesso público.

<img width="1920" height="965" alt="image" src="https://github.com/user-attachments/assets/2d33d232-d4ae-4b49-850a-4719ef4361f2" />

## Passo 3: Criar a Função Lambda

Esta é a parte central do desafio. Criei a função que processa a lógica.

1. No console **Lambda**, cliquei em **Create function**.

2. **Name**: `WordCountFunction`.

3. **Runtime**: `Python 3.9` (ou versão mais recente compatível).

4. **Permissions**: Selecionei Use an existing role e escolhi a role `LambdaAccessRole` (conforme instruído no laboratório para garantir permissões ao S3, SNS e CloudWatch).

**O Código Python**

No editor de código da Lambda, inseri o script abaixo. Ele captura o evento do S3, lê o arquivo, conta as palavras e publica no SNS.

**Nota**: Substituí a variável `TOPIC_ARN` pelo ARN que copiei no Passo 1.

```python
import boto3
import urllib.parse

s3 = boto3.client('s3')
sns = boto3.client('sns')

def lambda_handler(event, context):
    # ARN do Tópico SNS criado anteriormente
    TOPIC_ARN = 'arn:aws:sns:us-west-2:123456789012:WordCountTopic'

    # Obter bucket e nome do arquivo (key) do evento
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')

    try:
        # Ler o objeto do S3
        response = s3.get_object(Bucket=bucket, Key=key)
        content = response['Body'].read().decode('utf-8')

        # Contar as palavras
        word_count = len(content.split())

        # Formatar a mensagem
        message = f"The word count in the {key} file is {word_count}."
        subject = "Word Count Result"

        # Enviar notificação via SNS
        sns.publish(
            TopicArn=TOPIC_ARN,
            Message=message,
            Subject=subject
        )

        return word_count

    except Exception as e:
        print(e)
        raise e
```

Após colar o código, cliquei em Deploy.

<img width="1920" height="965" alt="image" src="https://github.com/user-attachments/assets/a8fad133-497e-40dc-8c8b-71b87030fc58" />

## Passo 4: Configurar o Gatilho (Trigger) S3

Para conectar o bucket à função:

1. Na visualização da função Lambda, cliquei em **Add trigger**.

2. Selecionei **S3**.

3. **Bucket**: Escolhi o bucket criado no Passo 2.

4. **Event types**: `PUT` (ou All object create events).

5. **Suffix**: `.txt` (para processar apenas arquivos de texto).

6. Confirmei a criação.

<img width="1920" height="965" alt="image" src="https://github.com/user-attachments/assets/5502bf98-2731-435c-8aa4-cbbea1a9634a" />

## Passo 5: Teste da Solução

Para validar o funcionamento:

1. Criei um arquivo local chamado `teste.txt` com o conteúdo: _"Hello world this is a test from AWS Lambda"_. (8 palavras).

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/c45f1820-af22-4e2b-ac4f-33790cb24345" />

2. Fiz o upload deste arquivo para o bucket S3.

<img width="1920" height="965" alt="image" src="https://github.com/user-attachments/assets/21886e85-07fa-40bb-9a83-89f335317960" />

3. Verifiquei minha caixa de e-mail.

Resultado:
Recebi um e-mail com o assunto **Word Count Result** e o corpo:
`The word count in the teste.txt file is 9`.

Isso confirma que o upload acionou a Lambda, que leu o arquivo corretamente e enviou a notificação via SNS.

<img width="1920" height="965" alt="image" src="https://github.com/user-attachments/assets/08e3d4da-fc5b-4100-9e3d-998e33d0f139" />
