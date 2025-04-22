# Chapter 3: HTTP Headers

In the previous chapters, we learned about the structure of [HTTP Message (Request/Response)](01_http_message__request_response__.md) and how the [URI (Uniform Resource Identifier)](02_uri__uniform_resource_identifier__.md) acts as the address. Now, let's talk about the extra instructions and details that travel with these messages: **HTTP Headers**.

## What Problem Do Headers Solve? The Need for Metadata

Imagine sending a package. Besides the address (URI) and the contents (body), you need a label with extra information: Who sent it? What's inside (fragile? documents?) How much does it weigh? Should the recipient sign for it?

Similarly, when a client sends an `HTTPRequest` or a server sends an `HTTPResponse`, just the action (method), address (URI), and content (body) aren't always enough. They need a way to send extra information – *metadata* – about the request or response itself. This metadata helps both sides understand how to handle the message correctly.

For example:
*   How does your browser tell the server it prefers HTML content?
*   How does the server tell your browser the content *is* HTML?
*   How does the server tell the browser which website it thinks the request is for (especially if the server hosts multiple sites)?
*   How does the server send back a "cookie" to remember you for the next visit?

This is where **HTTP Headers** come in.

## What are HTTP Headers? Key-Value Instructions

HTTP Headers are like the specific instructions or fields on that shipping label or an order form. They are **key-value pairs** that are sent along with both requests and responses.

Think of each header as a label: `Key: Value`

*   **Key:** The name of the piece of information (e.g., `Host`, `Content-Type`).
*   **Value:** The actual information itself (e.g., `google.com`, `text/html`).

These headers provide crucial context, such as:
*   **Content Type:** What kind of data is in the message body (e.g., HTML, JSON, image)? (`Content-Type: text/html`)
*   **Content Length:** How big is the message body in bytes? (`Content-Length: 1024`)
*   **Connection Status:** Should the network connection stay open after this message? (`Connection: keep-alive`)
*   **Cookies:** Small pieces of data sent by the server for the client to store and send back later. (`Set-Cookie: sessionid=abcde`)
*   **Target Host:** Which website does the client want to reach on the server? (`Host: www.example.com`)

## The `Headers` Collection in `lightbug_http`

In `lightbug_http`, headers are managed using the `Headers` struct (defined in `lightbug_http/header.mojo`). This struct acts like a collection or dictionary specifically designed to hold header information. Each individual header is represented by a `Header` struct.

**Creating Headers:**

You usually create a `Headers` collection by passing one or more `Header` objects to its initializer.

```mojo
from lightbug_http import Header, Headers, HeaderKey

# Create individual Header objects
var host_header = Header(HeaderKey.HOST, "example.com")
var type_header = Header(HeaderKey.CONTENT_TYPE, "text/plain")

# Create a Headers collection containing these headers
var my_headers = Headers(host_header, type_header)

# Print the collection (note keys are often lowercased internally)
print(my_headers)
# Possible Output:
# host: example.com
# content-type: text/plain
```

*Explanation:*
We create two `Header` structs, one for `Host` and one for `Content-Type`. We then pass them to the `Headers()` initializer to create a collection. The `HeaderKey` struct provides convenient aliases for common header names (like `HeaderKey.HOST`), but you can also just use strings like `"Host"`. Notice that when printed, the keys (`host`, `content-type`) are typically stored and compared in lowercase for consistency, as HTTP header keys are case-insensitive.

**Accessing Headers:**

You can access the value of a specific header using dictionary-like square brackets `[]`. Remember to use the lowercase key name for consistency, or use the `HeaderKey` aliases.

```mojo
from lightbug_http import Header, Headers, HeaderKey

# Assume 'response_headers' is a Headers object received from a server
var response_headers = Headers(
    Header("Content-Type", "application/json"),
    Header("Content-Length", "128"),
)

# Get the value of the Content-Type header
try:
    var content_type = response_headers[HeaderKey.CONTENT_TYPE] # Use HeaderKey alias
    print("Content-Type:", content_type) # Output: Content-Type: application/json
except e:
    print("Content-Type header not found.")

# Check if a header exists
if HeaderKey.CONTENT_LENGTH in response_headers: # Use 'in' operator
    print("Content-Length is present.") # Output: Content-Length is present.
else:
    print("Content-Length is not present.")

# Get a header value safely using .get() -> Optional[String]
var server_header = response_headers.get(HeaderKey.SERVER) # HeaderKey.SERVER = "server"
if server_header:
    print("Server:", server_header.value())
else:
    print("Server header not found.") # Output: Server header not found.
```

*Explanation:*
We access the `Content-Type` header using `response_headers[HeaderKey.CONTENT_TYPE]`. Since accessing a non-existent key raises an error, we often use a `try...except` block or check for existence first using the `in` keyword. A safer way is to use the `.get()` method, which returns an `Optional[String]`, preventing errors if the header is missing.

**Adding/Modifying Headers:**

You can add or change headers in a `Headers` collection using the same square bracket notation.

```mojo
from lightbug_http import Headers, HeaderKey

# Start with an empty Headers collection
var request_headers = Headers()

# Add a Host header
request_headers[HeaderKey.HOST] = "api.example.com"

# Add an Accept header
request_headers["Accept"] = "application/json" # Using a string key directly

# Modify the Accept header
request_headers["accept"] = "*/*" # Keys are case-insensitive

print(request_headers)
# Possible Output:
# host: api.example.com
# accept: */*
```

*Explanation:*
We create an empty `Headers` collection and then add `Host` and `Accept` headers using `[] =`. Modifying the `accept` header (using lowercase) updates the existing entry because keys are treated case-insensitively.

## Headers in Action: `HTTPRequest` and `HTTPResponse`

As we saw in [Chapter 1: HTTP Message (Request/Response)](01_http_message__request_response__.md), both `HTTPRequest` and `HTTPResponse` structs have a dedicated field to store their headers:

**Simplified `HTTPRequest`:**

```mojo
# From: lightbug_http/http/request.mojo (Simplified)
@value
struct HTTPRequest:
    var headers: Headers  # <-- Headers collection!
    var uri: URI
    var body_raw: Bytes
    var method: String
    var protocol: String
    # ... other fields ...
```

**Simplified `HTTPResponse`:**

```mojo
# From: lightbug_http/http/response.mojo (Simplified)
@value
struct HTTPResponse:
    var headers: Headers  # <-- Headers collection!
    var body_raw: Bytes
    var status_code: Int
    var status_text: String
    var protocol: String
    # ... other fields ...
```

*Explanation:*
Both structures contain a `headers: Headers` field. When you create a request or receive a response, you interact with this field to set or read the metadata associated with the message.

When `lightbug_http` sends a request or response over the network, it takes the `Headers` collection and formats it into the standard text-based HTTP header block (one `Key: Value` per line). When it receives a message, it parses this text block back into a `Headers` object.

## Under the Hood: Parsing Header Text

How does `lightbug_http` turn the raw text received from the network into a useful `Headers` object?

1.  **Identify Header Block:** When reading an incoming message (like in `HTTPRequest.from_bytes` or `HTTPResponse.from_bytes`), the parser first reads the initial line (Request-Line or Status-Line). Then, it reads subsequent lines until it encounters an empty line (typically signaled by a double CRLF: `\r\n\r\n`). These lines between the first line and the empty line constitute the header block.
2.  **Split Key/Value:** For each line in the header block, the parser finds the first colon (`:`). The text before the colon is the header key, and the text after it (often stripping leading/trailing whitespace) is the header value.
3.  **Normalize Key:** The extracted header key is usually converted to lowercase. This ensures that lookups (like `headers["Host"]` or `headers["host"]`) work consistently, regardless of how the sender capitalized the key.
4.  **Store in Dictionary:** The lowercase key and the corresponding value are stored as a pair in the internal dictionary (`_inner: Dict[String, String]`) within the `Headers` object. Special headers like `Set-Cookie` might receive slightly different handling because multiple `Set-Cookie` headers are allowed.

This parsing logic primarily resides within the `Headers.parse_raw` method (in `lightbug_http/header.mojo`), which is called internally by the `from_bytes` methods of `HTTPRequest` and `HTTPResponse`.

```mojo
# Simplified concept from lightbug_http/header.mojo

@value
struct Headers:
    # Internal storage for headers (lowercase key -> value)
    var _inner: Dict[String, String]

    # Simplified parsing logic used by request/response 'from_bytes'
    fn parse_raw(mut self, mut reader: ByteReader) raises -> ...:
        # (Assume first line like "GET / HTTP/1.1" or "HTTP/1.1 200 OK" was already read)

        # Loop through lines until an empty line is found
        while not is_newline(reader.peek()): # is_newline checks for '\r' or '\n' indicating end
            # Read 'Key' part up to the colon ':'
            key_bytes = reader.read_until(BytesConstant.colon)
            reader.increment() # Skip the ':'

            # Skip optional leading space after colon
            if is_space(reader.peek()):
                reader.increment()

            # Read 'Value' part up to the end of the line (CRLF)
            value_bytes = reader.read_line()

            # Convert to strings and normalize key to lowercase
            key_string = String(key_bytes).lower()
            value_string = String(value_bytes)

            # Store the key-value pair in the internal dictionary
            # (Special handling for repeated headers like Set-Cookie omitted for simplicity)
            self._inner[key_string] = value_string

        # Skip the final empty line (CRLF) that marks the end of headers
        reader.skip_carriage_return()

        # (... return other info like protocol, status, etc. ...)

```

*Explanation:*
This conceptual code shows how the parser reads each header line using a `ByteReader`. It splits the line at the colon, converts the key to lowercase, and stores the key-value pair in the `_inner` dictionary. This happens for every header line until the empty line separating headers from the body is reached.

## Conclusion

You've now learned about HTTP Headers, the essential metadata accompanying web requests and responses!

*   Headers are **key-value pairs** (e.g., `Header("Host", "google.com")`) that provide extra context for HTTP messages.
*   They convey information like content type, length, connection state, cookies, and the target host.
*   `lightbug_http` uses the `Headers` struct (a collection, often behaving like a dictionary with case-insensitive keys) to manage these pairs within `HTTPRequest` and `HTTPResponse` objects.
*   You can easily **create**, **access**, and **modify** headers using the `Headers` struct and its methods.
*   `lightbug_http` handles the **parsing** of text-based headers into `Headers` objects and **formatting** them back into text for network transmission.

Understanding headers is vital because they control many aspects of how clients and servers interact and interpret messages.

In the next chapter, we'll start putting these pieces together and see how to build a basic web server using `lightbug_http`.

Next: [Chapter 4: HTTP Server](04_http_server_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)