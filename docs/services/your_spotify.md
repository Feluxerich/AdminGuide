# Your Spotify

This is a little dashboard for your Spotify activity.

## Spotify

You have to create a application on the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard/applications). Then click on this application and click the button `Edit Settings`. You have to insert in the input field for `Redirect URIs` your callback URI looking like `https://music.domain.tld/api/oauth/spotify/callback`. Reveal and copy your client secret and your client id.

## Docker

You need a `docker-compose.yaml`. Here is a sample file for you:

```yaml
version: '3.9'

services:
  your-spotify-server:
    image: yooooomi/your_spotify_server
    restart: always
    ports:
      - "[::1]:8011:8080"
    environment:
      - API_ENDPOINT=https://music.domain.tld/api
      - CLIENT_ENDPOINT=https://music.domain.tld
      - SPOTIFY_PUBLIC=<spotify_public_key>
      - SPOTIFY_SECRET=<spotify_app_secret>
      - CORS=all
      - MONGO_ENDPOINT=mongodb://your_spotify:<secure_password>@mongodb:27017/your_spotify?authSource=your_spotify
    networks:
      - database
  your-spotify-web:
    image: yooooomi/your_spotify_client
    restart: always
    ports:
      - "[::1]:8010:3000"
    environment:
      - API_ENDPOINT=https://music.domain.tld/api
```

## Nginx

The `nginx`-configuration deviates a little bit from the sample config. You need two locations on one domain. Here is how to do it:

```nginx
server {
    server_name music.domain.tld;
    listen 0.0.0.0:443 ssl http2;

    ssl_certificate /home/felix/.acme.sh/music.domain.tld/fullchain.cer;
    ssl_certificate_key /home/felix/.acme.sh/music.domain.tld/music.domain.tld.key;
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
            proxy_pass http://[::1]:8010/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header X-Real-IP $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
    }
    location /api/ {
	    proxy_pass http://[::1]:8011/;
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

Then create a DNS record and issue a certificate like [here](../0_proxy.md#service-template). Start your container and reload Nginx.