# Udacity Full-Stack Developer (Projeto 6) - Configuração do Servidor Linux

Este projeto consiste em preparar uma instalação de referência de um servidor Linux para hospedar aplicações web.

Dentre as tarefas de configuração:
- oferecer segurança no servidor contra um número de vetores de ataque
- instalar e configurar um servidor de banco de dados
- Fazer deploy da aplicação [Item Calalog](https://github.com/oliveira-marcio/fsnd-item-catalog) (projeto também realizado durante o curso).

**OBS:** A aplicação Item Calog oferece autenticação tanto pelo Google quanto Facebook, mas por conta das alterações de 16/10/2018 nas políticas de autenticação da API de Login do Facebook (detalhes [aqui](https://developers.facebook.com/blog/post/2018/06/08/enforce-https-facebook-login/)), que passou a exigir que as aplicações utilizem HTTPS, não será possível utilizar a autenticação do Facebook nesse momento, visto que o escopo deste projeto de configuração do servidor requer que apenas as portas HTTP(80), SSH(2200) e NTP(123) estejam abertas.

## Endereço da aplicação:

http://35.175.176.232.xip.io

**OBS:** Foi utilizado o serviço da [xip.io](http://xip.io/) para criar rapidamente um nome de domínio para a aplicação, pois é exigido para o funcionamento do login por autenticação da Google/Facebook utilizado pela aplicação.

## Acesso ao servidor:
```
$ ssh grader@35.175.176.232 -p 2200 -i <path>/<chave privada>
```
**OBS:** A chave privada é requerida para autenticação do usuário `grader`.

## Tarefas Realizadas

#### 1) Criação da instância Linux

Instância Ubuntu criada no serviço [AWS Lightsail](https://aws.amazon.com/pt/lightsail/).
```
S.O.: Ubuntu 18.04 Lightsail
IP: 35.175.176.232
Porta: 22 (SSH)
```

#### 2) Configuração da Timezone (UTC)
```
sudo dpkg-reconfigure tzdata
```
Foi selecionado `"None of the above"` e depois `UTC`.

#### 3) Atualização de pacotes

```
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get autoremove
```

#### 4) Desabilitar login remoto do usuário `root`, autenticação por login/senha e alteração da porta SSH padrão
- Edição do arquivo de configuração do SSHD
```
$ sudo nano /etc/ssh/sshd_config
```
- Parâmetros configurados:
```
PermitRootLogin no
PasswordAuthentication no
Port 2200
```
**OBS:** Também foi necessário adicionar porta 2200 no Firewall do Lightsail

#### 5) Configuração de portas e Firewall (UFW)
```
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 2200/tcp
$ sudo ufw allow ntp
$ sudo ufw allow www
$ sudo ufw enable
$ sudo service ssh restart
$ sudo service sshd restart
```
**OBS:** Após esse ponto não é mais possível acessar o servidor pelo AWS Console (que utiliza a porta SSH  padrão, 22). Foi necessário usar um terminal externo (Ex: [Git Bash](https://gitforwindows.org/)) usando a porta 2200 e a chave privada disponibilizada para download no AWS Console para o usuário padrão (`ubuntu`).
```
$ ssh ubuntu@35.175.176.232 -p 2200 -i <path>/<chave privada>.pem
```

#### 6) Criação do usuário `grader`
- Criação do usuário
```
$ sudo adduser grader
```

- Atribuição de privilégios root
```
$ sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader
$ sudo nano /etc/sudoers.d/grader
```
**OBS**: Alterado usuário "ubuntu" para "grader" no arquivo
```
grader ALL=(ALL) NOPASSWD:ALL
```

#### 7) Criação e instalação do par de chaves de autenticação para `grader`
- No terminal local (criação do par de chaves):
```
$ ssh-keygen
$ cat <path>/<filename>.pub
# Copiado o conteúdo da chave pública
```

- No servidor (instalação da chave pública e permissões definidas):
```
$ cd /home/grader
$ sudo mkdir .ssh
$ sudo touch .ssh/authorized_keys
$ sudo nano .ssh/authorized_keys
# Colado conteúdo da chave pública no arquivo acima
$ sudo chmod 700 .ssh
$ sudo chmod 644 .ssh/authorized_keys
$ sudo chown grader .ssh
$ sudo chgrp grader .ssh
$ sudo chown grader .ssh/authorized_keys
$ sudo chgrp grader .ssh/authorized_keys
```

#### 8) instalação de pacotes e libraries Python necessárias

Apache, WSGI, PostgreSQL, Git, PIP, Flask, SQLAlchemy, oauth2client, requests, psycopg2
```
$ sudo apt-get install apache2
$ sudo apt-get install libapache2-mod-wsgi
$ sudo apt-get install postgresql
$ sudo apt-get install postgresql-contrib
$ sudo apt-get install git
$ sudo apt-get install python-pip
$ sudo pip install flask
$ sudo pip install sqlalchemy
$ sudo pip install sqlalchemy-citext
$ sudo pip install oauth2client
$ sudo pip install requests
$ sudo pip install psycopg2
$ sudo pip install psycopg2-binary
```

#### 9) Clonanda a aplicação [Item Catalog](https://github.com/oliveira-marcio/fsnd-item-catalog)

```
$ cd /var/www
$ sudo mkdir item-catalog
$ sudo git clone https://github.com/oliveira-marcio/fsnd-item-catalog.git item-catalog
```
#### 10) Configuração do Servidor Web (Apache)

- Criado arquivo de configuração WSGI da aplicação
```
$ cd item-catalog
$ sudo nano item-catalog.wsgi
```
- Conteúdo do arquivo acima:
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, '/var/www/item-catalog')
from application import app as application
application.secret_key='super_secret_key'
```
- Configuração do Apache para lidar com requisições usando o módulo WSGI
```
$ sudo nano /etc/apache2/sites-enabled/000-default.conf
```
- Conteúdo do arquivo acima:
```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    WSGIScriptAlias / /var/www/item-catalog/item-catalog.wsgi
    #Allow Apache to serve the WSGI app from our catalog directory
    <Directory /var/www/item-catalog>
         Order allow,deny
         Allow from all
    </Directory>
    #Allow Apache to deploy static content
    <Directory /var/www/item-catalog/static>
         Order allow,deny
         Allow from all
    </Directory>
</VirtualHost>
```

#### 11) Configuração do Banco de Dados (PostgreSQL)
- Revogados os acessos remotos ao banco de dados (checado se todos os tipos são `local` e apenas as conexões IPV4 e IPV6 do host local são do tipo `host`. Replicações já estavam comentadas)
```
sudo nano /etc/postgresql/10/main/pg_hba.conf
```
- Criação do usuário de banco (`catalog`) e do banco de dados da aplicação (`catalog`) com o usuário `postgres`:
```
$ sudo su - postgres
$ createuser --pwprompt catalog
$ createdb -O catalog catalog
```
- Criação da extenção CITEXT para permitir criação de colunas _case insensitive_.
**OBS:** Requerido porque na versão original do app foi utilizado o recurso `COLLATION="NOCASE"` em algumas colunas no SQLite que não é suportado pelo PostgreSQL.
```
$ psql
postgres=# CREATE EXTENSION IF NOT EXISTS citext WITH SCHEMA public;
postgres=# \q
$ exit
```

#### 12) Configuração dos arquivos de autenticação da aplicação
Criada as chaves de API necessárias conforme [README](https://github.com/oliveira-marcio/fsnd-item-catalog/blob/master/README.md) da aplicação.

**OBS:**
Por ser um servidor de produção, foi necessário adicionar nos respectivos Developer Consoles (Google/Facebook) o domínio e as origens JavaScript e URI's de redirecionamento autorizadas:
 - xip.io (domínio)
 - http://35.175.176.232.xip.io (origem JavaScript/URI de redirecionamento)
 - http://35.175.176.232.xip.io/gconnect (URI de redirecionamento)


Instalado os arquivos de autenticação no servidor
```
$ cd /var/www/item-catalog
$ sudo nano client_secrets.json
$ sudo nano fb_client_secrets.json
```
E copiado os respectivos conteúdos de cada chave.

#### 13) Alterações na aplicação:

- Conexões do banco de dados (migração de SQLite para PostgreSQL)
```
$ cd /var/www/item-catalog
$ sudo nano database_setup.py
$ sudo nano models.py
$ sudo nano server.py
```
 Linha original (em todos os arquivos acima):
```
engine = create_engine("sqlite:///catalog.db",
                       connect_args={"check_same_thread": False})
```
 Linha modificada:
```
engine = create_engine("postgresql://catalog:catalog@localhost/catalog")
```
- Configuração do "_case insentive_" de colunas
```
$ sudo nano models.py
```
Adicionar suporte ao `citext`
```
from citext import CIText
```
Migrar `collation` para `citext`:

 Código original (classes `Categories` e `Items`)
```
Column(String(100, collation="NOCASE"), nullable=False)
```
 Código modificado:
 ```
 Column(CIText(), nullable=False)
 ```
- Acesso aos arquivos de autenticação
```
$ sudo nano utils.py
```
Alterar todas as ocorrências de:
```
"client_secrets.json"
"fb_client_secrets.json"
```
por caminhos completos:
```
"/var/www/item-catalog/client_secrets.json"
"/var/www/item-catalog/fb_client_secrets.json"
```

#### 14) Reinicio do servidor Apache para aplicar todas as configurações
```
$ sudo apache2ctl restart
```
## Referências:

- [Udacity's Configuring Linux Web Servers](https://br.udacity.com/course/configuring-linux-web-servers--ud299)
- [HOW DO I DISABLE SSH LOGIN FOR THE ROOT USER?](https://mediatemple.net/community/products/dv/204643810/how-do-i-disable-ssh-login-for-the-root-user)
- [Flask - mod_wsgi (Apache)](http://flask.pocoo.org/docs/1.0/deploying/mod_wsgi/)
- [Developping a Flask Web App with a PostreSQL Database – Making all the Possible Errors](https://blog.theodo.fr/2017/03/developping-a-flask-web-app-with-a-postresql-database-making-all-the-possible-errors/)
- [How to manage PostgreSQL databases and users from the command line](https://www.a2hosting.com/kb/developer-corner/postgresql/managing-postgresql-databases-and-users-from-the-command-line)
- [Using insensitive-case columns in PostgreSQL with citext](https://nandovieira.com/using-insensitive-case-columns-in-postgresql-with-citext)
- [sqlalchemy-citext](https://github.com/mahmoudimus/sqlalchemy-citext)
