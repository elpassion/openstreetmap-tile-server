## Setting Up an OpenStreetMap Tile Server with Docker

(based on [Switch2OSM](https://switch2osm.org/serving-tiles/using-a-docker-container/) guide and docker image)

### Prerequisites

- Ensure that [Docker](https://www.docker.com/) is installed on the machine where you want to run the map tile server.
- Check the storage requirements for your selected maps [here](https://tools.geofabrik.de/calc/).

#### Downloading Required Files

Before proceeding, download the necessary map files for your region

- eg. https://download.geofabrik.de/europe/poland-latest.osm.pbf
- eg. https://download.geofabrik.de/europe/poland.poly

#### Running the Import Process

Execute the following commands to set up and import the map data:

```
  docker volume create osm-data
  docker volume create osm-tiles
  docker run \
    -v [ABSOLUTE_PATH_TO_DOWNLOADED_PBF_FILE]:/data/region.osm.pbf \
    -v osm-data:/data/database/ \
    overv/openstreetmap-tile-server import
```

It is also possible to let the container download files for you rather than mounting them in advance by using the `DOWNLOAD_PBF` and `DOWNLOAD_POLY` parameters:

```
  docker run -e DOWNLOAD_PBF=https://download.geofabrik.de/europe/poland-latest.osm.pbf -e DOWNLOAD_POLY=https://download.geofabrik.de/europe/poland.poly -v osm-data:/data/database/ overv/openstreetmap-tile-server import
```

⚠️ Important Notes:

- The path to the `.pbf` file must be absolute—relative paths will not work.
- If the process encounters an error, you may need to reset the volume:

```
  docker volume rm osm-data
  docker volume create osm-data
```

## Final Steps

The import process will take some time, depending on your internet speed and hardware capabilities. Before running the tile server on-premise, make sure all necessary maps are fully downloaded. Simply execute the commands above, and the setup should proceed automatically.

#### Run server

When everything is finished simply run:

```
  docker run -p 8080:80 -e ALLOW_CORS=enabled -v osm-data:/data/database/ -v osm-tiles:/data/tiles/ --shm-size="192m" -d overv/openstreetmap-tile-server run
```

## Additional configuration

### Using an alternate style

By default the container will use openstreetmap-carto if it is not specified. However, you can modify the style at run-time. Be aware you need the style mounted at `run` AND `import` as the Lua script needs to be run:

```
  docker run \
    -e DOWNLOAD_PBF=https://download.geofabrik.de/europe/poland-latest.osm.pbf \
    -e DOWNLOAD_POLY=https://download.geofabrik.de/europe/poland.poly \
    -e NAME_LUA=sample.lua \
    -e NAME_STYLE=test.style \
    -e NAME_MML=project.mml \
    -e NAME_SQL=test.sql \
    -v /home/user/openstreetmap-carto-modified:/data/style/ \
    -v osm-data:/data/database/ \
    overv/openstreetmap-tile-server \
    import
```

### Connecting to Postgres

To connect to the PostgreSQL database inside the container, make sure to expose port 5432:

```
  docker run \
    -p 8080:80 \
    -p 5432:5432 \
    -v osm-data:/data/database/ \
    -d overv/openstreetmap-tile-server \
    run
```

Use the user `renderer` and the database `gis` to connect.

```
  psql -h localhost -U renderer gis
```

The default password is `renderer`, but it can be changed using the `PGPASSWORD` environment variable:

```
  docker run \
    -p 8080:80 \
    -p 5432:5432 \
    -e PGPASSWORD=secret \
    -v osm-data:/data/database/ \
    -d overv/openstreetmap-tile-server \
    run
```

## Performance tuning and tweaking

Details for update procedure and invoked scripts can be found [here](https://ircama.github.io/osm-carto-tutorials/updating-data/).

#### THREADS

The import and tile serving processes use 4 threads by default, but this number can be changed by setting the THREADS environment variable. For example:

```
  docker run \
    -p 8080:80 \
    -e THREADS=24 \
    -v osm-data:/data/database/ \
    -d overv/openstreetmap-tile-server \
    run
```

#### CACHE

The import and tile serving processes use 800 MB RAM cache by default, but this number can be changed by option -C. For example:

```
  docker run \
    -p 8080:80 \
    -e "OSM2PGSQL_EXTRA_ARGS=-C 4096" \
    -v osm-data:/data/database/ \
    -d overv/openstreetmap-tile-server \
    run
```

#### AUTOVACUUM

The database use the autovacuum feature by default. This behavior can be changed with `AUTOVACUUM` environment variable. For example:

```
  docker run \
    -p 8080:80 \
    -e AUTOVACUUM=off \
    -v osm-data:/data/database/ \
    -d overv/openstreetmap-tile-server \
    run
```

#### FLAT_NODES

If you are planning to import the entire planet or you are running into memory errors then you may want to enable the `--flat-nodes` option for osm2pgsql. You can then use it during the import process as follows:

```
  docker run \
    -v /absolute/path/to/luxembourg.osm.pbf:/data/region.osm.pbf \
    -v osm-data:/data/database/ \
    -e "FLAT_NODES=enabled" \
    overv/openstreetmap-tile-server \
    import
```

## Troubleshooting

#### ERROR: could not resize shared memory segment / No space left on device

If you encounter such entries in the log, it will mean that the default shared memory limit (64 MB) is too low for the container and it should be raised:

```
renderd[121]: ERROR: failed to render TILE default 2 0-3 0-3
renderd[121]: reason: Postgis Plugin: ERROR: could not resize shared memory segment "/PostgreSQL.790133961" to 12615680 bytes: ### No space left on device
```

To raise it use `--shm-size` parameter. For example:

```
  docker run \
    -p 8080:80 \
    -v osm-data:/data/database/ \
    --shm-size="192m" \
    -d overv/openstreetmap-tile-server \
    run
```

For too high values you may notice excessive CPU load and memory usage. It might be that you will have to experimentally find the best values for yourself.

#### The import process unexpectedly exits

You may be running into problems with memory usage during the import. Have a look at the "Flat nodes" section in this README.
