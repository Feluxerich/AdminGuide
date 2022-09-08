# Proxy

## Install

In this configuration we will use `nginx` as reverse-proxy. First of all install
`nginx-full`:

```shell
sudo apt install nginx-full
```

Then add an alias to reload your proxy:

```shell
cat <<_EOF >> .bashrc
alias ngr='sudo nginx -t && sudo systemctl reload nginx'
_EOF
```

Put this in your `/etc/nginx/sites-available/default` to redirect to `https` by default:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 301 https://$host$request_uri;
}
```

## SSL Certificates

For securing our sites we use `.acme.sh`. Install it and run some additional commands:

```shell
curl https://get.acme.sh | sh -s email=certs@domain.tld # You probably do not need a Mailserver for this
sudo ln -s ~/.acme.sh/acme.sh /usr/bin/acme.sh # Relog
acme.sh --install-cronjob
```

To issue a certificate run following command:

```shell
acme.sh --issue --keylength ec-384 --dns dns_cf -d subdomain.domain.tld
```

The certificate will be generated in `~/.acme.sh/subdomain.domain.tld_ecc`. Move this folder to
`~/.acme.sh/subdomain.domain.tld`.

## Service Template

Create a folder for your service or your service stack and create a `docker-compose.yaml`:

```yaml
version: '3.9'
services:
  service:
    [ ... ]
    ports:
      - "[::1]:8000:<container_port>"
```

Issue a certificate for your website as I mentioned earlier and create a VHost-File like this:

```nginx
server {
    server_name subdomain.domain.tld;
    listen 0.0.0.0:443 ssl http2;

    ssl_certificate /home/<username>/.acme.sh/subdomain.domain.tld/fullchain.cer;
    ssl_certificate_key /home/<username>/.acme.sh/subdomain.domain.tld/subdomain.domain.tld.key;
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
            proxy_pass http://[::1]:8000/;
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

Link your VHost-File and reload:

```shell
sudo ln -s /etc/nginx/sites-available/subdomain.domain.tld /etc/nginx/sites-enabled/
ngr
```