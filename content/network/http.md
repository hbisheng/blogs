---
title: "Network Protocol: HTTP, Websocket"
date: "2023-01-04"
weight: 100
---

**TL;DR** HTTP (Hyper-Text Transfer Protocol) is an application layer protocol that enables you to browse websites.

---

We've covered most of the fundamentals such as routing, transport, and security, now it's finally time to look at some applications! One of the most popular activities on the internet is visiting websites. You type in a domain name and your browser will display you web pages. The data format of the underlying web content is HTML (Hyper-text Mark-up Language). HTTP is the protocol that transmits HTML from the web server to your browser.

### HTTP

HTTP follows the **client/server model**. The server serves some kinds of resources and clients send requests to fetch those resources.

HTTP follows the **stateless principle**. Every request is being evaluated on its own without references to requests in the past. You may be surprised how many things can be done by this simple pattern. You may first request a list of the articles and then request to visit one of the articles. The two requests are totally independent; they are not context-aware. You are basically saying “give me this”, “give me that”.

What about more sophisticated activities like login and adding to my shopping cart? You may ask. HTTP itself is stateless but your browser can store the context of your activity in **cookies.** This context is sent along with the request to the web server. You are basically saying things like “show me my shopping cart, I'm Bob”, and “I want to check out, I'm Bob”.

On the server side, the server uses **sessions** stored in a database to track user activities. Cookies and sessions enable the activities to be stateful. You can put a user session in a cookie as well, but the size is limited.

Common HTTP **actions** include GET, POST, DELETE, etc.

Common **status codes** include 200, 404, and 500.

An HTTP request can contain **headers**. Some are well-known (content-type); others can be random and application-specific.

**HTTPS** is HTTP + TLS.

### HTTP long polling

Long polling: keep the request open for the server to respond.

The TCP connection needs to be refreshed periodically.

### **Client-side rendering**

Traditionally, HTML is generated on the server side and transmitted to the browser. But it's also possible to generate HTML on the client side, i.e. your browser. The server may send you the resources and javascript programs. The programs run in your browser to generate the HTML.

### **Websocket**

HTTP is like a one-way channel; only the clients can initiate requests. If the server has something to send to the client, it can't do it until the client asks for it. Websocket is another TCP-based protocol that enables two-way communication. It's stateful. It maintains a TCP connection all the time. It's good for chat applications.
