# Lab - Gerenciando Recursos com Tags

Neste laboratório, utilizei a AWS CLI e scripts de automação (Shell e PHP) para gerenciar um grande número de instâncias EC2 com base em suas **Tags**. O cenário envolvia filtrar recursos por projeto, atualizar tags em massa, parar/iniciar ambientes de desenvolvimento e, no desafio final, terminar instâncias que não estavam em conformidade com as políticas de segurança.

A arquitetura consiste em um **Command Host** para administração e várias instâncias privadas marcadas com tags como `Project`, `Environment` e `Version`.

<img width="853" height="339" alt="image" src="https://github.com/user-attachments/assets/69ae0a5d-7405-42be-8187-58ff8846def2" />

## Tarefa 1: Usando Tags para Gerenciar Recursos

Conectei-me ao **Command Host** para executar os comandos de gerenciamento.

### 1.1 Filtrando Instâncias com JMESPath

Utilizei a AWS CLI para encontrar instâncias do projeto `ERPSystem`. Para tornar a saída legível, usei o parâmetro `--query` com sintaxe JMESPath, extraindo apenas ID, AZ, Projeto, Ambiente e Versão.

Comando final de visualização:

```bash
aws ec2 describe-instances --filter "Name=tag:Project,Values=ERPSystem" --query 'Reservations[*].Instances[*].{ID:InstanceId,AZ:Placement.AvailabilityZone,Project:Tags[?Key==`Project`] | [0].Value,Environment:Tags[?Key==`Environment`] | [0].Value,Version:Tags[?Key==`Version`] | [0].Value}'
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/64fe5529-02b9-4e72-b1d6-21118dbb4730" />

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/97f419c9-8f2d-4aad-a9c7-9e99dba77c78" />

### 1.2 Atualização em Massa

Precisava atualizar a tag `Version` de `1.0` para `1.1` apenas nas instâncias de development. Em vez de fazer manualmente, usei o script `change-resource-tags.sh` que automatiza essa busca e atualização.

```bash
./change-resource-tags.sh
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/6d730c41-722c-46c4-a9e5-8eda669c60bb" />

Validei a mudança rodando o comando de filtro novamente e confirmando que as instâncias de desenvolvimento agora mostravam versão 1.1.

## Tarefa 2: Parar e Iniciar Recursos por Tag

Para economizar custos, usei um script PHP (`stopinator.php`) que interage com a AWS SDK para parar ambientes específicos.

1. **Parar o Ambiente de Desenvolvimento**:
Executei o script filtrando pelo projeto e ambiente:

```bash
./stopinator.php -t"Project=ERPSystem;Environment=development"
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/5b9b5364-a336-472e-9ce2-18cf7865857e" />

Verifiquei no console que as duas instâncias de desenvolvimento foram paradas.

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/aa9886b6-a78b-4a71-a498-dddd2eae0e1c" />

2. **Reiniciar o Ambiente**:
Usei o mesmo script com a flag `-s` para iniciar as instâncias novamente:

```bash
./stopinator.php -t"Project=ERPSystem;Environment=development" -s
```

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/fbfad269-7b84-42e8-a0b9-a280ca08fbd8" />

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/d6ab5952-3203-40d9-8e64-5b3e4764fa0e" />

## Tarefa 3 (Desafio): Terminar Instâncias Não Conformes

O desafio era criar um processo para terminar automaticamente qualquer instância na sub-rede privada que não possuísse a tag `Environment` definida (política "Tag-or-Terminate").

### 3.1 Preparação do Cenário

Para testar o script, fui ao Console EC2 e removi manualmente a tag `Environment` de duas instâncias na sub-rede privada.

<img width="1920" height="963" alt="image" src="https://github.com/user-attachments/assets/ca0b8319-4ef6-41f4-a359-f9b16b8470d2" />

### 3.2 Execução da Remediação

Recuperei o ID da Subnet privada e a Região atual. Em seguida, executei o script de auditoria `terminate-instances.php`:

```bash
./terminate-instances.php -region [REGIAO-ATUAL] -subnetid [ID-DA-SUBNET]
```

O script escaneou a sub-rede, identificou as instâncias que eu havia modificado (sem a tag Environment) e as terminou automaticamente.

<img width="908" height="663" alt="image" src="https://github.com/user-attachments/assets/c4318154-16e4-41c6-abcf-39c3c202533d" />
