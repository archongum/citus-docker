FROM postgres:14.4
ARG VERSION=11.0.4
LABEL maintainer="Citus Data https://citusdata.com" \
      org.label-schema.name="Citus" \
      org.label-schema.description="Scalable PostgreSQL for multi-tenant and real-time workloads" \
      org.label-schema.url="https://www.citusdata.com" \
      org.label-schema.vcs-url="https://github.com/citusdata/citus" \
      org.label-schema.vendor="Citus Data, Inc." \
      org.label-schema.version=${VERSION} \
      org.label-schema.schema-version="1.0"

## Mongo FDW
ARG MONGO_FDW_VERSION=5_4_0
ARG MONGO_FDW_URL=https://github.com/EnterpriseDB/mongo_fdw/archive/REL-${MONGO_FDW_VERSION}.tar.gz
ARG SOURCE_FILES=/tmp/mongo_fdw
# compilation deps
RUN apt-get update\
    && apt-get install -y --no-install-recommends wget ca-certificates make gcc cmake pkg-config postgresql-server-dev-14 libssl-dev libzstd-dev \
    # mongo_fdw
    && export PKG_CONFIG_PATH=${SOURCE_FILES}/mongo-c-driver/src/libmongoc/src:${SOURCE_FILES}/mongo-c-driver/src/libbson/src \
    && export LD_LIBRARY_PATH=/lib \
    && mkdir -p ${SOURCE_FILES} \
    && wget -O - ${MONGO_FDW_URL} | tar -zx --strip-components=1 -C ${SOURCE_FILES} \
    && cd ${SOURCE_FILES} \
    # install
    && ./autogen.sh --with-master \
    && make \
    && make install \
    # cleanup
    && apt-get purge -y --auto-remove wget ca-certificates make gcc cmake pkg-config postgresql-server-dev-14 libssl-dev libzstd-dev \
    && cd - && rm -rf ${SOURCE_FILES}

## Citus
ENV CITUS_VERSION ${VERSION}.citus-1
ENV POSTGRES_DB citus
ENV PGDATA /data

RUN localedef -i zh_CN -c -f UTF-8 -A /usr/share/locale/locale.alias zh_CN.UTF-8 \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
ENV LANG zh_CN.UTF-8
ENV LC_COLLATE 'zh_CN.UTF-8'
ENV LC_CTYPE 'zh_CN.UTF-8'

# install Citus and needed extension on hyperscale
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       ca-certificates \
       curl \
    && curl -s https://install.citusdata.com/community/deb.sh | bash \
    && apt-get install -y postgresql-$PG_MAJOR-citus-11.0=$CITUS_VERSION \
                          postgresql-$PG_MAJOR-hll=2.16.citus-1 \
                          postgresql-$PG_MAJOR-topn=2.4.0 \
                          postgresql-14-postgis-3 \
                          postgresql-plperl-14 \
                          postgresql-contrib \
                          postgresql-14-partman \
                          postgresql-14-tdigest \
                          postgresql-14-mysql-fdw \
                          postgresql-14-cron \
    && apt-get purge -y --auto-remove curl

# set shared_preload_libraries to the list of extensions that needs to be loaded when postgresql starts
RUN echo "shared_preload_libraries='citus,pg_cron,pg_stat_statements'" >> /usr/share/postgresql/postgresql.conf.sample

# set default database name for pg_cron extension
RUN echo "cron.database_name = 'citus'" >> /usr/share/postgresql/postgresql.conf.sample


# add scripts to run after initdb
COPY hyperscale/001-db-user-creation.sql /docker-entrypoint-initdb.d/
COPY hyperscale/002-create_extension_functions.sql /docker-entrypoint-initdb.d/
COPY hyperscale/003-create_citus_extension.sql /docker-entrypoint-initdb.d/
COPY hyperscale/004-postgres_replication_origins.sql /docker-entrypoint-initdb.d/
COPY hyperscale/005-other_extensions.sql /docker-entrypoint-initdb.d/
COPY hyperscale/006-pg_cron_pg_partman_setup.sql /docker-entrypoint-initdb.d/
COPY hyperscale/007-grant_usage_fdw.sql /docker-entrypoint-initdb.d/
COPY hyperscale/008-other_grants.sql /docker-entrypoint-initdb.d/
COPY hyperscale/009-stat_statements_reset.sql /docker-entrypoint-initdb.d/

# copy access rules for postgres
COPY hyperscale/pg_hba.conf /pg_hba.conf

# add health check script
COPY pg_healthcheck wait-for-manager.sh /

# required for to replace citus user password in the execution of docker-entrypoint.sh
RUN chown -R postgres:postgres /docker-entrypoint-initdb.d

COPY hyperscale/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh

HEALTHCHECK --interval=4s --start-period=6s CMD ./pg_healthcheck

