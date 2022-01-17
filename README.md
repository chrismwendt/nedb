<img src="http://i.imgur.com/9O1xHFb.png" style="width: 25%; height: 25%; float: left;">

## The JavaScript Database

This module is a fork of [nedb](https://github.com/louischatriot/nedb)
written by Louis Chatriot.

Since the original maintainer doesn't support this package anymore, we forked it
and maintain it for the needs of [Seald](https://www.seald.io).

**Embedded persistent or in memory database for Node.js, Electron and browsers,
100% JavaScript, no binary dependency**. API is a subset of MongoDB's and it's
[plenty fast](#speed).

## Installation

Module name on npm is [`@seald-io/nedb`](https://www.npmjs.com/package/@seald-io/nedb).

```
npm install @seald-io/nedb
```

Then to import, you just have to:

```js
const Datastore = require('@seald-io/nedb')
```

## Documentation
The API is a subset of MongoDB's API (the most used operations).

### JSDoc
You can read the markdown version of the JSDoc [in the docs directory](./API.md).
It is generated by running `npm run generateDocs:markdown`.

### Promise-based interface vs callback-based interface
Since version 3.0.0, NeDB provides a Promise-based equivalent for each function
which is suffixed with `Async`, for example `loadDatabaseAsync`.

The original callback-based interface is still available, fully retro-compatible
(as far as the test suites can tell) and are a shim to this Promise-based
version.

Don't hesitate to open an issue if it breaks something in your project.

The rest of the readme will only show the Promise-based API, the full
documentation is available in the [`docs`](./docs) directory of the repository.

### Creating/loading a database

You can use NeDB as an in-memory only datastore or as a persistent datastore.
One datastore is the equivalent of a MongoDB collection. The constructor is used
as follows [`new Datastore(options)` where `options` is an object](./API.md#new_Datastore_new).

If the Datastore is persistent (if you give it [`options.filename`](./API.md#Datastore+filename),
you'll need to load the database using [Datastore#loadDatabaseAsync](./API.md#Datastore+loadDatabaseAsync),
or using [`options.autoload`](./API.md#Datastore+autoload).

```javascript
// Type 1: In-memory only datastore (no need to load the database)
const Datastore = require('@seald-io/nedb')
const db = new Datastore()

// Type 2: Persistent datastore with manual loading
const Datastore = require('@seald-io/nedb')
const db = new Datastore({ filename: 'path/to/datafile' })
try {
  await db.loadDatabaseAsync()
} catch (error) {
  // loading has failed
}
// loading has succeeded
 
// Type 3: Persistent datastore with automatic loading
const Datastore = require('@seald-io/nedb')
const db = new Datastore({ filename: 'path/to/datafile', autoload: true }) // You can await db.autoloadPromise to catch a potential error when autoloading.
// You can issue commands right away

// Of course you can create multiple datastores if you need several
// collections. In this case it's usually a good idea to use autoload for all collections.
db = {}
db.users = new Datastore('path/to/users.db')
db.robots = new Datastore('path/to/robots.db')

// You need to load each database
await db.users.loadDatabaseAsync()
await db.robots.loadDatabaseAsync()
```

### Persistence

Under the hood, NeDB's [persistence](./docs/Persistence.md) uses an append-only
format, meaning that all updates and deletes actually result in lines added at
the end of the datafile, for performance reasons. The database is automatically
compacted (i.e. put back in the one-line-per-document format) every time you
load each database within your application.

You can manually call the compaction function
with [`yourDatabase#persistence#compactDatafileAsync`](./API.md#Persistence+compactDatafileAsync).

You can also set automatic compaction at regular intervals
with [`yourDatabase#persistence#setAutocompactionInterval`](./API.md#Persistence+setAutocompactionInterval),
and stop automatic compaction with [`yourDatabase#persistence#stopAutocompaction`](./API.md#Persistence+stopAutocompaction).

### Inserting documents

The native types are `String`, `Number`, `Boolean`, `Date` and `null`. You can
also use arrays and subdocuments (objects). If a field is `undefined`, it will
not be saved (this is different from MongoDB which transforms `undefined`
in `null`, something I find counter-intuitive).

If the document does not contain an `_id` field, NeDB will automatically
generate one for you (a 16-characters alphanumerical string). The `_id` of a
document, once set, cannot be modified.

Field names cannot begin by '$' or contain a '.'.

```javascript
const doc = {
  hello: 'world',
  n: 5,
  today: new Date(),
  nedbIsAwesome: true,
  notthere: null,
  notToBeSaved: undefined,  // Will not be saved
  fruits: ['apple', 'orange', 'pear'],
  infos: { name: '@seald-io/nedb' }
}

try {
  const newDoc = await db.insertAsync(doc)
  // newDoc is the newly inserted document, including its _id
  // newDoc has no key called notToBeSaved since its value was undefined
} catch (error) {
  // if an error happens
}
```

You can also bulk-insert an array of documents. This operation is atomic,
meaning that if one insert fails due to a unique constraint being violated, all
changes are rolled back.

```javascript
const newDocs = await db.insertAsync([{ a: 5 }, { a: 42 }])
// Two documents were inserted in the database
// newDocs is an array with these documents, augmented with their _id


// If there is a unique constraint on field 'a', this will fail
try {
  await db.insertAsync([{ a: 5 }, { a: 42 }, { a: 5 }])
} catch (error) {
  // err is a 'uniqueViolated' error
  // The database was not modified
}
```

### Finding documents

Use `findAsync` to look for multiple documents matching you query, or `findOneAsync` to
look for one specific document. You can select documents based on field equality
or use comparison operators (`$lt`, `$lte`, `$gt`, `$gte`, `$in`, `$nin`, `$ne`)
. You can also use logical operators `$or`, `$and`, `$not` and `$where`. See
below for the syntax.

You can use regular expressions in two ways: in basic querying in place of a
string, or with the `$regex` operator.

You can sort and paginate results using the cursor API (see below).

You can use standard projections to restrict the fields to appear in the
results (see below).

#### Basic querying

Basic querying means are looking for documents whose fields match the ones you
specify. You can use regular expression to match strings. You can use the dot
notation to navigate inside nested documents, arrays, arrays of subdocuments and
to match a specific element of an array.

```javascript
// Let's say our datastore contains the following collection
// { _id: 'id1', planet: 'Mars', system: 'solar', inhabited: false, satellites: ['Phobos', 'Deimos'] }
// { _id: 'id2', planet: 'Earth', system: 'solar', inhabited: true, humans: { genders: 2, eyes: true } }
// { _id: 'id3', planet: 'Jupiter', system: 'solar', inhabited: false }
// { _id: 'id4', planet: 'Omicron Persei 8', system: 'futurama', inhabited: true, humans: { genders: 7 } }
// { _id: 'id5', completeData: { planets: [ { name: 'Earth', number: 3 }, { name: 'Mars', number: 2 }, { name: 'Pluton', number: 9 } ] } }

// Finding all planets in the solar system
const docs = await db.findAsync({ system: 'solar' })
// docs is an array containing documents Mars, Earth, Jupiter
// If no document is found, docs is equal to []


// Finding all planets whose name contain the substring 'ar' using a regular expression
const docs = await db.findAsync({ planet: /ar/ })
// docs contains Mars and Earth

// Finding all inhabited planets in the solar system
const docs = await db.findAsync({ system: 'solar', inhabited: true })
// docs is an array containing document Earth only

// Use the dot-notation to match fields in subdocuments
const docs = await db.findAsync({ 'humans.genders': 2 })
// docs contains Earth


// Use the dot-notation to navigate arrays of subdocuments
const docs = await db.findAsync({ 'completeData.planets.name': 'Mars' })
// docs contains document 5

const docs = await db.findAsync({ 'completeData.planets.name': 'Jupiter' })
// docs is empty

const docs = await db.findAsync({ 'completeData.planets.0.name': 'Earth' })
// docs contains document 5
// If we had tested against 'Mars' docs would be empty because we are matching against a specific array element


// You can also deep-compare objects. Don't confuse this with dot-notation!
const docs = await db.findAsync({ humans: { genders: 2 } })
// docs is empty, because { genders: 2 } is not equal to { genders: 2, eyes: true }


// Find all documents in the collection
const docs = await db.findAsync({})

// The same rules apply when you want to only find one document
const doc = await db.findOneAsync({ _id: 'id1' })
// doc is the document Mars
// If no document is found, doc is null
```

#### Operators ($lt, $lte, $gt, $gte, $in, $nin, $ne, $stat, $regex)

The syntax is `{ field: { $op: value } }` where `$op` is any comparison
operator:

* `$lt`, `$lte`: less than, less than or equal
* `$gt`, `$gte`: greater than, greater than or equal
* `$in`: member of. `value` must be an array of values
* `$ne`, `$nin`: not equal, not a member of
* `$stat`: checks whether the document posses the property `field`. `value`
  should be true or false
* `$regex`: checks whether a string is matched by the regular expression.
  Contrary to MongoDB, the use of `$options` with `$regex` is not supported,
  because it doesn't give you more power than regex flags. Basic queries are
  more readable so only use the `$regex` operator when you need to use another
  operator with it (see example below)

```javascript
// $lt, $lte, $gt and $gte work on numbers and strings
const docs = await db.findAsync({ 'humans.genders': { $gt: 5 } })
// docs contains Omicron Persei 8, whose humans have more than 5 genders (7).

// When used with strings, lexicographical order is used
const docs = await db.findAsync({ planet: { $gt: 'Mercury' } })
// docs contains Omicron Persei 8

// Using $in. $nin is used in the same way
const docs = await db.findAsync({ planet: { $in: ['Earth', 'Jupiter'] } })
// docs contains Earth and Jupiter

// Using $stat
const docs = await db.findAsync({ satellites: { $stat: true } })
// docs contains only Mars

// Using $regex with another operator
const docs = await db.findAsync({
  planet: {
    $regex: /ar/,
    $nin: ['Jupiter', 'Earth']
  }
})
// docs only contains Mars because Earth was excluded from the match by $nin
```

#### Array fields

When a field in a document is an array, NeDB first tries to see if the query
value is an array to perform an exact match, then whether there is an
array-specific comparison function (for now there is only `$size`
and `$elemMatch`) being used. If not, the query is treated as a query on every
element and there is a match if at least one element matches.

* `$size`: match on the size of the array
* `$elemMatch`: matches if at least one array element matches the query entirely

```javascript
// Exact match
const docs = await db.findAsync({ satellites: ['Phobos', 'Deimos'] })
// docs contains Mars

const docs = await db.findAsync({ satellites: ['Deimos', 'Phobos'] })
// docs is empty

// Using an array-specific comparison function
// $elemMatch operator will provide match for a document, if an element from the array field satisfies all the conditions specified with the `$elemMatch` operator
const docs = await db.findAsync({
  completeData: {
    planets: {
      $elemMatch: {
        name: 'Earth',
        number: 3
      }
    }
  }
})
// docs contains documents with id 5 (completeData)

const docs = await db.findAsync({
  completeData: {
    planets: {
      $elemMatch: {
        name: 'Earth',
        number: 5
      }
    }
  }
})
// docs is empty

// You can use inside #elemMatch query any known document query operator
const docs = await db.findAsync({
  completeData: {
    planets: {
      $elemMatch: {
        name: 'Earth',
        number: { $gt: 2 }
      }
    }
  }
})
// docs contains documents with id 5 (completeData)

// Note: you can't use nested comparison functions, e.g. { $size: { $lt: 5 } } will throw an error
const docs = await db.findAsync({ satellites: { $size: 2 } })
// docs contains Mars

const docs = await db.findAsync({ satellites: { $size: 1 } })
// docs is empty

// If a document's field is an array, matching it means matching any element of the array
const docs = await db.findAsync({ satellites: 'Phobos' })
// docs contains Mars. Result would have been the same if query had been { satellites: 'Deimos' }

// This also works for queries that use comparison operators
const docs = await db.findAsync({ satellites: { $lt: 'Amos' } })
// docs is empty since Phobos and Deimos are after Amos in lexicographical order

// This also works with the $in and $nin operator
const docs = await db.findAsync({ satellites: { $in: ['Moon', 'Deimos'] } })
// docs contains Mars (the Earth document is not complete!)
```

#### Logical operators $or, $and, $not, $where

You can combine queries using logical operators:

* For `$or` and `$and`, the syntax is `{ $op: [query1, query2, ...] }`.
* For `$not`, the syntax is `{ $not: query }`
* For `$where`, the syntax
  is `{ $where: function () { /* object is 'this', return a boolean */ } }`

```javascript
const docs = await db.findAsync({ $or: [{ planet: 'Earth' }, { planet: 'Mars' }] })
// docs contains Earth and Mars

const docs = await db.findAsync({ $not: { planet: 'Earth' } })
// docs contains Mars, Jupiter, Omicron Persei 8

const docs = await db.findAsync({ $where: function () { return Object.keys(this) > 6 } })
// docs with more than 6 properties

// You can mix normal queries, comparison queries and logical operators
const docs = await db.findAsync({
  $or: [{ planet: 'Earth' }, { planet: 'Mars' }],
  inhabited: true
})
// docs contains Earth
```

#### Sorting and paginating

[`Datastore#findAsync`](./API.md#Datastore+findAsync),
[`Datastore#findOneAsync`](./API.md#Datastore+findOneAsync) and
[`Datastore#countAsync`](./API.md#Datastore+countAsync) don't
actually return a `Promise`, but a [`Cursor`](./docs/Cursor.md) which is a
[`Thenable`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await#thenable_objects)
which calls [`Cursor#execAsync`](./API.md#Cursor+execAsync) when awaited.

This pattern allows to chain [`Cursor#sort`](./API.md#Cursor+sort),
[`Cursor#skip`](./API.md#Cursor+skip), 
[`Cursor#limit`](./API.md#Cursor+limit) and
[`Cursor#projection`](./API.md#Cursor+projection) and await the result.

```javascript
// Let's say the database contains these 4 documents
// doc1 = { _id: 'id1', planet: 'Mars', system: 'solar', inhabited: false, satellites: ['Phobos', 'Deimos'] }
// doc2 = { _id: 'id2', planet: 'Earth', system: 'solar', inhabited: true, humans: { genders: 2, eyes: true } }
// doc3 = { _id: 'id3', planet: 'Jupiter', system: 'solar', inhabited: false }
// doc4 = { _id: 'id4', planet: 'Omicron Persei 8', system: 'futurama', inhabited: true, humans: { genders: 7 } }

// No query used means all results are returned (before the Cursor modifiers)
const docs = await db.findAsync({}).sort({ planet: 1 }).skip(1).limit(2)
// docs is [doc3, doc1]

// You can sort in reverse order like this
const docs = await db.findAsync({ system: 'solar' }).sort({ planet: -1 })
// docs is [doc1, doc3, doc2]

// You can sort on one field, then another, and so on like this:
const docs = await db.findAsync({}).sort({ firstField: 1, secondField: -1 })
// ... You understand how this works!
```

#### Projections

You can give `findAsync` and `findOneAsync` an optional second argument, `projections`.
The syntax is the same as MongoDB: `{ a: 1, b: 1 }` to return only the `a`
and `b` fields, `{ a: 0, b: 0 }` to omit these two fields. You cannot use both
modes at the time, except for `_id` which is by default always returned and
which you can choose to omit. You can project on nested documents.

```javascript
// Same database as above

// Keeping only the given fields
const docs = await db.findAsync({ planet: 'Mars' }, { planet: 1, system: 1 })
// docs is [{ planet: 'Mars', system: 'solar', _id: 'id1' }]

// Keeping only the given fields but removing _id
const docs = await db.findAsync({ planet: 'Mars' }, {
  planet: 1,
  system: 1,
  _id: 0
})
// docs is [{ planet: 'Mars', system: 'solar' }]

// Omitting only the given fields and removing _id
const docs = await db.findAsync({ planet: 'Mars' }, {
  planet: 0,
  system: 0,
  _id: 0
})
// docs is [{ inhabited: false, satellites: ['Phobos', 'Deimos'] }]

// Failure: using both modes at the same time
const docs = await db.findAsync({ planet: 'Mars' }, { planet: 0, system: 1 })
// err is the error message, docs is undefined

// You can also use it in a Cursor way but this syntax is not compatible with MongoDB
const docs = await db.findAsync({ planet: 'Mars' }).projection({
  planet: 1,
  system: 1
})
// docs is [{ planet: 'Mars', system: 'solar', _id: 'id1' }]

// Project on a nested document
const doc = await db.findOneAsync({ planet: 'Earth' }).projection({
  planet: 1,
  'humans.genders': 1
})
// doc is { planet: 'Earth', _id: 'id2', humans: { genders: 2 } }
```

### Counting documents

You can use `countAsync` to count documents. It has the same syntax as `findAsync`.
For example:

```javascript
// Count all planets in the solar system
const count = await db.countAsync({ system: 'solar' })
// count equals to 3

// Count all documents in the datastore
const count = await db.countAsync({})
// count equals to 4
```

### Updating documents

[`db.updateAsync(query, update, options)`](./API.md#Datastore+updateAsync)
will update all documents matching `query` according to the `update` rules.

`update` specifies how the documents should be modified. It is either a new 
document or a set of modifiers (you cannot use both together):
* A new document will replace the matched docs;
* Modifiers create the fields they need to modify if they don't exist,
  and you can apply them to subdocs (see [the API reference]((./API.md#Datastore+updateAsync)))

`options` is an object with three possible parameters:
* `multi` which allows the modification of several documents if set to true.
* `upsert` will insert a new document corresponding if it doesn't exist (either
the `update` is a simple object with no modifiers, or the `query` modified by
the modifiers in the `update`) if set to `true`.
* `returnUpdatedDocs` will return the array of documents matched by the find
query and updated (updated documents will be returned even if the update did not
actually modify them) if set to `true`.

It resolves into an  Object with the following properties:
- `numAffected`: how many documents were affected by the update;
- `upsert`: if a document was actually upserted (not always the same as `options.upsert`;
- `affectedDocuments`: 
  - if `upsert` is `true` the document upserted;
  - if `options.returnUpdatedDocs` is `true` either the affected document or, if `options.multi` is `true` an Array of the affected documents, else `null`;

**Note**: you can't change a document's _id.

```javascript
// Let's use the same example collection as in the 'finding document' part
// { _id: 'id1', planet: 'Mars', system: 'solar', inhabited: false }
// { _id: 'id2', planet: 'Earth', system: 'solar', inhabited: true }
// { _id: 'id3', planet: 'Jupiter', system: 'solar', inhabited: false }
// { _id: 'id4', planet: 'Omicron Persia 8', system: 'futurama', inhabited: true }

// Replace a document by another
const { numReplaced } = await db.updateAsync({ planet: 'Jupiter' }, { planet: 'Pluton' }, {})
// numReplaced = 1
// The doc #3 has been replaced by { _id: 'id3', planet: 'Pluton' }
// Note that the _id is kept unchanged, and the document has been replaced
// (the 'system' and inhabited fields are not here anymore)


// Set an existing field's value
const { numReplaced } = await db.updateAsync({ system: 'solar' }, { $set: { system: 'solar system' } }, { multi: true })
// numReplaced = 3
// Field 'system' on Mars, Earth, Jupiter now has value 'solar system'


// Setting the value of a non-existing field in a subdocument by using the dot-notation
await db.updateAsync({ planet: 'Mars' }, {
  $set: {
    'data.satellites': 2,
    'data.red': true
  }
}, {})
// Mars document now is { _id: 'id1', system: 'solar', inhabited: false
//                      , data: { satellites: 2, red: true }
//                      }
// Not that to set fields in subdocuments, you HAVE to use dot-notation
// Using object-notation will just replace the top-level field
await db.updateAsync({ planet: 'Mars' }, { $set: { data: { satellites: 3 } } }, {})
// Mars document now is { _id: 'id1', system: 'solar', inhabited: false
//                      , data: { satellites: 3 }
//                      }
// You lost the 'data.red' field which is probably not the intended behavior


// Deleting a field
await db.updateAsync({ planet: 'Mars' }, { $unset: { planet: true } }, {})
// Now the document for Mars doesn't contain the planet field
// You can unset nested fields with the dot notation of course


// Upserting a document
const { numReplaced, affectedDocuments, upsert } = await db.updateAsync({ planet: 'Pluton' }, {
  planet: 'Pluton',
  inhabited: false
}, { upsert: true })
// numReplaced = 1, affectedDocuments = { _id: 'id5', planet: 'Pluton', inhabited: false }, upsert = true
// A new document { _id: 'id5', planet: 'Pluton', inhabited: false } has been added to the collection


// If you upsert with a modifier, the upserted doc is the query modified by the modifier
// This is simpler than it sounds :)
await db.updateAsync({ planet: 'Pluton' }, { $inc: { distance: 38 } }, { upsert: true })
// A new document { _id: 'id5', planet: 'Pluton', distance: 38 } has been added to the collection  


// If we insert a new document { _id: 'id6', fruits: ['apple', 'orange', 'pear'] } in the collection,
// let's see how we can modify the array field atomically

// $push inserts new elements at the end of the array
await db.updateAsync({ _id: 'id6' }, { $push: { fruits: 'banana' } }, {})
// Now the fruits array is ['apple', 'orange', 'pear', 'banana']


// $pop removes an element from the end (if used with 1) or the front (if used with -1) of the array
await db.updateAsync({ _id: 'id6' }, { $pop: { fruits: 1 } }, {})
// Now the fruits array is ['apple', 'orange']
// With { $pop: { fruits: -1 } }, it would have been ['orange', 'pear']


// $addToSet adds an element to an array only if it isn't already in it
// Equality is deep-checked (i.e. $addToSet will not insert an object in an array already containing the same object)
// Note that it doesn't check whether the array contained duplicates before or not
await db.updateAsync({ _id: 'id6' }, { $addToSet: { fruits: 'apple' } }, {})
// The fruits array didn't change
// If we had used a fruit not in the array, e.g. 'banana', it would have been added to the array

// $pull removes all values matching a value or even any NeDB query from the array
await db.updateAsync({ _id: 'id6' }, { $pull: { fruits: 'apple' } }, {})
// Now the fruits array is ['orange', 'pear']

await db.updateAsync({ _id: 'id6' }, { $pull: { fruits: { $in: ['apple', 'pear'] } } }, {})
// Now the fruits array is ['orange']


// $each can be used to $push or $addToSet multiple values at once
// This example works the same way with $addToSet
await db.updateAsync({ _id: 'id6' }, { $push: { fruits: { $each: ['banana', 'orange'] } } }, {})
// Now the fruits array is ['apple', 'orange', 'pear', 'banana', 'orange']


// $slice can be used in cunjunction with $push and $each to limit the size of the resulting array.
// A value of 0 will update the array to an empty array. A positive value n will keep only the n first elements
// A negative value -n will keep only the last n elements.
// If $slice is specified but not $each, $each is set to []
await db.updateAsync({ _id: 'id6' }, {
  $push: {
    fruits: {
      $each: ['banana'],
      $slice: 2
    }
  }
})
// Now the fruits array is ['apple', 'orange']


// $min/$max to update only if provided value is less/greater than current value
// Let's say the database contains this document
// doc = { _id: 'id', name: 'Name', value: 5 }
await db.updateAsync({ _id: 'id1' }, { $min: { value: 2 } }, {})
// The document will be updated to { _id: 'id', name: 'Name', value: 2 }


await db.updateAsync({ _id: 'id1' }, { $min: { value: 8 } }, {})
// The document will not be modified
```

### Removing documents

[`db.removeAsync(query, options)`](./API.md#Datastore#removeAsync)
will remove documents matching `query`. Can remove multiple documents if
`options.multi` is set. Returns the `Promise<numRemoved>`.

```javascript
// Let's use the same example collection as in the "finding document" part
// { _id: 'id1', planet: 'Mars', system: 'solar', inhabited: false }
// { _id: 'id2', planet: 'Earth', system: 'solar', inhabited: true }
// { _id: 'id3', planet: 'Jupiter', system: 'solar', inhabited: false }
// { _id: 'id4', planet: 'Omicron Persia 8', system: 'futurama', inhabited: true }

// Remove one document from the collection
// options set to {} since the default for multi is false
const { numRemoved } = await db.removeAsync({ _id: 'id2' }, {})
// numRemoved = 1


// Remove multiple documents
const { numRemoved } = await db.removeAsync({ system: 'solar' }, { multi: true })
// numRemoved = 3
// All planets from the solar system were removed


// Removing all documents with the 'match-all' query
const { numRemoved } = await db.removeAsync({}, { multi: true })
```

### Indexing

NeDB supports indexing. It gives a very nice speed boost and can be used to
enforce a unique constraint on a field. You can index any field, including
fields in nested documents using the dot notation. For now, indexes are only
used to speed up basic queries and queries using `$in`, `$lt`, `$lte`, `$gt`
and `$gte`. The indexed values cannot be of type array of object.

To create an index, use [`datastore#ensureIndexAsync(options)`](./API.md#Datastore+ensureIndexAsync).
It resolves when the index is persisted on disk (if the database is persistent)
and may throw an Error (usually a unique constraint that was violated). It can
be called when you want, even after some data was inserted, though it's best to
call it at application startup. The options are:

* **fieldName** (required): name of the field to index. Use the dot notation to
  index a field in a nested document.
* **unique** (optional, defaults to `false`): enforce field uniqueness.
* **sparse** (optional, defaults to `false`): don't index documents for which
  the field is not defined.
* **expireAfterSeconds** (number of seconds, optional): if set, the created
  index is a TTL (time to live) index, that will automatically remove documents
  when the indexed field value is older than `expireAfterSeconds`.

Note: the `_id` is automatically indexed with a unique constraint.

You can remove a previously created index with
[`datastore#removeIndexAsync(fieldName)`](./API.md#Datastore+removeIndexAsync).

```javascript
try {
  await db.ensureIndexAsync({ fieldName: 'somefield' })
} catch (error) {
  // If there was an error, error is not null
}

// Using a unique constraint with the index
await db.ensureIndexAsync({ fieldName: 'somefield', unique: true })

// Using a sparse unique index
await db.ensureIndexAsync({
  fieldName: 'somefield',
  unique: true,
  sparse: true
})

try {
  // Format of the error message when the unique constraint is not met
  await db.insertAsync({ somefield: '@seald-io/nedb' })
  // works
  await db.insertAsync({ somefield: '@seald-io/nedb' })
  //rejects
} catch (error) {
  // error is { errorType: 'uniqueViolated',
  //            key: 'name',
  //            message: 'Unique constraint violated for key name' }
}


// Remove index on field somefield
await db.removeIndexAsync('somefield')

// Example of using expireAfterSeconds to remove documents 1 hour
// after their creation (db's timestampData option is true here)
await db.ensureIndex({
  fieldName: 'createdAt',
  expireAfterSeconds: 3600
})

// You can also use the option to set an expiration date like so
await db.ensureIndex({
  fieldName: 'expirationDate',
  expireAfterSeconds: 0
})
// Now all documents will expire when system time reaches the date in their
// expirationDate field
```

## Other environments
NeDB runs on Node.js (it is tested on Node 12, 14 and 16), the browser (it is
tested on the latest version of Chrome) and React-Native using
[@react-native-async-storage/async-storage](https://github.com/react-native-async-storage/async-storage).

### Browser bundle
The npm package contains a bundle and its minified counterpart for the browser.
They are located in the `browser-version/out` directory. You only need to require `nedb.js`
or `nedb.min.js` in your HTML file and the global object `Nedb` can be used
right away, with the same API as the server version:

```
<script src="nedb.min.js"></script>
<script>
  var db = new Nedb();   // Create an in-memory only datastore
  
  db.insert({ planet: 'Earth' }, function (err) {
   db.find({}, function (err, docs) {
     // docs contains the two planets Earth and Mars
   });
  });
</script>
```

If you specify a `filename`, the database will be persistent, and automatically
select the best storage method available using [localforage](https://github.com/localForage/localForage)
(IndexedDB, WebSQL or localStorage) depending on the browser. In most cases that
means a lot of data can be stored, typically in hundreds of MB.

**WARNING**: the storage system changed between
v1.3 and v1.4 and is NOT back-compatible! Your application needs to resync
client-side when you upgrade NeDB.

NeDB uses modern Javascript features such as `async`, `Promise`, `class`, `const`
, `let`, `Set`, `Map`, ... The bundle does not polyfill these features. If you
need to target another environment, you will need to make your own bundle.

### Using the `browser` and `react-native` fields
NeDB uses the `browser` and `react-native` fields to replace some modules by an
environment specific shim.

The way this works is by counting on the bundler that will package NeDB to use
this fields. This is [done by default by Webpack](https://webpack.js.org/configuration/resolve/#resolvealiasfields)
for the `browser` field. And this is [done by default by Metro](https://github.com/facebook/metro/blob/c21daba415ea26511e157f794689caab9abe8236/packages/metro/src/ModuleGraph/node-haste/Package.js#L108)
for the `react-native` field.

This is done for:
- the [storage module](./docs/storage.md) which uses Node.js `fs`. It is
  [replaced in the browser](./docs/storageBrowser.md) by one that uses
  [localforage](https://github.com/localForage/localForage), and
  [in `react-native`](./docs/storageBrowser.md)  by one that uses
  [@react-native-async-storage/async-storage](https://github.com/react-native-async-storage/async-storage)
- the [customUtils module](./docs/customUtilsNode.md) which uses Node.js
  `crypto` module. It is replaced by a good enough shim to generate ids that uses `Math.random()`.
- the [byline module](./docs/byline.md) which uses Node.js `stream`
  (a fork of [`node-byline`](https://github.com/jahewson/node-byline) included in
  the repo because it is unmaintained). It isn't used int the browser nor
  react-native versions, therefore it is shimmed with an empty object.

## Performance

### Speed

NeDB is not intended to be a replacement of large-scale databases such as
MongoDB, and as such was not designed for speed. That said, it is still pretty
fast on the expected datasets, especially if you use indexing. On a typical,
not-so-fast dev machine, for a collection containing 10,000 documents, with
indexing:

* Insert: **10,680 ops/s**
* Find: **43,290 ops/s**
* Update: **8,000 ops/s**
* Remove: **11,750 ops/s**

You can run these simple benchmarks by executing the scripts in the `benchmarks`
folder. Run them with the `--help` flag to see how they work.

### Memory footprint

A copy of the whole database is kept in memory. This is not much on the expected
kind of datasets (20MB for 10,000 2KB documents).

## Use in other services

* An ODM for NeDB: [follicle](https://github.com/seald/follicle)

## Modernization

This fork of NeDB will be incrementally updated to:
* [ ] cleanup the benchmark and update the performance statistics;
* [ ] remove deprecated features;
* [ ] add a way to change the `Storage` module by dependency injection, which will
  pave the way to cleaner browser react-native versions (cf https://github.com/seald/nedb/pull/19).
* [x] use `async` functions and `Promises` instead of callbacks with `async@0.2.6`;
* [x] expose a `Promise`-based interface;
* [x] remove the `underscore` dependency;

## Pull requests guidelines
If you submit a pull request, thanks! There are a couple rules to follow though
to make it manageable:

* The pull request should be atomic, i.e. contain only one feature. If it
  contains more, please submit multiple pull requests. Reviewing massive, 1000
  loc+ pull requests is extremely hard.
* Likewise, if for one unique feature the pull request grows too large (more
  than 200 loc tests not included), please get in touch first.
* Please stick to the current coding style. It's important that the code uses a
  coherent style for readability (this package uses [`standard`](https://standardjs.com/)).
* Do not include stylistic improvements ('housekeeping'). If you think one part
  deserves lots of housekeeping, use a separate pull request so as not to
  pollute the code.
* Don't forget tests for your new feature. Also don't forget to run the whole
  test suite before submitting to make sure you didn't introduce regressions.
* Update the JSDoc and regenerate [the markdown files](./docs).
* Update the readme accordingly.

## License

See [License](LICENSE.md)
