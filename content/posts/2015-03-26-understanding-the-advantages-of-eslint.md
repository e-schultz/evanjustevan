---
template: post
title: Understanding the Real Advantages of Using ESLint
slug: understanding-the-real-advantages-of-using-eslint
draft: false
date: 2015-03-26T16:49:00.000Z
description: Don't become a human linter, let the tools help you out
socialImage: /media/es-lint-1.jpg
category: Development
tags:
  - ESLint
  - JavaScript
  - Web Development
---

![eslint logo](/media/es-lint-1.jpg 'Understanding the Real Advantages of Using ESLint')

# Introduction

I have been a fan of JavaScript for a number of years now. It's a dynamic and flexible language which gives it a great deal of power. However, unlike compiled languages it is easy for syntax errors and accidental globals to creep into your code without realizing it until you actually try and run the code.

This is why I have made linting an important part of my daily workflow. Either as a gentle reminder in my editor of choice while working, and keeping an eye on me with every save. Or as part of my build process by notifying me of possible errors and bad practices that could cause unexpected results or crashes.

I'm sure most developers that have had to support older versions of IE have been caught by the 'dangling comma' issue that can cause your application to stop dead in its tracks. Even the most mindful developer can let things like this creep in without noticing. Having a tool to help keep you in check can be an invaluable asset.

Over the years I have used a few linting tools. Trying to appease the overly strict JSLint, to finding a comfortable balance with JSHint, which allowed for some configuration of rules, and the ability to enable or disable them.

Recently, I was introduced to ESLint, a new JavaScript linting and style checking tool created by Nicholas C. Zakas. It was first released in 2013, but I hadn't heard about it until recently. I was curious to know more. JSHint was serving my needs, so I wondered what ESLint could offer that other tools couldn't? It turns out, quite a bit.

# ESLint

ESlint was first released in 2013, by Nicholas C. Zakas with the philosophy of having a linting tool where everything was pluggable, every rule is stand-alone, and it's agenda-free in that it doesn't enforce any particular coding style.

There are several features that appealed to me in ESLint that prompted me to make the switch:

- Robust set of default rules covering all of the rules that exist in JSLint and JSHint, making a migration to this tool fairly easy.
- Configurable rules - including error levels, allowing you to decide what is a warning, error, or simply disabled.
- Rules for style checking, which can help keep the code format consistent across teams.
- The ability to write your own plugins and rules.

# Getting started

Getting set up with ESLint is pretty straight forward, and there are a number of ways to configure it and integrate it into your workflow. I won't go into a deep dive of configuration as [The ESLint Documentation](https://eslint.org/docs/user-guide/configuring) covers this in depth, but I will cover a few quick points that gave me a snag of 'how do I do this...' when moving from JSHint.

Configuration
Configuring ESLint is easy. You are able to have a .eslintrc file, have an eslint field in your package.json, or do per-file configurations within the file you are linting.

An example .eslintrc file:

```
{
  "env": {
    "browser": true,
  },
  "globals": {
    "angular": true,
  },
  "rules": {
    "camelcase": 2,
    "curly": 2,
    "brace-style": [2, "1tbs"],
    "quotes": [2, "single"],
    "semi": [2, "always"],
    "space-in-brackets": [2, "never"],
    "space-infix-ops": 2,
  }
}
```

If this file lives in the root of your project, it is the set of rules that will be applied across the project. If ESLint finds another file in a sub-folder, those rules will be applied instead. This allows you to have custom sets of rules for unit tests, server side code, and client side code.

One of the first things I needed to do when making the transition was to figure out how to disable or modify a rule for just a section of code. Maybe it's a section of code that you have a good reason for disabling the rule.

ESLint provides a few ways to do in-file configuration, that will override the settings found in the configuration files.

## Disable Everything

```javascript
/* eslint-disable */

var obj = { key: 'value' }; // I don't care about IE8

/* eslint-enable */
```

## Disable a rule

```javascript
/*eslint-disable no-alert */

alert('doing awful things');

/* eslint-enable no-alert */
```

## Or just tweak a rule

```javascript
/* eslint no-comma-dangle:1 */
// Make this just a warning, not an error
var obj = { key: 'value' };
```

# Getting into the flow

In my opinion, you should lint early, and lint often. I remember my first time trying to run JSLint on an older project. It choked out at 25 errors, and fixing those errors caused 25 more to appear, before I hung my head in shame and gave up on linting for awhile.

The earlier you are able to lint, the better. Having it be something that is just part of your workflow and not a chore that you clean up after-the-fact can help ensure that it's always done, instead of becoming something that will always be done later.

Thankfully, ESLint has a number of integrations for many of the popular editors, IDEs and build systems:

Integrations

- [Sublime Text 3](https://github.com/SublimeLinter/SublimeLinter-eslint)
- [VIM](https://github.com/vim-syntastic/syntastic/tree/master/syntax_checkers/javascript)
- EMacs - [Flycheck](https://github.com/flycheck/flycheck)
- [Atom](https://atom.io/packages/linter-eslint)
- [.. And more](https://eslint.org/docs/user-guide/integrations)

## Build Systems

Most popular build systems such as Grunt, Gulp, Mimosa, Broccoli and Webpack, have plugins available for ESLint.

Setting up a gulp task for ESLint is pretty straight forward:

```javascript
var gulp = require('gulp');
var eslint = require('gulp-eslint');

gulp.task('lint', function() {
  return gulp
    .src('client/app/**/*.js')
    .pipe(eslint())
    .pipe(eslint.format());
});
```

By having your editor set up to check a file on save, you can get a reminder about linting issues every time you save a file, and correcting things as you go becomes second nature.

# Style Checking

Like a research paper with many authors, a consistency in the writing style and formatting can make things easier to understand. When working on a team, having everyone adhere to an agreed upon style guide can have a similar affect. When combined with automatic formatting tools such as JSPrettify, a style guide can make it easier to merge changes and spot the real differences in files.

The difference between:

```
function somethingImportant() { }
```

and

```
function somethingImportant () { }
```

is not important, but the function body itself is. Having diffs littered with minor changes like this can make it more difficult to do code review, perform merges and be the source of conflicts that could have been avoided in the first place.

By being able to align your automatic formatting settings with a set of style rules in ESLint, you can help ensure that style issues are not a source of conflict within the team.

# Custom Rules

One of the other features of ESLint that caught my eye is the ability to implement our own custom rules. Even the default rules with ESLint use the same plugin architecture that we can use to write our own rules.

As your team comes up with best practices, style guidelines, and conventions, you have the potential to write a custom rule to help solidify the practice within the project.

I have been working with AngularJS for a few years now and over time the community has come up with various style guides and best practices. There is a set of [custom rules designed for AngularJS](https://github.com/EmmanuelDemey/eslint-plugin-angular), which has rules for common best practices such as:

- Using Angular element instead of \$ or jQuery
- Using controllerAs syntax
- Specifying a prefix to be used for directives, and the prefix to be used.

This is only scratching the surface of the extra features that ESLint has, and this has played a large part in why I have been excited to adopt it in my recent projects.
