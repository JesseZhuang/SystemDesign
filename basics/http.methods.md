## GET

The GET method requests a representation of the specified resource. Requests using GET should only retrieve data.

## HEAD

The HEAD method asks for a response identical to a GET request, but without the response body.

The HTTP HEAD method requests the headers that would be returned if the HEAD request's URL was instead requested with the HTTP GET method. For example, if a URL might produce a large download, a HEAD request could read its Content-Length header to check the filesize without actually downloading the file.

## POST

The POST method submits an entity to the specified resource, often causing a change in state or side effects on the server.

## PUT

The PUT method replaces all current representations of the target resource with the request payload.

The difference between PUT and POST is that PUT is idempotent: calling it once or several times successively has the same effect (that is no side effect), whereas successive identical POST requests may have additional effects, akin to placing an order several times.

## DELETE

The DELETE method deletes the specified resource.

## CONNECT

The CONNECT method establishes a tunnel to the server identified by the target resource.

The HTTP CONNECT method starts two-way communications with the requested resource. It can be used to open a tunnel.

For example, the CONNECT method can be used to access websites that use TLS (HTTPS). The client asks an HTTP Proxy server to tunnel the TCP connection to the desired destination. The server then proceeds to make the connection on behalf of the client. Once the connection has been established by the server, the Proxy server continues to proxy the TCP stream to and from the client.

CONNECT is a hop-by-hop method.

## OPTIONS

The OPTIONS method describes the communication options for the target resource.

## TRACE

The TRACE method performs a message loop-back test along the path to the target resource.

## PATCH

The PATCH method applies partial modifications to a resource.

The HTTP PATCH request method applies partial modifications to a resource.

PATCH is somewhat analogous to the "update" concept found in CRUD (in general, HTTP is different than CRUD, and the two should not be confused).

A PATCH request is considered a set of instructions on how to modify a resource. Contrast this with PUT; which is a complete representation of a resource.

A PATCH is not necessarily idempotent, although it can be. Contrast this with PUT; which is always idempotent. The word "idempotent" means that any number of repeated, identical requests will leave the resource in the same state. For example if an auto-incrementing counter field is an integral part of the resource, then a PUT will naturally overwrite it (since it overwrites everything), but not necessarily so for PATCH.

PATCH (like POST) may have side-effects on other resources.

To find out whether a server supports PATCH, a server can advertise its support by adding it to the list in the Allow or Access-Control-Allow-Methods (for CORS) response headers.

Another (implicit) indication that PATCH is allowed, is the presence of the Accept-Patch header, which specifies the patch document formats accepted by the server.

## References

1. https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods
