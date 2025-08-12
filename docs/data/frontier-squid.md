title: Install the Frontier Squid HTTP Caching Proxy

# Install the Frontier Squid HTTP Caching Proxy

Frontier Squid is a distribution of the well-known [squid HTTP caching
proxy software](http://squid-cache.org) that is optimized for use with
applications on the Worldwide LHC Computing Grid (WLCG). It has
[many advantages](https://twiki.cern.ch/twiki/bin/view/Frontier/InstallSquid#Why_use_frontier_squid_instead_o)
over regular squid for common distributed computing applications, especially Frontier
and CVMFS. The OSG distribution of frontier-squid is a straight rebuild of the
upstream frontier-squid package for the convenience of OSG users.

This document is intended for System Administrators who are installing
`frontier-squid`, the OSG distribution of the Frontier Squid software.

## Frontier Squid Is Recommended

OSG recommends that all sites run a caching proxy for HTTP and HTTPS
to help reduce bandwidth and improve throughput. To that end, Compute
Element (CE) installations include Frontier Squid automatically. We
encourage all sites to configure and use this service, as described
below.

For large sites that expect heavy load on the proxy, it is best to run the proxy on its own host.
If you are unsure if your site qualifies, we recommend initially running the proxy on your CE host and monitoring its
bandwidth.
If the network usage regularly peaks at over one third of the bandwidth capacity, move the proxy to a new host.

## Before Starting

Before starting the installation process, consider the following points (consulting [the Reference section below](#reference) as needed):

-   **Hardware requirements:** If you will be supporting the Frontier application at your site, review the
    [hardware recommendations](https://twiki.cern.ch/twiki/bin/view/Frontier/InstallSquid#Hardware)
    to determine how to size your equipment.
-   **User IDs:** If it does not exist already, the installation will create the `squid` Linux user
-   **Network ports:**
    Clients within your cluster (e.g., OSG user jobs) will communicate with Frontier Squid on port 3128 (TCP).
    Additionally, central infrastructure will monitor Frontier Squid through port 3401 (UDP);
    see [this section](#networking) for more details.

As with all OSG software installations, there are some one-time (per host) steps to prepare in advance:

- Ensure the host has [a supported operating system](../release/supported_platforms.md)
- Obtain root access to the host
- Prepare the [required Yum repositories](../common/yum.md)

### Installing Frontier Squid

To install Frontier Squid, make sure that your host is up to date before installing the required packages:

1. Clean yum cache:

        ::console
        root@host # yum clean all --enablerepo=*

2. Update software:

        :::console
        root@host # yum update

    This command will update **all** packages

3. Install Frontier Squid:

        :::console
        root@host # yum install frontier-squid
        
## Configuring Frontier Squid

### Configuring the Frontier Squid Service

To configure the Frontier Squid service itself:

1.  Follow the
    [Configuration section of the upstream Frontier Squid documentation](https://twiki.cern.ch/twiki/bin/view/Frontier/InstallSquid#Configuration).
2.  Enable, start, and test the service (as described below).
3.  Register the squid (also as described [below](#registering-frontier-squid)).

!!! Note
    An important difference between the standard Squid software and
    the Frontier Squid variant is that Frontier Squid changes are in
    `/etc/squid/customize.sh` instead of `/etc/squid/squid.conf`.

### Configuring the OSG CE

To configure the OSG Compute Entrypoint (CE) to know about your Frontier Squid service:

1.  On your CE host (which may be different than your Frontier Squid host), edit `/etc/osg/config.d/01-squid.ini`
    -   Make sure that `enabled` is set to `True`
    -   Set `location` to the hostname and port of your Frontier Squid
        service (e.g., `my.squid.host.edu:3128`)
    -   Leave the other settings at `DEFAULT` unless you have specific
        reasons to change them

2.  Run `osg-configure -c` to propagate the changes on your CE.

!!! Note
    You may want to finish other CE configuration tasks before running
    `osg-configure`. Just be sure to run it once before starting CE
    services.

## Using Frontier-Squid

Start the frontier-squid service and enable it to start at boot time. As a reminder, here are common service commands (all run as `root`):

| To...                                   | Run the command...                            |
| :-------------------------------------- | :-------------------------------------------- |
| Start the service                         | `systemctl start frontier-squid`              |
| Stop the service                          | `systemctl stop frontier-squid`               |
| Enable the service to start on boot       | `systemctl enable frontier-squid`             |
| Disable the service from starting on boot | `systemctl disable frontier-squid`            |

## Validating Frontier Squid

As any user on another computer, do the following (where
`<MY.SQUID.HOST.EDU>` is the fully qualified domain name of your
squid server):

``` console hl_lines="1"
user@host $ export http_proxy=http://<MY.SQUID.HOST.EDU>:3128
user@host $ wget -qdO/dev/null http://frontier.cern.ch 2>&1|grep Cache
Cache-Status: MY.SQUID.HOST.EDU;fwd=miss;detail=mismatch
user@host $ wget -qdO/dev/null http://frontier.cern.ch 2>&1|grep Cache
Cache-Status: MY.SQUID.HOST.EDU;hit;detail=match
```

The above is how the responses look starting with frontier-squid-6.
For older versions they look like this:

``` console
user@host $ wget -qdO/dev/null http://frontier.cern.ch 2>&1|grep Cache
X-Cache: MISS from <MY.SQUID.HOST.EDU>
user@host $ wget -qdO/dev/null http://frontier.cern.ch 2>&1|grep Cache
X-Cache: HIT from <MY.SQUID.HOST.EDU>
```

If the grep doesn't print anything, try removing it from the pipeline
to see if errors are obvious. If the second try is a miss again,
something is probably wrong with the squid cache writes. Look at the squid
[access.log file](https://twiki.cern.ch/twiki/bin/view/Frontier/InstallSquid#Log_file_contents)
to try to see what's wrong.

## Registering Frontier Squid

To register your Frontier Squid host, follow the general registration instructions
[here](../common/registration.md#new-resources) with the following Frontier Squid-specific details.
Alternatively, [contact us](../common/help.md) for assistance with the registration process.

1.  Add a `Squid:` section to the `Services:` list, with any relevant fields for that service.
    This is a partial example:

        :::console
        ...
        FQDN: <FULLY QUALIFIED DOMAIN NAME>
        Services:
          Squid:
            Description: Generic squid service
        ...

    Replacing `<FULLY QUALIFIED DOMAIN NAME>` with your Frontier Squid server's DNS entry or in the case of multiple
    Frontier Squid servers for a single resource, the round-robin DNS entry.

    See the [BNL_ATLAS_Frontier_Squid](https://github.com/opensciencegrid/topology/blob/7b27d5878a24b9812a2ec73bb97852fa9098eb2e/topology/Brookhaven%20National%20Laboratory/BNL%20ATLAS%20Tier1/BNL-ATLAS.yaml#L271-L291) 
    for a complete example.

2.  Normally registered squids will be monitored by WLCG.  This is
strongly recommended even for non-WLCG sites so operations experts can
help with diagnosing problems.  However, if a site declines
monitoring, that can be indicated by setting `Monitored: false` in a
`Details:` section below `Description:`.  Registration is still
important for the sake of excluding squids from worker node failover
monitors.  The default if `Details:` `Monitored:` is not set is
`true`.

3. If you set Monitored to true, also enable monitoring as described in 
the [upstream documentation on enabling monitoring](https://twiki.cern.ch/twiki/bin/view/Frontier/InstallSquid#Enabling_monitoring).


A few hours after a squid is registered and marked `Active` (and not
marked `Monitored: false`), 
[verify that it is monitored by WLCG](https://twiki.cern.ch/twiki/bin/view/LCG/WLCGSquidRegistration#Verify_monitor).

## Reference

### Users

The frontier-squid installation will create one user account unless it
already exists.

| User    | Comment                                                                                                                                      |
|:--------|:---------------------------------------------------------------------------------------------------------------------------------------------|
| `squid` | Reduced privilege user that the squid process runs under. Set the default gid of the "squid" user to be a group that is also called "squid". |

The package can instead use another user name of your choice if you
create a configuration file before installation. Details are in the
[upstream documentation Preparation section](https://twiki.cern.ch/twiki/bin/view/Frontier/InstallSquid#Preparation).

### Networking

Open the following ports on your Frontier Squid hosts:

| Port Number | Protocol | WAN | LAN | Comment                                                                             |
|-------------|----------|-----|-----|-------------------------------------------------------------------------------------|
| 3128        | tcp      |     | ✓   | Also limited in squid ACLs. Should be limited to access from your worker nodes      |
| 3401        | udp      | ✓   |     | Also limited in squid ACLs. Should be limited to public monitoring server addresses |

The addresses of the WLCG monitoring servers for use in firewalls are
listed in the
[upstream documentation Enabling monitoring section](https://twiki.cern.ch/twiki/bin/view/Frontier/InstallSquid#Enabling_monitoring).

### Frontier Squid Log Files

Log file contents are explained in the
[upstream documentation Log file contents section](https://twiki.cern.ch/twiki/bin/view/Frontier/InstallSquid#Log_file_contents).
