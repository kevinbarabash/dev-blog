---
layout: post
title:  "Continuous Integration for ES6"
date:   2014-12-12 14:16:23
categories: ES6
---

ES6 is the next version of JavaScript.  It adds a lot of new and exciting
features many of which have already been implemented in your favorite browser.
[Kangax] maintains a compatibility table which shows which browsers have what
implemented.  If you look at raw percentage of features, IE is (surprisingly)
in the lead.

I've been working on a project, a JavaScript debugger, which makes use of one
of these new features, generators.  Unfortunately, this is one of the features
that can't be polyfilled so my debugger only works in Chrome 39+ and Firefox 26+.
This means that I can't use phantomjs for automated testing.  I have to use a
real browser.

## Testing With a Real Browser ##

In the past I would've turned to [karma], but I've had too many issues getting
karma to work reliably.  I decided to use [testee.js] instead. It's very simple
to get up and running.  The best thing about it is that there's no configuration
and you can use it with popular test frameworks such as jasmine, mocha, and qunit.

- npm install testee
- testee test/runner.html

runner.html is the same test harness that you'd open up in my browser to run the
tests manually.  testee.js also allows you to specify which browser(s) you want
to run the tests on.  Please refer to the website for more info.  One gotcha
that I ran into was I was just running mocha in a script file at the end of the
runner.  You'll need to start your tests from inside an `onload` handler like
this:

    window.onload = function () {
        mocha.run();
    };

The reason for this is that testee.js injects code which modifies the testing
framework to report results to a local server via websockets.  If the tests
start running before this happens you get no results in the shell.

## Automating With Travis-CI ##

It's super easy to get started with [travis-ci] and if your tests run using
phantomjs it just works.  Running tests in a real browser on travis-ci requires
a little more elbow grease but not much.

The nice thing about travis-ci is that everything you need to run a real browser
is part of the default ubuntu environment:

- firefox (you can also specify older versions if you need)
- xvfb (virtual framebuffer - lets you run X programs without a monitor)

Here's a minimal __.travis.yml__ config file to get things up and running:

    language: node_js
    node_js:
      - "0.10"
    before_install:
      - "export DISPLAY=:99.0"
      - "sh -e /etc/init.d/xvfb start"
    script: npm test

Here's parts of the corresponding __package.json__:

    {
      ...
      "scripts": {
        "test": "./node_modules/.bin/testee test/runner.html --browsers firefox"
      },
      ...
      "devDependencies": {
        ...
        "testee": "^0.1.7"
      }
    }

For those who haven't used travis-ci before, it automatically runs `npm install`
before it runs the script(s) specified in your __.travis.yml__.  I've used the
the `--browsers` flag to specify that the tests should be run using Firefox
instead of the default which is phantomjs.

## Next Steps ##

Every browser is different so I'd like to be able to test on more browsers.
travis-ci allows you to install ubuntu packages so Chromium shouldn't be too
hard to get working.


[Kangax]:       http://kangax.github.io/compat-table/es6/
[karma]:        http://karma-runner.github.io/0.12/index.html
[testee.js]:    http://daffl.github.io/testee.js/
[travis-ci]:    https://travis-ci.org/
