# CouchDB Best Practices
Collect best practices around the CouchDB universe.

**:warning: Currently this document is in a structure less collection mode,
sort of append only.**

* [Document Modeling To Avoid Conflicts](https://github.com/eHealthAfrica/couchdb-best-practices#document-modeling-to-avoid-conflicts)
* [Your Document ID Is The Best Index]()
* [Do Not Emit Entire Docs]()
* [Filtered Replication]()
* [Conflict Handling]()
* [Data Migrations]()
* [CouchDB Behind A Proxy]()
* [Linked Documents]()
* [Built-In Reduce Functions]()
* [Debugging PouchDB]()
* [Debugging Views]()
* [Testing Views]()
* [Change Password]()

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

## Your Document ID is the Best Index
Before deciding on using a random value as doc `_id`, read the section [When not
to use map
reduce](http://pouchdb.com/2014/05/01/secondary-indexes-have-landed-in-pouchdb.html)

*Use domain specific document ids* where possible. With CouchDB it is best
practice to use meaningful ids.

When splitting documents into different subdocuments I often include the parent
document id in the id. Use [docuri](https://github.com/jo/docuri/) to centralize
document id knowledge.

## Do Not Emit Entire Docs
You can query a view with `include_docs=true`. Then in the view result every row has
the whole doc included:
```json
{
  "rows": [
    {
      "id": "mydoc",
      "key": "mydoc",
      "value": null,
      "doc": {
        "_id": "mydoc",
        "_rev": "1-asd",
        "foo": "bar"
      }
    }
  ]
}
```
This has no disadvantages performance wise, at least on PouchDB. For CouchDB it
means additional lookups and prevents CouchDB from directly streaming the view
result from disk. But this is negligible. So don't emit the whole doc unless you
need the last bit of performance.

## Filtered Replication
Filtered replication is a great way limit the amount of data synchronized on a
device. But be aware that replication filters other than `_doc_ids` are very slow,
because they run on *every* document. Consider writing those filter functions in
Erlang.

## Conflict Handling
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

## CouchDB Behind A Proxy
Running CouchDB behind a proxy is recommended, eg. to handle ssl termination.

*Prefer subdomain over subdirectory*.  Nginx encodes urls on the way through.
So, for example, if you request
`http://my.couch.behind.nginx.com/mydb/foo%2Fbar` it gets routed to CouchDB as
`/mydb/foo/bar`, which is not what we want.

We can configure this mad behaviour away (by [not appending a slash to the
`proxy_pass`
target[(http://stackoverflow.com/questions/20496963/avoid-nginx-decoding-query-parameters-on-proxy-pass-equivalent-to-allowencodeds)).
But there is no way to convince nginx not messing with the url when rewriting
the proxy behind a subdirectory, eg
`http://my.couch.behind.nginx.com/_couchdb/mydb/foo%2Fbar`


## Linked Documents
Or How do I do SQL-like JOINs? Can I avoid them?
CouchDB (and PouchDB) supports [linked
documents](https://wiki.apache.org/couchdb/Introduction_to_CouchDB_views#Linked_documents).
Use them to join two types of documents together, by simply adding an `_id` to the
emitted value:
```js
function map(doc) {
  // join artist data to albums
  if (doc.type === 'album') {
    emit([doc._id, 'album'], null)
    emit([doc._id, 'artist'], { _id : doc.artistId })
  }
}
// query an album
db.query(map, {
  key: 'hunkydory',
  include_docs: true
})
```
And this is a result:
```json
{
  "rows": [
    {
      "doc": {
        "_id": "hunkydory",
        "title": "Hunky Dory",
        "year": 1971
      },
      "id": "hunkydory",
      "key": [
        "hunkydory",
        "album"
      ],
      "value": {
        "_id": "bowie"
      }
    }
    {
      "doc": {
        "_id": "bowie",
        "firstName": "David",
        "lastName": "Bowie"
      },
      "id": "hunkydory",
      "key": [
        "hunkydory",
        "artist"
      ],
      "value": {
        "_id": "bowie"
      }
    }
  ]
}
```
Using linked documents can be a way to group together related data.


## Built-In Reduce Functions
CouchDB has some build in reduce functions to accomplish common tasks. They are
native and very performant. Choose them wherever possible.
To use a built in reduce, insert the name string instead of the function code,
eg
```json
{
  "views": {
    "myview": {
      "map": "function(doc) { emit(doc._id, 1) }",
      "reduce": "_stats"
    }
  }
}
```

### `_sum`
Adds up the emitted values, which must be numbers.

### `_count`
Counts the number of emitted values.

### `_stats`
Calculates some numerical statistics on your emitted values, which must be
numbers.
For example:
```json
{
  "sum": 80,
  "count": 20,
  "min": 1,
  "max": 100,
  "sumsqr": 30
}
```

## Debugging PouchDB
I often assign the database instance to window and then I run queries on it.
Or you can replicate to a local CouchDB and debug your views there.

If you like PouchDB to be more verbose, [enable debug
output](http://pouchdb.com/api.html#debug_mode):
```js
PouchDB.debug.enable('*')
```

## Debugging Views
View debugging can be a pain when you're restricted to Futon or even Fauxton.
By using [couchdb-view-tester](https://github.com/gr2m/couchdb-view-tester) you
can write view code in your preferred editor and watch the results in real time.


## Testing Views
Use [couchdb-ddoc-test](https://github.com/eHealthAfrica/couchdb-ddoc-test), a a simple
CouchDB design doc testing tool.
```js
var DDocTest = require('couchdb-ddoc-test')
var test = new DDocTest({
    fixture: {a: 1},
      src: 'path/to/map.js'
})
var result = test.runMap()

assert.equals(result, fixture)
```


## Change Password
Since CouchDB 1.2 updating the user password has become much easyer:

1. Request the user doc: `GET /_users/org.couchdb.user:a-username`
2. Set a `password` property with the password
3. Save the doc

Use the [User authentication plugin for PouchDB and
CouchDB](https://github.com/nolanlawson/pouchdb-authentication).

