# OSM Vector Tile Server using OpenMapTiles and Postile

## Introduction

This tutorial aims to create a full vector tile server based on the great 
[OpenMapTiles](https://github.com/openmaptiles/openmaptiles) project.

This example is really close to the openmaptiles quickstart guide except that we don't want 
to store our data in a dockerised container. We also want to serve tiles directly 
from PostGIS instead of mbtiles. For this last point [Postile](https://github.com/oslandia/postile)
will be used to replace the generation of the MBTiles file.

## System requirements 

Docker. Minimum version is 1.12.3+.

## Database requirements

The dependencies below are extracted from this [Dockerfile](https://github.com/openmaptiles/postgis/blob/master/Dockerfile).

This procedure has been tested on Debian Sid only, you'll probably have to adapt it to your environment/OS. 

The couple PostgreSQL 10 / PostGIS 2.4.3 was tested for this tutorial.
You can simply install all the PG stuffs with: 

    sudo apt install postgresql-server-dev-10 postgresql-10-postgis-2.4

NB: **postgis >= 2.4.0** is required (with st_asmvt* functions)
    
    sudo apt install libkakasi2-dev libutf8proc-dev pandoc libicu-dev
    git clone https://github.com/openmaptiles/mapnik-german-l10n.git
    cd mapnik-german-l10n 
    make
    sudo make install

    sudo -u postgres psql 
    create user osm with password 'osm';
    create database osm with owner osm;
    \c osm 
    create extension postgis;
    create extension hstore;
    create extension unaccent;
    create extension fuzzystrmatch;
    create extension osml10n;

Storing the gateway IP of the Docker bridge interface to be able to connect from the container to our local PostgreSQL:

    $ export PG_GATEWAY=$(docker network inspect bridge | jq -r .[0].IPAM.Config[0].Gateway)

Change the PostgreSQL configuration to allow interacting with container's tools:

    # ensure listen_addresses = '*' is uncommented in postgresql.conf
    # ensure pg_hba config allows authenticating from docker "172.17.0.*" addresses 

## Prepare the tileset

Extracted from [docker-compose.yml](https://github.com/openmaptiles/openmaptiles/blob/master/docker-compose.yml) 
and the [OpenMapTiles README](https://github.com/openmaptiles/openmaptiles) 

1. Build the imposm mapping, the tm2source project and collect all SQL scripts

        mkdir -p build/openmaptiles.tm2source
        docker run -v $(pwd):/tileset -v $(pwd)/build:/sql --rm openmaptiles/openmaptiles-tools generate-tm2source openmaptiles.yaml --port=5432 --database="osm" --user="osm" --password="osm" > build/openmaptiles.tm2source/data.yml
        docker run -v $(pwd):/tileset -v $(pwd)/build:/sql --rm openmaptiles/openmaptiles-tools generate-imposm3 openmaptiles.yaml > build/mapping.yaml
        docker run -v $(pwd):/tileset -v $(pwd)/build:/sql --rm openmaptiles/openmaptiles-tools generate-sql openmaptiles.yaml > build/tileset.sql

2. Download and load OSM dataset (example used is the Rhône-Alpes area in France)
   
        wget -P data/ http://download.geofabrik.de/europe/france/rhone-alpes-latest.osm.pbf
        docker run --rm -v $(pwd)/data:/import -v $(pwd)/build:/mapping -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST=$PG_GATEWAY -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-osm:0.5
        docker run --rm -v $(pwd)/data:/import openmaptiles/generate-osmborder
        docker run --rm -v $(pwd)/data:/import -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST=$PG_GATEWAY -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-osmborder:0.4

3. Download additional non-OSM data

        docker run --rm -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST=$PG_GATEWAY -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-water:1.1
        docker run --rm -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST=$PG_GATEWAY -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-natural-earth:1.4
        docker run --rm -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST=$PG_GATEWAY -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-lakelines:1.0

4. Grab some wikipedia data used for translations (WARNING: this part can take a while since the file to download is ~30GB)
   If you don't care about translations, you can skip the first command, the second one will create the empty relations for next steps.

        docker run --rm -v $(pwd)/wikidata:/import --entrypoint /usr/src/app/download-gz.sh openmaptiles/import-wikidata:0.1
        docker run --rm -v $(pwd)/wikidata:/import -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST=$PG_GATEWAY -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-wikidata:0.1

5. Load sql wrappers used for rendering
   
        docker run --rm -v $(pwd)/build:/sql -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST="172.17.0.1" -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-sql:0.8

## Choose a Mapbox GL style

You can grab the `style.json` in this repository for testing 
(based on the [klokantech basic gl style](https://github.com/openmaptiles/klokantech-basic-gl-style))
and download the fonts needed for it:
```
mkdir fonts 
cd fonts 
wget https://github.com/openmaptiles/fonts/releases/download/v2.0/v2.0.zip
unzip v2.0.zip
rm v2.0.zip
```

## Postile

Now we are ready to serve our tiles with [Postile](https://github.com/oslandia/postile): 

    postile --help
    postile --cors --tm2 build/openmaptiles.tm2source/data.yml --style style.json --fonts fonts/

## Show me a map !!

Open the index.html with your favorite browser
