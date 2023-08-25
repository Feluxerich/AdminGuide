# Hedgedoc

Hedgedoc is a collaborative markdown editing tool. To set up Hedgedoc create a `docker-compose.yaml` like this:

```yaml
version: '3.9'
services:
  hedgedoc:
    image: quay.io/hedgedoc/hedgedoc
    restart: always
    env_file: .hedgedoc.env
    volumes:
      - "/srv/hedgedoc/uploads:/hedgedoc/public/uploads"
    networks:
      - database
    ports:
      - "[::1]:8001:3000"
networks:
  database:
    name: database
    external: true
```

As you may see in the configuration you also need a `.env`-File. This should look like this:

```shell
# .hedgedoc.env
CMD_DB_URL=postgres://<username>:<password>@postgres:5432/<database>
CMD_DOMAIN=md.domain.tld
CMD_PROTOCOL_USESSL=true
CMD_URL_ADDPORT=false

CMD_EMAIL=false
CMD_ALLOW_EMAIL_REGISTER=false

CMD_ALLOW_FREEURL=true
CMD_REQUIRE_FREEURL_AUTHENTICATION=true

CMD_GITHUB_CLIENTID=<github_oauth_id>
CMD_GITHUB_CLIENTSECRET=<github_oauth_secret>

CMD_ALLOW_ANONYMOUS=false
CMD_ALLOW_ANONYMOUS_EDITS=true

CMD_SESSION_SECRET=<session_secret>
```

If you want to create a new note with a custom alias you can just open the editor with a Custom-Aliased-URL.
To prevent not authenticated users to create a bunch of trash notes you have to be signed in to do this.

In this configuration only the authentication with Github is possible but if you want to allow local users via email
you have to remove the line that says `CMD_EMAIL=false` and create the accounts manually using the `bin/manage_users`-Script
in the container.

When you have done this start your container with `dc up -d`,
[issue a certificate and create a VHost-File](../0_proxy.md#service-template) containing something like this:

```nginx
# https://ssl-config.mozilla.org/#server=nginx&version=1.17.7&config=modern&openssl=1.1.1d&guideline=5.6
server {
    server_name md.domain.tld;
    listen 0.0.0.0:443 ssl http2;

    ssl_certificate /home/<username>/.acme.sh/md.domain.tld/fullchain.cer;
    ssl_certificate_key /home/<username>/.acme.sh/md.domain.tld/md.domain.tld.key;
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
            proxy_pass http://[::1]:8001/;
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