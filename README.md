# docker-osm2pgsql
Docker image with osm2pgsql for importing OpenStreetMap data into a PostGIS database. It is intended to be used with [docker-postgis-osm](https://github.com/ingmapping/docker-postgis-osm).

## docker-osm2pgsql set up

Can be built from the Dockerfile:

```
docker build -t ingmapping/osm2pgsql github.com/ingmapping/docker-osm2pgsql.git
```

or pulled from Docker Hub:

```
docker pull ingmapping/osm2pgsql
```

## docker-osm2pgsql run

To run osm2pgsql in a single container:

```
docker run -i -t --rm ingmapping/osm2pgsql -c 'osm2pgsql -h'
```

When used with a postgis-osm container, it can import data directly into the database.

First make a directory for your import. You can download an extract of the OpenStreetMap data set for your preferred region. We have taken the example of [faroe-islands-latest OSM PBF file](https://download.geofabrik.de/europe/faroe-islands-latest.osm.pbf), because of its small size (1.6 MB).  

```
mkdir ~/osm \
cd ~/osm \
wget https://download.geofabrik.de/europe/faroe-islands-latest.osm.pbf
```
Then to import the OSM data in your PostGIS database with postgis-osm and osm2pgsql. Make sure postgis-osm container is running.

```
docker run -d --name postgis-osm ingmapping/postgis-osm
```
Finally import the data in your PostGIS database with the following command and options:

```
 docker run -i -t --rm --link postgis-osm:pg -v ~/osm:/osm ingmapping/osm2pgsql -c 'osm2pgsql --create --slim --cache 2000 --database $PG_ENV_POSTGRES_DB --username $PG_ENV_POSTGRES_USER --host pg --port $PG_PORT_5432_TCP_PORT /osm/faroe-islands-latest.osm.pbf'
```

The options in the command:

* `-i`: Use an interactive stdin when running this container.
* `-t`: Allocate a pseudo-tty for this container.
* `--rm`: Delete this container when the command is finished. We will use this because we won't need the container anymore when the import finishes as we can always start a new container from the base image.
* `--link postgis-osm:pg`: Link the other running container named `postgis-osm` to this container, under the alias `pg`. This allows us access to that container's ports and data.
* `-v ~/osm:osm`: Mount the directory in our home folder called `osm` inside the container as `/osm`. Allows us to inject data into the container.
* `ingmapping/osm2pgsql`: The base image for this container.
* `-c`: The command to run inside the container. It will run under a bash shell, which is how we use the environment variables. The single quotes around the next portion are required.
* `--create`: Wipe the postgres database and set it up from a clean slate. This is our first import, so we will do that.
* `--slim`: Store temporary information in the database tables. Useful for setting up diff updates of OSM data later.
* `--hstore`: Store OpenStreetMap tag information in the Postgresql hstore column. If you omit this, you will not have extra tags on your nodes.
* `--cache 2000`: Use 2000MB of RAM as a cache for node information during import. Set this to as high as you can depending on your host RAM, but leave a bit for Postgres and other processes. `osm2pgsql` will fail if the number is too high; if so, try again with a lower number. Having it too low will just slow down the import. Default is 800MB.
* `--number-processes 2`: The number of CPU cores to use during the import. Set to as many CPU cores as you have on the host machine to speed up the import.
* `--database $PG_ENV_POSTGRES_DB`: Use the Postgres database named from the `postgres-osm` container. This is a special feature of Docker that reads it automatically from the other container without you having to specify the value for this container.
* `--username $PG_ENV_POSTGRES_USER`: Use the Postgres database user login named from the `postgres-osm` container.
* `--host pg`: As we linked the `postgis-osm` container to this container under the name `pg`, the hosts file in this container will link the hostname `pg` to the `postgis-osm` container.
* `--port $PG_PORT_5432_TCP_PORT`: This may look weird, but it is also Docker linking the port name from one container to the current one.
* `faroe-islands-latest.osm.pbf`: The file to import, as mounted on the container from our home directory's folder.
