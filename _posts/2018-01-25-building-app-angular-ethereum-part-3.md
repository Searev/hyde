---
layout: post
title: "Building a dApp with the Ethereum blockchain and Angular 5 - part 3: the application"
---

[Last time](http://huberisation.eu/2018/01/24/building-app-angular-ethereum-part-2/),
we saw how we could set up an angular app and write a service to interact with
a smart-contract. Today's session will be more focused on the Angular part,
and how we can use the service we built inside it.

Before we develop our app any further, launch the local web server as
well as our development blockchain:

```
$ ng serve
```

```
$ truffle develop
```

## Exporting our service to be available to all our components

The service we built is located in the `src/app/core` folder of our application,
which should look like this:
```
|-- core/
|   |-- core.module.ts
|   |-- gtmb.service.ts
|   |-- gtmb.service.spec.ts
```

Open the `core.module.ts` file. It is the definition of the Core module. Since
this file has been generated by the Angular CLI, it should look something like
this:

``` javascript
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';

@NgModule({
  imports: [CommonModule],
  declarations: [],
  providers: []
})
export class CoreModule { }
```

The `@NgModule` annotation specifies the class as an Angular Module, and is
given some arrays:
* the `imports` array specifies which existing module we should import to make
  this one work. By default, the `CommonModule` is imported: it allow us to use
  the usual angular directives and pipes like `ngFor`, `ngIf`... in our
  components. Since this module will only be used to provide the service
  globally, we can remove this inport.
* the `declarations` array specifies the various components used in this module,
  and that should be loaded. Since we don't have any, we can keep it empty.
* the `providers` section specifies the functions and services that
  will be used in the module, so they can be available through the dependency
  injection mechanism of Angular. As you have guessed, we will specify our
  service here. Add the `GtmbService` in the array, and make sure it is
  correctly imported.
* a section that is not shown here but that can be used is the `export` array; it
  specifies the components that can be used in other modules that would import
  this one. We don't need it here.

Just specifying our service in the `providers` section should be enough to use
it in our application, by imporing the `CoreModule` in the `AppModule`. In the
`src/app/app.module.ts` file, add the `CoreModule` in the `imports` section.

``` javascript
@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule,
    CommonModule,
    CoreModule,
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

## Feature module, lazy loading and routing

One thing to keep in mind when developing front-end apps is that they generate
a lot of code that must be downloaded by the end user. If our application is
quite small, other ones can easily have multiple feature modules, and tens
of thousands of lines of code. In those conditions, it makes no sense to load
*all* the application modules at once, when maybe only a fraction of them is to
be used.

To tackle that issue, Angular comes with *Lazy Loading*, a mechanism in which
modules are only loaded by the end user if he or she is to use them, by
navigating to a specific route for instance. Here, we are therefore going to see
how to lazy load our *GiveTheMoneyBackModule*, and specify its routes.

First of all, we must tell our application that this module is only to be loaded
for routes that start with, let's say, `/requests`. For those routes, the inner
routing module of the *GiveTheMoneyBackModule* should be used.

To do so, open the `src/app/app-routing.module.ts` file, and paste in it the
following lines:

``` javascript
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

import { AppComponent } from './app.component';

const routes: Routes = [
  { path: '', redirectTo: '/requests', pathMatch: 'full' },
  { path: 'requests', loadChildren: 'app/give-the-money-back/give-the-money-back.module#GiveTheMoneyBackModule' },
];

@NgModule({
  imports: [ RouterModule.forRoot(routes) ],
  exports: [ RouterModule ]
})
export class AppRoutingModule { }
```

As you can see, we defines here some of the routes of our applications. In the
first one, we specify that the application should redirect the user to the
`/requests` route by default. In the second one, we specify that all routes
starting with `requests` will be handled by the `GiveTheMoneyBackModule`, which
is therefore to be lazy loaded. Import this routing module in the `AppModule` by
adding it to the `imports` array.

Let's now see the routing module for this one. Open the
`src/app/give-the-money-back/give-the-money-back-routing.module.ts` file, and
paste in it the following lines:

``` javascript
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

import { ListComponent } from './list/list.component';
import { CreateComponent } from './create/create.component';
import { DetailComponent } from './detail/detail.component';

export const routes: Routes = [
  { path: '', component: ListComponent},
  { path: 'create', component: CreateComponent},
  { path: ':address', component: DetailComponent},
];

@NgModule({
  imports: [ RouterModule.forChild(routes) ],
  exports: [ RouterModule ]
})
export class GiveTheMoneyBackRoutingModule { }
```

First of all, as you can see, we do not specify that our routes should start by
`requests`; that was handled previously, so we should only worry here about the
sub-resources of that URI. Then, we specify which component should be used for
the routes. The `path: ':address'` part specifies that the path of this route can
be any type of string, and that it will be available to its component as the
`address` parameter.

Now, let's see our `GiveTheMoneyBackModule`. Here, we will need to add a few
things:
* in the `declarations` section, add all the components we generated previously:
  the `ListComponent`, the `ThumbnailComponent`, the `DetailComponent` and the
  `CreateComponent`.
* in the `imports` section, add the `GiveTheMoneyBackRoutingModule` we just
  wrote. We will also to import the modules that allow us to use the usual
  angular directives (`CommonModule`), as well as those that will help us
  handle forms: `FormsModule` and `ReactiveFormsModule`.

Great ! Now, we only have to actually implement our components.

## Writing the components

### Application Component

This is the component in which all the other ones will be loaded. As such,
it is a good place to put a navigation bar.

As all the logic and interaction will be placed in our services and components,
we can simply specify its template in the `app.component.html` file:

```
<div>
  <span id="toolbar-row">
    <button routerLink="/requests">GTMB!</button>
    <button routerLink="/requests/create">New</button>
  </span>
</div>

<router-outlet></router-outlet>
```

The `routerLink` option in the buttons specify that when they are clicked on,
the user should be redirected to the specified routes.

The `<router-outlet></router-outlet>` part specifies that the modules loaded
because of the route the user is on should be displayed here, just after our
navigation bar.

### Create Component

This component will be used to create new requests for money. It should display
a form that will allow the user to specify:
* the address of the request receiver
* the amount of ether to send
* a description of the request

Open the `create.component.ts` file. We need to add a few things there:

``` javascript
import { Component } from '@angular/core';
import { Request } from '../request';
import { GtmbService } from '../../core/gtmb.service';
import { Router } from '@angular/router';
import { OnDestroy } from '@angular/core/src/metadata/lifecycle_hooks';

@Component({
  selector: 'app-create',
  templateUrl: './create.component.html',
  styleUrls: ['./create.component.css']
})
export class CreateComponent {

  public request: Request = new Request();

  constructor(private gtmbService: GtmbService,
              private router: Router) { }

  onSubmit() {
    this.gtmbService.createContract(this.request)
    .then(
      data => this.router.navigate(['/requests', data.address]),
      err => console.log(err)
    );
  }

}
```

When the component is created, we directly create a `Request` object that
matches the request we want to create. It will be then used in
the `createContract` method of our service.

Talking about our service, let's import it using the dependency injection
mechanism. To do so, we only have to specify it in the constructor of our
component. Likewise, we import the `Router`, that will allow us to redirect
the user to the detail page of the contract once it is created.

In the `onSubmit()` function, we tell our service to create the request according
to the one we wrote. Once it is done, we simply navigate to the detail page.

In order to fill our `Request` object and call this function, we must write the
template of our component. Open the `create.component.html` file, and let's have
a look at what we must add in it:

``` html
<h1>Create a request</h1>

<form (submit)="onSubmit()">
  <input placeholder="Request money from" name="recipient" [(ngModel)]="request.receiver">
  <input placeholder="Amount in ether" name="amount" type="number" [(ngModel)]="request.amount">
  <textarea placeholder="description" name="recipient" [(ngModel)]="request.description"></textarea>
  <button type="submit">Request the money</button>
</form>
```

As you can see, this is simply a form with the input for everything we must
specify. There are a few things that may be new for new Angular developers:
* the `(submit)="onSubmit()"` part in the `form` tag specifies that when the
  form is submitted, we must trigger the function we wrote
* the `[(ngModel)]="..."` is what we call a *double binding*. It means that the
  value displayed in the field, as the page is loaded and as we fill it, is the
  same as the one specified. Here, we specify that we must change the attributes
  of our request object.

With a few styling (I use the [Angular Material components]()), the form looks
like this:

![form](https://raw.githubusercontent.com/Searev/searev.github.io/master/_assets/2018-01-25-building-app-angular-ethereum-part-3/form.png)

### Detail component

Once the contract is created, we must be able to see it in details. To do so,
let's write the `DetailComponent`. Open the `detail.component.ts` file.

``` javascript
import { Component, OnInit } from '@angular/core';
import { GtmbService } from '../../core/gtmb.service';
import { ActivatedRoute } from '@angular/router';
import { Request } from '../request';

@Component({
  selector: 'app-detail',
  templateUrl: './detail.component.html',
  styleUrls: ['./detail.component.css']
})
export class DetailComponent implements OnInit {

  public request: Request;

  constructor(private route: ActivatedRoute,
              private gtmbService: GtmbService) { }

  ngOnInit() {

    this.route.params.subscribe(
      params => {
        this.gtmbService.getContract(params['address'])
        .then(
          data => {
            this.request = data;
            this.request.address = params['address'];

            this.gtmbService.getRequestBackers(params['address'])
            .then(
              backers => this.request.backers = backers;,
              err => console.log(err)
            );
          },
          err => console.log(err),
        );


      }
    );
  }

  canBackUp(): boolean {
    return this.request.receiver !== this.gtmbService.getAccount()
        && !this.request.backers.some(backer => backer === this.gtmbService.getAccount());
  }

  canPayDebt(): boolean {
    return this.request.receiver === this.gtmbService.getAccount();
  }

  backUp() {
    this.gtmbService.backUp(this.request.address)
    .then(
      data => this.request.backers.push(data),
      err => console.log(err)
    );
  }

  pay() {
    this.gtmbService.payDebt(this.request.address)
    .then(
      data => { if (data) { alert('Thank you for giving the money back.'); } },
      err => console.log(err)
    );
  }

}
```

We store in the `request` attribute the Request we are going to display. We
import our service to fetch the data from the blockchain, and the `ActivatedRoute`
part to be able to read the address of the contract to fetch.

This components implements the `OnInit` interface: it means that when it will
be initialized, the `ngOnInit` function will be triggered. This function is part
of the Angular components life cycle, and is triggered after the constructor; when
the attributes and services have been loaded. Here, we listen to the parameters
of the route, to be able to fetch the address, as we defined it in our routing
module. Once the address is fetched, we use our own service to fetch the
contract, and load its backers.

The `canBackUp()` and `canPayDebt()` functions will be used to decide which
actions the user should be suggested.

They are followed by the actual `backUp()` and `pay()` functions, that use
our service to make transactions with our contract.

Our template should look something like this:

``` html
<div *ngIf="request">

  <div>
    <h1>Request</h1>
    <span>{{request.address}}</span>
  </div>

  <div> <h3>To:</h3> <span>{{request.receiver}}</span> </div>
  <div> <h3>Amount:</h3> <span>{{request.amount / 10e18 }} eth</span> </div>
  <div>
    <h3>Description</h3>
    <p>{{request.description}}</p>
   </div>

   <div>
     <h3>Supported by:</h3>
     <ul *ngIf="request.backers">
       <li *ngFor="let backer of request.backers">{{backer}}</li>
     </ul>
   </div>

   <div>
     <button *ngIf="canPayDebt()" (click)="goBack()">Keep the money</button>
     <button *ngIf="canPayDebt()" (click)="pay()">Give the money back</button>
     <button *ngIf="canBackUp()" (click)="backUp()">Support the request</button>
    </div>
</div>
```

Firstly, we use the `*ngIf` directive to tell Angular that this part should only
be displayed if the `request` object is not `null` or `undefined`; which happens
when it is loaded via our service.

The values in the double brackets, like `{{request.address}}`, tell Angular that
it should display the value from the objects in the components.

Then, we use the same `*ngIf` directive to chose which action should be
displayed.

With a little style, it can look like this:

![detail](https://raw.githubusercontent.com/Searev/searev.github.io/master/_assets/2018-01-25-building-app-angular-ethereum-part-3/detail.png)

### The search and thumbnail component.

The search component will contain a simple text input to write in it the
address of a contract; and launch a search for it with our service. If
one contract matches the address, a brief summary should be displayed. The user
can click on it, and be redirected to the detail page.

Firstly, let's open the `list.component.ts` file.

``` javascript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Router } from '@angular/router';
import { FormControl } from '@angular/forms';
import 'rxjs/add/operator/takeWhile';
import 'rxjs/add/operator/debounceTime';
import 'rxjs/add/operator/distinctUntilChanged';

import { GtmbService } from '../../core/gtmb.service';
import { Request } from '../request';

@Component({
  selector: 'app-list',
  templateUrl: './list.component.html',
  styleUrls: ['./list.component.css']
})
export class ListComponent implements OnDestroy {

  public filteredRequest: Request;

  private searchCtrl = new FormControl();
  private alive = true;

  constructor(private gtmbService: GtmbService,
              private router: Router) {
    this.searchCtrl
      .valueChanges
      .debounceTime(300)
      .distinctUntilChanged()
      .takeWhile(() => this.alive)
      .subscribe(
        term => {
          this.gtmbService.getContract(term)
          .then(
            data => this.filteredRequest = data,
            err => console.log(err)
          );
        }
      );
    }

  ngOnDestroy() {
    this.alive = false;
  }

  goToDetail() {
    this.router.navigate(['/requests', this.searchCtrl.value]);
  }

}
```

The request we search for will be written as `filteredRequest`. The `searchCtrl`
will be used to listen to the user input.

This input is available as an `Observable`. Observables are like promises in
that ease the task of making asynchronous calls, but with a few differences.
Unlike Promises, Observables must be *subscribed* to; Angular will then wait for
new responses until the Observable have been *unsubscribed*.

Here, we tell Angular in the constructor that it should subscribe to the value
changes of our form control; that is to say to our user input. The
`debounceTime(300)` and `distinctUntilChanged()` methods are used to make sure
that at least 300ms must pass between two searches, and that the search will
only be triggered if the value has actually changed.

As the search can go on and on as long as the component is open, we want to
unsubscribe from the Observable when the component is destroyed. To do so,
we use the `takeWhile()` operator to tell Angular that this Observable is to
be subscribed to as long as a value, here `alive`, is true. When the component
is destroyed (for instance, when we change to a different one), the
`ngOnDestroy()` function will be called, make `alive` `false`, which will
make Angular unsubscribe from the Observable.

The `goToDetail()` function, finally, is used to redirect our user to the detail
page of a request.

The template should look something like this:

``` HTML
<div>
      <input placeholder="Address of the contract" name="address" [formControl]="searchCtrl">

      <app-thumbnail [request]="filteredRequest" *ngIf="filteredRequest" (click)="goToDetail()"></app-thumbnail>
</div>
```

We bind the value of our input with our form control using
`[formControl]="searchCtrl"`. The `app-thumbnail` tag specifies that it should
load our `ThumbnailComponent`, and give as input the value of our filtered
request. When clicked on, the user will be redirected thanks to the event
`(click)="goToDetail()"`.

Now, let's check out this thumbnail we use, in the `thumbnail.component.ts`
file:

``` javascript
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-thumbnail',
  templateUrl: './thumbnail.component.html',
  styleUrls: ['./thumbnail.component.css']
})
export class ThumbnailComponent {

  @Input()
  request: Request;

  constructor() { }

}
```

This component is quite basic: we simply expect the `request` object as input,
which we did in our previous template. Then, we specify the template that it
should use to be displayed, for instance:

``` HTML
<div>
  <div>
    {{request.owner}} requested {{request.amount / 10e18}} eth from {{request.receiver}}
  </div>
  <div>
      <span>{{request.nbBackers}} backers</span>
  </div>
</div>
```

Here is what it can looked like:

![search](https://raw.githubusercontent.com/Searev/searev.github.io/master/_assets/2018-01-25-building-app-angular-ethereum-part-3/search.png)

## To go further: use MetaMask

For now, we have been working with the truffle development blockchain. However,
we must keep in mind that this application is to be executed in someone else's
browser; someone who is not likely to have set up a full blockchain node on
his or her computer. We therefore need to find another provider for our
web3js instance.

Thanksfully, an add-on can help us doing so easily: [Metamask](https://metamask.io/).
If you have not it yet, please install it in your browser before continuing.
Metamask injects in the web browser window an object called `web3`, that will
give us a provider to interact with the Ethereum network.

We need to update our service to work with Metamask instead of using the
Truffle development environment. This can be done by simply changing one line
of our constructor in `gtmb.service.ts`:

``` javascript
// other imports

@Injectable()
export class GtmbService {

  // properties

  constructor() {
    //const web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:9545'));
    const web3 = new Web3(window['web3'].currentProvider);
    this.account = web3.eth.accounts[0];
    this.contract = contract(GiveTheMoneyBack);
    this.contract.setProvider(web3.currentProvider);
  }

  // other functions
}
```

If you want to use it with the development blockchain, you will first need to
configure it to work with our private network, which is most likely running at
`http://localhost:9545`.

You will need to connect to this network using an account. You can either chose
one of those provided by Truffle - in that case, you might need to restore it
from a pass phrase, most likely *candy maple cake sugar pudding cream honey* 
*rich smooth crumble sweet treat*.

![truffle reset password](https://raw.githubusercontent.com/Searev/searev.github.io/master/_assets/2018-01-25-building-app-angular-ethereum-part-3/metamask_restore.png)

When using Metamask, if the user is about to perform a transaction,
like create a request, back one up or pay, he or she will then see
a pop-up scren to specify the amount of gas, confirm the transaction,
etc.

![truffle-approve](https://raw.githubusercontent.com/Searev/searev.github.io/master/_assets/2018-01-25-building-app-angular-ethereum-part-3/metamask_approve.png)

We could set up a Guard to check that the user who loads the application
has correctly set up MetaMask; and redirect him or her to a page that explains
it if not. But that goes beyond the scope of this project.

## Conclusion

In this last part, we've seen how we could:
* inject a service in the components of an Angular application
* use lazy loading and define our routes
* use the basic Angular directives
* interact with Observables
* use MetaMask as a provider

I hope you found this little project to be interesting; do not hesitate to
contact me if you have any question, or drop a line if you like it!

As usual, the code is hosted on [Github](https://github.com/searev-experiments/gtmb-app).

## Sources
For this last part, I mostly relied on:
* the [Angular Documentation](https://angular.io/docs)
* the excellent [Angular2 training book](https://angular-2-training-book.rangle.io/handout/modules/lazy-loading-module.html)
* the [MetaMask documentation](https://github.com/MetaMask/faq/blob/master/DEVELOPERS.md)
