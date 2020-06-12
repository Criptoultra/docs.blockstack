---
layout: learn
permalink: /:collection/:path.html
---
# Create and use models
{:.no_toc}

Radiks allows you to model your client data. You can then query this data and display it for a user in multi-player applications.  A social application where users want to see the comments of other users is an example of a multi-player application. This page explains how to create a model in your distributed application using Radiks.

* TOC
{:toc}

## Overview of Model class extension

Blockstack provides a `Model` class you should extend to easily create, save, and fetch models. To create a model class, import the `Model` class from `radiks` into your application.

```javascript
import { Model, User } from 'radiks';
```

 Then, create a class that extends this model, and provide a schema. Refer to <a href="https://github.com/blockstack/radiks/blob/master/src/model.ts" target="_blank">the <code>Model</code> class</a> in the `radiks` repo to get an overview of the class functionality.

Your new class must define a static `className` property. This  property is used when storing and querying information. If you fail to add a `className`, Radiks defaults to the actual model's class name (`foobar.ts`) and your application will behave unpredictably.

The example class code extends `Model` to create a class named `Todo`:

```javascript
import { Model, User } from 'radiks';

class Todo extends Model {
  static className = 'Todo';
  static schema = { // all fields are encrypted by default
    title: String,
    completed: Boolean,
  }
};

// after authentication:
const todo = new Todo({ title: 'Use Radiks in an app' });
await todo.save();
todo.update({
  completed: true,
});
await todo.save();

const incompleteTodos = await Todo.fetchOwnList({ // fetch todos that this user created
  completed: false
});
console.log(incompleteTodos.length); // 0
```

## How to create your own Model

The following sections guide you through the steps in defining your own class.

### Define a class schema

Every class must have a static `schema` property which defines the attributes of a model using field/value pairs, for example:

```javascript
class Todo extends Model {
  static className = 'Todo';
  static schema = { // all fields are encrypted by default
    title: String,
    completed: Boolean,
  }
};
```

The  `key` in this object is the field name and the value, for example,  `String`, `Boolean`, or `Number`.  In this case, the `title` is a `String` field.  Alternatively, you can pass  options instead of a type.  

To define options, pass an object, with a mandatory `type` field. The only supported option right now is `decrypted`. This defaults to `false`, meaning the field is encrypted before the data is stored publicly. If you specify `true`, then the field is not encrypted. 

Storing unencrypted fields is useful if you want to be able to query the field when fetching data. A good use-case for storing decrypted fields is to store a `foreignId` that references a different model, for a "belongs-to" type of relationship.

**Never add the `decrypted` option to fields that contain sensitive user data.** Blockstack data is stored in a decentralized Gaia storage and anyone can read the user's data. That's why encrypting it is so important. If you want to filter sensitive data, then you should do it on the client-side, after decrypting it. 

### Include defaults

You may want to include an optional `defaults` static property for some field values. For example, in the class below, the `likesDogs` field is a `Boolean`, and the default is `true`.

```javascript
import { Model } from 'radiks';

class Person extends Model {
  static className = 'Person';

  static schema = {
    name: String,
    age: Number,
    isHuman: Boolean,
    likesDogs: {
      type: Boolean,
      decrypted: true // all users will know if this record likes dogs!
    }
  }

  static defaults = {
    likesDogs: true
  }
}
```

If you wanted to add a default for `isHuman`, you would simply add it to the `defaults` as well. Separate each field with a comma.

### Extend the User model

Radiks also supplies <a href="https://github.com/blockstack/radiks/blob/master/src/models/user.ts" target="_blank">a default <code>User</code> model</a>. You can also extend this model to add your own attributes. 

```javascript
import { User } from 'radiks';

// For example I want to add a public name on my user model
class MyAppUserModel extends User {
  static schema = {
    ...User.schema,
    displayName: {
      type: String,
      decrypted: true,
    },
  };
}
```

The default `User` model defines a `username`, but you can add a `displayName` to allow the user to set unique name in your app.

## Use a model you have defined

In this section, you learn how to use a model you have defined.

### About the _id attribute

All model instances have an `_id` attribute. An `_id` is used as a primary key when storing data and is used for fetching a model. Radiks also creates a `createdAt` and `updatedAt` property when creating and saving models.

If, when constructing a model's instance, you don't pass an `_id`, Radiks creates an `_id` for you automatically. This automatically created id uses the  [`uuid/v4`](https://github.com/kelektiv/node-uuid) format. This automatic `_id` is returned by the constructor.


### Construct a model instance

To create an instance, pass some attributes to the constructor of that class:

```javascript
const person = new Person({
  name: 'Hank',
  isHuman: false,
  likesDogs: false // just an example, I love dogs!
})
```

### Fetch an instance

To fetch an existing instance of an instance, you need the instance's  `id` property. Then, call the `findById()` method or the `fetch()` method, which returns a promise.

```javascript
const person = await Person.findById('404eab3a-6ddc-4ba6-afe8-1c3fff464d44');
```


After calling these methods, Radiks automatically decrypts all encrypted fields.

### Access attributes

Other than `id`, all attributes are stored in an `attrs` property on the instance.

```javascript
const { name, likesDogs } = person.attrs;
console.log(`Does ${name} like dogs?`, likesDogs);
```

### Update attributes

To quickly update multiple attributes of an instance, pass those attributes to the `update` method.

```javascript
const newAttributes = {
  likesDogs: false,
  age: 30
}
person.update(newAttributes)
```

Important, calling `update` does **not** save the instance.

### Save changes

To save an instance to Gaia and MongoDB, call the `save()` method, which returns a promise. This method encrypts all attributes that do not have the `decrypted` option in their schema. Then, it saves a JSON representation of the model in Gaia, as well as in the MongoDB.

```javascript
await person.save();
```

### Delete an instance

To delete an instance, just call the `destroy` method on it.

```javascript
await person.destroy();
```

## Query a model

To fetch multiple records that match a certain query, use the class's `fetchList()` function. This method creates an HTTP query to Radiks-server, which then queries the underlying database. Radiks-server uses the `query-to-mongo` package to turn an HTTP query into a MongoDB query. 

Here are some examples:

```javascript
const dogHaters = await Person.fetchList({ likesDogs: false });
```

Or, imagine a `Task` model with a `name`, a boolean for `completed`, and an `order` attribute.

```javascript
class Task extends Model {
  static className = 'Task';

  static schema = {
    name: String,
    completed: {
      type: Boolean,
      decrypted: true,
    },
    order: {
      type: Number,
      decrypted: true,
    }
  }
}

const tasks = await Task.fetchList({
  completed: false,
  sort: '-order'
})
```

You can read the [`query-to-mongo`](https://github.com/pbatey/query-to-mongo) package documentation to learn how to do complex querying, sorting, limiting, and so forth.

## Count models

You can also get a model's `count` record directly.

```javascript
const dogHaters = await Person.count({ likesDogs: false });
// dogHaters is the count number
```

## Fetch models created by the current user

Use the `fetchOwnList` method to find instances that were created by the current user. By using this method, you can preserve privacy, because Radiks uses a `signingKey` that only the current user knows.

```javascript
const tasks = await Task.fetchOwnList({
  completed: false
});
```

## Manage relational data

It is common for applications to have multiple different models, where some reference another. For example, imagine a task-tracking application where a user has multiple projects, and each project has multiple tasks. Here's what those models might look like:

```javascript
class Project extends Model {
  static className = 'Project';
  static schema = { name: String }
}

class Task extends Model {
  static className = 'Task';
  static schema = {
    name: String,
    projectId: {
      type: String,
      decrypted: true,
    }
    completed: Boolean
  }
}
```

Whenever you save a task, you should save a reference to the project it's in:

```javascript
const task = new Task({
  name: 'Improve radiks documentation',
  projectId: project._id
})
await task.save();
```

Then, later you'll want to fetch all tasks for a certain project:

```javascript
const tasks = await Task.fetchList({
  projectId: project._id,
})
```

Radiks lets you define an `afterFetch` method. Use this method to automatically fetch child records when you fetch the parent instance.

```javascript
class Project extends Model {
  static className = 'Project';
  static schema = { name: String }

  async afterFetch() {
    this.tasks = await Task.fetchList({
      projectId: this.id,
    })
  }
}

const project = await Project.findById('some-id-here');
console.log(project.tasks); // will already have fetched and decrypted all related tasks
```
