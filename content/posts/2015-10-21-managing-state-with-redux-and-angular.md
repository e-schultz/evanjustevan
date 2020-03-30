---
template: post
title: Managing state with Redux and Angular
slug: managing-state-with-redux-and-angular
draft: false
date: 2015-10-21T16:49:00.000Z
description: redux beyond react - using it with AngularJS
socialImage: /media/flux.jpg
category: Development
tags:
  - AngularJS
  - Redux
  - JavaScript
  - Web Development
---

![redux action flow](/media/es-lint-1.jpg 'Managing state with Redux and Angular')

**Update:**

If you're using Angular 1.x and you're interested in Redux this post is still quite relevant today, just make sure to check out the [latest documentation](https://github.com/angular-redux/ng-redux).

There have been a few times over the years when a new library, framework, or approach to programming has been introduced to me that has made me really reconsider how I build and write modern web applications.

The move from jQuery to Angular was one of those turning points. Switching from ngRoute to ui-router gave me a new way of thinking about my applications, and helped me reason about the state of my application.

Recently a library called [Redux](https://redux.js.org/) has started to change my approach to building Angular applications.

Redux provides a predictable state container. It is inspired by Flux and helps you implement a one-way data flow in your Angular applications. This allows you to understand what is going on in your system in a more predictable way.

Having been involved with many Angular applications over the years, either as a developer, code reviewer, or simply talking with co-workers, some of the common problems that seem to arise again and again are:

- Where is the state of my application?
- What is the state of my application?
- How do I share this state across multiple components?

Combing Redux, Angular, and a bindings library called ng-Redux, we solve these problems.

This combination will allow you to:

- View the entire state of your application
- Derive your UI from this state
- See how actions modify the state of your application

Before getting into how to use Redux with Angular, lets take a quick look at a basic concept that is at the core of Redux and how it works - Reducers.

# Reducers

A reducer is something that iterates over a collection of items and gets a final result out of it. The simplest example is summing up an array of numbers to get the total value.

```javascript
let sum = [1, 2, 3].reduce((accumulator, value) => accumulator + value, 0);
console.log(sum); // sum = 6
```

The accumlator is a value that keeps getting passed into the reducer, along with the next value in the array. The flow of the code is:

- 0 + 1 = 1
- 1 + 2 = 3
- 3 + 3 = 6

But, what if you want your result to be of a different type than the values in your collection? That isn't a problem. A simple tweak to demonstrate this is below:

```javascript
let result = [1, 2, 3].reduce(
  (accumulator, value) => {
    accumulator.sum += value;
    return accumulator;
  },
  { sum: 0 }
);
console.log(result);
```

As you can see, the result of your reduce function does not need to be the same type as the values that you iterate over. This is a very simple concept; so simple you might be wondering:

> How can I use this to represent the state of my application?

To show you how, we will build out a simple application using Redux with Angular. The application is called TrendyBrunch, and is [available on GitHub](https://github.com/e-schultz/ng-summit-redux).

To get started with the application:

```bash
git clone https://github.com/e-schultz/ng-summit-redux.git
cd ng-summit-redux
npm install
npm start
```

The full application has a number of features, but for this first blog post I will go over building out a lineup component.

Redux is a framework-agnostic library - meaning it can be used with your framework of choice. The community has created a number of bindings for popular frameworks. One of these bindings is called ng-redux by William Buchwalter.

To get Redux working with Angular, we first need to configure ngRedux inside of our angular.config block. To do this, we will need to inject \$ngReduxProvider, and tell it which reducers and middleware we want to use.

```javascript
import reducers from './reducers';
import createLogger from 'redux-logger';

const logger = createLogger({
  level: 'info',
  collapsed: true
});

export default angular
  .module('app', [ngRedux, ngUiRouter])
  .config($ngReduxProvider => {
    $ngReduxProvider.createStoreWith(reducers, [logger]);
  }).name;
```

This configures ngRedux with a reducer to handle our application state, and a [logging middleware](https://github.com/LogRocket/redux-logger) that will output every action in the system - displaying the previous state, the action, and the next state after the action was applied. Lets take a closer look at the reducer and how it fits into the application.

If you have been reading up on Flux Architecture, and Flux implementations, then you are familiar with the concept of a store. In Redux though, instead of having multiple stores - you have a single application state which is broken down into Reducers.

Reducers in Redux do take a stream of events - things that have happened in the past - and re-create your application state based on those events.

In our TrendyBrunch application, what we want to do is create a reducer that manages people joining, leaving or being seated from a lineup.

```javascript
const INITIAL_STATE = [];

export default function lineup(state = INITIAL_STATE, action) {
  if (!action || !action.type) {
    return state;
  }
  switch (action.type) {
    case PARTY_JOINED:
      return R.append(action.payload)(state);
    case PARTY_SEATED:
      return R.reject(n => n.partyId === action.payload.partyId)(state);
    case PARTY_LEFT:
      return R.reject(n => n.partyId === action.payload.partyId)(state);
    default:
      return state;
  }
}
```

One of the key things to keep in mind when creating your reducers, is that you want to return copies of your state with each operation, and not be modifying it. This is why we are using Ramda to help out here, instead of simply doing a state.push(action.payload), as that would be mutating our state.

Now that we have something to manage the state of our lineup, let's flesh out our lineup actions, and hook them into an Angular directive.

With Redux, actions are defined as plain JavaScript objects which contain a type property, and an optional payload property. For more complex logic, such as handling asynchronous calls with promises, you will need to use middleware.

In vanilla Flux, actions are typically wrapped in a function which is responsible for dispatching that action. These are called ActionCreators. In Redux we also have ActionCreators, but instead of dispatching themselves, they simply return an action.

For now, lets take a look at our ActionCreators for the lineup:

```javascript
function joinLine(numberOfPeople) {
  return {
    type: PARTY_JOINED,
    payload: {
      partyId: getNextPartyId(),
      numberOfPeople: numberOfPeople
    }
  };
}

function leaveLine(id) {
  return {
    type: PARTY_LEFT,
    payload: {
      partyId: parseInt(id, 10)
    }
  };
}

export default { joinLine, leaveLine };
```

As you can see, these are simple functions - they take a few paramaters, and return a JSON object with action type and payload properties.

Your ActionCreators are also where your side-effects should be happening - such as generating IDs, or making API calls. This is because we want our reducers/states to be a reflection of 'what has happened', and why we do not want the side effects to be happening there.

Next, let's look at how we get ngRedux working with our controllers so we are able to subscribe to changes in our application state, and dispatch events to trigger updates in our application.

```javascript
import lineupActions from '../../actions/lineup-actions';

export default class LineupController {
  constructor($ngRedux, $scope) {
    function mapStateToParams(state) {
      return {
        parties: state.lineup,
        numberOfPeople: null
      };
    }

    let disconnect = $ngRedux.connect(
      mapStateToParams, // What we want to map to our target
      lineupActions // Actions we want to map to our target
    )(this); // Our target

    $scope.$on('$destroy', disconnect); // Cleaning house
  }
}
```

Breaking this down, the \$ngRedux.connect api expects a callback to be fired every time there is a change to your application state. This should return a plain JSON object that contains the properties from the application state that you care about. In this case - our lineup.

The next paramater that we pass in is our list of actions - \$ngRedux will bind these to your target, so that you can have your actions available to be called from the view.

Finally, we provide the target that we want to map these items onto. In this case we are passing in this since we are using the controllerAs syntax.

With a very few lines of code - we have managed to create a very thin controller that acts as glue between our global application state and the view.

Now, let's take a look at the template that will drive our lineup component:

```html
    <table class="table table-striped table-hover table-condensed">
      <thead>
        <tr>
          <th>Id</th>
          <th>Number Of People</th>
          <th>Remove</th>
        </tr>
      </thead>
      <tbody>
        <tr ng-repeat="party in lineup.parties">
        <!-- Data from our state -->
          <td>{{party.partyId}}</td>
          <td>{{party.numberOfPeople}}</td>
          <td>
            <div class="btn btn-group" role="group">
              <!-- Action we want to dispatch -->
              <button
                type="button"
                class="btn btn-default"
                ng-click="lineup.leaveLine(party.partyId)">X</button>
            </div>
          </td>
        </tr>
      </tbody>
    </table>
    <input ng-model="lineup.numberOfPeople">
    <!-- Action we want to dispatch -->
    <button type="button"
        ng-click="lineup.joinLine(lineup.numberOfPeople)"
        ng-disabled="lineup.numberOfPeople > 4">New Party</button>
    <div ng-show="lineup.numberOfPeople > 4">Can't seat more than 4 people</div>
</div>
```

Since we are using ngRedux to bind our state and actions to our target, we now have the functions that were imported from lineup-actions available to be used.

Load up your browser and navigate to, http://localhost:3000/src/simple-party/ where you can see just the list component on its own.

If you open up your developer tools and look at the console while using the application, you can see the logging middleware outputting some useful information.

![redux action log]('/media/Screenshot-2015-10-21-13-52-37-2.png 'redux action log')

With this, you can see what the state of your application was before an action was fired, what the action was, and the resulting state of your application.
