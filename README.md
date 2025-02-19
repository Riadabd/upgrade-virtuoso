# Upgrade with database size optimizations:

Before any upgrade starts, it is best to back up your `virtuoso` container:
* Stop the stack: `docker compose down`
* Start the maintenance frontend banner (if any exists)
* Start the database container: `docker compose up -d virtuoso && docker compose logs -ft virtuoso`
* Start the backup process using `virtuoso-backup.sh` inside the `/data/useful-scripts/` directory:
```shell
/data/useful-scripts/virtuoso-backup.sh `docker ps --filter "label=com.docker.compose.project=[YOUR-PROJECT]" --filter "label=com.docker.compose.service=[YOUR-DB-CONTAINER]" --format "{{.Names}}"`
```

Replace `[YOUR-PROJECT]` with your intended project (e.g., `app-digitaal-loket`, `app-worship-decisions-database`, etc...) and `[YOUR-DB-CONTAINER]` with your virtuoso container name (e.g., `virtuoso` or `triplestore` for some apps).

> Note: The upgrade section in [this README](https://github.com/redpencilio/docker-virtuoso/blob/dec36bd4a5ed4191c42e0a9b5ca979d67bc22cfe/README.md#upgrading) describes the process needed to upgrade the database and reduce its size. The steps have been copied from Niel's notes with minor adjustments:

## Steps for `Virtuoso`:

### Dump N-Quads (output files should have an `.nq.gz` extension)

When upgrading it is recommend (and sometimes required) to first dump to quads using the `dump_nquads` procedure:

```shell
docker compose exec virtuoso isql-v
```

```shell
SQL> dump_nquads ('dumps', 1, 1000000000, 1);
```

> Note: This procedure can take a couple of minutes.

Confirm the procedure works by checking the contents of `data/db/dumps`.

### Stop `Virtuoso`

```shell
docker compose down
```

### Remove the old db and corresponding files

After stopping the containers, rename the `dumps` folder to `toLoad`:

```shell
mv data/db/dumps data/db/toLoad
```

and make sure to delete the following files:

* `.data_loaded`
* `.dba_pwd_set`
* `virtuoso.db`
* `virtuoso.trx`
* `virtuoso.pxa`
* `virtuoso-temp.db`

```shell
rm -i data/db/virtuoso.{db,trx,pxa} data/db/virtuoso-temp.db data/db/.data_loaded data/db/.dba_pwd_set
```

### Update the `Virtuoso` version in `docker-compose.yml`

At this moment, the latest release of `Virtuoso` is `v1.3.0-rc.1`. The release candidate is nothing much to worry about; our builds of `Virtuoso` are based on stable releases of that database, but they are typically tagged as release candidates since we have not internally tried them out yet.

```diff
virtuoso:
-    image: redpencil/virtuoso:1.0.0
+    image: redpencil/virtuoso:1.3.0-rc.1
```

You can also opt to go with the *stable* version of `Virtuoso` that has been in use in most of our apps for more than a year.

```diff
virtuoso:
-    image: redpencil/virtuoso:1.0.0
+    image: redpencil/virtuoso:1.2.0
```

### Start the database again and monitor logs for any weird behavior:

```shell
docker compose up -d virtuoso && docker compose logs -ft virtuoso
```

You can monitor the dump loading state as follows:

```shell
docker compose exec virtuoso isql-v
```

```shell
SQL> select * from DB.DBA.load_list;
```

The query will output a table; you can validate the process by checking the `ll_state` of the load. A `ll_state` of 2 means a file has been successfully loaded.

### Remove `toLoad/` folder

Once you confirm `ll_state` of 2 for all nquad dump files, remove the `toLoad/` folder:

```shell
rm -rf data/db/toLoad
```
