# Next Metrics

A library for sending metrics to Graphite, that also provides drop in instrumentation for standard parts of Next applications e.g. [Express](https://expressjs.com/).

I'm adding more documentation, blah blah blah.

## Usage

### Getting started

Create an instance of the Metrics object::x

```javascript
const metrics = require('next-metrics');
```

Initialise it:

```javascript
metrics.init({
    app: 'example',
    flushEvery: 5000
});
```

Instrument the response object:

```javascript
app.get('/', function (req, res) {
    metrics.instrument(res, { as: 'express.http.res' });
    res.send('hello');
});
```

To allocate the response's metrics to a separate bucket to all other responses set `res.nextMetricsName = 'name_of_bucket'`

Add a counter for an arbitrary event in the application,

```javascript
var server = app.listen(port, function () {
    metrics.count('express.start', 1);
});
```

See the [example app](./examples/app.js) for more information.

### Configuration

To use this libary you need to set an environment variable named
`FT_GRAPHITE_APP_UUID`. This library will automatically pick up that
environment variable and use it to authenticate with FT's internal
Graphite server when sending metrics.

This library will only send metrics when it is running in production
(`NODE_ENV=production`).

If you don't want to send metrics from an app in production, you must explicitly
set the value of `FT_GRAPHITE_APP_UUID` to `false`.

_Note: Don't use the production FT Graphite API key on your `localhost` as you will fill up FT's internal Graphite server with your local data!_

The `Metrics.init` method takes the following options:

* `flushEvery` (required) - `integer|boolean` - Specify how frequently you want metrics pushed to Graphite, or `false` if you want to do it manually with `.flush()`
* `forceGraphiteLogging` (optional) - `boolean` - Set to `true` if you want to log metrics to Graphite from code running in a non-production environment (when `NODE_ENV != production`)
* `instance` (optional, default: dynamically generated string) - `string|boolean` - Specify a custom instance name in the [Graphite key](#metrics), or set to `false` to omit it
* `useDefaultAggregators` (optional, default: true) - `boolean` - Set to `false` if you want to disable default aggregators
* [DEPRECATED] `app` (required) - `string` - Application name e.g. router
* [DEPRECATED] `platform` (optional, default: heroku) - `string` - Specify a custom platform name in the [Graphite key](#metrics)

### Checking configuration

Configuration errors are logged using [`n-logger`](https://github.com/Financial-Times/n-logger).
It depends on your app configuration, but in most cases, for an app running
in production the logs will be sent to Splunk.

The `Metrics` class exposes a `hasValidConfiguration` boolean property which
you can use to determine if an instance of `Metrics` is correctly configured
to talk to FT Graphite. You might find it useful to check this property
after calling the `Metrics.init` method. See '[Custom use cases](#custom-use-cases)'
for more information on the `Metrics` class.

### Custom use cases

Typically you'll only want a single instance of the [`Metrics`](https://github.com/Financial-Times/next-metrics/blob/master/lib/metrics.js)
class to be used by your application. Because of this, when you
require `next-metrics`, the default export from the module is an
instance of [`Metrics`](https://github.com/Financial-Times/next-metrics/blob/master/lib/metrics.js),
which effectively acts as a singleton.

If you have a custom use case, this module exposes a couple of internal
classes that might help you out:

```javascript
// Create your own instance of Metrics
const { Metrics } = require('next-metrics');

const metrics = new Metrics;

metrics.init({
    platform: 'custom-platform',
    app: 'some-app',
    instance: false,
    useDefaultAggregators: false,
    flushEvery: false,
    forceGraphiteLogging: true
});

metrics.count('some_filename.size', 2454589);
metrics.count('some_filename.gzip_size', 45345);

metrics.flush();

// Send raw metrics directly to a Graphite server
const { GraphiteClient } = require('next-metrics');

const graphite = new GraphiteClient({
    destination: {
        port: 2003,
        host: 'some.host.com'
    },
    prefix: 'some_prefix.',
    noLog: false,
});

graphite.log({
    'build.time': 536,
    'build.count': 1,
});
```

You can also access the [list of Next services](https://github.com/Financial-Times/next-metrics/blob/master/lib/metrics/services.js) that are used for sending
`fetch` metrics to Graphite:

```javascript
const { services } = require('next-metrics');
```

## Instrumentation

The libary _understands_ certain types of objects within our set of
applications. This saves everyone implementing boilerplate metrics code and
avoids different applications inventing their own core measurements.

For example, to instrument an Express response object, put this inside one of
your route handlers:

```javascript
metrics.instrument(res, { as: 'express.http.res' });
```

The first argument is the object you want to instrument, and the second
argument specifies what type of object it is.

## Note about unit tests and Node v6.x

The unit tests must be run under Node v6.x.x due to the dev dependency of
`mitm@1.2.0` which doesn't work on newer versions of Node. The unit tests that
depend on `mitm` need rewriting if we want to use a newer version of `mitm`.

Related `next-metrics` issue: [mitm pinned to 1.2.0 due to failing tests (#111)](https://github.com/Financial-Times/next-metrics/issues/111)

## Metrics

Data is logged in the form of Graphite keys (dots denote hierarchy):

```
<team>.<platform>.<application>.<instance>.<metric>   <value>
```

e.g.

```
next.heroku.ads-api.web_1_process_cluster_worker_1_EU.express.concept_GET.res.status.200.time.sum 325.6
next.heroku.ads-api.web_1_process_cluster_worker_1_EU.system.process.mem_process_heapUsed 16213144
```

You can view data in [Graphite](http://graphite.ft.com/), or in a more user-friendly UI through [Grafana](http://grafana.ft.com).
