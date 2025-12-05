# Lab - Escalando e Balanceando sua Arquitetura

Nesse laboratório, irei utilizar o Elastic Load Balancing (ELB) e o Amazon EC2 Auto Scaling para balancear a carga e escalar automaticamente minha infraestrutura. O objetivo é garantir que a aplicação tenha tolerância a falhas e elasticidade para lidar com picos de tráfego.

A arquitetura final consistirá em um Load Balancer distribuindo tráfego entre instâncias EC2 privadas, gerenciadas por um Auto Scaling Group.

<img width="1202" height="688" alt="image" src="https://github.com/user-attachments/assets/0f41a1da-463d-4b23-8c10-24f8f65b8b10" />

<img width="2534" height="1550" alt="image" src="https://github.com/user-attachments/assets/c3a23d4f-0392-4050-826b-addd54ba9669" />

## Tarefa 1: Criar uma AMI para o Auto Scaling

Para garantir que as novas instâncias subam com a aplicação já configurada, criei uma Amazon Machine Image (AMI) a partir da instância ```Web Server 1``` existente.

Passos realizados:

1. No console EC2, selecionei a instância ```Web Server 1```.

2. Fui em **Actions > Image and templates > Create image**.

3. Defini o nome como ```Web Server AMI```.

4. Criei a imagem. Essa AMI será a base ("molde") para as futuras instâncias.

<img width="1917" height="959" alt="image" src="https://github.com/user-attachments/assets/59905a14-8205-4fb1-af27-ea5d358294ad" />

## Tarefa 2: Criar um Load Balancer

Criei um **Application Load Balancer (ALB)** para distribuir o tráfego HTTP entre diferentes zonas de disponibilidade.

Configurações do ```LabELB```:

* **VPC**: Lab VPC.

* **Subnets**: Selecionei as duas Zonas de Disponibilidade e escolhi as **Public Subnet 1** e **Public Subnet 2**.

* **Security Group**: Removi o padrão e adicionei o ```Web Security Group``` (que permite HTTP).

* **Target Group**: Criei um novo grupo chamado ```lab-target-group``` do tipo Instance.

Após a criação, copiei o **DNS name** do Load Balancer para usar posteriormente.

<img width="1917" height="959" alt="image" src="https://github.com/user-attachments/assets/149888fa-2520-4524-908b-3c074cac2bfd" />

## Tarefa 3: Criar um Launch Template

Criei um modelo de lançamento (Launch Template) que define como as instâncias do Auto Scaling devem ser configuradas.

Configurações do ```lab-app-launch-template```:

* **AMI**: Selecionei a ```Web Server AMI``` (criada na Tarefa 1).

* **Instance Type**: ```t3.micro```.

* **Key pair**: "Don't include in launch template" (não precisarei de acesso SSH para este teste).

* **Security Group**: ```Web Security Group```.

<img width="1917" height="959" alt="image" src="https://github.com/user-attachments/assets/377daf4b-ec35-4f35-835f-4d1f8bb8a6bf" />

## Tarefa 4: Criar um Auto Scaling Group

Utilizando o Launch Template, criei o Auto Scaling Group para gerenciar o ciclo de vida das instâncias.

Configurações do ```Lab Auto Scaling Group```:

* **Launch Template**: ```lab-app-launch-template```.

* **Rede**: Lab VPC, selecionando **Private Subnet 1** e **Private Subnet 2**.

* **Load Balancing**: Attach to an existing load balancer -> ```lab-target-group```.

* **Health Check**: Ativei o tipo ```ELB```.

* **Capacidade**:

    * Desired: 2

    * Minimum: 2

    * Maximum: 4

* **Scaling Policies**: Target tracking scaling policy com foco em **Average CPU utilization** em **50%**.

Isso garante que o grupo mantenha 2 instâncias e escale até 4 se a CPU passar de 50%.

<img width="1917" height="959" alt="image" src="https://github.com/user-attachments/assets/cf1d7e58-e575-441b-8099-0804fb6010c5" />

## Tarefa 5: Verificar se o Load Balancing está funcionando

Após o Auto Scaling subir as duas instâncias desejadas, verifiquei se elas estavam saudáveis.

1. Fui em **Target Groups** e verifiquei se as duas instâncias ```Lab Instance``` estavam com status **Healthy**.

2. Abri o endereço DNS do Load Balancer no navegador.

3. A aplicação "Load Test" carregou corretamente, confirmando que o tráfego estava sendo roteado.

<img width="1917" height="959" alt="image" src="https://github.com/user-attachments/assets/5fb89569-3f49-481a-a449-6e69f3a0a661" />

<img width="1917" height="959" alt="image" src="https://github.com/user-attachments/assets/82aea62b-01f0-405c-abd2-d364fda13706" />

<img width="1917" height="959" alt="image" src="https://github.com/user-attachments/assets/405cb050-01ed-4364-a972-751fcf7bc4a7" />

## Tarefa 6: Testar o Auto Scaling

Para validar a elasticidade, gerei uma carga artificial na aplicação.

1. No CloudWatch, observei os alarmes criados automaticamente pelo Auto Scaling (AlarmHigh e AlarmLow).

2. Na aplicação web, cliquei no logo Load Test para estressar a CPU.

3. Após alguns minutos, o alarme AlarmHigh entrou em estado In alarm.

4. Verifiquei no console EC2 que novas instâncias Lab Instance foram iniciadas automaticamente para lidar com a carga.

<img width="1917" height="959" alt="image" src="https://github.com/user-attachments/assets/9b623770-00eb-4357-a641-dd127e286525" />

<img width="1917" height="959" alt="image" src="https://github.com/user-attachments/assets/e5f314ab-bd2c-4109-bee5-a2c46455dbc3" />

<img width="1917" height="959" alt="image" src="https://github.com/user-attachments/assets/a276f8d6-4929-4b6f-823d-4e50753c57cb" />

## Tarefa 7: Encerrar a instância Web Server 1

Como o Auto Scaling agora gerencia a aplicação, a instância original ```Web Server 1``` tornou-se obsoleta. Procedi com o encerramento (Terminate) dela via console.

<img width="1917" height="959" alt="image" src="https://github.com/user-attachments/assets/e2356385-e284-40b5-bb33-83b2ae997dd9" />

<img width="1917" height="959" alt="image" src="https://github.com/user-attachments/assets/81b359c7-fca5-4b7d-9ada-042add11cd12" />

## Desafio Opcional: Criar uma AMI usando AWS CLI

Aproveitei para praticar a criação de AMIs via linha de comando.

1. Conectei em uma das instâncias via **Instance Connect**.

2. Configurei minhas credenciais com ```aws configure```.

3. Executei o comando para criar a imagem de uma instância em execução:

```bash
# Comando para criar a imagem
aws ec2 create-image --instance-id [id-da-intância] --name "AMI-Via-CLI" --description "AMI criada pelo desafio opcional" --no-reboot
```

O comando retornou o ID da nova AMI, confirmando o sucesso da operação.

<img width="1917" height="959" alt="image" src="https://github.com/user-attachments/assets/4fa58873-82e3-4e07-b6c0-571466ac5a1b" />
