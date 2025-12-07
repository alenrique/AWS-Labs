# Lab - Troubleshooting em uma VPC

Neste laboratório, atuei na resolução de problemas (troubleshooting) de rede em uma Virtual Private Cloud (VPC) e na análise de VPC Flow Logs. O cenário inicial consistia em uma arquitetura com conectividade quebrada, onde precisei identificar e corrigir configurações de rotas e listas de controle de acesso (NACLs) para restabelecer o acesso a um servidor web.

Além disso, configurei logs de fluxo (Flow Logs) para auditar o tráfego de rede e investigar tentativas de conexão rejeitadas.

<img width="1458" height="744" alt="image" src="https://github.com/user-attachments/assets/d055c962-d157-4f04-b83d-5046e342f5dc" />

## Tarefa 1: Conectar à instância CLI Host

Para realizar o diagnóstico e executar os comandos da AWS CLI, conectei-me à instância CLI Host utilizando o EC2 Instance Connect.

1. No console EC2, selecionei a instância CLI Host.

2. Cliquei em Connect e abri o terminal.

3. Configurei a CLI com as credenciais do laboratório:

```bash
aws configure
```

## Tarefa 2: Criar VPC Flow Logs

Antes de começar a corrigir os problemas, configurei o monitoramento para capturar o tráfego da rede.

1. **Criar Bucket S3**: Criei um bucket para armazenar os logs (substituindo `######` por números aleatórios):

```bash
aws s3api create-bucket --bucket flowlog217710 --region 'us-west-2' --create-bucket-configuration LocationConstraint='us-west-2'
```

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/682fe36f-3f4e-46f6-91a2-5065f5912f9f" />

2. **Identificar a VPC**: Busquei o ID da `VPC1`:

```bash
aws ec2 describe-vpcs --filters "Name=tag:Name,Values='VPC1'"
```

3. **Ativar Flow Logs**: Configurei a VPC para enviar logs de todo o tráfego (`ALL`) para o bucket S3 criado:

```bash
aws ec2 create-flow-logs --resource-type VPC --resource-ids [ID-DA-VPC1] --traffic-type ALL --log-destination-type s3 --log-destination arn:aws:s3:::flowlog123456
```

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/5470e646-d22f-4e3c-8f3e-9a9bc4a7903a" />

Verifiquei se o log estava ativo com `aws ec2 describe-flow-logs`.

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/1f9df4e3-c321-468c-8483-28543a78ff08" />

## Tarefa 3: Troubleshooting de Acesso (Desafios)

Tentei acessar o IP do servidor web (`WebServerIP`) no navegador, mas a conexão falhou. Também tentei conectar via SSH (EC2 Instance Connect), mas recebi erro. Comecei a investigação.

<img width="1920" height="1010" alt="image" src="https://github.com/user-attachments/assets/6d9b35a9-42ac-46f1-ae61-0fae07c638f9" />

<img width="1920" height="961" alt="image" src="https://github.com/user-attachments/assets/9f2b0dae-8cbc-4518-bcd9-5f69f903b829" />

### Desafio #1: Corrigindo acesso ao Web Server

**Investigação**:

1. Verifiquei se a instância estava rodando (`aws ec2 describe-instances`). **Resultado**: Sim.

<img width="1920" height="961" alt="image" src="https://github.com/user-attachments/assets/fbd6f8a5-da47-4d87-a58e-31b05631f7c4" />

2. Verifiquei o Security Group (`aws ec2 describe-security-groups`). **Resultado**: As portas 80 e 22 pareciam estar abertas corretamente.

<img width="1920" height="961" alt="image" src="https://github.com/user-attachments/assets/2c82b68d-2ec0-4eea-813c-20705676a09e" />

3. Verifiquei a Tabela de Rotas da sub-rede pública (`aws ec2 describe-route-tables`).

<img width="1920" height="961" alt="image" src="https://github.com/user-attachments/assets/031faea7-a1c1-407c-ba72-cc8cc3f49ee3" />

**Causa Raiz**:
Ao analisar a tabela de rotas associada à sub-rede pública, notei que não existia uma rota para a internet. O tráfego não tinha como sair para o Internet Gateway.

**Solução**:
Criei a rota faltante apontando `0.0.0.0/0` para o Internet Gateway (`igw-...`) da VPC1:

```bash
aws ec2 create-route --route-table-id [ID-DA-ROUTE-TABLE] --gateway-id [ID-DO-IGW] --destination-cidr-block '0.0.0.0/0'
```

<img width="1920" height="961" alt="image" src="https://github.com/user-attachments/assets/d5d45acf-934a-4e5d-861f-8228508565a0" />

**Validação**:
Atualizei a página no navegador e a mensagem "Hello From Your Web Server!" apareceu. O tráfego HTTP foi restabelecido.

<img width="1920" height="1009" alt="image" src="https://github.com/user-attachments/assets/0666ee95-707e-49ce-afe8-e4cc88ec5866" />

### Desafio #2: Corrigindo acesso SSH

Mesmo com o site funcionando, o acesso SSH (porta 22) via EC2 Instance Connect continuava falhando.

**Investigação**:
Como o Security Group e a Tabela de Rotas já haviam sido verificados, suspeitei das **Network ACLs (NACL)**, que atuam como um firewall stateless na sub-rede.

Executei o comando para listar as NACLs:

```bash
aws ec2 describe-network-acls --filter "Name=association.subnet-id,Values='[ID-DA-SUBNET-PUBLICA]'"
```

**Causa Raiz**:
Encontrei uma regra específica (Regra nº 40) que estava **NEGANDO (DENY)** tráfego de entrada na porta 22 ou tráfego SSH.

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/8c227f23-5770-4aa1-b252-f3487aaacaa4" />

**Solução**:
Removi a regra bloqueadora da NACL:

```bash
aws ec2 delete-network-acl-entry --network-acl-id [ID-DA-NACL] --ingress --rule-number 40
```

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/3cc3355a-d382-4581-bd58-2dc624a03c1e" />

**Validação**:
Tentei conectar novamente via EC2 Instance Connect e obtive sucesso. O acesso SSH foi restabelecido.

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/537e79b6-8e39-4deb-ab49-5e43e2795310" />

## Tarefa 4: Analisando Flow Logs

Com os problemas resolvidos, analisei os logs gerados durante as tentativas falhas de conexão para entender como o bloqueio foi registrado.

1. **Baixar Logs**: Criei um diretório local e baixei os logs do S3:

```bash
mkdir flowlogs
cd flowlogs
aws s3 cp s3://flowlog217710/ . --recursive
```

2. **Extrair Arquivos**: Descompactei os arquivos `.gz`:

```bash
gunzip *.gz
```

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/f86932ce-d9b7-4ee6-b9f5-756f5b17eaf5" />

3. **Análise (Grep)**:
Busquei nos logs por registros de **REJECT** na porta **22** associados ao meu IP (identifiquei meu IP público nas regras do Security Group para filtrar a busca):

```bash
grep -rn 22 . | grep [MEU-IP-PUBLICO]
```

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/ce72acab-36e3-4654-b952-b203ed177be3" />

O resultado mostrou as linhas de log.

4. **Interpretação**:
Utilizei o comando `date -d @TIMESTAMP` para converter os timestamps dos logs e confirmar que os bloqueios ocorreram no horário exato dos meus testes iniciais.

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/b4724f90-013a-49ad-897c-ae56e4de8c86" />
