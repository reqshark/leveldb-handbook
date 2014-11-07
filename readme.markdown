# leveldb-handbook

LevelDB is a simple embedded database. It was originally written for Google
Chrome's IndexDB implementation and implements many ideas from Google's
BigTable database.

This document covers using leveldb in [node.js](http://nodejs.org).

# introduction

The best thing about leveldb is its modular nature.

From a small set of low-level primitives, you can find or build abstractions for
nearly any kind of problem. But instead of baking in support for common
features, leveldb takes a step back and 

trivial to extend in novel ways.




If you need the database to perform a task that hasn't been done yet, just
publish your own abstractions.

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

