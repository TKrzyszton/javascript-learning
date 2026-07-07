# Day 2 — Runtime: Node.js vs Browser

## What is a runtime?

V8 alone can't do anything useful in the real world.
It can't read files, open network ports, or display web pages.

A **runtime** is V8 plus a set of additional APIs that let your code
interact with the outside world.

Analogy: V8 is a car engine. The runtime is the whole car —
wheels, steering wheel, gearbox. An engine without the rest goes nowhere.

---

## Two main runtimes

```
┌─────────────────────────────────┐    ┌─────────────────────────────────┐
│           BROWSER               │    │            NODE.JS              │
│                                 │    │                                 │
│  V8  +  Web APIs                │    │  V8  +  Node APIs               │
│         ├── DOM                 │    │         ├── fs (files)          │
│         ├── fetch               │    │         ├── http (server)       │
│         ├── localStorage        │    │         ├── path                │
│         ├── setTimeout          │    │         ├── process             │
│         └── window              │    │         └── child_process       │
└─────────────────────────────────┘    └─────────────────────────────────┘
```

Same V8 inside. Completely different APIs outside.

---

## Browser runtime

The browser gives you access to everything related to a web page:

```javascript
// Works ONLY in the browser:
document.querySelector('#btn')      // access HTML elements
window.location.href                // current URL
localStorage.setItem('key', 'val')  // persist data in browser
fetch('https://api.example.com')    // HTTP request
```

The browser deliberately **blocks** access to the filesystem and OS processes.
If a website could read your files, it would be a massive security risk.

---

## Node.js runtime

Node.js was created in 2009 by Ryan Dahl. He took V8 out of the browser
and ran it on a server, adding APIs to work with the operating system:

```javascript
// Works ONLY in Node.js:
const fs = require('fs')
fs.readFileSync('file.txt')         // read file from disk

const http = require('http')
http.createServer((req, res) => {   // HTTP server
  res.end('Hello World')
}).listen(3000)

process.env.DATABASE_URL            // environment variables
process.exit(1)                     // terminate the process
```

Node.js has no `document`, `window`, or `localStorage` —
because there is no browser. It processes data and handles requests,
it doesn't render pages.

---

## Global object

Every JS environment has a global object — but it's not the same one:

```javascript
// In the browser — global object is window:
window.setTimeout(fn, 1000)
window.alert('hello')
window === globalThis // true

// In Node.js — global object is global:
global.setTimeout(fn, 1000)
global.process.env
global === globalThis // true

// globalThis works everywhere — the safe universal option:
globalThis.setTimeout(fn, 1000) // works in both Node.js and browser
```

---

## Same code, different internals

This is a common trap for beginners:

```javascript
// This looks the same everywhere...
setTimeout(() => console.log('hello'), 1000)

// ...but works differently under the hood:
// Browser: setTimeout comes from Web API (C++ inside the browser engine)
// Node.js: setTimeout comes from libuv (C++ async I/O library in Node)

// Same result, different mechanism.
```

A more practical example:

```javascript
// fetch — HTTP request
fetch('https://api.example.com/data')
  .then(res => res.json())
  .then(data => console.log(data))

// Browser:  fetch has been built-in forever
// Node.js:  fetch is available from Node 18+
//           before that you needed 'node-fetch' or 'axios'
```

---

## Why this matters for you

You're building a backend with NestJS — your environment is **Node.js only**.
Understanding the difference between runtimes means you know:

- Why frontend tutorial code doesn't work in Node (`document is not defined`)
- Why libraries have separate "browser" and "Node" versions
- Where every API you use comes from — V8, Node.js, or an external library

---

## Key Terms

| Term | Definition |
|---|---|
| **Runtime** | V8 + platform-specific APIs (browser or Node.js) |
| **Web APIs** | Browser-provided APIs: DOM, fetch, localStorage, window |
| **Node APIs** | Node.js-provided APIs: fs, http, process, child_process |
| **libuv** | C++ library inside Node.js that handles async I/O |
| **globalThis** | Universal reference to the global object (works in both runtimes) |

---

## Common Errors and Their Meaning

| Error | Cause |
|---|---|
| `document is not defined` | Using a browser API in Node.js |
| `window is not defined` | Same — window doesn't exist in Node.js |
| `require is not defined` | Using CommonJS in a browser without a bundler |
| `fetch is not defined` | Using fetch in Node.js below version 18 |

---

## Questions to Think About

- What happens when you run `console.log(this)` in the browser vs Node.js?
- Why does Node.js need `libuv` if V8 already handles JavaScript execution?
- If `setTimeout` exists in both runtimes, why does it sometimes behave differently?
