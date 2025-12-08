# Lab - Trabalhando com AWS CloudTrail

Neste laboratório, assumi o papel de um detetive de segurança. O objetivo foi configurar uma trilha de auditoria (Trail) usando o **AWS CloudTrail**, investigar um incidente de segurança onde o site "Café" foi hackeado e, finalmente, identificar o culpado e remediar as vulnerabilidades.

A arquitetura envolve um servidor web EC2 e o uso de ferramentas de análise como AWS CLI, Grep e Amazon Athena para investigar os logs.

<img width="814" height="437" alt="image" src="https://github.com/user-attachments/assets/c0f7c6d5-4193-43af-aa4d-c42eea5930a2" />

## Tarefa 1: Modificar o Security Group e observar o site

Inicialmente, acessei a instância `Café Web Server` e modifiquei seu Security Group para permitir acesso SSH (porta 22) apenas para o meu IP (`My IP`), seguindo as boas práticas de segurança.
Acessei o site e ele estava normal.

<img width="1919" height="960" alt="image" src="https://github.com/user-attachments/assets/9bfa2879-76b2-4000-b6ea-f43e0f3076ff" />

## Tarefa 2: Criar uma trilha do CloudTrail e observar o ataque

Para monitorar a conta, criei uma trilha no CloudTrail:

* **Nome**: `monitor`

* **Armazenamento**: Novo bucket S3 (`monitoring####`).

<img width="1919" height="960" alt="image" src="https://github.com/user-attachments/assets/5adf6d71-b94c-4333-a796-bc504ed908f9" />

Pouco tempo depois, atualizei a página do Café e notei que o **site havia sido hackeado** (a imagem principal foi alterada).
Ao verificar novamente o Security Group da instância, percebi que uma nova regra havia sido adicionada misteriosamente, permitindo acesso SSH (22) para `0.0.0.0/0` (todo o mundo).

<img width="1920" height="1007" alt="image" src="https://github.com/user-attachments/assets/1a8e51a3-6718-44bd-9413-5e1655a90a2f" />

## Tarefa 3: Analisar logs com Grep e AWS CLI

Para investigar, conectei-me via SSH ao servidor web e baixei os logs do CloudTrail armazenados no S3:

```bash
mkdir ctraillogs
cd ctraillogs
aws s3 cp s3://[NOME-DO-BUCKET-MONITORING]/ . --recursive
gunzip *.gz
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/6ca9f586-974c-4c91-a7a4-0a059ca50944" />

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/f66634a5-f463-4d62-85f1-0104b6d6a353" />

Utilizei comandos `grep` e `jq` (ou python json.tool) para filtrar os logs brutos em busca de eventos vindos do IP do servidor web ou relacionados a alterações de segurança.

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/c1f5e30a-490e-4acf-a325-9b7d59e6d48f" />

Também utilizei a AWS CLI para filtrar eventos específicos de alteração de Security Group:

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::EC2::SecurityGroup --output text
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/90a08058-f7ad-49c7-8a8b-f2cd02207952" />

Esses métodos mostraram que havia atividade suspeita, mas a análise manual de arquivos JSON brutos é trabalhosa.

## Tarefa 4: Analisar logs com Amazon Athena

Para uma análise mais eficiente, utilizei o **Amazon Athena** para consultar os logs usando SQL.

1. **Criar Tabela**: No console do CloudTrail, usei a função "Create Athena table" apontando para o bucket `monitoring####`.

2. **Consultar**: No console do Athena, executei queries para filtrar as ações.

**O Desafio: Identificar o Hacker**
Executei uma query para listar usuários que realizaram ações de segurança (como `AuthorizeSecurityGroupIngress`) recentemente:

```sql
SELECT useridentity.userName, eventtime, eventsource, eventname, requestparameters
FROM cloudtrail_logs_monitoring####
WHERE eventname = 'AuthorizeSecurityGroupIngress'
```

<img width="1920" height="967" alt="image" src="https://github.com/user-attachments/assets/b46d104d-d12e-4fa0-a790-878f2e54a662" />

<img width="1920" height="967" alt="image" src="https://github.com/user-attachments/assets/b5b363c6-7d64-4c0c-9862-11aff4bc296e" />

<img width="1920" height="967" alt="image" src="https://github.com/user-attachments/assets/e52cd34e-09ed-4f89-86e4-6c79cd1ff00b" />

**Resultado da Investigação**:
Identifiquei que o usuário IAM **chaos** foi o responsável por abrir a porta 22 para o mundo. Descobri também o IP de origem e o horário exato do ataque.

<img width="1920" height="967" alt="image" src="https://github.com/user-attachments/assets/15c1ba04-9dd3-4d27-b2ce-88a7eacdd6a5" />

## Tarefa 5: Analisar o ataque e melhorar a segurança

Com o culpado identificado, passei para a remediação do servidor e da conta AWS.

### 5.1 Verificar usuários do SO

No terminal do servidor, verifiquei usuários logados:

```bash
who
```

Havia um usuário estranho chamado `chaos-user` logado.
Derrubei a conexão dele e deletei o usuário do sistema:

```bash
sudo kill -9 [PID]
sudo userdel -r chaos-user
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/f68ba267-ca86-4471-951b-1e18b2c39d49" />

### 5.2 Atualizar segurança SSH

Descobri que o hacker conseguiu entrar porque a autenticação por senha estava habilitada (má prática).
Editei o arquivo `/etc/ssh/sshd_config`:

* Mudei `PasswordAuthentication` para `no`.
Reiniciei o serviço SSH.

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/536ff9f0-c140-4ef5-980a-c1c740a0dc52" />

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/d1e3ec3a-6d98-4d12-8ded-faf3e7690796" />

Além disso, fui no Security Group e **removi** a regra que permitia SSH para `0.0.0.0/0`.

<img width="1920" height="967" alt="image" src="https://github.com/user-attachments/assets/b7a5f614-1efd-4d32-bf59-018a94e40209" />

### 5.3 Corrigir o site

Restaurei a imagem original do site que o hacker havia alterado:

```bash
cd /var/www/html/cafe/images/
sudo mv Coffee-and-Pastries.backup Coffee-and-Pastries.jpg
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/e85dc8e7-860e-4029-9402-60567671ca2a" />

<img width="1920" height="1007" alt="image" src="https://github.com/user-attachments/assets/d0f3d41d-b329-4788-ab65-c63a6e37365f" />

### 5.4 Deletar o usuário Hacker na AWS

Finalmente, fui ao console IAM e deletei o usuário chaos, garantindo que ele não tenha mais acesso à conta AWS.

<img width="1920" height="967" alt="image" src="https://github.com/user-attachments/assets/2b43ade9-37db-409e-8ad7-1177479ce95e" />

<img width="1920" height="967" alt="image" src="https://github.com/user-attachments/assets/d9d4294f-a0b9-4bb0-b33e-bbd14aa80dde" />
