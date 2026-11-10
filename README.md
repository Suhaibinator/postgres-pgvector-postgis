# PostgreSQL with pgvector, PostGIS and ParadeDB pg_search

A Docker image for PostgreSQL 18 with [pgvector](https://github.com/pgvector/pgvector), [PostGIS](https://postgis.net/) and [ParadeDB `pg_search`](https://github.com/paradedb/paradedb) extensions pre-installed.

Perfect for RAG (Retrieval-Augmented Generation) applications that need vector similarity search for embeddings, BM25 full-text search, and spatial/geographic queries in one database.

Note: The official [postgis/postgis](https://hub.docker.com/r/postgis/postgis) Docker image does not provide arm64 builds ([tracking issue](https://github.com/postgis/docker-postgis/issues/216)), making this image a good choice for Apple Silicon Macs and ARM-based servers.

## Extensions

| Component | Version | Description |
|-----------|---------|-------------|
| PostgreSQL | 18.6 (`postgres:18.6-trixie`, Debian 13) | Base image |
| pgvector  | 0.8.7   | Vector similarity search (built from source) |
| PostGIS   | 3.7.0   | Spatial and geographic objects (PGDG package) |
| pg_search | 0.26.6  | ParadeDB BM25 full-text search (prebuilt ParadeDB release package) |

All versions are pinned with build arguments in [`postgres/Dockerfile`](postgres/Dockerfile).

### `shared_preload_libraries`

`pg_search` must be listed in `shared_preload_libraries`. The image sets
`shared_preload_libraries = 'pg_search'` in the `postgresql.conf` template, so
**new** data directories get it automatically.

A data directory created by an older version of this image keeps its own
`postgresql.conf` and does not have it. Postgres starts normally, but
`CREATE EXTENSION pg_search` fails until you enable the preload and restart:

```sql
-- Include any libraries you already preload, e.g. 'pg_stat_statements,pg_search'
ALTER SYSTEM SET shared_preload_libraries = 'pg_search';
```

Then restart the container and run:

```sql
CREATE EXTENSION IF NOT EXISTS pg_search;
ALTER EXTENSION vector UPDATE;
ALTER EXTENSION postgis UPDATE;
```

Alternatively, pass it on the command line (this overrides the value in
`postgresql.conf`): `command: ["postgres", "-c", "shared_preload_libraries=pg_search"]`.

## Quick Start

```bash
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -p 5432:5432 \
  ghcr.io/suhaibinator/postgres-pgvector-postgis:latest
```

The extensions (`vector`, `postgis`, `pg_search`) are automatically created in the default database when the data directory is first initialized.

## Usage

### Pull the image

```bash
# Latest version
docker pull ghcr.io/suhaibinator/postgres-pgvector-postgis:latest

# Specific version
docker pull ghcr.io/suhaibinator/postgres-pgvector-postgis:1.0.1
```

### Docker Compose

```yaml
services:
  db:
    image: ghcr.io/suhaibinator/postgres-pgvector-postgis:latest
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      # PostgreSQL 18+ images keep data in /var/lib/postgresql/18/docker;
      # mount the parent directory, not /var/lib/postgresql/data.
      - postgres_data:/var/lib/postgresql

volumes:
  postgres_data:
```

### Verify extensions

```sql
SELECT extname, extversion FROM pg_extension WHERE extname IN ('vector', 'postgis', 'pg_search');
```

### BM25 search example

```sql
CREATE TABLE docs (id int PRIMARY KEY, body text, embedding vector(3));
INSERT INTO docs VALUES (1, 'PostgreSQL full text search with BM25 ranking', '[0,1,0]');

CREATE INDEX docs_search_idx ON docs USING paradedb (id, body);

SELECT id, pdb.score(id) AS score
FROM docs
WHERE body ||| 'search ranking'
ORDER BY score DESC;
```

See the [ParadeDB documentation](https://docs.paradedb.com) for tokenizers and query syntax.

## Building Locally

```bash
cd postgres
docker build -t postgres-pgvector-postgis .
```

### Build arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `PG_MAJOR` | 18 | PostgreSQL major version |
| `PG_MINOR` | 6 | PostgreSQL minor version (base image is `postgres:${PG_MAJOR}.${PG_MINOR}-${DEBIAN_RELEASE}`) |
| `DEBIAN_RELEASE` | trixie | Debian release of the base image |
| `PGVECTOR_VERSION` | 0.8.7 | pgvector release tag |
| `POSTGIS_DEB_VERSION` | 3.7.0+dfsg-1.pgdg13+1 | Exact PGDG PostGIS package version |
| `PG_SEARCH_VERSION` | 0.26.6 | ParadeDB pg_search release |
| `PG_SEARCH_SHA256_AMD64` / `PG_SEARCH_SHA256_ARM64` | (see Dockerfile) | SHA-256 of the pg_search `.deb` for each architecture |

```bash
docker build \
  --build-arg PGVECTOR_VERSION=0.8.6 \
  -t postgres-pgvector-postgis:custom \
  postgres/
```

When changing `PG_MAJOR`, `PG_SEARCH_VERSION` or `DEBIAN_RELEASE`, the pg_search
checksums must be updated to match
`postgresql-<major>-pg-search_<version>-1PARADEDB-<debian>_<arch>.deb` from the
[ParadeDB release](https://github.com/paradedb/paradedb/releases); the build fails
on a mismatch.

### Upgrading PostgreSQL major versions

A data directory created by one PostgreSQL major version cannot be opened by
another. Moving to an image with a different `PG_MAJOR` requires `pg_upgrade`
or a dump and restore. Pin deployments by digest or by a release tag rather
than relying on `latest` across major upgrades.

## Supported Architectures

- `linux/amd64`
- `linux/arm64`

## License

PostgreSQL is released under the [PostgreSQL License](https://www.postgresql.org/about/licence/).
pgvector is released under the [PostgreSQL License](https://github.com/pgvector/pgvector/blob/master/LICENSE).
PostGIS is released under the [GPLv2 License](https://postgis.net/documentation/faq/#license).
ParadeDB pg_search is released under the [AGPL-3.0 License](https://github.com/paradedb/paradedb/blob/main/LICENSE).
