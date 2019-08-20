---
layout: post
title:  "Installing ELK on a FreeNAS Jail"
date:   2019-08-18 16:05:17 -0400
categories: freebsd freenas jail elk
permalink: /2019/08/installing-elk-on-freenas-jail/
---


I wanted to keep track of how my home web server was being accessed, so I decided to try to install ElasticSearch, Logstash, and Kibana, the so-called ELK stack, on a FreeBSD jail running on my FreeNAS server.  While these technologies can scale up to clusters with hundreds of servers, I just wanted an instance of the stack to run on one FreeBSD (`iocage`) Jail instance.  That way it is self-contained, and I can blow it away without impact my system if I end up not liking it.

This blog post includes the steps I used to install and configure these services, along with some workarounds I'd found on the internet.

# Environment

A few important points about my environment:

* I am running `FreeNAS-11.2-U3`.  YMMV on older or later versions.
* My FreeNAS is running on bare metal with 32GB of RAM.  ELK tends to be memory hungry, as it will take two JVMs and at least one Node instance.  A fine-tuned C exectuable, it is not.
* The NAS is running on my home network, with a router that supports DHCP and DNS.  Noting too out of the ordinary there.

Your environment might be different.

# Instructions

The following instructions should install an ELK stack on your FreeNAS.

## Create a Jail

You can create a jail either via the Web GUI, or via the command line.  Instructions follow.

### Web GUI

Use the FreeNAS GUI to create a Jail, based off FreeBSD `11.2-Release`:

![Create Jail](/assets/images/elk-create-jail-1.png)

I am using a virtual network interface with DHCP:
 
![Create Jail](/assets/images/elk-create-jail-2.png)

Once created, go back and edit the properties for this jail.

In order to run logstash, we need access to the `/proc` filesystem.  This will  [require](https://ixsystems.com/documentation/freenas/11.2/jails.html#advanced-jail-creation) that the `allow_mount_procfs` jail property be set, which in turn requires that the `allow_mount` jail property be set, and that the `enforce_statfs` jail property is set to something less than 2.  We set this property to 1.

![Create Jail](/assets/images/elk-create-jail-3.png)

Save the configuration and start the jail.

### Command Line

If you are more comfortable with the command line, you can instead create a jail on one go:

    root@freenas[~]# iocage create -n "elk" -r 11.2-RELEASE vnet="on" bpf="yes" dhcp="on" allow_raw_sockets="1" boot="on" interfaces="vnet0:bridge0" resolver="search dushin.home;domain dushin.home;nameserver 192.168.1.1" allow_mount="1" allow_mount_procfs="1" enforce_statfs="1"


## Attach to Jail

You should now be able to access the console via the `console` `iocage` subcommand:

    root@freenas[~]# iocage console elk
    FreeBSD 11.2-STABLE (FreeNAS.amd64) #0 r325575+9a3c7d8b53f(HEAD): Wed Mar 27 12:41:58 EDT 2019
    
    Welcome to FreeBSD!
    
    Release Notes, Errata: https://www.FreeBSD.org/releases/
    Security Advisories:   https://www.FreeBSD.org/security/
    FreeBSD Handbook:      https://www.FreeBSD.org/handbook/
    FreeBSD FAQ:           https://www.FreeBSD.org/faq/
    Questions List: https://lists.FreeBSD.org/mailman/listinfo/freebsd-questions/
    FreeBSD Forums:        https://forums.FreeBSD.org/
    
    Documents installed with the system are in the /usr/local/share/doc/freebsd/
    directory, or can be installed later with:  pkg install en-freebsd-doc
    For other languages, replace "en" with a language code like de or fr.
    
    Show the version of FreeBSD installed:  freebsd-version ; uname -a
    Please include that output and any error messages when posting questions.
    Introduction to manual pages:  man man
    FreeBSD directory layout:      man hier
    
    Edit /etc/motd to change this login announcement.
    root@elk:~ # 

## Mount `/proc`

The JDK that is used by elasticsearch and logstash requires access to the proc file system.

You will want to add the following line to `/etc/fstab`:

    proc   /proc    procfs   rw  0  0

and then mount `/proc`:

    root@elk:~ # mount /proc

You should then be able to see directories for processes available in the `/proc` directory:

    root@elk:~ # ls -l /proc
    total 0
    dr-xr-xr-x  1 root   wheel  0 Aug 17 15:28 74278
    dr-xr-xr-x  1 _dhcp  _dhcp  0 Aug 17 15:28 74351
    dr-xr-xr-x  1 root   wheel  0 Aug 17 15:28 74563
    dr-xr-xr-x  1 root   wheel  0 Aug 17 15:28 74619
    dr-xr-xr-x  1 root   wheel  0 Aug 17 15:28 74703
    dr-xr-xr-x  1 root   wheel  0 Aug 17 15:28 74704
    dr-xr-xr-x  1 root   wheel  0 Aug 17 15:28 76112
    lr--r--r--  1 root   wheel  0 Aug 17 15:28 curproc -> 76112

## Install and update the `pkg` manager

We will use the `pkg` command to install ElasticSearch, Logstash, and Kibana.  We first need to make it available and updated:

    root@elk:~ # pkg update
    The package management tool is not yet installed on your system.
    Do you want to fetch and install it now? [y/N]: y
    Bootstrapping pkg from pkg+http://pkg.FreeBSD.org/FreeBSD:11:amd64/quarterly, please wait...
    Verifying signature with trusted certificate pkg.freebsd.org.2013102301... done
    [elk] Installing pkg-1.11.1...
    [elk] Extracting pkg-1.11.1: 100%
    Updating FreeBSD repository catalogue...
    [elk] Fetching meta.txz: 100%    944 B   0.9kB/s    00:01    
    [elk] Fetching packagesite.txz: 100%    6 MiB   6.6MB/s    00:01    
    Processing entries: 100%
    FreeBSD repository update completed. 31932 packages processed.
    All repositories are up to date.


## Install and Start Elasticsearch

Elasticsearch is the (Java) service that hosts all of the Lucene injestion, indexing, and search capability.

We will install the latest and greatest (at the time of writing), version 6.  This will drag down half of the internet:

    root@elk:~ # pkg install elasticsearch6
    Updating FreeBSD repository catalogue...
    FreeBSD repository is up to date.
    All repositories are up to date.
    Updating database digests format: 100%
    The following 32 package(s) will be affected (of 0 checked):
    
    New packages to be INSTALLED:
            elasticsearch6: 6.5.4
            bash: 5.0.7
            indexinfo: 0.3.1
            gettext-runtime: 0.20.1
            openjdk8: 8.212.4.1
            libXtst: 1.2.3_2
            libXi: 1.7.10,1
            libXfixes: 5.0.3_2
            libX11: 1.6.8,1
            libxcb: 1.13.1
            libXdmcp: 1.1.3
            xorgproto: 2019.1
            libXau: 1.0.9
            libxml2: 2.9.9
            libpthread-stubs: 0.4
            libXext: 1.3.4,1
            libXrender: 0.9.10_2
            libXt: 1.2.0,1
            libSM: 1.2.3,1
            libICE: 1.0.9_3,1
            fontconfig: 2.12.6,1
            expat: 2.2.6_1
            freetype2: 2.10.0
            dejavu: 2.37_1
            mkfontscale: 1.2.1
            libfontenc: 1.1.4
            javavmwrapper: 2.6
            java-zoneinfo: 2019.a
            giflib: 5.1.9
            libinotify: 20180201_1
            alsa-lib: 1.1.2_2
            jna: 4.5.1
            
    Number of packages to be installed: 32
    
    The process will require 428 MiB more space.
    173 MiB to be downloaded.
    
    Proceed with this action? [y/N]: y
    [elk] [1/32] Fetching elasticsearch6-6.5.4.txz: 100%   75 MiB  13.2MB/s    00:06    
    [elk] [2/32] Fetching bash-5.0.7.txz: 100%    2 MiB   1.6MB/s    00:01    
    [elk] [3/32] Fetching indexinfo-0.3.1.txz: 100%    6 KiB   5.8kB/s    00:01    
    [elk] [4/32] Fetching gettext-runtime-0.20.1.txz: 100%  151 KiB 154.5kB/s    00:01    
    [elk] [5/32] Fetching openjdk8-8.212.4.1.txz: 100%   80 MiB  11.9MB/s    00:07    
    [elk] [6/32] Fetching libXtst-1.2.3_2.txz: 100%   20 KiB  20.0kB/s    00:01    
    [elk] [7/32] Fetching libXi-1.7.10,1.txz: 100%  121 KiB 123.4kB/s    00:01    
    [elk] [8/32] Fetching libXfixes-5.0.3_2.txz: 100%   15 KiB  15.1kB/s    00:01    
    [elk] [9/32] Fetching libX11-1.6.8,1.txz: 100%    2 MiB   1.7MB/s    00:01    
    [elk] [10/32] Fetching libxcb-1.13.1.txz: 100%    1 MiB   1.0MB/s    00:01    
    [elk] [11/32] Fetching libXdmcp-1.1.3.txz: 100%   14 KiB  14.1kB/s    00:01    
    [elk] [12/32] Fetching xorgproto-2019.1.txz: 100%  250 KiB 255.9kB/s    00:01    
    [elk] [13/32] Fetching libXau-1.0.9.txz: 100%   11 KiB  11.4kB/s    00:01    
    [elk] [14/32] Fetching libxml2-2.9.9.txz: 100%  819 KiB 839.1kB/s    00:01    
    [elk] [15/32] Fetching libpthread-stubs-0.4.txz: 100%    2 KiB   2.0kB/s    00:01    
    [elk] [16/32] Fetching libXext-1.3.4,1.txz: 100%   94 KiB  96.4kB/s    00:01    
    [elk] [17/32] Fetching libXrender-0.9.10_2.txz: 100%   29 KiB  29.5kB/s    00:01    
    [elk] [18/32] Fetching libXt-1.2.0,1.txz: 100%  445 KiB 456.0kB/s    00:01    
    [elk] [19/32] Fetching libSM-1.2.3,1.txz: 100%   23 KiB  23.3kB/s    00:01    
    [elk] [20/32] Fetching libICE-1.0.9_3,1.txz: 100%   92 KiB  93.8kB/s    00:01    
    [elk] [21/32] Fetching fontconfig-2.12.6,1.txz: 100%  357 KiB 365.8kB/s    00:01    
    [elk] [22/32] Fetching expat-2.2.6_1.txz: 100%  119 KiB 122.3kB/s    00:01    
    [elk] [23/32] Fetching freetype2-2.10.0.txz: 100%    1 MiB   1.4MB/s    00:01    
    [elk] [24/32] Fetching dejavu-2.37_1.txz: 100%    2 MiB   2.5MB/s    00:01    
    [elk] [25/32] Fetching mkfontscale-1.2.1.txz: 100%   21 KiB  21.1kB/s    00:01    
    [elk] [26/32] Fetching libfontenc-1.1.4.txz: 100%   19 KiB  19.7kB/s    00:01    
    [elk] [27/32] Fetching javavmwrapper-2.6.txz: 100%   17 KiB  17.0kB/s    00:01    
    [elk] [28/32] Fetching java-zoneinfo-2019.a.txz: 100%   72 KiB  73.2kB/s    00:01    
    [elk] [29/32] Fetching giflib-5.1.9.txz: 100%  200 KiB 205.1kB/s    00:01    
    [elk] [30/32] Fetching libinotify-20180201_1.txz: 100%   26 KiB  26.4kB/s    00:01    
    [elk] [31/32] Fetching alsa-lib-1.1.2_2.txz: 100%  429 KiB 439.7kB/s    00:01    
    [elk] [32/32] Fetching jna-4.5.1.txz: 100%    7 MiB   7.6MB/s    00:01    
    Checking integrity... done (0 conflicting)
    [elk] [1/32] Installing xorgproto-2019.1...
    [elk] [1/32] Extracting xorgproto-2019.1: 100%
    [elk] [2/32] Installing libXdmcp-1.1.3...
    [elk] [2/32] Extracting libXdmcp-1.1.3: 100%
    [elk] [3/32] Installing libXau-1.0.9...
    [elk] [3/32] Extracting libXau-1.0.9: 100%
    [elk] [4/32] Installing libxml2-2.9.9...
    [elk] [4/32] Extracting libxml2-2.9.9: 100%
    [elk] [5/32] Installing libpthread-stubs-0.4...
    [elk] [5/32] Extracting libpthread-stubs-0.4: 100%
    [elk] [6/32] Installing libxcb-1.13.1...
    [elk] [6/32] Extracting libxcb-1.13.1: 100%
    [elk] [7/32] Installing libX11-1.6.8,1...
    [elk] [7/32] Extracting libX11-1.6.8,1: 100%
    [elk] [8/32] Installing libXfixes-5.0.3_2...
    [elk] [8/32] Extracting libXfixes-5.0.3_2: 100%
    [elk] [9/32] Installing libXext-1.3.4,1...
    [elk] [9/32] Extracting libXext-1.3.4,1: 100%
    [elk] [10/32] Installing libICE-1.0.9_3,1...
    [elk] [10/32] Extracting libICE-1.0.9_3,1: 100%
    [elk] [11/32] Installing expat-2.2.6_1...
    [elk] [11/32] Extracting expat-2.2.6_1: 100%
    [elk] [12/32] Installing freetype2-2.10.0...
    [elk] [12/32] Extracting freetype2-2.10.0: 100%
    [elk] [13/32] Installing libfontenc-1.1.4...
    [elk] [13/32] Extracting libfontenc-1.1.4: 100%
    [elk] [14/32] Installing libXi-1.7.10,1...
    [elk] [14/32] Extracting libXi-1.7.10,1: 100%
    [elk] [15/32] Installing libSM-1.2.3,1...
    [elk] [15/32] Extracting libSM-1.2.3,1: 100%
    [elk] [16/32] Installing fontconfig-2.12.6,1...
    [elk] [16/32] Extracting fontconfig-2.12.6,1: 100%
    Running fc-cache to build fontconfig cache...
    /usr/local/share/fonts: skipping, no such directory
    /usr/local/lib/X11/fonts: skipping, no such directory
    /var/db/fontconfig: cleaning cache directory
    fc-cache: succeeded
    [elk] [17/32] Installing mkfontscale-1.2.1...
    [elk] [17/32] Extracting mkfontscale-1.2.1: 100%
    [elk] [18/32] Installing indexinfo-0.3.1...
    [elk] [18/32] Extracting indexinfo-0.3.1: 100%
    [elk] [19/32] Installing libXtst-1.2.3_2...
    [elk] [19/32] Extracting libXtst-1.2.3_2: 100%
    [elk] [20/32] Installing libXrender-0.9.10_2...
    [elk] [20/32] Extracting libXrender-0.9.10_2: 100%
    [elk] [21/32] Installing libXt-1.2.0,1...
    [elk] [21/32] Extracting libXt-1.2.0,1: 100%
    [elk] [22/32] Installing dejavu-2.37_1...
    [elk] [22/32] Extracting dejavu-2.37_1: 100%
    [elk] [23/32] Installing javavmwrapper-2.6...
    [elk] [23/32] Extracting javavmwrapper-2.6: 100%
    [elk] [24/32] Installing java-zoneinfo-2019.a...
    [elk] [24/32] Extracting java-zoneinfo-2019.a: 100%
    [elk] [25/32] Installing giflib-5.1.9...
    [elk] [25/32] Extracting giflib-5.1.9: 100%
    [elk] [26/32] Installing libinotify-20180201_1...
    [elk] [26/32] Extracting libinotify-20180201_1: 100%
    [elk] [27/32] Installing alsa-lib-1.1.2_2...
    [elk] [27/32] Extracting alsa-lib-1.1.2_2: 100%
    [elk] [28/32] Installing gettext-runtime-0.20.1...
    [elk] [28/32] Extracting gettext-runtime-0.20.1: 100%
    [elk] [29/32] Installing openjdk8-8.212.4.1...
    [elk] [29/32] Extracting openjdk8-8.212.4.1: 100%
    [elk] [30/32] Installing bash-5.0.7...
    [elk] [30/32] Extracting bash-5.0.7: 100%
    [elk] [31/32] Installing jna-4.5.1...
    [elk] [31/32] Extracting jna-4.5.1: 100%
    [elk] [32/32] Installing elasticsearch6-6.5.4...
    ===> Creating groups.
    Creating group 'elasticsearch' with gid '965'.
    ===> Creating users
    Creating user 'elasticsearch' with uid '965'.
    [elk] [32/32] Extracting elasticsearch6-6.5.4: 100%
    
    adapt it to your taste (or use the new "FREETYPE_PROPERTIES" environment
    variable).
    
    The environment variable "FREETYPE_PROPERTIES" can be used to control the
    driver properties. Example:
    
    FREETYPE_PROPERTIES=truetype:interpreter-version=35 \
            cff:no-stem-darkening=1 \
            autofitter:warping=1
            
    This allows to select, say, the subpixel hinting mode at runtime for a given
    application.
    
    The controllable properties are listed in the section "Controlling FreeType
    Modules" in the reference's table of contents
    (/usr/local/share/doc/freetype2/reference/site/index.html, if documentation was installed).
    Message from dejavu-2.37_1:
    
    Make sure that the freetype module is loaded.  If it is not, add the following
    line to the "Modules" section of your X Windows configuration file:
    
            Load "freetype"
            
    Add the following line to the "Files" section of X Windows configuration file:
    
            FontPath "/usr/local/share/fonts/dejavu/"
            
    Note: your X Windows configuration file is typically /etc/X11/XF86Config
    if you are using XFree86, and /etc/X11/xorg.conf if you are using X.Org.
    Message from libinotify-20180201_1:
    
    ============================================================================
    
    Libinotify functionality on FreeBSD is missing support for
    
      - detecting a file being moved into or out of a directory within the
        same filesystem
      - certain modifications to a symbolic link (rather than the
        file it points to.)
        
    in addition to the known limitations on all platforms using kqueue(2)
    where various open and close notifications are unimplemented.
    
    This means the following regression tests will fail:
    
    Directory notifications:
       IN_MOVED_FROM
       IN_MOVED_TO
       
    Open/
    Kernel patches to address the missing directory and symbolic link
    notifications are available from:
    
    https://github.com/libinotify-kqueue/libinotify-kqueue/tree/master/patches
    
    =============================================================================
    You might want to consider increasing the kern.maxfiles tunable if you plan
    to use this library for applications that need to monitor activity of a lot
    of files.
    =============================================================================
    Message from openjdk8-8.212.4.1:
    
    ======================================================================
    
    This OpenJDK implementation requires fdescfs(5) mounted on /dev/fd and
    procfs(5) mounted on /proc.
    
    If you have not done it yet, please do the following:
    
            mount -t fdescfs fdesc /dev/fd
            mount -t procfs proc /proc
            
    To make it permanent, you need the following lines in /etc/fstab:
    
            fdesc   /dev/fd         fdescfs         rw      0       0
            proc    /proc           procfs          rw      0       0
            
    ======================================================================
    Message from jna-4.5.1:
    
    ===>   NOTICE:
    
    The jna port currently does not have a maintainer. As a result, it is
    more likely to have unresolved issues, not be up-to-date, or even be removed in
    the future. To volunteer to maintain this port, please create an issue at:
    
    https://bugs.freebsd.org/bugzilla
    
    More information about port maintainership is available at:
    
    https://www.freebsd.org/doc/en/articles/contributing/ports-contributing.html#maintain-port
    Message from elasticsearch6-6.5.4:
    
    ======================================================================
    
    Please see /usr/local/etc/elasticsearch for sample versions of
    elasticsearch.yml and logging.yml.
    
    ElasticSearch requires memory locking of large amounts of RAM.
    You may need to set:
    
    sysctl security.bsd.unprivileged_mlock=1
    
    !!! PLUGINS NOTICE !!!
    
    ElasticSearch plugins should only be installed via the elasticsearch-plugin
    included with this software. As we strive to provide a minimum semblance
    of security, the files installed by the package are owned by root:wheel.
    This is different than upstream hich expects all of the files to be
    owned by the user and for you to execute the elasticsearch-plugin script
    as said user.
    
    You will encounter permissions errors with configuration files and
    directories created by plugins which you will have to manually correct.
    This is the price we have to pay to protect ourselves in the face of
    a poorly designed security model.
    
    e.g., after installing X-Pack you will have to correct:
    
    /usr/local/etc/elasticsearch/elasticsearch.keystore file to be owned by elasticsearch:elasticsearch
    /usr/local/etc/elasticsearch/x-pack directory/files to be owned by elasticsearch:elasticsearch
    
    !!! PLUGINS NOTICE !!!
    
    ======================================================================

Add the following to your jail's `/etc/rc.conf`:

    elasticsearch_enable="YES"

and start the `elasticsearch` service:

    root@elk:~ # service elasticsearch start
    Starting elasticsearch.

You should be able to see `elasticsearch` in your process list (`ps -aux`):

    USER            PID  %CPU %MEM     VSZ     RSS TT  STAT STARTED    TIME COMMAND
    elasticsearch 80301 402.5  4.3 2814972 1452640 15  IJ   15:40   0:41.89 /usr/local/openjdk8/bin/java -Xms1g -Xmx1g -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccu

Wow.  Saturating 4 cores.  I hope that is just the cost of starting up for the first time...

Configuration files are in `/usr/local/etc/elasticsearch`:

    root@elk:~ # ls -l /usr/local/etc/elasticsearch
    total 32
    -rw-rw----  1 elasticsearch  elasticsearch    207 Aug  17 15:40 elasticsearch.keystore
    -rwxr-xr-x  1 root           wheel           2978 Aug  17 02:58 elasticsearch.yml
    -rwxr-xr-x  1 root           wheel           2978 Aug  17 02:58 elasticsearch.yml.sample
    -rwxr-xr-x  1 root           wheel           3202 Aug  17 02:58 jvm.options
    -rwxr-xr-x  1 root           wheel           3202 Aug  17 02:58 jvm.options.sample
    -rwxr-xr-x  1 root           wheel          12423 Aug  17 02:58 log4j2.properties
    -rwxr-xr-x  1 root           wheel          12423 Aug  17 02:58 log4j2.properties.sample

Elasticsearch should be listening on ports 9200 and 9300:

    root@elk:/usr/local/etc/elasticsearch # netstat -an -p tcp
    Active Internet connections (including servers)
    Proto Recv-Q Send-Q Local Address          Foreign Address        (state)
    tcp4       0      0 127.0.0.1.9200         *.*                    LISTEN
    tcp6       0      0 ::1.9200               *.*                    LISTEN
    tcp4       0      0 127.0.0.1.9300         *.*                    LISTEN
    tcp6       0      0 ::1.9300               *.*                    LISTEN

Log files are in `/var/log/elasticsearch`

    root@elk:~ # ls -l /var/log/elasticsearch
    total 7
    -rw-r--r--  1 elasticsearch  elasticsearch  7583 Aug 17 15:41 elasticsearch.log
    -rw-r--r--  1 elasticsearch  elasticsearch     0 Aug 17 15:40 elasticsearch_access.log
    -rw-r--r--  1 elasticsearch  elasticsearch     0 Aug 17 15:40 elasticsearch_audit.log
    -rw-r--r--  1 elasticsearch  elasticsearch     0 Aug 17 15:40 elasticsearch_deprecation.log
    -rw-r--r--  1 elasticsearch  elasticsearch     0 Aug 17 15:40 elasticsearch_index_indexing_slowlog.log
    -rw-r--r--  1 elasticsearch  elasticsearch     0 Aug 17 15:40 elasticsearch_index_search_slowlog.log

Okay, that is Elasticsearch.  For the most part this is just a "back end" service, and nothing needs to be done to it, yet.  Let's move on.

# Install and run Logstash

Logstash is the ingestion engine for Elasticsearch.  It will be the thing to which we will send logs, which in turn will get indexed and made searchable.

We will also install version 6 of this package:

    root@elk:~ # pkg install logstash6
    Updating FreeBSD repository catalogue...
    FreeBSD repository is up to date.
    All repositories are up to date.
    The following 1 package(s) will be affected (of 0 checked):
    
    New packages to be INSTALLED:
            logstash6: 6.4.2
            
    Number of packages to be installed: 1
    
    The process will require 247 MiB more space.
    123 MiB to be downloaded.
    
    Proceed with this action? [y/N]: y
    [elk] [1/1] Fetching logstash6-6.4.2.txz: 100%  123 MiB  12.9MB/s    00:10    
    Checking integrity... done (0 conflicting)
    [elk] [1/1] Installing logstash6-6.4.2...
    ===> Creating groups.
    Creating group 'logstash' with gid '893'.
    ===> Creating users
    Creating user 'logstash' with uid '893'.
    [elk] [1/1] Extracting logstash6-6.4.2: 100%
    Message from logstash6-6.4.2:
    
    To start logstash as an agent during startup, add
    
        logstash_enable="YES"
        
    to your /etc/rc.conf.
    
    Extra options can be found in startup script.

Add the following to your jail's `/etc/rc.conf`:

    logstash_enable="YES"

and start the `logstash` service:

    root@elk:~ # service logstash start
    Starting logstash.

You should be able to see `logstash` in your process list (`ps -aux`):

    USER            PID  %CPU %MEM     VSZ     RSS TT  STAT STARTED    TIME COMMAND
    logstash      86485 226.0  2.6 2817416  861772  -  IJ   16:02   1:50.87 /usr/local/openjdk8/bin/java -Xms1g -Xmx1g -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+Use

Logstash configuration files are in `/usr/local/etc/logstash`:

    root@elk:~ # ls -l /usr/local/etc/logstash
    total 53
    -rw-r--r--  1 root  wheel  1846 Aug  17 08:37 jvm.options
    -rw-r--r--  1 root  wheel  1846 Aug  17 08:37 jvm.options.sample
    -rw-r--r--  1 root  wheel  4466 Aug  17 08:37 log4j2.properties
    -rw-r--r--  1 root  wheel  4466 Aug  17 08:37 log4j2.properties.sample
    -rw-r--r--  1 root  wheel  1098 Aug  17 08:37 logstash.conf
    -rw-r--r--  1 root  wheel  1098 Aug  17 08:37 logstash.conf.sample
    -rw-r--r--  1 root  wheel  8200 Aug  17 08:37 logstash.yml
    -rw-r--r--  1 root  wheel  8200 Aug  17 08:37 logstash.yml.sample
    -rw-r--r--  1 root  wheel  3244 Aug  17 08:37 pipelines.yml
    -rw-r--r--  1 root  wheel  3244 Aug  17 08:37 pipelines.yml.sample

Logstash should be listening on a port between 9600 and 9700 on the loopback address:

    root@elk:~ # netstat -an -p tcp
    Active Internet connections (including servers)
    Proto Recv-Q Send-Q Local Address          Foreign Address        (state)
    tcp4       0      0 127.0.0.1.9600         *.*                    LISTEN
    tcp4       0      0 127.0.0.1.9200         *.*                    LISTEN
    tcp6       0      0 ::1.9200               *.*                    LISTEN
    tcp4       0      0 127.0.0.1.9300         *.*                    LISTEN
    tcp6       0      0 ::1.9300               *.*                    LISTEN

Log files are in `/var/log/logstash`

    root@elk:~ # ls -l /var/log/logstash
    total 5
    -rw-r--r--  1 logstash  logstash  1777 Aug 17 16:02 logstash-plain.log
    -rw-r--r--  1 logstash  logstash     0 Aug 17 16:02 logstash-slowlog-plain.log


## Install and run Kibana

Kibana is the Web GUI in front of Elasticsearch.

We will also install version 6 of this package:

    root@elk:~ # pkg install kibana6
    Updating FreeBSD repository catalogue...
    FreeBSD repository is up to date.
    All repositories are up to date.
    The following 7 package(s) will be affected (of 0 checked):
    
    New packages to be INSTALLED:
            kibana6: 6.5.4_2
            node8: 8.16.0
            libnghttp2: 1.39.2
            ca_root_nss: 3.45
            c-ares: 1.15.0
            libuv: 1.30.1
            icu: 64.2,1
            
    Number of packages to be installed: 7
    
    The process will require 494 MiB more space.
    154 MiB to be downloaded.
    
    Proceed with this action? [y/N]: y
    [elk] [1/7] Fetching kibana6-6.5.4_2.txz: 100%  139 MiB  13.3MB/s    00:11    
    [elk] [2/7] Fetching node8-8.16.0.txz: 100%    4 MiB   4.6MB/s    00:01    
    [elk] [3/7] Fetching libnghttp2-1.39.2.txz: 100%  114 KiB 117.1kB/s    00:01    
    [elk] [4/7] Fetching ca_root_nss-3.45.txz: 100%  295 KiB 301.6kB/s    00:01    
    [elk] [5/7] Fetching c-ares-1.15.0.txz: 100%  127 KiB 129.7kB/s    00:01    
    [elk] [6/7] Fetching libuv-1.30.1.txz: 100%  110 KiB 112.2kB/s    00:01    
    [elk] [7/7] Fetching icu-64.2,1.txz: 100%   10 MiB  10.4MB/s    00:01    
    Checking integrity... done (0 conflicting)
    [elk] [1/7] Installing libnghttp2-1.39.2...
    [elk] [1/7] Extracting libnghttp2-1.39.2: 100%
    [elk] [2/7] Installing ca_root_nss-3.45...
    [elk] [2/7] Extracting ca_root_nss-3.45: 100%
    [elk] [3/7] Installing c-ares-1.15.0...
    [elk] [3/7] Extracting c-ares-1.15.0: 100%
    [elk] [4/7] Installing libuv-1.30.1...
    [elk] [4/7] Extracting libuv-1.30.1: 100%
    [elk] [5/7] Installing icu-64.2,1...
    [elk] [5/7] Extracting icu-64.2,1: 100%
    [elk] [6/7] Installing node8-8.16.0...
    [elk] [6/7] Extracting node8-8.16.0: 100%
    [elk] [7/7] Installing kibana6-6.5.4_2...
    [elk] [7/7] Extracting kibana6-6.5.4_2: 100%
    Message from ca_root_nss-3.45:
    
    ********************************* WARNING *********************************
    
    FreeBSD does not, and can not warrant that the certification authorities
    whose certificates are included in this package have in any way been
    audited for trustworthiness or RFC 3647 compliance.
    
    Assessment and verification of trust is the complete responsibility of the
    system administrator.
    
    *********************************** NOTE **********************************
    
    This package installs symlinks to support root certificates discovery by
    default for software that uses OpenSSL.
    
    This enables SSL Certificate Verification by client software without manual
    intervention.
    
    If you prefer to do this manually, replace the following symlinks with
    either an empty file or your site-local certificate bundle.
    
      * /etc/ssl/cert.pem
      * /usr/local/etc/ssl/cert.pem
      * /usr/local/openssl/cert.pem
      
    ***************************************************************************
    Message from node8-8.16.0:
    
    Note: If you need npm (Node Package Manager), please install www/npm.
    

Kibaba configuration files are in `/usr/local/etc/kibana`:

    root@elk:~ # ls -l /usr/local/etc/kibana
    total 9
    -rw-r--r--  1 root  wheel  5054 Aug 17 07:03 kibana.yml
    -rw-r--r--  1 root  wheel  5054 Aug 17 07:03 kibana.yml.sample

Due to a bug in the FreeBSD Kibana package, we need to edit the `/usr/local/etc/kibana/kibana.yml` file and [disable the reporting module](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=231179) in Kibana.  This will limit functionality, but it will allow Kibana to start up properly.

Add the following configuration to this file:

    xpack.reporting.enabled: false

While we are at it, we want the Kibana Web UI to start up on a network interface other than localhost, so that we can access the UI from our home network.  We do this via the `server.host` configuration setting, e.g.,

    server.host: "elk.dushin.home"

Add the following to your jail's `/etc/rc.conf`:

    kibana_enable="YES"

and start the `kibana` service:

    root@elk:~ # service kibana start
    Starting kibana.

You should be able to see the `kibana` process in your process list (`ps -aux`):

    USER            PID %CPU %MEM     VSZ     RSS TT  STAT STARTED    TIME COMMAND
    www           99372 84.7  0.7  816560  247928  -  RJ   16:47   0:08.09 /usr/local/bin/node --no-warnings /usr/local/www/kibana6/src/cli serve --config /usr/local/etc/kibana/kibana.yml --log-file /var/lo

Kibana should be listening on port 5601 on the exposed IP address:

    root@elk:~ # netstat -an -p tcp                                                                                                                                                                           
    Active Internet connections (including servers)
    Proto Recv-Q Send-Q Local Address          Foreign Address        (state)
    tcp4       0      0 192.168.1.93.5601      *.*                    LISTEN
    tcp4       0      0 127.0.0.1.9600         *.*                    LISTEN
    tcp4       0      0 127.0.0.1.9200         *.*                    LISTEN
    tcp6       0      0 ::1.9200               *.*                    LISTEN
    tcp4       0      0 127.0.0.1.9300         *.*                    LISTEN
    tcp6       0      0 ::1.9300               *.*                    LISTEN


Logs are in `/var/log/kibana.log`:

    root@elk:~ # ls -l /var/log/kibana.log 
    -rw-r-----  1 www  www  319 Aug 17 16:48 /var/log/kibana.log
    

You should now be able to connect to port 5601 on your Jail, e.g., [http://elk:5601](http://elk:5601):

> Note.  It may take some time for Kibana to start the first time, and you may get the message "Kibana server is not ready yet" in your browser until it is ready.

![Kibana](/assets/images/elk-kibana.png)

# Summary

In this post, we have created a FreeBSD Jail on a FreeNAS server suitable for use with the ELK stack.  We have installed the ELK component services, and made the web front end avaialble to users.

In the next blog post, we will configure our logstash service to take log entries from an `nginx` server running on a separate (you guessed it) Jail instance, so that we can make sense of who is accessing what.

---
<blockquote>Copyright (c) 2019 dushin.net
This work is licensed under a <a href="http://creativecommons.org/licenses/by/4.0/" rel="license">Creative Commons Attribution 4.0 International License</a>.
<a href="http://creativecommons.org/licenses/by/4.0/" rel="license"><img style="border-width: 0;" src="https://i.creativecommons.org/l/by/4.0/88x31.png" alt="Creative Commons License" /></a></blockquote>
<strong>Comments</strong>

Because of the prevalence of automated spam on the internet, I turn off comments on my blog posts. If you have comments or questions about this, please send me email using <em>first-name</em> -at- <em>last-name</em> -dot- net, where you can find my first and last name in the [About](/about) page of this blog. 
