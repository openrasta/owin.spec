# OWIN: Open Web Server Interface for .NET

**Version**
1.0.1-draft.1

**Authors**
[OWIN working group][WGForum] /* Add list of authors from original spec */
Sebastien Lambla (Caffeine IT)
seb@serialseb.com

**License**
[Creative Commons Attribution 3.0 Unported License][CCLicense]

**Last updated**
18 March 2015  

## Abstract

This document defines OWIN, a standard interface between .NET web servers and web applications.

## Status

This is a working draft.

This document is a submission to the  OWIN Working Group and is valid for a
period of six months and may be updated, replaced, or obsoleted by other
documents at any time. It is inappropriate to use working drafts as reference
material or to cite them other than as "work in progress"

This working draft will expire on 23 September 2015.

## Copyright

Copyright (c) 2014-2015 the persons identified as the document authors. The work
is licensed as per the [Creative Commons Attribution 3.0 Unported License]
[CCLicense] in effect on the date of publication of this document.

## Table of Content

1. [Introduction](#1-introduction)  
  1.1. [Requirements Notation](#11-requirements-notation)
2. [Definitions](#2-definitions)
3. [Request Execution](#3-request-execution)
  3.1. [Application Delegate](#31-application-delegate)
  3.2. [Environment](#32-environment)
  3.3. [Headers][sec-headers]
  3.4. [Request Body][sec-req-body]
  3.5. [Response Body][sec-res-body]
  3.6. [Request Lifetime](#36-request-lifetime)
4. [Application Startup](#4-application-startup)
5. [URI Reconstruction](#5-uri-reconstruction)
  5.1. [URI Scheme][sec-uri-scheme]
  5.2. [Hostname][sec-hostname]
  5.3. [Paths][sec-paths]
  5.4. [URI Reconstruction Algorithm](#54-uri-reconstruction-algorithm)
  5.5. [Percent-encoding](#55-percent-encoding)
6. [Error Handling](#6-error-handling)
  6.1. [Application Errors](#61-application-errors)
  6.2. [Server Errors](#62-server-errors)
7. [Versioning][sec-versioning]


## 1. Introduction

Historically, web application development has relied on vendor-specific APIs and hosting solutions, often tightly coupled to one another. As open-source alternatives have increased in numbers and popularity, supporting different hosting APIs has become a challenge.

This specifications defines OWIN, the Open Web Interfaces for .net. OWIN's goal is to decouple server and application and, by being an open standard, stimulate the open-source ecosystem of .NET web development tools.

OWIN is defined in terms of its lambda, to allow interoperation without relying on compiled binaries.

In this document, the C# `Action`/`Func` syntax is used to notate some delegate structures. However, the delegate structure could be equivalently represented with F# native functions, CLR interfaces, named delegates, object methods or any other mean, but MUST be assignable to the normative lambda.


### 1.1. Requirements Notation

 The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119][rfc2119].

A component is deemed to be compliant with this specification if all absolute
requirements and prohibitions are conformed to.

## 2. Definitions

This document refers to the following::

1. **Server** – An HTTP server that directly communicates with the client, and uses OWIN semantics to process requests.  Servers not supporting OWIN natively may require an adapter layer that allows them to implement OWIN.

2. **Application** – A specific application, possibly built on top of a web framework, exposed as an OWIN component and hosted by an OWIN compatible Servers.

> Note: need to add Client and Server as per RFC 7230

3. **User Agent** - A process responsible for interacting with a web server, by establishing an HTTP connection to the Web Server

4. **Environment** - A bag of data, populated by the User Agent's HTTP request and the Web Application's HTTP response, and any other environmental and server state associated with the execution of the Request-Response exchange.

5. **HTTP Headers** - Add note to RFC 7230

6. **Request/Response Entity Body** - same

7. Interaction - A process starting with an HTTP Request and completing with an HTTP response or the termination of the Client's connection.

## 3. Request Execution

Upon connection from a user agent, a Web Server creates an Environment and populates it with the data received from the client. The Web Application is then invoked, resulting in either a response, or a failure to execute. Depending on the web server MAY transmit the response, transmit a new response with a representation of the failure, or terminate the User Agent's connection.

### 3.1. Application Delegate

The primary interface is called the application delegate or `AppFunc`. An application delegate takes the `IDictionary<string, object>` Environment and returns a `Task`, used to signal it has finished processing.

```c#
using AppFunc = Func<
    IDictionary<string, object>, // Environment Properties
    Task>; // Done
```

> An Application MUST complete the returned Task, or throw an exception.
> A Server MAY detect Applications that do not complete the Task and SHOULD take appropriate measures to avoid leaving connections to User Agents opened upon an Application's failure.

### 3.2. Environment

Environment Properties contain information about the request, the response, and any relevant environmental and server state. The Server MUST have initialized the HTTP headers and Entity Bodies for both requests and responses before invoking the Application.

The application then populates the appropriate fields with response data, writes the response body, and returns when done.

> * The environment dictionary MUST be non-null, mutable and MUST contain the keys listed as required in the tables below.

> * Keys MUST be compared using a byte-by-byte comparison (e.g. `StringComparer.Ordinal`)

> * The values associated with the keys defined by this specification MUST be non-null, unless otherwise explicitly specified.

In addition to these keys, Servers and Applications CAN add arbitrary data associated with the request or response to the environment dictionary, see the [Environment and Startup properties](CommonKeys).

### 3.2.1 Request Data

| Required | Key Name                  | Value Description |
|----------|---------------------------|-------------------|
| **Yes**  | `owin.RequestBody`        | A `Stream` with the request body, if any. `Stream.Null` MAY be used as a placeholder if there is no request body. See [Request Body](sec-req-body). |
| **Yes**  | `owin.RequestHeaders`     | An `IDictionary<string, string[]>` of request headers. See [Headers][sec-headers]. |
| **Yes**  | `owin.RequestMethod`      | A `string` containing the HTTP request method of the request (e.g., `"GET"`, `"POST"`). |
| **Yes**  | `owin.RequestPath`        | A `string` containing the request path. The path MUST be relative to the "root" of the application delegate. See [Paths][sec-paths]. |
| **Yes**  | `owin.RequestPathBase`    | A `string` containing the portion of the request path corresponding to the "root" of the application delegate; see [Paths][sec-paths]. |
| **Yes**  | `owin.RequestProtocol`    | A `string` containing the protocol name and version (e.g. `"HTTP/1.0"` or `"HTTP/1.1"`). |
| **Yes**  | `owin.RequestQueryString` | A `string` containing the query string component of the HTTP request URI, without the leading "?" (e.g., `"foo=bar&amp;baz=quux"`). The value may be an empty string. |
| **Yes**  | `owin.RequestScheme`      | A `string` containing the URI scheme used for the request (e.g., `"http"`, `"https"`); see [URI Scheme][sec-uri-scheme]. |

### 3.2.2 Response Data

| Required | Key Name                  | Value Description |
|----------|---------------------------|-------------------|
| **Yes**  | `owin.ResponseBody`       | A `Stream` used to write out the response body, if any. See [Response Body][sec-res-body]. |
| **Yes**  | `owin.ResponseHeaders`    | An `IDictionary<string, string[]>` of response headers. See [Headers][sec-headers]. |
| **No**   | `owin.ResponseStatusCode` | An optional `int` containing the HTTP response status code as defined in [RFC 2616][rfc2616] section 6.1.1. The default is 200. |
| **No**  | `owin.ResponseReasonPhrase`| An optional `string` containing the reason phrase associated the given status code. If none is provided then the server SHOULD provide a default as described in [RFC 2616][rfc2616] section 6.1.1 |
| **No**  | `owin.ResponseProtocol`    | An optional `string` containing the protocol name and version (e.g. `"HTTP/1.0"` or `"HTTP/1.1"`). If none is provided then the `"owin.RequestProtocol"` key's value is the default. |

### 3.2.3 Other Data

| Required | Key Name                  | Value Description |
|----------|---------------------------|-------------------|
| **Yes**  | `owin.CallCancelled`      | A `CancellationToken` indicating if the request has been canceled/aborted. See [Request Lifetime][sec-req-lifetime]. |
| **Yes**  | `owin.Version`            | A `string` indicating the OWIN version. See [Versioning][sec-versioning]. |

### 3.3. Headers

The headers of HTTP request and response messages are represented by objects of type `IDictionary<string, string[]>`. The requirements below are predicated on [RFC 2616 section 4.2][rfc2616-42].

> * The dictionary MUST be mutable.

> * Keys MUST be HTTP field-names without `':'` or whitespace.

> * Keys MUST be compared byte by byte in a case-insensitive fashion (eg. `StringComparer.OrdinalIgnoreCase`).

> * All characters in keys and value strings SHOULD be within the ASCII range of characters, as per RFC7xxxx.

> * Each value in the value array represents a single field line in the origin HTTP message, with the dictionary key being the field name.
    For example, the code `headers["Accept"] = new[] {"text/html", "text/plain"}` represents two field lines in an HTTP message:
    ```http
    Accept: text/html
    Accept: text/plain
    ```
    Servers SHOULD NOT split, merge or otherwise unnecessarily modify the http header dictionaries, as applications may have incorrectly associated semantics to the order or number of multi-value headers.

### 3.4. Request body and 100 Continue

If the request indicates there is an associated body the server SHOULD provide a `Stream` in the `"owin.RequestBody"` key to access the Request Entity Body data. `Stream.Null` MAY be used as a placeholder if there is no request body data expected.

The Expect header is often used for getting an early response from the Server before stating sending the Request Entity Body (e.g. when authenticated to a server, a User-Agent may want to verify its credentials allow it to upload a file before sending the file itself).

Applications SHOULD ignore the Expect header, and do any authorization or other processes as if the header was not present. The Server SHOULD send a `100 Continue` at the point the application starts reading the request stream. Applications MUST NOT set `"owin.ResponseStatusCode"` to `100` and servers SHOULD NOT honor such response codes should an application not implement this specification correctly, and SHOULD alert system administrators / developers of the error.

An Application that reads the Request Entity Body SHOULD NOT complete its returned `Task` and return control to the server until it is finished reading the body. Once the `AppFunc` `Task` is complete the application SHOULD NOT continue to read from the request stream.

 > NOTE: What does a server do in the face of a partial read by the app, I expect the server ought to finish reading the request entity body before writing a response, or kill the connection. See https://tools.ietf.org/html/rfc7230#section-6.5, we should describe how this maps to Servers behavior.

### 3.5 Completion semantics

The application MUST signal completion through its returned `Task`, or by throwing an exception. After completing the `Task`, the application SHOULD NOT write any further data to the Response stream.

Should the server signal the `"owin.CallCancelled"` `CancellationToken` during the execution of the application delegate, the application SHOULD NOT attempt further reads from the stream, and SHOULD promptly complete the Application's `Task`.

The application SHOULD NOT close or dispose the Request Entity Body stream unless it has completely consumed the request body. The owner of a Request Stream MUST do any necessary cleanup once the application delegate's Task completes.

> * Any exception thrown from the request body stream is fatal and SHOULD be returned to the server by being thrown synchronously from the `AppFunc` or by failing the Application `Task` with the given exception(s).

### 3.5. Response Body

The server provides a response body `Stream` with the `"owin.ResponseBody"` key in the initial environment dictionary. The Response Message, including headers, status code and reason phrase, can be modified up until the first write to the Response Entity Body. Upon the first write, the Server MUST send the headers as they exist at that point in time. Applications MAY choose to buffer response data to delay the header finalization.

The application MUST signal completion or failure of the body by completing its returned `Task` or throwing an exception. After completing the `Task`, the application SHOULD NOT write any further data to the stream.

> * If the Server signals the `"owin.CallCancelled"` `CancellationToken` during the execution of the Application, the Application SHOULD NOT attempt further writes to the Response Entity Body, and SHOULD complete the application delegate `Task`.

> * The application SHOULD NOT close or dispose the Response Entity Body stream, as other components may append additional data. The stream owner MUST perform any necessary cleanup once the application delegate's `Task` completes.

> QUESTION: this cannot possibly be normative? An application developer could not possibly do that, neither does "supporting multiple outstanding asynchronous writes" mean much, as there is no reference. I suggest removing the clause alltogether, or being more normative in what this means and what "verify xxx support this pattern" actually mean.

(An application SHOULD NOT assume the given stream supports multiple outstanding asynchronous writes. The application developer SHOULD verify that the server and all middleware in use support this pattern before attempting to use it.)

### 3.6. Interaction lifetime

The lifetime of an Interaction is dependent on the Application, the Client and the Server.

In the common scenario, a request's lifetime ends when the Application has completed and the Server gracefully ends the request. Failures at any level may cause the request to terminate prematurely, or may be handled internally, allowing the request to continue.

> QUESTION: "ends the request" is awfully sounding like "terminate the request", and I'm pretty sure thats not what was meant.
> QUESTION: "allowing the request to continue?" and "any level" is imprecise and i don't understand what it's trying to say.

The `"owin.CallCancelled"` key is associated with a `CancellationToken` that the server uses to signal if the request has been aborted. This SHOULD be triggered if the request becomes faulted before the `AppFunc` `Task` is completed. It MAY be triggered at any point at the providers discretion. Middleware MAY replace this token with their own to provide added granularity or functionality, but they SHOULD chain their new token with the one originally provided to them.

## 4. Application Startup

When the host process starts there are a number of steps it goes through to set up the application.

1. The host creates a Properties `IDictionary<string, object>` and populates any startup data or capabilities provided by the host.

2. The host selects which server will be used and provides it with the Properties collection so it can similarly announce any capabilities.

3. The host locates the application setup code and invokes it with the Properties collection.

4. The application reads and/or sets configuration in the Properties collection, constructs the desired request processing pipeline, and returns the resulting application delegate.

5. The host invokes the server startup code with the given application delegate and the Properties dictionary. The server finishes configuring itself, starts accepting requests, and invokes the application delegate to process those requests.

The Properties dictionary may be used to read or set any configuration parameters supported by the host, server, middleware, or application.

> * The startup properties dictionary MUST be non-null, mutable and MUST contain the keys listed as required in the table below.

> * Keys MUST be compared using StringComparer.Ordinal.

> * The values associated with the keys MUST be non-null, unless otherwise specified.

> | Required | Key Name       | Value Description |
|----------|----------------|-------------------|
| **Yes**  | `owin.Version` | A `string` indicating the OWIN version. Added by the server in step 2 above. See [Versioning][sec-versioning]. |

In addition to these keys the host, server, middleware, application, etc. may add arbitrary data associated with the application configuration to the properties dictionary. Guidelines for additional keys and a list of commonly defined keys can be found in the [CommonKeys addendum][CommonKeys] to this spec.

## 5. URI Reconstruction

Applications often require the ability to reconstruct the complete URI of a request. This process cannot be perfect since HTTP clients do not usually transmit the complete URI which they are requesting, but OWIN makes provisions for the purpose of reconstructing the approximate URI of a request.

### 5.1. URI Scheme

This information is usually not transmitted by an HTTP client and depending on network configuration it may not be possible for an OWIN server to determine a correct value. In these cases, the server may have to manually configure or compute a value.

> Servers MUST provide a best-guess value for `"owin.RequestScheme"`.

### 5.2. Hostname

In the context of an HTTP/1.1 request, the name of the server to which the client is making a request is usually indicated in the Host header field-value of the request, although it might be specified using an absolute Request-URI (see RFC 2616, sections [5.1.2][rfc2616-512], [19.6.1.1][rfc2616-19611]).

> A server MUST provide a value for the `"Host"` key in the request header dictionary. The format of the value MUST be `"<hostname>[:<port>]"`. The value SHOULD be deduced by the host using the following steps:
>
> 1. If the Request-URI of the incoming request is an absolute URI, the value of the `"Host"` key MUST be taken from the host part of the absolute URI.
> 2. If the Request-URI of the incoming request is not an absolute URI, the value of the `"Host"` key MUST be taken from the Host header field-value of the incoming request.
> 3. If the Host header is not present in the incoming request (as in an HTTP/1.0 request), or if its value consists entirely of linear whitespace, the server MUST provide a sensible best-guess value for the `"Host"` key.

### 5.3. Paths

Servers may have the ability to map application delegates to some base path. For example, a server might have an application delegate configured to respond to requests beginning with `"/my-app"`, in which case it should set the value of `"owin.RequestPathBase"` in the environment dictionary to `"/my-app"`. If this server receives a request for `"/my-app/foo"`, the `"owin.RequestPath"` value of the environment dictionary provided to the application configured to respond at `"/my-app"` should be `"/foo"`.

> * The value associated with the environment key `"owin.RequestPathBase"` MUST NOT end with a slash and MUST either start with a slash or be `String.Empty`.

> * The value associated with the environment key `"owin.RequestPath"` MUST start with a slash, or MAY be `String.Empty` if the value associated with `"owin.RequestPathBase"` is not `String.Empty`.

### 5.4. URI Reconstruction Algorithm

The following algorithm can be used to approximate the complete URI of the current request:

```c#
var uri =
   (string)Environment["owin.RequestScheme"] +
   "://" +
   Headers["Host"].First() +
   (string)Environment["owin.RequestPathBase"] +
   (string)Environment["owin.RequestPath"];

if(Environment["owin.RequestQueryString"] != "")
{
   uri += "?" + (string)Environment["owin.RequestQueryString"];
}
```

The result of this algorithm may not be identical to the URI the client used to make the request; for example, the server may have done some rewriting to canonicalize the request. Further, it is subject to the caveats described in the [URI Scheme][sec-uri-scheme] and [Hostname][sec-hostname] sections above.

### 5.5. Percent-encoding

A URI uses percent encoding to transport characters outside its normally allowed character rules ([RFC 3986][rfc3986] section 2). Percent-encoding is used to express the underlying octets present in a URI component, whose octets are interpreted via UTF-8 encoding. Most web server implementations will perform percent-decoding on paths in order to perform request routing (as they rightly may, see: RFC 2616 section [5.1.2][rfc2616-512], also [3.2.3][rfc2616-323]), and OWIN follows this precedent. The request query string in OWIN is presented in its percent-encoded form; a percent-decoded query string could contain `'?'` or `'='` characters, which would render the string unparseable.

> * The server MUST provide percent-decoded values for `"owin.RequestPath"` and `"owin.RequestPathBase"`.

> * The server MUST provide a percent-encoded value for `"owin.RequestQueryString"`

## 6. Error Handling

While there are standard exceptions such as `ArgumentException` and `IOException` that may be expected in normal request processing scenarios, handling only such exceptions is insufficient when building a robust server or application. If a server wishes to be robust it SHOULD consistently address all exception types thrown or returned from the application delegate or body delegate. The handling mechanism (e.g. logging, crashing and restarting, swallowing, etc.) is up to the server and host process.

### 6.1. Application Errors

An application may generate an exception in the following places:

* Thrown from an invocation of the application delegate.
* Provided as the result of the application delegate `Task`.

An application SHOULD attempt to trap its own internal errors and generate an appropriate (possibly 500-level) response rather than propagating an exception up to the server.

After an application provides a response, the server SHOULD wait to receive at least one write from the response `Stream` before writing the response headers to the underlying transport. In this way, if instead of a write to the `Stream` the server gets an exception from the application delegate `Task`, the server will still be able to generate a 500-level response. If the server gets a write to the `Stream` first, it can safely assume that the application has caught as many of its internal errors as possible; the server can begin sending the response. If an exception is subsequently received, the server MAY handle it as it sees fit (e.g. logging, write a textual description of the error to the underlying transport, and/or close the connection).

### 6.2. Server Errors

When a server encounters errors during a request's lifetime it SHOULD signal the `CancellationToken` it provided in `"owin.CallCancelled"`. The server MAY then take any necessary actions to terminate the request, but it SHOULD be tolerant of completion delays of the application delegate.

## 7. Versioning

Future updates to this standard may contain breaking changes (e.g. signature changes, key additions or modifications, etc.) or non-breaking additions. While addressing specific changes is the responsibility of later versions of the standard, here are initial guidelines for expected changes:

* This standard uses [Semantic Versioning][semver] (e.g. `Major.Minor.Patch`).

* Breaking changes to the API signatures or existing keys will require incrementing the major version number (e.g. OWIN 2.0)

* Adding new keys or delegates in a backwards compatible way only requires incrementing the minor version number (e.g. OWIN 1.1).

* Making corrections and clarifications to the document alone only requires incrementing the patch version number and last modified date (e.g. OWIN 1.0.1).

* The `"owin.Version"` key in the startup Properties and request Environment dictionaries indicates the latest version of the standard implemented by the server and may be used to dynamically adjust behaviors of applications.

* All implementers SHOULD clearly document the full version number(s) of the OWIN standard they support.

* The keys listed in the [CommonKeys addendum][CommonKeys] to this spec are strictly optional. Additions may be made there without directly affecting the OWIN standard or version number.


## 8. Changelog

 - Requirements haven't changed, but references covered by other specifications removed
 - host has been removed
 - Added mention of Server detecting Task non-completion
 - Rules on versioning updated to discourage semver
 - Added Startup Properties and renamed environment dictionary to properties
 - Relaxed specification on the committing of header values, the rule probably wasn't intended to only specify the indexer
 - Removed section on application, as this is vendor-specific, isnt normative and is out of context for this specification.

 headers
 - Rewrote the part about #LIST header values
 - Removed discussion about developer rights, replaced by normative language for server implementations.
 - Removed notion that arrays ought to support multiple values, they do, we're good.
 3.5
   Removed "failure of response body" because it has no meaning in this spec.
 - Cleaned-up references to ASCII
 - Updated to RFC-style format
 - Updated language
 - Rewrote introduction
----

[WGForum]: https://github.com/owin/owin.github.com/issues
[CCLicense]: http://creativecommons.org/licenses/by/3.0/deed.en_US
[CommonKeys]: http://owin.org/spec/CommonKeys.html
[sec-req-body]: #34-request-body-100-continue-and-completed-semantics
[sec-res-body]: #35-response-body
[sec-headers]: #33-headers
[sec-paths]: #53-paths
[sec-uri-scheme]: #51-uri-scheme
[sec-hostname]: #52-hostname
[sec-versioning]: #7-versioning
[semver]: http://semver.org/
[rfc2119]: http://www.ietf.org/rfc/rfc2119.txt
[rfc2616]: http://www.ietf.org/rfc/rfc2616.txt
[rfc3986]: http://www.ietf.org/rfc/rfc3986.txt
[rfc2616-42]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec4.html#sec4.2
[rfc2616-512]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec5.html#sec5.1.2
[rfc2616-19611]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec19.html#sec19.6.1.1
[rfc2616-323]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.2.3
