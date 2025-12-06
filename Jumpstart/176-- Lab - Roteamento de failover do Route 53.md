# Lab - Amazon Route 53 Failover Routing

Neste laboratório, configurei o roteamento de failover para uma aplicação web utilizando o Amazon Route 53. O ambiente consiste em duas instâncias EC2 (CafeInstance1 e CafeInstance2) executando a aplicação Café em Zonas de Disponibilidade diferentes para garantir alta disponibilidade.

O objetivo foi configurar o DNS de forma que, se a instância primária falhar, o tráfego seja automaticamente redirecionado para a instância secundária.

A arquitetura final implementada é a seguinte:

<img width="1136" height="922" alt="image" src="https://github.com/user-attachments/assets/e0e3a5cd-fc94-45a4-b624-9867cdc62ef8" />

## Tarefa 1: Confirmar os websites do Café

Primeiro, analisei os recursos criados automaticamente pelo CloudFormation. Recuperei os endereços IPs e URLs das duas instâncias.

1. Acessei a **PrimaryWebSiteURL** no navegador e confirmei que a aplicação estava rodando na Zona de Disponibilidade primária (ex: ```us-west-2a```).

2. Acessei a **SecondaryWebsiteURL** e confirmei que a versão de backup estava rodando na Zona de Disponibilidade secundária (ex: ```us-west-2b```).

3. Realizei um pedido de teste em um dos sites para validar a funcionalidade do banco de dados e da aplicação.

<img width="1920" height="1012" alt="image" src="https://github.com/user-attachments/assets/450334d6-4b14-4cc3-abbf-ca3451960e0e" />

<img width="1920" height="1012" alt="image" src="https://github.com/user-attachments/assets/367e1399-f60d-4538-80fd-b148da5d7be7" />

## Tarefa 2: Configurar um Health Check do Route 53

Para que o failover funcione, o Route 53 precisa saber quando o servidor principal parou de responder. Para isso, criei um Health Check.

Configurações utilizadas:

* **Name**: ```Primary-Website-Health```

* **What to monitor**: Endpoint (IP address).

* **IP address**: IP Público da ```CafeInstance1```.

* **Path**: ```cafe```

* **Request interval**: Fast (10 seconds) - para detecção rápida de falhas.

* **Failure threshold**: 2

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/336f52a1-d930-4085-9971-a41f461e275b" />

Além disso, configurei um alarme para me notificar por e-mail caso a saúde falhasse:

* **Create alarm**: Yes

* **Topic name**: ```Primary-Website-Health```

* **Email**: Meu e-mail pessoal (confirmei a inscrição recebida na caixa de entrada).

Após a criação, aguardei o status mudar para Healthy.

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/447631ad-5970-4b4b-9ef1-124fa355a5c6" />

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/fc3b698a-4961-4a3b-b2cb-e4da9216efe2" />

## Tarefa 3: Configurar registros do Route 53

Com o monitoramento ativo, configurei a Zona Hospedada (Hosted Zone) do meu domínio (```.vocareum.training```) para usar a política de Failover.

3.1 Criar registro A para o site primário

Criei o registro ```www``` apontando para o servidor principal:

* **Routing policy**: Failover

* **Failover record type**: Primary

* **Value**: IP da ```CafeInstance1```

* **Health check ID**: Selecionei o ```Primary-Website-Health``` criado anteriormente.

* **Record ID**: ```FailoverPrimary```

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/0d9a323c-dd78-4581-8bf5-a14198c38efd" />

3.2 Criar registro A para o site secundário

Criei o mesmo registro ```www``` apontando para o servidor de backup:

* **Routing policy**: Failover

* **Failover record type**: Secondary

* **Value**: IP da ```CafeInstance2```

* **Record ID**: ```FailoverSecondary```

_Nota: Não é necessário associar health check ao secundário neste cenário simples, pois ele assume todo o tráfego se o primário falhar._

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/4d877845-ce12-45a1-90f4-3a3fec9beb93" />

## Tarefa 4: Verificar a resolução de DNS

Para testar a configuração normal:

1. Copiei o nome do registro A completo (ex: ```www.XXXXXX.vocareum.training```).

2. Acessei ```http://www.MEU-DOMINIO.../cafe``` no navegador.

3. A página carregou e, na seção "Server Information", confirmou que estava sendo servida pela instância da **Zona Primária**.

<img width="1920" height="1005" alt="image" src="https://github.com/user-attachments/assets/27b36779-39d4-4d1c-b824-dfbcef8509af" />

## Tarefa 5: Verificar a funcionalidade de Failover

Para simular um desastre e validar a resiliência da arquitetura:

1. No console EC2, parei a instância ```CafeInstance1``` (**Stop instance**).

<img width="1919" height="965" alt="image" src="https://github.com/user-attachments/assets/a751c9da-a048-4c06-be13-c14712807e71" />

2. No console do Route 53 > Health Checks, observei o monitoramento. Em poucos minutos, o status mudou para **Unhealthy**.

3. Recebi um e-mail de notificação do AWS SNS informando sobre o alarme.

<img width="1919" height="965" alt="image" src="https://github.com/user-attachments/assets/136921c4-8b97-4d62-af2a-1c6bbf0b7fdb" />

4. Voltei ao navegador e recarreguei a página do meu domínio (```www.../cafe```).

5. A aplicação carregou com sucesso, mas a seção "Server Information" agora mostrava a Zona Secundária (ex: ```us-west-2b```).

<img width="1920" height="1010" alt="image" src="https://github.com/user-attachments/assets/bd07c186-9ccc-4f4c-8486-98bff5fd2a17" />

Isso confirmou que o Route 53 detectou a falha e redirecionou o tráfego automaticamente para a infraestrutura de backup.
