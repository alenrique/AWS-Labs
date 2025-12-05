# Lab - Troubleshooting na Criação de uma Instância EC2

Nesse laboratório, o objetivo é utilizar a AWS CLI para lançar uma instância EC2 configurada com uma pilha LAMP (Linux, Apache, MySQL/MariaDB, PHP). No entanto, o script fornecido contém erros intencionais. Minha tarefa é identificar esses problemas, corrigir o script e garantir que a aplicação web "Café" seja implantada corretamente.

A arquitetura final será uma instância EC2 hospedando a aplicação web e o banco de dados.

<img width="3136" height="1536" alt="image" src="https://github.com/user-attachments/assets/37123fda-2cf9-48cc-b93c-759c5c830a63" />

## Tarefa 1: Conectar à instância CLI Host

Para executar os comandos da AWS CLI, conectei-me a uma instância pré-configurada chamada "CLI Host" usando o **EC2 Instance Connect**.

1. No console EC2, selecionei a instância CLI Host.

2. Cliquei em **Connect** e abri o terminal no navegador.

<img width="1917" height="963" alt="image" src="https://github.com/user-attachments/assets/60aa2584-f36e-4642-a6e8-466eb33d9c78" />

## Tarefa 2: Configurar a AWS CLI

Dentro da instância CLI Host, precisei configurar as credenciais para que a CLI pudesse interagir com os serviços da AWS.

Executei o comando:

```bash
aws configure
```

Preenchi com as informações fornecidas no laboratório:

* **AWS Access Key ID**: [Minha Access Key]

* **AWS Secret Access Key**: [Minha Secret Key]

* **Default region name**: [Região do Lab, ex: us-west-2]

* **Default output format**: json

<img width="1917" height="963" alt="image" src="https://github.com/user-attachments/assets/ab0bcdac-3ad2-43f0-b9d7-9947aeb46b9d" />

## Tarefa 3: Criar uma instância EC2 usando a AWS CLI (Troubleshooting)

Nesta etapa, trabalhei com o script ```create-lamp-instance-v2.sh```.

### 3.1 Preparação e Análise

Primeiro, criei um backup do script original para segurança:

```bash
cd ~/sysops-activity-files/starters
cp create-lamp-instance-v2.sh create-lamp-instance.backup
```

Analisei o script usando o ```vi```. Notei que ele busca a VPC correta, define variáveis, limpa recursos antigos e tenta lançar uma nova instância com um script de User Data.

<img width="1917" height="963" alt="image" src="https://github.com/user-attachments/assets/285be7e1-6ed2-4216-aa39-309bc673b4c6" />

### 3.2 Primeira Tentativa e Erro #1

Ao tentar executar o script:

```bash
./create-lamp-instance-v2.sh
```

Recebi o erro: ```An error occurred (InvalidAMIID.NotFound) ... The image id '[ami-xxxxxxxxxx]' does not exist```.

<img width="1917" height="963" alt="image" src="https://github.com/user-attachments/assets/79d5ddf4-4571-4f19-add9-564aaed3784c" />

**Diagnóstico e Solução**:
O erro indicava que a AMI especificada não existia na região configurada ou o comando estava buscando na região errada.

1. Editei o script com ```vi create-lamp-instance-v2.sh```.

2. Verifiquei a linha do comando ```run-instances```.

3. Ajustei o parâmetro de região/AMI para garantir que ele estivesse utilizando a AMI correta para a região ```us-west-2```.

<img width="1917" height="963" alt="image" src="https://github.com/user-attachments/assets/07c7eb1c-598a-4815-98f3-7a05de4ba52d" />

4. Salvei o arquivo e executei novamente. O comando funcionou e a instância foi lançada.

<img width="1917" height="963" alt="image" src="https://github.com/user-attachments/assets/1da922ec-a46f-4d65-862f-4212cddad8a0" />

### 3.3 Segunda Tentativa e Erro #2

A instância foi criada e recebeu um IP público, mas ao tentar acessar ```http://<IP-PUBLICO>``` no navegador, a página não carregava.

**Diagnóstico**:
Voltei ao terminal da CLI Host para investigar a conectividade. Instalei o ```nmap``` para escanear as portas:

```bash
sudo yum install -y nmap
nmap -Pn <IP-PUBLICO-DA-INSTANCIA>
```

O resultado mostrou que a porta **80 (HTTP)** não estava aberta/listada, apenas a 22 (SSH). Isso indicava um problema no Security Group.

<img width="1917" height="963" alt="image" src="https://github.com/user-attachments/assets/ca1a24ac-8278-4614-9209-3717c71c06e7" />

**Solução**:

1. Editei novamente o script ```create-lamp-instance-v2.sh```.

2. Localizei a seção onde o Security Group é criado e as regras de entrada (ingress) são autorizadas.

3. Adicionei/Corrigi a regra para permitir tráfego TCP na porta 80 (```--port 80 --cidr 0.0.0.0/0```).

<img width="1917" height="963" alt="image" src="https://github.com/user-attachments/assets/37a8ccda-593f-4e68-ab33-276321df6d4a" />

4. Executei o script novamente (confirmando a exclusão da instância e SG antigos criados na tentativa anterior).

<img width="1917" height="963" alt="image" src="https://github.com/user-attachments/assets/9ea5bba0-9f1d-4fa9-aa63-a32635121f55" />

<img width="1917" height="1007" alt="image" src="https://github.com/user-attachments/assets/0e1111b7-fcd1-4477-b473-460cd30bb036" />

**Validação do User Data**

Após a correção, verifiquei os logs dentro da nova instância (acessando via SSH ou EC2 Instance Connect) para garantir que o script de instalação rodou sem erros:

```bash
sudo tail -f /var/log/cloud-init-output.log
```

<img width="1917" height="963" alt="image" src="https://github.com/user-attachments/assets/931f14a0-8603-4117-95e0-2417c9f6fe9a" />

Vi as mensagens de confirmação: "Create Database script completed".

## Tarefa 4: Verificar a funcionalidade do site

Com os problemas de infraestrutura resolvidos, testei a aplicação final.

1. Acessei ```http://<IP-PUBLICO>/cafe no navegador```.

2. A página do Café carregou com sucesso.

<img width="1917" height="1005" alt="image" src="https://github.com/user-attachments/assets/d9bbd13c-4f19-4e8e-9aa7-bfe6ea293c94" />

3. Cliquei no link **Menu**. Uma nova página carregou em ```http://<IP-PUBLICO>/cafe/menu.php```.

4. Escolhi algumas sobremesas para pedir e cliquei em **Submit Order**. A página de confirmação apareceu com os detalhes do pedido.

5. Fiz outro pedido com itens diferentes. Em seguida, acessei a página **Order History**.

6. Confirmei que os detalhes de ambos os pedidos foram capturados, o que valida que a conexão com o banco de dados na instância LAMP está funcionando corretamente.

<img width="1917" height="1005" alt="image" src="https://github.com/user-attachments/assets/d7f248ee-e002-4369-9045-1881a43a8818" />
