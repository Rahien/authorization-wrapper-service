# sparql-authorization-wrapper-service

Wrapper for executing SPARQL queries on a mu-semtech stack with some extra
authorization checks.

This service forwards SPARQL queries to the database (mu-authorization, or a
triplestore) in the sense that a proxy server would. However, the request is
intercepted and some authorization rules can be configured to allow or disallow
the request from being proxied. These rules can be written in JavaScript with
the possibility to send SPARQL queries for checking permissions. This service
relies on the presence of the `mu-session-id` header that is introduced in the
request by the [mu-identifier](https://github.com/mu-semtech/mu-identifier) and
checks for existing sessions after a log in via the [Vendor Login
Service](https://github.com/lblod/vendor-login-service).

## Adding to a stack

Add the SPARQL Authorization Service to a mu-semtech stack by placing the
following snippet in the `docker-compose.yml` file as a service:

```yaml
sparql-authorization-wrapper:
  image: lblod/sparql-authorization-wrapper-service:0.0.1
  volumes:
    - ../path-to-config:/config
```

where `../path-to-config` contains the configuration files (see below).

Add the following lines to the dispatcher's configuration:

```elixir
match "/sparql" do
  Proxy.forward conn, [], "http://sparql-authorization-wrapper/sparql"
end
```

Note that you should use `match` in the above configuration, because this
service can respond to both GET and POST on the same URL.

## Writing rules

Rules shoud be written in a file called `filter.js` that should be placed in
the configuration folder that is mounted as a subvolume (see the Docker
configuration above). Define a JavaScript function called `isAuthorized` that
accepts a single parameter, namely the client's session-id. By importing from
`mu` and `mu-auth-sudo`, you get access to the `uuid()`, `sparqlEscape*()`,
`query()`, `update()`, `querySudo()` and `updateSudo()` functions that you can
use to verify authorization of the current client. The function should return a
truthy value to indicate the client is authorized to execute the SPARQL query,
or a falsy value to indicate it is not allowed to. A typical `filter.js` file
might, e.g., look like the following:

```javascript
import * as mu from 'mu';
import * as mas from '@lblod/mu-auth-sudo';

export async function isAuthorized(sessionUri) {
  const checkSessionQuery = `
    # A SPARQL query
  `;
  const response = await mas.querySudo(checkSessionQuery);
  return response.results?.bindings?.length === 1;
}
```

## API

According to the [SPARQL 1.1
Protocol](https://www.w3.org/TR/sparql11-protocol/#query-operation) there are
only a few standard mechanisms for sending SPARQL queries. This service aims to
support all of these.

### URL-encoded POST `/sparql`

This is probably the most used in mu-semtech stacks. Send a POST with
`Content-Type: application/x-www-form-urlencoded` header with the `query` as a
URL encoded parameter as the request body.

**Example request body**

```
query=SELECT+DISTINCT+?s+?p+?o+WHERE+{+?s+?p+?o+.+}+LIMIT+10
```

**Response**

On a succesful proxy, the result you get depends on the database's response. If
not successful, you might get one of the following:

`400 Bad Request`

This means the request is not formatted correctly. Supply a `query` parameter
in the body and use method POST (or use another method from below).

`401 Unauthorized`

This means the `mu-session-id` header is missing from the request. Make sure to
check if the mu-identifier is in place and works as expected. (Also make sure
your browser or SPARQL client is receiving and sending cookies.)

`403 Forbidden`

After processing authorization using the configuration, it seems this client
does not have the correct authorization or is not logged in.


### Direct POST `/sparql`

You can post a SPARQL query using `Content-Type: application/sparql-query` and
method POST with a body that is simply comprised of the SPARQL query without
special encoding.

**Response**

On a succesful proxy, the result you get depends on the database's response. If
not successful, you might get one of the following:

`400 Bad Request`

This means the request is not formatted correctly. Supply a query as a body and
use method POST (or use another method from this section).

`401 Unauthorized`

This means the `mu-session-id` header is missing from the request. Make sure to
check if the mu-identifier is in place and works as expected. (Also make sure
your browser or SPARQL client is receiving and sending cookies.)

`403 Forbidden`

After processing authorization using the configuration, it seems this client
does not have the correct authorization or is not logged in.

### GET `/sparql`

This method is not recommended due to limitations on query length and
dispatcher configuration, but allowed. Supply the SPARQL as a URL encoded
`query` parameter in the URL of the request. The body has to be empty in this
case and there should be no `Content-Type` header.

**Response**

On a succesful proxy, the result you get depends on the database's response. If
not successful, you might get one of the following:

`400 Bad Request`

This means the request is not formatted correctly. Supply a query as a URL
parameter, use method GET and make sure the body is empty (or use another
method from this section).

`401 Unauthorized`

This means the `mu-session-id` header is missing from the request. Make sure to
check if the mu-identifier is in place and works as expected. (Also make sure
your browser or SPARQL client is receiving and sending cookies.)

`403 Forbidden`

After processing authorization using the configuration, it seems this client
does not have the correct authorization or is not logged in.

## Configuration

These are environment variables that can be used to configure this service.
Supply a value for them using the `environment` keyword in the
`docker-compose.yml` file.

* `DATABASE_HOST`: *(optional, default: "http://database:8890")* hostname (with
  protocol and portnumber) of the database service. Since this service acts as
  a proxy only on the `/sparql` path, requests are always forwarded to that
  same path on the database host.
* `LOGLEVEL`: *(optional, default: "silent")* level of logging of request
  processing and authorization checking. Possible values are: `["error",
  "silent"]`.
* `PROXY_LOGLEVEL`: *(optional, default: "silent")* level of logging for the
  proxying middleware. Possible values are: `["debug", "info", "warn", "error",
  "silent"]`.

## Model

This service check for the existence of a session with a model that is
specified by the [Vendor Login
Service](https://github.com/lblod/vendor-login-service). It might also throw
errors in JSON-LD format that have the same structure as errors from the Vendor
Login Service.
