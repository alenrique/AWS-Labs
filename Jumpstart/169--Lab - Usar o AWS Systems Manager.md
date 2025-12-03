# 169--Lab - Usar o AWS Systems Manager

Nesse laboratório irei utilizar o AWS Systems Manager para centralizar dados operacionais e automatizar tarefas em recursos da AWS. O objetivo é verificar configurações, executar tarefas em servidores, atualizar configurações de aplicações e acessar a linha de comando de uma instância sem a necessidade de SSH direto via porta 22.

Abaixo segue uma visão geral da infraestrutura:

<img width="1572" height="808" alt="image" src="https://github.com/user-attachments/assets/172f9c43-7a11-4fb6-ae3a-5a40aeedf6ca" />

## Tarefa 1: Gerar listas de inventário para instâncias gerenciadas

Nessa parte, utilizei o **Fleet Manager** para coletar informações do sistema operacional e da aplicação na instância EC2.

Passos realizados:
* Na barra de pesquisa, pesquisei "System manager" e cliquei no serviço.
* No console do Systems Manager, acessei **Fleet Manager** no menu de navegação.
* Em "Account management", selecionei **Set up inventory**.
* Configurei a associação com o nome `Inventory-Association`.
* Em targets, selecionei a opção para escolher instâncias manualmente e marquei a **Managed Instance**.
* Após a configuração, acessei a aba **Inventory** dentro da instância gerenciada para validar os softwares instalados.

<img width="1919" height="962" alt="image" src="https://github.com/user-attachments/assets/df2cead0-9b38-48da-bdf6-e28796fa5e4c" />

## Tarefa 2: Instalar uma aplicação personalizada usando o Run Command

Agora, utilizarei o **Run Command** para instalar uma aplicação web personalizada (Widget Manufacturing Dashboard) sem precisar logar na máquina.

Passos:
* No menu do Systems Manager, fui em **Run Command**.
* Pesquisei pelo documento clicando na barra de busca e selecionando: `Owner` -> `Owned by me`.
* Selecionei o documento **Install Dashboard App**.
* Em "Target selection", escolhi a instância manualmente (**Managed Instance**).
* Desmarquei a opção de escrita no S3 e executei o comando.

Após o sucesso da execução (Status: Success), validei a instalação copiando o IP Público da instância e colando no navegador:

<img width="1919" height="1006" alt="image" src="https://github.com/user-attachments/assets/fb87e906-e7fe-4b62-bd06-fe6ed2c88812" />

## Tarefa 3: Usar o Parameter Store para gerenciar configurações de aplicativos

Utilizei o **Parameter Store** para armazenar uma variável de configuração que ativa uma funcionalidade "beta" na aplicação instalada anteriormente.

Passos:
* No menu **Application Management**, cliquei em **Parameter Store**.
* Criei um novo parâmetro com as seguintes configurações:
  * **Name:** `/dashboard/show-beta-features`
  * **Value:** `True`
* Salvei o parâmetro.

Ao recarregar a página da aplicação no navegador, o gráfico extra (feature beta) apareceu automaticamente, pois a aplicação lê esse parâmetro.

<img width="1919" height="1005" alt="image" src="https://github.com/user-attachments/assets/7a8b6209-b89b-4226-9b3e-ad437d8790a4" />

## Tarefa 4: Usar o Session Manager para acessar instâncias

Para finalizar, acessei a instância de forma segura usando o **Session Manager**, que permite acesso ao shell via browser sem abrir portas de entrada (inbound ports) ou gerenciar chaves SSH.

Passos:
* No menu **Node Management**, selecionei **Session Manager**.
* Cliquei em **Start session** e selecionei a instância.
* Com o terminal aberto no navegador, executei os comandos abaixo para listar os arquivos da aplicação:

```bash
ls /var/www/html
```

Também executei o script abaixo para obter a região atual e listar detalhes da instância via CLI dentro da sessão:

```bash
# Obter região
AZ=`curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone`
export AWS_DEFAULT_REGION=${AZ::-1}

# Listar informações sobre instâncias EC2
aws ec2 describe-instances
```

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/128c9de1-5054-45fa-88f7-dfabed11c5fe" />
