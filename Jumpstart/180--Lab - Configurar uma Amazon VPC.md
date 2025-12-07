# Lab - Configurando uma VPC

Neste laboratório, construí uma Virtual Private Cloud (VPC) do zero, configurando sub-redes públicas e privadas, tabelas de rotas e gateways para controlar o fluxo de tráfego. O objetivo final foi criar uma arquitetura de rede segura onde recursos privados podem acessar a internet (via NAT Gateway) sem serem acessíveis diretamente por ela.

A arquitetura inclui um Bastion Host na sub-rede pública para acesso administrativo às instâncias privadas.

<img width="1460" height="878" alt="image" src="https://github.com/user-attachments/assets/cbb4b81b-1b83-49cf-8ba5-4bc57eb6e4f8" />

## Tarefa 1: Criando a VPC

Criei uma nova VPC chamada `Lab VPC` com o bloco CIDR `10.0.0.0/16`.
Após a criação, editei as configurações para habilitar **DNS hostnames**, garantindo que as instâncias recebam nomes de domínio públicos automaticamente.

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/e65e41f2-72ef-44e1-8c80-e323b1fd9ffb" />

## Tarefa 2: Criando sub-redes

Para segmentar a rede, criei duas sub-redes em Zonas de Disponibilidade específicas:

1. **Public Subnet**:

    * CIDR: `10.0.0.0/24`

    * Configuração adicional: Habilitei a opção **Enable** auto-assign **public IPv4 address** para que as instâncias aqui recebam IP público automaticamente.
  
<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/70a024ea-107b-451f-ae47-6613ad5dce2f" />

2. **Private Subnet**:

    * CIDR: `10.0.2.0/23` (Note que este bloco é maior, abrangendo 10.0.2.x e 10.0.3.x).

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/7be0cf83-93ae-4e99-98e7-368e0a6e1609" />

## Tarefa 3: Criando um Internet Gateway (IGW)

Para permitir a comunicação da VPC com a internet, criei um Internet Gateway chamado `Lab IGW` e o anexei à `Lab VPC`.

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/e983fd62-f128-4f63-9c3b-5a4d59fb011a" />

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/ded94522-e10f-4061-827c-814304954b57" />

## Tarefa 4: Configurando Tabelas de Rotas

Configurei o roteamento para definir o que é público e o que é privado.

1. **Private Route Table**: Renomeei a tabela de rotas principal (Main) para `Private Route Table`. Por enquanto, ela só possui a rota local.

2. **Public Route Table**:

    * Criei uma nova tabela chamada `Public Route Table`.

    * Adicionei uma rota para `0.0.0.0/0` apontando para o **Internet Gateway** (`Lab IGW`).

    * Associei esta tabela à **Public Subnet**.

Agora, qualquer recurso na sub-rede pública tem uma rota direta para a internet.

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/272b5c32-3eb1-4310-84da-9bb4e8da373f" />

## Tarefa 5: Lançar um servidor Bastion na sub-rede pública

Lancei uma instância EC2 para servir como ponto de entrada (Jump box).

* **Nome**: `Bastion Server`

* **AMI**: Amazon Linux 2023

* **Rede**: `Lab VPC` / `Public Subnet`

* **Security Group**: Criei o `Bastion Security Group` permitindo SSH (porta 22) de qualquer lugar (`0.0.0.0/0`).

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/d9d45780-e2d2-4475-afa3-096c0cd5373f" />

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/f302c111-ea15-46ec-88b0-34e57018047d" />

## Tarefa 6: Criando um NAT Gateway

Para que as instâncias na sub-rede privada possam baixar atualizações da internet sem estarem expostas, configurei um NAT Gateway.

1. **Criação**: Criei o `Lab NAT gateway` na **Public Subnet** e aloquei um Elastic IP para ele.

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/c79cdc4c-9583-4216-821d-a2ae76df0010" />

2. **Roteamento**:

    * Fui na `Private Route Table`.

    * Adicionei uma rota para `0.0.0.0/0` apontando para o **NAT Gateway**.
  
<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/5898486a-7987-47c2-9412-e43d1ded71f2" />

Agora, o tráfego de saída da sub-rede privada flui através do NAT na sub-rede pública.

## Desafio Opcional: Testando a sub-rede privada

Para validar a arquitetura, lancei uma instância na sub-rede privada e testei a conectividade.

1. **Lançar a Instância Privada**

Lancei a instância `Private Instance` na **Private Subnet**.

* **Security Group**: `Private Instance SG`, permitindo SSH apenas da rede interna (`10.0.0.0/16`).

* **User Data**: Inseri um script para habilitar autenticação por senha (para facilitar o laboratório):

```bash
#!/bin/bash
echo 'lab-password' | passwd ec2-user --stdin
sed -i 's|[#]*PasswordAuthentication no|PasswordAuthentication yes|g' /etc/ssh/sshd_config
systemctl restart sshd.service
```

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/cac32c5f-c9b4-4692-96be-138c693bf460" />

2. **Conectar via Bastion**

Como a instância privada não tem IP público, conectei primeiro no Bastion Server:

```bash
ssh -i chave.pem ec2-user@[IP-PUBLICO-DO-BASTION]
```

(No caso do lab, usei EC2 Instance Connect).

De dentro do Bastion, conectei na instância privada usando o IP privado dela:

```bash
ssh ec2-user@[IP-PRIVADO-DA-INSTANCIA]
# Senha: lab-password
```

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/2b2b9c44-7957-484f-87d6-5e9ebf98c1db" />

3. Testar o NAT Gateway

Dentro da instância privada, executei um ping para a internet:

```bash
ping -c 3 google.com
```

O ping respondeu com sucesso, confirmando que a instância privada consegue acessar a internet através do NAT Gateway, mas permanece inacessível diretamente de fora.

<img width="1918" height="950" alt="image" src="https://github.com/user-attachments/assets/650ddf97-c301-4e72-a664-62c659367f5b" />
