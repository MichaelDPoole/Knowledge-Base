# Chapter 2: URI (Uniform Resource Identifier)

In [Chapter 1: HTTP Message (Request/Response)](01_http_message__request_response__.md), we learned that an `HTTPRequest` is like a digital letter asking a server for something. One crucial part of that letter is the **address** – where does the server live, and what specific thing are we asking for? This address is called a **URI (Uniform Resource Identifier)**.

Imagine you want to send a physical letter. Just writing "John Smith" isn't enough. You need the street, city, state, and zip code to make sure it gets to the right place. Similarly, on the web, we need a precise address to find a specific resource (like a webpage, image, or data).

## What is a URI? The Web's Mailing Address

Think of a URI as a detailed mailing address for anything on the web. It's a string of characters that uniquely identifies a resource. A common type of URI you see every day is a URL (Uniform Resource Locator), which is basically a URI that also tells you *how* to get the resource (like using HTTP or HTTPS).

Let's look at a typical web address (a URL, which is a specific kind of URI):

`https://www.example.com:443/products/shoes?color=red&size=10#details`

This looks complicated, but it's just a structured address with different parts, each telling the client (your browser) and the server something specific:

```
  https://  www.example.com  :443   /products/shoes   ?color=red&size=10   #details
  \______/  \_____________/ \__/   \_______________/ \__________________/ \_______/
     |             |         |            |                  |                |
  Scheme       Host/Domain   Port         Path              Query           Fragment
  (How?)       (Where?)    (Which door?) (What resource?) (Extra details?) (Specific part?)

  \-------------------------/
          Authority
        (Server Info)
```

Let's break down these components:

1.  **Scheme:** `https`
    *   Tells us the *protocol* or method to use for communication. `http` (HyperText Transfer Protocol) and `https` (HTTP Secure) are the most common for web browsing. Think of it as specifying "Air Mail" vs. "Ground Mail".
2.  **Authority:** `www.example.com:443`
    *   Specifies *where* the resource is located. It includes:
        *   **Host:** `www.example.com` - The domain name or IP address of the server. This is the main part of the server's address.
        *   **Port:** `443` - A specific "door" or channel number on the server for the connection. Web servers usually listen on standard ports: 80 for `http` and 443 for `https{}. If the port is omitted, the browser uses the default for the scheme.
        *   *(Optional)* User Info: Sometimes you might see `user:password@host`, but this is less common and often insecure.
3.  **Path:** `/products/shoes`
    *   Identifies the specific resource *on that server*. It's like navigating through folders to find a particular file. `/` usually represents the main or root page.
4.  **Query:** `?color=red&size=10`
    *   Starts with a `?` and contains key-value pairs (separated by `&`). It provides extra parameters for the request, like filtering results or passing data. Think of it like adding "Subject: Urgent" or "Filter by: Red Shoes" to your request.
5.  **Fragment:** `#details`
    *   Starts with a `#`. It points to a specific section *within* the resource (like a specific heading on a long webpage). Usually, the fragment is handled only by the client (your browser) and isn't sent to the server.

Having the address broken down like this makes it easy for both the client and the server to understand exactly what is being requested and where to find it.

## The `URI` Struct in `lightbug_http`

When you work with `lightbug_http`, you don't usually deal with the raw URI string directly. Instead, `lightbug_http` provides a handy `URI` struct (defined in `lightbug_http/uri.mojo`) that holds all these parsed components for you.

Remember from Chapter 1 how we created an `HTTPRequest`? We needed a `URI` object:

```mojo
from lightbug_http import URI

# Let's parse a web address string into a URI object
var uri_string = "https://www.example.com/search?q=mojo"
var my_uri = URI.parse(uri_string)

# Now `my_uri` holds the parsed components
print("Parsed URI object:", my_uri)
```

*Explanation:*
The `URI.parse()` static method takes a string containing a web address and intelligently breaks it down into its parts, storing them in a `URI` object. If the string isn't a valid URI format, it might raise an error.

## Accessing the Parts of a `URI`

Once you have a `URI` object, you can easily access its different components:

```mojo
from lightbug_http import URI, QueryMap

# Let's use the URI from the previous example
var uri_string = "https://www.example.com:443/products?item=shoes#info"
var uri = URI.parse(uri_string)

# Accessing the components
print("Scheme:", uri.scheme)         # Output: Scheme: https
print("Host:", uri.host)             # Output: Host: www.example.com
# Port is Optional[UInt16], needs .value() if present
if uri.port:
    print("Port:", uri.port.value()) # Output: Port: 443
else:
    print("Port: (default)")
print("Path:", uri.path)             # Output: Path: /products
print("Query String:", uri.query_string) # Output: Query String: item=shoes

# Queries are parsed into a dictionary (QueryMap)
print("Query Parameter 'item':", uri.queries["item"]) # Output: Query Parameter 'item': shoes

# The full original URI is also stored
print("Full URI:", uri.full_uri)     # Output: Full URI: https://www.example.com:443/products?item=shoes#info
```

*Explanation:*
The `URI` object has fields like `.scheme`, `.host`, `.port` (which is an `Optional`), `.path`, and `.query_string`. Conveniently, `.queries` is a `QueryMap` (a dictionary) that automatically parses the key-value pairs from the query string for easy access.

## How `URI.parse()` Works (Under the Hood)

You don't *need* to know the exact details of how `URI.parse()` works internally to use it, but understanding the basic idea can be helpful.

When you call `URI.parse("https://www.example.com/path?query=val")`:

1.  **Find Scheme:** It looks for the `://` sequence. If found, the part before it (`https`) is the scheme. If not, it might default to `http`.
2.  **Find Authority:** After `://`, it looks for the next `/`, `?`, or `#` to find the end of the authority part (`www.example.com`).
3.  **Split Host/Port:** Within the authority, it looks for a `:` to separate the host (`www.example.com`) from the port (if present).
4.  **Find Path:** After the authority, the characters up to the first `?` or `#` (or the end of the string) form the path (`/path`). It also performs *unquoting* (decoding things like `%20` back into spaces).
5.  **Find Query:** If a `?` is found, the characters between `?` and the first `#` (or the end) form the query string (`query=val`).
6.  **Parse Queries:** The query string is split by `&` and then by `=` to create the `QueryMap` dictionary. Keys and values are also unquoted (e.g., `+` becomes a space).
7.  **Find Fragment:** If a `#` is found, the rest of the string is the fragment (though `lightbug_http` doesn't heavily use the fragment currently).

This parsing happens primarily within the `URI.parse` static method in `lightbug_http/uri.mojo`. It uses helper functions and a `ByteReader` to efficiently scan through the input string and extract these components according to the rules defined for URIs (specifically RFC 3986).

```mojo
# Simplified conceptual look at lightbug_http/uri.mojo
@value
struct URI:
    # ... fields for scheme, host, port, path, etc. ...
    var scheme: String
    var host: String
    var port: Optional[UInt16]
    var path: String
    var query_string: String
    var queries: QueryMap
    # ... and others like full_uri, username, password ...

    @staticmethod
    fn parse(owned uri_string: String) raises -> URI:
        # 1. Use a ByteReader to read the uri_string efficiently
        var reader = ByteReader(uri_string.as_bytes())

        # 2. Find "://" to parse the scheme
        var scheme = parse_scheme(reader) # (Conceptual function)

        # 3. Find "@" to parse user info (if any)
        var user_info = parse_user_info(reader) # (Conceptual function)

        # 4. Find "/" or "?" or "#" to parse authority (host:port)
        var host, port = parse_authority(reader) # (Conceptual function)

        # 5. Find "?" or "#" to parse the path
        var path = parse_path(reader) # (Conceptual function)

        # 6. Find "#" to parse the query string
        var query_string = parse_query_string(reader) # (Conceptual function)

        # 7. Parse the query string into key-value pairs
        var queries = parse_queries(query_string) # (Conceptual function)

        # 8. Create and return the URI object with all parsed parts
        return URI(...)
```

*Explanation:*
This simplified structure shows the fields where `URI` stores the parsed components. The `parse` method acts like a conductor, calling internal logic (represented here as conceptual functions like `parse_scheme`, `parse_authority`, etc.) to find the delimiters (`://`, `:`, `/`, `?`, `&`, `=`) and extract each part of the original URI string.

## Conclusion

You've now learned about URIs – the essential addressing system for the web.

*   A URI is like a detailed mailing address for a web resource.
*   It has distinct components: **Scheme**, **Authority** (Host, Port), **Path**, **Query**, and **Fragment**.
*   `lightbug_http` uses the `URI` struct to represent these addresses in a parsed, easy-to-use format.
*   You can create a `URI` object from a string using `URI.parse()` and access its components like `uri.scheme`, `uri.host`, `uri.path`, and `uri.queries`.

Understanding URIs is fundamental because they tell your client *where* to send the [HTTP Message (Request/Response)](01_http_message__request_response__.md) and *what specific resource* on the server it wants.

In the next chapter, we'll explore another crucial part of HTTP messages: the headers, which carry extra instructions and information between the client and server.

Next: [Chapter 3: HTTP Headers](03_http_headers_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)