---
lastmod: 2018-11-28
date: 2018-11-27
linktitle: Debug endpoint
menu:
  documentation:
    parent: endpoints
title: The `/__debug` endpoint
weight: 35
---
The `/__debug` endpoint is available when you start the server with the `-d` flag.

The endpoint can be used as a fake backend to see its activity in the log. When developing, add KrakenD itself as a backend using the `/__debug/` endpoint so you can see exactly what headers and query string parameters your backends are receiving.

The debug endpoint might save you a lot of trouble, as your application might not work when certain headers or parameters are not present, and you might be relying upon what your client is sending, and not in what the gateway is sending.

For instance, your client might be sending a `Content-Type` header, but unless this header is recognized by the gateway (is added in `headers_to_pass`), it is not going to reach the backend. Seeing the specific headers and parameters in the log clears all the doubts, and you can reproduce the call and conditions easily.

# Debug endpoint configuration example
The following configuration demonstrates how to test what headers and query string parameters are sent and received by the backends by using the `/__debug` endpoint.

We are going to test the following endpoints:

- `/default-behavior`: No client headers, query string or cookies forwarded.
- `/known-params`: Forwards known parameters and headers
    - Recognizes `a` and `b` as query string
    - Recognizes `User-Agent` and `Accept` as forwarded headers

To test it right now, save the content of this file in a `krakend-test.json` and start the server with the `-d` flag:

    {
        "version": 2,
        "port": 8080,
        "endpoints": [
            {
            "endpoint": "/default-behavior",
            "method": "GET",
            "backend": [
                {
                "url_pattern": "/__debug/all",
                "host": [ "http://127.0.0.1:8080" ]
                }
            ]
            },
            {
            "endpoint": "/known-params",
            "method": "GET",
            "querystring_params": [
                "a",
                "b"
            ],
            "headers_to_pass": [
                "User-Agent",
                "Accept"
            ],
            "backend": [
                {
                "url_pattern": "/__debug/some",
                "host": [ "http://127.0.0.1:8080" ]
                }
            ]
            }
        ]
    }

Start the server:

    krakend run -d -c krakend-test.json


Now we can test that the endpoints behave as expected:

**Default behavior:**

    curl -i 'http://localhost:8080/default-behavior?a=1&b=2&c=3'

In the KrakenD log, we can see that `a`, `b`, and `c` do not appear in the backend call, neither its headers. The `curl` command automatically sends the `Accept` and `User-Agent` headers and as you can see below the user agent is the gateway itself:

{{< highlight go "hl_lines=5 8" >}}
DEBUG: Method: GET
DEBUG: URL: /__debug/all
DEBUG: Query: map[]
DEBUG: Params: [{param /all}]
DEBUG: Headers: map[User-Agent:[KrakenD Version {{% version %}}] X-Forwarded-For:[::1] Accept-Encoding:[gzip]]
DEBUG: Body:
[GIN] 2018/11/27 - 22:32:44 | 200 |     118.543µs |             ::1 | GET      /__debug/all
[GIN] 2018/11/27 - 22:32:44 | 200 |     565.971µs |             ::1 | GET      /default-behavior?a=1&b=2&c=3
{{< /highlight >}}

Now let's repeat the same request but to the `/known-params` endpoint:

    curl -i 'http://localhost:8080/known-params?a=1&b=2&c=3'

In the KrakenD log we can see now that the `User-Agent` and `Accept` are present (as they are implicitly sent by curl):

{{< highlight go "hl_lines=5 8" >}}
DEBUG: Method: GET
 DEBUG: URL: /__debug/some?a=1&b=2
 DEBUG: Query: map[a:[1] b:[2]]
 DEBUG: Params: [{param /some}]
 DEBUG: Headers: map[User-Agent:[curl/7.54.0] Accept:[*/*] X-Forwarded-For:[::1] Accept-Encoding:[gzip]]
 DEBUG: Body:
[GIN] 2018/11/27 - 22:33:23 | 200 |     122.507µs |             ::1 | GET      /__debug/some?a=1&b=2
[GIN] 2018/11/27 - 22:33:23 | 200 |     542.483µs |             ::1 | GET      /known-params?a=1&b=2&c=3
{{< /highlight >}}