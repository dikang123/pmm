.. _faq:

==========================
Frequently Asked Questions
==========================

.. contents::
   :local:
   :depth: 1

What are the minimum system requirements for PMM?
=================================================

:PMM Server: Any system which can run Docker version 1.10 or later,
 and kernel 3.x-4.x

:PMM Client: Any modern 64-bit Linux distribution.
 We recommend the latest versions of
 Debian, Ubuntu, CentOS, and RedHat Enterprise Linux.

How to control memory consumption for Prometheus?
=================================================

By default, Prometheus in PMM Server uses up to 256 MB of memory
for storing the most recently used data chunks.
Depending on the amount of data coming into Prometheus,
you may require a higher limit to avoid throttling data ingestion,
or allow less memory consumption if it is needed for other processes.

You can control the allowed memory consumption for Prometheus
by passing the ``METRICS_MEMORY`` environment variable
when :ref:`creating and running the PMM Server container <server-container>`.
To set the environment variable, use the ``-e`` option.
The value must be passed in kilobytes.
For example, to set the limit to 4 GB of memory::

 -e METRICS_MEMORY=4194304

.. note:: The limit affects only memory reserved for data chunks.
   Actual RAM usage by Prometheus is higher.
   It is recommended to have at least three times more memory
   than the expected memory taken up by data chunks.

.. _data-retention:

How to control data retention for Prometheus?
=============================================

By default, Prometheus in PMM Server stores time-series data for 30 days.
Depending on available disk space and your requirements,
you may need to adjust data retention time.

You can control data retention time for Prometheus
by passing the ``METRICS_RETENTION`` environment variable
when :ref:`creating and running the PMM Server container <server-container>`.
To set the environment variable, use the ``-e`` option.
The value is passed as a combination of hours, minutes, and seconds.
For example, the default value of 30 days is ``720h0m0s``.
You probably do not need to be more precise than the number hours,
so you can discard the minutes and seconds.
For example, to decrease the retention period to 8 days::

 -e METRICS_RETENTION=192h

.. _service-location:

Where are the services created by PMM Client?
=============================================

When you add a monitoring instance using the ``pmm-admin`` tool,
it creates a corresponding service.
The name of the service has the following syntax:
``pmm-<type>-exporter-<port>``

The location of the services depends on the service manager:

+-----------------+-----------------------------+
| Service manager | Service location            |
+=================+=============================+
| ``systemd``     | :file:`/etc/systemd/system/`|
+-----------------+-----------------------------+
| ``upstart``     | :file:`/etc/init/`          |
+-----------------+-----------------------------+
| ``systemv``     | :file:`/etc/init.d/`        |
+-----------------+-----------------------------+

To see which service manager is used on your system,
run ``sudo pmm-admin info``.

Where is DSN stored?
====================

Every service created by ``pmm-admin`` when you add a monitoring instance
gets a DSN from the credentials provided, auto-detected, or created
(when adding the instance with the ``--create-user`` option).

For MySQL and MongoDB metrics instances
(``mysql:metrics`` and ``mongodb:metrics`` services),
the DSN is stored with the corresponding service files.
For more information, see :ref:`service-location`.

For QAN instances (``mysql:queries`` service),
the DSN is stored in local configuration files
under :file:`/usr/local/percona/qan-agent`.

Also, a sanitized copy of DSN (without the passowrd)
is stored in Consul API for information purposes
(used by the ``pmm-admin list`` command).

Where are PMM Client log files located?
=======================================

Every service created by ``pmm-admin`` when you add a monitoring instance
has a separate log file located in :file:`/var/log/`.
The file names have the following syntax: ``pmm-<type>-exporter-<port>.log``

For example, the log file for the QAN monitoring service is
:file:`/var/log/pmm-queries-exporter-42001.log`.

You can view all available monitoring instance types and corresponding ports
using the ``pmm-admin list`` command.
For more information, see :ref:`pmm-admin-list`.

.. _performance-issues:

What are common performance considerations?
===========================================

If a MySQL server has a lot of schemas or tables,
it is recommended to disable per table metrics when adding the instance:

.. prompt:: bash

   sudo pmm-admin add mysql --disable-tablestats

.. note:: Table statistics are disabled automatically
   if there are over 10 000 tables.

For more information, run ``sudo pmm-admin add mysql --help``.

Can I stop all services at once?
================================

Yes, you can use ``pmm-admin`` to start and stop either individual services
that correspond to the added monitoring instances,
or all of them at once.

To stop all services:

.. prompt:: bash

   sudo pmm-admin stop --all

To start all services:

.. prompt:: bash

   sudo pmm-admin start --all

For more information about starting and stopping services,
see :ref:`pmm-admin-start`.

You can view all available monitoring instances
and the states of the corresponding services
using the ``pmm-admin list`` command.
For more information, see :ref:`pmm-admin-list`.

.. _privileges:

What privileges are required to monitor a MySQL instance?
=========================================================

When adding MySQL instance to monitoring,
you can specify the MySQL server superuser account credentials,
which has all privileges.
However, monitoring with the superuser account is not secure.
If you also specify the ``--create-user`` option,
it will create a user with only the necessary privileges for collecting data.

You can also set up the ``pmm`` user manually with necessary privileges
and pass its credentials when adding the instance.

To enable complete MySQL instance monitoring,
a command similar to the following is recommended:

.. prompt:: bash

   sudo pmm-admin add mysql --user root --password root --create-user

The superuser credentials are required only to set up the ``pmm`` user
with necessary privileges for collecting data.
If you want to create this user yourself,
the following privileges are required:

.. code-block:: sql

   GRANT SELECT, PROCESS, SUPER, REPLICATION CLIENT ON *.* TO 'pmm'@' localhost' IDENTIFIED BY 'pass' WITH MAX_USER_CONNECTIONS 5;
   GRANT SELECT, UPDATE, DELETE, DROP ON performance_schema.* TO 'pmm'@' localhost';

If the ``pmm`` user already exists,
simply pass its credential when you add the instance:

.. prompt:: bash

   sudo pmm-admin add mysql --user pmm --password pass

For more information, run ``sudo pmm-admin add mysql --help``.

Can I monitor multiple MySQL instances?
=======================================

Yes, you can add multiple MySQL instances
to be monitored from one *PMM Client*.
In this case,
you will need to provide a distinct port and socket for each instance
using the ``--port`` and ``--socket`` variables,
and specify a unique name for each instance
(by default, it uses the name of the PMM Client host).

For example, if you are adding complete MySQL monitoring
for two local MySQL servers,
the commands could look similar to the following:

.. code-block:: bash

   $ sudo pmm-admin add mysql --user root --password root --create-user --port 3001 instance-01
   $ sudo pmm-admin add mysql --user root --password root --create-user --port 3002 instance-02

For more information, run ``sudo pmm-admin add mysql --help``.

Can I rename instances?
=======================

You can remove any monitoring instance as described in :ref:`pmm-admin-rm`
and then add it back with a different name.

When you remove a ``linux:metrics``, ``mysql:metrics``,
or ``mongodb:metrics`` monitoring service,
previously collected data remains available in Grafana.
However, the metrics are tied to the instance name.
So if you add the same instance back with a different name,
it will be considered a new instance with a new set of metrics.

When you remove a QAN instance (``mysql:queries`` service),
previously collected data will no longer be available after you add it back,
regardless of the name you use.

.. _service-port:

Can I use non-default ports for instances?
==========================================

When you add an instance with the ``pmm-admin`` tool,
it creates a corresponding service that listens on a predefined client port:

+--------------------+---------------------+-------+
| General OS metrics | ``linux:metrics``   | 42000 |
+--------------------+---------------------+-------+
| Query analytics    | ``mysql:queries``   | 42001 |
+--------------------+---------------------+-------+
| MySQL metrics      | ``mysql:metrics``   | 42002 |
+--------------------+---------------------+-------+
| MongoDB metrics    | ``mongodb:metrics`` | 42003 |
+--------------------+---------------------+-------+

If a default port for the service is not available,
``pmm-admin`` automatically chooses a different one.

If you want to assign a different port, use the ``--service-port`` option
when :ref:`adding instances <pmm-admin-add>`.

.. _metrics-resolution:

What resolution is used for metrics?
====================================

The ``mysql:metrics`` service collects metrics with different resolutions
(1 second, 5 seconds, and 60 seconds)

The ``linux:metrics`` and ``mongodb:metrics`` services
are set up to collect metrics with 1 second resolution.

In case of bad network connectivity between *PMM Server* and *PMM Client*
or between *PMM Client* and the database server it is monitoring,
scraping every second may not be possible when latency is higher than 1 second.
You can change the minimum resolution for metrics
by passing the ``METRICS_RESOLUTION`` environment variable
when :ref:`creating and running the PMM Server container <server-container>`.
To set this environment variable, use the ``-e`` option.
The values can be between ``1s`` (default) and ``5s``.
If you set a higher value, Prometheus will not start.

For example, to set the minimum resolution to 3 seconds::

 -e METRICS_RESOLUTION=3s

.. note:: Consider increasing minimum resolution
   when *PMM Server* and *PMM Client* are on different networks,
   or when :ref:`amazon-rds`.

