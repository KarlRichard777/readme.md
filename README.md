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

Após esse procedimento, vinculei um Elastic IP para a instância com o IP x.x.x.x para poder acessar através deste IP.

Com todos os requisitos da instância configurados, prossegui para o próximo objetivo que é configurar um NFS e um Apache com alguns scripts.

## Configurando NFS com um diretório com o nome "karl"

Diferente de outras distribuições como Debian e Ubuntu, no Amazon Linux 2 o serviço Network File System já vem instalado. Nesse caso, foi necessário somente criar um diretório com o nome "karl".
O caminho para o diretório é ```/mnt/karl/```

Abaixo, segue imagem evidenciando o serviço NFS online:
![image](https://github.com/user-attachments/assets/f31881fc-d88c-487f-a2d4-2a8e3b4b525d)

Detalhe, ao iniciar a instância, os serviços do Apache e NFS não estarão rodando. Por esse motivo, também configurei eles para iniciarem junto com o sistema através do comando:
```sudo systemctl enable httpd``` e ```sudo systemctl enable nfs-server.service```.

Agora, com o diretório no NFS configurado, 
