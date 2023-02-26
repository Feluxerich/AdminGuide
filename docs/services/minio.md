# MinIO

If you want to setup MinIO your first step should be to check if you have enough storage for whatever you want to do.
Maybe you want to [mount a Samba File Share](../1_additional_configs.md) to save your MinIO Data on it. Then you can create a `docker-compose.yaml`-file like this:

```yaml
version: '3.9'
services:
  minio:
    image: quay.io/minio/minio
    env_file: .minio.env
    command: server /data --console-address ":9090"
    ports:
      - "[::1]:8002:9090"
      - "9000:9000"
    volumes:
      - "/srv/minio:/data"
```

If you now look at your created `docker-compose.yaml` you may see that a `.minio.env`-file is required. In this
you have to put the following content:

```shell
# .minio.env
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=YOUR-SUPER-SECRET-PASSWORD
```

When you have done this start your container with `dc up -d`,
[issue a certificate and create a VHost-File](../0_proxy.md#service-template) containing something like this:

```nginx
# https://ssl-config.mozilla.org/#server=nginx&version=1.17.7&config=modern&openssl=1.1.1d&guideline=5.6
server {
    server_name minio.domain.tld;
    listen 0.0.0.0:443 ssl http2;

    ssl_certificate /home/<username>/.acme.sh/minio.domain.tld/fullchain.cer;
    ssl_certificate_key /home/<username>/.acme.sh/minio.domain.tld/minio.domain.tld.key;
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
            proxy_pass http://[::1]:8002/;
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

And you are done. Congratulations!
