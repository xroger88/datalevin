<p align="center"><img src="logo.png" alt="datalevin logo" height="140"></img></p>
<h1 align="center">Datalevin</h1>
<p align="center"> 🧘 Simple, fast and durable Datalog database for everyone 💽 </p>
<p align="center">
<a href="https://clojars.org/datalevin"><img src="https://img.shields.io/clojars/v/datalevin.svg?color=sucess" alt="datalevin on clojars"></img></a>
</p>

## :hear_no_evil: What and why

> I love Datalog, why hasn't everyone used this already?

Datalevin is a simple durable Datalog database.

The rationale is to have a simple, fast and free Datalog query engine running on
durable storage.  It is our observation that many developers prefer the flavor
of Datalog popularized by [Datomic®](https://www.datomic.com) over any flavor of
SQL, once they get to use it. Perhaps it is because Datalog is more declarative
and composable than SQL, e.g. the automatic implicit joins seem to be its killer
feature.

Datomic® is an enterprise grade software, and its feature set may be an overkill
for some use cases. One thing that may confuse casual users is its [temporal
features](https://docs.datomic.com/cloud/whatis/data-model.html#time-model). To
keep things simple and familiar, Datalevin does not store transaction history,
and behaves the same way as most other databases: when data are deleted, they
are gone.

Datalevin started out as a port of
[Datascript](https://github.com/tonsky/datascript) in-memory Datalog database to
[Lightning Memory-Mapped Database
(LMDB)](https://en.wikipedia.org/wiki/Lightning_Memory-Mapped_Database). It
retains the library property of Datascript, and it is meant to be embedded in
applications to manage state. Because data is persistent on disk in Datalevin,
application state can survive application restarts, and data size can be larger
than memory.

Datalevin relies on the robust ACID transactional database features of LMDB.
Designed for concurrent read intensive workloads, LMDB is used in many projects,
e.g.
[Cloudflare](https://blog.cloudflare.com/introducing-quicksilver-configuration-distribution-at-internet-scale/)
global configuration distribution. LMDB also [performs
well](http://www.lmdb.tech/bench/ondisk/) in writing large values (> 2KB).
Therefore, it is fine to store documents in Datalevin.

Datalevin uses cover index and has no write-ahead log, so once the data are
written, they are indexed. There are no separate processes or threads for
indexing, compaction or doing any database maintenance work that compete with
your applications for resources. Since Datalog is simply a more ergonomic query
language than SQL, Datalevin can serves the role of an easier-to-use and
more lightweight relational database (RDBMS), e.g. where SQLite is called for.

Independent from Datalog, Datalevin can be used as a fast key-value store for
[EDN](https://en.wikipedia.org/wiki/Extensible_Data_Notation) data, with support
for range queries, predicate filtering and more. For example, one can use this
feature in situations where something like Redis is called for.

Our goal is to simplify data storage and access by supporting diverse use cases
and paradigms, because maximal flexibility is the core strength of a Datalog
store. Datalevin may not be the fastest or the most scalable solution for one
particular use case, but it would surely support the most number of them in a
coherent and elegant manner.

Using one data store for different use cases simplifies and reduces the cost of
software development, deployment and maintenance. Therefore, we plan to
implement necessary extensions to make Datalevin also a search engine, a
production rule engine, a graph database, and a document database, since the
storage and index structure of Datalevin is already compatible with all of them.

Presentation:

* [2020 London Clojurians Meetup](https://youtu.be/-5SrIUK6k5g)

## :truck: Installation

### Clojure library

Datalevin is a Clojure library, simply add it to your project as a dependency
and start using it!

If you use [Clojure CLI](https://clojure.org/guides/deps_and_cli) and
`deps.edn`, declare the dependency on Datalevin:

```Clojure
{:deps
 {datalevin/datalevin {:mvn/version "0.4.6"}}}
```

If you use [Leiningen](https://leiningen.org/) build tool, add this to the
`:dependencies` section of your `project.clj` file:

```Clojure
[datalevin "0.4.6"]
```

### Native image and command line tool

Datalevin supports compilation into a GraalVM native image. Native Datalevin should have
better performance, for the native image version does not incur JNI
overhead and uses a comparator written in C, see [blog
post](https://yyhh.org/blog/2021/02/writing-c-code-in-javaclojure-graalvm-specific-programming/).

The release contains a command line tool called `dtlv` that is built with
Datalevin native image. It can be used to work with Datalevin database files in shell
scripting, for the purposes of database backup, data import, export,
query/transaction execution, database compaction, and so on.

## :tada: Library Usage

### Use as a Datalog store

```clojure
(require '[datalevin.core :as d])

;; Define an optional schema.
;; Note that pre-defined schema is optional, as Datalevin does schema-on-write.
;; However, attributes requiring special handling need to be defined in schema,
;; e.g. many cardinality, uniqueness constraint, reference type, and so on.
(def schema {:aka  {:db/cardinality :db.cardinality/many}
             ;; :db/valueType is optional, if unspecified, the attribute will be
             ;; treated as EDN blobs, and may not be optimal for range queries
             :name {:db/valueType :db.type/string
                    :db/unique    :db.unique/identity}})

;; Create DB on disk and connect to it
(def conn (d/get-conn "/var/datalevin/mydb" schema))

;; Transact some data
;; Notice that :nation is not defined in schema, so it will be treated as an EDN blob
(d/transact! conn
             [{:name "Frege", :db/id -1, :nation "France", :aka ["foo" "fred"]}
              {:name "Peirce", :db/id -2, :nation "france"}
              {:name "De Morgan", :db/id -3, :nation "English"}])

;; Query the data
(d/q '[:find ?nation
       :in $ ?alias
       :where
       [?e :aka ?alias]
       [?e :nation ?nation]]
     @conn
     "fred")
;; => #{["France"]}

;; Retract the name attribute of an entity
(d/transact! conn [[:db/retract 1 :name "Frege"]])

;; Pull the entity, now the name is gone
(d/q '[:find (pull ?e [*])
       :in $ ?alias
       :where
       [?e :aka ?alias]]
     @conn
     "fred")
;; => ([{:db/id 1, :aka ["foo" "fred"], :nation "France"}])

;; Close DB connection
(d/close conn)
```

### Use as a key value store
```clojure
(require '[datalevin.core :as d])
(import '[java.util Date])

;; Open a key value DB on disk and get the DB handle
(def db (d/open-kv "/var/datalevin/mykvdb"))

;; Define some table (called "dbi" in LMDB) names
(def misc-table "misc-test-table")
(def date-table "date-test-table")

;; Open the tables
(d/open-dbi db misc-table)
(d/open-dbi db date-table)

;; Transact some data, a transaction can put data into multiple tables
;; Optionally, data type can be specified to help with range query
(d/transact-kv
  db
  [[:put misc-table :datalevin "Hello, world!"]
   [:put misc-table 42 {:saying "So Long, and thanks for all the fish"
                       :source "The Hitchhiker's Guide to the Galaxy"}]
   [:put date-table #inst "1991-12-25" "USSR broke apart" :instant]
   [:put date-table #inst "1989-11-09" "The fall of the Berlin Wall" :instant]])

;; Get the value with the key
(d/get-value db misc-table :datalevin)
;; => "Hello, world!"
(d/get-value db misc-table 42)
;; => {:saying "So Long, and thanks for all the fish",
;;     :source "The Hitchhiker's Guide to the Galaxy"}

;; Delete some data
(d/transact-kv db [[:del misc-table 42]])

;; Now it's gone
(d/get-value db misc-table 42)
;; => nil

;; Range query, from unix epoch time to now
(d/get-range db date-table [:closed (Date. 0) (Date.)] :instant)
;; => [[#inst "1989-11-09T00:00:00.000-00:00" "The fall of the Berlin Wall"]
;;     [#inst "1991-12-25T00:00:00.000-00:00" "USSR broke apart"]]

;; Close key value db
(d/close-kv db)
```
### API doc

Please refer to the [API
documentation](https://juji-io.github.io/datalevin/index.html) for more details.

## :rocket: Status

Both Datascript and LMDB are mature and stable libraries. Building on top of
them, Datalevin is extensively tested with property-based testing. It is also used
in production at [Juji](https://juji.io).

Running the [benchmark suite adopted from
Datascript](https://github.com/juji-io/datalevin/tree/master/bench) on a Ubuntu
Linux server with an Intel i7 3.6GHz CPU and a 1TB SSD drive, here is how it
looks.

<p align="center">
<img src="bench/datalevin-bench-query-01-05-2021.png" alt="query benchmark" height="300"></img>
<img src="bench/datalevin-bench-write-01-05-2021.png" alt="write benchmark" height="300"></img>
</p>

In all benchmarked queries, Datalevin is faster than Datascript. Considering
that we are comparing a disk store with a memory store, this result may be
counter-intuitive. One reason is that Datalevin caches more
aggressively, whereas Datascript chose not to do so (e.g. see [this
issue](https://github.com/tonsky/datascript/issues/6)). Before we introduced
caching in version 0.2.8, Datalevin was only faster than Datascript for single
clause queries due to the highly efficient reads of LMDB. With caching enabled,
Datalevin is now faster across the board. In addition, we will soon move to a
more efficient query implementation.

Writes are slower than Datascript, as expected, as Datalevin is writing to disk
while Datascript is in memory. The bulk write speed is good, writing 100K datoms
to disk in less than 0.5 seconds; the same data can also be transacted with all
the integrity checks as a whole in less than 2 seconds. Transacting one datom or
five datoms at a time, it takes more or less than that time.

In short, Datalevin is quite capable for small or medium projects right now.
Large scale projects can be supported when distributed mode is implemented.

## :earth_americas: Roadmap

These are the tentative goals that we try to reach as soon as we can. We may
adjust the priorities based on feedback.

* 0.4.0 ~~Native image and native command line tool.~~ [Done 2021/02/27]
* 0.5.0 A new Datalog query engine with improved performance.
* 0.6.0 Datalog query parity with Datascript: composite tuples and persisted transaction functions.
* 0.7.0 Fully automatic schema migration on write.
* 0.8.0 As a product rule engine: implementing Rete/UL algorithm.
* 0.9.0 As a search engine: fuzzy fulltext search across multiple attributes.
* 0.10.0 As a graph database: implementing [loom](https://github.com/aysylu/loom) graph protocols.
* 0.11.0 As a document database: auto indexing of document fields.
* 1.0.0 Distributed mode with raft based replication.

We appreciate and welcome your contribution or suggestion. Please file issues or pull requests.

## :floppy_disk: Differences from Datascript

Datascript is developed by [Nikita Prokopov](https://tonsky.me/) that "is built
totally from scratch and is not related by any means to" Datomic®. Although
currently a port, Datalevin differs from Datascript in more significant ways
than just the difference in data durability:

* As mentioned, Datalevin is not an immutable database, and there is no
  "database as a value" feature.  Since history is not kept, transaction ids are
  not stored.

* Datoms in a transaction are committed together as a batch, rather than being
  saved by `with-datom` one at a time.

* Respects `:db/valueType`. Currently, most [Datomic® value
  types](https://docs.datomic.com/on-prem/schema.html#value-types) are
  supported, except bigint, bigdec, uri and tuple. Values of the attributes that
  are not defined in the schema or have unspecified types are treated as
  [EDN](https://en.wikipedia.org/wiki/Extensible_Data_Notation) blobs, and are
  de/serialized with [nippy](https://github.com/ptaoussanis/nippy).

* Has a value leading index (VEA) for datoms with `:db.type/ref` type attribute;
  The attribute and value leading index (AVE) is enabled for all datoms, so
  there is no need to specify `:db/index`, similar to Datomic® Cloud. Does not
  have AEV index, in order to save storage and improve write speed.

* Attributes are stored in indices as integer ids, thus attributes in index
  access are returned in attribute creation order, not in lexicographic order
  (i.e. do not expect `:b` to come after `:a`). This is the same as Datomic®.

* Has no features that are applicable only for in-memory DBs, such as DB as an
  immutable data structure, DB pretty print, etc.

This project would not have started without the existence of Datascript, we will
continue submitting pull requests to Datascript with our improvements where they
are applicable to Datascript.

## :baby: Limitations

* Attribute names have a length limitation: an attribute name cannot be more
  than 511 bytes long, due to LMDB key size limit.

* Because keys are compared bitwise, for range queries to work as expected on an
  attribute, its `:db/valueType` should be specified.

* Floating point `NaN` cannot be stored.

* The maximum individual value size is 4GB. In practice, value size is
  determined by LMDB's ability to find large enough continuous space on disk and
  Datelevin's ability to pre-allocate off-heap buffers in JVM for them.

* The total data size of a Datalevin database has the same limit as LMDB's, e.g.
  128TB on a modern 64-bit machine that implements 48-bit address spaces.

* There's no network interface as of now, but this may change.

* Currently only supports Clojure on JVM, but adding support for other
  Clojure-hosting runtime is possible, since bindings for LMDB
  exist in almost all major languages and available on most platforms. However,
  I would prefer other languages to work with GraalVM compiled native library
  when it is available, so I only need to maintain one language binding.

## :shopping: Alternatives

If you are interested in using the dialect of Datalog pioneered by Datomic®, here are your current options:

* If you need time travel and rich features backed by the authors of Clojure, you should use [Datomic®](https://www.datomic.com).

* If you need an in-memory store that has almost the same API as Datomic®, [Datascript](https://github.com/tonsky/datascript) is for you.

* If you need an in-memory graph database, [Asami](https://github.com/threatgrid/asami) is fast.

* If you need features such as bi-temporal graph queries, you may try [Crux](https://github.com/juxt/crux).

* If you need a durable store with some storage choices, you may try [Datahike](https://github.com/replikativ/datahike).

* There was also [Eva](https://github.com/Workiva/eva/), a distributed store, but it is no longer in active development.

* If you need a simple and fast durable store with a battle tested backend, give [Datalevin](https://github.com/juji-io/datalevin) a try.

Version: 0.4.6

## License

Copyright © 2020-2021 [Juji, Inc.](https://juji.io)

Licensed under Eclipse Public License (see [LICENSE](LICENSE)).
