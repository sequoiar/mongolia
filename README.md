    ooo        ooooo                                            oooo   o8o
    `88.       .888'                                            `888   `"'
     888b     d'888   .ooooo.  ooo. .oo.    .oooooooo  .ooooo.   888  oooo   .oooo.
     8 Y88. .P  888  d88' `88b `888P"Y88b  888' `88b  d88' `88b  888  `888  `P  )88b
     8  `888'   888  888   888  888   888  888   888  888   888  888   888   .oP"888
     8    Y     888  888   888  888   888  `88bod8P'  888   888  888   888  d8(  888
    o8o        o888o `Y8bod8P' o888o o888o `8oooooo.  `Y8bod8P' o888o o888o `Y888""8o
                                           d"     YD
                                           "Y88888P'


Flexible wrapper for the nodejs MongoDB driver.
Its not a ORM, but it can be used to handle the logic of your models.
No magic, no pain.

## Install

    $ npm install mongolia

Mongolia contains two independent modules:

  * Model: An object representing a collection with some wrappers/hooks of MongoDB calls.
  * Validator: An object that validates MongoDB documents and returns errors if found.

# Model

Each model has a colection name and a reference to the database.

Models don't map data from MongoDB, they are just a layer to centralize all the logic.

    var Model = require('model');

    var User = function (db) {
      var user = Model(db, 'users');

      // implement some user logic

      return user;
    };

Calls to the database are done using the function mongoCall.

    var User = User(db);
    User.mongoCall('function', args, callback);

All the collection.js functions from the driver are supported.
If you need more information visit the official driver http://github.com/christkv/node-mongodb-native.

## Custom MongoDB functions

  * findArray: find + toArray.
  * mapReduceArray: mapReduce that returns an array with the results.
  * mapReduceCursor: mapReduce that returns a cursor.

## Wrapped functions with hooks

  * insert: triggers onCreate and afterCreate hooks.
  * update: triggers onUpdate and afterUpdate hooks.
  * findAndModify: triggers onUpdate and afterUpdate hooks.

Example:

    var Comment = function (db) {
      var comment = Model(db, 'comments');

      comment.onCreate = function (element) {
        element.created_at = new Date();
      };

      comment.atferCreate = function (element) {
        var post = require('./models/post')(this.db);
        post.mongoCall('update', {_id: element.post_id}, {'$inc': {num_posts: 1}});
      };

      return comment;
    };

## Working with embedded documents

Mongolia helps you to denormalizing your MongoDB database.
You can define _skeletons_ for your embedded documents.
Those _skeletons_ define which data do you want to denormalize.

### setEmbedObject
Returns a denormalized and filtered embed object.

    setEmbedObject(name, object);

Example:

    var Post = function (db) {
      var post = Model(db, 'posts');

      // only embed the comment's title
      post.skeletons= {
        comment: ['_id', 'title']
      };

      return post;
    };

    var comment = {'_id': 1, title: 'foo', body: 'Lorem ipsum'}
    Post(db).setEmbedObject('comment', comment) // => {'_id': 1, title: 'foo'};

### updateEmbedObject
Updates an embed object. Plays really well with the update hooks.

    updateEmbedObject(model, data, name, options, callback);

Example:

    var Post = function (db) {
      var post = Model(db, 'posts');

      post.afterUpdate = function (element, update) {
        Author(db).updateEmbedObject(element, update, 'post', null, function (error, doc) {
          // Author.post updated!
        });
      };

      return post;
    };

### pushEmbedObject
Pushes an embed object. Plays really well with the insert hooks.

    pushEmbedObject(model, data, name, options, callback);

Example:

    var Post = function (db) {
      var post = Model(db, 'posts');

      post.afterInsert = function (element) {
        Author(db).pushEmbedObject(element.author, update, 'posts', null, function (error, doc) {
          // Author.posts[] now contains this post
        });
      };

      return post;
    }


## Create and update instances

Mongolia provides with two methods that allow you to create and update using the Validator.
Its important to notice that createInstance and updateInstance are asynchronous methods as some validations may require database call.

    createInstance(element, callback);
    updateInstance(element, update, callback);

The callback returns _error_, and a _validator_ object.
Those two methods trigger onCreateInstance/onUpdateInstance, asynchronously.

In order to validate an insertion/update, the model have to implement a _validate_ function.

    validate(element, update, callback);

Example:

    var Post = function (db) {
      var post = Model(db, 'posts');

      post.validate = function (element, update, callback) {
        var validator = require('./validator');

        validator.validateRegex({
          title: [validator.regex.title, 'Incorrect title'],
          body: [/.{4,200}/, 'Incorrect body'],
        });

        if (!update.body === 'Lorem ipsum') {
          validator.addError('body', 'You can be a little bit more creative');
        }
        callback(null, validator);
      }

      return post;
    };

    var post = {title: 'This is a post', body: 'Lorem ipsum'};
    var Post = Post(db);

    Post.createInstance(post, function (error, validator) {
      if (validator.hasErrors()) {
        res.render('/posts/new', {
          locals: {
            title: 'New Post',
            post: validator.updated_model,
            validator: validator
          }
        });
      } else {
        res.redirect('/posts/' + validator.updated_model._id.toHexString());
      }
    });

# Validator

Returns true if the validator is handling an updateInstance operation.

    isUpdating()

Returns true if the validator is handling an createInstance operation.

    isInserting()

Returns true if the attributed changed

    attrChanged(attr)

Adds an error to your validator. Accept dot notation to add nested errors.

    addError(field, value)

Returns true if the attributed failed a validation. Accept dot notation to check nested errors.

    hasError(field)

Returns true if any attributed failed a validation

    hasErrors()

It fills your validator with errors if any of the elements are empty

    validateExistence(validations)

It fills your validator with errors if any of the elements fail the regex

    validateRegex(validations)

It fills your validator with errors if any of the elements fail the confirmation (good for passwords)

    validateConfirmation(validations)

It fills your validator with errors if any of the queries fail (good to avoid duplicated data)

    validateQuery(validations, callback)

Example using some of the validator features:

    var User = function (db) {
      var user = Model(db, 'users');

      user.validate = function (user, data, callback) {
        var validator = $.model('validator', [user, data]);

        validator.validateRegex({
          name: [validator.regex.username, 'Incorrect name'],
          email: [validator.regex.email, 'Incorrect email'],
          password: [validator.regex.password, 'Incorrect password'],
          description: [validator.regex.description, 'Incorrect description']
        });

        if (validator.attrChanged('password')) {
          validator.validateConfirmation({
            'password': ['password_confirmation', 'Passwords must match']
          });
        }

        if (!data.tags || data.tags.length <= 0) {
          validator.addError('tags', 'Select at least one tag');
        }

        if (validator.isUpdating()) {
          validator.validateQuery({
            name: [this, {name: data.name}, false, 'There is already a user with this name'],
            email: [this, {email: data.email}, false, 'There is already a user with this email']
          }, function () {
            callback(null, validator);
          });
        } else {
          callback(null, validator);
        }
      }

      return user;
    };
