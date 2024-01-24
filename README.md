# arches-package-manage
This repo will mainly contain pseudo-code and narrative descriptions of how to manage Arches packages. The initial focus will be on migrating Arches v6.x packages to Arches v7.x.




## Migrate a Package from Arches v6.x to v7.x
This workflow (not an *Arches workflow*, just a workflow in a more general sense of the term) uses Docker to deploy Arches v6 and v7 instances, some Python executed in a shell to manage some files and execute some bash/shell commands, and a little SQL to modify databases. While I'm sure this can get streamlined more, it's probably not that useful to fully automate this process since package migration from v6 to v7 is unlikely to be a super recurrent kind of need.


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

The `arches_data` will be mounted into the Docker container running Arches to make it more convenient to share files between the Arches container and the host machine's file-system. Inside the `arches_data`, store your version 6 Arches package (for example: `arches_pkg_v_6`).


### Step 1: Launch Arches v6.x
In a terminal, CD into the `arches-via-docker` and switch to the `v6-local` branch. Then make a copy the default edit_dot_env file and save it as a file called `.env`:

``` bash
git checkout v6-local
cp edit_dot_env .env
```

Now start up the Arches v6 instance along with dependencies for running on your localhost machine.
``` bash
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
docker exec -it arches bash -c "pg_dump -U postgres -h arches_db  -F c -b arches_v6 > '/arches_data/arches_v6.dump'"

```
