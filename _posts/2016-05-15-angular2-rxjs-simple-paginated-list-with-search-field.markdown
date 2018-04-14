---
layout: post
title: 'Angular2 and Rxjs : a simple paginated list with search field'
description: Use Observable to create a simple list with a search field and a pagination
date: 2016-05-15
tags:
    - angular
    - javascript
    - rxjs
---

## Introduction

Coming from an imperative background, I had a hard time wrapping my head around the reactive and functional approach. I read some tutorials and tried to dabble a little with Rxjs but without too much success. Of course, I succeeded in doing basic stuff but without any real understanding.

At this time, I said to myself stop this. Like anything in computer science, if you are only a copycat, it means nothing. So I started to read this doc : [RxJS](http://xgrommx.github.io/rx-book/index.html). And I must say it was a real eye opener. In particular the [introduction](http://xgrommx.github.io/rx-book/content/guidelines/introduction/index.html). I encourage everyone to read it carefully. It is great stuff !!!!

I am still at the start of the doc but I wanted to share some codes I put together based on this introduction page.

**UPDATE 07 January 2017: As requested by a few people, I have put together a simple working example. It is not exactly the same code as in the blog post but it is based on the same principles : [Working example on github pages](https://jbouzekri.github.io/angular-search-list/) and [Code source](https://github.com/jbouzekri/angular-search-list)**

## Context

We are going to build a simple list of posts by displaying the id and title of each post. This list will provide pagination and a single seach field to filter the posts by their title.

The posts are coming from a simple REST JSON api `GET /posts` :

``` json
{
  "items": [
    {
      "id": 1,
      "title": "Title of my post 1",
      "text": "Text of my post 1"
    }
  ],
  "total": 5
}
```

So we have an array of posts and the total number of items matching the query string (it can be more than the number of posts in the response)

The available params are :

* `limit` : limit the number of results returned by the API (default 10)
* `page` : offset the result (page - 1) * limit
* `search` : filter the result (ilike '%search%' in title)

I suppose that you already have a working component used to display the list. Let's say in the file 'app/components/post/post-list.component.ts'.

## Initialize interface and services

So we are going to query a HTTP API returning posts. We will need :

* an entity to represent our Posts
* an interface to describe the API response
* and of course a service to execute the query to the API

Let's get started with the Post entity.

I will put it in `app/components/post/post.entity.ts` :

``` js
export class Post {
  id: number

  title: string

  text: string
}
```

Then the API response interface (useful if you have a lot of API with the same json contract). Create a file `app/services/api/list-result.interface.ts` :

``` js
export interface ListResult<T> {
    items: T[]

    total: number
}
```

And finally the service to query the API in `app/components/post/post.service.ts` :

``` js
import { Injectable } from '@angular/core'
import { Observable } from 'rxjs/Observable'
import { URLSearchParams } from '@angular/http'
import { HTTP } from '@angular/http'

import { Post } from './post.entity'
import { ListResult } from '../../services/api/list-result.interface'

@Injectable()
export class PostService {
  constructor(protected http: HTTP) {}

  list(search: string = null, page: number = 1, limit: number = 10): Observable<ListResult<Post>> {
    var params = new URLSearchParams();
    if (search) params.set('search', search)
    if (page) params.set('page', String(page))
    if (limit) params.set('limit', String(limit))

    return this.http.get('http://myapidomain.com/post', { search: params }).map(res => res.json())
  }
}

```

Okay we are all set for starting our work in the component. But first let's explain the approach with Rxjs and the Observable.

## Rxjs and Observable === Streams

Why did I *LOVE* the introduction article of the Rxjs tutorial mentioned above ? Because of 2 things. First, it cleared for me what Observable really are. In a single word : *STREAMS*. Then it provided a great way to describe these streams with ASCII. Everybody working with Rxjs should write the same kind of ASCII description to describe the streams and the different transformation applied to them. We will do just that.

So what are we looking for our list. In fact it is simple, just 2 streams :

* The first one, the inputs filled in the search field
* Then the page number changing by clicking on the pagination

These events can be observed using native Rxjs functions however Angular does not expose observable for its view events (check [this discussion on github](https://github.com/angular/angular/issues/4062)). So the way to observe these events is by using [Subject](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/gettingstarted/subjects.md). This entity is both an Observer and an Observable. It is a kind of pass-through. As an Observer, it exposes a next method you can call to pass a value which it will expose as an Observable to all subscribers. Angular2 uses this in its documentation as an example of a [debouncing search field](https://angular.io/docs/ts/latest/guide/server-communication.html#!#more-observables) (which we will use too)

So as stated previously, let's describe our streams.

First the search field.

What should it do ?

1. Streams the value of the input search field
2. Debounce the field for a better user experience
3. Only pass distinct values
4. If a search is triggered, always get back to the first page

If all these conditions are filled, we can query the API to get the results matching the searched text.

In ASCII :

```
searchSource:     ---t----i--t--l-e---------------------------------------------c--->
                  vvvvvvvvv debounce vvvvvvvvvvvvvvvvvvvvvvvcccvvvvvvvvvvvvvvvvvvvvvv
                  vvvvvvvvv distinct vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv
                  ---------------------(1sec)---title------------------------------->
                  vvvvvvvvv map vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv
observableSource: ------------------------------{"search": "title", "page": 1}------>
```

Then the second stream, the pagination. It is even simpler. Each click on a page number streams this number (c = click, C = completed).

```
pageSource:      ---c------------------------------------c------------------------------C->
                 ---1------------------------------------3-------------------------------->
                 vvvvvvvvv map vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv
observablePage:  ---{"search": this.search, "page": 1}---{"search": this.search, "page": 3}
```

Whatever happens in both streams, when we receive a value it triggers a query to the API. So let's merge them. I will call QueryParam the object describing `{"search": this.search, "page": this.page}`

```
observableSource: --QueryParam------------------------------------------>
observablePage:   -------------------------QueryParam------------------->
                  vvvvvvvvv merge vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv
                  --TriggerAPIcall---------TriggerAPIcall--------------->
                  vvvvvvvvv flatmap vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv
observable:       -------- ListResult<Post> --------- ListResult<Post> --
```

Okay, we have described a stream triggering an API call for each event we observe. When we receive an API answer, we just have to update the view with the response content.

Let's code.

## The code

As we have described the stream above, we almost only have to describe it as it is.

Let's look at the search observable :

``` js
terms: string = ""
private searchTermStream = new Subject<string>()

ngOnInit() {
  const searchSource = this.searchTermStream
    .debounceTime(1000)
    .distinctUntilChanged()
    .map(searchTerm => {
      this.terms = searchTerm
      return {search: searchTerm, page: 1}
    })
}

search(terms: string) {
  this.searchTermStream.next(terms)
}
```

As you can see, there is almost no difference between this code and the ASCII above (`terms` is the current value of the search field)

The pagination streams :

``` js
private pageStream = new Subject<number>()

ngOnInit() {
  const pageSource = this.pageStream.map(pageNumber => {
    this.page = pageNumber
    return {search: this.terms, page: pageNumber}
  })
}

goToPage(page: number) {
  this.pageStream.next(page)
}
```

We merge the 2 streams and trigger an API call :

``` js
ngOnInit() {
  ...
  const source = pageSource
    .merge(searchSource)
    .startWith({search: this.terms, page: this.page})
    .mergeMap((params: {search: string, page: number}) => {
      return this.postService.list(params.search, params.page)
    })
    .share()
}
```

And we separate the observable for the post list and the total result :

``` js
total$: Observable<number>
items$: Observable<Post[]>

ngOnInit() {
  ...

  this.total$ = source.pluck('total')
  this.items$ = source.pluck('items')
}
```

We now have 2 attributes to our component : `total$` and `items$`. Both are observable available to the view to display the list of posts and the total number of posts matching the query params.

Let's link all this with out view :

``` html
<div class="input-group input-group-sm" style="margin-bottom: 10px;">
  <input #term (keyup)="search(term.value)" value="{{ terms }}" class="form-control" placeholder="Search" autofocus>
  <div class="input-group-btn">
    <button type="submit" class="btn btn btn-default btn-flat"><i class="fa fa-search"></i></button>
  </div>
</div>
<table class="table table-striped table-hover">
  <tbody>
    <tr>
      <th>id</th>
      <th>title</th>
    </tr>
    <tr *ngFor="let post of items$ | async">
      <td>{{ post.id }}</td>
      <td>{{ post.title }}</td>
    </tr>
  </tbody>
</table>
<pagination [total]="total$ | async" [page]="page" (goTo)="goToPage($event)" [params]="{q: terms}"></pagination>
```

What do we have here ?

* A search field using a local variable `#term` and a listener on the `keyup` event. This listener triggers a method of our controller to push the value in the search field into the subject linked with search observable stream.
* I have moved the pagination into a dedicated component to be reusable. I will put the code bellow later. You should note the goToPage call. It pushed the page number clicked to the pagination component and to the page number observable stream.
* the `async` pipe on the `total$` and `items$` variable allows the view to directly use the observable. You don't have to subscribe/unsubscribe to anything. Let Angular do it for you.

## Conclusion

Everything is set. You have a list which will be updated when a user clicks on the pagination or use the search field. I hope everything is clear. Do not hesitate to write a message for any question or error in this post.

Please find bellow the full code (with the one for the pagination)

## Full code

The component :

``` js
import { Component, OnInit } from '@angular/core'
import { Observable } from 'rxjs/Observable'
import { Subject } from 'rxjs/Subject'
import { RouteParams } from '@angular/router-deprecated'

import { Post } from './post.entity'
import { PostService } from './post.service'
import { PaginationComponent } from '../pagination/pagination.component'

@Component({
    selector: 'post-list',
    templateUrl: '/app/components/post/post-list.html',
    directives: [PaginationComponent],
    providers: [PostService]
})
export class PostListComponent implements OnInit {
  total$: Observable<number>
  items$: Observable<Post[]>

  page: number = 1
  terms: string = ""

  private searchTermStream = new Subject<string>()
  private pageStream = new Subject<number>()

  constructor(protected params: RouteParams, protected postService: PostService) {
    this.page = parseInt(params.get('page')) || 1
    this.terms = params.get('q') || ""
  }

  ngOnInit() {
      const pageSource = this.pageStream.map(pageNumber => {
        this.page = pageNumber
        return {search: this.terms, page: pageNumber}
      })

      const searchSource = this.searchTermStream
        .debounceTime(1000)
        .distinctUntilChanged()
        .map(searchTerm => {
          this.terms = searchTerm
          return {search: searchTerm, page: 1}
        })

      const source = pageSource
        .merge(searchSource)
        .startWith({search: this.terms, page: this.page})
        .mergeMap((params: {search: string, page: number}) => {
          return this.postService.list(params.search, params.page)
        })
        .share()

      this.total$ = source.pluck('total')
      this.items$ = source.pluck('items')
  }

  search(terms: string) {
    this.searchTermStream.next(terms)
  }

  goToPage(page: number) {
    this.pageStream.next(page)
  }
}

```

The view is the same as above.

The pagination component :

``` js
import * as _ from 'lodash' // sorry use lodash for this example (another dependency ...)
import { Component, Input, EventEmitter, Output } from '@angular/core'
import { ROUTER_DIRECTIVES, Router } from '@angular/router-deprecated'
import { Location } from '@angular/common'

@Component({
    selector: 'pagination',
    templateUrl: '/app/components/pagination/pagination.html',
    directives: [ROUTER_DIRECTIVES]
})
export class PaginationComponent {
  totalPage: number = 0

  @Input()
  params: {[key: string]: string | number} = {}

  @Input()
  total: number = 0

  @Input()
  page: number = 1

  @Output()
  goTo: EventEmitter<number> = new EventEmitter<number>()

  constructor(protected _location: Location, protected _router: Router) {}

  totalPages() {
    // 10 items per page per default
    return Math.ceil(this.total / 10)
  }

  rangeStart() {
    return Math.floor(this.page / 10) * 10 + 1
  }

  pagesRange() {
    return _.range(this.rangeStart(), Math.min(this.rangeStart() + 10, this.totalPages() + 1))
  }

  prevPage() {
    return Math.max(this.rangeStart(), this.page - 1)
  }

  nextPage() {
    return Math.min(this.page + 1, this.totalPages())
  }

  pageParams(page: number) {
    let params = _.clone(this.params)
    params['page'] = page
    return params
  }

  pageClicked(page: number) {
    // this is not ideal but it works for me
    const instruction = this._router.generate([
      this._router.root.currentInstruction.component.routeName,
      this.pageParams(page)
    ])
    // We change the history of the browser in case a user press refresh
    this._location.go('/'+instruction.toLinkUrl())
    this.goTo.next(page)
  }
}

```

The pagination view :

``` html
<ul *ngIf="totalPages() > 1" class="pagination pagination-sm no-margin pull-right">
  <li *ngIf="page != 1"><a (click)="pageClicked(prevPage())">«</a></li>
  <li *ngFor="let p of pagesRange()"><a (click)="pageClicked(p)">{{p}}</a></li>
  <li *ngIf="totalPages() > page"><a (click)="pageClicked(nextPage())">»</a></li>
</ul>
```
