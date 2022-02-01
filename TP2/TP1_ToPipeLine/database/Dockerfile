FROM postgres:11.6-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY init_db/*.sql /docker-entrypoint-initdb.d/

