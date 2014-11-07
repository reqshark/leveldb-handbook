# leveldb-handbook

LevelDB is a simple embedded database. It was originally written for Google
Chrome's IndexDB implementation and implements many ideas from Google's
BigTable database.

This document covers using leveldb in [node.js](http://nodejs.org).

# introduction

The best thing about using leveldb in node is the modular nature of the core
library and the [ecosystem of abstractions](https://npmjs.org)
that have been built up around that core interface.

From a small set of low-level primitives, you can find or build abstractions for
nearly any kind of problem. This is very different from almost every other
popular database, where the core database server ships with many features that
handle common use-cases. Those convenient built-in features can be good for
normalizing quality, but they make it harder for an ecosystem to get off the
ground.

Running a big, well-known database service has an ineffable heaviness to it:
they bind to a custom port and the service must already be running before your
program will work. Worse, if you install the database from a package manager
like apt-get, the database will be auto-daemonized and put into your init
scripts. Good luck hunting down where it decided to put the configuration file.

If you write a program that requires an SQL or MongoDB or
Couch or whatever database, you've got to tell other people who will want to run
your program that "oh by the way you've got to set up a database and here the
place for you to put all your configuration for the host, port, username,
password, database name, whatever." And then you'll invariably want to run
multiple applications on the same database because setting up a database in the
first place is such a production, so you'll need a scheme for partitioning data
so that applications don't interfere. Some popular databases have built-in
features for that and some don't, but there is a kind of tension just the
same.

LevelDB is different. You just do:

```
npm install level
```

and then in your program, you can just:

```
var level = require('level');
var db = level('./beep.db');
```

and now `db` is ready to go hosting a database out of `./beep.db`.

Because we installed `level` with npm, we can write programs that add `level` as
a dependency and then `level` will be installed automatically, without the user
even caring or particularly noticing that leveldb is being used behind the
scenes. Now many of the dependency-management techniques of modules that apply
to npm packages can be applied to database abstractions!

There are hundreds of packages you can use on top of leveldb to do all kinds of
things. This document covers a few of them in later chapters.

# get started

First install [node.js](http://nodejs.org).

Now you can install the [level](https://npmjs.org/package/level)
package from [npm](https://npmjs.org):

``` js
npm install level
```

# core methods

## db.get

## db.put

## db.del

## db.batch

## db.createReadStream

# key comparisons

* opts.lt / opts.start
* opts.lte / opts.le
* opts.gt / opts.end
* opts.gte / opts.ge

* opts.reverse

# thinking lexicographically

# encodings

## keyEncoding

### bytewise

## valueEncoding

### json

# sublevel

## wrapping encodings

since sublevel 6.3.0, you can wrap a `db` object with your own encodings:

``` js
var newdb = sublevel(db, {
    keyEncoding: require('bytewise'),
    valueEncoding: 'json'
});
```

This is very handy from a module usability perspective because you can expose an
interface where people can provide a `db` to your package without needing to set
up the key and value encodings themselves. This also means that you can properly
encapsulate the encodings so you will be more free to change the encodings
internally later without requiring consumers of your module to update their
code.

# live streaming

## level-live-stream

## level-live-feed

# nested streams

Sometimes you'll want to pair up streams together, similar to a join in
relational databases.

For example, to get a list of users with the latest 5 comments included inline:

``` js
var through = require('through2');
var opts = { gt: 'user!', lt: 'user!~' };
db.createReadStream(opts).pipe(through(function (row, enc, next) {
  var outer = this;
  var iopts = { gt: 'comment!' + row.key + '!', lt: 'comment!' + row.key + '!~', limit: 5 };
  row.comments = [];
  db.createReadStream(iopts).pipe(through(write, end));
  function write (irow, ienc, inext) {
    row.comments.push(irow.value.message);
    inext();
  }
  function end () {
    outer.push(row);
    next();
  }
}));
```

or another approach that works well is to just tack on a function to your object
stream of results so that you can lazily consume the sub-streams elsewhere:

``` js
var through = require('through2');
var merge = require('xtend');
var opts = { gt: 'user!', lt: 'user!~' };
db.createReadStream(opts).pipe(through(function (row, enc, next) {
  row.comments = function (opts) {
    return db.createReadStream(merge(opts, {
      gt: 'comment!' + row.key + '!',
      lt: 'comment!' + row.key + '!~'
   }));
  };
  this.push(row);
  next();
}));
```

then outside as you consume each row you can call `row.comments({ limit: 5 })`
to get a stream of comments for each user lazily as you need it.

# bespoke databases

The best part about leveldb is its modular nature. Instead of

