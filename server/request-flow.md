# Node.js Solid server: HTTP request handling flow

This document specifies the flow for handling HTTP requests
as we would want it in a future version of node-solid-server.
It aims to minimize the knowledge each component needs,
and to achieve a layered system.

When an agent sends an HTTP request,
the Solid server should take the following steps.

## Step 1: Determine whether the request is directed to the personal data store
Based on the URL path, the server distinguishes between two kinds of incoming requests:
1. **Personal data store requests**, which can follow arbitrary URL patterns
2. **Account and server management requests**, which follow predefined URL patterns such as `/.well-known/`, `/.auth/`, etc.

Requests of the second kind are sent to specifically wired handlers,
which typically only accept `GET`, `HEAD`, `OPTIONS`, and `POST`.
<br>
Personal data store requests follow the steps below.

## Step 2: Parse the request to the personal data store
The HTTP request's URL, headers, and optional body and/or client certificate
are parsed into a structure consisting of the following components:

- The **target** identifies the resource that is the subject of the request.
  It corresponds to the full request URL
  (taking into account protocol and host)
  after normalization (removal of dot segments)
  and a sanity check.

- The **body** is a parsed version of the request body, which can be empty.
  The server has access to a list of parsers per content type,
  such as `text/turtle`, `application/sparql-update`.
  Parsing should be lazy,
  such that the contents are only created when their parsed form is needed
  (which might not be the case, for instance, with `PUT` requests).
  If no parser for the given content type is available,
  the object is considered a textual blob (if a charset was given),
  or a binary blob.

- The **method** is the HTTP method (an uppercase string).
  Valid options are
  `GET`, `HEAD`, `OPTIONS`, `POST`, `PUT`, `PATCH`, `DELETE`.

- The **required permissions** are a set of flags
  _read_, _append_, _write_, and _delete_.
  They are set to _true_ or _false_ based on the method
  (and, in the case of `PATCH`, the parsed body).
  If _write_ is set to _true_, so should _append_.

- The **credentials** are an object
  with a WebID field indicating the URL of the logged-in agent,
  which is empty if no agent is logged in.
  To determine this value from the request headers and/or client certificate,
  the server has access to a list of authenticators,
  such as WebID-TLS (which reads the WebID from a client certificate)
  and OIDC (which reads the WebID based on HTTP headers).

- The **preferences** are a key/value object of categorized lists
  that indicate the agent's representation preferences,
  parsed from the HTTP headers.
  They include media type and language preferences,
  and are parsed along with their `q` values.

Minor parsing failures at this stage
are automatically corrected through the insertion of defaults.
For example, if preferences are missing or invalid,
the server could assume `text/turtle` and `en`;
and missing authentication results in no WebID being set on the credentials.

For major parsing failures, the server responds with:
- `405` if the method is invalid
- `413` if the body is too large
- `415` when no body parser was found for a `PATCH` request
- `400` in other cases

When possible,
the body of this response should respect
the agent's representation preferences.

## Step 3: Verify the agent's permissions
The server passes the **credentials**, **target**, and the **required permissions**
to the authorization component,
which determines whether the agent has sufficient permissions.

If validation fails, the server responds with:
- `401` if the agent did not authenticate (no WebID set in credentials)
- `403` if the agent does not have the appropriate permissions
- `404` if the server prefers not to disclose the existence of the resource

When possible,
the body of this response should respect
the agent's representation preferences.
In case of `401` and `403` responses,
the body should include a link to authenticate.

The authorization component might or might not be the component
that provides access to the underlying data store.

## Step 4: Perform the modification
This step is only executed
when **required permissions** includes `append`
(or `write`, which assumes `append`).

The server passes the **target**, **method**, and **body**
to the data storage component.
This component will attempt to perform the modification.

In case of a successful `PUT`, `PATCH`, or `DELETE`,
the server responds with a `204`.

If case of successful resource creation through `POST`,
**resource** is set to the newly created resource identifier.

In case of failure, server responds with:
- `404` if the resource does not exist (anymore)
- `409` if the `PATCH` operation could not be performed
- `412` if the resource has been modified since the client's last access
- `500` in other cases

When possible,
the body of this response should respect
the agent's representation preferences.

## Step 5: Return a representation
In case of `POST`, the server responds with:
- `201` and a `Location` header of **resource**.

The server passes **target** and **preferences** to the data store
in order to retrieve metadata and a representation of the resource.

The server responds with:
- `205` if **method** is `PUT` or `PATCH`
- `200` if an adequate representation was found
- `406` otherwise

When possible,
the body of this response should respect
the agent's representation preferences.

## Returning error information

Error reporting should always be as articulate as possible. 
Wherever there is information which will help the user, or the client, developer, or the server administrator,
to understand what has happened, it should be shared with the client.

Any unexpected runtime exception should be caught and will generally return a

- `500` status

Possibly, there should be two modes as a function of the server configuration, in one of which 
also included with a 500 is a complete stack trace, so that the development team 
get an accurate idea of what the new bug is.  

Do not assume that users are not technical and therefore should not be given the technical details of an error.
They may still find that information useful, e.g. in remembering similar errors, looking them up on the web, or sharing them with more technically minded people.

We could for instance rely on [RFC7807](https://tools.ietf.org/html/rfc7807),
possibly by adding a JSON-LD context to it,
and providing this result in different formats through content negotiation.
