# Exercise 06 — Manual Instrumentation

[← Exercise 05](05-processing.md)

In this exercise you write OpenTelemetry instrumentation by hand — adding DB tracing to the
backend and a login instrumentation shell on the frontend.

Unlike the earlier exercises, this one touches ~17 files (including a new ~250-line Go file)
and is meant to be read and **explored**, not typed out. The doc walks through the main
design decisions; the full diff lives on GitHub.

> [!TIP]
> Open the [full diff for this exercise](https://github.com/immavalls/opentelemetry-instrumentation-workshop/compare/05-processing...06-manual-instrumentation)
> in a second tab and read it alongside this walkthrough. To run the finished version
> locally: `git checkout 06-manual-instrumentation`.

## Contents

- [The instrumentation as a wrapper pattern](#the-instrumentation-as-a-wrapper-pattern)
- [Part 1 — Backend DB (Go)](#part-1--backend-db-go)
- [Part 2 — Frontend Login (Node.js)](#part-2--frontend-login-nodejs)
- [Tests](#tests)
- [Verify](#verify)
- [Catch up](#catch-up)

---

## The instrumentation as a wrapper pattern

Both parts follow the same principle: wrap business logic in an **instrumentation shell**
that starts a span, records metrics, handles errors, and ends the span — while the wrapped
code stays unaware of OTel.

In Go, `DB` is the wrapper — its methods call `startSpan`/`endSpan` around the real `sql.DB`
calls. In JS, `instrumentLogin` is the wrapper — it wraps an async callback `fn` that contains
pure login logic.

> [!IMPORTANT]
> Manual instrumentation is tedious and error-prone. Only do it when there is no high-quality
> library available, or when you have specific requirements a library cannot meet. In the backend
> case there is no library that follows OTel DB semantic conventions precisely enough;
> in the frontend case it wraps custom application auth flow.

---

## Part 1 — Backend DB (Go)

[**→ Browse Part 1 on GitHub**](https://github.com/immavalls/opentelemetry-instrumentation-workshop/compare/05-processing...06-manual-instrumentation#files_bucket) — filter to `backend/`.

A new [`backend/db/instrumented.go`](https://github.com/immavalls/opentelemetry-instrumentation-workshop/blob/06-manual-instrumentation/backend/db/instrumented.go) wraps `*sql.DB` with OTel. `Connect()` in
[`db.go`](https://github.com/immavalls/opentelemetry-instrumentation-workshop/blob/06-manual-instrumentation/backend/db/db.go) now returns the wrapper, so every handler and middleware that took a
`*sql.DB` needs to accept `*dbpkg.DB` instead (`go build ./...` lists them all).

The key pieces to look for in `instrumented.go`:

- **`DB` struct embeds `*sql.DB`** — methods we don't override (e.g. `PingContext`) pass
  through unchanged. We only wrap the three that execute queries: `QueryContext`,
  `QueryRowContext`, and `ExecContext`.

- **Span attributes follow [DB semantic conventions](https://opentelemetry.io/docs/specs/semconv/database/database-spans/).**
  The span name and `db.query.summary` are a low-cardinality summary like `SELECT restaurants`
  (computed by `querySummary` from the first word of the SQL and the table name). The full
  SQL goes into `db.query.text`. `server.address`, `server.port`, and `db.namespace` are
  parsed from the DSN once at startup and reused for every span.

- **The `*Row` wrapper is the non-obvious bit.** `*sql.Row` only reveals its error when
  the caller calls `Scan` or `Err` — so if we end the span inside `QueryRowContext`, every
  single-row query always ends with status OK, even on real errors. The fix: return a
  custom `*Row` type that ends the span inside _its_ `Scan`/`Err` methods, guarded by
  `sync.Once` so it ends exactly once.

- **Errors are emitted as logs, not span events.** `endSpan` calls `slog.ErrorContext`,
  which (via the slog→OTel bridge in [`backend/telemetry.go`](https://github.com/immavalls/opentelemetry-instrumentation-workshop/blob/06-manual-instrumentation/backend/telemetry.go))
  emits a log correlated to the active span. This follows the [exceptions-as-logs spec](https://opentelemetry.io/docs/specs/semconv/exceptions/exceptions-logs/).
  `sql.ErrNoRows` is treated as a normal outcome and does not set error status.

- **Metric attributes get `error.type` on failure.** The `db.client.operation.duration`
  histogram is recorded on every call; on error we append `error.type` so failures show
  up as a distinct dimension on the same metric.

---

## Part 2 — Frontend Login (Node.js)

[**→ Browse Part 2 on GitHub**](https://github.com/immavalls/opentelemetry-instrumentation-workshop/compare/05-processing...06-manual-instrumentation#files_bucket) — filter to `frontend/`.

A new [`frontend/otel-auth.js`](https://github.com/immavalls/opentelemetry-instrumentation-workshop/blob/06-manual-instrumentation/frontend/otel-auth.js) exports `instrumentLogin(provider, fn)`.
[`server.js`](https://github.com/immavalls/opentelemetry-instrumentation-workshop/blob/06-manual-instrumentation/frontend/server.js) uses it to wrap the OAuth callback, and records the
`auth.client.login.duration` metric directly in the plain-username login handler.

What to look for:

- **`instrumentLogin` takes a pure async callback.** `fn` does the actual login work
  and returns `{ outcome, user?, isNewUser? }` — it knows nothing about OTel. The
  wrapper starts a span, maps `outcome` to attributes and span status, records the
  histogram, and ends the span in `finally` so it always ends exactly once — even on
  a thrown exception.

- **Outcome-based error signalling.** Instead of `throw`-ing for business failures
  (`user_not_found`, `state_mismatch`), the callback returns them as `outcome` strings.
  The wrapper copies that onto `error.type` and sets span status ERROR. Thrown errors
  (e.g. network failure) are a separate path — `error.type` becomes the exception
  class name.

- **Plain-username login has no span, only a metric.** It is a single backend
  round-trip, not a multi-step flow, so starting a span adds noise without insight.
  The `loginDuration` histogram is exported from `otel-auth.js` and recorded directly
  in the `POST /login` handler with `auth.provider.name = "local"`.

- **`enduser.id` and `enduser.pseudo.id`** are set on the login span when the user
  is known, so you can search traces by user after the fact.

---

## Tests

Both parts use **in-memory exporters** for fast, hermetic unit tests — no real trace
backend required.

**Backend** — [tests/backend/instrumentation_test.go](https://github.com/immavalls/opentelemetry-instrumentation-workshop/blob/06-manual-instrumentation/tests/backend/instrumentation_test.go) uses an in-memory span recorder against a real (ephemeral) database.

**Frontend** — [tests/frontend/otel-auth.test.js](https://github.com/immavalls/opentelemetry-instrumentation-workshop/blob/06-manual-instrumentation/tests/frontend/otel-auth.test.js) uses in-memory span and metric exporters.

```bash
make test
```

---

## Verify

```bash
docker compose up --build
```

> [!NOTE]
> Traces and metrics may take up to a minute to appear after the services start. If panels are empty or spans are missing, wait a moment and refresh.

Open Grafana at <http://localhost:3000/d/apm-dashboard>.

> [!TIP]
> Exercise 05's collector changes affect what you see below: `tail_sampling` only keeps error
> traces and traces slower than one second, and `transform/anonymize_enduser` replaces
> `enduser.id` with a hash. To see every trace, temporarily remove `tail_sampling` from the
> `traces` pipeline in [otel-collector/config.yaml](https://github.com/immavalls/opentelemetry-instrumentation-workshop/blob/06-manual-instrumentation/otel-collector/config.yaml)
> and restart the collector with `docker compose restart otel-collector`.

### Backend — DB calls

The APM dashboard opens on the `frontend` service, which has no database. Set **Name** at
the top of the dashboard to `backend`. In **Outgoing client calls**, the **DB client calls**
panel lists one row per query (for example `db/tapas | SELECT restaurants` and
`db/tapas | SELECT users`) with its rate, error rate and latency. Submit a few ratings in
the app and an `INSERT ratings` row shows up too.

That panel is built from metrics. To look at the spans themselves,
[open the backend DB spans in Grafana (Tempo)](http://localhost:3000/explore?schemaVersion=1&orgId=1&panes=%7B%22abc%22%3A%7B%22datasource%22%3A%22tempo%22%2C%22queries%22%3A%5B%7B%22refId%22%3A%22A%22%2C%22queryType%22%3A%22traceql%22%2C%22query%22%3A%22%7B%20resource.service.name%20%3D%20%5C%22backend%5C%22%20%26%26%20span.db.system.name%20%3D%20%5C%22postgresql%5C%22%20%7D%22%7D%5D%2C%22range%22%3A%7B%22from%22%3A%22now-15m%22%2C%22to%22%3A%22now%22%7D%7D%7D),
then click a span to verify:

- `db.system.name` = `postgresql`
- `db.query.summary` = `SELECT restaurants` (or similar)
- `db.query.text` = the full SQL string
- `server.address` = `db`
- `server.port` = `5432`
- `db.namespace` = `tapas`

> [!NOTE]
> Tail sampling from [Exercise 05](05-processing.md) only keeps error traces and traces
> slower than one second. With Chaos Mode on (the default) you will see spans such as
> `SELECT restaurants`, `SELECT users` and `SELECT photos`, but fast successful requests
> like submitting a rating are dropped, so `INSERT ratings` only shows up in the
> **DB client calls** panel, not as a span (unless you remove `tail_sampling`, see the tip above).

For error spans (e.g. with Chaos Mode enabled), check the **Logs** panel — you should see
an error log with `exception.type` and `exception.message` correlated to the failing span.

### Frontend — login spans

Open the app at <http://localhost:8080> and log in via **Acme SSO** (OAuth).
[Open the Acme login spans in Grafana (Tempo)](http://localhost:3000/explore?schemaVersion=1&orgId=1&panes=%7B%22abc%22%3A%7B%22datasource%22%3A%22tempo%22%2C%22queries%22%3A%5B%7B%22refId%22%3A%22A%22%2C%22queryType%22%3A%22traceql%22%2C%22query%22%3A%22%7B%20span.auth.provider.name%20%3D%20%5C%22acme%5C%22%20%7D%22%7D%5D%2C%22range%22%3A%7B%22from%22%3A%22now-1h%22%2C%22to%22%3A%22now%22%7D%7D%7D)
(`{ span.auth.provider.name = "acme" }`) and click the `login` span to verify:

- `auth.provider.name` = `acme`
- `enduser.id` = the username, hashed by the collector from [Exercise 05](05-processing.md):
  the first 8 characters of its SHA-256 (for `alice` that is `2bd806c9`)
- `enduser.pseudo.id` = the user's id in the database, an 8-character hex string

> [!NOTE]
> A fast, successful login can be dropped by tail sampling. If you don't see the `login`
> span, see the tip above to see every trace.

The plain username login has no span — it only records the `auth.client.login.duration` metric.

Check out the [metrics drilldown](http://localhost:3000/a/grafana-metricsdrilldown-app/), [traces drilldown](http://localhost:3000/a/grafana-exploretraces-app/), and [logs drilldown](http://localhost:3000/a/grafana-lokiexplore-app/) — great tools to see what telemetry is available.

---

## Learn more

- [Instrumentation concepts](https://opentelemetry.io/docs/concepts/instrumentation/) — when to reach for manual instrumentation vs. libraries
- [Go language guide](https://opentelemetry.io/docs/languages/go/) — tracer, meter, and logger APIs used throughout Part 1
- [Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/) — including [database](https://opentelemetry.io/docs/specs/semconv/database/) attributes (`db.query.summary`, `db.query.text`, `db.system.name`, …)

---

## Catch up

```bash
git checkout 06-manual-instrumentation
```

---

[← Exercise 05](05-processing.md)
