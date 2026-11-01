# HTTP Protocol

## Overview

The **Hypertext Transfer Protocol (HTTP)** is an application-level protocol that is being widely used over the Web.
HTTP is a **request/response** protocol, which means, the client sends a request to the server (request method, URI, protocol version, followed by headers, and possible body content).
The server responds (status code line, a success or error code, followed by server headers information, and possible entity-body content).

![][http-req-res]

Under the hood, HTTP requests and responses are sent over a TCP socket with default port 80 (on the server side).
Servers should be able to handle thousands of simultaneous TCP connections.
The AgroPilot registry service in this course listens on port **8000**.

## Endpoints

An **endpoint** is a specific URL path on a server that can be accessed by clients to perform a particular action or retrieve specific information.

The registry service already has an endpoint, `GET /health`, that clients can access to check if the server is running.

```python
@app.get("/health")
def health():
    return {"status": "ok"}
```

This endpoint returns a simple JSON response indicating that the server is running.
Start the service the same way as in the [Python environment tutorial](python_env.md):

```bash
cd services/registry-service
uv run uvicorn main:app --host 127.0.0.1 --port 8000
```

In another terminal, test the endpoint:

```bash
curl http://127.0.0.1:8000/health
```

You should see: `{"status":"ok"}`

A path can name one resource. `GET /v1/farms/b7e2c0a0-3c1d-4f0a-9a11-0c5d2e6f1001` asks for that farm. The id is part of the path.

A **query** sits after `?` and narrows a list. `GET /v1/plots?farm_id=b7e2c0a0-3c1d-4f0a-9a11-0c5d2e6f1001` asks for the plots of that farm. With no query, `GET /v1/plots` asks for every plot.

Those `/v1/...` routes are the registry task. They are not on the service yet. `/health` is the endpoint you can call today.

## HTTP Request and Response

We can learn a lot by taking a closer look at a raw HTTP request and response sent over the network:

```text
curl -v http://127.0.0.1:8000/health
```

Below is the actual raw HTTP request sent by `curl` to the server:

```text
GET /health HTTP/1.1
Host: 127.0.0.1:8000
User-Agent: curl/7.68.0
Accept: */*
```

The server response is:

```text
HTTP/1.1 200 OK
date: Sun, 04 May 2025 12:52:02 GMT
server: uvicorn
content-length: 15
content-type: application/json

{"status":"ok"}
```

The first request line is the method, the path, and the HTTP version. Headers follow. A blank line ends the headers, and the body comes after that line. `/health` has no request body. The response body is JSON, so `content-type` is `application/json`.

A create sends a body. The same blank line separates the headers from the JSON:

```text
POST /v1/farms HTTP/1.1
Host: 127.0.0.1:8000
Content-Type: application/json
Accept: application/json

{"name": "South farm"}
```

`Content-Type` tells the server what the body is. `Accept` tells the server what the client can read. `POST /v1/farms` is part of the task. The message shape is what matters here.

The MDN web docs [specify the core components of request and response objects](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview), review this resource.

## Status code

HTTP response status codes indicate how a specific HTTP request was completed.

Responses are grouped in five classes:

- Informational (100–199)
- Successful (200–299)
- Redirection (300–399)
- Client error (400–499)
- Server error (500–599)

The registry API uses these codes:

- `200 OK` — a read or an update worked. A list with no rows is still `200`, and the body is `[]`.
- `201 Created` — `POST` stored a new farm or plot. The body is that object, including its `id`.
- `204 No Content` — `DELETE` worked. The response has no body.
- `404 Not Found` — no farm, plot, crop, soil, or emitter has that id.
- `405 Method Not Allowed` — that method is not served on that path. Crops, soils, and emitters are read-only: `GET` works, and any other method returns `405`.
- `409 Conflict` — the request clashes with data already stored. The plot name is already used on that farm, or the farm being deleted still has plots.
- `422 Unprocessable Entity` — the server read the JSON, and a field is invalid, or the crop, soil, or emitter is not in the catalogue.
- `503 Service Unavailable` — the database cannot be reached. The response includes a `Retry-After` header.

Every error body is `application/problem+json`, as defined by [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457), with a stable `code`:

```json
{
  "type": "about:blank",
  "title": "Unprocessable Entity",
  "status": 422,
  "detail": "Unknown emitter",
  "code": "VALIDATION"
}
```

`status` repeats the HTTP status code. `detail` is a sentence a person can read. Clients branch on `code`, which is one of:

- `VALIDATION` — `422`. A field is invalid, or the crop, soil, or emitter is unknown.
- `NOT_FOUND` — `404`. The id is unknown.
- `CONFLICT` — `409`. Duplicate plot name on the same farm, or a delete of a farm that still has plots.
- `UNAVAILABLE` — `503`. The database cannot be reached.

The registry task asks for this body. A body of `{"detail": "..."}` with `Content-Type: application/json` does not meet it.


## APIs

An **API (Application Programming Interface)** can be thought of as a collection of all server **endpoints** that together define the functionality that is exposed to the client.
Each endpoint typically accepts input parameters in a specific format and returns output data in a standard format such as JSON.

The registry API is that collection for AgroPilot: farms, plots, and three catalogues (crops, soils, and emitters).
A farm has an `id` and a `name`. A plot belongs to one farm and points at one crop, one soil, and one emitter. Catalogue rows are read with `GET`. The client does not create them through this API.
Lists are JSON arrays.

Many platforms expose both API, and GUI. Like Spotify, OpenAI and GitHub.

#### REST API

**REST (Representational State Transfer)** is an architectural style for designing APIs.
A REST API uses standard HTTP methods (`GET`, `POST`, `PUT`, `DELETE`) to perform operations on resources, which are identified by URLs.
This API uses `PATCH` in place of `PUT`: a `PATCH` body contains only the fields that change.
Key principles include statelessness (each request contains all information needed to process it), a uniform interface, and representation of resources as JSON.

Read one farm:

```text
GET /v1/farms/b7e2c0a0-3c1d-4f0a-9a11-0c5d2e6f1001
```

Create a farm:

```text
POST /v1/farms
Content-Type: application/json

{"name": "South farm"}
```

Create a plot. `farm_id` selects the farm. `crop_id`, `soil_id`, and `emitter_id` select catalogue rows:

```text
POST /v1/plots
Content-Type: application/json

{
  "farm_id": "b7e2c0a0-3c1d-4f0a-9a11-0c5d2e6f1001",
  "name": "South block",
  "area_m2": 1000,
  "crop_id": "tomato",
  "planting_date": "2026-03-01",
  "soil_id": "loam",
  "row_spacing_m": 1.5,
  "emitter_spacing_m": 0.4,
  "emitter_id": "dripnet-pc-1.0",
  "lat": 31.25,
  "lon": 34.79
}
```

`201` returns that object plus `id`.

Rename a farm. Fields absent from the body stay as they are:

```text
PATCH /v1/farms/b7e2c0a0-3c1d-4f0a-9a11-0c5d2e6f1001
Content-Type: application/json

{"name": "South cooperative"}
```

Read one catalogue row:

```text
GET /v1/emitters/dripnet-pc-1.0
```

```json
{ "id": "dripnet-pc-1.0", "name": "DripNet PC", "flow_lph": 1.0 }
```

Crops and soils use the same pattern: `GET /v1/crops/tomato`, `GET /v1/soils/sand`.

For a deeper dive, see the [restapitutorial.com](https://www.restapitutorial.com/introduction/restconstraints).

## Introducing Postman

Postman is a powerful and user-friendly tool for testing, debugging, and documenting APIs.
The exercises below use `curl`. The same request in Postman is a method, a URL, headers, and an optional JSON body.

1. Download from: https://learning.postman.com/docs/getting-started/first-steps/get-postman/.
2. Create a Postman account to unlock some important features.


[http-req-res]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/networking_http-req-res.png
[networking_cookies]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/networking_cookies.png
