# PostgreSQL with pgvector and PostGIS

A Docker image for PostgreSQL 18 with [pgvector](https://github.com/pgvector/pgvector) and [PostGIS](https://postgis.net/) extensions pre-installed.

Perfect for RAG (Retrieval-Augmented Generation) applications that need vector similarity search for embeddings alongside spatial/geographic queries.

## Extensions

| Extension | Version | Description |
|-----------|---------|-------------|
| pgvector  | 0.8.1   | Vector similarity search |
| PostGIS   | 3.x     | Spatial and geographic objects |

## Quick Start

```bash
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -p 5432:5432 \
  ghcr.io/suhaib/postgres-pgvector-postgis:latest
```

The extensions are automatically created on the default database at startup.

## Usage

### Pull the image

```bash
# Latest version
docker pull ghcr.io/suhaib/postgres-pgvector-postgis:latest

# Specific version
docker pull ghcr.io/suhaib/postgres-pgvector-postgis:1.0.0
```

### Docker Compose

```yaml
services:
  db:
    image: ghcr.io/suhaib/postgres-pgvector-postgis:latest
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### Verify extensions

```sql
SELECT * FROM pg_extension WHERE extname IN ('vector', 'postgis');
```

## Building Locally

```bash
cd postgres
docker build -t postgres-pgvector-postgis .
```

### Build arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `PG_MAJOR` | 18 | PostgreSQL major version |
| `PGVECTOR_VERSION` | 0.8.1 | pgvector version |

```bash
docker build \
  --build-arg PG_MAJOR=17 \
  --build-arg PGVECTOR_VERSION=0.8.0 \
  -t postgres-pgvector-postgis:pg17 \
  postgres/
```

## Supported Architectures

- `linux/amd64`
- `linux/arm64`

## License

PostgreSQL is released under the [PostgreSQL License](https://www.postgresql.org/about/licence/).
pgvector is released under the [PostgreSQL License](https://github.com/pgvector/pgvector/blob/master/LICENSE).
PostGIS is released under the [GPLv2 License](https://postgis.net/documentation/faq/#license).
