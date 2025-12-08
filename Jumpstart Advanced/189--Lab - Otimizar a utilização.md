# Activity - Optimize Utilization

Nesta atividade, o objetivo foi otimizar os recursos da AWS utilizados pela aplicação web Café para reduzir custos. Após a migração do banco de dados para o Amazon RDS, o banco de dados local tornou-se obsoleto.

As ações tomadas foram:

1. Desinstalar o banco de dados local da instância EC2.

2. Redimensionar a instância EC2 de `t3.small` para `t3.micro`.

3. Estimar a economia de custos utilizando a **AWS Pricing Calculator**.

<img width="1244" height="580" alt="image" src="https://github.com/user-attachments/assets/f3d2e49b-c5bc-46f2-b774-24fff654cf34" />

## Tarefa 1: Otimizar o site para reduzir custos

### 1.1 Desinstalar o MariaDB

Primeiro, conectei-me à instância `CafeInstance` via SSH para remover o software de banco de dados que não estava mais sendo usado, liberando recursos.

```bash
sudo systemctl stop mariadb
sudo yum -y remove mariadb-server
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/99a1c423-28a3-4ba2-95d9-6afcd6576252" />

### 1.2 Redimensionar a Instância (Downsizing)

1. Utilizei a instância `CLI Host` para executar comandos da AWS CLI e modificar a instância da aplicação.

**Parar a instância**:
Recuperei o ID da instância e executei o comando de parada:

```bash
aws ec2 stop-instances --instance-ids [ID-DA-INSTANCIA]
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/ad5bcba3-d5fc-4348-954a-52ef6dcfd429" />

2. **Modificar o tipo da instância**:
Alterei o tipo de `t3.small` para `t3.micro`, que é mais barato e suficiente para a aplicação sem o banco de dados local:

```bash
aws ec2 modify-instance-attribute --instance-id [ID-DA-INSTANCIA] --instance-type "{\"Value\": \"t3.micro\"}"
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/3bfdf95d-4bff-4afd-8571-8ab161234581" />

3. **Iniciar a instância**:
Reiniciei a instância com a nova configuração:

```bash
aws ec2 start-instances --instance-ids [ID-DA-INSTANCIA]
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/a5c72303-800f-4544-bec4-6035be8c8a91" />

4. **Verificação**:
Monitorei o status até que estivesse `running`, recuperei o novo DNS público e acessei o site para garantir que a aplicação continuava funcionando corretamente após o redimensionamento.

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/ac507023-4d28-41ac-8564-89be1c6ab079" />

<img width="1920" height="1006" alt="image" src="https://github.com/user-attachments/assets/516b8583-b050-4fd6-a492-a2cae023d6c0" />

## Tarefa 2: Usar a AWS Pricing Calculator para estimar custos

Para quantificar a economia, utilizei a calculadora oficial da AWS (https://calculator.aws).

### 2.1 Estimativa Antes da Otimização

Configurei a calculadora com o cenário original:

* **EC2**: 1 instância `t3.small` (Linux, On-Demand) + 40 GB EBS.

* **RDS**: 1 instância `db.t3.micro` + 20 GB Storage.

* **Custo Mensal Estimado**: ~$74.04

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/9fab3938-45f6-4588-a66b-19b604a8bdff" />

[Link da primeira estimativa](https://calculator.aws/#/estimate?id=fe062e678e5b19ae1948cd96e1d951af66e220e8)

### 2.2 Estimativa Depois da Otimização

Ajustei a configuração para refletir as mudanças realizadas (e a potencial redução de armazenamento):

* **EC2**: 1 instância `t3.micro` (Linux, On-Demand) + 20 GB EBS.

* **RDS**: Mantido igual.

* **Custo Mensal Estimado**: ~$64.45

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/4f60a74a-bcb6-4bba-9251-a11bbca78ee1" />

[Link da segunda estimativa](https://calculator.aws/#/estimate?id=ddd57c62be0f475dd18086eb6434ecc083c80159)

### 2.3 Resultado da Economia

Comparando os dois cenários, a otimização resultou em uma economia mensal estimada de aproximadamente $9.59. Embora pareça pouco em uma única instância, essa prática gera grandes economias em escala.
