Docker Compose Archive format
=============================

Goal
----

A Docker Compose Archive (DCA) is an archive format that contains everything to be deployed on a containers platform based on `docker-compose`.

The DCA format is a simple tar with a docker compose specification and ready-to-run docker images.

Tar compression
---------------

`gzip` compression is expected on archive.

It is fast enough to compress, fast to decompress and still save some bytes on the network.

Content
-------

Tar content files:

- `metadata`: **required**
- `context/`: **required**
- `context/docker-compose.yml`: **required**
- `images/`: **required**
- `images/app-comp--target_env-version.tar.gz`: **optional**, for each component

Metadata format
---------------

This is a `key=value` unix-like text file, with the following keys:

- `app` which is the application name as it will be known when deployed (**required**).  
    It can only contains simple alphanumeric characters along with dashes and underscores.
- `target_env` which should contains one of the following supported environment (**required**):
    - `dev`
    - `integ`
    - `recette`
    - `prod`
- `COMPONENT_version` is the scm (*git*) version of the component (**required** for each component). It helps figure out the exact deployed version.
- `COMPONENT_base_vhost` is the first part of the final DNS name of the COMPONENT (**optional** if your component is not web-based).  
    The base host will be appended to it on a *production* target environment.  
    `-ENV` and then the base host will be appended to it on any other target environment.
- `partial` with value `1` when only one or more, but not all, components are present is the archive for the application (**optional**).

Comments should start with a `#` on its own line.

Context directory
-----------------

This directory is used to provide the context of the build. It contains the `docker-compose.yml` file but also all the contextual data the application needs: files, sql dumps, etc. This allows to link contextual data to the deployment and not the application's code source.

A `docker-compose.yml` file should be prepared with services for components and related containers: frontend, backend and database for instance.

The compose version file should be the latest `2.x` version (`3.x` is not supported at the moment). As of writing this, it's `2.4`.

Use named-volumes for storage.  
Do not use any arbitrary disk location for storage/binding.  
You can use `.` for reading files in the `context` directory.

Data dumps (like postgresql dumps) could be dropped in the context directory to be loaded at database start (if the dump is not too big).

Images directory
----------------

Image files are docker images extracted in gzipped tar format.

The saved image name should have been in the `app/component:target_env-version` format.

This should be in sync with what is described in the `metadata` file, especially the `app`, `target_env` and `version` for each component.

Signature
---------

Each `.dca` file should be joined by `.sha256` file. This is the hash of the `.dca` file and should be kept with it.

For instance a `app--integ--master--master.dca` file be joined by a `app--integ--master--master.dca.sha256` file.

Verify a Docker Compose Archive
===============================

Use the `verify-dca` tool with a `dca` file. Example:

```shell_session
$ ./verify-dca myapp--integ--master--master.dca
Verify checksums
Extract archive
Verify files presence
Verify docker compose file
Verify metadata file
Verify docker image archives
  Verify myapp-back--integ-master.tar.gz image
  Verify myapp-front--integ-master.tar.gz image
OK
$ echo $?
0
```

If the exit code is 0, then it means the archive is ok.

The tool depends on python3.
