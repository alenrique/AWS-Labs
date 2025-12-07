# Lab - Monitoramento de Infraestrutura

Neste laboratório, utilizei um conjunto de ferramentas de monitoramento da AWS para garantir a confiabilidade e conformidade da infraestrutura. Utilizei o **Amazon CloudWatch** (Metrics, Logs e Events) para coletar dados, visualizar logs e configurar alertas em tempo real. Além disso, configurei o **AWS Config** para auditar a conformidade dos recursos.

A arquitetura envolve a instalação do agente do CloudWatch em uma instância EC2 via Systems Manager e a configuração de regras de monitoramento.

## Tarefa 1: Instalando o Agente do CloudWatch

<img width="2391" height="926" alt="image" src="https://github.com/user-attachments/assets/ff408924-d70e-440f-9779-27bbaaa6fa65" />

Para coletar métricas internas (como uso de memória e disco) e logs do sistema, instalei o CloudWatch Agent na instância Web Server usando o **AWS Systems Manager (SSM)**.

1. **Instalação**: No console do Systems Manager, utilizei o **Run Command** com o documento `AWS-ConfigureAWSPackage` para instalar o `AmazonCloudWatchAgent` na instância.

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/3dab2ed5-2dc1-4b64-8cab-c8fed93ab9d4" />

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/e8214f1d-e33d-4353-90cb-17cf2e85a54f" />

2. **Configuração (Parameter Store)**: Criei um parâmetro no Parameter Store chamado `Monitor-Web-Server` contendo o arquivo de configuração JSON. Esse JSON define quais logs (Apache Access/Error logs) e métricas (CPU, Disk, Memory) o agente deve coletar.

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/1e9cd6d7-a13a-48e9-a523-37e0569119bf" />

3. **Inicialização**: Executei novamente o Run Command, desta vez usando o documento `AmazonCloudWatch-ManageAgent` para iniciar o agente apontando para a configuração salva no Parameter Store.

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/4cd648cc-b9e7-4464-8a44-73fa9b53b506" />

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/9ee7f3ef-4a86-4ae4-b0e6-8cb0d82ed376" />

## Tarefa 2: Monitorando logs de aplicação com CloudWatch Logs

<img width="2391" height="799" alt="image" src="https://github.com/user-attachments/assets/f3318c67-a554-4bcb-a743-63d509e0061e" />

Com o agente rodando, os logs do servidor web começaram a ser enviados para o CloudWatch. Configurei filtros e alarmes baseados nesses logs.

1. **Geração de Logs**: Acessei URLs inexistentes no servidor web (ex: `/start`) para gerar erros 404 nos logs.

<img width="1919" height="1006" alt="image" src="https://github.com/user-attachments/assets/7b088f42-baaa-4b81-a9b5-f6a76d7367d9" />

2. **Visualização**: No console do CloudWatch, verifiquei o grupo de logs `HttpAccessLog` e confirmei que as requisições estavam sendo registradas.

<img width="1919" height="963" alt="image" src="https://github.com/user-attachments/assets/c9347f73-eabe-495d-a05d-e722277ead55" />

3. **Filtro de Métricas**: Criei um filtro de métrica com o padrão:
`[ip, id, user, timestamp, request, status_code=404, size]`
Isso permite transformar registros de log brutos em uma métrica numérica chamada `404Errors`.

<img width="1919" height="963" alt="image" src="https://github.com/user-attachments/assets/31087b2b-d142-49db-ae13-6e5a51f4073a" />

4. **Alarme**: Criei um alarme que dispara se a contagem de erros 404 for maior ou igual a 5 em um período de 1 minuto. Configurei para enviar uma notificação para um tópico SNS (e-mail).

<img width="1919" height="963" alt="image" src="https://github.com/user-attachments/assets/51f15e04-08ec-4571-be3d-bf447f7fa79f" />

5. **Teste**: Gerei vários erros 404 propositalmente e confirmei que o alarme entrou em estado **ALARM** e recebi o e-mail de notificação.

<img width="1919" height="963" alt="image" src="https://github.com/user-attachments/assets/6da7c081-0f6c-4809-ae05-821eafb586f5" />

## Tarefa 3: Monitorando métricas de instância

<img width="2391" height="982" alt="image" src="https://github.com/user-attachments/assets/c38e509a-5825-430e-9564-47a052264a9b" />

Além das métricas padrão (vistas de fora da instância), visualizei as métricas internas coletadas pelo agente.

1. No CloudWatch Metrics, acessei o namespace **CWAgent**.

2. Visualizei métricas personalizadas como uso de disco (`disk_used_percent`) e memória, que não estão disponíveis no monitoramento padrão do EC2.

<img width="1919" height="963" alt="image" src="https://github.com/user-attachments/assets/e6cbadf9-eca0-4ab1-9168-0ec7c19ffb9e" />

## Tarefa 4: Criando notificações em tempo real (CloudWatch Events)

Configurei o CloudWatch Events (EventBridge) para reagir a mudanças de estado na infraestrutura.

1. **Regra**: Criei uma regra chamada `Instance_Stopped_Terminated`.

2. **Padrão de Evento**: Configurei para detectar mudanças de estado do serviço EC2 para stopped ou terminated.

3. **Target**: Configurei para enviar uma mensagem para um tópico SNS.

4. **Teste**: Parei a instância Web Server manualmente e recebi uma notificação por e-mail quase instantaneamente, contendo os detalhes do evento em JSON.

<img width="1919" height="963" alt="image" src="https://github.com/user-attachments/assets/89ff1b07-85d2-4934-ba45-16da7e3ec650" />

## Tarefa 5: Monitorando conformidade com AWS Config

Para garantir que a infraestrutura segue os padrões da empresa, ativei o **AWS Config**.

1. **Configuração**: Ativei o gravador de configurações.

2. **Regras de Conformidade**: Adicionei duas regras gerenciadas:

    * `required-tags`: Verifica se os recursos possuem a tag "project".

    * `ec2-volume-inuse-check`: Verifica se existem volumes EBS soltos (não anexados).

3. **Auditoria**: Verifiquei o painel de conformidade. Identifiquei quais recursos estavam "Non-compliant" (por exemplo, volumes não utilizados ou instâncias sem a tag obrigatória).

<img width="1919" height="963" alt="image" src="https://github.com/user-attachments/assets/35242799-0f33-4d33-a399-991844113630" />
