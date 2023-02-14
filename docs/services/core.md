# core

The core-project is a web app which has no specific use case. To set it up correctly you first have to create a `docker-compose.yaml`:

```yaml
version: "3.9"
services:
  core:
    image: ghcr.io/feluxerich/core:latest
    env_file: .core.env
    ports:
      - "[::1]:8004:3000"
    volumes:
      - "./.core.env:/app/.env"
```

After that create a `.env`-File. As you may have seen above you have to name it `.core.env`. In this file you have to set the following values:

```
MONGO_URI="<mongodb connection uri>"
JWT_SECRET_KEY="SUPER_SECRET_KEY"
NEXT_PUBLIC_BASE_URL="https://domain.tld"
```

Now you are ready to fire up the container.
