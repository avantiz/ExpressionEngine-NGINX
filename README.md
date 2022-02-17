# Como instalar o ExpressionEngine em Ubuntu 20LTS, Nginx e MariaDB, com otimização MemCached e certificado SSL

O ExpressionEngine precisa de um webserver executando PHP e MySQL. No momento em que escrevo este tutorial, ele precisa, no mínimo:

- PHP 7.0 ou mais recente, com as seguintes extensões:
	 - gd
	 - fileinfo
	 - intl
	 - mbstring

- MySQL 5.6 ou mais recente ou Percona 5.6 ou mais recente. Neste guia vamos utilizar o MySQL.
- Um servidor web usando Apache ou NGINX. Neste guia vamos utilizar o NGINX.

## Antes de começar:

Verifique a sua versão do Ubuntu

```
lsb_release -ds
# Ubuntu 20.04.2 LTS
```

Crie uma conta non-root com sudo e troque para ela:

```
adduser fulano --gecos "Fulano"
usermod -aG sudo fulano  
su - fulano
```

NOTA: troque ```fulano``` pelo seu nome de usuário

Defina o fuso horário:

```
sudo dpkg-reconfigure tzdata
```

Tenha certeza que seu sistema está atualizado:

```
sudo apt update && sudo apt upgrade -y
```

Instale os pacotes que forem necessários:

```
sudo apt install -y zip unzip curl wget git
```

# Instale o PHP

Instale o PHP, assim como as extensões que são necessárias:

```
sudo apt install php7.4-fpm php7.4-common php7.4-mysql php7.4-gmp php7.4-curl php7.4-intl php7.4-mbstring php7.4-xmlrpc php7.4-gd php7.4-bcmath php7.4-xml php7.4-cli php7.4-zip
```

Confira a versão:

```
php --version
```

Instale o PHP Imagick

```
sudo apt-get install php-imagick
```

Edite o arquivo de configuração do PHP:

```
sudo nano /etc/php/7.2/fpm/php.ini 
```

busque as seguintes variáveis e altere para os valores abaixo:

```
file_uploads = On
allow_url_fopen = On
short_open_tag = On
memory_limit = 256M
cgi.fix_pathinfo = 0
upload_max_filesize = 100M
max_execution_time = 360
```

recarregue o Nginx

```
sudo systemctl restart nginx.service
```

# Instale MariaDB

Instale o banco de dados MariaDB, a seguir:

```
sudo apt-get install mariadb-server mariadb-client
```

Após instalar o MariaDB, insira os comandos abaixo para parar, iniciar e ativar o MariaDB sempre que o servidor reiniciar :

```
sudo systemctl stop mariadb.service
sudo systemctl start mariadb.service
sudo systemctl enable mariadb.service
```

Execute o script mysql_secure_installation para incrementar sua instalação MySQL:

```
sudo mysql_secure_installation
```

Quando aparecer o prompt, responda as seguintes questões:

```
    Enter current password for root (enter for none): Dê Enter
    Set root password? [Y/n]: Y
    New password: Insira a senha
    Re-enter new password: Repita a senha
    Remove anonymous users? [Y/n]: Y
    Disallow root login remotely? [Y/n]: Y
    Remove test database and access to it? [Y/n]:  Y
    Reload privilege tables now? [Y/n]:  Y
```


# Instale o NGINX:

Instale o servidor web Nginx com o seguinte comando:

```
sudo apt install -y nginx
```

Execute os seguintes comandos para iniciar o nginx quando o servidor der boot, automaticamente

```
sudo systemctl stop nginx.service
sudo systemctl start nginx.service
sudo systemctl enable nginx.service
```

Verifique a versão:
```
sudo nginx -v
```


Agora vamos configurar o Nginx para comprimir as páginas html e arquivos, usando GZIP:

Para realizar essas alterações, abra o arquivo de configuração principal do nginx em seu editor de texto (nesse caso, o Nano):\

```
sudo nano /etc/nginx/nginx.conf
```

Localize a seção de configurações do arquivo, que deve se parecer mais ou menos com isso:

```
##
# `gzip` Settings
#
#
gzip on;
gzip_disable "msie6";

# gzip_vary on;
# gzip_proxied any;
# gzip_comp_level 6;
# gzip_buffers 16 8k;
# gzip_http_version 1.1;
# gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
```

Você pode verificar que por padrão, a compressão gzip está ativada, mas várias configurações adicionais estão comentadas com o sinal de comentário, portanto vamos fazer várias alterações por aqui:

- Ativar as configurações adicionais, descimentando todas as linhas comentadas;
- Adicionar a diretiva ```gzip_min_lenght 256;``` que diz para o Nginx não compactar arquivos menores que 256 bytes, otimizando nossa performance de compressão.
- Adicionar fontes, ícones e imagens SVG aos parâmetros de compressão.
Depois que realizarmos todas essas alterações, essa seção na configuração ficará algo parecido com isso:

```
##
# `gzip` Settings
#
#
gzip on;
gzip_disable "msie6";

gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_buffers 16 8k;
gzip_http_version 1.1;
gzip_min_length 256;
gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/vnd.ms-fontobject application/x-font-ttf font/opentype image/svg+xml image/x-icon;
```
 
 Salve, feche o arquivo e saia.
 Recarregue o Nginx:

 ```
 sudo systemctl reload nginx
 ```
 
 ## Limitando o tamanho do upload de arquivos para o ExpressionEngine no Nginx

Por padrão, o Nginx limita o tamanho do upload de arquivos que são feitos através dos campos de upload do ExpressioNEngine para 1MB, mas muitas vezes uma foto, um arquivo de áudio ou um vídeo ultrapassam facilmente esse limite - e é bem simples de alterar essa configuração. Abra e edite o arquivo ```/etc/nginx/nginx.conf```:

Procure pelo bloco *http* e altere o limite para 100M, por exemplo:

```
http {
    ...
    client_max_body_size 100M;
}   
```

Agora faça a mesma coisa no *server*:

```
server {
    ...
    client_max_body_size 100M;
}
```  

E no bloco *location*, que afeta uma pasta particular (uploads) de um site ou app:

```
location /uploads {
    ...
    client_max_body_size 100M;
} 
```


Salve o arquivo e dê um _restart_ no serviço Nginx com um dos comandos abaixo:

```
# systemctl restart nginx       #systemd
# service nginx restart         #sysvinit
```

 
 
# Otimizando sua instalação com o MemCached (opcional)

Agora vamos instalar e deixar seguro o memcached no Ubuntu. O memcached é um sistema de cache de objetos, armazenando temporariamente objetos na memória do servidor, retendo registros frequentes ou acessados recentemente. Com isto, reduziremos a quantidade de requisições diretas em seus bancos de dados, reduzindo a carga do sistema.

## Instale o Memcached dos repositórios oficiais

Vamos garantir que tudo esteja atualizado antes de prosseguirmos:
```
sudo apt update
```
Agora vamos instalar o pacote oficial
```
sudo apt install memcached
```
E vamos instalar também ```libmemcached-tools```, uma biblioteca que fornece várias ferramentas para trabalhar com nosso servidor Memcached
```
sudo apt install libmemcached-tools
sudo apt install net-tools
```
Feito isso, o Memcached estará instalado no servidor como um serviço, junto com ferramentas que nos permitirão testar a conectividade.

## Deixando o Memcached seguro

Abra ```/etc/memcached.conf``` com nano:

```
sudo nano /etc/memcached.conf
```
Veja se existe a seguinte linha no arquivo:

```
-l 127.0.0.1
```

Se você visualizar que a configuração padrão é ```-l 127.0.0.1``` , então não há necessidade de modificar esta linha, caso contrário modifique-a para ficar assim. Agora vamos desabilitar o UDP do Memcached para evitar _exploits_ de ataque de _DoS_. Para desativar o UDP (enquanto deixa o TCP), adicione a seguinte opção na parte final desse arquivo:

```
-U 0
```

Salve e feche o arquivo quando você terminar. Dê um restart no serviço Memcached para aplicar as alterações:

```
sudo systemctl restart memcached
```

Verifique se o memcached está ativado na interface local e recebendo as conexões TCP, digitando:

```
sudo netstat -plunt
```

Você deverá visualizar o seguinte:

```
Output
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
. . .
tcp        0      0 127.0.0.1:11211         0.0.0.0:*               LISTEN      2279/memcached
. . .
```

Isso confirma que o ```memcached``` está ativo e limitado ao ```127.0.0.1``` usando apenas TCP.

## Adicionando usuários autorizados

Para adicionar usuários autorizados em nosso serviço MemCached, vamos usar o framework Simple Authentication anda Security layer (SASL):

Para confirmar que o Memcached está ativo e sendo executado, digite o seguinte:

``` memcstat --servers="127.0.0.1" ```

Você verá uma saída semelhante a essa:

```
Output
Server: 127.0.0.1 (11211)
         pid: 2279
         uptime: 65
         time: 1546620611
         version: 1.5.6
     . . .
```

Agora vamos ativar o SASL, adicionando o parâmetro ``` -S ``` em nosso arquivo de configiuração  ``` /etc/memcached.conf. ```

``` sudo nano /etc/memcached.conf ```

E lá no final do arquivo, adicione:

```
-S
```
Agora localize e descomente a opção ``` -vv ``` nesse mesmo arquivo. Salve e feche o arquivo, dando um _restart_ no serviço  Memcached:

``` sudo systemctl restart memcached ```

Vamos dar uma olhada nos logs, para ver se o suporte a SASL foi inicializado:

``` sudo journalctl -u memcached ```

Você verá uma linha mais ou menos assim, indicando que o suporte SASL foi estartado:

Output

```
. . .
Jan 04 16:51:12 memcached systemd-memcached-wrapper[2310]: Initialized SASL.
. . .

```

Feito isso, vamos baixar o pacote ``` sasl2-bin ```, um pacote que contém programas administrativos para o o banco de dados de usuários do SASL. Isso nos permitirá criar um usuário autenticado:

```
sudo apt install sasl2-bin
```

O próximo passo é criar a pasta e o arquivo que o Memcached vai verificar pelas configurações SASL:

```
sudo mkdir /etc/sasl2
sudo nano /etc/sasl2/memcached.conf 
```

Adicione o seguinte, no seu arquivo de configuração ```memcached.conf``` :

```
mech_list: plain
log_level: 5
sasldb_path: /etc/sasl2/memcached-sasldb2
```

Agora vamos criar um banco de dados SASL com nossas credenciais de usuário (escolha o seu nome de usuário):

```
sudo saslpasswd2 -a memcached -c -f /etc/sasl2/memcached-sasldb2 meuusuario
```

Entre com sua senha e confirmação de senha e finalmente, dê ao ```memcached``` a autoridade sobre o banco de dados SASL:

```
sudo chown memcache:memcache /etc/sasl2/memcached-sasldb2
```

Dê um _restart_ do serviço do Memcached:

```
sudo systemctl restart memcached
```

Confira se tudo está funcionando, de acordo com as credenciais que nós criamos:

```
memcstat --servers="127.0.0.1" --username=meuusuario --password=sua_senha
```

E se tudo estiver, ok, veremos uma resposta semelhante à abaixo:

```
Output
Server: 127.0.0.1 (11211)
         pid: 2772
         uptime: 31
         time: 1546621072
         version: 1.5.6 Ubuntu
     . . .
```

Nosso serviço de Memcached está sendo executado com sucesso, com suporte a SASL e autenticação de usuário.

Agora é só instalar o suporte ao PHP, digitando o seguinte:

```
sudo apt-get install -y php-memcached 
```

E pronto, nosso ExpressionEngine agora terá uma maior velocidade  acessando e servindo seu banco de dados.
 

# Criando o Banco de dados do ExpressionEngine 

Efetue o login no MySQL como usuário root:

```
sudo mysql -u root -p
# Enter password:
```

Crie um novo banco de dados:

```
CREATE DATABASE seubancodedados
```

Crie um usuário para acessar esse banco de dados

```
CREATE USER 'usuariobancodedados'@'localhost' IDENTIFIED BY 'suasenhadobancodedados';
```

Agora vamos garantir ao usuário o acesso total ao banco de dados que foi criado:

```
GRANT ALL ON seubancodedados.* TO 'usuariobancodedados'@'localhost' IDENTIFIED BY 'suasenhadobancodedados' WITH GRANT OPTION;
```

Finalmente, salve suas alterações e saia:
```
FLUSH PRIVILEGES;
EXIT;
```

NOTA: Substitua ```seubancodados```, ```usuáriobancodedados``` e ```suasenhadobancodedados``` pelos seus dados. Use uma senha segura.

Agora vamos configurar o Nginx para o ExpressionEngine. Execute 
``` sudo nano /etc/nginx/sites-available/seusite.conf ``` e preencha o arquivo com a seguinte configuração:

```
server {

  listen [::]:80;
  listen 80;

  server_name seusite.com;
  root /var/www/seusite;

  index index.php;

  location / {
    index index.php;
    try_files $uri $uri/ @ee;
  }

  location @ee {
    rewrite ^(.*) /index.php?$1 last;
  }

  location ~* \.php$ {
    fastcgi_pass unix:/run/php/php7.4-fpm.sock;
    include fastcgi_params;
    fastcgi_index index.php5;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
  }

}

```

Salve o arquivo e saia com ```Control``` + ```X``` + ```Yes```

Ative a nova configuração ```expressionengine.conf``` criada, linkando o arquivo à pasta ```sites-enabled```.

```
sudo ln -s /etc/nginx/sites-available/seusite /etc/nginx/sites-enabled/
```

Teste a configuração:

``` sudo nginx -t ```

Recarregue o Nginx:

```
sudo systemctl reload nginx.service
```

 

# Instalando o ExpressionEngine

Crie uma pasta na raiz:

```
sudo mkdir -p /var/www/expressionengine
```

Altere o proprietário da pasta /var/www/expressionengine  para fulano (o nome de usuário que você escolheu lá em cima):

```
sudo chown -R fulano:fulano /var/www/expressionengine
```

Navegue para a pasta root:
```
cd /var/www/expressionengine
```

Baixe a última versão do ExpressionEngine e descompacte os arquivos para uma pasta no seu servidor:

```
wget -O ee.zip --referer https://expressionengine.com/ 'https://expressionengine.com/?ACT=243'
unzip ee.zip
rm ee.zip
```

Altere o proprietário da pasta ``` /var/www/expressionengine ``` para ```www-data```:

```
sudo chown -R www-data:www-data /var/www/expressionengine
```

*ATENÇÃO: qualquer pasta ou arquivo subido por FORA do sistema do ExpressionEngine (via FTP ou SFTP, por exemplo) não será reconhecido pelo mesmo, até você alterar o proprietário para ```www-data```. Isso é uma camada extra de segurança, portanto não se esqueça deste detalhe.*

Caso isso aconteça, use o seguinte comando para converter a propriedade dos arquivos ou pastas subidas:

```
sudo chown -R www-data:www-data /var/www/expressionengine/arquivos
```

Onde "arquivos"é a pasta para onde foram subidos os novos arquivos via FTP.

Na sequência, dê um restart no Nginx:

```
sudo systemctl restart nginx
```

Aponte seu navegador para a URL do arquivo ```admin.php``` do ExpressionEngine. Por exemplo: ```https://seusite.com/admin.php``` e siga as instruções para instalar o ExpressionEngine. Assim que você finalizar a instalação, remova a pasta ```system/ee/installer/``` do seu servidor.


