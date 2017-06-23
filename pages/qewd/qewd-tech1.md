---
title:  Explaining QEWDjs
keywords: QEWD
sidebar: home_sidebar
permalink: qewd-explained.html
folder: qewd
filename: qewd-tech1.md
---



**QEWDjs**- Integration Framework- Quick & Easy/Quality Enterprise Integration

It is a multi-functional and high-performance solution, powered by the QEWD.js framework, which fulfils a number of roles:


 **Key features of QEWD**

**1) Front End Facing RESTful API service**

- QEWD acts as a middleware service (including web-server) to support the RESTful API endpoints 

- QEWD.js uses Express.js to provide the web server functionality (Koa.js is also a configurable alternative)

-     The middle tier implements the RESTful endpoint server-side logic

**2) Security Broker**

-       It implements and manages the security for frontend applications and external/backend systems. 

-  The security approach involved in the showcase "secure mode" is currently based on OAuth2/openID Connect using JSON Web Tokens (JWTs). 

- The QEWD.js middleware provides several levels of audit trail, one via the QEWD-monitor application, and also, optionally, using its "resilient mode".   These facilities mean that any/all access can be examined at a session- or individual message-based level

**3) Integration & Federation Framework**

-       Qewd also acts as an integration and  federation platform, communicating with any/all internal data sources required plus any/all legacy/external systems that need to be connected to/integrated with. 


- To simplify and automate this transformation work, the middle tier includes a JSON transformation engine ([https://github.com/robtweed/qewd-transform-json)](https://github.com/robtweed/qewd-transform-json)) which allows the transformations to be described via a template which is, itself, a JSON document.  The template syntax allows not only mapping from locations within the source (input) JSON document, but also the use of custom functions for more complex transformations (eg date formatting) to transform as the external (output) system requires.  An associated graphical editor tool ([https://github.com/robtweed/qewd-transform-json-editor)](https://github.com/robtweed/qewd-transform-json-editor)) can be used to facilitate the design and maintenance of the transformation template document files, which greatly simplifying the transformation process.

**4) High Performance Cache**

-     Qewdjs also provides an integrated high-performance session cache, which minimises the traffic to and from external systems.  This makes use of the persistent JSON objects and document database within QEWD.js which, in turn, is supported by either the Redis or GT.M open source databases acting as the cache.


**5) Catering for the Unique Database Technology of "Global Storage"**

In designing the QEWDjs has been critically important that it is capable of:

* supporting mainstream databases (eg MySQL etc) via the npm ecosystem

* federating external data repositories such as Ethercis via standard protocols such as RESTful APIs

These, of course, are standard requirements for most middle tier platforms.  However, there’s a further capability that QEWD.js offers.

One of the specific design features of QEWD.js is its ability to seamlessly integrate with what are often referred to as "Global Storage" databases. This database technology has a long history, but is little known and poorly understood in the IT mainstream.  They are described in this paper on Global Storage technologies: [http://www.mgateway.com/docs/universalNoSQL.pdf](http://www.mgateway.com/docs/universalNoSQL.pdf)

This paper describes how such databases can behave, simultaneously, as: 

* Key Value Storage

* Tabular( and Columnar) Storage

* Document (inc JSON/XML) Storage

* Graph DB

* Object Storage (e.g. Images)

* Relational DB (SQL)




There are two leading databases in this category: Intersystem Cache (proprietary) and FIS GT.M (open source) - all with In-memory Performance and On-disk Integrity.  These technologies are made natively accessible to the Node.js ecosystem via QEWD.js (the only Node.js framework designed to do so). 
Furthermore, the Ripple Foundation has supported QEWD.js development work to create an abstraction on top of the highly-popular, well-understood Redis database, so that it, too, can be leveraged as a multi-model "Global Storage" database using identical APIs to those used for Cache and GT.M.

As a result, by using QEWD.js as the basis of its middle tier, Ripple is uniquely positioned to easily integrate with the most important and most common database technologies that are found in highly challenging enterprises such as the world of finance and healthcare.

