# Hoodie Worker

This document explains how to write a Hoodie Worker.


## What is a Hoodie Worker?

A hoodie Worker is a server-side module that implemets a feature that client-side code can’t implement. A good example is an email-delivery worker. To deliver an email, one needs to send data over SMTP, which is a TCP connection. Client side code doesn’t (usually) have the facilities to do so.

To keep things simple, we’ll create a *Logger* worker. It reads log messages from objects and writes them into a file.

A worker communicates with the frontend over [*state machine documents*](TODO LINK).

See the [Hoodie Architecture](TODO LINK) for more details.


## Getting Started

The Hoodie reference implementation uses Node.js and CouchDB. While the Hoodie Specification is implementation agnostic, these are the tools we chose to start because the helped us to get going quick.

As more language implementations for workers come up, expect language-specific documentation. Until then, we’ll show you how we write Hoodie Workers in Node.js.

This documentation assumes you want to develop your Hoodie Worker locally and only later plan on deploying it. See the [Hoodie Worker Deployment Documentation](TODO LINK) for details.

The only dependency is Node.js. Here are a few ways to install it:

Mac OS X:

    $ brew install node

[TODO: linux, windows]

Let’s make a new project directory. We are using the `worker-` prefix to make things easier to read in git.

    $ mkdir worker-log
    $ cd worker-log

We start by creating an `index.js` file. This will be the main entry point into the worker’s code. When starting out, we just put all the code we need in here, and when it gets a bit unwieldy, we move things out into other files and modules.

Copy over this template code, we’ll explain it in detail in a minute:

    module.exports = WorkerLog;
    function WorkerLog()
    {
		console.log("Logger started.");
    }

    var log = new WorkerLog();

That’s all! You can now start your worker:

    $ node index.js
    Logger Started
    $

Yay!

Well, this isn’t terribly exciting, but we’ve got the foundation going.

## The Changes Listener

You notice that the worker exits to the shell after it has output our `console.log()` message. A real worker is meant to sit in the background and react to messages sent to it.

The way this works in Hoodie is via CouchDB documents and a CouchDB feature called the *changes feed*. A CouchDB document is just a JSON object, but persistent. Documents are stored in databases. Each database has a changes feed that you can subscribe to. It sends you near-real-time notifications about what happens in the database.

An object and a document are really the same thing. We call it object when we are in the context of JavaScript and we call it document when we are in the context of CouchDB.

What we are going to do is subscribe to a database’s changes feed and listen for the addition of documents of the type `log`. We then read these documents and compile a log message from its contents and write this to a log file. So far so simple.

Luckily, the mechanics of listening to a database’s changes feed are abstracted away in the handy node module `CouchDB Changes`. To install it, run:

    $ npm install CouchDBChanges

Add this to the top of your `index.js` file:

    var CouchDBChanges = require("CouchDBChanges");

To start listening, we first need to create a *callback function* that is called for every new change that is sent to us.

Our callback for now, will only log the changes object to the command line:

    WorkerLog.prototype._changesCallback = function(error, message)
    {
        console.log(message);
    }

See *Helper Methods* below for an explanation of why we use the `prototype` syntax.

The callback will get passed two arguments, we ignore the error for now. See *Error Handling* below for more details.

Now we can set up the changes listening:

    var changes = new CouchDBChanges("http://127.0.0.1:5984/");
    changes.follow("mydatabase", this._changesCallback, {}, {
        include_docs: true}
    );

Before we can test our new changes listener, we need to start up a CouchDB instance. Make sure its location matches what we put in the code above. We assume that it runs on your local machine on the default port, and that we are using a database called `mydatabase` to test things. See [*CouchDB Setup Options*](TBD) for alternative ways to use CouchDB.

Now we can start the worker again:

    $ node index.js
    Logger started.

And we see that it does not return to the command line, but keeps “hanging there”. That’s what we want!

TBD: errors, wrong CouchDB url, database doesn't exist, etc.

If we now create a new document in the CouchDB database `mydatabase`, we should get that document logged to the command line. Open a new terminal window and type this:

    $ curl -X POST http://127.0.0.1:5984/mydatabase/ -d '{"message":"hello world"}' -H "Content-Type: application/json"

You should see something like:

    {"ok":true,"id":"e72c9af9291eae530b28a3f15d00094d","rev":"1-baa23d8189d19e166f6e0393e23b1085"}

If you switch back to the terminal where your worker is running, you should see:

    { seq: 1,
      id: 'e72c9af9291eae530b28a3f15d00094d',
      changes: [ { rev: '1-baa23d8189d19e166f6e0393e23b1085' } ],
      doc: 
       { _id: 'e72c9af9291eae530b28a3f15d00094d',
         _rev: '1-baa23d8189d19e166f6e0393e23b1085',
         message: 'hello world' } }

To stop your worker, just hit `ctrl-c`:

    ^C$


## Helper Methods

Helper methods are methods that we need during the operation of our worker, but that are not exposed to the outside of the module.

The `_changesCallback` method above is one such helper method.

// TBD expand

## Testing

## Configuring Workers

## Workers Callbacks

## Error Handling

## Organising Code

## Using Modules

## Serving Multiple Databases