# CouchDB Best Practices
Collect best practices around the CouchDB universe.

## Document Modeling to Avoid Conflicts
At the moment of writing, most of our data documents are modeled as ‘one big
document’. This is not according to CouchDB best practice. *Split data into many
small documents* depending on how often and when data changes.  That way, you can
avoid conflicts and conflict resolution by just appending new documents to the
database. Its often a good idea to have many small documents versus less big
documents. As a guideline on how to model documents *think about when data
changes*. Data that changes together, belongs in a document. Modelling documents
this way a) avoids conflicts and b) keeps the number of revisions low, which
improves replication performance and uses less storage.

## Your document ID is the best index
Before deciding on using a random value as doc `_id`, read the section [When not
to use map
reduce](http://pouchdb.com/2014/05/01/secondary-indexes-have-landed-in-pouchdb.html)

*Use domain specific document ids* where possible. With CouchDB it is best
practice to use meaningful ids.

When splitting documents into different subdocuments I often include the parent
document id in the id. Use [docuri](https://github.com/jo/docuri/) to centralize
document id knowledge.

## CouchDB Merge Conflicts
Some things need to and should be conflicts. CouchDB *conflicts are first class
citicens*, (or at least [should be treaded
so](https://gist.github.com/rnewson/2387973#file-gistfile1-txt-L6)). If 2
different users enter different addresses for the same person at the same time,
that probably should create a conflict. Your best option is to have a conflict
resolution daemon running on the server. While we don’t have this at the moment,
look at the
[pouch-resolve-conflicts](https://github.com/jo/pouch-resolve-conflicts) plugin.

## Data Migrations
Use [pouchdb-migrate](https://github.com/eHealthAfrica/pouchdb-migrate), a
PouchDB plugin to help with migrations.

## Proxy
Running CouchDB behind a proxy is recommended, iA. to handle ssl termination.
*Prefer subdomain over subdirectory*. At least nginx has problems to proxy a
CouchDB on a subdirectory.


