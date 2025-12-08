# Activity - Troubleshoot CloudFormation

Nesta atividade, pratiquei a resolução de problemas (troubleshooting) em implantações do AWS CloudFormation. O cenário envolveu tentar implantar uma stack que falha propositalmente, investigar logs em uma instância EC2 para encontrar a causa raiz, corrigir o template, detectar desvios de configuração (Drift) e resolver problemas de deleção de stack.

A arquitetura utilizada envolve um CLI Host para administração e a tentativa de criação de um Web Server e recursos de rede via código.

<img width="921" height="410" alt="image" src="https://github.com/user-attachments/assets/e0d3f31b-1fa1-46e6-bb23-67943e1b0b75" />

## Tarefa 1: Praticar consultas JMESPath

Antes de iniciar a implantação, pratiquei a sintaxe JMESPath para filtrar saídas JSON. Isso é fundamental para usar a AWS CLI de forma eficiente. Testei consultas para filtrar elementos específicos em arrays e objetos JSON.

Exemplo de consulta para pegar o ID Lógico de uma instância:

```bash
StackResources[?ResourceType == 'AWS::EC2::Instance'].LogicalResourceId
```

## Tarefa 2: Troubleshooting e criação da Stack

Conectei-me ao CLI Host via SSH e configurei a AWS CLI.

### 2.1 Primeira tentativa (Falha e Rollback)

Tentei criar a stack `myStack` usando o arquivo `template1.yaml`:

```bash
aws cloudformation create-stack --stack-name myStack --template-body file://template1.yaml --capabilities CAPABILITY_NAMED_IAM --parameters ParameterKey=KeyName,ParameterValue=vockey
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/937b0419-8cad-4f01-85c9-00906bbb3368" />

Monitorei a criação e notei que o recurso `WaitCondition` falhou por timeout, causando o **ROLLBACK_COMPLETE** da stack. Como o rollback deleta os recursos, não consegui investigar a instância criada.

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/8213544d-8a86-42e7-b0c7-7f4e6fddb6f7" />

### 2.2 Segunda tentativa (Sem Rollback)

Para investigar, recriei a stack adicionando o parâmetro `--on-failure DO_NOTHING`. Isso manteve a instância rodando mesmo após o erro.

<img width="878" height="663" alt="image" src="https://github.com/user-attachments/assets/4600f837-e98f-4eee-b79d-fd3edf211202" />

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/671213f3-2f1c-4d63-9759-4cf72bcbc6a2" />

### 2.3 Investigação da Causa Raiz

Conectei-me via SSH na instância **Web Server** (que ficou em estado de falha) e analisei os logs de inicialização (`cloud-init`):

```bash
tail -50 /var/log/cloud-init-output.log
```

<img width="1920" height="1052" alt="image" src="https://github.com/user-attachments/assets/3f25a3bf-5d93-4a39-89f4-93dd70808d03" />

Encontrei o erro: `No package http available`.
Analisei o script de User Data em `/var/lib/cloud/instance/scripts/part-001` e percebi que o comando tentava instalar `http` em vez de `httpd` (o nome correto do pacote Apache no Amazon Linux).

### 2.4 Correção e Sucesso

Voltei ao CLI Host, editei o arquivo `template1.yaml` (linha 128) corrigindo `http` para `httpd`.
Deletei a stack falha e criei novamente. Desta vez, o status foi **CREATE_COMPLETE**.

<img width="878" height="663" alt="image" src="https://github.com/user-attachments/assets/acf0c836-84e0-4ee9-9be0-b8686b45413e" />

<img width="878" height="663" alt="image" src="https://github.com/user-attachments/assets/19bbb0b5-eb3e-4ab2-bd13-b55e7a4de3e5" />

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/bb20a0db-1813-4a5d-9b93-6836a60d38b0" />

## Tarefa 3: Detectar Drift (Desvios)

Para testar a detecção de alterações manuais:

1. **Alteração Manual**: Fui ao Console EC2 e modifiquei o Security Group do Web Server, alterando a regra de SSH para permitir apenas o "My IP".

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/0c269eb8-fb00-4031-b0ce-8f758d0898fe" />

2. **Alteração no S3**: Criei um arquivo e fiz upload para o bucket S3 criado pela stack.

<img width="878" height="663" alt="image" src="https://github.com/user-attachments/assets/2176ead7-7104-474d-8c2a-410e09b2fe5f" />

Executei a detecção de drift:

```bash
aws cloudformation detect-stack-drift --stack-name myStack
aws cloudformation describe-stack-resource-drifts --stack-name myStack
```

<img width="878" height="663" alt="image" src="https://github.com/user-attachments/assets/73793487-35af-45eb-9292-119c1bcbab88" />

**Resultado**:

* O Security Group apareceu como **MODIFIED** (Drift detectado).

* O Bucket S3 apareceu como **IN_SYNC** (Adicionar arquivos não conta como drift de configuração de infraestrutura).

<img width="1920" height="1052" alt="image" src="https://github.com/user-attachments/assets/b8604ff9-a672-4889-9173-238840566eaa" />

## Tarefa 4 (Desafio): Deletar a Stack mantendo o Bucket

Ao tentar deletar a stack (`aws cloudformation delete-stack`), a operação falhou (`DELETE_FAILED`).
**Motivo**: O CloudFormation não deleta buckets S3 que contenham objetos, para evitar perda de dados.

<img width="1920" height="1052" alt="image" src="https://github.com/user-attachments/assets/bf63d632-3795-45de-8076-d5604f9ba20c" />

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/558e5ca2-b53d-42fd-876c-28aef0863baf" />

**O Desafio**: Deletar a stack, mas manter o bucket e seus arquivos.

**Solução**:
Utilizei a CLI para deletar a stack especificando que o recurso do bucket deveria ser retido (`--retain-resources`).

1. Descobri o ID Lógico do Bucket:

```bash
aws cloudformation describe-stack-resources --stack-name myStack --query "StackResources[?ResourceType=='AWS::S3::Bucket'].LogicalResourceId"
# Saída: "S3Bucket"
```

2. Executei o comando de deleção com retenção:

```bash
aws cloudformation delete-stack --stack-name myStack --retain-resources S3Bucket
```

A stack foi deletada com sucesso (`DELETE_COMPLETE`), mas o bucket S3 permaneceu na minha conta, preservando os dados.

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/830be8fb-bcd5-4183-96bf-a394db43bad5" />

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/be2af907-09af-485e-a40d-a626861e08ce" />
