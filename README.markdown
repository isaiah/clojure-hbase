# clojure-hbase

Clojure-HBase is a simple library for accessing HBase conveniently from Clojure. 

## Introduction

There are now a number of Clojure libraries for accessing HBase. This project
provides the first two.

* [Clojure-HBase](http://github.com/davidsantiago/clojure-hbase) - Provides a 
low-level wrapping of the main HBase "CRUD" Java API. While it
reflects the fundamental API conventions of HBase with little or no elaboration,
it does provide macros and functions to remove any need for the extensive Java 
interop necessary to use the main API in most cases. This library is most useful
for someone who wants to code to HBase at a low level, or as building blocks for
a custom storage solution based on HBase.
* [Clojure-HBase Admin](http://github.com/davidsantiago/clojure-hbase) - Provides
a wrapper for most of the administrative functions useful for using HBase, such as
creating, enabling, disabling, and deleting tables and column-families, as well
as setting various options. These functions are widely useful to almost everyone
using HBase.
* [Clojure-HBase Schema](http://github.com/eslick/clojure-hbase) - Ian Eslick's
HBase library provides a much higher level API for using HBase. This library
builds abstractions on HBase to provide support for table schemas and constraints, 
which make coding against HBase much nicer. 

## Usage

HBase supports four main operations: `Get`, `Put`, `Delete`, and `Scan`. The API is 
based around creating objects of the same name, and then submitting those to 
the HTable representing a given table in the database. Clojure-HBase is 
intended to provide a convenient API for creating these objects and then 
submitting them for you. Here's an example session: 

      (require ['clojure-hbase.core :as 'hb])
 
      (hb/with-table [users (hb/table "test-users")]
		     (hb/put users "testrow" :values [:account [:c1 "test" :c2 "test2"]]))
      nil

      (hb/with-table [users (hb/table "test-users")]
		     (hb/get users "testrow" :columns [:account [:c1 :c2]]))
      #<Result keyvalues={testrow/account:c1/1265871284243/Put/vlen=4, testrow/account:c2/1265871284243/Put/vlen=5}>

      (hb/with-table [users (hb/table "test-users")]
		     (hb/delete users "testrow" :columns [:account [:c1 :c2]]))
      nil

      (hb/with-table [users (hb/table "test-users")]
		     (hb/get users "testrow" :columns [:account [:c1 :c2]]))
      #<Result keyvalues=NONE>

Creating an HTable object is potentially an expensive operation in HBase, 
so HBase provides the class HTablePool, which keeps track of already created
HTables and lets them be reused. Clojure-HBase transparently uses an 
HTablePool to manage tables for you. It's not strictly necessary, but 
surrounding your calls with the `with-table` statement will ensure that any 
tables requested in the bindings are returned to the HTablePool at the end of
the code. This can be manually managed with the `table` and `release-table`
functions. Perhaps a better way to write the above code would have been:

      (hb/with-table [users (hb/table "test-users")]
         (hb/put users "testrow" :values [:account [:c1 "test" :c2 "test2"]])
         (hb/get users "testrow" :columns [:account [:c1 :c2]])
         (hb/delete users "testrow" :columns [:account [:c1 :c2]])
         (hb/get users "testrow" :columns [:account [:c1 :c2]]))
      #<Result keyvalues=NONE>

The `get`, `put`, and `delete` functions will take any number of arguments after
the table and row arguments. Options to the function are keywords followed by
0 or more arguments, depending on the function. The arguments can be pretty 
much anything that can be turned into a byte array (see below). For now, see 
the source code for the list of options.

HBase identifies data by rows, which have column-families, which contain 
columns. In general, when specifying a column, you need to give a row, a 
column-family, and a column. When operating on multiple columns within a
family, you can specify the column-family and then any number of columns 
in a vector, as above. 

It may sometimes be useful to have access to the raw HBase Get/Put/Delete
objects, perhaps for interoperability with another library. The functions
`get*`, `put*`, `scan*` and `delete*` will return those objects without submitting 
them:

      (hb/get* "testrow" :column [:account :c1]))
			#<Get row=testrow, maxVersions=1, timeRange=[0,9223372036854775807), families={(family=account, columns={c1}}>

Note that the table argument is not necessary here. Such objects can be 
submitted for processing to an HTable using the functions `query` (for `Get` and
`Scan` objects) or `modify` (for `Put` and `Delete` objects):

      (hb/with-table [users (hb/table "test-users")]
        (hb/modify users 
          (hb/put* "testrow" :value [:account :c1 "test"])))
      nil

Alternatively, you may have already-created versions of these objects from
existing code that you would then like to augment from Clojure. The functions
support a :use-existing option that lets you pass in an existing version of
the expected object and performs all its operations on that instead of 
creating a new one.

HBase only thinks of byte arrays; this includes column-family, columns, and
values. This means that any object you can serialize to a byte array can be
used for any of these. You don't have to do any of this manually (though you
can if you want, byte-arrays are perfectly acceptable arguments). In all the
code above, keywords and strings are used interchangeably, and several other
types can be used as well. You can also allow more types to be used for this 
purpose by adding your own method for the `to-bytes-impl` multimethod. Remember, 
though, HBase is basically untyped. We can make it easy to put stuff in, but
you have to remember what everything was and convert it back yourself.

`Scan` objects are a bit different from the other 3. They are created similarly,
but they will return a `ResultScanner` that lets you iterate through the scan
results. Since `ResultScanner` implements `Iterable`, you should able to use it
places where Clojure expects one (ie, `seq`). `ResultScanners` should be 
.close()'d when they are no longer needed; by using the `with-scanner` macro
you can ensure that this is done automatically.

The `Result` objects that come out of get and scan requests are not always the
most convenient to work with. If you'd prefer to deal with the result as a
set of hierarchical maps, you can use the `as-map` function to create a map out
of the result. For example: 

      (hb/with-table [users (hb/table "test-users")]
						     (hb/get users "testrow" :column [:account :c1]))
			#<Result keyvalues={testrow/account:c1/1266054048251/Put/vlen=4}>
			
can become:

      (hb/with-table [users (hb/table "test-users")]
						     (hb/as-map (hb/get users "testrow" :column [:account :c1])))
			{#<byte[] [B@54231c3> {#<byte[] [B@3cd0fbe7> {1266054048251 #<byte[] [B@3c4a19e2>}}}
			
The Clojure function `get-in` can be very useful for pulling what you want out
of this structure. We can do even better, using the options to `as-map`, which 
let you specify a function to map onto each family, qualifier, timestamp, or 
value as you wish.

      (hb/with-table [users (hb/table "test-users")]
						     (hb/as-map (hb/get users "testrow" :column [:account :c1]) :map-family #(keyword (Bytes/toString %)) :map-qualifier #(keyword (Bytes/toString %)) :map-timestamp #(java.util.Date. %) :map-value #(Bytes/toString %) str))
			{:account {:c1 {#<Date Sat Feb 13 03:40:48 CST 2010> "test"}}}

Depending on your use case, you may prefer to have all of the values in the 
`Result` as a series of [family qualifier timestamp value] vectors. The function
`as-vector` accepts the same arguments and returns a vector, each value of which
is a vector of the form just mentioned: 

      (hb/with-table [users (hb/table "test-users")]
			           (hb/as-vector (hb/get users "testrow" :column [:account :c1]) :map-family #(keyword (Bytes/toString %)) :map-qualifier #(keyword (Bytes/toString %)) :map-timestamp #(java.util.Date. %) :map-value #(Bytes/toString %) str))
      [[:account :c1 #<Date Sat Feb 13 03:40:48 CST 2010> "test"]]

To connect to remote zookeeper quorum:

    (let [conf (HBaseConfiguration/create)]
      (.set conf "hbase.zookeeper.quorum" "remote-host")
      (hb/set-htable-pool! (HTablePool. conf 10)))

## Status

Basic unit tests passing. No known bugs. Bug reports and input welcome.

## Lately...

- Update API to HBase 0.90 series API.
- Update to Clojure 1.2.
- Reorganized namespaces: com.davidsantiago.clojure-hbase -> clojure-hbase.core, etc.
- Removed dozens of reflective calls.
- Now creates the HTablePool on first access, instead of on load.

- Added some utility functions for converting hbase output back into usable objects.
- Added multimethods to to-bytes so that lists, maps, and vector can be easily inserted
  into and converted back from hbase.
- Added the :map-default keyword option for as-map, latest-as-map, and as-vector. Makes it
  easier to give those options; the more specific keywords override the default.
- Added unit tests for new utility functions.

Added basic unit tests.
Added a first cut at most of the Admin functions.

## License

Eclipse Public License
