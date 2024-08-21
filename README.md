---

# Advanced HTTP Server with Security and Proxy Features
A feature-rich Node.js server with HTTPS support, rate limiting, anomaly detection, IP blacklisting, reverse proxy, and an Express-based admin panel for easy management. Ideal for securing and optimizing HTTP servers.

## Overview

Welcome to your ultimate guide for building a supercharged HTTP server with Node.js! This server isn't just about handling requests—it's packed with features to keep things secure and efficient, like rate limiting, anomaly detection, blacklisting, and reverse proxying. We'll walk through both the basics and the advanced features step-by-step.

## Table of Contents

- [Introduction](#introduction)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Basic HTTP Server](#basic-http-server)
- [Advanced Features](#advanced-features)
  - [Anomaly Detection](#anomaly-detection)
  - [Rate Limiting](#rate-limiting)
  - [Blacklisting](#blacklisting)
  - [Reverse Proxy](#reverse-proxy)
  - [Express Admin Panel](#express-admin-panel)
- [Endpoints](#endpoints)
  - [Status Endpoint](#status-endpoint)
  - [Blacklist Management](#blacklist-management)
  - [Rate Limit Update](#rate-limit-update)
- [Contributing](#contributing)
- [License](#license)

## Introduction

This guide is your roadmap to creating a robust and feature-rich HTTP server using Node.js. We'll start with the basics and then dive into advanced features like anomaly detection, which helps safeguard your server from suspicious traffic.

## Features

- **Basic HTTP Server**: Simple server setup to handle HTTP requests.
- **HTTPS Server**: Secure your communications with SSL/TLS.
- **Rate Limiting**: Control the rate of requests from each IP.
- **Anomaly Detection**: Identify and respond to unusual traffic patterns.
- **IP Blacklisting**: Block requests from specified IP addresses.
- **Reverse Proxy**: Forward requests to another server.
- **Express Admin Panel**: Manage server configurations with ease.

## Prerequisites

Make sure you have these installed:

- **Node.js** (v12 or higher)
- **npm** (Node Package Manager)

## Installation

1. **Clone the repository:**

   ```bash
   git clone https://github.com/SurekingDevone/Advanced-HTTP-Server-with-Security-Proxy-Features.git
   cd Advanced-HTTP-Server-with-Security-Proxy-Features
   ```

2. **Install the dependencies:**

   ```bash
   npm install
   ```

3. **Prepare your SSL certificates (for HTTPS):**

   Place your SSL key and certificate files in the root directory of the project, naming them `server.key` and `server.crt`.

4. **Configure the server:**

   Edit the `config/main.json` file to suit your needs.

## Basic HTTP Server

Here’s how to get a basic HTTP server up and running:

### `server.js`

```js
const http = require('http');
const path = require('path');
const fs = require('fs');
const url = require('url');

const port = process.argv[2] || 8080;

const server = http.createServer((req, res) => {
  const parsedUrl = url.parse(req.url);
  const pathname = `.${parsedUrl.pathname}`;
  const ext = path.parse(pathname).ext;

  fs.exists(pathname, exist => {
    if (!exist) {
      res.statusCode = 404;
      res.end('Not Found');
      return;
    }

    fs.readFile(pathname, (err, data) => {
      if (err) {
        res.statusCode = 500;
        res.end('Server Error');
      } else {
        res.setHeader('Content-Type', getMimeType(ext));
        res.end(data);
      }
    });
  });
});

server.listen(port, () => {
  console.log(`Server running at http://localhost:${port}/`);
});

function getMimeType(ext) {
  const map = {
    '.ico': 'image/x-icon',
    '.html': 'text/html',
    '.js': 'text/javascript',
    '.json': 'application/json',
    '.css': 'text/css',
    '.png': 'image/png',
    '.jpg': 'image/jpeg',
    '.wav': 'audio/wav',
    '.mp3': 'audio/mpeg',
    '.svg': 'image/svg+xml',
    '.pdf': 'application/pdf',
    '.doc': 'application/msword'
  };
  return map[ext] || 'text/plain';
}
```

## Advanced Features

Ready to take your server to the next level? Let’s dive into some cool features that make your server smart and secure.

### Anomaly Detection

Anomaly detection helps you spot unusual patterns in traffic, which could indicate malicious activity or attacks. Here’s how it works:

1. **Track Requests**: The server logs when each request arrives from different IPs.
2. **Detect Anomalies**: If an IP sends too many requests in a short period, it's flagged as potentially problematic.

**How It’s Implemented:**

```js
const anomalyDetection = {
  requestTimestamps: new Map(),
  threshold: 100, // Number of requests allowed within the time window
  timeWindow: 60000, // 1 minute in milliseconds

  logRequest(ip) {
    const now = Date.now();
    if (!this.requestTimestamps.has(ip)) {
      this.requestTimestamps.set(ip, []);
    }
    this.requestTimestamps.get(ip).push(now);
  },

  detectAnomaly(ip) {
    const now = Date.now();
    if (this.requestTimestamps.has(ip)) {
      const timestamps = this.requestTimestamps.get(ip);
      // Remove old timestamps
      while (timestamps.length > 0 && now - timestamps[0] > this.timeWindow) {
        timestamps.shift();
      }
      if (timestamps.length > this.threshold) {
        return true; // Anomaly detected
      }
    }
    return false;
  },

  clearOldRecords() {
    const now = Date.now();
    for (const [ip, timestamps] of this.requestTimestamps.entries()) {
      while (timestamps.length > 0 && now - timestamps[0] > this.timeWindow) {
        timestamps.shift();
      }
      if (timestamps.length === 0) {
        this.requestTimestamps.delete(ip);
      }
    }
  }
};

// Clean up old records periodically
setInterval(() => {
  anomalyDetection.clearOldRecords();
}, 60000); // 1 minute interval
```

**How It Works:**

1. **Log Requests**: Every time a request comes in, we log the timestamp.
2. **Check for Anomalies**: If an IP sends more requests than allowed in the time window, we flag it.
3. **Clean Old Records**: Regularly remove old records to keep things tidy.

### Rate Limiting

Rate limiting ensures that no single user overwhelms your server with too many requests. Here’s how to set it up:

```js
const { RateLimiterMemory } = require('rate-limiter-flexible');
const rateLimiter = new RateLimiterMemory({
  points: config.ratelimit || 100, // Number of allowed requests
  duration: 15 * 60, // 15 minutes
});
```

**Rate Limiter Middleware:**

```js
const rateLimiterMiddleware = (req, res, next) => {
  const ip = req.headers['x-forwarded-for'] || req.connection.remoteAddress;

  rateLimiter.consume(ip)
    .then(() => {
      next();
    })
    .catch(() => {
      console.log(`Rate limit exceeded for IP: ${ip}`);
      res.status(429).send('Too many requests, please try again later.');
    });
};
```

**How It Works:**

1. **Configure Limits**: Set how many requests are allowed in a specified period.
2. **Check Requests**: Each request is checked against these limits.

### Blacklisting

Blacklisting is a way to block requests from specific IP addresses that are known to be problematic.

**Implementation:**

```js
const blacklist = new Set();

const isBlacklisted = (ip) => {
  return blacklist.has(ip);
};
```

**Adding to Blacklist:**

```js
app.post('/blacklist', (req, res) => {
  const ip = req.body.ip;
  if (ip) {
    blacklist.add(ip);
    res.json({ message: `IP ${ip} added to blacklist.` });
  } else {
    res.status(400).json({ message: 'IP address is required.' });
  }
});
```

**Removing from Blacklist:**

```js
app.delete('/blacklist', (req, res) => {
  const ip = req.body.ip;
  if (ip) {
    blacklist.delete(ip);
    res.json({ message: `IP ${ip} removed from blacklist.` });
  } else {
    res.status(400).json({ message: 'IP address is required.' });
  }
});
```

**How It Works:**

1. **Maintain a List**: Keep a list of IP addresses to block.
2. **Check Requests**: Block any requests coming from these IPs.

### Reverse Proxy

A reverse proxy forwards incoming requests to another server. This is useful for load balancing and integrating with other services.

**Implementation:**

```js
const httpProxy = require('http-proxy');
const proxy = httpProxy.createProxyServer({});
```

**Proxy Requests:**

```js
proxy.web(req, res, { target: config.betareverseproxytarget, secure: false });
```

**How It Works:**

1. **Forward Requests**: Route incoming requests to a different server.
2. **

Handle Responses**: Send back the response from the target server to the client.

### Express Admin Panel

An Express app to manage your server settings and view status.

**Setup Express:**

```js
const express = require('express');
const app = express();
app.use(express.static(path.join(__dirname, 'public')));
```

**Endpoints:**

- **Status Endpoint:**

  ```js
  app.get('/status', (req, res) => {
    res.json({
      throttleConnections: config.throttleConnections,
      rateLimit: config.ratelimit,
      blacklistedIPs: Array.from(blacklist),
    });
  });
  ```

- **Rate Limit Update:**

  ```js
  app.post('/rate-limit', (req, res) => {
    const newLimit = req.body.rateLimit;
    if (newLimit) {
      config.ratelimit = newLimit;
      rateLimiter = new RateLimiterMemory({
        points: newLimit,
        duration: 15 * 60,
      });
      res.json({ message: `Rate limit updated to ${newLimit}.` });
    } else {
      res.status(400).json({ message: 'Rate limit is required.' });
    }
  });
  ```

## Contributing

Want to contribute? Here’s how:

1. **Fork the repository** to your own GitHub account.
2. **Create a new branch** for your feature or bugfix.
3. **Make your changes** and commit them with descriptive messages.
4. **Push your changes** to your forked repository.
5. **Submit a pull request** to the main repository.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for more details.

---

