---
title: "Quick Start"
description: "One page summary of how to start a new Pioche project."
lead: "One page summary of how to start a new Pioche project."
date: 2022-05-26T13:59:39+01:00
lastmod: 2022-05-26T13:59:39+01:00
draft: false
images: []
menu:
  docs:
    parent: "tutorial"
weight: 110
toc: true
---

## Introduction to Workers

Read [Intro to Workers]({{< relref "intro-to-workers" >}}) for an introduction to the workers ecosystem.

## Install the Workers CLI

First, ensure you have [node.js](https://nodejs.org/en/) and [npm](https://www.npmjs.com/get-npm) installed. We recommend using a node version manager
like [nvm](https://github.com/nvm-sh/nvm) to make changing versions and upgrading easier.  

Now install Cloudflare Wrangler (the Workers CLI tool) globally

```sh
$ npm install -g wrangler
```

## Initialize a New Project

We can use [pioche-scripts]({{< relref "create" >}}) to create a fully set up pioche project in a single command using the npm package runner [npx](https://docs.npmjs.com/cli/v7/commands/npx). This will create a directory called YourAppName in the current working directory.

```sh
$ npx pioche-scripts create YourAppName
$ cd YourAppName
```

After creating the project, you will be prompted to login to Cloudflare, if you don't have an account, you can make one at this step or just skip for now (`ctrl+c` to cancel). To login later, just run

```sh
$ wrangler login
```

## Running the Basic Project

If you've logged into wrangler, we can deploy our project to Cloudflare using

```sh
$ npm run deploy
```

Then the app will be available at `yourappname.workers.dev` where you can see the `Hello, World!` response. (If your Cloudflare account has a default zone configured, the Worker will be available at `yourappname.zone.workers.dev`)  

If you have not logged into Wrangler, you can still run the app locally using

```sh
$ npm run serve
```

## Pioche App Structure

There are only two restrictions on file structure for a Pioche project:
1. Everything must be located in `src/`
2. You cannot create a file called `entry.ts`

A Pioche project consists of:
 > [__Handlers__]({{< relref "handlers" >}}): Like middleware in other frameworks, adds functionality on the project or route level to be executed before (preHandler) or after (postHandler) the mapped controller handler.

 > [__Controllers__]({{< relref "controllers" >}}): Classes that define mapped handlers using mapping decorators.

 > [__Views__]({{< relref "views" >}}): Objects or classes defining structure for data  

We also have our [pioche.config.js]({{< relref "pioche.config" >}}) file at the project root. Here, we can define app level functionality such as preHandlers, postHandlers, external controllers (from packages), and kv_namespaces.

Each incoming request is first parsed into `Session` and `OutboundResponse` objects. If the incoming request matches a mapped handler, these objects will then be passed around as follows:  
1. App level preHandlers (defined in `pioche.config.js` preHandlers section)
2. Route level preHandlers (defined using `@UseBefore()`)
3. Mapped Controller handler
4. Route level postHandlers (defined using `@UseAfter()`)
5. App level postHandlers (defined in `pioche.config.js` postHandlers section)

## Adding Functionality

The default pioche app has a single file at `src/controllers/helloworld.ts` (copied below). This contains a [controller]({{< relref "controllers" >}}) class which adds a single endpoint to your workers application that will give a `Hello, World!` response at the base domain.  

``` js
import { BaseMap, GetMap, WorkerController, OutboundResponse, Session } from "pioche";

@BaseMap("")
export class HelloWorldController extends WorkerController {

    @GetMap("")
    async helloWorld(session: Session, res: OutboundResponse){
        res.body = "Hello, World!";
    }
}
```

### Environments

See [controllers]({{< relref "controllers" >}}) for full details

Three environments are currently supported by extending our Controller class.

```js
export class Controller1 extends WorkerController{}
export class Controller2 extends DurableObjectController{}
export class Controller3 extends WebsocketController{}
```

Worker controllers execute their code on the worker, Durable Object controllers and websocket controllers execute their code on a durable object defined by the controller class, and websocket controllers have extra features included to make working with websockets easier. If we are making calls to Durable Object storage, it is both faster and cheaper to run the associated code on the Durable Object iself. With Pioche, this is possible by just changing the environment of the controller.

### Routing

See [routing]({{< relref "routing" >}}) for full details

Let's say you want to add a new mapping to the app to say hello to anyone. The routing system uses [path-to-regexp](https://github.com/pillarjs/path-to-regexp) behind the scenes so we can just add a path parameter as follows to a new handler within our controller

``` js
...
    @GetMap("/:name")
    async helloName(session: Session, res: OutboundResponse){
        res.body = `Hello, ${session.request.params.name}!`;
    }
...
```

After deploying or running locally you can navigate to `/Johnny` to see `Hello, Johnny!`.  

All path-to-regexp features are supported so check their docs to implement further flexibility. `@BaseMap("/hello")` with `@PostMap("/:name")` is accessible with a POST request to `/hello/Johnny` this is a simple concatenation of the two routes. A request with a trailing `/` can match a route without a trailing `/` but not vice versa. All HTTP verbs are supported as well as `@AnyMap(<route>)` to match any method.  

Durable Objects have a special routing property allowing you specify the name, hex id, or `DurableObjectId` given the request as shown below.

``` js
import { BaseMap, GetMap, WorkerController, OutboundResponse, Session, DOTarget } from "pioche";

function targeter(session: Session, res: OutboundResponse, targetNS: DurableObjectNamespace): DOTarget{
    // We could also specify a hex 'idstring' or a DurableObjectID 'id'
    return {name: session.request.params.name}
}

// Since our targeter uses params.name, it should be part of the BaseMap
@BaseMap("/:name")
// Now tell our requests go to a durable object with name = params.name
@TargetDO(targeter)
export class HelloWorldController extends WorkerController {

    @GetMap("")
    async helloWorld(session: Session, res: OutboundResponse){
        res.body = `Hello, ${session.request.params.name}!`;
    }
}
```

### Storage

See [storage]({{< relref "storage" >}}) for full details.

Pioche wraps the native storage APIs to unify the Durable Object and KV storage syntaxes and allow object-like use of remote storage. Using this new API we can use chaining (optional chaining not supported) to access and write to our storages as shown below

``` js
import { BaseMap, GetMap, WorkerController, OutboundResponse, Session, }from "pioche";

@BaseMap("/:name")
export class HelloWorldController extends WorkerController {

    @GetMap("/config")
    async getUserConfig(session: Session, res: OutboundResponse){
        // Asynchronously retrieve the user's config using chaining
        res.body = this.storage[session.request.params.name].get();
    }

    @GetMap("/regdate")
    async getUserAge(session: Session, res: OutboundResponse){
        // Asynchronously retrieve the user's config using chaining
        res.body = this.storage[session.request.params.name].regdate.get();
    }

    @PutMap("/config")
    async putUserConfig(session: Session, res: OutboundResponse){
        if(await this.storage.get(session.request.params.name))
            this.storage[session.request.params.name] = session.request.json();
        res.body = "Updated user config if exists";
    }

    @PostMap("/config")
    async addUserConfig(session: Session, res: OutboundResponse){
        this.storage[session.request.params.name] = {regdate: new Date()};
        res.body = "Added new user config";
    }
}
```

## Views

See [views]({{< relref "views" >}}) for full details.

We can define intended structures for our data using the view and checks component of Pioche. A view defines the structure of data, a check validates a piece of data. We could define a view as follows to validate usernames, emails, and age

```js
class UserView{
  // Checks for 5-20 alphanumeric characters
  username = and(lenbt(5, 20), rx(/^[a-zA-Z0-9]+$/));
  // Checks for email regex
  email = rx(/^[^@]+@[^@]+\.[^@]+$/);
  // Not required, but if provided must be >13
  age = optional(gt(13));
}
```

Then we can apply that view to some incoming data. The View function returns an instance of the passed object/class with the checks replaced with the found values. The returned object also has:
* `getStatus()`: Returns 0 if keys were missing or failing checks and 1 if everything passed
* `getMissing()`: Returns a list of missing keys or paths to missing keys, i.e. if `CVV` was missing from a sub-object `creditCard` it will return `[["creditCard", "CVV"]]`
* `getFailing()`: Returns a list of keys or paths to keys which failed their respective checks.

```js
...
    @PostMap("/register/:name")
    async register({request}, res){
        if(View(request.json(), UserView).getStatus() === 0){
            this.storage[request.params.name] = request.json();
            res.body = "Successfully registered";
        } else {
            res.body = "Registration failed"
        }
    }
...
```


{{< alert icon="✔️" text="That's it, happy hacking!" />}}