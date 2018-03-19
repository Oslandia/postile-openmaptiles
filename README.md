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

We have to find the Docker Gateway IP to be able to connect from the container to the host:

    $ docker network inspect bridge | jq .[0].IPAM.Config[0].Gateway
    "172.17.0.1"

Change the PostgreSQL configuration to allow interacting with container's tools:

    # ensure listen_addresses = '*' is uncommented in postgresql.conf
    # ensure pg_hba config allows authenticating from docker "172.17.0.*" addresses 

## Prepare the tileset

Extracted from [docker-compose.yml](https://github.com/openmaptiles/openmaptiles/blob/master/docker-compose.yml) 
and the [OpenMapTiles README](https://github.com/openmaptiles/openmaptiles) 

1. Download tools to generate tm2source for all layers

        pip install openmaptiles-tools  # inside a python 2 venv
        git clone git@github.com:openmaptiles/openmaptiles.git
        cd openmaptiles
        # make the imposm3 mapping, tm2source file and aggregate SQL layers 
        make

2. Download and load OSM dataset (example used is the Rhône-Alpes area in France)
   
        wget -P data/ http://download.geofabrik.de/europe/france/rhone-alpes-latest.osm.pbf
        docker run --rm -v $(pwd)/data:/import -v $(pwd)/build:/mapping -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST="172.17.0.1" -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-osm:0.5
        docker run --rm -v $(pwd)/data:/import openmaptiles/generate-osmborder
        docker run --rm -v $(pwd)/data:/import -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST="172.17.0.1" -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-osmborder:0.4

3. Download additional non-OSM data

        docker run --rm -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST="172.17.0.1" -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-water:0.6
        docker run --rm -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST="172.17.0.1" -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-natural-earth:1.4
        docker run --rm -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST="172.17.0.1" -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-lakelines:1.0

4. Grab some wikipedia data used for translations (WARNING: this part can take a while since the file to download is ~30GB)
   If you don't care about translations, you can skip this part.

        docker run --rm -v $(pwd)/wikidata:/import --entrypoint /usr/src/app/download-gz.sh openmaptiles/import-wikidata:0.1
        docker run --rm -v $(pwd)/wikidata:/import -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST="172.17.0.1" -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-wikidata:0.1

5. Load sql wrappers used for rendering
   
        docker run --rm -v $(pwd)/build:/sql -e POSTGRES_DB="osm" -e POSTGRES_PORT="5432" -e POSTGRES_HOST="172.17.0.1" -e POSTGRES_PASSWORD="osm" -e POSTGRES_USER="osm" openmaptiles/import-sql:0.7

## Choose a Mapbox GL style

You can grab the `style.json` in this repository for testing 
(based on the [klokantech basic gl style](https://github.com/openmaptiles/klokantech-basic-gl-style))  

## Postile

Now we are ready to serve our tiles with [Postile](https://github.com/oslandia/postile): 

    postile --help
    postile --cors --tm2 build/openmaptiles.tm2source/data.yml --style style.json 

## Show me a map !!

copy/paste this simple index.html: 

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset='utf-8' />
    <title>Postile/OpenMapTiles example</title>
    <meta name='viewport' content='initial-scale=1,maximum-scale=1,user-scalable=no' />
    <script src='https://api.tiles.mapbox.com/mapbox-gl-js/v0.44.0/mapbox-gl.js'></script>
    <link href='https://api.tiles.mapbox.com/mapbox-gl-js/v0.44.0/mapbox-gl.css' rel='stylesheet' />
    <style>
        body { margin:0; padding:0; }
        #map { position:absolute; top:0; bottom:0; width:100%; }
    </style>
</head>
<body>

<div id='map'></div>
<script>
mapboxgl.accessToken = ' ';
var map = new mapboxgl.Map({
    container: 'map', // container id
    style: 'http://localhost:8080/style.json', // stylesheet location
    center: [5.1642882824, 44.7035735379], // centering
    zoom: 6, // starting zoom
    attributionControl: false,
});
map.addControl(new mapboxgl.AttributionControl({
    compact: true,
}));
map.addControl(new mapboxgl.NavigationControl());
</script>
```
