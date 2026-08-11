# HTTP Response Status Codes

HTTP response status codes are divided into 5 distinct categories based on their first digit.

## 1xx: Informational Responses
The request was received and the server is continuing the process.

* **100 Continue:** The server received the initial request headers; the client should proceed to send the request body.
* **101 Switching Protocols:** The server agrees to switch protocols as requested by the client (e.g., upgrading to WebSockets).
* **102 Processing (WebDAV):** The server has received and is processing the request, but no response is available yet.
* **103 Early Hints:** Used to return some response headers before the final HTTP message (e.g., preloading assets).

## 2xx: Successful Responses
The action requested by the client was successfully received, understood, and accepted.

* **200 OK:** Standard success response for HTTP requests.
* **201 Created:** The request succeeded and a new resource was created.
* **202 Accepted:** The request was accepted for processing, but processing is not yet complete.
* **203 Non-Authoritative Information:** Returned transformed metadata from a proxy instead of the origin server.
* **204 No Content:** The request succeeded, but there is no payload/body to return.
* **205 Reset Content:** Asks the client to reset the document view that sent the request.
* **206 Partial Content:** Sent when the client requests only part of a resource using a Range header.
* **207 Multi-Status (WebDAV):** Conveys status for multiple independent operations in an XML payload.
* **208 Already Reported (WebDAV):** Used inside a 207 response to avoid enumerating internal members repeatedly.
* **226 IM Used:** Server fulfilled a GET request for the resource, and the response is a representation of the result of one or more instance-manipulations applied to the current instance.

## 3xx: Redirection Messages
Further action must be taken by the user/client to complete the request.

* **300 Multiple Choices:** Indicates multiple options for the resource that the client can follow.
* **301 Moved Permanently:** The target resource has been assigned a new permanent URI.
* **302 Found:** The target resource resides temporarily under a different URI.
* **303 See Other:** Redirects the client to get the requested resource at another URI using a GET request.
* **304 Not Modified:** Indicates the resource has not changed since the last request (used for caching).
* **305 Use Proxy (Deprecated):** The requested resource must be accessed through a proxy specified in the Location header.
* **306 Switch Proxy (Unused):** No longer used; original standard intended to mean "Subsequent requests should use the specified proxy."
* **307 Temporary Redirect:** Redirects to a temporary URI while guaranteeing the request method and body do not change.
* **308 Permanent Redirect:** Redirects to a permanent URI while guaranteeing the request method and body do not change.

## 4xx: Client Error Responses
The request contains bad syntax or cannot be fulfilled due to a client-side error.

* **400 Bad Request:** Malformed request syntax, invalid framing, or request routing deception.
* **401 Unauthorized:** Authentication is required and has failed or has not yet been provided.
* **402 Payment Required:** Reserved for future use; occasionally used by digital payment systems.
* **403 Forbidden:** The client does not have access rights to the content.
* **404 Not Found:** The server cannot find the requested resource.
* **405 Method Not Allowed:** The request method is known by the server but not supported by the target resource.
* **406 Not Acceptable:** The server cannot produce a response matching the list of acceptable values defined in the request's Accept headers.
* **407 Proxy Authentication Required:** Authentication must be done via a proxy first.
* **408 Request Timeout:** The server timed out waiting for the request from the client.
* **409 Conflict:** The request conflicts with the current state of the target resource.
* **410 Gone:** The target resource is permanently deleted and will not be available again.
* **411 Length Required:** The server requires a Content-Length header field.
* **412 Precondition Failed:** One or more conditions given in the request header fields evaluated to false.
* **413 Payload Too Large:** Request payload is larger than limits defined by the server.
* **414 URI Too Long:** The URI requested by the client is longer than the server is willing to interpret.
* **415 Unsupported Media Type:** The payload format is in an unsupported media type.
* **416 Range Not Satisfiable:** The range specified by the Range header in the request cannot be fulfilled.
* **417 Expectation Failed:** The expectation given in the request's Expect header could not be met.
* **418 I'm a Teapot:** An IETF April Fools' joke (RFC 2324); returned by teapots asked to brew coffee.
* **421 Misdirected Request:** The request was directed at a server that is not able to produce a response.
* **422 Unprocessable Content (WebDAV):** Well-formed request, but unable to be followed due to semantic errors.
* **423 Locked (WebDAV):** The resource being accessed is locked.
* **424 Failed Dependency (WebDAV):** The request failed due to failure of a previous request.
* **425 Too Early:** The server is unwilling to risk processing a request that might be replayed.
* **426 Upgrade Required:** The server refuses to perform the request using the current protocol until the client upgrades.
* **428 Precondition Required:** The origin server requires the request to be conditional.
* **429 Too Many Requests:** The user has sent too many requests in a given amount of time ("rate limiting").
* **431 Request Header Fields Too Large:** Server refuses request because individual/all header fields are too large.
* **451 Unavailable For Legal Reasons:** Resource access is denied as a result of legal demands (e.g., censorship/court orders).

## 5xx: Server Error Responses
The server failed to fulfill an apparently valid request.

* **500 Internal Server Error:** A generic error message given when an unexpected condition was encountered.
* **501 Not Implemented:** The server does not support the functionality required to fulfill the request.
* **502 Bad Gateway:** The server, acting as a gateway/proxy, received an invalid response from an upstream server.
* **503 Service Unavailable:** The server is currently unable to handle the request (due to overload or maintenance).
* **504 Gateway Timeout:** The server, acting as a gateway/proxy, did not receive a timely response from the upstream server.
* **505 HTTP Version Not Supported:** The HTTP version used in the request is not supported by the server.
* **506 Variant Also Negotiates:** Internal server configuration error where the chosen variant resource is configured to engage in transparent content negotiation itself.
* **507 Insufficient Storage (WebDAV):** Server is unable to store the representation needed to complete the request.
* **508 Loop Detected (WebDAV):** Server detected an infinite loop while processing the request.
* **510 Not Extended:** Further extensions to the request are required for the server to fulfill it.
* **511 Network Authentication Required:** Indicates that the client needs to authenticate to gain network access (e.g., captive portals).