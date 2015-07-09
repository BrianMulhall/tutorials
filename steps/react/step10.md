{{#template name="react-step10"}}

# Security with methods

Before this step, any user of the app could edit any part of the database. This might be okay for very small internal apps or demos, but any real application needs to control permissions for its data. In Meteor, the best way to do this is by declaring _methods_. Instead of the client code directly calling `insert`, `update`, and `remove`, it will instead call methods that will check if the user is authorized to complete the action and then make any changes to the database on the client's behalf.

### Removing `insecure`

Every newly created Meteor project has the `insecure` package added by default. This is the package that allows us to edit the database from the client. It's useful when prototyping, but now we are taking off the training wheels. To remove this package, go to your app directory and run:

```bash
meteor remove insecure
```

If you try to use the app after removing this package, you will notice that none of the inputs or buttons work anymore. This is because all client-side database permissions have been revoked. Now we need to rewrite some parts of our app to use methods.

### Defining methods

First, we need to define some methods. We need one method for each database operation we want to perform on the client. Methods should be defined in code that is executed on the client and the server - we will discuss this a bit later in the section titled _Latency compensation_.

```js
// At the bottom of simple-todos.js, outside of the client-only block
Meteor.methods({
  addTask(text) {
    // Make sure the user is logged in before inserting a task
    if (! Meteor.userId()) {
      throw new Meteor.Error("not-authorized");
    }

    Tasks.insert({
      text: text,
      createdAt: new Date(),
      owner: Meteor.userId(),
      username: Meteor.user().username
    });
  },
  deleteTask(taskId) {
    Tasks.remove(taskId);
  },
  setChecked(taskId, setChecked) {
    Tasks.update(taskId, { $set: { checked: setChecked} });
  }
});
```

Now that we have defined our methods, we need to update the places we were operating on the collection to use the methods instead:

```js
// replace Tasks.insert( ... ) with:
Meteor.call("addTask", text);

// replace Tasks.update( ... ) with:
Meteor.call("setChecked", this._id, ! this.checked);

// replace Tasks.remove( ... ) with:
Meteor.call("deleteTask", this._id);
```

Now all of our inputs and buttons will start working again. What did we gain from all of this work?

1. When we insert tasks into the database, we can now securely verify that the user is logged in, that the `createdAt` field is correct, and that the `owner` and `username` fields are correct and the user isn't impersonating anyone.
2. We can add extra validation logic to `setChecked` and `deleteTask` in later steps when users can make tasks private.
3. Our client code is now more separated from our database logic. Instead of a lot of stuff happening inside our event handlers, we now have methods that can be called from anywhere.

### Optimistic UI

So why do we want to define our methods on the client and on the server? We do this to enable a feature we call _optimistic UI_.

When you call a method on the client using `Meteor.call`, two things happen in parallel:

1. The client sends a request to the server to run the method in a secure environment, just like an AJAX request would work
2. A simulation of the method runs directly on the client to attempt to predict the outcome of the server call using the available information

What this means is that a newly created task actually appears on the screen _before_ the result comes back from the server.

If the result from the server comes back and is consistent with the simulation on the client, everything remains as is. If the result on the server is different from the result of the simulation on the client, the UI is patched to reflect the actual state of the server.

With Meteor methods and optimistic UI, you get the best of both worlds &mdash; the security of server code and no round-trip delay. Read more in our [blog post about optimistic UI](http://info.meteor.com/blog/optimistic-ui-with-meteor-latency-compensation).
{{/template}}