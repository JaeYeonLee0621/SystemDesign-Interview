# ETag

## Generation
- the content of the response
- a hash or some other identifier that uniquely represents the current state of the resource

## Sending ETag to the Client
- The client stores this ETag along with the cached resource

## Subsequent Request and Validation
- When the client makes another request for the same resource, it sends the previously stored ETag value in the `If-None-Match` request header
- The server then regenerate the ETag for the current state of the resource

## Comparision
- If the ETag match : `304 Not Modified` without the actual content
- It the ETag don't match : The server sends the new content along with a new ETag in the response headers

<br/>

# Cache-Control, Expires

- They primarily serve to define how long a piece of content should be considered fresh and thus can be served from the cache without going back to the server to check for a newer version. 

+) Expires
- HTTP/1.0 way of controlling cache lifetime
- It uses absolute timestamps, it can be less reliable due to clock difference between servers and clients

## Example (Expires)

### Client Request
```
GET /logo.png HTTP/1.1
Host: example.com
```

### Server Response
```
HTTP/1.1 200 OK
Content-Type: image/png
Expires: Thu, 01 Dec 2022 16:00:00 GMT
Content-Length: 5120

(png image data)
```

- The response should be considered fresh until December 1, 2022, at 16:00 GMT.

## Example (Cache-Control)

### Client Request
```
GET /image.jpg HTTP/1.1
Host: example.com
```

### Server Response
```
HTTP/1.1 200 OK
Content-Type: image/jpeg
Cache-Control: max-age=86400, public
Content-Length: 2048

(image data)
```
- The response can be cached and considered fresh for 86,400 seconds (24 hours)
- A resource will be considered fresh from the moment the response is generated.

<br/>

# Last-Modified, ETag

`Network Resource Usage`
- Server checks if the resource has been modified since the data provided or if the ETag still matches the current version of the resouce

`Database Resource Usage`
- To determine whether the content has changed, the server might need to query the database or access the file system

## Example (Last-Modified)

### Client Request
```
GET /article.html HTTP/1.1
Host: example.com
```

### Server Response
```
HTTP/1.1 200 OK
Content-Type: text/html
Last-Modified: Wed, 21 Oct 2020 07:28:00 GMT
Content-Length: 6789

(html content)
```

### Client Request
```
GET /article.html HTTP/1.1
Host: example.com
If-Modified-Since: Wed, 21 Oct 2020 07:28:00 GMT
```

### Server Response
```
HTTP/1.1 304 Not Modified
```

## Example (ETag)

### Scenario 1: First Request (Without ETag)
This is the initial request for a resource from a client. Since the client has never requested the resource before, there is no ETag to send.

### Client Request
```
GET /resource HTTP/1.1
Host: example.com
```

### Server Response
```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 12345
ETag: "abcdef1234567890"

(content of the resource)
```

### Scenario 2: Subsequent Request (With Unchanged ETag)

### Client Request
```
GET /resource HTTP/1.1
Host: example.com
If-None-Match: "abcdef1234567890"
```

### Server Response
```
HTTP/1.1 304 Not Modified
ETag: "abcdef1234567890"
```

### Scenario 3: Subsequent Request (With Changed ETag)

### Client Request
```
GET /resource HTTP/1.1
Host: example.com
If-None-Match: "abcdef1234567890"
```

### Server Response
```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 12345
ETag: "newetag9876543210"

(new content of the resource)
```
