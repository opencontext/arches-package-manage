# arches-package-manage
This repo will mainly contain pseudo-code and narrative descriptions of how to manage Arches packages. The initial focus will be on migrating Arches v6.x packages to Arches v7.x.




## Migrate a Package from Arches v6.x to v7.x
This workflow (not an *Arches workflow*, just a workflow in a more general sense of the term) uses Docker to deploy Arches v6 and v7 instances, some Python executed in a shell to manage some files and execute some bash/shell commands, and a little SQL to modify databases. While I'm sure this can get streamlined more, it's probably not that useful to fully automate this process since package migration from v6 to v7 is unlikely to be a super recurrent kind of need.


### Arches via Docker
To deploy different versions of Arches, we'll use this: https://github.com/opencontext/arches-via-docker , specifically:

  1. Branch `v6-local` for deplopying Arches v6.x on a local machine (no Nginx or Apache web server)
  2. Branch `local` for deplopying Arches v7.x on a local machine (no Nginx or Apache web server)

In setting up Arches via Docker, be sure to make a directory called `arches_data` that is a SIBLING directory as the `arches-via-docker` directory (git repo) as so:

```
  /my_stuff/
  └─ arches-via-docker
  └─ arches_data
      └─ arches_pkg_v6
```

The `arches_data` will be mounted into the Docker container running Arches to make it more convenient to share files between the Arches container and the host machine's file-system. Inside the `arches_data`, store your version 6 Arches package (for example: `arches_pkg_v_6`).


  
