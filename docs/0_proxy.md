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
    [...]
    ports:
      - "[::1]:8000:<container_port>"
```