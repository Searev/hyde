---
layout: post
title: "Building a DApp with the Ethereum blockchain and Angular 5 - part 1: the Smart Contract"
---

2017 has been a rich year for blockchain-based applications. Literally. Bitcoin
went from about $1000 to almost $15000 before plummeting this January. The price
of Ethereum was multiplied by 250, thanks to the craze of ICOs (Initial Coin
Offerings), that allowed start-ups and frauds to raise about $1.3 billion in
just the first half of the year. Some companies even saw their stock increase
just by putting "blockchain" in their name.

But behind the hype and the greed, no real big public application has yet been
developed. This might be because of the very nature of the blockchain
(registries are, by definition, kept in back-office, and must be transparent to
the end users), or simply because this kind of development has not yet been
democratized as is web development today.

In this new series of articles, we'll try together to create a small

blockchain-based application. I will not go into detail about what a blockchain
is or how it works. Plenty has been written on the matter, and I can only
recommend you to have a look at:
* [Bitcoin: A Peer to-Peer Electronic Cash System](https://bitcoin.org/bitcoin.pdf):
the original article that started Bitcoin
* [The Ethereum White Paper](https://github.com/ethereum/wiki/wiki/White-Paper),
that laid the foundation of Ethereum, decentralized applications, autonomous
organizations, and more
* [The Beginner's guide to Blockchain Technology](https://www.coindesk.com/information/)
from coindesk

For this tutorial, I have chosen to work with the
[Ethereum](https://www.ethereum.org) blockchain. Unlike Bitcoin, Ethereum is
made for developing decentralized, or distributed applications (known as
[DApps](https://www.stateofthedapps.com)), and not just for trading tokens.
It also comes with a [Javascript API](https://github.com/ethereum/wiki/wiki/JavaScript-API)
that should ease our integration of the blockchain.

Let's get to it!

## Designing the application

The application we are going to build is a small refund app. Its principle was
inspired to me by some of the controversies we had in France in the past year:
important political figures were found to have misused public money, and lots of
people asked them to repay their expenses. It even became a meme called "Rend
l'argent" (Give the money back).

We are therefore going to build an app that will allow someone to ask some other
person to give an amount of money back. If he or she agrees with the request,
the money will be equally dispatched between all the people who will have
supported it. In the blockchain universe, this will take the form of a
"smart-contract": a piece of code distributed on the blockchain that anybody can
check and execute.

Here is a glimpse of what the application could look like once finished.

![Mockup](https://raw.githubusercontent.com/Searev/searev.github.io/master/_assets/2018-01-19-making-app-angular-ethereum/mockup.jpg)

## Developing the Ethereum application

Now that we know what we aim to develop, let's start by building the Smart
Contract that will help running our application. To do so, we first need to set
up a proper development environment.

### Setting up the development environment

To develop applications on Ethereum, the first thing you should have is a
Wallet. Wallets are applications that interact with the blockchain, allowing you
to write, deploy new smart contracts, and perform operations on existing ones. I
use the [Ethereum wallet (Mist)](https://github.com/ethereum/mist/releases), but
feel free to use any other one.

Developing right into the Wallet is not something to recommend, as those
applications cannot really help us debug our contracts, or follow best practices.
The second thing you will need is therefore
[Truffle](http://truffleframework.com/): a framework that will help us write,
build and test our contracts. You can simply install it with npm:
```
$ npm install -g truffle
```

Finally, you will need a proper editor. I use
[VS Code](https://code.visualstudio.com/) with a [plugin to support Solidity Language](https://github.com/juanfranblanco/vscode-solidity/).
[Solidity](https://solidity.readthedocs.io/en/develop/) is the language in which
smart contracts can be written; it looks a lot like Javascript.

### Setting up the project

Let's create a new project using truffle. Create a new folder called `gtmb`
(for Give the money back), and initialize the Truffle project.

```
$ mkdir gtmb
$ cd gtmb
$ truffle init
```

Your folder should now look something like this
```
|-- contracts/
|  |-- Migrations.sol
|-- migrations/
|  |-- 1_initial_migration.js
|-- test/
|-- truffle-config.js
|-- truffle.js
```

As you can imagine, your contracts will be located in the `contracts` folder.
The `migrations` one is only here to ease the deployment of our contracts; `test`
will contain our test (!) written using [Mocha](https://mochajs.org/). The
`truffle.js` and `truffle-config.js` files are here for configuration. If you
are on Windows, delete the `truffle.js` file; all your configuration will be
located in `truffle-config.js`. If you are on Linux, you can stay with `truffle.js`.

Truffle allows us to start a development blockchain on our own computer, by
prompting `truffle develop`. This runs the client on `http://127.0.0.1:9545`,
so we need to configure our development environment. Open our configuration file
(`truffle-config.js` on Windows, `truffle.js`, try to keep up :P), and make sure
it looks like this:

```js
// truffle.js

module.exports = {
  networks: {
    develop: {
      host: "127.0.0.1",
      port: 9545,
      network_id: "*" // Match any network id
    }
  }
};
```

### Writing the smart contract

Now, let's generate our new contract. Prompt

```
$ truffle create contract GiveTheMoneyBack
```

This will create a file called `GiveTheMoneyBack.sol` in the `contracts` folder.
It should look something like this.

```
pragma solidity ^0.4.4;

contract GiveTheMoneyBack {
  function GiveTheMoneyBack() {
    // constructor
  }
}
```

If you are familiar with Object-Oriented Programming, you should not be
destabilized too much. Smart Contracts look a lot like usual classes; the main
difference being that they are identified by the keyword `contract`. The
function you find in the brackets is simply the constructor for the contract.

Any code written for the Ethereum blockchain should specify in which version of
the language it is written. Here, we want to use at least the version 0.4.18, so we'll modify it.

The first thing we need to do is specify our variables. We need to save:
* the identity of the person who must send money
* the amount of money to send
* the identity of the person who created the request
* the identity of anyone who support the request
* the description of the request

The beginning of our contract should therefore look like:

```
pragma solidity ^0.4.18;

contract GiveTheMoneyBack {

  address public owner; // creator of the contract
  address public receiver; // person who owes money
  address[] public backers; // people who back up the request
  uint256 public amount; // amount of wei owed. 1 ether = 10^18 wei
  string public description; // why the money is owed

  /**
   * Constructor
   *
   * @param receiverAddress address of the one who owes ether
   * @param etherAmount amount of ether owed
   * @param explanation explanation of the request
   */
  function GiveTheMoneyBack(address receiverAddress, uint256 etherAmount, string explanation) public {
      owner = msg.sender;
      receiver = receiverAddress;
      amount = etherAmount * 10 ** 18; // converts the ether into wei.
      description = explanation;
      backers.push(owner); // add the creator of the contract as a backer
  }
}
```

Let's break this down:
* In the Ethereum blockchain, any member is identified by a public address
(for instance: 0xaD98626112895261332180A1613bf04D95fF102A). In solidity, this
is identified by the `address` type. Here, we therefore store:
    * the address of the contract creator (the `owner`)
    * the address of the person who owes money (the `receiver`)
    * the addresses of the ones who supported the request (the `backers`). The
    `[]` simply signifies that we store an array instead of a simple, plain
    value.
* Ether can be divided up to the 18th decimal. As working with money using float
is a reason for death penalty in multiple countries, we will work using the
smallest possible unit of ether, the "wei". The amount of wei to send is
therefore stored using the `uint256` type.
* the description is stored using the usual `string` type.
* the `public` keyword is the *visibility* of the element, it means that anyone
can access it.
* the constructor function initializes the contract with the values we want. We
specify the address of the receiver and the amount of ether to send (which is
  converted into wei). We don't have to specify the address of the owner of the
  contract, since it can be found using `msg.sender`.

Now, let's create the functions we will need for this contract to work:
* A member other than the owner or the receiver must be able to support the
  request
* The receiver must be able to send an amount of ether to pay his or her debts

We now have:

```js
pragma solidity ^0.4.18;

contract GiveTheMoneyBack {

    address public owner; // creator of the contract
    address public receiver; // person who owes money
    address[] public backers; // people who back up the request
    uint256 public amount; // amount of wei owed
    string public description; // why the money is owed

    /* Generates a public event that notify clients when a new user backes the request */
    event Backed(address backer);

    /* Generates a public event that notify clients when the receiver has given the money back */
    event MoneyGivenBack();

    /**
     * Constructor
     *
     * @param receiverAddress address of the one who owes ether
     * @param etherAmount amount of ether owed
     * @param explanation explanation of the request
     */
    function GiveTheMoneyBack(address receiverAddress, uint256 etherAmount,
      string explanation) public {
        owner = msg.sender;
        receiver = receiverAddress;
        amount = etherAmount * 10 ** 18; // converts the ether into wei. 1 ether = 10^18 wei
        description = explanation;
        backers.push(owner); // add the creator of the contract as a backer
    }

    /**
    * Back up the request for money
    */
    function backUp() public returns (bool) {
        uint nbBackers = backers.length;
        for (uint index = 0; index < nbBackers; index++) {
            require(backers[index] != msg.sender);
        }
        backers.push(msg.sender);
        Backed(msg.sender);
        return true;
    }

    /**
    * Allows the receiver to pay his or her debts.
    */
    function payDebt() public payable returns (bool) {
        require(msg.sender == receiver && msg.value >= amount);
        uint nbBackers = backers.length;
        uint256 amountToDistribute = amount / nbBackers;
        for (uint index = 0; index < nbBackers; index++) {
            backers[index].transfer(amountToDistribute);
        }
        owner.transfer(amountToDistribute + (amount % nbBackers));
        MoneyGivenBack();
        return true;
    }

}
```

We have added quite a few things here:
* two events, identified thanks to the `event` keyword, that will be used to

send notifications when new users back up the request, or when the receiver pays.
* in the `backUp` function:
  * we start by iterating into the backers table to check if the user who wants
  to back up the request is not already in it.
  * The `require()` function will test if the assertion is true, otherwise throw
  an error.
  * If the user is not already a backer, he or she is then added and the `Backed`
  event is triggered.
* in the `payDebt`:
  * we have a new modifier called `payable`. It is used to signify that it is
  possible to send Ether when triggering the function.
  * We check if the user who triggered the function is the receiver, and if the
  amount of ether sent is the one expected.
  * We divide the amount of money by the number of backers. If there is a rest,
  it will be given to
  the contract owner.
  * the `transfer` function is used to transfer ether to an address. If the
  transfer fails, an error will be thrown.
  * We trigger the `MoneyGivenBack` event to notify the clients.

### Testing our contract

This should complete the code of our contract. We'll now see how we can test it. Firstly,
open your `1_initial_migration.js` file, and add tell Truffle to deploy our new contract:

```js
var Migrations = artifacts.require("./Migrations.sol");
var GiveTheMoneyBack = artifacts.require("./GiveTheMoneyBack.sol");

module.exports = function(deployer) {
  deployer.deploy(Migrations);
  deployer.deploy(GiveTheMoneyBack);
};
```

Then, generate the test file:
```
$ truffle create test GiveTheMoneyBack
```

Open the newly generated `give_the_money_back.js` file in the `test` folder. It
should look something like:

```js
contract('GiveTheMoneyBack', function(accounts) {
  it("should assert true", function(done) {
    var give_the_money_back = GiveTheMoneyBack.deployed();
    assert.isTrue(true);
    done();
  });
});
```

Before we can launch the test, we need to start the development blockchain:

```
$ truffle develop
```

Your CLI should now look like this:

```
$ truffle develop
Truffle Develop started at http://localhost:9545/

Accounts:
(0) 0x627306090abab3a6e1400e9345bc60c78a8bef57
(1) 0xf17f52151ebef6c7334fad080c5704d77216b732
(2) 0xc5fdf4076b8f3a5357c5e395ab970b5b54098fef
(3) 0x821aea9a577a9b44299b9c15c88cf3087f3b5544
(4) 0x0d1d4e623d10f9fba5db95830f7d3839406c6af2
(5) 0x2932b7a2355d6fecc4b5c0b6bd44cc31df247a2e
(6) 0x2191ef87e392377ec08e7c08eb105ef5448eced5
(7) 0x0f4f2ac550a1b4e2280d04c21cea7ebd822934b5
(8) 0x6330a553fc93768f612722bb8c2ec78ac90b3bbc
(9) 0x5aeda56215b167893e80b4fe645ba6d5bab767de

Private Keys:
(0) c87509a1c067bbde78beb793e6fa76530b6382a4c0241e5e4a9ec0a0f44dc0d3
(1) ae6ae8e5ccbfb04590405997ee2d52d2b330726137b875053c36d94e974d162f
(2) 0dbbe8e4ae425a6d2687f1a7e3ba17bc98c673636790f1b8ad91193c05875ef1
(3) c88b703fb08cbea894b6aeff5a544fb92e78a18e19814cd85da83b71f772aa6c
(4) 388c684f0ba1ef5017716adb5d21a053ea8e90277d0868337519f97bede61418
(5) 659cbb0e2411a44db63778987b1e22153c086a95eb6b18bdf89de078917abc63
(6) 82d052c865f5763aad42add438569276c00d3d88a2d062d36b2bae914d58b8c8
(7) aa3680d5d48a8283413f7a108367c7299ca73f553735860a87b08f39395618b7
(8) 0f62d96d6675f32685bbdb8ac13cda7c23436f63efbb9d07700d8669ff12b7c4
(9) 8d5366123cb560bb606379f90a0bfd4769eecc0557f1b362dcae9012b548b1e5

Mnemonic: candy maple cake sugar pudding cream honey rich smooth crumble sweet treat

truffle(develop)>
```

What you can see here is the list of the accounts generated on the development
blockchain. If you have paid closely attention to the generated test code, they
correspond to the `accounts` parameter that is given in the first callback.

We will use them to mock our users. The `truffle(develop)>` shows that you are
in development mode; in this mode, you can use the truffle commands without
using `truffle <command>`, but simply by prompting the command name.

Try it by launching the test:
```
truffle(develop)> test
```

You should get an error. This is because we have not yet defined the variable
`GiveTheMoneyBack` as an abstraction of our contracts. To do so, we need to add,
at the beggining of the test:

```js
// Requests an abstraction of the contract to tests
var GiveTheMoneyBack = artifacts.require("GiveTheMoneyBack");
```

If you launch the test, it should now work.

Let's therefore write our test cases. We need to make sure that:
* a user can back up the request once
* a user cannot back up the request twice
* the receiver cannot back up the request
* the receiver cannot send a different amount that the one required
* the amount of ether is equally dispatched between the backers

Our test should thus look like:

```js
// Requests an abstraction of the contract to tests
var GiveTheMoneyBack = artifacts.require("GiveTheMoneyBack");

contract('GiveTheMoneyBack', function(accounts) {

    it ('Non-backer can back up the request', function() {
      return GiveTheMoneyBack.new(accounts[1], 10, 'Lorem Ipsum', { from: accounts[0] })
      .then( function(instance) {
        return instance.backUp.call({from: accounts[2]});
      }).then(function(result) {
        assert.isTrue(result);
      });
    });

    it ('Cannot back up the request twice', function() {

      var meta;

      return GiveTheMoneyBack.new(accounts[1], 10, 'Lorem Ipsum', { from: accounts[0] })
      .then( function(instance) {
        meta = instance;
        return instance.backUp({from: accounts[2]});
      }).then(function(result) {
        assert.equal(result.logs[0].event, "Backed");
        return meta.backUp({from: accounts[2]});
      }).then(function(result) {
        assert.equal(result.logs[0], undefined);
      });

    });

    it ('Receiver cannot back up the request', function() {
      return GiveTheMoneyBack.new(accounts[1], 10, 'Lorem Ipsum', { from: accounts[0] })
      .then( function(instance) {
        return instance.backUp.call({from: accounts[1]});
      }).then(function(result) {
        assert.isFalse(result);
      });
    });

    it ('Receiver cannot send a different amount that the one required', function() {

      return GiveTheMoneyBack.new(accounts[1], 1, 'Lorem Ipsum', { from: accounts[0] })
      .then( function(instance) {
        return instance.payDebt.call({from: accounts[1], value: web3.toWei(0.5, 'ether')});
      }).then(function(result) {
        assert.isFalse(result);
      });

    });

    it ('Ether is evenly dispatched between the backers', function() {

      var meta;

      //Create the variables for the balance in ether of accounts 3 & 4
      var account_zero_starting_balance;
      var account_three_starting_balance;
      var account_four_starting_balance;

      return GiveTheMoneyBack.new(accounts[1], 1, 'Lorem Ipsum', { from: accounts[0] })
      // User 3 backs up the request
      .then( function(instance) {
        meta = instance;
        return meta.backUp({from: accounts[3]});
      })
      // User 4 backs up the request
      .then( function(result) {
        assert.equal(result.logs[0].event, "Backed");
        return meta.backUp({from: accounts[4]});
      })
      // User 1 pays
      .then( function(result) {
        assert.equal(result.logs[0].event, "Backed");

        // Estimate the balance of accounts 0, 3 and 4 before they receive the ether.
        account_zero_starting_balance = web3.eth.getBalance(accounts[0]).toNumber();
        account_three_starting_balance  = web3.eth.getBalance(accounts[3]).toNumber();
        account_four_starting_balance = web3.eth.getBalance(accounts[4]).toNumber();

        return meta.payDebt({from: accounts[1], value: web3.toWei(1, 'ether')});
      })
      // Checks the event has been emited, and get the amount to split between the backers.
      .then(function(result) {
        assert.equal(result.logs[0].event, "MoneyGivenBack");

        // Checks the new balance is superior to the old one
        assert.isTrue(account_zero_starting_balance < web3.eth.getBalance(accounts[0]).toNumber());
        assert.isTrue(account_three_starting_balance < web3.eth.getBalance(accounts[3]).toNumber());
        assert.isTrue(account_four_starting_balance < web3.eth.getBalance(accounts[4]).toNumber());

        // Checks every account has earn as much ether
        assert.equal(account_three_starting_balance - web3.eth.getBalance(accounts[3]).toNumber(), account_zero_starting_balance - web3.eth.getBalance(accounts[0]).toNumber());
        assert.equal(account_three_starting_balance - web3.eth.getBalance(accounts[3]).toNumber(), account_four_starting_balance - web3.eth.getBalance(accounts[4]).toNumber());

      });
    });

});
```

What have we done here ?
* As previously said, Truffle uses Mocha to run unit tests. All the tests are
therefore written like this:

  `it ("test name", function() { "test code" } )`
* Every test is run independently, so the state of one test does not affect the
  other one
* All our tests start with `GiveTheMoneyBack.new(accounts[1], 10, 'Lorem Ipsum', { from: accounts[0] })`.
  This tells Truffle to deploy our contract using the abstraction provided by
  Truffle. It is best to use this abstraction instead of the standard web3js API,
  since it allows us to use Promises instead of having a callback hell. The
  three first parameters (`accounts[1]`, `10`, `Lorem Ipsum`) are the parameters
  given to the constructor function. ` { from: accounts[0] }` tells Truffle that
  the contract should be deployed by the first account.
* Their are two ways of using the functions of our contract: by making a *call*,
 or a *transaction*:
  * to make a call, we simply need to write the name of the function, and use
  the `call` method on it. For instance, `instance.backUp.call({from: accounts[2]});`
  makes a call for the backUp function of our contract instance. Calls are
  interesting because they do not consume any gas, nor do they change the state
  of our contracts; and the `result` given is the one expected at the end of our
  method.
  * to change a contract, we need to make a *transaction*. Instead of using the
  `call` method, we directly use the ones written in our contract. However, the
  `result` given by this function will not be the actual result of our contract
  method, but rather an object defining the transaction. Therefore, the only way
  we have to make sure the function executed correctly is to look at the events
  that were emitted, if any.
* In our last test, we cannot directly check if the accounts were given as much
as we would have expected, because every transaction comes with a *transaction
fee*. We can only check if every account has received as much ether.

Let's launch the tests again:
```
truffle(develop)> test

Using network 'develop'.

Compiling .\contracts\GiveTheMoneyBack.sol...


  Contract: GiveTheMoneyBack
    √ Non-backer can back up the request (548ms)
    √ Cannot back up the request twice (569ms)
    √ Receiver cannot back up the request (258ms)
    √ Receiver cannot send a different amount that the one required (258ms)
    √ Ether is evenly dispatched between the backers (3853ms)


  5 passing (6s)
```

All the tests are successful !

## Conclusion

In this article, we have seen how we could set up an Ethereum project using
Truffle, how we could write a smart-contract, and how we could test it before
publishing it. I hope you have found it useful; do not hesitate to ask me any
question if there is some grey area.

All the code of this project is available on [Gitlab](https://github.com/searev-experiments/gtmb-smart-contract)

In the next article, we will see how we can create our own application working
with using Angular 5, and how to make it interact with our contract.
