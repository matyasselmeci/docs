Installing the StashCache Cache Using Containers
================================================

This document describes how to install a StashCache cache service.  This service allows a site or regional
network to cache data frequently used on the OSG, reducing data transfer over the wide-area network and
decreasing access latency.

See [the overview](/data/stashcache/overview) for information about StashCache.

This document describes how to install the cache using containers.
If you prefer to use RPMs, see the document for [installing the StashCache Cache using RPMs].


Before Starting
---------------

Before starting the installation process, consider the following requirements:

* __Operating system:__ An operating system capable of running containers, such as RHEL 7.
* __Container runtime:__ For example, Docker.
  You must have the ability to start containers (e.g. for Docker, belong to the `docker` Unix group).
* __Network ports:__ The cache service requires the following ports open:
  * Inbound TCP port 8000 for file access via HTTP/S
  * Outbound UDP port 9930 for reporting to `xrd-report.osgstorage.org` and `xrd-mon.osgstorage.org` for monitoring
* __File system:__ Stash Cache needs persistent storage on the host to store cached data;
  at least 1 TB is recommended.
* __Hardware requirements:__ In addition to storage, we recommend that a cache has at least 10Gbps connectivity and 8 GB of RAM.

Configuring Stashcache
----------------------

In addition to the required configuration above (ports and file systems), you may also configure the behavior of your cache with the following variables using an enviroment variable file:

Where the environment file on the docker host, `/opt/xcache/.env`, has (at least)the following contents:
```file
XC_RESOURCENAME=ProductionCache
XC_ROOTDIR=/cache
```

### Optional Configuration ###

Further behaviour of the stashcache can be cofigured by setting the followin in the enviroment variable file

- `XC_SPACE_HIGH_WM`: High watermark for disk usage;
      when usage goes above the high watermark, the cache deletes until it hits the low watermark
- `XC_SPACE_LOW_WM`: Low watermark for disk usage;
      when usage goes above the high watermark, the cache deletes until it hits the low watermark
- `XC_PORT`: TCP port that XCache listens on
- `XC_RAMSIZE`: Amount of memory to use for storing blocks before writting them to disk. (Use higher for slower disks).
- `XC_BLOCKSIZE`: The size of the blocks in the cache
- `XC_PREFETCH`: Number of blocks to prefetch from a file at once.
       This is a value of how aggressive the cache is to request portions of a file. Set to `0` to disable

### Disabling OSG monitoring ###

By default, XCache reports to the OSG so that OSG staff can monitor the health of data federations.
If you would like to report monitoring information to another destination, you can disable the OSG monitoring by setting
the following in your environment variable configuration:

```file
DISABLE_OSG_MONITORING = true
```

!!! warning
    Do not disable OSG monitoring in any service that is to be used for any other than testing

Running Stashcache
------------------

To run the container, use docker run with the following options, replacing the text within angle brackets with your own values:

```console
user@host $ docker run --rm --publish <HOST PORT>:8000 \
             --volume <HOST PARTITION>:/cache \
             --env-file=/opt/xcache/.env \
             opensciencegrid/stash-cache:stable
```

It is recommended to use a container orchestration service such as [docker-compose](https://docs.docker.com/compose/) or [kubernetes](https://kubernetes.io/), or start the StashCache container with systemd.

### Running Stashcache on container with systemd

An example systemd service file for StashCache.
This will require creating the environment file in the directory `/opt/xcache/.env`. 

!!! note
    This example systemd file assumes `<HOST PORT>` is `8000` and  `<HOST PARTITION>` is `/srv/cache`.

```file
[Unit]
Description=StashCache Container
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
Restart=always
ExecStartPre=-/usr/bin/docker stop %n
ExecStartPre=-/usr/bin/docker rm %n
ExecStartPre=/usr/bin/docker pull opensciencegrid/stash-cache:stable
ExecStart=/usr/bin/docker run --rm --name %n --publish 8000:8000 --volume /srv/cache:/cache --env-file /opt/xcache/.env opensciencegrid/stash-cache:stable

[Install]
WantedBy=multi-user.target
```

This systemd file can be saved to `/etc/systemd/system/docker.stash-cache.service` and started with:

```console
root@host $ systemctl enable docker.stash-cache
root@host $ systemctl start docker.stash-cache
```

!!! warning
    You must [register](/data/stashcache/install-cache/#registering-the-cache) the cache before considering it a production service.



### Network Optimization ###

For caches that are connected to NIC's over `40Gbps` we recommend to disable the virtualized network and "bind" the container to the host network:

```console
user@host $ docker run --rm  \
             --network="host" \
             --volume <HOST PARTITION>:/cache \
             --env-file=/opt/xcache/.env \
             opensciencegrid/stash-cache:stable
```

### Memory Optimization ###

Stashcache uses the hosts memory in two ways:

1. Uses the own linux kernel as a way to cache files read from disk
1. As a buffer for writting blocks of files first in memory and then in disk (to account for slow disks).

An easy way to increase the performance of stashcache is to assign it more memory. You can use the [docker option](https://docs.docker.com/config/containers/resource_constraints/#limit-a-containers-access-to-memory) `--memory` and set it up to at least twice of `XC_RAMSIZE`.

```console
user@host $ docker run --rm --publish <HOST PORT>:8000 \
             --memory=64g \
             --volume <HOST PARTITION>:/cache \
             --env-file=/opt/xcache/.env \
             opensciencegrid/stash-cache:stable
```


### Multiple Disks ###

For caches that store over `10 TB` or that have assigned space for storing the cached files over multiple partitions (`/partition1, /partition2, ...`) we recommend the following.

1. Create a config file `90-my-stash-cache-disks.cfg` with the following contents

        :::file
        pfc.spaces data
        oss.space data /data1
        oss.space data /data2
        .
        .
        .

1. Run the container with the following options:

        :::console
        user@host $ docker run --rm --publish <HOST PORT>:8000 \
             --volume <HOST PARTITION>:/cache \
             --volume /partition1:/data1 \
             --volume /partition2:/data2 \
             --volume /opt/xcache/90-my-stash-cache-disks.cfg:/etc/xrootd/config.d/90-stash-cache-disks.cfg \
             --env-file=/opt/xcache/.env \
             opensciencegrid/stash-cache:stable

!!! note
    Under this configuration the `<HOST PARTITION>` is not used to store the files rather to store symlinks to the files in `/partition1` and `/partition2`

!!! warning
    For over 100 TB of assigned space we highly encourage to use this setup and mount `<HOST PARTITION>` in solid state disks or NVME.


Validating StashCache
---------------------

For example, if you've chosen `8212` as your host port, you can verify that it worked with the command:

```console
user@host $ curl http://localhost:8212/user/dweitzel/public/blast/queries/query1
```

Which should output:

```
>Derek's first query!
MPVSDSGFDNSSKTMKDDTIPTEDYEEITKESEMGDATKITSKIDANVIEKKDTDSENNITIAQDDEKVSWLQRVVEFFE
```

Getting Help
------------

To get assistance, please use the [this page](/common/help) or contact <help@opensciencegrid.org> directly.

