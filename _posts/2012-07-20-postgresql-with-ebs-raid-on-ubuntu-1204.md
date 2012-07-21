---
layout: post
title: "PostgreSQL 9.1 with EBS RAID on Ubuntu 12.04"
description: ""
category: tutorial
tags: [beginner, sysadmin, postgresql, pgbouncer, RAID, EC2, EBS, ubuntu]
---
{% include JB/setup %}

**Difficulty**: Beginner

*****

This tutorial is for setting up Postgresql 9.1 on EC2 using RAIDed EBS
drives on the default Ubuntu 12.04 64-bit AMI. This guide is aimed at
developers who are very skilled at writing code but have very little sysadmin
experience.

The goal of this guide is to use "best practices" while straying
minimally from the defaults. Doing things like compiling packages from
source instead of using ``apt-get`` or heavily modifying config files
will only make systems harder to automate and harder to upgrade in the
future

Finally, this guide will provide rationale for the commands being used
and plenty of links and resources to better understand what's actually
going on. The overall reference for this architecture was inspired by
[this End Point blog post](http://blog.endpoint.com/2010/02/postgresql-ec2-ebs-raid0-snapshot.html).

Knowledge of basics such as [ssh-ing into the
instance](http://serverfault.com/questions/227804/why-cant-i-ssh-into-my-new-ec2-instance)
and [assigning AWS security groups](http://docs.amazonwebservices.com/AWSEC2/latest/UserGuide/using-network-security.html) to permit PostgreSQL and PgBouncer ports
5432 and 6432 respectively is assumed.

## RAID your EBS drives

The biggest problem with Amazon's EBS drives is unpredictable
performance. On good days, latencies are < 10 ms, but often times a
drive can enter a sluggish state where latencies are > 1 s, clearly
unsuitable for production databases. Software RAID thus serves to
smooth out performance variations on individual drives. The two
biggest camps on EBS RAID favor either
[RAID 0 or RAID 10](http://www.thegeekstuff.com/2010/08/raid-levels-tutorial/).

To clarify a common misconception, EBS drives guarantee that your data
is **safe**. Your data is duplicated and stored on multiple machines,
making complete data loss **highly unlikely**, apart from
complete and simultaneous failure on multiple drives. However, that
does not mean your database is always *available*. As
[proponents of RAID 10](http://blog.9minutesnooze.com/raid-10-ebs-data/)
would say, that would be kind of a problem for a website where
waiting for Amazon to reconstruct your data over the next few days
would mean losing significant revenue. RAID 10 also gives you the
option of hot-swapping single unperformant drives whereas RAID 0
requires reconstructing the array.

That being said, I have talked to an Amazon Solution Architect (yes
these guys exist), and they actually advise against RAID 10 and
suggest master-slave replication across multiple availability zones.
In the event of an EC2 outage (where an entire availability zone is
unavailable), RAID 10 will not help you.

RAID 0 on the other hand provides significant advantages in
performance and storage space, which is likely more important for the
target audience of this tutorial. For starting a small web application
with no ultra-critical data. The important criteria here are
performance and money. If your array fails, spin up a new instance
with yesterday's snapshot, and you can always scale/increase
availability later with master-slave replication.

For configuring RAID, this guide is mostly derived from
[this blog post by Nick Brunn](http://bruun.co/2012/06/06/software-raid-on-ec2-with-mdadm).

### Install Everything

From now on, everything should be run as root for the sake of
simplicity. Install all the necessary packages from aptitude.

{% highlight bash %}
$ apt-get install postgresql pgbouncer mdadm xfsprogs
{% endhighlight %}

### Creating the RAID Array

Starting off, you should learn the basics of [fstab](http://en.wikipedia.org/wiki/Fstab), [mount](http://linux.die.net/man/8/mount), [mdadm](http://en.wikipedia.org/wiki/Mdadm), [mkfs](http://manpages.ubuntu.com/manpages/precise/man8/mkfs.xfs.8.html).

Create 4 EBS instances and attach them to your EC2 instance as
``/dev/sdf``, ``/dev/sdg``, ``/dev/sdh``, and ``/dev/sdi``.

{% highlight bash %}
$ mdadm -C /dev/md0 -n 4 -l 0 -z max /dev/xvdf /dev/xvdg /dev/xvdh /dev/xvdi
$ mkfs.xfs /dev/md0
{% endhighlight %}

Edit ``/etc/fstab`` and append the following line:

    /dev/md0    /pgdata    xfs    noatime,nodiratime,nobootwait

Use the automatic ``mkconf`` program in mdadm so that mdadm can
reconstruct your RAID array on reboot.

{% highlight bash %}
$ mkdir /pgdata
$ mount -a
$ /usr/share/mdadm/mkconf > /etc/mdadm/mdadm.conf # (must run as root!)
{% endhighlight %}

Edit ``/etc/initramfs-tools/conf.d/mdadm`` so that your instance can still
boot with a failing RAID drive:

    BOOT_DEGRADED=true

## Configuring Postgresql 9.1

In Ubuntu, Postgres commands ``initdb`` and ``pg_ctl`` are not
symlinked onto ``$PATH``. This is done on purpose because Ubuntu has
its own tools for managing Postgres databases which will properly
setup the configuration and data directories for Ubuntu. This will
make it much easier to upgrade and manage in the future. For more
information, here is a more complete
[introduction to postgres](http://www.markus-gattol.name/ws/postgresql.html).
These tools all follow the pattern of ``pg_*cluster``, of which we
will use ``pg_createcluster`` and ``pg_dropcluster``.

These tools are necessary because we delete the default cluster
configuration and create a new cluster with a data directory on the
RAID array. We also want to set the Postgres data directory not
directly on ``/pgdata`` or else ``pg_dropcluster`` won't be able to
properly delete:

{% highlight bash %}
$ mkdir /pgdata/main
$ chown -R postgres.postgres /pgdata
$ chmod -R 700 /pgdata/main
$ service postgresql stop
$ pg_dropcluster 9.1 main
$ pg_createcluster 9.1 main -D /pgdata/main
{% endhighlight %}

Edit
[``/etc/postgresql/9.1/main/postgresql.conf``](http://www.postgresql.org/docs/8.2/static/runtime-config-connection.html)
to listen to incoming TCP connections. For
[basic server tuning](http://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server),
set shared\_buffers to 1/4 and effective\_cache_size to 1/2 of your total memory.
So if ``free -m`` reports ~1400 MB of total memory:

    listen_addresses = ‘*’
    shared_buffers = 350MB
    effective_cache_size = 700MB

To make your [kernel
happy](http://www.postgresql.org/docs/current/static/kernel-resources.html), run:

{% highlight bash %}
$ sysctl -w kernel.shmmax=786432000
{% endhighlight %}

Add the above line to ``/etc/sysctl.conf`` to preserve after reboot.

Edit
[``/etc/postgresql/9.1/main/pg_hba.conf``](http://www.postgresql.org/docs/9.1/static/auth-pg-hba-conf.html)
and add these lines to allow incoming TCP md5 password authentication
from any origin:

    host    all             all             0.0.0.0/0               md5
    host    all             all             ::/0                    md5

Now you can finally start the database:

{% highlight bash %}
$ service postgresql start
$ sudo -u postgres createuser myapp --pwprompt # (say no to everything)
$ sudo -u postgres createdb myapp --owner=myapp
{% endhighlight %}

## Configuring PgBouncer 1.4.2

PgBouncer is useful for enhancing Postgres performance even on small
installations because it can pool database connections.
[This simple thought experiment](http://stackoverflow.com/a/10420469/470668)
does a great job of explaining why this is true. In this example we'll
install it on the same instance, but you can easily imagine how this
will let you scale your database by handing off connections to lots of
backend database servers.

When pgbouncer is first installed, it is not registered with Ubuntu's
Upstart until it is configured.

Edit ``/etc/pgbouncer/pgbouncer.ini``, and set your ``pool_size`` to be
``((2 * core_count) + effective_spindle_count) = 6`` on a single
core machine:

{% highlight ini %}
[databases]
myapp = dbname=myapp host=127.0.0.1 pool_size=6

[pgbouncer]
listen_addr = *
auth_type = md5
{% endhighlight %}

Edit ``/etc/pgbouncer/userlist.txt`` and add your real password:

    “myapp”	"password”

Edit ``/etc/default/pgbouncer`` to register PgBouncer with Upstart:

    START=1
    OPTS="-d /etc/pgbouncer/pgbouncer.ini"

Now you can start PgBouncer:

{% highlight bash %}
$ service pgbouncer start
{% endhighlight %}

Test out the new database and user using
[``psql``](http://www.postgresql.org/docs/9.1/static/app-psql.html) to
verify everything is working properly. Use ``-p 5432`` to test the
underlying Postgresql database as well:

{% highlight bash %}
$ psql -U premonit -p 6432 -h <ip-address>  # (for remote access)
$ psql -U premonit -p 6432 -h localhost  # (for localhost access)
{% endhighlight %}

Feel free to comment below if you have any suggestions!
