# PROJETO LINUX/AWS (COMPASS-UOL)
##### Versão 1.0
##### Por Karl Richard
## Introdução

Neste documento vamos encontrar o processo de instalação e configuração de um ambiente Linux na AWS. Onde será configurado um Network File System e um Apache com alguns scripts de monitoramento dos serviços.

## Criando ambiente Linux

Começando pela AWS, vamos gerar uma instância EC2 a partir de uma imagem Amazon Linux 2 com as características t3.small, 16GB de SSD e com as portas de comunicação para acesso público 22, 111, 2049, 80 e 443.

Selecionando AMI:

![image](https://github.com/user-attachments/assets/fe9f3acd-cb3d-47c0-9848-8d75deca6b63)

Definindo o tipo de instância:

![image](https://github.com/user-attachments/assets/860cd064-5d23-4a65-9124-1aca6eadcd0a)

No momento da seleção das especificações da instância, em tempo, criei uma VPC nova, com uma subnet pública, com uma tabela de rotas para um gateway de internet para dessa forma a instância ter rota para a internet.
E juntamente, criei um security group com as regras inbound para as devidas portas citadas anteriormente serem liberadas.
Antes de lançar a instância também gerei um arquivo .ppk para poder acessá-la através de uma conexão SSH.

Abaixo, as configurações de rede: 

![image](https://github.com/user-attachments/assets/64c1f5c9-31da-41e0-bc11-ce6ba2f4db01)

Disco necessário:

![image](https://github.com/user-attachments/assets/abe01b43-c26e-4ba5-9230-7d5e71832e63)

Com a instância criada, executei o teste de conexão via Putty, onde consegui abrir uma sessão com o user ec2-user. Abaixo, imagem evidenciando a conexão bem sucedida:

![image](https://github.com/user-attachments/assets/7ac8a294-cf3d-4438-95ce-85567dd36ccf)


Após esse procedimento, vinculei um Elastic IP para a instância com o IP 44.221.5.127 para poder acessar através deste IP.

Abaixo, o sumário do IP elástico configurado:

![image](https://github.com/user-attachments/assets/6aecea77-770e-4fab-bdc3-0c0e89f05770)


Com todos os requisitos da instância configurados e com o Linux operante, prossegui para o próximo objetivo que é configurar um NFS e um Apache com alguns scripts.

## Configurando NFS com um diretório com o nome "karl"

Diferente de outras distribuições como Debian e Ubuntu, no Amazon Linux 2 o serviço Network File System já vem instalado. Nesse caso, foi necessário somente criar um diretório com o nome "karl".
O caminho para o diretório é ```/mnt/karl/```

Abaixo, segue imagem evidenciando o serviço NFS online:

![image](https://github.com/user-attachments/assets/f31881fc-d88c-487f-a2d4-2a8e3b4b525d)

Por questões de segurança, como está requisitado um diretório específico para a saída do script que vamos configurar em seguida, nesse diretório criado, acrescentei mais dois diretórios, o "scripts" e "scripts_logs". Dessa forma, o caminho que vou compartilhar será o "scripts_logs" para estar acessível para quem for acessar, onde o usuário não terá acesso aos comandos do script que estará dentro do diretório "scripts".
Agora, com o diretório no NFS configurado, publiquei o caminho ```/mnt/karl/scripts_logs``` através do arquivo ```/etc/exports``` digitando o comando ```/mnt/karl/scripts_logs *(rw,sync,no_subtree_check,no_root_squash,insecure)``` para estar 
acessível por qualquer IP. 

Abaixo uma imagem demonstrando o arquivo publicado e devidamente montado em uma outra máquina:

![image](https://github.com/user-attachments/assets/535b0ff1-4a1c-4782-a848-74aa2543c7a0)

Montagem em outra máquina Linux:

![image](https://github.com/user-attachments/assets/538abdfb-1c7d-4248-b7bc-498bcb9b1368)

A montagem do NFS foi feita em um Ubuntu através do comando ```sudo mount 44.221.5.127:/mnt/karl /mnt/karl```.

## Subindo o Apache

Como próximo tópico, temos que configurar um Apache nessa mesma instância. Vamos começar instalando e executando o serviço.

Para instalar, vamos usar o comando: ```sudo yum install httpd``` e em seguida clicando Y para aceitar a instalação dos pacotes. 

Com o Apache instalado, vamos executar ele através do comando: ```sudo systemctl start httpd```. E para confirmar se ele está online, podemos executar o comando ```sudo systemctl status httpd``` onde ele irá mostrar a mensagem RUNNING caso estiver rodando corretamente.

Segue imagem exemplo abaixo do serviço que subi na instância:

![image](https://github.com/user-attachments/assets/6b17e679-354c-438d-816b-0652c3e1c7c5)

Após esse procedimento e com o status RUNNING, podemos testar a página do Apache em um navegador através do Elastic IP da instância. Onde ele irá apresentar essa página, caso estiver tudo certo:

![image](https://github.com/user-attachments/assets/ef88760f-e2e5-41ed-976c-3e5af54be390)

Detalhe importante sobre os dois serviços, ao iniciar a instância, os serviços do Apache e NFS não estarão rodando. Por esse motivo, também configurei eles para iniciarem junto com o sistema através do comando:
```sudo systemctl enable httpd``` e ```sudo systemctl enable nfs-server.service```.

## Trabalhando com scripts de estado dos serviços

Pronto! Agora que temos os dois serviços online e acessíveis, vamos criar um script que valide a cada 5 minutos se o Apache está online, onde será apresentando a data, hora, nome do serviço, status e uma mensagem personalizada.
Detalhe, esse script deverá enviar dois arquivos de saída (offline e online) com o resultado para o diretório compartilhado no NFS.

Comecei criando um diretório ```scripts``` onde irei armazenar os scripts, e o diretório ```scripts_logs``` onde irei armazenar os arquivos de saída dos scripts. Ambos criados no diretório do NFS em ```/mnt/karl/```.

Comandos: ```mkdir /mnt/karl/scripts``` e ```mkdir /mnt/karl/scripts_logs```.

Agora, dentro do diretório "scripts", criei um arquivo bash (.sh) com as seguintes instruções:

- Para verificar o nome do serviço:
```
SERVICE="httpd"
```

- Verificar data e hora atuais:
```
DATE_TIME=$(date '+%Y-%m-%d %H:%M:%S')
```

- Define onde serão salvos os arquivos gerados : 
```
LOG_DIR="/mnt/karl/scripts_logs/"
ONLINE_FILE="$LOG_DIR/apache_online.log"
OFFLINE_FILE="$LOG_DIR/apache_offline.log"
```

- Função que verifica o serviço:
```
check_service() {
    local service_name=$1
    local log_file=$2
    local service_status
```

- Obtendo o status do serviço:
Para online:
```
if systemctl is-active --quiet "$service_name"; then
        service_status="ONLINE"
        {
            echo "  Data e Hora: $DATE_TIME"
            echo "  Serviço: $service_name"
            echo "  Status: $service_status"
            echo "  Mensagem personalizada:
```

Para offline:
```
  } >> "$log_file"
    else
        service_status="OFFLINE"
        {
            echo "  Data e Hora: $DATE_TIME"
            echo "  Serviço: $service_name"
            echo "  Status: $service_status"
            echo "  Mensagem personalizada:
```
Com o script montado, salvei ele como ```apache_nfs_status.sh``` no diretório ```/mnt/karl/scripts``` e configurei a permissão de execução do script através do comando:
```
chmod +x /mnt/karl/scripts/apache_nfs_status.sh
```

Personalizei a mensagem de resultado com um esquema de fonte onde mostra uma mensagem de Serviço online ou offline.

- Agora, finalizamos com a verificação do status do serviço e registra no arquivo correspondente:
```
if systemctl is-active --quiet $SERVICE; then
    check_service $SERVICE $ONLINE_FILE
else
    check_service $SERVICE $OFFLINE_FILE
fi
```

Após finalizar o script, salvei e mudei a permissão dele para executável através do comando ```chmod +x apache_status.sh```, e para testar o funcionamento do script, executei o mesmo através do comando ```./apache_status.sh```. Onde foi gerado um arquivo de saída .log no diretório com o data, hora, status, serviço e uma mensagem personalizada de online ou offline.

Abaixo, veja imagens do resultado com o Apache online e offline:

Quando está ONLINE:

![image](https://github.com/user-attachments/assets/7b1bb00e-765a-44c7-a5b6-f6e26879e05d)

Quando está OFFLINE:

![image](https://github.com/user-attachments/assets/a2076ac0-b902-4ba8-860c-808c61fbf820)


## Concluíndo o projeto

Após todos esses processos, agora temos um ambiente público Linux com serviço de NFS e Apache online e operante para subir qualquer tipo de serviço e um monitoramento básico para verificar se está UP ou DOWN. E com esse README e seus versionamentos, podemos subir esse mesmo ambiente em outras instâncias caso for necessário e também podemos aprimorá-lo, pois agora teremos um passo a passo bem detalhado sobre todos os comandos e requisitos para configurar tudo corretamente e suas versões para cada atualização e melhoria que for aplicada.
