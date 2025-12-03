# 168--Lab - Instalar e configurar a CLI da AWS

Nesse laboratório irei instalar o Amazon CLI em um EC2 Amazon Linux. Irei fazer uma comunicação segura através de um par de chaves SSH.

Segue a infraestrutura desse laboratório:

<img width="915" height="384" alt="image" src="https://github.com/user-attachments/assets/9eb9fbf7-7d41-4bf1-a098-a725f7e2804a" />

## Tarefa 1: Conecte-se a uma instância Amazon usando SSH

Nessa parte irei pegar uma chave ```.pem``` pois estou utilizando Linux se tiver utilizando Windows terá que pegar um arquivo ```.ppk``` e se conectar através do Putty.

Sequencia para Linux: 
* Baixar o arquivo e entrar na pasta onde o arquivo foi baixado
```bash
cd ~/Downloads
```
* Mudar as permissões do arquivo
```bash
chmod 400 [nome_do_arquivo].pem
```
* Acessar a instância com SSH
```bash
ssh -i [nome_do_arquivo].pem ec2-user@[ip_da_instancia]
```
*O IP da instância você pode pegar na aba do EC2 e selecionando a instância em questão, lá irá aparecer as informções da instâcia e terá o IP Público para acessa-la*

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/83636c79-9ee8-45cc-8aa4-c7619838924c" />

## Tarefa 2: Instalar a AWS CLI em uma instância do Amazon Linux

Agora vamos instalar o Amazon CLI na instância que estamos utilizando.

Passos:
* Para baixar o arquivo e gravar no diretório atual
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```
* Para descompactar o instalador
```bash
unzip -u awscliv2.zip
```
* Para descompactar o instalador
```bash
unzip -u awscliv2.zip
```
* Para executar o instalador
```bash
sudo ./aws/install
```
* Para confirmar a instalação vendo a versão que está instalada
```bash
aws --version
```
* Para verificar se está rodando corretamente e ver a lista de comandos do Amazon CLI
```bash
aws help
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/8c156c2e-ba6b-4a56-977f-ec276add1abd" />

## Tarefa 3: Observe os detalhes da configuração do IAM no Console de Gerenciamento da AWS

No painel da AWS, na barra de pesquisa digitei "IAM" e cliquei no serviço. Depois cliquei em **Users** e cliquei no usuário *awsstudent*.

<img width="1919" height="962" alt="image" src="https://github.com/user-attachments/assets/0fe95deb-a950-4a44-92d0-568580a7b0e3" />

Depois cliquei em **Security credentials** e fui na aba **Access Keys** e copiei a chave de acesso daquele usuário.

## Tarefa 4: Configure a AWS CLI para conectar-se à sua conta da AWS.

Para acessar o Amazon CLI usaremos o comando:
```bash
aws configure
```
e então colocaremos as credenciais.

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/c8500c33-3f3e-4879-8e58-c0e543032293" />

## Tarefa 5: Observe os detalhes da configuração do IAM usando a AWS CLI

Agora vamos obsevar o IAM através da instância e do Amazon CLI. Usaremos o comando:
```bash
aws iam list-users
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/93f66e86-f13e-4e6a-9a32-b77ef4aca7f8" />

