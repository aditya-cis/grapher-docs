---
title: Managing Links
description: Managing Links (Relationships) between Mongo Collections easily
---

## Introduction

This module is a collection relationship manager, lets you specify linking information between collections, 
gives you ability to easily manipulate these links (like add, remove), and provides an easy way to fetch them.

In addition to that, you also have the ability create *resolver* links that can make REST-API calls or link with any type of database.

When linking MongoDB collections, we support 4 different ways:

#### One

We use this when we want to reference a single Document. For example, a comment has one author. Or a post has one author.

```js
// Comment Document
{
    ...
    userId: 'XXX'
}
```

#### Many

We use this when we want to store multiple references to a collection. This can be viewed as a *Many-To-Many* SQL relationship, but not necessarily.
For example, a user may belong in multiple groups:

```js
// User Document
{
    ...
    groupIds: ['XXX', 'YYY']
}
```

#### One Meta
Meta comes from *metadata*. This is used when we want to store additional information about the link. 
For example, if a user belongs to a single group, we want to store the fact that he is an Admin or not.

This relationship can be emulated in different ways. We could also store via a *Many* relationship "adminIds" at Group Level.
There is no right or wrong. Just choose what's best fit.

```js
// User Document
{
    ...
    groupId: {
        _id: 'XXX', isAdmin: true
    }
}
```

#### Many Meta
Same as the scenario in One Meta, but in this case, a user can belong to multiple groups and he can be admin only to some.

```js
// User Document
{
    ...
    groupIds: [
        {_id: 'XXX', isAdmin: true},
        {_id: 'YYY', isAdmin: false}
    ]
}
```

To emulate that you can also have done it in Group Document:

```js
// Group Document
{
    ...
    userIds: [
        {_id: 'XXX', isAdmin: true},
        {_id: 'YYY', isAdmin: false},
    ]
}
```

Our First Link
--------------

You have a Comment that is posted by an User. 
We either store userId at Comment level, or store commentIds at User level.
The second one doesn't make much sense, so we'll explore the first one.

If we store "userId" in Comment document. We have a "One" relationship.

```js
// comment document would look like:
{
    _id: 'XXXXXXXXXXXXXXX',
    text: 'Hello Grapher!',
    userId: 'XXXXXXXXXXXXXXX'
}
```

To create this kind of link we would do:
```js
const Users = Meteor.users;
const Comments = new Mongo.Collection('comments');

Comments.addLinks({
    user: { // user represents the link name, can be any name you wish
        type: 'one', // we would have used 'many' if we wanted to store multiple user ids
        collection: Users,
        field: 'userId' // optional, if not specified it will generate a specific field.
    }
});
```

We will use the *getLink* method, which is an object that allows us to fetch, and modify the linked elements.

```js
const commentId = Comments.insert({text: 'Our first linked comment.'});
const userLink = Comments.getLink(commentId, 'user'); // we retrieve the link object for commentId and 'user'

// if you have the comment object with an _id, you can also do Comments.getLink(comment, 'user')
userLink.set(this.userId);  // will update userId
userLink.unset(); // will make userId null

userLink.fetch(); // returns the user object, if it was a multi relationship it would have returned an array
userLink.find(); // returns a Mongo.Cursor, but if you run .fetch() on it will return an array regardless of the relationship type.

// set/unset makes an update immediately into the database, should be run server side.
```

{% pullquote 'warning' %}
Removing/unsetting the link, will not affect the related document. If you want to remove it from the database, you will have to do it manually.
{% endpullquote %}

Inversed Links
--------------

All good but I may want at the user level to fetch all my comments I posted. This is where we introduce the concept of *inversed links*.
An *inversed link* basically means that the information about the link is stored on the other side. In our case, in the Comment document.

```js
Users.addLinks({
    'comments': {
        collection: Comments,
        inversedBy: 'user'
    }
});

let comments = [];
const commentsLink = Users.getLink(this.userId, 'comments');
comments = commentsLink.find().fetch()
// or
comments = commentsLink.fetch()
// or filter your query even more using find(filters, options)
comments = commentsLink.find({text: 'Our first linked comment.'}, {limit: 10}).fetch()
// or
comments = commentsLink.fetch({text: 'Our first linked comment.'}, {limit: 10})
```

{% pullquote 'warning' %}
If you use filters when fetching from a link, the filters will be applied only for the linked documents.
{% endpullquote %}

Now, let's continue our journey and assume the comment might have different tags. 
So let's use a *Many* relationship:

```js
Comments.addLinks({
    tags: {
        collection: Tags,
        type: 'many'
    }
})

const tagLink = Comments.getLink(commentId, 'tags');
tagLink.add(tagId);
tagLink.add(tagObject); // object must contain _id to be able to identify it, 

// IMPORTANT: if the object does not contain _id, the object will be created in the database for you automatically
tagLink.add({name: 'New Tag'}) // will create a tag document in Tags collection

// Also supports:
tagLink.add([tagId1, tagId2]);
tagLink.add([tagObject1, tagObject2]);

// Same arguments are supported by tagLink.remove(...)
```

Keep in mind:
For single relationships *One* and *One Meta* we use set() and unset().
For many relationships *Many* and *Many Meta* we use add() and remove().

#### Chain commands that run updates easily:

```js
tagLink.remove(oldTagId).add(newTagId);
exampleLink.set('XXX').unset();
```

Meta Links
----------

A meta relationship is very useful because you may need to store information *about the relationship*. 
Such as an user can belong to certain groups, but he is an admin only to some group
 
So instead of creating a separate collection for this, or poluting the groups document, we could use meta relationships.

```js
Users.addLinks({
    groups: {
        type: 'many'
        metadata: {} // it is enough to specify metadata as an empty object to make it clear we are dealing with a meta relation
    }
});

const groupsLink = Users.getLink(userId, 'groups');
groupsLink.add(groupId, {isAdmin: true});

// metadata getter
groupsLink.metadata(groupId) // returns {isAdmin: true, _id: groupId}}
// metadata setter
groupsLink.metadata(groupId, {isAdmin: false}) // runs the update for you automatically
```

The same principles apply to *One Meta* relationships, but you don't need to specify the _id:

```js
Users.addLinks({
    group: {type: 'one', metadata: {}}
})

const groupLink = Users.getLink(userId, 'group');
groupLink.set(groupId, {isAdmin: true});
groupLink.metadata() // returns {isAdmin: true, _id: groupId}}
groupLink.metadata({isAdmin: false}) // runs the update in the database.
```

Meta filters
------------

Given a relationship between 2 entities: Users and Groups
```js
Users.addLinks({
    groups: {
        collection: Groups,
        type: 'many',
        metadata: {
            isAdmin: { type: Boolean }
        }
    }
})
```

You can now fetch only the groups you are admin to like this:

```js
const groupsLink = Users.getLink(userId, 'groups');
groupsLink.fetch({$meta: {isAdmin: true}});
```

It also works from the inversed side as well:
```js
Groups.addLinks({
    users: {
        collection: Users,
        inversedBy: 'groups'
    }
})


const usersLink = Groups.getLink(groupId, 'users');
usersLink.fetch({$meta: {isAdmin: true}});
```

{% pullquote 'warning' %}
Meta filters also works with Query
{% endpullquote %}

Link Loopback
-------------

Reference the same collection in the link. For tree-like database structures this is great.

```js
Users.addLinks({
    children: {
        collection: Users,
        type: 'many'
    }
    parent: {
        collection: Users,
        inversedBy: 'children'
    }
})
```

```js
const childrenLink = Users.getLink(this.userId, 'children');
childrenLink.fetch() // array of ob
const parentLink = Users.getLink(this.userId, 'children');
parentLink.fetch() // single object
```

Resolver Links
--------------
```js
Users.addLinks({
    tickets: {
        resolver(user, arg1, arg2) { // first argument will be the parent, next arguments are what is passed in fetch() or find()
            const runner = Meteor.wrapAsync(HTTP.call);
            return runner("GET", "https://api.example.com/tickets", {id: user.exampleApiId});
        }
    }
});

const ticketLink = Users.getLink(this.userId, 'tickets');
ticketsLink.fetch(arg1, arg2);
ticketsLink.find(arg1, arg2); // fetch() and find() they are equivalent for Resolver Links 
```

{% pullquote 'warning' %}
You must use a sync function for this to work. Read more about [Meteor.wrapAsync](https://docs.meteor.com/api/core.html#Meteor-wrapAsync).
{% endpullquote %}

{% pullquote 'info' %}
You can also use resolver for special database queries for example, you may need to get only the messages that he has not seen yet.
{% endpullquote %}


```js
Users.addLinks({
    unreadMessages: {
        resolve(user) {
            return Messages.find({receiverId: user._id, isRead: false}).fetch();
        }
    }
});
```


SimpleSchema
-----------------------------

It is very likely that you would use SimpleSchema to ensure a data-structure for your documents, and prevent bad data to be inserted.
This library automatically detects whether you have a schema attached to your collection or not, and will add fields with proper schema definitions.

IMPORTANT! In order for this to work without problems, make sure your schema is attached before defining links. These are the appended schemas by link type.

#### One Relationships

```js
fieldName: {
    type: String,
    optional: true
}
```

#### Many Relationships

```js
fieldName: {
    type: [String], 
    optional: true
}
```

#### Meta Relationships

For meta relationships, it creates a blackbox schema if the metadata option contains no keys

Example:
```js
Users.addLinks({
    group: {
        type: 'one', 
        field: 'groupId',
        metadata: {}
     }
});
```

This will append to your schema:
```js
groupId: {
    type: Object,
    blackbox: true, 
    optional: true
}
```

Example:
```js
Users.addLinks({
    group: {
        type: 'one', 
        field: 'groupId',
        metadata: {
            isAdmin: {type: Boolean, optional: true}
        }
     }
});
```

This will append to your schema:
```js
groupId: {
    type: Object,
    blackbox: true, 
    optional: true
}
groupId.$._id: {type: String}
groupId.$.isAdmin: {type: Boolean, optional: true}
```

{% pullquote 'warning' %}
_id field is put by default.
{% endpullquote %}

If you have a many meta relationship:
```js
const metadataSchema = {
    _id: {type: String},
    isAdmin: {type: Boolean, optional: true}
}
```

Appended schema will look like:
```js
groupIds: {
    type: [metadataSchema],
    optional: true
}
```


Data Consistency
----------------

Let's say I have a "Thread" with multiple "Members". If a "Member" is deleted from the database, we don't want to keep unexisting references.
This is why if we delete a member, thread document should be cleaned.

{% pullquote 'warning' %}
This works for any kind of relationship.
{% endpullquote %}

This is achieved by using [https://atmospherejs.com/matb33/collection-hooks](https://atmospherejs.com/matb33/collection-hooks) package.
And it only works if Member contains the inversed link to Thread.

Let's see how that works'

```js
Threads.addLinks({
    'members': {
        collection: Members,
        type: 'many'
    }
});

Members.addLinks({
    'threads': {
        collection: Threads,
        inversedBy: 'members'
    }
});
```

When *Member* is removed from the database, it will look for all the threads of that member.
And it will remove it from the fieldStorage. This way your data will be *consistent* without having to deal with it.

Autoremoval
-----------
```js
Members.addLinks({
    'posts': {
        collection: Posts,
        type: 'many',
        autoremove: true
    }
});
```

Be careful with this one! 
When Member document is deleted, all posts will be deleted.

This works from direct and inversed side as well. Use with caution.

Indexing
--------

```js
Members.addLinks({
    'posts': {
        collection: Posts,
        type: 'many',
        index: true
    }
});
```


{% pullquote 'warning' %}
By using *index: true* option, it will call _ensureIndex automatically on your collection.
This will give you a performance boost when you are searching from the "related" link, in our case, from "Posts".
{% endpullquote %}


