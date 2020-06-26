---
title: Executors
excerpt: ''
hideFromSidebar: false
---

k6 v0.27.0 introduces the concept of _executors_: configurable modes of executing a
JavaScript function that can model diverse traffic scenarios in load tests.

Multiple executors can be used in the same script, they can be scheduled to run
parallel or in sequence, and each executor can run a different JS function with
different environment variables and tags, so this brings a lot of flexibility with
organizing and modeling testing scenarios.

Existing global execution options such as `vus`, `duration` and `stages` were
formalized into standalone executors, while new executors were added to support more
advanced modes of execution. A major benefit of this restructuring is that support
for new testing scenarios can be added relatively easily to k6, making it very
extensible.

> ### ⚠  Backwards compatibility
>
> Note that pre-v0.27.0 scripts and options should continue to work
> the same as before (with a few minor breaking changes, as mentioned
> in the [release notes](https://github.com/loadimpact/k6/releases/tag/v0.27.0))
> but please [create a bug issue](https://github.com/loadimpact/k6/issues/new?labels=bug&template=bug_report.md)
> if that's not the case.


## Executor types


### Shared iterations

A fixed number of iterations are "shared" by all VUs, and the test ends once all
iterations are executed. This executor is equivalent to the global `vus` and
`iterations` options.

| Option        | Type    | Description                                                                    | Default |
|---------------|---------|--------------------------------------------------------------------------------|---------|
| `vus`         | integer | Number of VUs to run concurrently.                                             | `1`     |
| `iterations`  | integer | Total number of script iterations to execute across all VUs.                   | `1`     |
| `maxDuration` | string  | Maximum test duration before it's forcibly stopped (excluding `gracefulStop`). | `"10m"` |


### Constant VUs

A fixed number of VUs execute as many iterations as possible for a specified amount
of time. This executor is equivalent to the global `vus` and `duration` options.

| Option     | Type    | Description                                     | Default |
|------------|---------|-------------------------------------------------|---------|
| `vus`      | integer | Number of VUs to run concurrently.              | `1`     |
| `duration` | string  | Total test duration (excluding `gracefulStop`). | -       |


### Ramping VUs

A variable number of VUs execute as many iterations as possible for a specified
amount of time. This executor is equivalent to the global `stages` option.

| Option             | Type    | Description                                                                   | Default |
|--------------------|---------|-------------------------------------------------------------------------------|---------|
| `startVUs`         | integer | Number of VUs to run at test start.                                           | `1`     |
| `stages`           | array   | Array of objects that specify the target number of VUs to ramp up or down to. | `[]`    |
| `gracefulRampDown` | string  | Time to wait for iterations to finish before starting new VUs.                | `"30s"` |


### Externally controlled

Control and scale execution at runtime via k6's REST API or the CLI.

This executor formalizes what was [previously possible globally](https://k6.io/blog/how-to-control-a-live-k6-test),
and using it is required in order to use the `pause`, `resume`, and `scale` CLI commands.

| Option     | Type    | Description                                         | Default |
|------------|---------|-----------------------------------------------------|---------|
| `vus`      | integer | Number of VUs to run concurrently.                  | -       |
| `duration` | string  | Total test duration.                                | -       |
| `maxVUs`   | integer | Maximum number of VUs to allow during the test run. | -       |


### Per VU iterations

Each VU executes a fixed number of iterations.

| Option        | Type    | Description                                                                    | Default |
|---------------|---------|--------------------------------------------------------------------------------|---------|
| `vus`         | integer | Number of VUs to run concurrently.                                             | `1`     |
| `iterations`  | integer | Number of script iterations to execute with each VU.                           | `1`     |
| `maxDuration` | string  | Maximum test duration before it's forcibly stopped (excluding `gracefulStop`). | `"10m"` |


### Constant arrival rate

A fixed number of iterations are executed in a specified period of time. This allows
k6 to dynamically change the amount of VUs during a test run to achieve the specified
amount of iterations per period, which is useful for a more accurate representation
of RPS, for example. See [issue #550](https://github.com/loadimpact/k6/issues/550) for details.
<!-- Should we instead link to an article about open/closed testing models? -->

| Option            | Type    | Description                                                                             | Default |
|-------------------|---------|-----------------------------------------------------------------------------------------|---------|
| `rate`            | integer | Number of iterations to execute each `timeUnit` period.                                 | -       |
| `timeUnit`        | string  | Period of time to apply the `rate` value.                                               | `"1s"`  |
| `duration`        | string  | Maximum test duration before it's forcibly stopped (excluding `gracefulStop`).          | -       |
| `preAllocatedVUs` | integer | Number of VUs to pre-allocate before test start in order to preserve runtime resources. | -       |
| `maxVUs`          | integer | Maximum number of VUs to allow during the test run.                                     | -       |


### Ramping arrival rate

A variable number of iterations are executed in a specified period of time. This is
similar to the ramping VUs executor, but for iterations instead, and k6 will attempt
to dynamically change the number of VUs to achieve the configured iteration rate.

| Option            | Type    | Description                                                                             | Default |
|-------------------|---------|-----------------------------------------------------------------------------------------|---------|
| `startRate`       | integer | Number of iterations to execute each `timeUnit` period at test start.                   | `0`     |
| `timeUnit`        | string  | Period of time to apply the `startRate` the `stages` `target` value.                    | `"1s"`  |
| `stages`          | array   | Array of objects that specify the target number of iterations to ramp up or down to.    | `[]`    |
| `preAllocatedVUs` | integer | Number of VUs to pre-allocate before test start in order to preserve runtime resources. | -       |
| `maxVUs`          | integer | Maximum number of VUs to allow during the test run.                                     | -       |


## Shared configuration

All executors share a base configuration, so the following options can be used for all:

| Option         | Type   | Description                                                                      | Default     |
|----------------|--------|----------------------------------------------------------------------------------|-------------|
| `startTime`    | string | Time offset since test start this scenario should begin execution.               | `"0s"`      |
| `gracefulStop` | string | Time to wait for iterations to finish executing before stopping them forcefully. | `"30s"`     |
| `exec`         | string | Name of exported JS function to execute.                                         | `"default"` |
| `env`          | object | Environment variables specific to this scenario.                                 | `{}`        |
| `tags`         | object | [Tags](/using-k6/tags-and-groups) specific to this scenario.                     | `{}`        |


## Examples

- Execute a constant 200 RPS for 1 minute, allowing k6 to dynamically schedule up to 100 VUs.

Note that in order to reliably achieve a fixed request rate, it's recommended to keep
the function being executed very simple, with preferably only a single request call,
and no additional processing or `sleep()` calls.

<div class="code-group" data-props='{"labels": [ "constant-arr-rate.js" ], "lineNumbers": "[true]"}'>

```js
import http from 'k6/http';

export let options = {
  discardResponseBodies: true,
  scenarios: {
    constant_arr_rate: {  // arbitrary scenario name
      executor: 'constant-arrival-rate',
      rate: 200,
      timeUnit: '1s',
      duration: '1m',
      preAllocatedVUs: 50,
      maxVUs: 100,
    }
  }
};

export default function() {
  http.get('https://test.k6.io/');
}
```

</div>


- Execute a variable RPS test, starting at 50, ramping up to 200 and then back to 0,
  over a 1 minute period:

<div class="code-group" data-props='{"labels": [ "ramping-arr-rate.js" ], "lineNumbers": "[true]"}'>

```js
import http from 'k6/http';

export let options = {
  discardResponseBodies: true,
  scenarios: {
    ramping_arrival: {  // arbitrary scenario name
      executor: 'ramping-arrival-rate',
      startRate: 50,
      timeUnit: '1s',
      preAllocatedVUs: 50,
      maxVUs: 100,
      stages: [
        { target: 200, duration: '30s' },
        { target: 0, duration: '30s' },
      ],
    }
  }
};

export default function() {
  http.get('https://test.k6.io/');
}
```

</div>


- Run as many iterations as possible with 50 VUs for 30s, and then run 100 iterations
  per VU of another scenario.

Note the use of `startTime`, and different `exec` functions for each scenario.

<div class="code-group" data-props='{"labels": [ "multiple-scenarios.js" ], "lineNumbers": "[true]"}'>

```js
import http from 'k6/http';

export let options = {
  discardResponseBodies: true,
  scenarios: {
    const_vus: {
      executor: 'constant-vus',
      exec: 'scenario1',
      vus: 50,
      duration: '30s',
    },
    per_vu_iters: {
      executor: 'per-vu-iterations',
      exec: 'scenario2',
      vus: 50,
      iterations: 100,
      startTime: '30s',
      maxDuration: '1m',
    },
  }
};

export function scenario1() {
  http.get('https://test.k6.io/', { tags: { scenario: '1' }} );
}

export function scenario2() {
  http.get('https://test.k6.io/', { tags: { scenario: '2' }} );
}
```

</div>

- Use different environment variables and tags per scenario.

In the previous example we set tags on individual HTTP request metrics, but this
can also be done per scenario, which would apply them to other
[taggable](https://k6.io/docs/using-k6/tags-and-groups#tags) objects as well.

<div class="code-group" data-props='{"labels": [ "multiple-scenarios-env-tags.js" ], "lineNumbers": "[true]"}'>

```js
import http from 'k6/http';
import { fail } from 'k6';

export let options = {
  discardResponseBodies: true,
  scenarios: {
    const_vus: {
      executor: 'constant-vus',
      exec: 'scenario1',
      vus: 50,
      duration: '30s',
      tags: { scenario: '1' },
      env: { SCENARIO: '1' },
    },
    per_vu_iters: {
      executor: 'per-vu-iterations',
      exec: 'scenario2',
      vus: 50,
      iterations: 100,
      startTime: '30s',
      maxDuration: '1m',
      tags: { scenario: '2' },
      env: { SCENARIO: '2' },
    },
  }
};

export function scenario1() {
  if (__ENV.SCENARIO != '1') fail();
  http.get('https://test.k6.io/');
}

export function scenario2() {
  if (__ENV.SCENARIO != '2') fail();
  http.get('https://test.k6.io/');
}
```

</div>
