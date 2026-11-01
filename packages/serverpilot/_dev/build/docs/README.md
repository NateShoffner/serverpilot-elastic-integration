{{- generatedHeader }}
# ServerPilot Integration for Elastic

## Overview

The ServerPilot integration collects and parses the per-app Nginx, Apache and PHP-FPM logs that [ServerPilot](https://serverpilot.io) writes on the servers it manages, and collects server, system user, app and database inventory from the ServerPilot management API.

ServerPilot configures and manages a PHP and WordPress hosting stack on your own Ubuntu server at any cloud provider. It writes each app's logs to its own directory using its own log formats. The stock Nginx and Apache integrations can read those files and will parse the common prefix of each line, but they drop the extra fields ServerPilot appends, leave most of both error formats unparsed inside `message`, and attribute nothing to the app that served the request. No Elastic integration parses PHP-FPM access, error or slow-request logs at all. This integration fills those gaps and tags every event with the app and system user that produced it.

### Compatibility

This integration is compatible with ServerPilot-managed servers running Ubuntu LTS 20.04, 22.04, 24.04 or 26.04, with the ServerPilot stack of Nginx, Apache, PHP-FPM and MySQL 8.0. The API data streams use version 1 of the ServerPilot API at `https://api.serverpilot.io/v1`.

The log data streams do not depend on a particular PHP version. The PHP version is read from each log file's name, so servers running several PHP versions side by side are handled correctly, and apps on an end-of-life PHP release can be identified.

### How it works

The integration has two independent halves, offered as two separate policy templates so you can use either or both.

**ServerPilot App Logs** runs Elastic Agent on the ServerPilot server itself and reads the per-app log files under `/srv/users/SYSUSER/log/APPNAME/`. The default path patterns are globs, so a single agent policy picks up every app on the server automatically, including apps added later. Each event is tagged with `serverpilot.app.name`, `serverpilot.sysuser.name` and `service.name`, derived from the log file's path, which is the only place ServerPilot records which app a log line belongs to.

**ServerPilot API** polls the ServerPilot management API for inventory. These endpoints return a complete snapshot with no incremental mode, so every poll re-reports every object and the default interval is one hour. The agent for this half does not need to run on the ServerPilot server; it only needs outbound HTTPS.

## What data does this integration collect?

Seven log data streams, collected on the ServerPilot server:

| Data stream | Source |
|---|---|
| `nginx_access` | Per-app Nginx access logs, HTTP and HTTPS |
| `nginx_error` | Per-app Nginx error logs |
| `apache_error` | Per-app Apache error logs |
| `php_access` | Per-app PHP-FPM access logs, with per-request duration, peak memory and CPU |
| `php_error` | Per-app PHP error logs, including uncaught exceptions and stack traces |
| `php_slow` | Per-app PHP-FPM slow request logs, as parsed stack traces |

Four inventory data streams, collected from the ServerPilot API:

| Data stream | Endpoint |
|---|---|
| `server` | `GET /servers` |
| `sysuser` | `GET /sysusers` |
| `app` | `GET /apps` |
| `database` | `GET /dbs` |

### Supported use cases

Attribute every request, error and slow page to the app that served it, across every app on a server, from a single agent policy.

Find which PHP function is making a site slow by aggregating on the innermost frame of the slow request log. Track per-request PHP memory and CPU to size a server before it saturates. Identify every app still running an end-of-life PHP release.

Monitor the security posture of each server over time, including whether the firewall, unattended updates and forced HTTPS are enabled, and find apps with no TLS certificate at all.

### What this integration does not collect

ServerPilot runs an otherwise stock Ubuntu stack, so existing integrations already cover the rest and this integration does not duplicate them. The ServerPilot API has no metrics endpoint and no audit or event endpoint, so host metrics and operating system audit activity must come from the integrations below. Install them alongside this one.

| Need | Use | Notes |
|---|---|---|
| CPU, memory, disk, network and process metrics | `system` | Works unmodified. |
| Syslog, SSH and sudo activity, package manager | `system` | Covers ServerPilot's SSH activity and package manager logs. |
| MySQL metrics and error log | `mysql` | Error log is at `/var/log/mysql/error.log`. |
| MySQL slow query log | `mysql` | At `/var/log/mysql/mysql-slow.log`, enabled by default with a 0.1 second threshold. |
| Nginx service log | `nginx` | Override the error log path to `/var/log/nginx-sp/error.log`. ServerPilot writes no server-wide Nginx access log, so leave the access log path empty or the integration will report a missing file. |
| Apache service log | `apache` | Override the error log path to `/var/log/apache-sp/error.log`. |
| PHP-FPM pool and process metrics | `php_fpm` | Requires a status endpoint that ServerPilot does not expose by default. The `php_access` data stream already provides per-request duration, memory and CPU. |
| PHP-FPM pool manager log | `custom_logs` | Glob `/var/log/php*-fpm-sp.log`. Not yet covered by a data stream; this is where `max_children reached` warnings appear. |

The Nginx and Apache service logs above are server-wide startup and configuration errors. They are not the same as this integration's `nginx_error` and `apache_error` data streams, which collect the per-app error logs. Install both, as they answer different questions.

## What do I need to use this integration?

For the log data streams, you need an Elastic Agent installed on the ServerPilot server with read access to `/srv/users/*/log/*/*`. Elastic Agent normally runs as root, which satisfies this.

To run the agent unprivileged instead, grant its user read access using ACLs rather than changing ownership or modes under `/srv/users`, because ServerPilot manages those paths and incorrect permissions there will break your sites:

```bash
sudo setfacl -m u:AGENTUSER:--x /srv
sudo setfacl -m u:AGENTUSER:r-x /srv/users
for h in /srv/users/*/; do sudo setfacl -m u:AGENTUSER:--x "$h"; done
for d in /srv/users/*/log/; do
  sudo setfacl -R  -m u:AGENTUSER:rX "$d"
  sudo setfacl -R -d -m u:AGENTUSER:rX "$d"
done
```

Granting only traversal on each system user's home directory keeps the agent out of `apps/`, which contains application code and database credentials. The `-d` pass sets a default ACL so newly rotated log files inherit the permission. Re-run this after adding an app, because ServerPilot creates the app's log directory at that point.

For the API data streams, you need a ServerPilot API client ID and key.

## How do I deploy this integration?

### Agent-based deployment

Elastic Agent must be installed. For more details, check the Elastic Agent [installation instructions](https://www.elastic.co/guide/en/fleet/current/elastic-agent-installation.html). You can install only one Elastic Agent per host.

Elastic Agent is required to stream data from the syslog or log file receiver and ship the data to Elastic, where the events will then be processed via the integration's ingest pipelines.

### Set up steps in ServerPilot

Only the API half requires configuration in ServerPilot. In the ServerPilot dashboard, go to Account, then API Credentials. Note the Client ID and generate an API Key.

The API key grants full management access to every server on the account, so treat it as a privileged credential. This integration only ever issues read requests.

The log half requires no ServerPilot-side setup, because ServerPilot writes the per-app logs by default.

#### Vendor resources

- [Where to find your log files](https://serverpilot.io/docs/where-to-find-your-log-files)
- [ServerPilot API documentation](https://github.com/ServerPilot/API)
- [What ServerPilot installs](https://serverpilot.io/docs/what-serverpilot-installs)

### Set up steps in Kibana

Add the ServerPilot integration and configure the half or halves you need.

For **ServerPilot App Logs**, assign the policy to the agent running on the ServerPilot server and enable the data streams you want. The default path patterns already match ServerPilot's layout and cover every app on the server, so override them only if you have relocated the log directories. Enable "Preserve original event" to retain the raw log line in `event.original`.

For **ServerPilot API**, enter the Client ID and API Key. The default interval is one hour, which is appropriate because these endpoints return a complete snapshot on every request rather than only what has changed. Lowering the interval multiplies storage without adding information.

### Validation

For the log half, generate a request against one of your apps, then search `logs-serverpilot.nginx_access-*` in Discover. Confirm that documents arrive with `serverpilot.app.name` and `serverpilot.sysuser.name` set to the expected app and system user.

For the API half, search `logs-serverpilot.app-*`. The first collection happens immediately when the policy is applied, so data should appear within about a minute. If it does not, credentials are the most likely cause.

Once either half is flowing, open the **[Logs ServerPilot] App traffic overview** or **[Logs ServerPilot] Inventory and posture** dashboard. They use whatever time range you have selected rather than forcing one of their own, so narrow it to the last 15 minutes while you are confirming that a freshly configured agent is shipping.

## Troubleshooting

- No log data and the agent reports permission errors: the agent cannot read `/srv/users/*/log/*/`. Either run the agent as root or apply the ACLs described above, then confirm with `sudo -u AGENTUSER ls /srv/users/*/log/*/`.
- No log data and no errors: check that the path patterns match your server. ServerPilot's layout is `/srv/users/SYSUSER/log/APPNAME/APPNAME_nginx.access.log`.
- serverpilot.app.name is missing on some events: the file is not under a path matching `/srv/users/SYSUSER/log/APPNAME/`. The event is still collected and fully parsed; only the app attribution is absent.
- API data stream reports an authentication error: ServerPilot returned 401, meaning the Client ID or API Key is wrong. This will not recover on retry and requires the policy to be corrected. Verify the credentials with `curl -u "CLIENTID:APIKEY" https://api.serverpilot.io/v1/servers`.
- Timestamps are offset by several hours on nginx_error, apache_error or php_slow: those three log formats carry no timezone, so the agent's own timezone is used to interpret them. The agent must run on the same server that wrote the logs, which is the normal deployment.
- Each request appears twice: this is expected. A request that reaches PHP is logged by Nginx and again by PHP-FPM, producing a document in both `nginx_access` and `php_access`. Distinguish them with `event.dataset`. Use `nginx_access` for client-facing analysis and `php_access` for per-request PHP cost.
- PHP memory values look far too small: `serverpilot.php.memory.peak.bytes` is in bytes. ServerPilot names the source field `PEAK_MEMORY_MB` and documents it as megabytes, which is incorrect.

## Performance and scaling

The per-app access logs are by far the highest volume data streams; the error and slow request streams are comparatively small. `nginx_access` and `php_access` both carry one document per request that reached PHP, so on a busy server disable whichever you do not query: `nginx_access` is the client's view and the only one covering requests Nginx served without PHP, while `php_access` is the only source of per-request PHP memory and CPU.

This integration does not collect the per-app Apache access log. ServerPilot's Apache is an internal origin behind Nginx, so that log records the same requests as `nginx_access` with strictly less information, and nothing in it is specific to ServerPilot. If you want it anyway, the stock `apache` integration parses it; point it at `/srv/users/*/log/*/*_apache.access.log` and add a `logs-apache.access@custom` ingest pipeline with one grok on `log.file.path` to recover the app and system user:

```
'\A.*/srv/users/(?<serverpilot.sysuser.name>[^/]+)/log/(?<serverpilot.app.name>[^/]+)/[^/]+\Z'
```

The API data streams are negligible in volume but return a full snapshot on each poll, so their storage grows linearly as the interval shortens. One hour is a sensible default.

For more information on architectures that can be used for scaling this integration, check the [Ingest Architectures](https://www.elastic.co/docs/manage-data/ingest/ingest-reference-architectures) documentation.

## Reference

### Inputs used
{{ inputDocs }}

### API usage

These ServerPilot API endpoints are used, all read-only:

* `GET /servers`
* `GET /sysusers`
* `GET /apps`
* `GET /dbs`

The API uses HTTP Basic authentication, with the Client ID as the username and the API Key as the password. No endpoint supports pagination; each returns the complete collection in a single response wrapped in a `data` key. No rate limits are documented, which is why the default poll interval is conservative.

**Secrets are not indexed.** `GET /apps` returns each app's TLS private key and certificate in `ssl.key`, `ssl.cert` and `ssl.cacerts`, and server creation returns a server `apikey`. This integration drops all of those in the CEL program before any event is created, so they never reach Elasticsearch, including inside `event.original`. Only the non-sensitive certificate facts are kept: whether AutoSSL is enabled and whether HTTPS is forced.

There is one exception worth understanding. The **Enable request tracing** option on the `app` data stream writes the raw API request and response to `http-request-trace-*.ndjson` on the Elastic Agent host, and the `GET /apps` response contains the unredacted private keys. The drop happens after tracing, so it does not protect that file, and the `redact.fields` setting cannot help because it redacts CEL state rather than response bodies. Enable request tracing on the `app` data stream only if you accept writing every app's TLS private key to the agent's filesystem in plaintext, and delete the trace files when you are finished.

### Vendor documentation links

- [ServerPilot documentation](https://serverpilot.io/docs/)
- [Where to find your log files](https://serverpilot.io/docs/where-to-find-your-log-files)
- [ServerPilot API](https://github.com/ServerPilot/API)

### Dashboards

Four dashboards ship with the integration. Each one shows only its own data streams, and a navigation bar at the top links to the other three, keeping your time range and filters.

The log dashboards have controls for host, app and other fields. Pick a host first and the app list shows only that host's apps. App names are only unique per server (every server has a `0default` app), so the per-app charts split each app by host and app counts treat the same name on two hosts as two apps. They count `serverpilot.app.qualified_name` (`HOST/APP`), which is only set on data ingested by version 0.2.0 or later.

| Dashboard | Data streams | Shows |
|---|---|---|
| **App traffic overview** | `nginx_access` | Traffic, errors and response times per app, plus top paths and clients. |
| **PHP health** | `php_access`, `php_error`, `php_slow` | PHP errors, slow requests and where they stall, and memory and CPU use per request. |
| **Web server errors** | `nginx_error`, `apache_error` | Nginx and Apache errors by app, severity, path and client. |
| **Inventory and posture** | `server`, `app`, `database`, `sysuser` | What's in the account, which apps run an end-of-life PHP version, and which servers have the firewall, automatic updates or forced HTTPS turned off. |

### Map

A **Client locations** map is installed alongside the dashboards, under **Analytics > Maps**. It clusters `client.geo.location` by request count across the log data streams that carry it: `nginx_access`, `php_access`, `nginx_error` and `apache_error`. Add a `data_stream.dataset` filter to narrow it to one stream.

It is not placed on any of the four dashboards. A packaged dashboard may only reference an index pattern, so a map panel would have to be stored by value, and Kibana 9.5.3 silently drops a by-value map panel at load time. Add it to a dashboard of your own and it works normally, because a dashboard you own may reference it. The packaged dashboards show the same data as `client.geo.country_iso_code` bar charts instead.

Two conventions are worth knowing before reading them.

The time series and top-N panels on the first three dashboards count documents, which for log data is one document per log line. The inventory dashboard does not: `GET /servers`, `/apps`, `/dbs` and `/sysusers` each return a full snapshot on every collection, so a document count there would only count how many times the agent polled. Every number on that dashboard is a distinct count of the ServerPilot id instead, which is why "Apps: 4" stays 4 no matter how long the integration has been running.

Request time comes from `event.duration`, which only the Nginx and PHP-FPM access logs record. ServerPilot's Apache access log format has no duration field at all, so an Apache-fronted app shows `(null)` in the latency columns rather than a zero. That is the honest answer: the information is not in the log.

### Data streams

#### nginx_access

Per-app Nginx access logs, from both `APPNAME_nginx.access.log` and `APPNAME_nginx.access_ssl.log`. ServerPilot's format is the combined format plus `x_forwarded_for`, `request_length`, `request_time`, `upstream_response_length`, `upstream_response_time` and `host`.

GeoIP and ASN enrichment target `client.ip`, which is the first X-Forwarded-For hop when a proxy chain is present and a copy of `source.ip` otherwise. Dashboards and detection rules must therefore use `client.geo.*`; `source.geo.*` is never populated. `url.scheme` is derived from the filename, because HTTPS traffic is written to a separate file. When Nginx retries a failed upstream it logs several comma-separated upstream values; these are summed into totals.

X-Forwarded-For is set by the client and Nginx does not validate it, so the header is accepted only when it is a chain of IP addresses end to end. A header that is not gets no `network.forwarded_ip`, leaves `client.ip` as a copy of `source.ip`, and tags the document `serverpilot_nginx_access_invalid_forwarded_ip`; the header as received remains in `event.original`. Query that tag to find clients sending forged forwarding headers.

##### nginx_access fields

{{ fields "nginx_access" }}

##### nginx_access sample event

{{ event "nginx_access" }}

#### nginx_error

Per-app Nginx error logs. The parser handles messages with or without a connection ID, and the variable set of trailing `client:`, `server:`, `request:`, `upstream:`, `host:` and `referrer:` pairs. The Nginx errno is kept in `serverpilot.nginx.error.*` so that a parsed upstream failure is never confused with an ingest failure.

These timestamps carry no timezone, so they are interpreted using the agent's timezone via `event.timezone`.

##### nginx_error fields

{{ fields "nginx_error" }}

##### nginx_error sample event

{{ event "nginx_error" }}

#### apache_error

Per-app Apache error logs. Multi-line entries are supported, including the PHP stack traces that `proxy_fcgi` relays. The `[client]` block is optional and absent on server lifecycle messages. Apache `AHxxxxx` codes are extracted into `serverpilot.apache.error.code`, and PHP source locations into `log.origin.file.*`.

The client port Apache logs is always `0` on a ServerPilot server, because Nginx proxies over loopback and `mod_remoteip` restores the address without a port, so it is deliberately not mapped to `source.port`. These timestamps carry no timezone and use `event.timezone`.

##### apache_error fields

{{ fields "apache_error" }}

##### apache_error sample event

{{ event "apache_error" }}

#### php_access

Per-app PHP-FPM access logs, the innermost hop of the request path. This is the richest performance data the stack produces: per-request duration, peak memory, and separate user and system CPU figures.

`serverpilot.php.memory.peak.bytes` is in **bytes**. ServerPilot names the source field `PEAK_MEMORY_MB` and its documentation states the value is in megabytes, which is incorrect: PHP-FPM's `%M` directive emits bytes. The CPU fields `serverpilot.php.cpu.user.percent` and `serverpilot.php.cpu.system.percent` hold the logged percentage as-is on a 0-100 scale, are per single core, and can exceed 100. They are named `.percent` rather than `.pct` because a `.pct` field in Beats and ECS holds a 0-1 fraction, so a consumer reading the field metadata would scale these values by 100.

`url.original` holds the pre-rewrite URI, consistent with `nginx_access`, while the post-rewrite URI that PHP actually executed is kept in `serverpilot.php.request_uri`.

##### php_access fields

{{ fields "php_access" }}

##### php_access sample event

{{ event "php_access" }}

#### php_error

Per-app PHP error logs. Handles all PHP severities, including the multi-word ones, and both the ` in FILE on line N` and ` in FILE:N` location forms. For uncaught exceptions the exception class is extracted into `error.type` and the trace into `error.stack_trace`.

Unlike the other error streams, these timestamps carry an explicit timezone identifier, which is parsed from the entry itself rather than assumed.

##### php_error fields

{{ fields "php_error" }}

##### php_error sample event

{{ event "php_error" }}

#### php_slow

Per-app PHP-FPM slow request logs. Each entry is a stack trace captured when a request passed PHP-FPM's slow request threshold.

This log contains no request duration, only the fact that the threshold was passed, so no duration is recorded. Use `php_access` for measured request time. The stack frames are parsed into `serverpilot.php.slow.frames`, and the innermost frame, which is usually the bottleneck, is promoted to `log.origin.function` and `log.origin.file.*` so that "which function is making my site slow" is a single aggregation.

The trace in its original order, innermost frame first and one frame per line, is in `serverpilot.php.slow.stack_trace`. Read the trace from there, not from `serverpilot.php.slow.frames`: these indices use `logsdb` mode, under which each sub-field of that array is rebuilt independently, sorted and deduplicated, so neither the frame order nor the pairing of a function with its file and line survives in the stored document. The array is still the right field for asking whether a trace involves a given function or file anywhere.

These timestamps carry no timezone and use `event.timezone`.

##### php_slow fields

{{ fields "php_slow" }}

##### php_slow sample event

{{ event "php_slow" }}

#### server

Server inventory from `GET /servers`. The most valuable content is the security posture of each server: `autoupdates`, `firewall` and `deny_unknown_domains`.

`serverpilot.server.name` is the label the server was given in ServerPilot and is not necessarily its hostname, which is why it is not mapped to `host.name`. `serverpilot.server.last_address` is the address the server last connected to ServerPilot from, so it is deliberately not enriched as a request client.

##### server fields

{{ fields "server" }}

##### server sample event

{{ event "server" }}

#### sysuser

System user inventory from `GET /sysusers`. `serverpilot.sysuser.name` is the join key to the seven log data streams, which derive the same value from the log file path. The account's home directory is always `/srv/users/NAME`.

##### sysuser fields

{{ fields "sysuser" }}

##### sysuser sample event

{{ event "sysuser" }}

#### app

App inventory from `GET /apps`. `serverpilot.app.name` and `service.name` are the join keys to the seven log data streams.

`serverpilot.app.runtime.version` is the numeric PHP version split out of `runtime`, so that apps on an end-of-life PHP release can be found and alerted on. It should agree with `serverpilot.php.version` in the PHP log data streams. `serverpilot.app.ssl.enabled` records whether the app has any TLS certificate at all.

##### app fields

{{ fields "app" }}

##### app sample event

{{ event "app" }}

#### database

Database inventory from `GET /dbs`. Joins to the `app` data stream through `serverpilot.app.id`. Database passwords are accepted by the API's create and update endpoints but are never returned on a read, so none is ever collected.

##### database fields

{{ fields "database" }}

##### database sample event

{{ event "database" }}

{{ ilm }}

{{ transform }}
