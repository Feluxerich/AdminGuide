# MongoDB

Create `docker-compose.yaml` and your `.env`-File:

```yaml
version: '3.9'

services:
  mongodb:
    image: mongo
    restart: always
    env_file: .mongodb.env
    ports:
      - "27017:27017"
    volumes:
      - "/etc/localtime:/etc/localtime:ro"
      - "/srv/mongodb/data:/data/db"
    networks:
      - database
networks:
  database:
    name: database
    external: true
```

```shell
# .mongodb.env
MONGO_INITDB_ROOT_USERNAME=root
MONGO_INITDB_ROOT_PASSWORD=<password>
```

Now you have set up MongoDB.