
Update from master repo
1. git fetch origin
   git merge origin/master

 Download the data 
2. data folder: curl https://download.geofabrik.de/north-america/mexico-latest.osm.pbf --output mexico-latest.osm.pbf
                curl  https://www.nominatim.org/data/wikimedia-importance.sql.gz --output wikimedia-importance.sql.gz
                curl https://www.nominatim.org/data/wikipedia_article.sql.bin --output wikipedia_article.sql.bin
                curl https://www.nominatim.org/data/wikipedia_redirect.sql.bin --output wikipedia_redirect.sql.bin 
                gzip -d wikimedia-importance.sql.gz

Build the container
3. docker build --no-cache --pull --rm -t dataplor/nominatim .

Initialize Database
4. docker run -t -v /Volumes/CRUCIAL_SSD/code/nominatim-docker/nominatim-data:/data dataplor/nominatim  sh /app/init.sh /data/mexico-latest.osm.pbf postgresdata 8

Start data container
5. docker run --restart=always -p 6432:5432 -d -v /Volumes/CRUCIAL_SSD/code/nominatim-docker/nominatim-data/postgresdata:/var/lib/postgresql/11/main dataplor/nominatim sh /app/startpostgres.sh






# Nominatim Docker (Nominatim version 3.5)

1. Build
  ```
  docker build --pull --rm -t nominatim .
  ```
  See below for optional build arguments to include postcode data in your image.

2. Copy <your_country>.osm.pbf to a local directory (i.e. /home/me/nominatimdata)

3. Initialize Nominatim Database
  ```
  docker run -t -v /home/me/nominatimdata:/data nominatim  sh /app/init.sh /data/<your_country>.osm.pbf postgresdata 4
  ```
  Where 4 is the number of threads to use during import. In general the import of data in postgres is a very time consuming
  process that may take hours or days. If you run this process on a multiprocessor system make sure that it makes the best use
  of it. You can delete the /home/me/nominatimdata/<your_country>.osm.pbf once the import is finished.


4. After the import is finished the /home/me/nominatimdata/postgresdata folder will contain the full postgress binaries of
   a postgis/nominatim database. The easiest way to start the nominatim as a single node is the following:
   ```
   docker run --restart=always -p 6432:5432 -p 7070:8080 -d --name nominatim -v /home/me/nominatimdata/postgresdata:/var/lib/postgresql/12/main nominatim bash /app/start.sh
   ```

5. Advanced configuration. If necessary you can split the osm installation into a database and restservice layer

   In order to set the  nominatib-db only node:

   ```
   docker run --restart=always -p 6432:5432 -d -v /home/me/nominatimdata/postgresdata:/var/lib/postgresql/12/main nominatim sh /app/startpostgres.sh
   ```
   After doing this create the /home/me/nominatimdata/conf folder and copy there the docker/local.php file. Then uncomment the following line:

   ```
   @define('CONST_Database_DSN', 'pgsql://nominatim:password1234@192.168.1.128:6432/nominatim'); // <driver>://<username>:<password>@<host>:<port>/<database>
   ```

   You can start the  nominatib-rest only node with the following command:

   ```
   docker run --restart=always -p 7070:8080 -d -v /home/me/nominatimdata/conf:/data nominatim sh /app/startapache.sh
   ```

6. Configure incremental update. By default CONST_Replication_Url configured for Monaco.
If you want a different update source, you will need to declare `CONST_Replication_Url` in local.php. Documentation [here] (https://github.com/openstreetmap/Nominatim/blob/master/docs/admin/Import-and-Update.md#updates). For example, to use the daily country extracts diffs for Gemany from geofabrik add the following:
  ```
  @define('CONST_Replication_Url', 'http://download.geofabrik.de/europe/germany-updates');
  ```

  Now you will have a fully functioning nominatim instance available at : [http://localhost:7070/](http://localhost:7070). Unlike the previous versions
  this one does not store data in the docker context and this results to a much slimmer docker image.

# Postcodes

Nominatim requires additional data files to accurately assign postcodes data in the US and Great Britain (Northern Ireland postcodes are not included in this file) as described in [these Nominatim docs](https://nominatim.org/release-docs/latest/admin/Import-and-Update/#downloading-additional-data). Without this data, you may get incorrect postcodes for some address lookups.

These data files aren't downloaded by default, but you can add them with additional arguments at the build stage. To include the US postcode data file, add "--build-arg with_postcodes_us=1" to the command line in stage 1, above. To include GB postcodes, run with "--build-arg with_postcodes_gb=1". You can run with both at once if desired, eg:
  ```
  docker build --pull --rm -t nominatim --build-arg with_postcodes_us=1 --build-arg with_postcodes_gb=1 .
  ```

# Update

Full documentation for Nominatim update available [here](https://github.com/openstreetmap/Nominatim/blob/master/docs/admin/Import-and-Update.md#updates). For a list of other methods see the output of:
  ```
  docker exec -it nominatim sudo -u postgres ./src/build/utils/update.php --help
  ```

To initialise the updates run
  ```
  docker exec -it nominatim sudo -u postgres ./src/build/utils/update.php --init-updates
  ```

The following command will keep your database constantly up to date:
  ```
  docker exec -it nominatim sudo -u postgres ./src/build/utils/update.php --import-osmosis-all
  ```
If you have imported multiple country extracts and want to keep them
up-to-date, have a look at the script in
[issue #60](https://github.com/openstreetmap/Nominatim/issues/60).

# Upgrade Guide for 3.5.x

## Upgrade to 3.5.2

As referenced in the Nominatim release ([https://github.com/osm-search/Nominatim/releases/tag/v3.5.2](https://github.com/osm-search/Nominatim/releases/tag/v3.5.2)) you need to make sure to run the following command after the upgrade: 

```
docker exec -it nominatim sudo -u postgres ./src/build/utils/setup.php --create-functions --enable-diff-updates
```

# Docker image upgrade to 3.5

With 3.5 we have switched to Ubuntu 20.04 (LTS) which uses PostgreSQL 12. If you want to reuse your old data dictionary without importing the data again you have to make sure to migrate the data from PostgreSQL 11 to 12 with a command like ```pg_upgrade``` (see: [https://www.postgresql.org/docs/current/pgupgrade.html](https://www.postgresql.org/docs/current/pgupgrade.html)). 

You can try a script like [https://github.com/tianon/docker-postgres-upgrade](https://github.com/tianon/docker-postgres-upgrade) with some modifications.