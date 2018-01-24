---
layout: post
title: "Building a dApp with the Ethereum blockchain and Angular 5 - part 2: the service"
---

Last week, we covered [how we could write smart contracts using
Truffle](http://huberisation.eu/2018/01/18/making-app-angular-ethereum/). Today,
we will see how we can deploy and interact with this smart contract inside an
Angular application.

## Setting up the project

For those not familiar with it, Angular is a front-end, component-based
framework written in Typescript (a superset of Javascript developed by
Microsoft). As such, it is made to be executed directly into the browser of the
user. The *components* refer to classes that control a piece of the display;
they are linked to a *template* (an HTML file) and some style files (css or scss).
The idea behind components is to make them as atomic and re-usable as possible,
in order not to write the same code twice. They can be assisted by other classes,
like services, models...

If you are not familiar with Angular and want to learn more, I can only
recommend you to look at the [Angular documentation](https://angular.io/docs),
which is filled with example and detailed explanations of how the framework
works.

To develop our Angular application, we will rely on the
[Angular CLI](cli.angular.io), that eases the project creation and build.

You can install it using npm:

```
$ npm install -g @angular/cli
```

Once the CLI is installed, create the new project:

```
$ ng new gtmb-app
$ cd gtmb-app
```

Now, launch the development webserver and navigate to `127.0.0.1:4200`

```
$ ng serve
```

You should see following page:

![New Angular project](https://raw.githubusercontent.com/Searev/searev.github.io/master/_assets/2018-01-24-building-app-angular-ethereum-part-2/angular-new-app.png)

## Importing the Truffle project

We could interact with some Ethereum network by simply using the official API,
[web3js](https://github.com/ethereum/wiki/wiki/JavaScript-API). But since
we've started using the abstraction provided by Truffle in the test we wrote
last time, let's continue using it. It will allow us to work with Promises
instead of having to deal with callbacks every time we interact with our
contract.

The abstraction layer of Truffle can simply be installed in our project using:

```
$ npm install --save truffle-contract
```

This being done, we can simply copy the content of our
[Truffle project](https://github.com/searev-experiments/gtmb-smart-contract) at
the root folder of our application (without the `node_modules` folder, obviously).

## Designing the application structure

Your project folder should now look like this:

```
|-- build/
|  |-- contracts/
|    |-- GiveTheMoneyBack.json
|    |-- Migrations.json
|-- contracts/
|  |-- GiveTheMoneyBack.sol
|  |-- Migrations.sol
|-- e2e
|-- migrations/
|  |-- 1_initial_migration.js
|-- node_modules/
|  |-- // dependencies
|-- src/
|  |-- app/
|     |-- app.component.css
|     |-- app.component.html
|     |-- app.component.spec.ts
|     |-- app.component.ts
|     |-- app.module.ts
|  |-- assets/
|  |-- environments/
|     |-- environment.prod.ts
|     |-- environment.ts
|  |-- favicon.ico
|  |-- index.html
|  |-- main.ts
|  |-- styles.css
|  |-- ...
|-- tests/
|  |-- give_the_money_back.js
|-- .angular-cli.json
|-- package.json
|-- ...
```

The classes of our application will be located in the `src/app` folder. As said,
an Angular application is divided into different reusable *components*, that can
be grouped into *modules*. We can, according to the
[Angular Style Guide](https://angular.io/guide/styleguide), divide our applications
in three types of modules:
* the *Shared* module contains elements that can be re-used in many parts of
  the application; for instance model classes.
* the *Core* module contains the classes that should only be created once, like
  singleton services.
* the *Features* modules contains components that match a specific... feature,
  and that will not be reused in other parts of the application.

Here, we will create a `CoreModule` to contain the services that will interact
with the blockchain, and a `GiveTheMoneyBackModule` to contain the component of
our application.

We will also need some modules to define the routes of our application. A good
practice is to have a routing class for each feature module, and import them
in a global routing module of the application.

In terms of components, we will need:
* a component to search existing requests
* a component to sum up a request, that will be used in the search view
* a component to create the requests
* a component to see the details of a request, back it up, pay it...
* a global component to display the whole view (the AppComponent, already
  generated)

We will also need a class to specify the properties of our refund requests. As
it will only be used in the feature module, we will simply put it inside it.

Finally, we will need a service to publish our contract, and interact with
already deployed ones.

We can generate them using the Angular CLI:

```
ng generate module app-routing --flat
ng generate module give-the-money-back
ng generate module give-the-money-back/give-the-money-back-routing --flat
ng generate component give-the-money-back/list
ng generate component give-the-money-back/thumbnail
ng generate component give-the-money-back/detail
ng generate component give-the-money-back/create
ng generate class give-the-money-back/request
ng generate module core
ng generate service core/gtmb
```

The `--flat` option for our routing modules tells the angular cli that the
module should not be created in a new directory.

Let's specify right away the content of our newly created `Request` object:

``` Typescript
// src/app/give-the-money-back/request.ts

export class Request {
    public owner: string;
    public receiver: string;
    public amount: number;
    public description: string;
    public backers: string[];
    public address: string;
    public nbBackers: number;
}
```

You can see the object basically has the same attributes as the smart contract
we created: we only added the `address` attribute to define the address of the
contract.

## Writing the service

Now that our application is set up, we can connect it to our development
blockchain and try to interact with our contract. This will be made in the
service we generated previously, so it can be available through all our application.

### Imports and attributes

Open the `src/app/core/gtmb.service.ts` file. Firstly, we need to tell our
service what smart contract it should interact with, and that it should use
the abstraction provided by Truffle to do so. We will also need to use
`web3js` to define the provider interface for our development blockchain.

Before the service class definition, add the following imports to the file:

``` Typescript
import * as Web3 from 'web3';
import * as contract from 'truffle-contract';
import * as GiveTheMoneyBack from '../../../build/contracts/GiveTheMoneyBack.json';
```

We will need to store two elements in our service: the contract abstraction, and
the account to use to interact with the blockchain. Declare them at the beginning
of our service, and define them in the constructor.

``` Typescript
@Injectable()
export class GtmbService {
  private contract;
  private account;

  constructor() {
    const web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:9545'));
    this.account = web3.eth.accounts[0];
    this.contract = contract(GiveTheMoneyBack);
    this.contract.setProvider(web3.currentProvider);
  }
}
```

As you can see, we temporarily specify a `web3` object to connect to our
development blockchain, and we use it to specify the account to use. Of course,
this only works if we have a development blockchain running on our own computer;
we will need to change this latter.

Create the getter for the `account` attribute; we will need it later in our
components.

``` Typescript
  public getAccount(): string {
    return this.account;
  }
```

### Deploying the contract

Now, let's the fun part begin. We are going to write the functions that will
actually perform the calls to interact with the blockchain. First, the contract
creation function:

``` Typescript
public createContract(request: Request): Promise<Request> {

  return this.contract
    .new(request.receiver, request.amount, request.description,
        { from: this.account, gas: 4712388 })
    .then (
      instance => {
        request.address = instance.address;
        return request;
      });

}
```

What's happening there ?
* We specify that our function needs, as input, a request, to be able to fetch
  the receiver, amount and description.
* We also specify that our function will return an `Promise` which should contain
  a `Request`.
* In the `this.contract.new()` function, we specify the parameters our smart
  contract needs. In the brackets, we specify with which account it should be
  deployed with, and the amount of gas to use. Usually, the amount of gas
  should be automatically set at the contract deployment; but to prevent any
  problem, we set our own to the default amount used by our development blockchain.
* In the callback of the `Promise` (I use the ES6 [Arrow function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions) to
  define mine; it is shorter and it does not have its own `this` binding, which
  can save a lot of headache), we set the address of our newly created contract,
  and return the completed `Request` object.

Easy enough, right ? Then, let's have a look at the `getContract()` function,
which will allow us to fetch a `Request` from an existing address.

### Fetching an existing contract

``` Typescript
public getContract(address: string): Promise<Request> {

  // Creates the request we will send in the Observable.
  const request = new Request();

  // Allow us to store the instance of the contract and use it the various callbacks.
  let meta;

  return this.contract.at(address)
  .then ( instance    => { meta = instance; return meta.owner.call(); })
  .then ( owner       => { request.owner = owner; return meta.receiver.call(); })
  .then ( receiver    => { request.receiver = receiver; return meta.amount.call(); })
  .then ( amount      => { request.amount = amount.toNumber(); return meta.description.call(); })
  .then ( description => { request.description = description; return meta.nbBackers.call(); })
  .then ( nbBackers   => { request.nbBackers = nbBackers.toNumber(); return request; });

}
```

That's a lot of callbacks! Indeed, we cannot (yet ?) make a single call to fetch
all the attributes of a contract. Therefore, we need to make one for each attribute
we want to fetch; the first one being made to store the instance of the contract
so we can use it in all our other callbacks.

### Fetching an array of addresses

However, this does not allow us to fetch the backers of a contract. This is
because we cannot fetch an array of addresses; we need to iterate through
all the array with calls for each index. Hence the need for a `nbBackers`
attribute in our contract. As we may not need it every time we need to display
a contract, I decided to split it into a different function. Moreover, it is
a bit more challenging to write, see for yourself:

``` Typescript
public getRequestBackers(address: string): Promise<string[]> {

  let meta;

  return this.contract.at(address)
  .then (instance => { meta = instance; return meta.nbBackers.call()})
  .then (nbBackers => {
    const promises = [];
    for (let i = 0; i < nbBackers; i++) {
      promises.push(meta.backers.call(i).then(backer =>  backer));
    }

    return Promise.all(promises).then(values => values);
  });

}
```

The first callback is similar to those we made in the previous function, but
the second one is more complex. Indeed, we need to iterate through the table,
hence the `for` loop. But we need to keep in mind that all we can do is
*asynchronous* calls. Therefore, we cannot simply push the backers addresses
inside an array and return it after the loop. What we need to do is to store the
*Promises* inside the array, and then use the `Promise.all()` function, which
resolves the Promise when *all* the Promises inside the array have resolved.

Also, don't be puzzled by the `backer =>  backer` part; it is just short for
`backer => { return backer; }`. Same for `values`.

### Making transactions

Now, we just need to write two more functions: those which will allow us to
back up a request and pay it. Here they are.

``` Typescript
public backUp(address: string): Promise<string> {
  let meta;

  return this.contract.at(address)
  .then(instance => { meta = instance; return instance.backUp.call({from: this.account}); })
  .then(result => {
    if (result) {
      return meta.backUp({from: this.account});
    } else {
      throw 'Cannot back up the request';
    }
  })
  .then(result => {
    if (result.logs[0].event === 'Backed') {
      return this.account;
    } else {
      throw 'Could not back up the request';
    }
  });
}
```

**Pay one's debts**
```Typescript
public payDebt(address: string): Promise<any> {
  let meta;
  let amountToPay;

  return this.contract.at(address).then(function(instance) {
    meta = instance;
    // First, fetch the amount of wei to send
    return instance.amount.call();
  }).then(amount => {
    amountToPay = amount;
    return meta.payDebt().call({from: this.account, value: amount});
  }).then(result => {
    if (result.logs[0].event === 'MoneyGivenBack') {
      return meta.payDebt({from: this.account, value: amountToPay});
    } else {
      throw 'Cannot pay the debt';
    }
  });
}
```

There is not much to explain about those functions. At first, we make a `call()`
to check that it is possible to execute the function. If it is, we make the
transaction, and check if the expected *Event* is triggered in the logs, just
like we did in our Mocha tests last week.

## Conclusion

In this part, we have seen:
* How we could quickly set up an Angular application using the *Angular CLI*
* How to connect the application to an Ethereum blockchain
* How to deploy and interact with an existing contract using the *Truffle*
  abstraction and *Promises*.

Next time, we will see how we can use the service we wrote inside our
components, to make it more visual.

As usual, the code for the application is available on
[Github](https://github.com/searev-experiments/gtmb-app)

## Sources for this part
* the [Angular CLI documentation](https://github.com/angular/angular-cli/wiki)
* the [Truffle-contract documentation](https://github.com/trufflesuite/truffle-contract)
* the [Truffle Angular Box](https://github.com/Quintor/angular-truffle-box/tree/master/src/app)
  which is an example of Angular-Truffle interaction
