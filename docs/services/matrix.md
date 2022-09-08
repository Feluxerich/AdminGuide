# Matrix

We will create a matrix server which usernames look like `@username:domain.tld`. For this we need to host `.well-known`
config, or we do this in the `nginx`-Config. Put this in your `/etc/nginx/sites-available/domain.tld`:

## .well-known

```nginx
# https://ssl-config.mozilla.org/#server=nginx&version=1.17.7&config=modern&openssl=1.1.1d&guideline=5.6
server {
    server_name domain.tld;
    listen 0.0.0.0:443 ssl http2;
    
    [...]

    location = /.well-known/matrix/server {
        default_type application/json;
        add_header Access-Control-Allow-Origin *;
        return 200 '{"m.server": "synapse.domain.tld:443"}';
    }
    location = /.well-known/matrix/client {
        default_type application/json;
        add_header Access-Control-Allow-Origin *;
        return 200 '{"m.homeserver":{"base_url":"https://synapse.domain.tld/"}}';
    }
}
```

## Configuration

For this configuration we use Postgres as database instead of SQLite. To create the right database for matrix execute
this command in `psql`:

```postgresql
CREATE USER matrix WITH PASSWORD '<securepassword>';
CREATE DATABASE matrix OWNER matrix LOCALE 'C' TEMPLATE 'template0';
```

To create the `homeserver.yaml` and some other configuration files for our homeserver run this command:

```shell
docker run -it --rm -v "/srv/matrix/synapse:/data" -e "SYNAPSE_SERVER_NAME=domain.tld" -e "SYNAPSE_REPORT_STATS=no" matrixdotorg/synapse generate
```

Now go to `/srv/matrix/synapse/homeserver.yaml`. Look for the lines:

```yaml
database:
  name: sqlite3
  args:
    database: /data/homeserver.db
```

and replace them with

```yaml
database:
  name: psycopg2
  args:
    user: matrix
    password: <securepassword>
    database: matrix
    host: postgres
    cp_min: 5
    cp_max: 10
```

## Nginx

Do not forget to issue a certificate for `synapse.domain.tld`

```nginx
# https://ssl-config.mozilla.org/#server=nginx&version=1.17.7&config=modern&openssl=1.1.1d&guideline=5.6
server {
    server_name synapse.domain.tld;
    listen 0.0.0.0:443 ssl http2;

    ssl_certificate /home/felix/.acme.sh/synapse.domain.tld/fullchain.cer;
    ssl_certificate_key /home/felix/.acme.sh/synapse.domain.tld/synapse.domain.tld.key;
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;  # about 40000 sessions
    ssl_session_tickets off;

    ssl_protocols TLSv1.3;
    ssl_prefer_server_ciphers off;

    # HSTS (ngx_http_headers_module is required) (63072000 seconds)
    add_header Strict-Transport-Security "max-age=63072000" always;

    ssl_stapling on;
    ssl_stapling_verify on;

    location ~ ^(/_matrix|/_synapse/client) { # You have to do it this way otherwise matrix will throw you singin errors
            proxy_pass http://[::1]:8000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header X-Real-IP $proxy_add_x_forwarded_for;
    	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
    }
}
```

## Federation

Create a `srv`-Record in your DNS-Settings:

```
;; SRV Records
_matrix._tcp.synapse.domain.tld.    1    IN    SRV    10 5 443 synapse.domain.tld.
```

## Docker

This is the `docker-compose.yaml` for this configuration:

```yaml
version: '3.9'

services:
  synapse:
    image: matrixdotorg/synapse
    restart: always
    volumes:
      - "/srv/matrix/synapse:/data"
    networks:
      - database
      - matrix
    ports:
      - "[::1]:8000:8008"
networks:
  database:
    name: database
    external: true
  matrix:
    name: matrix
    external: true
```

Run `dc up -d`.

## Users

To create users use the command:

```
dc exec synapse register_new_matrix_user -u <username> -p <password> -a -c /data/homeserver.yaml https://synapse.domain.tld
# or this one to see command reference
dc exec synapse register_new_matrix_user -a -c /data/homeserver.yaml https://synapse.domain.tld --help
```