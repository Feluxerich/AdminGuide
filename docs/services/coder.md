# Coder

!!! note ""
	Coder is an Open Source solution as an equivalent to Github Codespaces. You do not need to pay anything than your server.
  
## Docker

To setup a Coder instance your `docker-compose.yaml` should look like this:

```yaml
version: '3.9'
services:
  coder:
    image: ghcr.io/coder/coder:latest
    restart: always
    env_file: .coder.env
    ports:
      - "[::1]:8012:7080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    networks:
      - database
    group_add:
      - "999"
networks:
  database:
    name: database
    external: true
```

The flag under `group_add` has to be the Docker GroupID. The `.env` file should the following content:

```
# .coder.env
CODER_PG_CONNECTION_URL=postgresql://<username>:<password>@postgres:5432/<database>?sslmode=disable
CODER_ADDRESS=0.0.0.0:7080
CODER_ACCESS_URL=https://coder.domain.tld
CODER_WILDCARD_ACCESS_URL=https://*.coder.domain.tld # This is for the internal portforwarding. If you don't want to use this you just remove this key.
```

## Nginx

Issue a certificate for your domain and create your VHost-File:

```nginx
# https://ssl-config.mozilla.org/#server=nginx&version=1.17.7&config=modern&openssl=1.1.1d&guideline=5.6
server {
    server_name coder.domain.tld;
    listen 0.0.0.0:443 ssl http2;

    ssl_certificate /home/<username>/.acme.sh/coder.domain.tld/fullchain.cer;
    ssl_certificate_key /home/<username>/.acme.sh/coder.domain.tld/coder.domain.tld.key;
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;  # about 40000 sessions
    ssl_session_tickets off;

    ssl_protocols TLSv1.3;
    ssl_prefer_server_ciphers off;

    # HSTS (ngx_http_headers_module is required) (63072000 seconds)
    add_header Strict-Transport-Security "max-age=63072000" always;

    ssl_stapling on;
    ssl_stapling_verify on;

    location / {
            proxy_pass http://[::1]:8012/;
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

## Port forwarding

!!! warning ""
	The Coder Port Forwarding is currently not working. If you want you can do this configuration anyway.

If you need this append the following line to your `.env`-File:

```
CODER_WILDCARD_ACCESS_URL=*.coder.domain.tld
```

The command to issue your certificate is now:

```sh
acme.sh --issue --keylength ec-384 --dns dns_cf -d coder.domain.tld -d *.coder.domain.tld
```

And you also have to create a DNS-Record named `*.coder.domain.tld` pointing to `coder.domain.tld`
