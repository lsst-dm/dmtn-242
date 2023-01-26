:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml

.. TODO: Delete the note below before merging new content to the main branch.

.. note::

   **This technote is a work-in-progress.**

Abstract
========

In DMTN-176 we examined the requirements for a Butler client/server and proposed a path towards a prototype. In this document we expand on the implementation plan based on that experience.

This is a living document that we hope to evolve as we finalize design decisions and implementations.

Introduction
============

A client/server Butler is now a key part of the Butler infrastructure required for user-facing deployments on the Cloud Data Facility.
In :cite:`DMTN-176` we described the initial plans for a Butler client/server implementation.
As part of that work a prototype was written that implemented the core ``Registry`` APIs.
The prototype was implemented as follows:

* The ``registry`` section of the Butler configuration can now specify a Python class that should be configured.
  For client/server this registry class is a ``RemoteRegistry`` that takes a URL of a corresponding server as a parameter.
  It used the Python ``httpx`` package to talk to the server.
* The prototype server code was written using FastAPI_ as recommended by the SQuaRE team.
* To leverage the FastAPI_ infrastructure we constructed Pydantic models of all the key data structures (``DatasetRef``, ``DataCoordinate``, ``DimensionRecord``) and those are what are sent over the server connection to the client in JSON form.

With this implementation we were able to demonstrate that the command line tools such as ``butler query-datasets`` and ``butler query-dimension-records`` could work without modification solely by changing the Butler configuration file to point to the relevant server and registry class.

We did not consider authentication and authorization in the prototype.

Prototype Limitations
=====================

As a prototype it was a great success but did raise some issues:

1. Lack of pagination.
2. Rich query objects.
3. Registry modification.
4. Situations where many queries are needed (e.g., graph building).

Query Pagination
----------------

It is possible for some queries to be very large, too large to efficiently return to the client, such that a pagination interface would be required where the server would have to know which queries it had active and return the results to the correct client as needed.
The server would need to cache these query objects and expire them if a client has not requested more results in a while.

We need to investigate the different scenarios available for pagination and determine the best way forward.

Currently the butler query system supports ``ORDER BY`` and ``LIMIT`` when querying datasets or dimension records.
For pagination either the server has to remember state or the client has to resend the query with sufficient information to retrieve subsequent pages.
Remembering state in the server might be more efficient in terms of database resource usage but comes with difficulties associated with timing out of queries and making the state available if multiple Butler servers can handle requests.

A simpler alternative is to add ``OFFSET`` to the query objects and use that in the client to request successive pages, generating a new query in the server each time.
This would require that the client always specifies an ``ORDER BY`` if one is not provided by the user and also handles the user-specified ``LIMIT`` by converting that into pages.
Using ``OFFSET`` can be problematic if deletions occur during the page queries since that can result in some results being skipped.
This is not a problem for dimension records (that are essentially append only in most use cases) but is something that can occur for dataset queries.

Russ Allbery recommends that we also consider keyset pagination where, effectively, an additional piece of information is returned to the client which can then be sent back to the server when the next page is needed and that information turned into a ``WHERE X > Y`` modifier to the query.
An interesting discussion of different pagination options can be found at https://www.citusdata.com/blog/2016/03/30/five-ways-to-paginate/

A further suggestion is for the server to write all the paged results to a temporary file or files (in either JSON or parquet format) and return to the client the locations.
The client can then read each page and return the results.
File deletion can be handled by informing the object store of an expiration date.

Rich Query Objects
------------------

The return values of ``queryDatasets`` and ``queryDimensionRecords`` were modified since the prototype such that the object they return can be inspected before triggering the return of results.
The prototype did not allow this and some thought is required as to how to handle this in the client where a result is needed from the server before the full query is executed.
In particular we now recommend that people inspect the returned object for "doomed" queries to explain why no results are returned as well as the ``expanded`` method implementation indicating that full dimension records should be attached to the results.
This requires that we change the interface to return the new object and then use that object to return the results -- potentially requiring the query to be constructed multiple times unless query state is retained in the server.

Registry Modification
---------------------

Almost no methods were implemented in the prototype that involved modification of registry.
The only exception was a simple interface to registering a ``RUN`` collection to show that it could be done even if there was no possibility of rollback.

Transaction management is a key facility for methods that involve updates to the registry since there are many cases where multiple registry calls are made, possibly in conjunction with updates to datastore, where any failure should trigger a full rollback of the transaction.

The handling of transactions in a client/server environment is possible, for example returning a transaction identifier and using that explicitly in future calls using that transaction, but adds complications in terms of transaction lifetimes (for example how long does the server wait for the client to contact it again) and state management.

We may decide that some write interfaces in registry should be deferred initially in favor of higher-level APIs that can keep the transaction entirely on the server side.

The registry currently supports the following interfaces that require transaction handling:

* Certification, decertification, association, and disassociation of datasets.
* Removal of datasets and collections.
* Modification of collection chains and registering new collections.
* Inserting/importing of datasets.

Removing, inserting and importing of datasets is almost always invoked from a ``Butler`` and if there is a client/server ``Butler`` class we could avoid a need to implement those registry methods in the client code.

Certification and association are not part of the ``Butler`` interface and should be part of the client/server interface.
The API for these is relatively simply and could be implemented in an initial release with the caveat that we might not support its use in transactions.

Many Queries
------------

Queries are already thought to be a little slow but queries in a client/server context will have additional overhead over direct connection to the database that could become significant.
One common scenario where this will be impactful is in Quantum Graph building.
This is already something that can take hours or days to complete and adding additional overhead will not be acceptable.
This suggests that there should be a client/server interface for graph building, possibly with the option of writing the graph to a temporary location and then returning a URL to that location -- the graph I/O interface already supports this type of access.

Given graph building can take many hours, this should involve an async interface where the initial request returns an identifier that can be used to monitor progress in the client.

This also implies that we will likely need a client/server interface to BPS submissions.


Butler Only?
============

We have briefly considered whether a Butler client/server could be constructed solely by defining a new expanded interface that includes some of the more commonly-used registry methods but does not include explicit access to a fully-featured datastore and registry.
This would, though, still require that common registry query methods for querying datasets, dimension records, dataset types, and dataIds, be available to clients, possibly requiring that they be moved to ``Butler`` methods (but still with the associated problems of pagination described above).

The prototype has already demonstrated that these query interfaces can be implemented within a "remote" registry implementation, and changing all the existing user code to stop using ``.registry`` methods would be a large undertaking that will likely not be approved.
Furthermore, the conceptual divide between butler, datastore, and registry has shown itself to be very useful.
Allowing a butler datastore to be a combination of a remote datastore and other types of datastores, is a very useful feature in our datastore implementation and losing that would be a backwards step.

For this reason we no longer believe that we gain anything by trying to make a special ``Butler`` interface for client/server and will pursue the original plan of remote registry, remote datastore and potentially a remote ``Butler`` that can call those interfaces.

It is still unreasonable for a client user to have to know whether they are using a client/server Butler or a direct-connection Butler and so ``butler = Butler(config)`` must still be seamless from the user perspective.
Requiring butler users to have different code paths depending on whether they think they are attaching to a remote server or a local butler would be a terrible situation prone to confusion.


Butler Client/Server
====================

In this section we discuss the important ``Butler`` APIs and discuss how they might be implemented in a client/server situation and compare them with the current ``Butler`` implementation.

Many of the discussions below involve signed URLs and/or staging areas.
An object store or WebDAV server are not required for butler client/server since in theory any form of URI can be used that is supported by ``ResourcePath``.
Signed URLs are required in order to conform with the authorization requirements when using the Butler client/server in production.

Butler "get"
------------

Retrieving data from a Butler is one of the simplest butler operations.
The default implementation is:

1. Map the provided parameters to a single ``DatasetRef``.
2. Ask the datastore to retrieve the Python object associated with that ref.

This can be the same implementation for the client/server implementation if the code involved in step 1 (expanding the dataId to handle dimension record information being provided rather than full dimensions, querying the registry), ``Butler._findDatasetRef()``, was moved into the registry.
This new registry method could still be treated as a private method even if it becomes a public part of the server interface.

Step 2 would have to involve a client/server-aware file-based datastore.
The individual steps inside datastore are:

1. Query the datastore internal registry for the relevant records.
2. Use the relevant formatters to read the data and assemble into a Python object.

The internal registry will be on the server and will be required to return full URLs, generally signed, either to an object store or WebDAV server (as supported by the ``resources`` package) to allow the client to access the data.
This is different from the usual approach where paths are used and the URL is constructed from the datastore root, but is required given the authentication and authorization requirements for data access from butler client/server.

The file datastore implementation currently uses a "bridge" class to access registry "opaque" tables.
The simplest way to implement a server-aware datastore would be to change the bridge class without changing any of the datastore implementation itself.
This will need to be investigated and might require that the bridge APIs be modified to make them more efficient.

Butler "put"
------------

A ``put()`` is more complicated than a ``get()`` and has to worry about transactions and rollbacks and whether the client has permissions to write to the collection :cite:`DMTN-182`.
There can not be a ``put()`` server method that takes the python object because the point of the datastore is to serialize the python object into a form that can be transferred to the server.

The current implementation of ``Butler.put()`` runs as a transaction that can roll back the registry inserts and datastore writes if there is a problem.
It does this by starting a transaction, then inserting the registry record, and finally asking datastore to do the serialization and the internal record table updates (which currently require that registry has the primary key).
The registry entry is inserted first to quickly ensure that there is no entry already present for this collection, dataset type, and data ID, before the datastore is asked to do the much slower serialization step.
It used to be that the registry insert had to be done early because the registry was required to allocate the dataset ID when an incrementing integer was used and this could not be predicted.
UUIDs do not have this problem and can be allocated without asking registry.

If there is no support for client/server transactions during a ``put()`` the writes to the registry and datastore must be handled in a transaction solely contained within the server.
This then suggests that the client/server ``put()`` should look something like:

1. The Butler client allocates a new ``DatasetRef`` for this dataset.
   Optionally this dataset ref is compared to registry to see if there is a conflict -- no conflict does not imply that the put will succeed but a conflict will allow the attempt to be curtailed early.
2. The client calls ``datastore.put`` with this ref.
   This will fail if the client/server datastore is using a bridge manager that relies on the server already knowing about this ref.
3. The client datastore serializes all the files locally and creates associated datastore records.
4. The client datastore requests the required number of signed URLs from the server for an upload location (presumably to a staging area, possibly with temporary file names).
5. Client datastore transfers the files.
6. Client datastore sends the ``DatasetRef`` and datastore records (modified to use the relevant temporary file names) to the server -- it is possible that these records are stored in a staging area pending acceptance.
7. The client tells the Butler server to finalize the put for this ref.
   The server ensures that the ref can be stored (the server must have explicit knowledge of a butler and associated registry), and ingests the files from the temporary staging area as if this is a standard Butler import.
   This would likely be a private API in the Butler server interface.
   This step can rollback if necessary since it is self-contained in the server.
8. The server tells the client that the dataset has been accepted.

It might be possible for this to be hidden from datastore using the bridging interface but the bridging interface would likely have to be modified to understand that the final file URIs can not be guessed by the datastore implementation.

Changing the ``put()`` to be generic such that it is the same for client/server and direct SQL access does not seem to be desirable because of the rollback requirement even if we are happy always serializing first before finding out if the dataset can be accepted.
Our assumption is that this will need a client/server aware ``Butler`` class.

Butler "transfer_from"
----------------------

Once there is a client/server Butler, people will want to be able to do transfers from that butler to a local butler or vice versa.
Transferring from a client/server Butler to a local Butler is fairly straight forward at the Butler level since it receives a collections of refs.
The butler part mostly ensures that dimension records and run collections are transferred over -- this code could be identical in both client/server Butler and direct-access Butler except that a transaction is involved to allow rollback of any dimension inserts and registry imports.
Since dataset inserts are necessary there must be a bespoke client/server implementation.

The Datastore side is more complicated in that we currently only support ``FileDatastore`` to ``FileDatastore`` transfers (which can take shortcuts by realizing that they both share the same records format and so allow for use of internal methods that access opaque tables) and there is not even support for transfers involving a chained datastore.
More thought would be needed to allow two different datastore classes to transfer file records but it might help if ``ServerFileDatastore`` is a subclass of ``FileDatastore`` and all records access is handled through server methods.
This might be possible with the bridge interface and for transfers from the server signed URLs would be generated.

For transfers to the server it would be required that pre-signed URLs are generated for every file.
There then has to be a decision as to whether those transfers are to a staging area or to the final server datastore location.
Currently the ``Datastore.transfer_from()`` method does not return any values, implying that a similar approach to ``put()`` must be used where the client datastore uploads the files and the records and the Butler server then locates the records file and finalizes import within the transaction.

Packaging
=========

The prototype was implemented with all the client code distributed as part of ``daf_butler`` via a ``RemoteRegistry`` class that was selectable by changing the Butler configuration file.
The server was distributed as a standalone package (https://github.com/lsst-dm/butler-server) which made it difficult to include in tests.
The SQuaRE team recommend that eventually the client code be distributed on its own and for testing purposes have it depend on the server code, and then have both of those depend on ``daf_butler``.
This will make it simpler for the client/server interface to change at a different cadence to core ``daf_butler`` and potentially simplify server version migrations.

For the initial development, where client/server interfaces will likely be changing continually, along with potentially internal changes to ``daf_butler`` as features are needed, we recommend that we add both the client and server code to the ``daf_butler`` distribution and mark them as experimental.

Async
=====

FastAPI will use threading if an API is not async/await aware.
This can involve some overhead and is not the recommended way to run a FastAPI server.
None of the Butler code is async compatible and in the past there wasn't great support for async in Sqlalchemy.
Now that Sqlalchemy supports async and there are the `asyncpg` and `aiosqlite` libraries for database connectivity, we should at least consider a timeline for migrating all of butler to support async.
This will be a large amount of work but can be started from the bottom up and improved over time.
Performance of the server will become critically important as we approach Data Release 1 with 10,000 users who will all be required to access Butler resources through the server.

Conclusions
===========

It would seem that to satisfy the main use cases we would need more than a single Butler client/server interface.

* A Butler client/server is the only efficient way to support put and get operations (rather than trying to use a generic Butler with server registry and datastore) but we need to be able to create a local butler or client/server butler from ``Butler(config)`` to avoid confusion and code changes when switching from a local to remote Butler.
* People will still need registry query methods so we still need a way to implement pagination and a query object in the client even if most calls are queries and not updates.
* For dataset association and certification, how are transactions handled?
  Do we ignore transactions in the client and assume that all refs will be sent to the server with no ability to rollback if a later registry call fails?
  Do we try to rollback as we do in datastore by keeping a record of the calls made to the server and try to apply the reverse and, say, decertify on raise?
* A client/server Butler being able to use a ``Datastore`` that may or may not be a client/server ``Datastore`` (and could therefore support a ``ChainedDatastore``) seems like it could be useful given the requirement for the client to reuse large parts of ``FileDatastore`` to do the reading and writing of files.
* Graph building (and possibly BPS submissions) will need their own client/server code.
  The difficulty is determining whether it is possible to make ``pipetask qgraph`` work out automatically that it is attached to a server or if an entirely new ``pipetask-client`` is needed.
* The new interface must support authorization tokens, even if they are not checked initially.
  Some design work is needed to determine what the server does with collection constraints -- are all collection requests checked before execution or are results filtered before being returned to the client?
* ``httpx`` will have to be added to the base ``rubin-env``.

.. _FastAPI: https://fastapi.tiangolo.com

.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
