# Atividade-Docker-Compass
Tarefa desenvolvida no âmbito do estágio da Compass-UOL DevSecOps AWS.

## Objetivo da atividade:
Desenvolver um guia prático para criação e configuração de recursos na cloud computing da AWS, atendendo aos requisitos descritos no arquivo **“requisitos.pdf”** deste repositório:

## Configurações prévias ⚙️
Para solucionar a atividade é necessário criar e configurar alguns recursos da AWS, tal como descrito abaixo:

**VPC** ☁️

- No painel VPC da AWS, clicar em **“Suas VPCs”** no menu lateral esquerdo
- Em seguida, clicar no botão **“criar VPC”**
- Na nova janela, selecionar a opção **“somente VPC”**
- Nomear a VPC
- Selecionar entrada manual de CIDR
- Indicar o CIDR IPv4
- Selecionar a opção de nenhum bloco CIDR IPv6
- Manter a locação como **“padrão”**
- Por fim, clicar no botão **“criar VPC”** para confirmar.

**Sub-rede**
- No painel VPC da AWS, clicar em **“Sub-redes”** no menu lateral esquerdo
- Em seguida, clicar no botão **“criar sub-rede”**
- Na nova janela, selecionar a opção correspondente à VPC criada anteriormente
- Criar duas sub-rede diferentes e os respectivos blocos CIDR IPv4 
- Importante indicar zonas de disponibilidades diferentes para cada sub-rede
- Por fim, clicar no botão **“criar sub-rede”** para confirmar.

**Tabela de rotas** 
-	No painel VPC da AWS, clicar em **“tabela de rotas”** no menu lateral esquerdo
-	Em seguida, clicar no botão **“criar tabela de rotas”**
-	Na nova janela, criar o nome da tabela de rotas 
-	Selecionar a opção correspondente à VPC criada anteriormente
-	Clicar no botão **“criar tabela de rotas”** para confirmar
-	Selecionar a tabela de rotas criada
-	Clicar em **"Ações"** > **"Editar rotas"**
-	Clicar em **"Adicionar rota"**
-	Configurar da seguinte forma:
    - Destino: 0.0.0.0/0
    - Alvo: Selecionar o gateway de internet criado anteriormente
-	Clicar em **"Salvar alterações"**.

**Gateway de internet**
-	No painel VPC da AWS, clicar em **“Gateways da internet”** no menu lateral esquerdo
-	Em seguida, clicar no botão **“criar gateway da internet”**
-	Na nova janela, criar o nome do gateway
-	 Clicar no botão **“criar gateway da internet”** para confirmar
-	Selecionar o internet gateway criado
-	Clicar em **"Ações"** > **"Associar à VPC"**
---
### Criar Security Groups e configurar as regras de segurança 🖥️

**SG-LOAD BALANCER**

- Regra de entrada
  
  | Type         | Protocol | Port Range | Source Type | Source      |
  |--------------|----------|------------|-------------|-------------|
  | HTTP         | TCP      | 80         | Anywhere    | 0.0.0.0/0   |
 
**SG-INST_EC2**

- Regra de entrada
  
  | Type         | Protocol | Port Range | Source Type |  Source          |
  |--------------|----------|------------|-------------|------------------|
  | SSH          | TCP      | 22         | Anywhere    | 0.0.0.0/0        |
  | HTTP         | TCP      | 80         | Custom      | SG-LOAD BALANCER |
  | NFS          | TCP      | 2049       | Anywhere    | 0.0.0.0/0   |
  

**SG-RDS**

- Regra de entrada
  
  | Type         | Protocol | Port Range | Source Type | Source      |
  |--------------|----------|------------|-------------|-------------|
  | MYSQL/Aurora | TCP      | 3306       | Custom      | SG-INST_EC2 |

**SG-EFS**
  | Type         | Protocol | Port Range | Source Type | Source      |
  |--------------|----------|------------|-------------|-------------|
  | NFS          | TCP      | 2049       | Custom      | SG-INST_EC2 |

### Banco de Dados - Amazon Relational Database Service (RDS)
- No painel RDS da AWS, clicar em **“Banco de Dados”** no menu lateral esquerdo
- Em seguida, clicar no botão **“criar banco de dados”**
- Na nova janela, selecionar a opção "criação padrão"
- Opções de mecanismo > MySQL;
- Configurações > Nome do BD > Autogerenciada > Senha do BD
- Configuração da Instância > db.t3.micro;
- Conectividade > Não se conectar em nenhum recurso de computação do EC2 > IPV4 > Sua VPC > Grupo de Segurança Existente (SG-RDS) > Sem Preferência;
- Por fim, clicar no botão **“criar banco de dados”** para confirmar.

### Sistema de Arquivos - Amazon Elastic File System (EFS)
- No painel EFS da AWS, clicar em **“Sistema de arquivos”** no menu lateral esquerdo
- Em seguida, clicar no botão **“criar sistema de arquivos”**
- Na nova janela, indicar o nome desejado e selecionar a VPC criada anteriormente
- clique em "personalizar". Na etapa de acesso à rede, confira os dados do ponto de montagem, a fim de que constem as duas sub-redes e o security group criados anteriormente;
- Por fim, clicar no botão **“criar”** para confirmar.
  
### Criar 2 instâncias EC2 com o sistema operacional Ubuntu (Família t2.micro) 🖥️

- No painel EC2 da AWS, clicar em **"Instâncias"** no menu lateral esquerdo
-	Em seguida, clicar no botão **"Executar instâncias"**
-	Configurar as Tags (Name, Project e CostCenter) para instâncias e volumes
-	Selecionar a imagem Ubuntu
-	Selecionar o tipo de instância t2.micro
-	Escolher a chave gerada anteriormente
-	No item **“configurações de rede”**, clicar em **"Editar"**
-	Selecionar a rede e sub-rede criadas anteriormente
-	Habilitar a opção **“atribuir IP público automaticamente”**
-	Selecionar o grupo de segurança criado anteriormente (SG-INST-EC2)
- Em detalhes avançados, no campo "dados do usuário", inserir o script start instance (user_data.sh) que contém o Docker compose e a montagem do NFS, conforme abaixo;

```
#!/bin/bash

# Atualiza os pacotes e instala Docker e NFS 
sudo apt update -y
sudo apt install -y docker.io nfs-common 

# Inicia o serviço Docker e ativa-o para iniciar automaticamente
sudo systemctl start docker
sudo systemctl enable docker

# Monta o EFS
sudo mkdir /mnt/efs
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-0fddc4b20fad08761.efs.us-east-1.amazonaws.com:/ /mnt/efs


# Instala o Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Cria o arquivo Docker Compose para Wordpress e MySQL 
sudo tee /mnt/efs/docker-compose.yml > /dev/null <<EOF
version: '3.1'

services:

  wordpress:
    image: wordpress:latest
    restart: always
    ports:
      - 80:80
    environment:
      WORDPRESS_DB_HOST: dbprojeto.c3guukiom6qb.us-east-1.rds.amazonaws.com
      WORDPRESS_DB_USER: *******
      WORDPRESS_DB_PASSWORD: *******
      WORDPRESS_DB_NAME: dbprojeto
    volumes:
      - /mnt/efs:/var/www/html

  db:
    image: mysql:latest
    restart: always
    environment:
      MYSQL_DATABASE: dbprojeto
      MYSQL_USER: *******
      MYSQL_PASSWORD: *******
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
EOF

if ! grep -q "fs-0fddc4b20fad08761.efs.us-east-1.amazonaws.com:" /etc/fstab; then
    echo "fs-0fddc4b20fad08761.efs.us-east-1.amazonaws.com:/ /mnt/efs nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport 0 0" | sudo
tee -a /etc/fstab
fi

# Inicia o Docker Compose
cd /mnt/efs
sudo docker-compose up -d

```
-	Clicar em **"Executar instância"**.
-	Repita os mesmos procedimentos acima, porém, a segunda instância deve ser criada na segunda sub-rede, **garantido que esteja numa zona de disponibilidade diferente da primeira**  
---
### AMI
- Selecione qualquer uma das instâncias EC2 criandas anteriomente;
- clique no botão **"açoes"** e, em seguida, **"imagens e modelos"** > **"criar imagem"**;
- Na nova janela, defina o nome e as configurações da AMI;
- Por fim, clique em **"criar imagem"** para confirmar.
  
### MODELO DE EXECUÇÃO 
- No painel EC2 da AWS, clicar em **"Modelos de Execução"** no menu lateral esquerdo
-	Em seguida, clicar no botão **"criar modelo de execução"**
- Nomeie o template e siga os mesmo passos de criação da instância EC2, inclusive, colocando o script de inicialização no campo "dados do usuário";
  
### GRUPO DE DESTINO (Target Group)
- No painel EC2 da AWS, clicar em **"Grupos de destino"** no menu lateral esquerdo
-	Em seguida, clicar no botão **"criar grupo de destino"**
- Selecione a configuração Básica como "Instância"
- Atribua um nome ao TG e escolha o protocolo HTTP e porta 80
- Selecione a VPC criada anteriormente e as instâncias EC2 criadas anteriormente;
- Por fim, clique em **"criar"** para confirmar.
  
### LOAD BALANCER
- No painel EC2 da AWS, clicar em **"Load balancers"** no menu lateral esquerdo
-	Em seguida, clicar no botão **"Criar load balancer"**
- Na nova janela, selecione **APLICAÇÃO DE LOAD BALANCER** e nomeie o ALB
- escolha a opção **"Voltado para internet"**
- Selecione a VPC, as sub-redes e o Grupo de Segurança (SG-ALB) criados anteriormente
- Em listeners e roteamente, selecione a PORTA HTTP 80 e o Grupo de destino já criado;
- Por fim, clique em **"criar load balancer"** para confirmar.
  
###  AUTO SCALLING
- No painel EC2 da AWS, clicar em **"Grupos de Auto Scaling"** no menu lateral esquerdo
-	Em seguida, clicar no botão **"Criar Grupo de Auto Scaling"**
- Atribua um nome ao Auto Scaling
- selecione o modelo de execução, a VPC e as sub-redes, todos já criados
- Configurar o tamanho do grupo e ajuste de escala, conforme a necessidade, como exemplo: tamanho do grupo: 2, min: 2 e max: 4;
- Sem política.
---
###  Teste da Aplicação
- Acessar o **Load balancer** criado anteriormente e copiar o DNS;
- abrir o browser e colar o DNS no navegador;
- esperado que seja aberta a tela de login do WordPress.
