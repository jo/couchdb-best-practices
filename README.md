# CouchDB Best Practices
Collect best practices around the CouchDB universe.

**:warning: Currently this document is in a structure less, append only.**

* [Creating Admin User](#creating-admin-user)
* [Creating User](#creating-user)
* [Document Modeling To Avoid Conflicts](#document-modeling-to-avoid-conflicts)
* [Your Document ID Is The Best Index](#your-document-id-is-the-best-index)
* [Do Not Emit Entire Docs](#do-not-emit-entire-docs)
* [Filtered Replication](#filtered-replication)
* [Conflict Handling](#conflict-handling)
* [Data Migrations](#data-migrations)
* [CouchDB Behind A Proxy](#couchdb-behind-a-proxy)
* [Linked Documents](#linked-documents)
* [Built-In Reduce Functions](#built-in-reduce-functions)
* [Debugging PouchDB](#debugging-pouchdb)
* [Debugging Views](#debugging-views)
* [Testing Views](#testing-views)
* [Deploying Views](#deploying-views)
* [Modularize View Code](#modularize-view-code)
* [Change Password](#change-password)
* [PouchDB and AngularJS](#pouchdb-and-angularjs)
* [Full Text Search](#full-text-search)
* [Using Replication](#using-replication)


## Creating Admin User
First thing to do is setup the user accout

* Go to [http://localhost:5984/_utils/](http://localhost:5984/_utils) in your
* web browser
* Click on `Setup more admins` in the bottom right hand corner of the sidebar
* Enter a username + password (this will be the root admin of your CouchDB)
* Click on `Logout` which is also in bottom right had corner

Great, now you've configured your Admin User. However, you don't want to
actually use this account to do things, so proceed!


## Creating User
To create a non admin user, follow these steps:

* While still on `http://localhost:5984/_utils/` in your browser (and logged
* out)
* Click on `Signup` in the bottom right of the sidebar
* Enter username + password


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
device.

Filters can be by id (`_doc_ids`), by filter function, or by view (`_view`).

Be aware that replication filters other than `_doc_ids` are very slow,
because they run on *every* document. Consider writing those filter functions in
Erlang.

When using filtered replication think about deleted documents, and whether they
pass the filter. One way is to filter by id (this might be an argument for keeping
`type` in the id).  
Or deletion can be implemented as an update with `_deleted: true`. That way data
is still there and can be used in the filter.


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

You can either run migration on startup (the plugin remembers if a replication has
already been run, so the extra cost on startup is getting a single local document)
or you can setup a daemon on the server.

Its also possible to run a migration manually from a dev computer.

Please take care of your migration function. It *must* recognice whether a doc
already has been migrated, otherwise the migration is cought in a loop!


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


## Deploying Views
You want to keep your view code in VCS. Now you need a way to install the view
function. You can use the [Python Couchapp
Tool](https://github.com/couchapp/couchapp). Its a CLI which reads a directory,
compiles documents from it and saves the documents in a CouchDB.
Since this is a common task a quasi standard for the directory layout has been
emerged, which is supported by a range of compatible tools.
For integration in JavaScript projects using a tool written in JavaScript avoids
additional dependencies. I wrote
[couch-compile](https://github.com/jo/couch-compile) for that purpose. It reads
a directory, JSON file or node module and converts it into a CouchDB document,
which can be deployed to a CouchDB database with
[couch-push](https://github.com/jo/couch-push).
There is also a handy Grunt task,
[grunt-couch](https://github.com/jo/grunt-couch) which integrates both projects
into the Grunt toolchain.


## Modularize View Code
While you work with views at some point you might want to share code between
views or simply break your code into smaller modules. For that purpose CouchDB
has [support for CommonJS
modules](http://docs.couchdb.org/en/1.6.1/query-server/javascript.html?highlight=commonjs#commonjs-modules),
which you know from node.
CouchDB implements [CommonJS 1.1](http://wiki.commonjs.org/wiki/Modules/1.1).
Please keep the version in mind. Some features are not supported, for example
the assignment of a function, like `module.exports = function() {}`.

Take this example:
```js
{
  views: {
    lib: {
      {
        models: "                                   \
          exports.person = function(doc) {           \
            if (!doc._id.match(/^person\//)) return   \
            return {                                   \
              name: doc.firstname + ' ' + doc.lastname, \
              createdAt: Date.parse(doc.created_at)      \
            }                                             \
          }                                                \
        "
      }
    },
    'people-by-name': {
      map: function(doc) {
        var person = require('views/lib/models').person(doc)
        if (!person) return
        emit(person.name, null)
      },
      reduce: '_count'
    },
    'people-by-created-at': {
      map: function(doc) {
        var person = require('views/lib/models').person(doc)
        if (!person) return
        emit(person.createdAt.getTime(), null)
      },
      reduce: '_count'
    }
  }
}
```
CommonJS modules can also be used for shows, lists and validate\_doc\_update
functions.
Note that the `person` module is inside the `views` object. This is needed for
CouchDB in order to detect changes on the view code.
Reduce functions *can NOT* use modules.


## Change Password
Since CouchDB 1.2 updating the user password has become much easyer:

1. Request the user doc: `GET /_users/org.couchdb.user:a-username`
2. Set a `password` property with the password
3. Save the doc

The [User authentication plugin for PouchDB and
CouchDB](https://github.com/nolanlawson/pouchdb-authentication) provides a
[`changePassword`](https://github.com/nolanlawson/pouchdb-authentication#user-content-dbchangepasswordusername-password--opts-callback)
function for your convenience.


## PouchDB and AngularJS
To use PouchDB with AngularJS you should use
[angular-pouchdb](https://github.com/angular-pouchdb/angular-pouchdb), which
wraps PouchDBs promises with Angulars `$q`s.


## Full Text Search
While basic full text search is possible just by using views, its not
convenient and you should make use of a dedicated FTI.

### Client Side
For PouchDB the situation is clear: Use [PouchDB Quick
Search](https://github.com/nolanlawson/pouchdb-quick-search). Its a PouchDB
plugin based on [lunr.js](http://lunrjs.com/).

### Server Side
On a CouchDB server you have options: Lucene (via
[couchdb-lucene](http://github.com/rnewson/couchdb-lucene)) or ElasticSearch via
[The River](https://www.elastic.co/blog/the-river/). At eHealth Africa we use
the latter.


## Using Replication
There are two ways to start a replication: the `_replicator` database and the
`_replicate` API endpoint.
Use `_replicator` database when in doubt.

### `_replicate` API endpoint
When you use Futon you use the replicator endpoint. To initiate a replication
post a json:
```json
{
  "source": "a-db",
  "target": "another-db",
  "continuous": true
}
```
The response includes a replication id (which can also be optained from
`_active_tasks`):
```json
{
  "ok": true,
  "_local_id": "0a81b645497e6270611ec3419767a584+continuous"
}
```
Having this id you can cancel a continuous replication by posting
```json
{
  "replication_id": "0a81b645497e6270611ec3419767a584+continuous",
  "cancel": true
}
```
to the `_replicate` endpoint.


### `_replicator` Database
Replications created via the `_replicator` database are persisted and survive a
server restart. Its just a normal database which means you have the default
operations. Replications are initiated by creating replication documents:
```json
{
  "_id": "initial-replication/a-db/another-db",
  "source": "a-db",
  "target": "another-db",
  "continuous": true
}
```
Look, you can have meaningful ids! Cancelling is straight forward - just delete
the document.
Read [more about the `_replicator`
database](https://gist.github.com/fdmanana/832610).
