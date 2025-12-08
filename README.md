# AWS Hands-on Labs & Challenges ☁️

Este repositório documenta a minha jornada prática de aprendizagem na Amazon Web Services (AWS). Contém relatórios detalhados, scripts e configurações de laboratórios "Jumpstart" e "Advanced", cobrindo desde a infraestrutura básica até automação complexa, resolução de problemas (troubleshooting) e arquiteturas serverless.

Cada laboratório foca num serviço ou cenário específico, demonstrando competências em **Infrastructure as Code (IaC)**, **Monitorização**, **Segurança** e **Redes**.

## 🛠️ Tecnologias e Serviços Explorados

* **Computação:** Amazon EC2, AWS Lambda, Auto Scaling, Elastic Load Balancing (ELB).
* **Armazenamento e Bases de Dados:** Amazon S3, Amazon EBS, Amazon RDS (MariaDB).
* **Redes:** Amazon VPC, Route 53, NAT Gateways, Internet Gateways, Security Groups, NACLs.
* **Gestão e Governação:** AWS CloudFormation, AWS Systems Manager (SSM), AWS CloudTrail, AWS Config, AWS CLI.
* **Monitorização:** Amazon CloudWatch (Logs, Metrics, Alarms, Events/EventBridge).

---

## 📂 Estrutura do Portefólio

Os laboratórios estão organizados por áreas de domínio:

### 🖥️ Computação e Escalabilidade
* **Gestão de EC2:** Criação de instâncias via Consola e CLI, criação de AMIs personalizadas e gestão de snapshots.
* **Alta Disponibilidade:** Implementação de Auto Scaling Groups com Application Load Balancers (ALB) para garantir elasticidade e tolerância a falhas.
* **Serverless:** Criação de funções AWS Lambda com camadas (Layers) e gatilhos (Triggers) S3/SNS para automação de tarefas.

### 🌐 Redes e Entrega de Conteúdo
* **VPC do Zero:** Construção manual de VPCs com sub-redes públicas/privadas, tabelas de rotas e Bastion Hosts.
* **DNS e Failover:** Configuração de zonas hospedadas no Route 53 com políticas de encaminhamento de failover (Active-Passive) para Disaster Recovery.
* **Troubleshooting de Rede:** Diagnóstico de problemas de conectividade, análise de VPC Flow Logs e correção de regras de NACL e Security Groups.

### 📦 Armazenamento e Bases de Dados
* **Amazon S3:** Alojamento de websites estáticos, versionamento de ficheiros, políticas de ciclo de vida e notificações de eventos via SNS.
* **Amazon EBS:** Gestão de volumes, formatação de sistemas de ficheiros (ext3), montagem e recuperação de dados via snapshots.
* **Migração de Bases de Dados:** Migração de uma base de dados local (MariaDB em EC2) para uma instância gerida Amazon RDS para maior escalabilidade.

### 🤖 Automação e Infraestrutura como Código (IaC)
* **CloudFormation:** Implementação de stacks completas (VPC + EC2 + S3) via templates YAML, deteção de desvios (Drift Detection) e resolução de falhas de deployment.
* **Systems Manager:** Gestão de inventário, execução de comandos remotos (Run Command) sem SSH e gestão de parâmetros (Parameter Store).
* **AWS CLI:** Utilização extensiva da linha de comandos para filtrar recursos (JMESPath), lançar instâncias e gerir buckets.

### 🛡️ Segurança e Monitorização
* **Auditoria de Segurança:** Investigação de incidentes de segurança utilizando o CloudTrail e Amazon Athena para identificar acessos não autorizados.
* **Monitorização Proativa:** Instalação do agente CloudWatch, criação de alarmes personalizados (ex: memória/disco) e respostas automáticas a eventos.
* **Conformidade:** Utilização de Tags para gestão de recursos e scripts de remediação para terminar instâncias não conformes.
* **Otimização de Custos:** "Rightsizing" de instâncias e estimativas de custos com a AWS Pricing Calculator.

---

## 🏆 Destaques e Desafios (Challenge Labs)

Este repositório inclui a resolução de vários "Challenge Labs", onde a arquitetura foi construída sem guias passo-a-passo:

1.  **VPC & EC2 com CloudFormation:** Criação de um template YAML do zero para provisionar uma infraestrutura de rede completa e servidores.
2.  **Serverless Word Count:** Criação de uma arquitetura orientada a eventos onde o upload de um ficheiro `.txt` no S3 aciona uma Lambda para contar palavras e enviar o resultado por e-mail (SNS).
3.  **Troubleshooting de VPC:** Resolução de um cenário "quebrado" onde a conectividade estava bloqueada por falta de rotas e regras de NACL incorretas.
4.  **Recuperação de Desastre:** Configuração de Health Checks no Route 53 para redirecionar tráfego automaticamente de uma região primária para uma secundária em caso de falha.

---

## 🚀 Como Utilizar

Cada pasta/ficheiro neste repositório representa a documentação de um laboratório específico.
* Os ficheiros **Markdown (.md)** contêm o passo-a-passo da execução, capturas de ecrã da arquitetura e comandos utilizados.
* Verifique os ficheiros com o prefixo `[Desafio]` ou `Troubleshoot` para ver cenários de resolução de problemas mais complexos.

## 👤 Autor

**Henrique Alencar**
*Cloud Enthusiast & AWS Practitioner*

---
*Este repositório serve para fins educacionais e de demonstração de competências práticas em ambiente AWS.*
