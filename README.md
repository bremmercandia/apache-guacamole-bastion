# PMWEB - Guacamole Docker-Compose
Configurando o guacamole em uma Bastion Instance para acesso RDP

## Pré-Requisitos
Precisaremos dos serviço do **docker** e a ferramenta **docker-compose** instaladas na instância.

## Iniciando Projeto
Copie a pasta apache-guacamole deste repositório para a instancia que será usada e em seguida execute o script para instalação do postgres e inicio das pastas necessárias:

~~~bash
scp -r /docker/apache-guacamole remote_username@10.10.0.2:/
cd apache-guacamole
./prepare.sh
~~~

### Networking
Esta parte do docker-compose.yml criará a rede `guacnetwork` em modo `bridged`.
~~~python
...
# networks
# create a network 'guacnetwork_compose' in mode 'bridged'
networks:
  guacnetwork:
    driver: bridge
...
~~~

### Services
#### guacd
A seguinte parte do docker-compose.yml criará o serviço guacd. guacd é o backend do Guacamole. O contêiner será chamado `guacamole_backend` com base na imagem docker `guacamole/guacd` conectada à nossa rede criada anteriormente `guacnetwork`. Além disso, mapeamos as 2 pastas locais `./drive` e `./record` para o contêiner. Podemos usá-los posteriormente para mapear unidades de usuários e armazenar gravações de sessões.

~~~python
...
services:
  # guacd
  guacd:
    container_name: guacamole_backend
    image: guacamole/guacd
    networks:
      guacnetwork:
    restart: always
    volumes:
    - ./drive:/drive:rw
    - ./record:/var/lib/guacamole/recordings:rw
...
~~~

#### PostgreSQL
A parte a seguir do docker-compose.yml criará uma instância do PostgreSQL usando a imagem oficial do docker. Esta imagem é altamente configurável usando variáveis ​​de ambiente. Por exemplo, ele inicializará um banco de dados se um script de inicialização for encontrado na pasta `/docker-entrypoint-initdb.d` dentro da imagem. Como mapeamos a pasta local `./init` dentro do contêiner como `docker-entrypoint-initdb.d`, podemos inicializar o banco de dados para guacamole usando nosso próprio script (`./init/initdb.sql`).

~~~python
...
 postgres:
    container_name: postgres_guacamole_database
    environment:
      PGDATA: /var/lib/postgresql/data/guacamole
      POSTGRES_DB: guacamole_db
      POSTGRES_PASSWORD: 'abc123'
      POSTGRES_USER: guacamole_user
    image: postgres:15.3-alpine
    networks:
      guacnetwork:
    restart: always
    volumes:
    - ./init:/docker-entrypoint-initdb.d:z
    - ./data:/var/lib/postgresql/data:Z
...
~~~

#### Guacamole
A parte a seguir do docker-compose.yml criará uma instância do guacamole usando a imagem do docker `guacamole` do docker hub. Também é altamente configurável usando variáveis ​​de ambiente. Nesta configuração ele está configurado para se conectar à instância postgres criada anteriormente usando um nome de usuário e senha e o banco de dados `guacamole_db`. A porta 8080 só é exposta localmente! Anexaremos uma instância do nginx para exibição pública na próxima etapa. Definimos aqui também uma extensão para gravação de sessões.

~~~python
...
  guacamole:
    container_name: guacamole_frontend
    depends_on:
    - guacd
    - postgres
    environment:
      GUACD_HOSTNAME: guacd
      RECORDING_SEARCH_PATH: /var/lib/guacamole/recordings
      POSTGRES_DATABASE: guacamole_db
      POSTGRES_HOSTNAME: postgres
      POSTGRES_PASSWORD: 'abc123'
      POSTGRES_USER: guacamole_user
      EXTENSIONS: history-recording-storage
    image: guacamole/guacamole
    volumes:
    - ./record:/var/lib/guacamole/recordings:ro
    - ./drive:/drive:rw
    links:
    - guacd
    networks:
      guacnetwork:
    ports:
## enable next line if not using nginx
##    - 8080:8080/tcp # Guacamole is on :8080/guacamole, not /.
## enable next line when using nginx
    - 8080/tcp
    restart: always
...
~~~

#### nginx
A parte a seguir do docker-compose.yml criará uma instância do nginx que mapeia a porta pública 443 para a porta interna 443. A porta interna 443 é então mapeada para guacamole usando o arquivo `./nginx/templates/guacamole.conf.template `. O contêiner usará o certificado autoassinado gerado anteriormente (`prepare.sh`) em `./nginx/ssl/` com `./nginx/ssl/self-ssl.key` e `./nginx/ssl/self .cert`. Aqui iremos configurar o certificado da pmweb e editar o apontamento do mesmo no mapeamento dos volumes.

~~~python
...
  # nginx
  nginx:
   container_name: nginx_guacamole
   restart: always
   image: nginx
   volumes:
   - ./nginx/templates:/etc/nginx/templates:ro
   - ./nginx/ssl/pmweb.cert:/etc/nginx/ssl/pmweb.cert:ro
   - ./nginx/ssl/pmweb.key:/etc/nginx/ssl/pmweb.key:ro
   ports:
   - 443:443
   links:
   - guacamole
   networks:
     guacnetwork:
...
~~~

## prepare.sh
`prepare.sh` é um script que cria o arquivo `./init/initdb.sql` baixando a imagem `guacamole/guacamole` e é iniciado abaixo com o comando:

~~~bash
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > ./init/initdb.sql
~~~

Ele cria o arquivo de inicialização de banco de dados necessário para o postgres.

`prepare.sh` também cria o certificado autoassinado `./nginx/ssl/self.cert` e a chave privada `./nginx/ssl/self-ssl.key` que são usados
por nginx para https.

## reset.sh
Para resetar toda a configuração, execute o script `./reset.sh`.

# Configurando a instância 

## Permissão de Usuários e Grupos locais da instância

Precisaremos criar 2 usuários então executaremos os passos a seguir:

~~~bash
adduser guacd
adduser guacamole
usermod -u 1000 -g 1000 guacd
usermod -u 1001 -g 1001 guacamole

cat /etc/passwd
cat /etc/group
~~~
Necessário que configuremos deste jeito pois o container precisa acessar algumas pastas e precisa ter permissão para armazenar informações de gravação.

## Permissionamento de Pastas

Atribui para a pasta do projeto dentro da instancia uma permissão 2750.

Criando a pasta record:
~~~bash
cd /apache-guacamole
mkdir record
chown -R 1000:1001 record
chmod -R 2750 record
~~~
## Gravação de sessão

Cheque se a extensão guacamole-history-recording-storage.jar foi corretamente baixada para dentro do container do guacamole_frontend:

~~~bash
docker exec -it guacamole_frontend /bin/bash
cd /opt/guacamole/recordings/
~~~

Caso não será necessário fazer o processo manualmente.

*OBS*: Para sabermos se ela foi carregada podemos olhar os logs do mesmo container, encontraremos uma linha informando o load ou não da extensão.

