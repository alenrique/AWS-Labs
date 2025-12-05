# Challenge Lab - Amazon EC2 Instances

Neste desafio, apliquei os conhecimentos adquiridos sobre Amazon EC2 para configurar uma rede virtual do zero, lançar uma instância Amazon Linux, instalar um servidor web e realizar o deploy de uma aplicação simples.

O objetivo foi configurar tudo manualmente, garantindo conectividade e permissões corretas.

## Passo 1: Configurar a Rede Virtual (VPC)

Antes de lançar a instância, precisei preparar a infraestrutura de rede para garantir que a máquina tivesse acesso à internet.

1. VPC: Criei uma nova VPC.

2. Internet Gateway: Criei um Internet Gateway e o anexei à VPC criada.

3. Subnet: Criei uma Subnet pública dentro da VPC.

4. Route Table: Configurei a tabela de rotas da subnet para direcionar o tráfego de internet (0.0.0.0/0) para o Internet Gateway.

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/c0b29801-0039-4b2f-bd47-354462494ece" />

## Passo 2: Lançar a Instância EC2

Com a rede pronta, iniciei o processo de lançamento da instância no console EC2 com as seguintes especificações:

* **AMI**: Amazon Linux 2 AMI.

* **Instance Type**: t3.micro (Atendendo ao requisito de ser menor que medium e da família T3).

* **Network**: Selecionei a VPC e a Subnet criadas no passo anterior.

* **Auto-assign Public IP**: Habilitei (Enable) para garantir acesso externo.

* **Storage**: Configurei o volume raiz como General Purpose SSD (gp2).

**User Data**

Para automatizar a instalação do servidor web, inseri o seguinte script no campo User Data (Advanced Details). Esse script instala o Apache (```httpd```), inicia o serviço e ajusta as permissões da pasta raiz:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
chmod -R 777 /var/www/html
```

**Security Group**

Configurei um novo Security Group permitindo o tráfego necessário:

* **SSH (Porta 22)**: Para conexão remota.

* **HTTP (Porta 80)**: Para acesso ao servidor web.

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/d3a00c89-3574-40b5-ae66-f1446511a90b" />

## Passo 3: Verificação do System Log

Após a instância mudar o status para "Running", verifiquei se o script de User Data foi executado corretamente. Fui em **Actions > Monitor and troubleshoot > Get system log**.

O log confirmou que o serviço ```httpd``` foi instalado e iniciado com sucesso.

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/a0794af5-a6a2-42ef-8452-62a08faba9d2" />

## Passo 4: Deploy da Aplicação Web

Utilizei o **EC2 Instance Connect** para acessar o terminal da instância via browser.

1. Naveguei até o diretório do servidor web:

```bash
cd /var/www/html
```

2. Criei o arquivo da aplicação:

```bash
nano projects.html
```

3. Colei o código HTML abaixo, substituindo "YOUR-NAME" pelo meu nome:

```bash
<!DOCTYPE html>
<html>
<body>
<h1>ALENRIQUE's re/Start Project Work</h1>
<p>EC2 Instance Challenge Lab</p>
</body>
</html>
```

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/33164e43-6d90-467f-8594-2bb901952a8b" />

4. Salvei o arquivo e fechei o editor.

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/852c45a5-e39b-4b1f-a5ca-69bbe06f66f8" />

## Passo 5: Teste Final

Para validar o desafio, abri uma nova aba no navegador e acessei o endereço público da instância seguido do nome do arquivo criado:

```bash
http://[IP-PUBLICO]/projects.html
```

A página carregou corretamente, exibindo o título com meu nome e o texto do projeto.

<img width="1917" height="1007" alt="image" src="https://github.com/user-attachments/assets/7543fda8-9c5e-4b29-ac63-d78137e16081" />
