# Lab - Trabalhando com AWS Lambda

Neste laboratório, implementei uma solução de computação serverless usando **AWS Lambda**. O objetivo foi criar um sistema automatizado que gera relatórios de análise de vendas extraindo dados de um banco de dados (hospedado em uma instância EC2) e enviando os resultados por e-mail diariamente.

A arquitetura envolve duas funções Lambda, o Systems Manager Parameter Store para credenciais e o Amazon SNS para notificações.

<img width="1588" height="850" alt="image" src="https://github.com/user-attachments/assets/76e66329-f5c8-4996-a4a5-e74d60e0a4ce" />

## Tarefa 1: Observar as configurações de IAM Role

Antes de criar as funções, analisei as Roles de IAM pré-criadas para entender as permissões:

* **salesAnalysisReportRole**: Permite acesso ao SNS, Systems Manager e invocação de outras Lambdas.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/4706df9c-8c85-4635-9e53-04dfefef3fff" />

* **salesAnalysisReportDERole**: Permite acesso básico de execução e gerenciamento de interfaces de rede (necessário para conectar a função à VPC).

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/23f457b6-7911-4c6d-88c6-2be76cba7f6a" />

## Tarefa 2: Criar uma Lambda Layer e a função extratora de dados

Para evitar duplicidade de código e gerenciar dependências, utilizei o recurso de **Lambda Layers**.

* **Lambda Layer**: Criei uma layer chamada `pymysqlLibrary` e fiz o upload do arquivo `.zip` contendo a biblioteca PyMySQL.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/a0d257d9-c922-434b-8340-fdba8d63fb51" />

* **Função Data Extractor**: Criei a função `salesAnalysisReportDataExtractor` (Runtime Python 3.9) usando a role `salesAnalysisReportDERole`.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/e231502c-777f-4da3-aa24-7f736a925547" />

* **Associação**: Adicionei a layer `pymysqlLibrary` a esta função.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/c18b7dee-fed0-4d87-8f2f-085b2eb1b5dd" />

* **Código**: Fiz o upload do código fonte (`salesAnalysisReportDataExtractor-v3.zip`) e configurei o Handler.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/44d841a5-4b26-4bfa-8c97-6bf81d221a5c" />

* **Configuração de Rede (VPC)**: Para que a função acesse o banco de dados na instância EC2, configurei a VPC, Subnet e o Security Group (`CafeSecurityGroup`).

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/fc1d0779-b555-4d99-859a-a8455940b36f" />

## Tarefa 3: Testar a função Lambda (Troubleshooting)

### 3.1 Configuração do Teste

Recuperei as credenciais do banco de dados (URL, Nome, Usuário, Senha) armazenadas no **Systems Manager Parameter Store**.
Criei um evento de teste na Lambda injetando esses parâmetros via JSON.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/e1052c53-1351-4a64-b0b3-abd0946fb48c" />

### 3.2 Erro e Solução

Ao executar o primeiro teste, recebi um erro de Timeout (a função excedeu o tempo limite de 3 segundos).

**Diagnóstico**: A função tentava conectar ao banco de dados na porta 3306, mas a conexão falhava silenciosamente até o timeout.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/9f417d2c-0d86-4fd8-9311-f351b37fdfda" />

**Solução**: Verifiquei as regras de entrada (Inbound Rules) do Security Group da instância de banco de dados e garanti que a porta **3306 (MySQL)** estava liberada para o Security Group da Lambda.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/3bc1ceda-671a-4de4-9ef6-ef1509e820af" />

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/da72684c-aea6-4014-b93c-f7cb7f9e4016" />

### 3.3 Povoando o Banco de Dados

Para validar o relatório, acessei o site do Café (hospedado na EC2) e fiz alguns pedidos para gerar dados de vendas no banco.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/bf63cf7c-12ba-47d0-a435-f5f609b8e4ad" />

### 3.4 Reteste

Executei o teste novamente e obtive sucesso (statusCode: 200), com o corpo da resposta contendo os dados dos pedidos em JSON.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/2c818025-0131-4d8b-a593-b27522363cc1" />

## Tarefa 4: Configurar notificações (SNS)

Para o envio do relatório por e-mail:

1. Criei um tópico no **Amazon SNS** chamado `salesAnalysisReportTopic`.

2. Criei uma assinatura (subscription) para o meu e-mail.

3. Confirmei a assinatura através do link recebido na caixa de entrada.

4. Copiei o ARN do tópico para usar na próxima etapa.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/6c625329-6e9f-4c90-9728-79469661b0fa" />

## Tarefa 5: Criar a função de Relatório (salesAnalysisReport)

Esta função é o "orquestrador": ela busca credenciais, invoca a função extratora (criada na Tarefa 2), formata o relatório e envia para o SNS.

### 5.1 Criação via AWS CLI

Desta vez, criei a função utilizando a linha de comando a partir da instância **CLI Host**.

1. Conectei à instância via EC2 Instance Connect.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/7463d179-ef60-4276-a52c-d5ffe8d2c4e5" />

2. Configurei a AWS CLI (`aws configure`).

3. Executei o comando para criar a função:

```bash
aws lambda create-function \
--function-name salesAnalysisReport \
--runtime python3.9 \
--zip-file fileb://salesAnalysisReport-v2.zip \
--handler salesAnalysisReport.lambda_handler \
--region us-west-2 \
--role [ARN-DA-ROLE-SALES-ANALYSIS]
```

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/29e4d1d6-d00a-4b27-8533-702bcc2eec16" />

### 5.2 Configuração de Variáveis de Ambiente

No console da Lambda, adicionei uma variável de ambiente chamada `topicARN` com o valor do ARN do tópico SNS criado na Tarefa 4. Isso permite que o código saiba para onde enviar o e-mail sem precisar hardcode.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/ceeadf01-94ed-4c85-a706-1e2305d9c185" />

### 5.3 Teste Final e Agendamento

1. Testei a função e recebi o e-mail com o relatório de vendas formatado.

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/26affe3a-9f9f-4fc9-b7c2-611c47efb916" />

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/31f78dd0-89ea-4ded-a84d-ba8f25703ac4" />

2. Adicionei um gatilho (**Trigger**) usando o **EventBridge (CloudWatch Events)**.

3. Configurei uma expressão **Cron** para agendar a execução automática do relatório (ex: `cron(35 11 ? * MON-SAT *)`).

<img width="1920" height="959" alt="image" src="https://github.com/user-attachments/assets/6da6ec79-79b6-4ccd-b788-4330890f7531" />
