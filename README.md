# PHP DataDog StatsD Client

[![Build Status](https://travis-ci.org/DataDog/php-datadogstatsd.svg?branch=master)](https://travis-ci.org/DataDog/php-datadogstatsd)

This is an extremely simple PHP [datadogstatsd](http://www.datadoghq.com/) client.
Requires PHP >= 5.6.0.

See [CHANGELOG.md](CHANGELOG.md) for changes.

*For a Laravel-specific implementation that wraps this library, check out [laravel-datadog-helper](https://github.com/chaseconey/laravel-datadog-helper).*

## Installation

### Composer

Add the following to your `composer.json`:

```
"datadog/php-datadogstatsd": "1.4.*"
```

Note: The first version shipped in composer is 0.0.3

### Or manually

Clone repository at [github.com/DataDog/php-datadogstatsd](https://github.com/DataDog/php-datadogstatsd)

Setup: `require './src/DogStatsd.php';`

## Usage

### instantiation

To instantiate a DogStatsd object using `composer`:

```php
require __DIR__ . '/vendor/autoload.php';

use DataDog\DogStatsd;
use DataDog\BatchedDogStatsd;

$statsd = new DogStatsd();
$statsd = new BatchedDogStatsd();
```

DogStatsd constructor, takes a configuration array. The configuration can take any of the following values (all optional):

- `host`: the host of your DogStatsd server, if not set tries to get it from the `DD_AGENT_HOST` environment variable, default to `localhost`
- `port`: the port of your DogStatsd server, if not set tries to get it from the `DD_DOGSTATSD_PORT` environment variable, default to `8125`
- `socket_path`: the path to the DogStatsd UNIX socket (overrides `host` and `port`, only supported with `datadog-agent` >= 6). default to `null`
- `global_tags`: tags to apply to all metrics sent, the `dd.internal.entity_id` tag is appended to `global_tags` from the `DD_ENTITY_ID` environment variable

When sending `events` over TCP the following options can be set (see [Events section](#submitting-events)):

- `api_key`: needed to send `event` over TCP
- `app_key`: needed to send `event` over TCP
- `curl_ssl_verify_host`: Config pass-through for `CURLOPT_SSL_VERIFYHOST` defaults 2
- `curl_ssl_verify_peer`: Config pass-through for `CURLOPT_SSL_VERIFYPEER` default 1
- `datadog_host`: where to send events default `https://app.datadoghq.com`

### Tags

The 'tags' argument can be a array or a string. Value can be set to `null`.

```php
# Both call will send the "app:php1" and "beta" tags.
$statsd->increment('your.data.point', 1, array('app' => 'php1', 'beta' => null));
$statsd->increment('your.data.point', 1, "app:php1,beta");
```

### Increment

To increment things:

``` php
$statsd->increment('your.data.point');
$statsd->increment('your.data.point', .5);
$statsd->increment('your.data.point', 1, array('tagname' => 'value'));
```

### Decrement

To decrement things:

``` php
$statsd->decrement('your.data.point');
```

### Timing

To time things:

``` php
$start_time = microtime(true);
run_function();
$statsd->microtiming('your.data.point', microtime(true) - $start_time);

$statsd->microtiming('your.data.point', microtime(true) - $start_time, 1, array('tagname' => 'value'));
```

### Submitting events

For documentation on the values of events, see
[http://docs.datadoghq.com/api/#events](http://docs.datadoghq.com/api/#events).

#### Submitting events via TCP vs UDP

- **TCP** - High-confidence event submission. Will log errors on event submission error.
- **UDP** - "Fire and forget" event submission. Will **not** log errors on event submission error. No acknowledgement of submitted event from Datadog.

No matter wich transport is used the `event` function has the same API.

_[Differences between TCP/UDP](http://stackoverflow.com/a/5970545)_

##### UDP Submission to local dogstatsd

Since the UDP method uses the a local dogstatsd instance we don't need to setup any additional application/api access.

```php
$statsd = new DogStatsd();
$statsd->event('Fire and forget!',
    array(
        'text'       => 'Sending errors via UDP is faster but less reliable!',
        'alert_type' => 'success'
    )
);
```

- Default method
- No configuration
- Faster
- Less reliable
- No logging on communication errors with Datadog (fire and forget)

##### TCP Submission to Datadog API

To submit events via TCP, you'll need to first configure the library with your
Datadog credentials, since the event function submits directly to Datadog
instead of sending to a local dogstatsd instance.

You can find your api and app keys in the [API tab](https://app.datadoghq.com/account/settings#api).

```php
$statsd = new DogStatsd(
    array('api_key' => 'myApiKey',
          'app_key' => 'myAppKey',
     )
  );

$statsd->event('A thing broke!',
    array(
        'alert_type'      => 'error',
        'aggregation_key' => 'test_aggr'
    )
);
$statsd->event('Now it is fixed.',
    array(
        'alert_type'      => 'success',
        'aggregation_key' => 'test_aggr'
    )
);
```

- Slower
- More reliable
- Logging on communication errors with Datadog (uses cURL for API request)
- Logs via error_log and try/catch block to not throw warnings/errors on communication issues with API

### Origin detection over UDP in Kubernetes

Origin detection is a method to detect which pod DogStatsD packets are coming from in order to add the pod's tags to the tag list.

To enable origin detection over UDP, add the following lines to your application manifest
```yaml
env:
  - name: DD_ENTITY_ID
    valueFrom:
      fieldRef:
        fieldPath: metadata.uid
```
The DogStatsD client attaches an internal tag, `entity_id`. The value of this tag is the content of the `DD_ENTITY_ID` environment variable, which is the pod’s UID.
The agent uses this tag to infer packets' origin, and tag their metrics accordingly.

## Roadmap

- Add a configurable timeout for event submission via TCP
- Write unit tests
- Document service check functionality

## Contributing

### Tests

```bash
composer test
```
