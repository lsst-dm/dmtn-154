:tocdepth: 1

.. sectnum::

Overview
========

Data Backbone (DBB) buffer managers are two separate pieces of software. The
first being the **handoff manager** which is responsible for transferring files
from a remote location (**handoff site**) to a designated location at the data
facility, an **endpoint site**. The second piece of software is the **endpoint
manager** which is responsible for ingesting files into different data
management systems operating on the endpoint site, e.g., Gen2 and/or Gen3
Butler repositories.

Handoff manager
===============

The handoff buffer manager transfers files between two **buffers**. New image
files are written to a storage location (handoff buffer) by the Archiver
process.  After successfully transferring the file to the endpoint buffer, the
manager moves the file from original storage location to another directory, a
**holding area**, thus managing what files have been transferred to the
endpoint site.

.. figure:: /_static/dataflow.svg
   :width: 500px
   :align: center
   :name: Figure 1

   Schematic representation of the data flow between handoff and endpoint
   buffers.

It is worth noting that:

#. The manager is a daemon-like process running continuously on the handoff
   site.
#. The manager considers each file in the handoff buffer to be a new file ready
   for transfer. Hence, for the manager to operate correctly, writing files to
   the handoff buffer must be an atomic operation.
#. Similarly to `rsync`_, the manager replicates the directory structure
   between buffers. For example, the file in the handoff buffer located at
   ``foo/bar/baz.fits`` will be written to exactly the same location in the
   endpoint buffer.
#. File transfer mechanism can be easily changed via the configuration file
   providing that the transfer of specific files (not directories) can be
   expressed as a command with placeholders for the source(s) and destination.

.. note::

   The current deployments of the handoff manager use `bbcp`_, so the handoff
   manager must first transfer the file to a scratch staging area before
   atomically moving to the endpoint buffer.

.. _rsync: https://rsync.samba.org/
.. _bbcp: https://www.slac.stanford.edu/~abh/bbcp/

Endpoint manager
================

The endpoint manager consists of two types of core components: **Finders** and
**Ingesters**. They use a relational database management system (RDBMS) to
provide useful information about files in the storage area and details
regarding ingestion attempts that were made for each file.

.. figure:: /_static/endmgr.svg
   :width: 500px
   :align: center
   :name: Figure 2

   Overview of the architecture of the endpoint buffer manager.  Solid arrows
   indicate file movement between the buffer and the storage area, dashed
   arrows show ingestion process (no file movement, only linking/copying), and
   dotted lines show database updates.

A Finder is responsible for discovering new files arriving at a specified
location and moving them, if necessary, to their final destination, **storage
area**. While doing so it populates the database table that acts as a single
source of truth with regard to the content of the storage area. Each Finder
monitors a single location, but multiple Finders can be deployed on a given
endpoint site, each monitoring a different location.

An Ingester is responsible for making ingest attempts to a data management
system on the endpoint site.  Similarly to Finders, multiple Ingesters can be
deployed on a given endpoint site to ingest images to different data management
systems, e.g., Gen2 and Gen3 Butler repositories. In addition to LSST Butler
repositories, an Ingester could also register files in place with `Rucio`_
instead of uploading the files to Rucio as the files in the storage area can
possibly be used by multiple Ingesters.

Each Ingester stores in the RDBMS a complete history of events for each file
allowing monitoring systems (or interested parties) to quickly find out
information including, but not limited to:

* at files are currently ingested,
* which ingests attempts succeeded or failed,
* when an attempt was made,
* how long it took to ingest a file.

Finders and Ingesters operate independently hence catastrophic failure in one
of the components will not propagate to others taking the entire DBB endpoint
manager offline. It also means that they can be maintained and/or upgraded at
different cadences.

.. _Rucio: https://rucio.cern.ch/
