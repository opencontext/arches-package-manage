# arches-package-manage
This repo will mainly contain pseudo-code and narrative descriptions of how to manage Arches packages. The initial focus will be on migrating Arches v6.x packages to Arches v7.x.




## Migrate a Package from Arches v6.x to v7.x
This workflow (not an *Arches workflow*, just a workflow in a more general sense of the term) uses Docker to deploy Arches v6 and v7 instances, and execute some bash/shell commands to run Arches migrations and interact with the databases. While I'm sure this can get streamlined more, it's probably not that useful to fully automate this process since package migration from v6 to v7 is unlikely to be a super recurrent kind of need.


### Arches via Docker
To deploy different versions of Arches, we'll use this: https://github.com/opencontext/arches-via-docker , specifically:

  1. Branch `v6-local` for deplopying Arches v6.x on a local machine (no Nginx or Apache web server)
  2. Branch `local` for deplopying Arches v7.x on a local machine (no Nginx or Apache web server)

In setting up Arches via Docker, be sure to make a directory called `arches_data` that is a SIBLING of the `arches-via-docker` directory (git repo) as so:

```
  /my_stuff/
  └─ arches-via-docker
  └─ arches_data
      └─ arches_pkg_v6
```

The `arches_data` will be mounted into the Docker container running Arches to make it more convenient to share files between the Arches container and the host machine's file-system. Inside the `arches_data`, store your version 6 Arches package (for example: `arches_pkg_v_6`). Please note that the files (`*.xml`, `ontolog_config.json`) that make up a given ontology must be grouped together in a subdirectory of the package's `ontologies` directory (see: https://arches.readthedocs.io/en/stable/administering/ontologies-in-arches/#loading-an-ontology).


### Step 1: Launch Arches v6.x
In a terminal, CD into the `arches-via-docker` and switch to the `v6-local` branch. Then make a copy the default edit_dot_env file and save it as a file called `.env`:

``` bash
git checkout v6-local
cp edit_dot_env .env
```

Now start up the Arches v6 instance along with dependencies for running on your localhost machine.
``` bash

# Delete any old Arches v6 images that may exist. This may give an error if you don't have an Arches image existing.
docker image rm arches

docker compose up
```

You can follow along and watch the long process of installing all the dependencies and launching Arches v6. Once this is done you should be able to verify that it is running properly by using your browser to access Arches at `http://127.0.0.1:8004` (Note the non-standard port, 8004. We're using that port so we don't conflict with other processes that maybe using the more usual port 8000).


### Step 2: Load the v6 Package into the Arches v6.x Instance
Open another terminal and load the v6 version of your Arches package into your Arches v6.x instance deployed via Docker. You'll note that the `arches_data` directory is mounted and usable by the `arches` Docker container. From the perspective of the `arches` Docker container, the path the v6 version of your Arches package is `/arches_data/arches_pkg_v6`.

``` bash
docker exec -it arches python3 manage.py packages -o load_package -s '/arches_data/arches_pkg_v6'
# Also load RDM related concepts, collections if present:
docker exec -it arches python3 manage.py packages -o import_reference_data -s '/arches_data/arches_pkg_v6/reference_data/concepts/Arches8001.skos' -ow 'ignore' -st 'keep'
docker exec -it arches python3 manage.py packages -o import_reference_data -s '/arches_data/arches_pkg_v6/reference_data/collections/Collections 8001.skos' -ow 'ignore' -st 'keep'
```

### Step 3: Make an Export Dump of your Arches v6.x Database (with installed package)
Once you've installed the version 6 package into your Arches v6.x instance, you should now make a Postgres dump of the Arches database (that includes data from your package). Assuming you're using the default database connection:

``` bash
docker exec -it arches bash -c "pg_dump -U postgres -h arches_db  -F c -b arches_v6local > '/arches_data/arches_v6local.dump'"

```


### Step 4: Stop the Arches v6.x Docker Instance and Start Arches v7.x
At this point we should have an Arches v6 package loaded into an Arches v6 database that has been exported to a Postgres dump file. Now we can turn off the Docker containers used by Arches v6.x and then deploy Docker containers for Arches 7.x, using the following steps:

``` bash
cd arches-via-docker
docker compose down

# Once Docker has finished deactivating various containers, we can set up Arches v7.x
# checkout the branch that is for local installation of the latest version of Arches (now v7.x)
git checkout local

# Make sure we have the .env file using the defaults for v7.x (so overwrite and replace the .env file we had for v6.x)
cp edit_dot_env .env

# Delete the old Arches v6 image. We'll need to rebuild it with an Arches v7 image
docker image rm arches

# Now start up the Arches v7 instance along with dependencies for running on your localhost machine.
docker compose up
```
You can follow along and watch the long process of installing all the dependencies and launching Arches v7. Once this is done you should be able to verify that it is running properly by using your browser to access Arches at `http://127.0.0.1:8004` (Note the non-standard port, 8004. We're using that port so we don't conflict with other processes that maybe using the more usual port 8000).


### Step 5: Replace the Arches v7 Database with the Arches v6 database
Once you have a working Arches v7.x instance running in Docker, it's time to break it (!). We will import the exported Arches v6 database (that has the package data loaded into it) and then replace our Arches version 7 database with the version 6 database containing package data.

``` bash
# First restore the v6 database (containing package data) into Postgres via your Arches v7 docker container (I know, confusing)...
docker exec -it arches bash -c "pg_restore --create --clean -U postgres -h arches_db -d postgres '/arches_data/arches_v6local.dump'"
```

Note: Sometimes the above command fails (for some reason that I don't have time to diagnose). You can get an error like:
```
pg_restore: while PROCESSING TOC:
pg_restore: from TOC entry 5249; 1262 20641 DATABASE arches_v6local postgres
pg_restore: error: could not execute query: ERROR:  database "arches_v6local" does not exist
Command was: DROP DATABASE arches_v6local;
pg_restore: warning: errors ignored on restore: 1
```

In this event, simply try the command again, and it seems to work:
``` bash
# Second attempt works...
docker exec -it arches bash -c "pg_restore --create --clean -U postgres -h arches_db -d postgres '/arches_data/arches_v6local.dump'"
```


``` bash
# Now drop the existing Arches v7 database and recreate it using the Arches v6 database as the template.
docker exec -it arches psql -U postgres -tc "DROP DATABASE arches_slocal WITH (FORCE);"
docker exec -it arches psql -U postgres -tc "SELECT pg_terminate_backend(pid) from pg_stat_activity where datname='arches_v6local'";
docker exec -it arches psql -U postgres -tc "CREATE DATABASE arches_slocal WITH TEMPLATE arches_v6local;"
```


### Step 6: Do various Arches 7 Migrations
We're now at the point where Arches version 7 software is talking to an Arches version 6 database. We can now use the software to run various migration and update processes to transform the version 6 database into a working version 7 database, and in the process, update the version 6 package information.
``` bash
docker exec -it arches python3 manage.py migrate
docker exec -it arches python3 manage.py updateproject
docker exec -it arches python3 manage.py es reindex_database

# You'll likely need to run the yarn build_development again because the frontend will likely be broken
docker exec -it arches bash -c "yarn --cwd /arches_app/arches_slocal/arches_slocal build_development"

```
Once this is done you should be able to verify that it is running properly by using your browser to access Arches at `http://127.0.0.1:8004`. Hopefully the Arches v7 will have the package happily installed!


### Step 7: Export your Arches v7 Package
If everything worked (yep, good luck with that), then you should be ready to export the Arches v7 package to share with other Arches administrators:

``` bash

docker exec -it arches python3 manage.py packages -o create_package -d '/arches_data/arches_pkg_v7'

# NOTE: Check to make sure the branches and resource_models actually exported (they sometimes don't get exported with the command above).
# If branches are missing, do this:
docker exec -it arches python3 manage.py packages -o export_graphs -d '/arches_data/arches_pkg_v7/graphs/branches' -g 'branches'

# If resource_models are missing, do this:
docker exec -it arches python3 manage.py packages -o export_graphs -d '/arches_data/arches_pkg_v7/graphs/resource_models' -g 'resource_models'


# Update permissions so users outside of the Docker host can have full permissions to the package.
docker exec -it arches bash -c 'chmod 777 -R /arches_data/arches_pkg_v7'

