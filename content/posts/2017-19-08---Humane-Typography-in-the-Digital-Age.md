---
template: post
title: Are Your Unit Tests Failing for the Expected Reasons?
slug: are-your-unit-tests-failing-for-the-expected-reasons-4n26
draft: false
date: 2020-01-01T23:40:00.000Z
description: >-
  Unit tests can be an invaluable tool in the developers toolbox. You don't need
  to be a strict TDD purist to make unit testing worthwhile. Once you get into
  the flow of writing tests, it can be rather satisfying to watch the Nyan Cat
  Reporter go across your screen as the number of tests passing increase
category: JavaScript
tags:
  - JavaScript
  - Web Development
  - Testing
socialImage: '/media/ths7adeebulbxhk3pp6z.gif'
---

![check box image](/media/ths7adeebulbxhk3pp6z.gif 'Are Your Unit Tests Failing for the Expected Reasons?')

Unit tests can be an invaluable tool in the developers toolbox. You don't need to be a strict TDD purist to make unit testing worthwhile. Once you get into the flow of writing tests, it can be rather satisfying to watch the Nyan Cat Reporter go across your screen as the number of tests passing increase.

As with any other tool though, it can be misused, and not always provide the benefit that you want or expect.

The other day I was doing some code clean up, and came across a test that started to make me ask questions instead of having them answered.

Some of the questions that a unit test should be answering:

- What is it testing?
- What is it doing?
- What is expected behavior?
- What is the actual behavior?
- Does it pass or fail for the expected reasons?

The test that made me go 'hrmm?' looked something like this:

```javascript
/// .... mock removed for now

it('should call /location/calculate-distance correctly', () =>
  locationApi.calculateDistance(10, 20, 30, 40).then(res => {
    expect(res.distance).to.equal('ok');
  }));
```

Reading this unit test, it's not really clear what is actually being tested. The it block is saying that it should call `/location/calculate-distance` correctly, but the expect at the bottom is looking at the response.

Currently this test is passing, but it's not passing for a reason that the test states it should be. Yes, there is something responding, but response.distance being ok has nothing to do with the behavior we wanting to verify.

This application was using [ismorphic-fetch](https://github.com/matthew-andrews/isomorphic-fetch) and [fetch-mock](https://www.npmjs.com/package/fetch-mock) to mock out HTTP requests. The mock for this test looks like:

```javascript
before(() => {
  fetchMock.mock(
    (url, options) =>
      url ===
        [
          LOCATION_ENDPOINT,
          'calculate-distance?&lat1=10&long1=20&lat2=30&long2=40'
        ].join('/') && options.method === 'GET',
    {
      body: JSON.stringify({ distance: 'ok' }),
      status: 200,
      headers: { 'content-type': 'application/json' }
    }
  );
});
```

This starts to give a bit more insight, and after digging around that was going on in `locationApi.calculateDistance` - it seems like this test is wanting to verify that for a given set of parameters, the URL is formed up correctly to query the API to calculate the distance.

When the test runs, it currently passes. If it fails though, am I getting useful information? If I tweak the location code in how it forms the URL, the errors that get reported look like:

```shell
1) api/location should call /location/calculate-distance correctly:
     Error: only absolute urls are supported
      at node_modules/node-fetch/index.js:54:10
      at new Fetch (node_modules/node-fetch/index.js:49:9)
      at Fetch (node_modules/node-fetch/index.js:37:10)
      at module.exports (node_modules/isomorphic-fetch/fetch-npm-node.js:8:19)
      at FetchMock.fetchMock (node_modules/fetch-mock/src/fetch-mock.js:265:17)
      at exports.default (src/api/helpers/composer-request.js:14:12)
      at Object.exports.getTaxEstimate (src/api/location-api.js:8:10)
      at Context.<anonymous> (src/api/location-api.test.js:27:13)
```

And hidden away at the top of the unit tests running and easy to miss, is:

```shell
api/location
unmatched call to /location!calculate-distance?lat1=10&long1=20&lat2=30&long2=40
```

This isn't terribly useful information. The error that is getting reported doesn't tell me anything about what the expected and actual results were.

> Error: only absolute urls are supported

This is an error thrown by fetch-mock before our expect statements have even been hit. There is a hint at the top of the unit test reports where fetch-mock will complain about an unmatched call.

This message is easy to miss, and it also requires the person reading the unit test results to understand a bit of how fetch-mock works get pointed in the right direction.

For a seemingly simple unit test, it asks more questions than it answers.

A more accurate description of this test would be:

```javascript
describe('fetch-mock behavior', () => {
  it('should return the object I told it to if no error is thrown', () => {});
});
```

If we were the authors of [fetch-mock](https://www.npmjs.com/package/fetch-mock), that could possibly be a useful test. But, we are wanting to write unit tests for the system we are building, not for the mocking frameworks we are using.

The fact that `expect(res.distance).to.equal('ok');` is more of a coincidence than the behavior you want to test.

What can we do to make it clear what the intended purpose of this test is, and that it provides meaningful errors when it fails?

Let's revisit the questions we asked at the start, and clean up the test to start answering them.

## What is it testing?

When the `calculateDistance` is called, then an API is called with a specific URL and query parameters. The response object is inconsequential here. Let's adjust the unit test to start being more descriptive and accurate.

```javascript
it('should call calculate-distance with correct query parameters', () => {
  //
});
```

The 'it' sentence starts to describe what we are doing without needing to read the test body.

## What is it doing?

The initial test wasn't that bad at answering this one, it's calling the `locationApi.calculateDistance`,

```javascript
it('should call calculate-distance with correct query parameters', () => {
  return locationApi.calculateDistance(10, 20, 30, 40).then(() => {
    // ..
  });
});
```

## What is the expected / actual behavior

This is where the previous test started to fail at answering these questions.

We don't care if the response object has a distance of 'ok', we want to verify the URL that is being hit.

In answering this question, we can state what the expected and actual results are.

```javascript
it('should call calculate-distance with correct query parameters', () => {
  const EXPECTED_URL = `${LOCATION_ENDPOINT}/calculate-distance?lat1=10&long1=20&lat2=30&long2=40`;
  return locationApi.calculateDistance(10, 20, 30, 40).then(() => {
    const ACTUAL_URL = fetchMock.lastUrl();
    expect(EXPECTED_URL).to.equal(ACTUAL_URL);
  });
});
```

Now when reading the unit test, the expected reasons for passing/failing are more obvious. The thing that I am asserting is reflective of the behavior that I want.

However, if the test fails, the output still isn't very useful and still complains about: `Error: only absolute urls are supported`

This leads to answering the final question:

## Does it pass or fail for the expected reasons?

Currently fetch-mock is throwing an error before we even hit our expect statement, so we can't guarantee that things are passing or failing for the reasons we expect.

Adjusting our mock is pretty straightforward. Instead of having the mock be very specific for a URL, I am adjusting it to match just about anything that begins with a slash.

```javascript
before(() =>
  fetchMock.mock(`^/`, {
    body: JSON.stringify({ data: 'ok' }),
    status: 200,
    headers: { 'content-type': 'application/json' }
  })
);
```

When we run our test and if it fails, the error that gets reported is now:

```shell
1) should call calculate-distance with correct query parameters:

      AssertionError: expected '/location!calculate-distance?lat1=10&long1=20&lat2=30&long2=40' to equal '/location/calculate-distance?lat1=10&long1=20&lat2=30&long2=40'
      + expected - actual

      -/location!calculate-distance?lat1=10&long1=20&lat2=30&long2=40
      +/location!calculate-distance?lat1=10&long1=20&lat2=30&long2=40

      at src/api/location-api.test.js:21:29
      at process._tickCallback (internal/process/next_tick.js:103:7)
```

With a few minor changes to the unit test and the mock, the test is starting to answer questions, instead of making us ask them.

- The mock has been simplified and is easier to understand.
- The expect statement makes it clear the behavior that we are testing.
- The test fails for the correct reason - actual does not meet expected behavior.
- When the test fails, it is descriptive to what the error is.

Next time you're writing a unit test or reviewing others, maybe double check to ensure that they are passing or failing for the reasons that you expect.

If reading a unit test makes you ask questions, then it could be a sign that you need to clean them up to make them more useful.

---

_initially posted on the [rangle.io blog](https://rangle.io/blog/are-your-unit-tests-failing-for-the-expected-reasons)_
