cottz-publish
=============================

Edit your documents before sending without too much stress

## Installation

```sh
$ meteor add cottz:publish
```

provides a number of methods to easily manipulate data using internally observe and observeChanges in the server

## Quick Start
Assuming we have the following collections
```js
// Authors
{
  _id: 'someAuthorId',
  name: 'Luis',
  profile: 'someProfileId',
  bio: 'I am a very good and happy author',
  interests: ['writing', 'reading', *others*]
}

// Reviews
{
  _id: 'someReviewId',
  authorId: 'someAuthorId',
  book: 'meteor for pros',
  text: 'this book is not better than mine'
}

// Books
{
  _id: 'someBookId',
  authorId: 'someAuthorId',
  name: 'meteor for dummies'
}

// Comments
{
  _id: 'someCommentId',
  bookId: 'someBookId',
  text: 'This book is better than meteor for pros :O'
}
```
I want publish the autor with his books
```js
Meteor.publish('author', function (authorId) {
  Publish.relations(this, Authors.find(authorId), function (id, doc) {
    this.cursor(Books.find({authorId: id})).publish();
  });
  
  return this.ready();
});
```
and comments of the books
```js
Meteor.publish('author', function (authorId) {
  Publish.relations(this, Authors.find(authorId), function (id, doc) {
    this.cursor(Books.find({authorId: id})).publish(function (id, doc) {
      this.cursor(Comments.find({bookId: id})).publish();
    });
  });
  
  return this.ready();
});
```
I also want to bring the profile of the author but within the author not apart
```js
Meteor.publish('author', function (authorId) {
  Publish.relations(this, Authors.find(authorId), function (id, doc) {
    this.cursor(Books.find({authorId: id})).publish(function (id, doc) {
      this.cursor(Comments.find({bookId: id})).publish();
    });
    
    doc.profile = this.cursor(Profiles.find(doc.profile))
    .changeParentDoc(function (profileId, profile) {
      return profile;
    });
  });
  
  return this.ready();
});
```
I want to include the reviews of the author within this
```js
Meteor.publish('author', function (authorId) {
  Publish.relations(this, Authors.find(authorId), function (id, doc) {
    this.cursor(Books.find({authorId: id})).publish(function (id, doc) {
      this.cursor(Comments.find({bookId: id})).publish();
    });
    
    doc.profile = this.cursor(Profiles.find(doc.profile))
    .changeParentDoc(function (profileId, profile) {
      return profile;
    });
    
    doc.reviews = this.cursor(Reviews.find({authorId: id}))
    .group(function (doc, index) {
      return doc;
    }, 'reviews');
  });
  
  return this.ready();
});
// doc.reviews = [{data}, {data}]
```
To finish I want to show only some interests of the author
```js
Meteor.publish('author', function (authorId) {
  Publish.relations(this, Authors.find(authorId), function (id, doc) {
    this.cursor(Books.find({authorId: id})).publish(function (id, doc) {
      this.cursor(Comments.find({bookId: id})).publish();
    });
    
    doc.profile = this.cursor(Profiles.find(doc.profile))
    .changeParentDoc(function (profileId, profile) {
      return profile;
    });
    
    doc.reviews = this.cursor(Reviews.find({authorId: id}))
    .group(function (doc, index) {
      return doc;
    }, 'reviews');
    
    doc.interests = this.paginate({interests: doc.interests}, 5);
  });
  
  return this.ready();
});
// doc.reviews = [{data}, {data}]

// Client
// skip 5 interest and show the next 5
Meteor.call('changePagination', 'authorId', 'interests', 5);
```
## Publish.cursor (cursor, sub, collectionName)
publishes a cursor, `collectionName` is not required

## Publish.observe / Publish.observeChanges (cursor, callbacks, sub)
observe or observe changes in a cursor without sending anything to the client. callbacks are the same as those   used by meteor
* Note: sub will only be used to stop observers, if you not send it you have to stop manually

### Publish.relations (sub, options, callback)
* You can edit the document directly (doc.property = 'some') or send it in the return.
* **sub** is the this of the publication
* **options** is an object like this: `{cursor: Authors.find(), name: 'authors'}` or just a cursor and the name will be the default name of the cursor
* **callback** receives 3 parameters: (`id`, `doc`, `changed`)

## Publish.relations methods
after starting a Publish.relations you can use the methods in `this` within the `callback`

### this.cursor (cursor, collection)
allow you to use cursor methods in the `cursor`
* `collection` is the collection where the cursor will be sent. if not sent, is the default cursor collection name

### this.paginate (field, limit, infinite)
page within an array without re run the publication or callback
* returns the paginated array, be sure to change it in the document
* **field** is an object where the key is the field in the document and the value an array
* **limit** the total number of values in the array to show
* **infinite** if true the above values are not removed when the paging is increased
* **Meteor.call('changePagination', _id, field, skip)** change the pagination of the document with that `id` and `field`. skip is the number of values to skip

## Cursor methods
after creating an instance of `this.cursor` can use the following methods

### publish (callback (id, doc, changed))
publishes a cursor
* If you send a callback you can use relations methods again and you can edit the document directly (doc.property = 'some') or send it in the return.

### observe / observeChanges (callbacks)
observe or observe changes in the cursor without sending anything to the client. callbacks are the same as those used by meteor

### changeParentDoc (callbacks, onRemoved)
designed to change something in the document with the return of the `callbacks`.
* **callbacks** is an object with `added``changed``removed` or a function that executes when it is added and changed
* **onRemoved** is a function that executes when is removed. is not used if callbacks is an object

### group (callback, field, options)
returns an array of elements with all documents in the cursor. When there is a change it will update the element change in the resulting array and send it back to the document
* **callback** receive (`doc`, `atIndex`) when is added and (`doc`, `atIndex`, `oldDoc`) when is changed
* **field** is the field in the main document that has the array
* **options (not required)** is an object like this: `{sort: array, sortField: '_id'}` implements changes based on the position within the `sort`. `sort` is an array of values and `sortField` is the field of the document where they are, by default is the _id

## Important
* all cursors returns an object with the stop() method except for changeParentDoc, group and paginate
* all cursors are stopped when the publication stop
* when the parent cursor is stopped or a document with cursors is removed all related cursors are stopped
* `group` use observe with absolute position information (addedAt, changedAt, removedAt), if possible try using only `publish` to maintain performance
* if when the callback is re-executes not called again some method (within an If for example), the method continues to run normally, if you re-call method (because the query is now different) the previous method is replaced with the new

### Note: do not forget to use this.ready() to finish writing the publication, not included by default.

## Limitations
I keep all cursors by name and if within the same instance is called at the same cursor will stop the first and second replace him
```js
Meteor.publish('movie', function (directorId) {
  Publish.relations(this, Movies.find({director: directorId}), function (id, doc) {
    this.cursor(Meteor.users.find({_id: directorId})).publish(); // this cursor will not receive updates
    this.cursor(Meteor.users.find({_id: doc.producerId})).publish();
  });
  
  return this.ready();
});
```
you can avoid this problem in the following two ways
```js
Meteor.publish('movie', function (directorId) {
  Publish.relations(this, Movies.find({director: directorId}), function (id, doc) {
    this.cursor(Meteor.users.find({_id: directorId}).publish(function () {
      this.cursor(Meteor.users.find({_id: doc.producerId})).publish();
    });
  });
  
  return this.ready();
});
```
this is much better than the first way
```js
Meteor.publish('movie', function (directorId) {
  Publish.relations(this, Movies.find({director: directorId}), function (id, doc) {
    this.cursor(Meteor.users.find({_id: directorId}), 'users').publish(); // users is the default of Meteor.users
    this.cursor(Meteor.users.find({_id: doc.producerId}), 'producers').publish();
  });
  
  return this.ready();
});
```
