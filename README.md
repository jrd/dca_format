Docker Compose Archive format
=============================

Goal
----

A Docker Compose Archive (DCA) is an archive format that contains everything to be deployed on a containers platform based on `docker-compose`.

A `.dca` file is defined by a DCA format as a **compressed tar** with a docker compose specification and ready-to-run docker images.

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
    `app` is the application name, `comp` the component name, `target_env` the environment and `version` the component version.
- `proxy/`: **optional**
- `proxy/comp-server`: **optional**
- `proxy/comp-location`: **optional**
    `comp` is the component name

Metadata format
---------------

This is a `key=value` unix-like text file, with the following keys:

- `version` for `DCA` file version (**optional** default to `1`). It could take value `1` or `2`.
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

`docker-compose.yml` allowed keys
---------------------------------

**Version 2** format only allow the following keys in the `docker-compose.yml` file:
- Service config reference:
    - `build`
        - `context`
        - `dockerfile`
        - `args`
        - `cache_from`
        - `extra_hosts`
        - `labels`
        - `shm_size`
        - `target`
    - `cap_drop`
    - `command`
    - `depends_on`
    - `entrypoint`
    - `env_file`
    - `environment`
    - `expose`
    - `extends`
        - `file`
        - `service`
    - `extra_hosts`
    - `group_add`
    - `healthcheck`
        - `test`
        - `interval`
        - `timeout`
        - `retries`
        - `start_period`
        - `disable`
    - `image`
    - `init`
    - `labels`
    - `networks`
    - `pid` but not the `host` value
    - `scale`
    - `stop_grace_period`
    - `stop_signal`
    - `sysctls`
    - `tmpfs`
    - `ulimits`
    - `volumes` but **SOURCE** cannot only start with an alphabetic caracter or `./`. One can use *short* or *long* syntax.
    - `volumes_from`
    - `restart`
    - `shm_size`
    - `tty`
    - `user`
    - `working_dir`
- Volume config reference
    - `external`
    - `labels`
    - `name`
- Network config reference
    - `external`
    - `internal`
    - `labels`
    - `name`
**Version 2** format also specify the following global extension keys in the `docker-compose.yml` file:
- `x-resources`, then for each service:
    - `disk` max required disk usage, default to `100M`. Max value for this field depends on server configuration.
    - `memory` max required user memory, default to `300M`. Max value for this field depends on server configuration.
    - `memory_avg` average user memory, default to `100M` (**⅓** of `memory`. This should be much lower than `memory`.
    - `cpu` default to `4`. You can choose a value from `1` to `16` to indicate the service cpu proportion regarding to other application services.  
        Be careful when using this setting.
If `disk`, `memory` or `memory_avg` values are beyond the max server configuration, the application will fail to deploy.

**Version 1** format cannot specify the `x-resources` and will have the following hard-coded restrictions on disk, memory and cpu:
    - `disk`: `1G`
    - `memory`: `1G`
    - `memory_avg`: `300M`
    - `cpu`: `4`

Size units could be `B`, `K`, `M` or `G`. Decimal values is allowed, separated with a dot `.`: `1.5G`.

Images directory
----------------

Image files are docker images extracted in gzipped tar format.

The saved image name should have been in the `app/component:target_env-version` format.

This should be in sync with what is described in the `metadata` file, especially the `app`, `target_env` and `version` for each component.

Proxy directory
---------------

This directory is taken into account only at format **version 2** minimum.

`COMPONENT-server` is a `nginx` server configuration file that could be specified to tweak some [server section](https://nginx.org/en/docs/http/ngx_http_core_module.html#server) values.

`COMPONENT-location` is a `nginx` location configuration file that could be specified to tweak some [location section](https://nginx.org/en/docs/http/ngx_http_core_module.html#location) values.

`COMPONENT` is one web-based component name (i.e. a - `COMPONENT_base_vhost` should be defined in `metadata` file).  

Checksum
--------

Each `.dca` file should be accompanied by a `.sha256` file in the same directory. It contents the `.dca` file hash and should be kept with it.

For instance a `app--integ--master--master.dca` file be accompanied by a `app--integ--master--master.dca.sha256` file.

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

License
-------

[MIT](./LICENSE)
