:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml

.. TODO: Delete the note below before merging new content to the main branch.

.. note::

   **This technote is a work-in-progress.**

Abstract
========

In DMTN-176 we examined the requirements for a Butler client/server and proposed a path towards a prototype. In this document we expand on the implementation plan based on that experience.

Introduction
============

A client/server Butler is now a key part of the Butler infrastructure required for user-facing deployments on the Cloud Data Facility.
In :cite:`DMTN-176` we described the plans for a Butler client/server implementation.
As part of that work a prototype was written that implemented the core ``Registry`` APIs.
The prototype was implemented as follows:

* The ``registry`` section of the Butler configuration can now specify a Python class that should be configured.
  For client/server this registry class is a ``RemoteRegistry`` that takes a URL of a corresponding server as a parameter.
  It used the Python ``httpx`` package to talk to the server.
* The prototype server code was written using FastAPI_ as recommended by the SQuaRE team.
* To leverage the FastAPI_ infrastructure we constructed Pydantic models of all the key data structures (``DatasetRef``, ``DataCoordinate``, ``DimensionRecord``) and those are what are sent over the server connection to the client in JSON form.

With this implementation we were able to demonstrate that the command line tools such as ``butler query-datasets`` and ``butler query-dimension-records`` could work without modification solely by changing the Butler configuration file to point to the relevant server and registry class.

As a prototype it was a great success but did raise some questions:

1. It is possible for some queries to be very large such that a pagination interface would be required where the server would have to know which queries it had active and return the results to the correct client as needed.
   The server would need to cache these query objects and expire them if a client has not requested more results in a while.
2. The return values of ``queryDatasets`` and ``queryDimensionRecords`` were modified since the prototype such that the object they return can be inspected before triggering the return of results.
   The prototype did not allow this and some thought is required as to how to handle this in the client where a result is needed from the server before the full query is executed.
   In particular we now recommend that people inspect the returned object for "doomed" queries to explain why no results are returned as well as the ``expanded`` method implementation indicating that full dimension records should be attached to the results.
   This presumably requires the server to cache results as in the previous point.
3. No methods were implemented that involve modification of registry.
   This is particularly important for transaction management since there has to be a way for multiple updates to be combined within a single database transaction inside the server such that the client can cause a rollback (or a rollback can happen if the client disappears with the transaction still active).
   How are transactions managed in the server with multiple clients connecting simultaneously?
4. Queries are already thought to be a little slow but it seems unreasonable for ``pipetask qgraph`` to try to build a quantum graph with many queries and dataset lookups involving a server.
   This suggests that there should be a client/server interface for graph building, possibly with the option of writing the graph to a temporary bucket and then returning a signed URL to its location.

Query Pagination
----------------

Currently the butler query system supports ``ORDER BY`` and ``LIMIT`` when querying datasets or dimension records.
For pagination either the server has to remember state or the client has to resend the query with sufficient information to retrieve subsequent pages.
Remembering state in the server might be more efficient in terms of database resource usage but comes with difficulties associated with timing out of queries and making the state available if multiple Butler servers can handle requests.

A simpler alternative is to add ``OFFSET`` to the query objects and use that in the client to request successive pages, generating a new query in the server each time.
This would require that the client always specifies an ``ORDER BY`` if one is not provided by the user and also handles the user-specified ``LIMIT`` by converting that into pages.

Russ Allbery recommends that we also consider keyset pagination where, effectively, an additional piece of information is returned to the client which can then be sent back to the server when the next page is needed and that information turned into a ``WHERE X > Y`` modifier to the query.
An interesting discussion of different pagination options can be found at https://www.citusdata.com/blog/2016/03/30/five-ways-to-paginate/

Butler Only?
============

We are now considering implementing client/server as purely a ``Butler`` interface with no direct access to ``registry`` or ``datastore``.
This would, though, still require that common registry query methods for querying datasets, dimension records, dataset types, and dataIds, be available to clients, possibly requiring that they be moved to ``Butler`` methods (but still with the associated problems of pagination).

It is still unreasonable for a client user to have to know whether they are using a client/server Butler or a direct-connection Butler and so ``butler = Butler(config)`` must still be seamless from the user perspective.

The registry currently supports the following interfaces that require transaction handling:

* Certification, decertification, association, and disassociation of datasets.
* Removal of datasets and collections.
* Modification of collection chains.
* Inserting/importing of datasets.

Removing, inserting and importing of datasets is almost always invoked from a ``Butler``.
Certification and association are not part of the ``Butler`` interface and should be part of the client/server interface.

In the following sections we discuss whether it is feasible to have solely a Butler client/server for common operations.

Butler "get"
------------

Retrieving data from a Butler server must be split into a server component and a client component.
First the relevant signed URIs for the dataset must be requested from the server along with the rest of the information stored in the file datastore records.
Then the client has to load the files using the specified formatters and assemble them into the final python dataset applying the parameters and any storage class overrides.
This would require some reorganization of the ``FileDatastore.get`` method to allow for a new method that could be called by the client when the relevant datastore records are already known.

What is not clear is whether this is compatible with a chained datastore.
In the current Butler design we explicitly separate datastore from butler and allow the datastore to be itself an abstraction.
This abstraction is fundamentally at odds with explicitly stating that a client/server butler can only support a single ``FileDatastore``.
Keeping this abstraction would require us to retain an explicit ``datastore.get()`` server interface such that an in-memory datastore could be chained with a remote file datastore (although how a client could set up a local chain if they are retrieving their butler configuration file from the server is a difficult question).
The ``butler.get()`` would then effectively be a registry query for the client which would return the ``DatasetRef``.
Then that would be passed to the datastore and the remote datastore would be a separate URI and datastore records query before the client reading the files as if part of a normal ``FileDatastore``.

A possible scenario:

1. Butler client sends query parameters to Butler server which will return the relevant ``DatasetRef`` (can be skipped if the user provides a ``DatasetRef``).
2. Butler client will then call ``datastore.get()`` with that ref and parameters, rejecting it if the user does not have permission.
3. If the ``Datastore`` is a client/server datastore implementation (a subclass of ``FileDatastore``) it will pass the ref to the server and receive the datastore records.
4. With the records it will now have formatter and (signed) URLs to the relevant files.
   These should now be used to create the Python object just like any other ``FileDatastore`` would work.
5. Return the Python object to the user.


Butler "put"
------------

A ``put()`` is more complicated than a ``get()`` and has to worry about transactions and rollbacks and whether the client has permissions to write to the collection :cite:`DMTN-182`.
There can not be a ``put()`` server method that takes the python object because the point of the datastore is to serialize the python object into a form that can be transferred to the server.

The client must first register the dataset with the server (or at least determine the ``DatasetRef`` and check that it does not already exist in registry), then, assuming this is a ``FileDatastore``, the client must serialize the data to one or more (local) files and calculate the associated datastore records.
The server must then issue signed URLs for the client to use to transfer the files to the datastore.
Should the server store the records at that point or wait for the client to report that the files were transferred?
What happens if the client never reports completion?
Should the client transfer the files to a staging area and then call a method in the server with the ``DatasetRef`` definition and datastore records so that the server only receives one standalone write request?

The existence of chained datastores suggests that the client must be able to support a client/server datastore implementation that can be called from the client/server Butler.
This does raise the question of when the dataset should be registered and whether the client/server datastore can receive a dataset and implicitly do the registration with registry (a server "datastore" could know about the registry in a way that the client datastore can not).

A possible scenario:

1. The Butler client allocates a new ``DatasetRef`` for this dataset.
2. The client calls ``datastore.put`` with this ref (this will fail if any of the datastores are somehow using a datastore that has an opaque table attached to a registry that will not have had the ref defined but this should not be possible since no opaque registry can be visible to the client if there is no support for opaque data in the client/server registry implementation).
3. The client datastore serializes all the files locally and creates associated datastore records.
4. The client datastore requests the required number of signed URLs from the server for an upload location (presumably to a staging area, possibly with temporary file names).
5. Client datastore transfers the files.
6. Client datastore sends the ``DatasetRef`` and datastore records (modified to use the relevant temporary file names) to the server.
7. The server ensures that the ref can be stored (the server must have explicit knowledge of a butler and associated registry), and ingests the files from the temporary staging area as if this is a standard Butler import.
8. The server tells the client that the dataset has been accepted.

Butler "transfer_from"
----------------------

Once there is a client/server Butler, people will want to be able to do transfers from that butler to a local butler or vice versa.
Transferring from a client/server Butler to a local Butler is fairly straight forward at the Butler level since it receives a collections of refs.
The Datastore side is more complicated in that we currently only support ``FileDatastore`` to ``FileDatastore`` transfers (which can take shortcuts by realizing that they both share the same records format and so allow for use of internal methods that access opaque tables).
There is not even support for transfers involving a chained datastore.
More thought would be needed to allow two different datastore classes to transfer file records but it might help if ``ServerFileDatastore`` is a subclass of ``FileDatastore`` and all records access is handled through server methods.
This effectively moves some of the private python methods into public server methods, but removes the need to try to support a full opaque storage manager client/server plugin for datastore.

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
* Is the server code included in ``daf_butler`` itself or is it a new package?
  Given the client code is tightly coupled to the server implementation it seems reasonable for testing if the client and server code is in the same package and butler users would likely prefer to install a single package for all butler implementations.
* ``httpx`` will have to be added to the base ``rubin-env``.

.. _FastAPI: https://fastapi.tiangolo.com

.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
