# Vaultwarden

Create your `docker-compose.yaml`:

```yaml
version: '3.9'
services:
  vaultwarden:
    image: vaultwarden/server:alpine
    restart: always
    env_file: .vaultwarden.env
    ports:
      - "[::1]:8003:80"
      - "[::1]:3012:3012"
    volumes:
      - "/srv/vaultwarden:/data/"
    networks:
      - database
networks:
  database:
    name: database
    external: true
```

Then create a `.env`-File:

```shell
# .vaultwarden.env
DOMAIN=https://vault.domain.tld/
SIGNUPS_ALLOWED=false
INVITATIONS_ALLOWED=false
SHOW_PASSWORD_HINT=false
DATABASE_URL=postgresql://<user>:<password>@postgres:5432/<database>
ADMIN_TOKEN=<yoursecureadmintoken>
WEBSOCKET_ENABLED=true
```

Issue a certificate and create a VHosts-File like [here](../0_proxy.md#service-template). Your
VHosts-File should look like this:

```nginx
# https://ssl-config.mozilla.org/#server=nginx&version=1.17.7&config=modern&openssl=1.1.1d&guideline=5.6
server {
    server_name vault.domain.tld;
    listen 0.0.0.0:443 ssl http2;

    ssl_certificate /home/<username>/.acme.sh/vault.domain.tld/fullchain.cer;
    ssl_certificate_key /home/<username>/.acme.sh/vault.domain.tld/vault.domain.tld.key;
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
            proxy_pass http://[::1]:8003/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header X-Real-IP $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
    }

    location /notifications/hub/negotiate {
            proxy_pass http://[::1]:8003/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header X-Real-IP $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
    }
    
    location /notifications/hub {
	    proxy_pass http://[::1]:3012/;
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

Now you may go to `https://vault.domain.tld/admin` and enter your admin secret. Here you can manage your Vaultwarden
instance.