# Deploy PostgreSQL on Podman/Docker with TLS

Security is an essential requirement for all systems, and requires the
engagement of all actors in the process from project manager, product owners,
developers, testers and DevOps engineers.

PostgreSQL is one of the most popular open source relational databases, used
from large to small projects. It is available on different platforms and as a
Docker Container, although to enable TLS (new SSL) on a basic deployment, for
example to develop a small project, it requires a few additional steps described
in this post.

In the following examples, I'll be using Podman an alternative to Docker, to run
it on Docker should be as simple as replacing `podman` by `docker` is the
commands below.

This script is included into a new image built with the following definition on
`Containerfile`:

```ini
FROM postgres:15-alpine

VOLUME /var/lib/postgresql/data

# ENV PGDATA=/var/lib/postgresql/data -- the default
ENV POSTGRES_USER dev
ENV POSTGRES_PASSWORD devS3cret
ENV POSTGRES_DB development
# COPY ./database/init.sql /docker-entrypoint-initdb.d/ # load initial data

COPY ./out/postgresdb.key /var/lib/postgresql
COPY ./out/postgresdb.crt /var/lib/postgresql

COPY ./out/devCA.crt /var/lib/postgresql
COPY ./out/devCA.crl /var/lib/postgresql

RUN chown root:postgres /var/lib/postgresql/postgresdb.key && chmod 640 /var/lib/postgresql/postgresdb.key
RUN chown root:postgres /var/lib/postgresql/postgresdb.crt && chmod 640 /var/lib/postgresql/postgresdb.crt

RUN chown root:postgres /var/lib/postgresql/devCA.crt && chmod 640 /var/lib/postgresql/devCA.crt
RUN chown root:postgres /var/lib/postgresql/devCA.crl && chmod 640 /var/lib/postgresql/devCA.crl

ENTRYPOINT ["docker-entrypoint.sh"]
CMD [ "-c", "ssl=on",\
  "-c", "ssl_cert_file=/var/lib/postgresql/postgresdb.crt",\
  "-c", "ssl_key_file=/var/lib/postgresql/postgresdb.key",\
  "-c", "ssl_ca_file=/var/lib/postgresql/devCA.crt",\
  "-c", "ssl_crl_file=/var/lib/postgresql/devCA.crl" ]
```

To keep everything in context is a good practice, develop build scripts it helps
as documentation of the entire process, for this, I'm a fan of GNU Make what I
have used to put together additional steps related to different remaining steps.

```makefile
## install-tools: certstrap is used to create a new certificate authority
install-tools:
	@echo "certstrap is used to create a new certificate authority"
	go install github.com/square/certstrap@latest

## create-certs: create certificate authority
create-certs:
	@echo "create certificate authority"
	certstrap init --common-name devCA
	certstrap request-cert --common-name postgresdb --domain localhost
	certstrap sign postgresdb --CA devCA
	@mkdir -p ~/.postgresql
	@cp out/devCA.crt ~/.postgresql/root.crt
	@cp out/postgresdb.crt ~/.postgresql/postgresql.crt
	@cp out/postgresdb.key ~/.postgresql/postgresql.key
	@chmod 0600 ~/.postgresql/*

## build-image: builds container image from Containerfile
build-image:
	@echo "create container image from Containerfile"
	@podman build -t postgres15ssl .
	@podman image inspect localhost/postgres15ssl:latest -f '{{.Created}}'

## run: run local built image
run:
	@echo "run local built image"
	@podman run -d -p 5432:5432 --name=pg localhost/postgres15ssl:latest
	@sleep 10
	@podman exec -it pg bash -c "echo 'hostssl all all all cert clientcert=verify-ca' > /var/lib/postgresql/data/pg_hba.conf"
	psql "host=127.0.0.1 port=5432 user=dev dbname=development" -c 'select pg_reload_conf();'
	# podman logs -f pg

## start: start podman instance pg
start:
	@echo "start podman pg"
	podman start pg

## start: start podman instance pg
stop:
	@echo "stop podman pg"
	podman stop pg

## psql: run psql with ssl
psql:
	psql "host=127.0.0.1 port=5432 user=dev dbname=development"

## all: run all three AWS for instances, images, costs
all: install-tools create-certs build-image run psql

## deletes all container and image
clean: stop
	podman rm pg
	podman rmi postgres15ssl

## help: displays help
help: Makefile
	@echo " Choose a command:"
	@sed -n 's/^##//p' $< | column -t -s ':' |  sed -e 's/^/ /'
```

It's all for this first post, my goal is to leave here small posts for my future
self, and anyone who ends up in these pages able to make use of it.
