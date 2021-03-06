POSIX `readdir(3)` streams directory entries, but, alas, Node.js' `fs.readdir()` calls back with an array.  An array is fine ... unless you've _a massive number of files in the directory_, and want to handle them one at a time.  Frankly, if you want an array, you might as well `scandir(3)`, which is exactly what libuv does.  [There is no `readdir(3)` in `fs.readdir()`.][no-readdir]

[no-readdir]: https://github.com/libuv/libuv/blob/b12624c13693c4d29ca84b3556eadc9e9c0936a4/docs/src/migration_010_100.rst#uv_fs_readdir-rename-and-api-change

This module builds a very thin native Node.js add-on wrapper around `opendir(3)`, `readdir(3)`, and `closedir(3)` for streaming directory reads.  The binding is a bit clunky, tracking the C-language API.

**Unless you're building a stream, you probably want to use a stream package that in turn uses this package.**

Behold!  A test suite:

```javascript
var Reader = require('native-readdir')
var fs = require('fs')
var path = require('path')
var assert = require('assert')
var series = require('run-series')

// Create a directory.
var directory = '/tmp/example-' + new Date().toISOString()
fs.mkdirSync(directory)

// Write some files in it.
fs.writeFileSync(path.join(directory, 'a'), 'Apple')
fs.writeFileSync(path.join(directory, 'b'), 'Banana')

// Create a readdir object.  Unfortunately, this object encapsulates
// magic:  The POSIX `DIR` pointer from `opendir(3)`.
var reader = new Reader(directory)
var entries = []

series([
  function (done) {
    // You must call `.open()`.
    reader.open(function (error) {
      assert.ifError(error)
      done()
    })
  },
  readEntry,
  readEntry,
  readEntry,
  readEntry,
  function (done) {
    assert.deepEqual(entries.sort(), ['.', '..', 'a', 'b'])
    // `.read()` returns `null` on end.
    reader.read(function (error, entry) {
      assert.ifError(error)
      assert.equal(entry, null)
    })
  },
  function (done) {
    // You must call `.close()`.
    reader.close(function (error) {
      assert.ifError(error)
    })
  }
])

function readEntry (done) {
  reader.read(function (error, entry) {
    assert.ifError(error)
    assert.equal(typeof entry, 'string')
    entries.push(entry)
    done()
  })
}
```

This package started with Fabian Canas' [node-native-boilerplate]. Fabian was kind enough to license his boilerplate under The MIT License.  I've licensed my code under The MIT License, too.

[node-native-boilerplate]: https://github.com/fcanas/node-native-boilerplate
