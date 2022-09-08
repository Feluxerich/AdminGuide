# Postgres

Create `docker-compose.yaml`:

```yaml
version: '3.9'

services:
  postgres:
    image: postgres
    restart: always
    env_file: .postgres.env
    volumes:
      - "/srv/postgres:/var/lib/postgresql/data"
    networks:
      - database
networks:
  database:
    name: database
    external: true
```

And your `.env`-File:

```shell
# .postgres.env
POSTGRES_USER=root
POSTGRES_PASSWORD=<password>
POSTGRES_DB=postgres
```

Then just run `dc up -d` and you have your own PostgreSQL.