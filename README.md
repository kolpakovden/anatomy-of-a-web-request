# What Happens When You Type `google.com` into Your Browser and Press Enter?

**English | [Русский](README_RU.md)**

It looks like the simplest action in the world: type an address and press Enter. Yet that single keystroke crosses several layers of hardware, operating-system code, networking infrastructure, server software, and graphics processing before the page appears on your screen.

Let's trace the journey, starting not with HTTP but with the keyboard itself. The exact path depends on the device, operating system, browser, network, and protocols negotiated along the way.

## Contents

- [Pressing the `g` key](#pressing-the-g-key)
- [Pressing Enter](#pressing-enter)
- [An interrupt reaches the CPU](#an-interrupt-reaches-the-cpu)
- [Windows delivers `WM_KEYDOWN`](#windows-delivers-wm_keydown)
- [macOS delivers an `NSEvent`](#macos-delivers-an-nsevent)
- [GNU/Linux processes keyboard input](#gnulinux-processes-keyboard-input)
- [The browser parses the address](#the-browser-parses-the-address)
- [Address or search query?](#address-or-search-query)
- [Unicode and domain names](#unicode-and-domain-names)
- [Checking HSTS](#checking-hsts)
- [Resolving an IP address with DNS](#resolving-an-ip-address-with-dns)
- [Finding the next hop with ARP](#finding-the-next-hop-with-arp)
- [Creating a socket and TCP connection](#creating-a-socket-and-tcp-connection)
- [Securing the connection with TLS](#securing-the-connection-with-tls)
- [Dealing with lost packets](#dealing-with-lost-packets)
- [Speaking HTTP](#speaking-http)
- [Handling the request on a web server](#handling-the-request-on-a-web-server)
- [What happens inside the browser](#what-happens-inside-the-browser)
- [Browser architecture](#browser-architecture)
- [Parsing HTML](#parsing-html)
- [Processing CSS](#processing-css)
- [Rendering the page](#rendering-the-page)
- [The graphics processor](#the-graphics-processor)
- [The window compositor and display](#the-window-compositor-and-display)
- [After the page loads](#after-the-page-loads)

## Pressing the `g` key

You focus the browser's address bar and press `g`. Even before you finish typing, a suggestion dropdown may appear. Depending on settings, the browser might consult your history, bookmarks, open tabs, saved addresses, or a search provider. A private-browsing session may apply different rules.

With every subsequent character of `google.com`, the browser can recompute suggestions and redraw parts of its interface. It may guess your intended destination before the final character arrives.

Each keypress is already a small pipeline: hardware detects a change, a controller sends input data, the operating system dispatches an event, an application updates its state, and the display is refreshed. Some steps can also involve disk or network access.

## Pressing Enter

You press `Enter`. On a typical mechanical keyboard, a switch closes an electrical circuit. Capacitive and optical switches detect the change differently. The keyboard's microcontroller scans its key matrix to identify the changed key.

Physical switch contacts can briefly bounce between open and closed. Firmware typically uses *debouncing* to avoid treating one press as several.

The keyboard then encodes the event. A physical scan code, a USB HID usage, a platform-specific virtual key code, and a printable character are distinct concepts. `Enter` usually signals an action rather than inserting a character into the address bar.

### USB keyboard

1. The USB interface supplies power to the device, normally through a 5 V VBUS connection.
2. The keyboard controller produces a HID report describing which keys are pressed.
3. The host controller schedules transfers to the device's interrupt endpoint. The interval depends on its descriptors, USB speed, and host behavior; it is not universally 10 milliseconds.
4. On USB 2.0, data travels over the `D+` and `D−` lines. Devices may use low, full, or high speed, depending on their hardware.
5. The host controller and HID driver expose the new state to the operating system's input subsystem.

### Bluetooth keyboard

The keyboard sends input reports over a Bluetooth HID transport. The computer's Bluetooth stack receives them, applies any required decryption, and forwards the resulting events to the normal input subsystem.

### On-screen keyboard

For a touchscreen, the touch controller detects changes in the capacitive sensor grid and reports coordinates. The operating system delivers a touch event, and the software keyboard maps the touched region to its `Enter` action.

## An interrupt reaches the CPU

Hardware controllers often notify the processor that work is available by generating an *interrupt request*, or IRQ. An interrupt controller routes the request to an appropriate handler.

On x86, the operating system sets up an Interrupt Descriptor Table (`IDT`). When the CPU accepts an interrupt, it preserves the required execution state and transfers control to the kernel's handler. A driver processes the controller's data and makes the input event available to higher layers.

This is deliberately simplified: a USB keypress does **not** require one traditional keyboard IRQ per key. USB controller interrupts, scheduled transfers, and driver processing work differently from the legacy PS/2 path.

The important result is the same: the operating system learns that Enter was pressed.

## Windows delivers `WM_KEYDOWN`

Windows handles keyboard devices through its input-driver stack. For example, `kbdhid.sys` can process USB HID keyboard input, while `kbdclass.sys` provides a common keyboard device class.

After the system processes the relevant key data, a traditional Win32 application may receive a `WM_KEYDOWN` message in its thread's message queue. The virtual key code for Enter is `VK_RETURN` (`0x0D`).

A simple window procedure might look like this:

```c
case WM_KEYDOWN:
    if (wParam == VK_RETURN) {
        navigate_to_address();
    }
    break;
```

The `lParam` field can describe repeat counts, scan codes, and prior key state. Real browsers add their own event dispatching and UI logic on top of the native platform messages; the example only illustrates the basic idea.

Applications often retrieve and dispatch messages through functions such as `GetMessage`, `PeekMessage`, and `DispatchMessage`.

## macOS delivers an `NSEvent`

On macOS, the HID and windowing subsystems process keyboard input and deliver it to the active application. At the Cocoa level, keyboard events can appear as `NSEvent` instances of type `keyDown`.

The system determines which window and control have focus. The application's event loop routes the event to the relevant responder. If the address bar has focus, Enter triggers navigation.

## GNU/Linux processes keyboard input

Linux device drivers feed events into the kernel input subsystem, commonly exposed to user space through `evdev` devices such as `/dev/input/event*`.

The path then depends on the graphical environment:

- With **X11**, an X server processes input, keyboard mappings, and event delivery to clients.
- With **Wayland**, a compositor handles input routing and focus. Xorg may not be involved at all.

Once the browser receives the focused key event, it asks its navigation code to interpret the address bar's content.

## The browser parses the address

The browser doesn't send raw keystrokes directly to a website. It first interprets the address bar as a potential URL, resolves shortcuts or search behavior, and constructs a navigation request.

A URL has several components:

```text
scheme://userinfo@host:port/path?query#fragment
```

For `https://google.com/`, the scheme is `https`, the host is `google.com`, the default HTTPS port is 443, and the path is `/`.

The fragment, when present, is generally processed by the browser and not transmitted as part of the HTTP request target. A real parser also handles escaping, normalization, special characters, and many edge cases specified by URL parsing rules.

## Address or search query?

Modern address bars double as search fields. A space-containing phrase often becomes a search query, while a recognizable hostname or URL generally triggers navigation.

The browser uses parsing rules and heuristics to make the distinction. Depending on settings, typing a bare word may search the web, try an intranet hostname, or produce a suggestion. `google.com` is normally recognized as a domain.

A selected suggestion may change the destination. The browser can also reuse known navigation data, existing connections, cached redirects, or stored security policies.

## Unicode and domain names

Domain names can contain internationalized characters. DNS itself works with an ASCII-compatible form of these labels, so applications use the Internationalized Domain Names in Applications (IDNA) processing rules.

For example, a Unicode domain label may be converted to an `xn--` Punycode representation before DNS lookup. This is distinct from percent-encoding characters in a URL path.

Browsers also apply spoofing and display policies: some internationalized hostnames are shown in Unicode, while others are deliberately displayed in their ASCII-compatible form.

For `google.com`, no internationalized-label conversion is necessary.

## Checking HSTS

Before opening a connection, the browser checks whether the hostname is covered by an HTTP Strict Transport Security (HSTS) policy.

HSTS instructs a browser to use HTTPS for a site for a specified period. Policies can have been learned from an earlier response or included in a preload list. If a policy applies, a request that would otherwise use HTTP is upgraded to HTTPS before contacting the server.

For an explicitly entered `https://` URL, the connection is already secure by construction. HSTS still affects how the browser handles possible insecure attempts and certificate failures.

## Resolving an IP address with DNS

A hostname is useful to humans; network packets need destination addresses. The browser therefore needs an IP address for `google.com`, unless a usable address is already known.

### Local sources first

The browser or operating system may consult:

1. A browser DNS or host cache.
2. The operating system's resolver cache.
3. Local host overrides, such as a hosts file.
4. An enterprise or network-specific resolver configuration.

The precise ordering is platform-dependent. A browser using DNS over HTTPS may follow a different route from the operating system's conventional resolver.

Cached DNS answers have a time-to-live (TTL), which constrains how long they should be reused. The browser may request `A` records for IPv4 and `AAAA` records for IPv6.

### If the resolver has no cached answer

A recursive resolver can walk the DNS hierarchy:

```text
Client / stub resolver
          |
          v
Recursive DNS resolver
          |
          +--> Root name servers
          |           |
          |           v
          +--> .com authoritative infrastructure
          |           |
          |           v
          +--> Authoritative DNS for google.com
          |
          v
       DNS answer
```

The iterative details depend on cached delegations and records. A resolver typically doesn't query every level from scratch for every lookup.

Traditional DNS queries often use UDP port 53, with TCP available when needed. Other deployments may use DNS over TLS (DoT) or DNS over HTTPS (DoH). DNS messages can include extensions such as EDNS; DNSSEC, when deployed and validated, provides authenticity of DNS data rather than transport encryption.

Large services can return multiple addresses, use load balancing, and choose answers based partly on network topology. The returned address need not identify one unique physical machine.

## Finding the next hop with ARP

Once the destination IP is known, the operating system chooses an outgoing route. If the destination is outside the local network, the next hop is usually a default gateway.

On an IPv4 Ethernet network, the sender needs the next hop's MAC address. It first checks its neighbor or ARP cache. If there is no usable entry, it broadcasts an ARP request asking who owns the next-hop IPv4 address.

A device with the requested address responds with its MAC address. The sender can then build an Ethernet frame addressed to that next hop.

For IPv6, *Neighbor Discovery* (NDP) fills a similar role rather than ARP. Not all networks use Ethernet, so a MAC-address lookup is not a universal prerequisite for every Internet connection.

### The local equipment

The Ethernet frame may pass through a switch and router. A switch forwards frames within a layer-2 domain using its MAC address table. A router examines network-layer destination addresses and forwards packets between networks.

A home router may perform source NAT for IPv4, rewriting addresses and ports while tracking connection state. Firewalls can inspect or filter packets. The next router repeats forwarding until the traffic reaches the destination network.

### Sending the DNS query

If DNS resolution required network communication, a broadly similar local-delivery path already occurred for those DNS packets. DNS may use a different destination and transport from the eventual HTTPS connection.

In other words, obtaining the web server's address can itself involve several frames, packets, routing decisions, and protocol exchanges.

## Creating a socket and TCP connection

With an address selected, the browser asks the operating system to establish a connection, often through a socket API.

For HTTPS over TCP, a new connection commonly begins with the three-way handshake:

```text
Client                                 Server
  | -------- SYN --------------------> |
  | <------- SYN + ACK --------------- |
  | -------- ACK --------------------> |
  |                                    |
  |       TCP connection ready         |
```

TCP uses sequence numbers, acknowledgments, flow control, and congestion-control algorithms. The source port is normally an ephemeral local port; the destination port for HTTPS is typically 443.

The operating system may use a previously established connection rather than starting a new one. Browsers pool and reuse connections where protocol and security constraints permit.

There is also an important alternative: **HTTP/3 uses QUIC over UDP**, not TCP. QUIC includes its own reliable-stream delivery, congestion control, and cryptographic handshake behavior. The TCP-and-TLS story below describes the common HTTPS-over-TCP case, not every modern navigation.

## Securing the connection with TLS

For HTTPS over TCP, the browser and server negotiate TLS. Modern deployments commonly use TLS 1.3, although actual support depends on both peers.

A simplified TLS 1.3 exchange looks like:

```text
Client                                 Server
  | -------- ClientHello ------------> |
  | <------- ServerHello ------------- |
  | <------- EncryptedExtensions ----- |
  | <------- Certificate / Verify ---- |
  | <------- Finished ---------------- |
  | -------- Finished ----------------> |
  |          Encrypted traffic         |
```

The client advertises supported capabilities and key-share material. The server selects parameters and proves its identity with a certificate chain and a signature. The browser validates the certificate's hostname, trust chain, validity, and applicable security policies.

The handshake produces shared traffic keys. Actual page data is then protected using authenticated encryption, which provides confidentiality and integrity.

Some resumed sessions can shorten setup. TLS 1.3 also supports 0-RTT early data in specific resumption scenarios, subject to replay-safety restrictions. QUIC integrates TLS 1.3 into its transport handshake rather than running TLS as a separate record layer on top of TCP.

## Dealing with lost packets

The Internet does not guarantee that every packet will arrive. Congestion, radio interference, damaged links, or queue drops can cause loss.

TCP tracks acknowledged data and retransmits missing segments as needed. Duplicate acknowledgments and retransmission timers help detect loss. Modern TCP implementations may use selective acknowledgments and sophisticated loss-recovery algorithms.

TCP also adapts how much data it sends to the receiver's window and its estimate of network congestion. A packet loss event may reduce throughput, but not every loss requires restarting the connection.

QUIC implements its own acknowledgment, loss detection, and recovery mechanisms over UDP. Its independently delivered streams can avoid some transport-level head-of-line blocking across streams that occurs with HTTP/2 over TCP.

## Speaking HTTP

With transport and security established, the browser sends an HTTP request. In a human-readable HTTP/1.1 representation, it might resemble:

```http
GET / HTTP/1.1
Host: google.com
User-Agent: ExampleBrowser/1.0
Accept: text/html
Accept-Language: en
```

The real headers depend on the browser, its settings, cookies, caching state, and request context. HTTP/2 and HTTP/3 encode requests differently, with binary framing and header compression, so they do not literally put that HTTP/1.1 text onto the wire.

The browser can negotiate an HTTP version via TLS ALPN. It may also learn that HTTP/3 is available through advertised alternatives.

A response contains a status, metadata, and possibly a body:

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Cache-Control: private

<!doctype html>
<html>...</html>
```

Depending on the request, the server may instead send a redirect, a cached response indicator, a compressed representation, or an error. Content encoding and transfer framing are separate details, and HTTPS encrypts the transmitted HTTP data.

## Handling the request on a web server

The server-side path is not necessarily one process on one machine.

A DNS answer may lead to an edge endpoint or load balancer. Incoming traffic can pass through a reverse proxy, TLS termination layer, request router, cache, and application services before any HTML is generated.

A possible flow is:

```text
Internet
   |
   v
Edge / load balancer
   |
   v
TLS and HTTP endpoint
   |
   +--> Cache
   |
   +--> Application / backend services
                |
                +--> Databases and storage
   |
   v
HTTP response
```

The application may check cookies, session state, user preferences, localization settings, experiment assignments, and request parameters. It might assemble HTML dynamically or serve a cached representation.

A service as large as Google operates distributed infrastructure, but its internal production implementation cannot be inferred simply from a browser request. This diagram illustrates common web-server architecture, not a claim about a specific proprietary request path.

## What happens inside the browser

As data arrives, the browser can process it incrementally instead of waiting for every byte to download.

The response headers determine the content type, caching behavior, security rules, and sometimes character encoding. The browser begins parsing HTML and can discover additional resources: style sheets, scripts, fonts, images, and more.

Each resource has its own rules for priority, cache lookup, same-origin checks, CORS where applicable, and network retrieval. Some resources may share an established connection; others may come from local cache or separate origins.

At the same time, a previous page's state may be discarded or preserved depending on navigation behavior and back-forward caching.

## Browser architecture

Modern browsers commonly separate responsibilities across multiple processes, although the exact layout differs by browser and platform.

Typical roles include:

- A **browser process** that coordinates tabs, navigation, permissions, and parts of the UI.
- **Renderer processes** responsible for web content, DOM, style, layout, and JavaScript execution.
- A **network service** that handles requests, caching, connections, and protocol machinery.
- A **GPU process** or graphics subsystem that manages rendering resources and compositing tasks.

Process isolation and sandboxing help limit the damage that untrusted web content can cause. Site isolation policies influence which documents may share a renderer process.

None of these boundaries implies that every individual operation always happens in a separate process or thread.

## Parsing HTML

The browser decodes the response body and tokenizes the HTML stream. Tokens drive a tree-building algorithm that constructs the **Document Object Model**, or DOM.

Consider:

```html
<!doctype html>
<html lang="en">
  <head><title>Example</title></head>
  <body><h1>Hello, world!</h1></body>
</html>
```

Conceptually, the resulting tree contains a document node with `html`, `head`, and `body` descendants. The browser also handles malformed markup according to HTML parsing rules; it does not simply reject a page because tags are missing.

Scripts can change the DOM while parsing is underway. Certain classic scripts may block the HTML parser; `async`, `defer`, and module loading change execution timing and dependencies.

The DOM represents document structure, not the final pixels displayed on the screen.

## Processing CSS

CSS describes how elements should look and participate in layout. The browser parses style sheets, matches selectors to elements, and calculates styles using the cascade, inheritance, specificity, and applicable user-agent rules.

For example:

```css
h1 {
  color: #1764aa;
  font-size: 2rem;
  margin: 1rem 0;
}
```

Not every DOM node becomes a visible box. A node with `display: none` contributes no layout box, while pseudo-elements may generate boxes without corresponding DOM elements.

The browser builds the internal structures needed for layout and painting. You may see these described informally as a render tree, although actual engine data structures are more complex.

Fonts, viewport dimensions, device pixel ratio, media queries, and writing direction all influence the eventual appearance.

## Rendering the page

Rendering is often described as three broad phases: **layout**, **paint**, and **compositing**. Engines optimize these phases heavily, and the details vary.

### Layout: computing geometry

The engine decides how large boxes are and where they belong. It resolves available widths, line breaking, positioning, flex and grid constraints, margins, overflow, and other layout rules.

A change in an element's dimensions can force part of the layout to be recalculated. Engines try to limit such work to affected portions of the document.

### Paint: describing what to draw

The engine produces drawing commands for visible content: backgrounds, borders, text, images, shadows, and other visual effects.

Painting does not mean the CPU individually changes every pixel on the monitor. Commands and intermediate representations are prepared for rasterization and composition.

### Compositing: combining layers

Some elements or paint chunks may be promoted into independently composited layers. This can make operations such as scrolling, transforms, and opacity changes cheaper than repainting the full page.

Layers are rasterized as necessary and combined into a final scene. Which elements receive separate layers is an implementation detail, not a fixed CSS rule.

## The graphics processor

The GPU helps rasterize or composite graphical content, process textures, transform geometry, and render the final image efficiently. Exactly how much work runs on the GPU depends on the browser, drivers, operating system, and feature configuration.

A rough pipeline is:

```text
DOM + CSS
    |
    v
Style and layout
    |
    v
Paint commands
    |
    v
Rasterized surfaces / textures
    |
    v
GPU composition
    |
    v
Window surface / framebuffer
```

Graphics APIs such as Direct3D, Metal, Vulkan, or OpenGL may participate, directly or through browser abstraction layers. The GPU's command processor runs work submitted by the system, and display hardware later scans out completed image buffers.

The exact implementation may use multiple buffers and synchronized presentation. A GPU does not independently decide what the web page means: the browser's layout and rendering machinery establishes that structure first.

## The window compositor and display

The browser's completed surface is generally not the only thing on screen. The desktop also has window decorations, the cursor, other applications, and system overlays.

On Windows, the Desktop Window Manager participates in compositing. On macOS, the WindowServer and associated graphics infrastructure manage on-screen presentation. With Wayland, the compositor coordinates window surfaces and output; with X11, a separate compositor may be used.

The resulting frame reaches the display pipeline. Depending on the display technology, the screen refreshes pixels by driving LCD subpixels and their backlight, or individual OLED emitters.

Refresh rate, display synchronization, buffering, and input-to-photon latency affect when you actually see the change. The visual result arrives only after all the necessary upstream work has happened.

## After the page loads

A displayed page is not necessarily a finished page. It may still:

- download images, fonts, scripts, or deferred resources;
- execute JavaScript and react to timers or user input;
- initiate additional API requests;
- update DOM and CSS state;
- recompute layout and repaint changed content;
- composite new frames;
- update cookies, caches, and other storage according to browser security rules.

Every new interaction—moving the pointer, scrolling, typing, or opening another link—can start fresh chains of input events, application logic, network traffic, and graphics work.

And all of this began with a single press of `Enter`.
