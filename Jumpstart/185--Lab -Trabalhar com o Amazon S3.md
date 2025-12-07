# Lab - Trabalhando com Amazon S3

Neste laboratório, configurei uma solução de compartilhamento de arquivos usando o Amazon S3. O objetivo foi permitir que um usuário externo (simulando uma agência de mídia) fizesse upload de imagens de produtos, enquanto o sistema envia automaticamente notificações por e-mail para o administrador sempre que o conteúdo do bucket é modificado.

A arquitetura envolve o S3 disparando eventos para um tópico Amazon SNS.

<img width="1068" height="724" alt="image" src="https://github.com/user-attachments/assets/5d6cbd8a-e930-469c-9042-a6e7b86404da" />

## Tarefa 1: Conectar e Configurar a CLI

Para iniciar, conectei-me à instância CLI Host via EC2 Instance Connect e configurei a AWS CLI com as credenciais administrativas (usuário voclabs).

```bash
aws configure
# Access Key: [Valor copiado]
# Secret Key: [Valor copiado]
# Region: us-west-2
# Output: json
```

## Tarefa 2: Criar e inicializar o bucket de compartilhamento

Utilizei a CLI para criar o bucket e povoá-lo com imagens iniciais.

1. **Criar o Bucket**:

```bash
aws s3 mb s3://cafe-[SEU-NOME-UNICO] --region 'us-west-2'
```

2. **Sincronizar Imagens**:
Copiei as imagens locais da pasta `~/initial-images/` para a pasta `images/` dentro do bucket:

```bash
aws s3 sync ~/initial-images/ s3://cafe-[SEU-NOME-UNICO]/images
```

3. **Verificar**:

```bash
aws s3 ls s3://cafe-[SEU-NOME-UNICO]/images/ --human-readable --summarize
```

<img width="1919" height="960" alt="image" src="https://github.com/user-attachments/assets/5e2ffa79-ec49-433b-89a6-563778444c1c" />

## Tarefa 3: Revisar Permissões de IAM

Analisei as permissões do grupo IAM `mediaco` e do usuário `mediacouser`. A política `mediaCoPolicy` permite que eles listem o bucket e façam upload/delete apenas na pasta `images/`.

**Teste via Console**

Para validar as permissões, loguei no Console da AWS como `mediacouser` em uma aba anônima.

<img width="1919" height="960" alt="image" src="https://github.com/user-attachments/assets/0c6c3772-f37d-4156-82d8-01b5afff7c8d" />

1. **Visualizar**: Consegui abrir e visualizar a imagem `Donuts.jpg`.

<img width="1919" height="1008" alt="image" src="https://github.com/user-attachments/assets/fd268158-b66f-4393-a4a5-3898f05cc921" />

2. **Upload**: Consegui fazer upload de uma imagem local para a pasta `images/`.

<img width="1919" height="1008" alt="image" src="https://github.com/user-attachments/assets/aa2d68c1-8f6b-41da-814c-3d39e00ec0cb" />

3. **Delete**: Consegui deletar a imagem `Cup-of-Hot-Chocolate.jpg`.

<img width="1919" height="1008" alt="image" src="https://github.com/user-attachments/assets/97c84b94-3857-4a38-84fc-4299a84d30c4" />

4. **Permissão Negada**: Tentei alterar as permissões do bucket na aba **Permissions**, e recebi o erro "Insufficient permissions" (esperado, pois o usuário tem permissões restritas).

<img width="1919" height="1008" alt="image" src="https://github.com/user-attachments/assets/0e5ca075-4e4e-4ee8-85b0-c736fb7577db" />

## Tarefa 4: Configurar Notificações de Evento

Configurei o sistema para me avisar quando arquivos forem alterados.

### 4.1 Configurar SNS

1. Criei um Tópico Standard chamado `s3NotificationTopic`.

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/8f94028d-4663-456b-b857-1eaa6e05c168" />

2. Copiei o ARN do tópico.

3. **Política de Acesso**: Editei a política do tópico para permitir que o serviço `s3.amazonaws.com` publique mensagens nele.

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/04787fbd-90d9-48a8-a64d-c6cfc7f544cf" />

4. **Assinatura**: Criei uma subscrição de e-mail para o meu endereço e confirmei na caixa de entrada.

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/cb5e0174-281f-4345-8a10-aaf1068380f0" />

### 4.2 Configurar o Bucket S3

Voltei ao terminal da CLI Host para configurar a notificação via arquivo JSON.

1. Criei o arquivo `s3EventNotification.json` definindo que eventos de `ObjectCreated` e `ObjectRemoved` na pasta `images/` devem disparar o SNS.

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/b2043a05-6ab4-4d41-8208-2cd07584f8cc" />

2. Apliquei a configuração ao bucket:

```bash
aws s3api put-bucket-notification-configuration --bucket cafe-[SEU-NOME-UNICO] --notification-configuration file://s3EventNotification.json
```

Recebi um e-mail de teste "s3:TestEvent", confirmando a integração.

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/c51074d4-c272-4de7-8510-5c2c93625dd2" />

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/7cdac007-4369-41fd-9d7c-420f06394df5" />

## Tarefa 5: Testar as notificações

Para o teste final, simulei as ações da agência de mídia usando a CLI com as credenciais restritas.

1. **Reconfigurar CLI**: Executei `aws configure` novamente, mas desta vez inseri as credenciais do usuário `mediacouser` (obtidas no console IAM).

2. **Teste de Upload (PUT)**:

```bash
aws s3api put-object --bucket cafe-[SEU-NOME-UNICO] --key images/Caramel-Delight.jpg --body ~/new-images/Caramel-Delight.jpg
```

_Resultado_: Recebi um e-mail de notificação `ObjectCreated:Put`.

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/5587b7e3-6e3d-4f8e-a378-e747aebb4981" />

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/db162f2e-7140-4965-9551-28a052faa940" />

3. **Teste de Download (GET)**:

```bash
aws s3api get-object --bucket cafe-[SEU-NOME-UNICO] --key images/Donuts.jpg Donuts.jpg
```

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/d4d9e162-7881-4ffa-9be2-4fc205fef0b0" />

_Resultado_: O download funcionou, mas não recebi e-mail (pois configurei apenas para criação/remoção).

4. **Teste de Exclusão (DELETE)**:

```bash
aws s3api delete-object --bucket cafe-[SEU-NOME-UNICO] --key images/Strawberry-Tarts.jpg
```

_Resultado_: Recebi um e-mail de notificação `ObjectRemoved:Delete`.

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/d4c2b51e-95f5-4a41-8607-4d459da23477" />

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/17d82b37-fd19-4c7d-a8fe-d3d7ca809220" />

5. **Teste de Acesso Indevido**:
Tentei tornar um objeto público:

```bash
aws s3api put-object-acl --bucket cafe-[SEU-NOME-UNICO] --key images/Donuts.jpg --acl public-read
```

_Resultado_: `An error occurred (AccessDenied)`, confirmando que as permissões de segurança estão funcionando.

<img width="1919" height="961" alt="image" src="https://github.com/user-attachments/assets/dda04d60-9062-4cb3-9ee3-94082af24f67" />
