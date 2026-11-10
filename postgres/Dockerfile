# Component versions. Each bump is a one-line change here (pg_search also
# needs its per-architecture checksums updated below).
#
# PG_MAJOR/PG_MINOR/DEBIAN_RELEASE select the official postgres base image tag.
ARG PG_MAJOR=18
ARG PG_MINOR=6
ARG DEBIAN_RELEASE=trixie

FROM postgres:${PG_MAJOR}.${PG_MINOR}-${DEBIAN_RELEASE}

# Re-import the global values above into this build stage.
ARG PG_MAJOR
ARG DEBIAN_RELEASE

# pgvector release tag (built from source).
ARG PGVECTOR_VERSION=0.8.7

# Exact PGDG Debian package version of PostGIS. The "pgdg13" suffix is tied to
# Debian 13 (trixie). List available versions with:
#   apt-cache madison postgresql-${PG_MAJOR}-postgis-3
ARG POSTGIS_DEB_VERSION=3.7.0+dfsg-1.pgdg13+1

# ParadeDB pg_search release and the SHA-256 of its prebuilt .deb per
# architecture (see the "digest" of each asset on the GitHub release).
ARG PG_SEARCH_VERSION=0.26.6
ARG PG_SEARCH_SHA256_AMD64=7553b976363afd7144e51585d281fee006edc17b6e83b75ce0d23921a13150a9
ARG PG_SEARCH_SHA256_ARM64=af76ca5a1ebff0c2a9ec9c5b5b383ffc8557965f72829a4f35c72846a4ec987f

LABEL maintainer="suhaib"
LABEL description="PostgreSQL ${PG_MAJOR} with PostGIS, pgvector and ParadeDB pg_search extensions"

# Install PostGIS from PGDG and build pgvector from source.
# The PGDG archive repository keeps superseded package versions, so the exact
# PostGIS pin keeps working after PGDG publishes a newer release. Plain http
# matches the base image's PGDG source; packages are verified by signed-by.
RUN set -eux; \
    echo "deb [ signed-by=/usr/local/share/keyrings/postgres.gpg.asc ] http://apt-archive.postgresql.org/pub/repos/apt ${DEBIAN_RELEASE}-pgdg-archive main" \
        > /etc/apt/sources.list.d/pgdg-archive.list; \
    apt-get update; \
    apt-mark hold locales; \
    apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        git \
        postgresql-server-dev-${PG_MAJOR} \
        postgresql-${PG_MAJOR}-postgis-3=${POSTGIS_DEB_VERSION} \
        postgresql-${PG_MAJOR}-postgis-3-scripts=${POSTGIS_DEB_VERSION}; \
    cd /tmp; \
    git clone --depth 1 --branch v${PGVECTOR_VERSION} https://github.com/pgvector/pgvector.git; \
    cd pgvector; \
    make clean; \
    make OPTFLAGS=""; \
    make install; \
    cd /; \
    rm -rf /tmp/pgvector; \
    apt-get remove -y build-essential git postgresql-server-dev-${PG_MAJOR}; \
    apt-get autoremove -y; \
    apt-mark unhold locales; \
    rm -rf /var/lib/apt/lists/*

# Install ParadeDB pg_search (BM25 full-text search) from its prebuilt release
# package. pg_search >= 0.25 depends on pgvector, which is installed above.
RUN set -eux; \
    arch="$(dpkg --print-architecture)"; \
    case "$arch" in \
        amd64) checksum="${PG_SEARCH_SHA256_AMD64}" ;; \
        arm64) checksum="${PG_SEARCH_SHA256_ARM64}" ;; \
        *) echo "pg_search: unsupported architecture: $arch" >&2; exit 1 ;; \
    esac; \
    apt-get update; \
    apt-get install -y --no-install-recommends curl; \
    curl -fsSL -o /tmp/pg_search.deb \
        "https://github.com/paradedb/paradedb/releases/download/v${PG_SEARCH_VERSION}/postgresql-${PG_MAJOR}-pg-search_${PG_SEARCH_VERSION}-1PARADEDB-${DEBIAN_RELEASE}_${arch}.deb"; \
    echo "${checksum} */tmp/pg_search.deb" | sha256sum --strict --check -; \
    apt-get install -y --no-install-recommends /tmp/pg_search.deb; \
    apt-get purge -y --auto-remove curl; \
    rm -f /tmp/pg_search.deb; \
    rm -rf /var/lib/apt/lists/*

# pg_search must be in shared_preload_libraries. This edits the sample config
# that initdb copies into new data directories; existing data directories keep
# their own postgresql.conf (see README).
RUN set -eux; \
    sed -ri "s!^#?(shared_preload_libraries)\s*=\s*'[^']*'!\1 = 'pg_search'!" /usr/share/postgresql/postgresql.conf.sample; \
    grep -q "^shared_preload_libraries = 'pg_search'" /usr/share/postgresql/postgresql.conf.sample

COPY initdb-extensions.sh /docker-entrypoint-initdb.d/10-create-extensions.sh
RUN chmod +x /docker-entrypoint-initdb.d/10-create-extensions.sh

EXPOSE 5432
