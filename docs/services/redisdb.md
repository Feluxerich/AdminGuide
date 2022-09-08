# RedisDB

For RedisDB you only need a `docker-compose.yaml`:

```yaml
version: '3.9'

services:
  redis:
    image: redis
    restart: always
    command: "redis-server --appendonly yes"
    volumes:
      - "/srv/redis:/data"
    networks:
      - database
networks:
  database:
    name: database
    external: true
```

That's it.