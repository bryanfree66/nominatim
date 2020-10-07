
Update from master repo
1. git fetch origin
   git merge origin/master

Download the Mexico data 
2. data folder: curl https://download.geofabrik.de/north-america/mexico-latest.osm.pbf --output mexico-latest.osm.pbf

Build the container
2. docker build --no-cache --pull --rm -t dataplor/nominatim .

Initialize Database
3. docker run -t -v /Volumes/CRUCIAL_SSD/code/nominatim-docker/nominatim-data:/data dataplor/nominatim  sh /app/init.sh /data/mexico-latest.osm.pbf postgresdata 8

Get shell into container
5. docker exec -it 52227ca00a65 /bin/bash

6. sudo -u postgres -i

7. cd /app/src/data-sources/wikipedia-wikidata

8. ./import_wikipedia.sh

9. Start Container
docker run --restart=always -p 6432:5432 -p 7070:8080 -d --name nominatim -v /Volumes/CRUCIAL_SSD/code/nominatim-docker/nominatim-data/postgresdata:/var/lib/postgresql/12/main nominatim bash /app/start.sh (one container)

docker run --restart=always -p 6432:5432 -d -v /Volumes/CRUCIAL_SSD/code/nominatim-docker/nominatim-data/postgresdata:/var/lib/postgresql/11/main dataplor/nominatim sh /app/startpostgres.sh (DB ONLY)

After doing this create the /home/me/nominatimdata/conf folder and copy there the docker/local.php file. Then uncomment the following line:

   ```
   @define('CONST_Database_DSN', 'pgsql://nominatim:password1234@192.168.1.128:6432/nominatim'); // <driver>://<username>:<password>@<host>:<port>/<database>
   ```

   docker run --restart=always -p 7070:8080 -d -v /home/me/nominatimdata/conf:/data nominatim sh /app/startapache.sh (front end)

 
  wget  https://www.nominatim.org/data/wikimedia-importance.sql.gz 
  wget https://www.nominatim.org/data/wikipedia_article.sql.bin 
  wget https://www.nominatim.org/data/wikipedia_redirect.sql.bin
  gzip -d wikimedia-importance.sql.gz


6. Configure incremental update. By default CONST_Replication_Url configured for Monaco.
If you want a different update source, you will need to declare `CONST_Replication_Url` in local.php. Documentation [here] (https://github.com/openstreetmap/Nominatim/blob/master/docs/admin/Import-and-Update.md#updates). For example, to use the daily country extracts diffs for Gemany from geofabrik add the following:
  ```
  @define('CONST_Replication_Url', 'http://download.geofabrik.de/europe/germany-updates');
  ```

  Now you will have a fully functioning nominatim instance available at : [http://localhost:7070/](http://localhost:7070). Unlike the previous versions
  this one does not store data in the docker context and this results to a much slimmer docker image.

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
