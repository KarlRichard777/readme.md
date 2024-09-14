# PROJETO LINUX/AWS (COMPASS-UOL)
##### Versão 1.0
##### Por Karl Richard
## Introdução

Neste documento vamos encontrar o processo de instalação e configuração de um ambiente Linux na AWS. Onde será configurado um Network File System e um Apache com alguns scripts de monitoramento dos serviços.

## Criando ambiente

Começando pela AWS, vamos gerar uma instância EC2 a partir de uma imagem Amazon Linux 2 com as características t3.small, 16GB de SSD e com as portas de comunicação para acesso público 22, 111, 2049, 80 e 443.

No momento da seleção das especificações da instância, em tempo, criei uma VPC nova, com uma subnet pública, com uma tabela de rotas para um gateway de internet para dessa forma a instância ter rota para a internet.
E juntamente, criei um security group com as regras inbound para as devidas portas citadas anteriormente serem liberadas.
Antes de lançar a instância também gerei um arquivo .ppk para poder acessá-la através de uma conexão SSH.

Com a instância criada, executei o teste de conexão via Putty, onde consegui abrir uma sessão com o user ec2-user. Abaixo, imagem evidenciando a conexão bem sucedida.

Após esse procedimento, vinculei um Elastic IP para a instância com o IP 44.221.5.127 para poder acessar através deste IP.

Com todos os requisitos da instância configurados, prossegui para o próximo objetivo que é configurar um NFS e um Apache com alguns scripts.

## Configurando NFS com um diretório com o nome "karl"

Diferente de outras distribuições como Debian e Ubuntu, no Amazon Linux 2 o serviço Network File System já vem instalado. Nesse caso, foi necessário somente criar um diretório com o nome "karl".
O caminho para o diretório é ```/mnt/karl/```

Abaixo, segue imagem evidenciando o serviço NFS online:

![image](https://github.com/user-attachments/assets/f31881fc-d88c-487f-a2d4-2a8e3b4b525d)

Agora, com o diretório no NFS configurado, publiquei o caminho ```/mnt/karl``` através do arquivo ```/etc/exports``` digitando o comando ```/mnt/karl *(rw,sync,no_subtree_check,no_root_squash,insecure)``` para estar 
acessível por qualquer IP. 

Abaixo uma imagem demonstrando o arquivo publicado e devidamente montado em uma outra máquina:

![image](https://github.com/user-attachments/assets/535b0ff1-4a1c-4782-a848-74aa2543c7a0)

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

Pronto! Agora que temos os dois serviços online e acessíveis, vamos criar um script que valide a cada 5 minutos se eles estão online, onde estará apresentando a data, hora, nome do serviço, status e uma mensagem personalizada.
Detalhe, esse script deverá enviar o arquivo de saída com o resultado para o diretório compartilhado no NFS.

Comecei criando um diretório ```scripts``` onde irei armazenar os scripts, e o diretório ```scripts_logs``` onde irei armazenar os arquivos de saída dos scripts. Ambos criados no diretório do NFS em ```/mnt/karl/```.

Comandos: ```mkdir /mnt/karl/scripts``` e ```mkdir /mnt/karl/scripts_logs```.

Agora, dentro do diretório "scripts", criei um arquivo bash (.sh) com as seguintes instruções:

- Para definir o diretório de logs e definir o nome do arquivo de saída:
```
LOG_DIR="/mnt/karl/scripts_logs"
LOG_FILE="${LOG_DIR}/services_status_$(date +'%Y-%m-%d').log"
```

- Função para verificar o status de um serviço:
```
check_service() {
    local service_name=$1
    local service_display_name=$2
    local service_status
}
```

- Verificar o status do serviço: 
```
if systemctl is-active --quiet "$service_name"; then
        service_status="ONLINE"
        echo "$(date +'%Y-%m-%d %H:%M:%S') - Serviço: $service_display_name - Status: $service_status - O serviço está funcionando corretamente." >> "$LOG_FILE"
    else
        service_status="OFFLINE"
        echo "$(date +'%Y-%m-%d %H:%M:%S') - Serviço: $service_display_name - Status: $service_status - ATENÇÃO: O serviço não está em execução!" >> "$LOG_FILE"
    fi
```

- E por fim, verificar os serviços do Apache e NFS:
```
check_service "httpd" "Apache"
check_service "nfs-server" "NFS"
```

Com o script montado, salvei ele como ```apache_nfs_status.sh``` no diretório ```/mnt/karl/scripts``` e configurei a permissão de execução do script através do comando:
```
chmod +x /mnt/karl/scripts/apache_nfs_status.sh
```

Para testar o funcionamento do script, executei o mesmo através do comando ```./apache_nfs_status.sh```. Onde o mesmo gerou um arquivo de saída .log no diretório com o seguinte conteúdo:

