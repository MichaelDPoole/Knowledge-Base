# Chapter 4: Network Address Representation (`sockaddr`, `in_addr`, etc.)

Welcome back! In the [previous chapter](03_address_resolution___getaddrinfo___.md), we learned how to use `getaddrinfo` to look up the numerical IP address and port number for a given hostname and service, like getting the phone number from directory assistance. `getaddrinfo` gives us back a list of results, and each result contains a pointer to the actual address information.

But what does that "address information" look like in memory? How is an IP address like `127.0.0.1` and a port like `80` actually stored so that low-level C network functions can understand it? This chapter dives into the standard structures used to represent network addresses: `sockaddr`, `sockaddr_in`, `in_addr`, and their IPv6 counterparts.

## Why Do We Need Special Address Structures?

Imagine you're sending a physical letter. You can't just scribble "John Doe, Big Street, Some Town" anywhere on the envelope. The post office needs the address in a specific format and place (recipient name, street address, city, postal code) to deliver it correctly.

Similarly, when our Mojo program interacts with the operating system's C network functions (like `connect` to establish a connection, or `bind` to associate a server socket with an address), these functions expect the destination or local address information to be provided in a very specific, standardized binary format. They don't understand convenient Mojo `String`s like `"192.168.1.100"` or simple integers for ports directly in their main arguments.

The `sockaddr` family of structs provided by `mojo-libc` define these standard "address label" formats. They are Mojo structs designed to precisely match the memory layout expected by the underlying C functions.

## The Address Structures: Envelopes and Labels

Think of these structures like envelopes and address labels for different types of destinations:

**1. The Generic Envelope: `sockaddr`**

The most fundamental structure is `sockaddr`. It's designed to be a generic placeholder that's large enough to hold *any* type of socket address (IPv4, IPv6, Unix domain sockets, etc.).

```mojo
# Simplified from libc/_libc.mojo
from libc import sa_family_t, c_char
from utils import StaticTuple

@value
@register_passable("trivial")
struct sockaddr:
    var sa_family: sa_family_t   # Address family (e.g., AF_INET for IPv4)
    var sa_data: StaticTuple[c_char, 14] # The actual address data
```

*   `sa_family`: This is the most important field! It's a number (like `AF_INET` for IPv4 or `AF_INET6` for IPv6) that tells you what *kind* of address is actually stored in the rest of the structure (in `sa_data`). It identifies the specific format being used.
*   `sa_data`: This holds the actual destination address (like the combination of IP address and port number), but its format depends entirely on the value of `sa_family`.

Because `sockaddr` is generic, you almost never use its `sa_data` field directly. Instead, you look at `sa_family` to figure out the *real* address type, and then you **cast** the `sockaddr` pointer to a pointer of the more specific structure type (like `sockaddr_in` for IPv4).

**2. The IPv4 Address Label: `sockaddr_in`**

When `sa_family` is `AF_INET`, it means the address is an IPv4 address. The *actual* structure holding the data is `sockaddr_in` (the 'in' stands for 'internet').

```mojo
# Simplified from libc/_libc.mojo
from libc import sa_family_t, in_port_t, in_addr, c_char
from utils import StaticTuple

@value
@register_passable("trivial")
struct sockaddr_in:
    var sin_family: sa_family_t   # Address family (always AF_INET here)
    var sin_port: in_port_t     # Port number (in network byte order!)
    var sin_addr: in_addr       # IPv4 address (in network byte order!)
    var sin_zero: StaticTuple[c_char, 8] # Padding, unused. Must be zero.
```

*   `sin_family`: For `sockaddr_in`, this will always be `AF_INET`.
*   `sin_port`: This holds the 16-bit port number (like 80 for HTTP). **Crucially**, this value must be in *network byte order* (more on this in [Chapter 5](05_network_byte_order_conversion___htonl____htons____ntohl____ntohs___.md)).
*   `sin_addr`: This holds the 32-bit IPv4 address. It's actually another structure called `in_addr`. This must also be in *network byte order*.
*   `sin_zero`: Just some padding bytes to make the `sockaddr_in` structure the same size as the generic `sockaddr`. You should always ensure these are zero.

**3. The IPv4 Address Itself: `in_addr`**

The `sin_addr` field within `sockaddr_in` is of type `in_addr`. This simple structure just holds the 32-bit IPv4 address.

```mojo
# Simplified from libc/_libc.mojo
from libc import in_addr_t

@value
@register_passable("trivial")
struct in_addr:
    var s_addr: in_addr_t # The 32-bit IPv4 address (network byte order!)
```

*   `s_addr`: This holds the actual numerical IPv4 address (like `0x7F000001` for `127.0.0.1`) in *network byte order*.

**4. The IPv6 Equivalents: `sockaddr_in6` and `in6_addr`**

For newer IPv6 addresses (`sa_family` is `AF_INET6`), there are corresponding structures:

*   `sockaddr_in6`: Holds IPv6 family, port, flow info, the 128-bit IPv6 address (`in6_addr`), and scope ID.
*   `in6_addr`: Holds the 128-bit IPv6 address.

We won't detail them here, but they follow the same principle: specific structures designed to hold all necessary information for an IPv6 address in the format C functions expect.

## Using the Structures: Accessing `getaddrinfo` Results

Let's revisit the `getaddrinfo` function from [Chapter 3](03_address_resolution___getaddrinfo___.md). It returns a pointer (`results_ptr`) that points to an `addrinfo` structure. Inside *that* structure is a field `ai_addr` which is a `UnsafePointer[sockaddr]` - a pointer to our generic envelope!

How do we peek inside the envelope?

```mojo
from libc import (
    addrinfo, AF_INET, AF_INET6, sockaddr, sockaddr_in, sockaddr_in6,
    # Assume getaddrinfo was called successfully and results_ptr points to the first result
    # var results_ptr: UnsafePointer[addrinfo]
)
from memory import UnsafePointer

fn process_address(results_ptr: UnsafePointer[addrinfo]) raises:
    # Get the first result structure
    let result: addrinfo = results_ptr.load() # Read the addrinfo struct

    # Get the pointer to the generic address structure
    let generic_addr_ptr: UnsafePointer[sockaddr] = result.ai_addr

    # Check the family to know what kind of address it is
    let family = generic_addr_ptr.load().sa_family # Read the sa_family field

    if family == AF_INET:
        print("Found an IPv4 address!")
        # Cast the generic pointer to the specific IPv4 type
        let ipv4_addr_ptr = generic_addr_ptr.bitcast[sockaddr_in]()

        # Now we can safely access the IPv4 specific fields
        let ipv4_addr_struct: sockaddr_in = ipv4_addr_ptr.load()
        let port = ipv4_addr_struct.sin_port
        let ip_struct: in_addr = ipv4_addr_struct.sin_addr
        let ip_address_num = ip_struct.s_addr

        # NOTE: port and ip_address_num are still in Network Byte Order!
        # We need Chapter 5 functions (ntohs, ntohl) to make them readable.
        print(" Port (raw):", port)
        print(" IP Addr (raw):", ip_address_num)

    elif family == AF_INET6:
        print("Found an IPv6 address!")
        # Cast the generic pointer to the specific IPv6 type
        let ipv6_addr_ptr = generic_addr_ptr.bitcast[sockaddr_in6]()
        # ... access IPv6 fields similarly (sin6_port, sin6_addr) ...
        print(" (IPv6 details omitted for brevity)")

    else:
        print("Found an address of unknown family:", family)

# Example usage (assuming getaddrinfo call happened before):
# try:
#     process_address(results_ptr)
# except e:
#     print("Error processing address:", e)
# finally:
#     # Remember to free the results list using freeaddrinfo() in real code!
#     # freeaddrinfo(results_ptr) # (Currently not exposed in mojo-libc example)
#     pass
```

**Explanation:**

1.  We get the `addrinfo` structure pointed to by `results_ptr`.
2.  We access its `ai_addr` field, which gives us an `UnsafePointer[sockaddr]`.
3.  We load the `sockaddr` struct just to read its `sa_family` field. This tells us the *actual* type of address hidden inside.
4.  **If `sa_family` is `AF_INET`:**
    *   We know it's safe to treat the `ai_addr` pointer as if it points to a `sockaddr_in` structure.
    *   We use `bitcast[sockaddr_in]()` to change the pointer type *without* changing the memory address it points to.
    *   Now we can load the `sockaddr_in` struct from this correctly-typed pointer.
    *   We access the `sin_port` and `sin_addr` fields. `sin_addr` itself contains the `s_addr` field with the IP address number.
5.  **If `sa_family` is `AF_INET6`:** We would do a similar cast to `sockaddr_in6`.
6.  We print the raw numerical values. Notice we mention they are still in "Network Byte Order" - a concept we'll tackle next.

This process of checking `sa_family` and then casting the `sockaddr` pointer is fundamental when working with socket addresses returned by functions like `getaddrinfo`, `accept`, or `getpeername`.

## How It Works Under the Hood: Memory Layout

These Mojo structs (`sockaddr`, `sockaddr_in`, `in_addr`) are defined in `mojo-libc` using `@value` and `@register_passable("trivial")`. This ensures they behave like simple data containers and match the exact memory layout of their corresponding C structure definitions.

Let's look at the definitions in `mojo-libc`:

```mojo
# --- File: src/libc/_libc.mojo ---

alias sa_family_t = c_ushort # Usually an unsigned short (16 bits)
alias in_port_t = c_ushort   # Port number (16 bits)
alias in_addr_t = c_uint     # IPv4 Address (32 bits)

# The generic socket address structure
@value
@register_passable("trivial")
struct sockaddr:
    var sa_family: sa_family_t
    var sa_data: StaticTuple[c_char, 14] # 14 bytes of data

# The IPv4 socket address structure
@value
@register_passable("trivial")
struct sockaddr_in:
    var sin_family: sa_family_t   # Typically 2 bytes
    var sin_port: in_port_t     # Typically 2 bytes
    var sin_addr: in_addr       # Contains the 4-byte IP below
    var sin_zero: StaticTuple[c_char, 8] # 8 bytes of padding

# The IPv4 address structure itself
@value
@register_passable("trivial")
struct in_addr:
    var s_addr: in_addr_t       # Typically 4 bytes (32 bits)

# The IPv6 structures (simplified view)
# struct in6_addr: contains 16 bytes for the IPv6 address
# struct sockaddr_in6: contains family, port, flowinfo, in6_addr, scope_id
```

These definitions tell the Mojo compiler exactly how to arrange the fields in memory. When you create a `sockaddr_in` variable, it reserves space for the family, then the port, then the `in_addr` structure (which contains the IP), and finally the padding.

Here's a conceptual diagram of how a `sockaddr_in` might look in memory (assuming common sizes):

```mermaid
graph TD
    subgraph sockaddr_in (Total 16 bytes)
        direction LR
        A[sin_family (2 bytes, e.g., AF_INET)] --> B(sin_port (2 bytes));
        B --> C(sin_addr (4 bytes));
        C --> D(sin_zero (8 bytes));
    end
    subgraph in_addr (Inside sin_addr)
      E[s_addr (4 bytes, IP Address)];
    end
    C --> E;

    subgraph sockaddr (Generic, Size matches sockaddr_in)
      direction LR
      F[sa_family (2 bytes)] --> G[sa_data (14 bytes)];
    end

    style A fill:#f9f,stroke:#333,stroke-width:2px
    style B fill:#ccf,stroke:#333,stroke-width:2px
    style C fill:#9cf,stroke:#333,stroke-width:2px
    style D fill:#eee,stroke:#333,stroke-width:2px
    style E fill:#9cf,stroke:#333,stroke-width:2px
    style F fill:#f9f,stroke:#333,stroke-width:2px
    style G fill:#fcc,stroke:#333,stroke-width:2px

```

Because `sockaddr_in` (and `sockaddr_in6`) are designed to fit perfectly within the space allocated for a generic `sockaddr`, casting pointers between them (like `generic_addr_ptr.bitcast[sockaddr_in]()`) works correctly – you're just telling the compiler to reinterpret the *same* block of memory according to a more specific layout.

When you pass a pointer to one of these structs (correctly cast) to a C function like `connect` or `bind`, the C function knows exactly where to find the `family`, `port`, and `address` bytes within that block of memory because the layout is standardized.

## Conclusion

In this chapter, we peeled back the cover on how network addresses are represented in memory for C interoperability. We learned about:

*   The generic `sockaddr` structure, acting as a universal envelope.
*   The specific `sockaddr_in` (IPv4) and `sockaddr_in6` (IPv6) structures, like detailed address labels.
*   The `in_addr` and `in6_addr` structures holding the actual IP addresses.
*   The importance of the `sa_family` field for identifying the address type.
*   How to cast `sockaddr` pointers to specific types (`sockaddr_in`, `sockaddr_in6`) to access address details.

These structures are the essential "address labels" required by C networking functions. However, we saw that the numerical values inside (ports and IP addresses) must be in a special format called "network byte order". What is that, and how do we convert to and from it? Let's find out in the next chapter!

**Next:** [Chapter 5: Network Byte Order Conversion (`htonl`, `htons`, `ntohl`, `ntohs`)](05_network_byte_order_conversion___htonl____htons____ntohl____ntohs___.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)