---
title: QEWDjs Release News/Notes
keywords: release, news
sidebar: home_sidebar
permalink: releasenotes.html
folder: qewd
filename: releasenotes.md
---

## Creating a QEWD microservice

Creating a QEWD microservice

If you read my previous post that introduced QEWD Micro-Services, you're hopefully eager to learn how to use them.  So in this post I'll explain what you need to know in order to get started.

If you look in the QEWD repository, you'll find the folder: 

/example/jwt

In a previous posting I used the example application that you'll find there to explain how to use JSON Web Tokens (JWTs).  This example application also demonstrates how to set up a simple Micro-Service, in this case a service that handles user authentication.  So let me now delve into that aspect of the example application.

If you're wanting to use QEWD Micro-Services, you must also use JWTs - these provide the means by which the user's authentication and session can be cross-communicated and handled by multiple, discrete QEWD servers.  So, take a look at the startup file:

https://github.com/robtweed/qewd/blob/master/example/jwt/startup_file/qewd-jwt.js

You'll see that it configures QEWD to use JWTs, and defines the secret string that is used for signing and encrypting the JWT:

           jwt: {
            secret: 'someSecret123'
          },

However, you'll also see that it defines a micro-service:

  u_services: [
    {
      application: 'jwt',
      types: {
        login: {
          url: 'http://192.168.1.97:3000',
          application: 'test-app'
        }
      }
    }
  ]

You use the "u_services" config property to define an array of micro-services.  Each array element specifies the incoming request's application name, in this case "jwt", and then one or more message types for that application.  In the example, we're just specifying a single request message type: "login".  For each message type, you specify:

- the URL of the QEWD system that you wish to handle that message type
- the QEWD application name on the remote system that will handle the incoming message

So , our example startup file is saying that any incoming QEWD request messages for the application named "jwt" and type "login" will be handled by a an application named "test-app" on a QEWD system at http://192.168.1.97:3000

When you fire up this startup file, one of the first things it will do is to load a module named "socketClient", which establishes a Web-Socket connection to each of the QEWD systems specified in the u_services array.  If you monitor the QEWD system log on the remote QEWD system, you'll see it accepting a standard "ewd-register" message, as if someone had started the test-app application in a browser - to it, there's no difference.

Note that the remote QEWD server's startup file must also configure the use of JWTs, using the same secret string as the master QEWD system, ie:

           jwt: {
            secret: 'someSecret123'
          },

The value of the secret string is up to you, but it must be the same on all QEWD master and QEWD Micro-Service systems

So that's both our QEWD systems configured and running.  Now let's look at the application.  Take a look at:

https://github.com/robtweed/qewd/blob/master/example/jwt/index.html

My previous posting on JWTs explained most of what's going on in this browser-side logic, but look at lines 13-34 which handle the submission of the login form.  As far as this application is concerned, it's just sending a message of type "login" to the QEWD system which started it up and on which it was registered - the fact that this will be handled by a Micro-Service is irrelevant to the developer of the browser-side logic - it's just a standard QEWD message call.

However by virtue of the u_service config mapping we added to the QEWD system's startup file, this request will be automatically forwarded to the second QEWD system that is handling the user authentication micro-service.  So now let's look at its handler module, which, if you remember, by virtue of the u_service mapping, is handled by a module named test-app:

https://github.com/robtweed/qewd/blob/master/example/jwt/microservice/test-app.js

Now, as far as this handler is concerned, an incoming message of type "login" might as well have come from a browser, so it's handled in the standard way.  It will have already verified and unpacked/decrypted the JWT contents and provides it via the session object:

    login: function(messageObj, session, send, finished) {
      // session contains the decrypted JWT payload
    }

So now we can handle the username and password values that came from the user's login form - there's no difference to how you'd have done this in a "conventional" QEWD application.  In a proper production application, we'd use some kind of authentication service or database, but here I'm just doing a simple hard-coded test for a user named "rob" whose password is "secret":

      var username = messageObj.params.username;
      var password = messageObj.params.password;
      if (username === 'rob' && password === 'secret') {
        // successful login in
      }
      else {
        //  unsuccessful login
      }

Let's deal with the un-successful login.  Just as in a "conventional" QEWD application, we use the finished() function to return an error message and release the worker process:

        return finished({error: 'Invalid login'});

If the login is successful, however we want to add a number of things to the session (and therefore the JWT):

- set the authenticated flag (which is invisible within the JWT to the browser user)
- change the session/JWT timeout, probably to a longer timeout such as 3600 seconds

We also might want to convey to the browser some information about the logged in user, eg how to display the user's name, eg:

        session.userText = 'Rob';

We might also want to store the user's identity in the JWT as a secret field (ie invisible to the browser, but visible to a back-end QEWD handler):

        session.username = username;
        session.makeSecret('username');

The "logged in successfully" response message is then returned and the worker released in the standard way:

        return finished({ok: true});

So that's it as far as the login micro-service developer is concerned - it's just a standard JWT-based message handler function.  You neither know nor care if the response is being returned to a browser user or a QEWD system acting as a web-socket client!

What will happen automatically is that the response + modified JWT will be returned to the QEWD client system, which, in turn, forwards the response + modified JWT to the browser.  As far as the browser is concerned, it's received the response + JWT it expected from the QEWD system it communicates with - it's unaware that it was actually handled by a QEWD Micro-Service.

The JWT in the browser now contains the user's identity in a secret claim value.  That will be conveyed in any subsequent requests to the QEWD system which, in turn, may forward it to a different QEWD Micro-Service.  Provided all the QEWD systems share the same JWT secret string, the JWT and its secret claims are available for their use.  The fact that the secret authenticated flag is set to true means the user must have been correctly authenticated, and the secret username JWT claim securely provides the user's identity for use against a database etc.

One cool additional thing to note - we've used the finished() function in our example, but remember that QEWD also allows you to optionally have intermediate messages that you send using the send() function.  You'll find that these are also supported across QEWD micro-services, and are conveyed back to the correct browser client.  I believe this is a feature unique to QEWD Micro-Services.

So, from a developer's point of view, that's all there is to defining QEWD micro-services.  It's really just business-as-usual, with everything handled automatically for you by QEWD simply by you defining the u_services mapping array in your QEWD startup file.  All very simple and straightforward, but, as you're probably beginning to realise, allowing all manner of possibilities in terms of defining a complex and highly-scalable mesh of QEWD Micro-Services.

Rob


## Intro to QEWD microservices 

Intro to QEWD microservices 

In my previous posting about the new support in QEWD for JSON Web Token (JWT) support, I mentioned that it was a key step in enabling MicroService support in QEWD.  In this post I'll give some background to how they work and the thinking behind them.

If you haven't heard about MicroServices and/or want to learn more, there's lots of information available if you do a Google Search.  Here's a good starting point:

https://smartbear.com/learn/api-design/what-are-microservices/

In a nutshell, they are all about breaking down, into separate self-contained units, the processes and functionality that you want an application to provide.  So instead of you having a single, monolithic back-end that handles all the incoming requests, they are separated out into separate systems - usually separate servers (often using containers such as provided by Docker and either Docker Swarm or Kubernates).  Incoming requests are routed to the system that will handle that particular business function or operation, usually using an inter-networking "fabric" based around HTTP(S).

Like everything, there are pros and cons to MicroServices:

- the big pro is being able to separate out business functionality to sub-systems that are designed solely for one dedicated activity, often the responsibility of a specialist team dedicated to that one piece of business functionality

- the second pro is scalability and resilience - you can scale out your application over numerous different systems, with no single point of failure

- a significant con is the increased potential complexity of the resulting mesh of sub-systems, each of which needs managing and maintaining, in addition to some kind of oversight management and maintenance that monitors and ensures the availability of all the individual sub-systems

- a second significant con can relate to the use of HTTPS to provide a secure micro-service inter-networking fabric.  The handshaking required to establish an SSL connection involves multiple round-trips between the client and server system which can take a significant number of milliseconds.  If a request involves the invocation of multiple micro-services across this fabric, or a chain of microservice invocations, this latency can start to add up.  Once completed, the HTTPS connection(s) is/are torn down, and need to be re-established anew when the next request comes in.  Companies such as nginx provide specialist Micro-Service support technologies to work around such issues, which is great, but such solutions introduce more moving parts into an already complex inter-networked fabric.

As it turns out, QEWD was in a pretty good place, once JWT support was added, to achieve the pros of a Micro-Service architecture, whilst going a long way to addressing those two worrying cons.

The key lies in the web-socket support that is built-in to QEWD.  Normally you make use of this by using the ewd-client module in your browser-side logic.  It creates and maintains a persistent web-socket connection to a QEWD server, making use of the socket.io library.  If your browser is connecting over SSL to QEWD, then the web-socket connection is secured over that connection also.  Once the SSL handshaking has completed and the socket connection is established, there is no further overhead involved in using and re-using the web-socket connection, and it's a bi-directional channel.  QEWD provides a simple set of APIs for communicating over the web-socket connection between your browser and back-end logic.

Since QEWD's master process is a Node.js/JavaScript process, there's no real difference between its run-time environment and that of the browser.  So what if the equivalent of ewd-client was made available to a QEWD master process, so that it could connect to another QEWD server over a socket.io-based web-socket connection?  This would allow two or more QEWD servers to establish a secure, high-performance inter-networking fabric.  The cool part of it is that a "server" QEWD system in such a set-up wouldn't actually know the difference between a browser client and a QEWD server acting as a web-socket client.  As a result, a QEWD micro-service could be implemented in just the same way as a normal QEWD back-end application!  

That's what I've done and how QEWD can now support micro-services.  Provided all your QEWD micro-service systems share the same JWT secret string, a master QEWD server (or farm of servers) can establish a web-socket-based inter-networking fabric to as complex a mesh of other QEWD-based micro-service servers as you like.  

You simply define on a QEWD server the micro-service mappings you want for handling incoming requests, in terms of the message application name and type.  If a matching incoming request is received, the QEWD server's master process forwards it to the mapped micro-service QEWD system, complete with the incoming JWT, rather than handling it locally, via its queue, on one of its worker processes.  The micro-service QEWD system to which the request is forwarded could, itself, have mappings defined that further forward the request to other QEWD systems.  On each "client" QEWD system, a callback is automatically set up to handle the response it gets back from the "server" QEWD system, so the response is automatically returned to the client that originally sent the incoming request. 

How you stratify your applications and break them into micro-services is up to you.  It can be as simple or as complex as you like.  Each micro-service is simply implemented as standard QEWD back-end message handler functions.  

There's no set-up and tear-down costs for the SSL fabric - all the traffic is over persistent web-socket connections between QEWD servers that are set up at start-up time.  There's no additional moving parts or technologies - it's just a set of QEWD systems, each of which is just 100% Node.js.

So that's the background to QEWD MicroServices.  If that's whetted your appetite, in the next post I'll explain how to set up a simple MicroService

Rob



## QEWDjs now supports JSON Web Tokens

QEWDjs now supports JSON Web Tokens

We've just pushed out build 2.15 of QEWD which adds a major new capability: full support for JSON Web Tokens (JWT).

By this I mean that you can use JWTs to create completely stateless applications, with the application's state information held within the JWT rather than in the back-end QEWD database-stored Session.

If you're not familiar with JWTs, here's a good place to start:

https://jwt.io/introduction/

Essentially, JWTs provide a secure way of maintaining user authentication and, optionally user-specific session data within a token that is held in the browser or client, not the server.

Before explaining in detail how to use JWTs in QEWD, you're probably wondering what the big deal is about them, and why you might use them in preference to the standard server-side Sessions supported by QEWD.  Essentially there are two advantages:

- 1) a QEWD system can be scaled out to use a farm of QEWD servers.  Provided they all share the same JWT secret string (see below), then it doesn't matter which QEWD server handles an incoming request - all the authentication and state information needed to handle the request is defined securely in the JWT attached to the request.  So there's no need for "sticky" sessions where a user's requests can only be handled by the QEWD system that registered the session in the first place.

- 2) they make it possible to add proper micro-service support to QEWD.  Build 2.15 includes Micro-service support, which I'll explain in a later posting.  Micro-services provides another way of modularising and further scaling out your QEWD systems.

- 3) You can afford to give your sessions very long expiry times if this is something you want to do.  Since the JWT, and therefore the session information, resides in the client/browser, there's no impact in terms of back-end database storage.


Use of JWTs is optional in QEWD.  However, if you want it to be available to your applications, you need to add the following configuration parameter to your QEWD startup file:

       jwt: {
         secret: 'myJWTSecretString'
      }

It's up to you what your JWT secret text string contains - it's a good idea to make it a string that's not easily guessable.  The secret will be used to decrypt, encrypt and sign your JWTs.  Note that your existing non-JWT QEWD applications will continue to work as before, even if you enable JWTs.

You'll also need to update one other module:

- ewd-client (to build 1.17 or later)

If you're building a "traditional" web application where you load all your JavaScript manually at run-time using < script > "tags, you'll need to copy the ewd-client,js file to your QEWD web server root path (eg ~/qewd/www)

You'll also need to load a JWT JavaScript client library into the browser, in order to access data within the JWT.  Probably the most well-known one is:

https://github.com/auth0/jwt-decode

Install this locally into your QEWD web server root directory and load it into the browser, eg:

    <script src="jwt-decode.js"></script>


In your application's browser-side JavaScript, you start up the application (ie once everything is loaded into the browser) using:

        EWD.start({

          application: 'myJWTApp',

          jwt: true,

          jwt_decode: jwt_decode,

          io: io

        });


In other words, you need to tell the EWD client that you're using JWTs for this application, and that it should use the jwt_decode library to decode and access data within the JWT

Having done that, you just build out the client/browser-side of your QEWD application exactly as before, and the ewd-client will do the rest for you.  Send your messages using EWD.send as usual.  ewd-client will automatically send and maintain the JWT for you behind the scenes.

The one thing you'll need to know is how to access information in the JWT that has been sent from the back-end:  the JWT payload is automatically available via the object EWD.jwt

eg to discover the application name:

   var appName = EWD.jwt.application;

The JWT is updated after every request to the back-end.  At the very least, the expiry time will be updated.  However, the back-end QEWD message handler function may add values into the JWT that you can access in the browser.


So how about the server-side of your applications?

Again, there's not a great deal of difference from the developer's point of view.  The incoming JWT will be automatically authenticated and decrypted, and its contents presented to you as the session object that you'd normally have access to, eg:


    myRequestType: function(messageObj, session, send, finished) {
      // session is an object containing the decrypted contents of the JWT attached to the incoming request
    }

Unlike when using QEWD's server-side sessions, you access properties directly within the session object, eg:

   var application = session.application;

To add a property (which may be a string, numeric, boolean, object or array) to the JWT, just add it to the session object, eg:

  session.currentDate = new Date().getTime();

when the JWT is returned to the browser, this property can be viewed using:

   var currentTime = EWD.jwt.currentDate

A key feature of JWTs is that values (aka claims) within it cannot be arbitrarily changed within the client, as this will invalidate the signature that accompanies the JWT.  This means that data that is for later use at the back-end can be safely made available to the browser.  Nevertheless, there mayl be data that you want to store in the JWT for subsequent use at the back-end (ie when handling some subsequent request from the browser) but which you feel is too sensitive to be viewable in the browser (eg by someone using the browser's JavaScript console).  For such data, you can use QEWD's session.makeSecret() function, eg:

   session.myPrivateValue = 'a sensitive piece of info'l;
   session.makeSecret('myPrivateValue');

All session properties that are made secret in this way are encrypted using the JWT secret key and, although sent to the browser, cannot be used or decrypted (the browser doesn't have access to the JWT secret key).  All you'll see in the browser is a JWT claim named "qewd" that has an unusable encrypted value.

However, when the JWT is returned to the back-end in some subsequent request, the secret values are decrypted automatically for you, so you simply do this in a later message handler function:

   var myPrivateValue = session.myPrivateValue;   // it's automatically decrypted and made available to you!

User authentication is handled similarly to "standard" QEWD, ie, initially, on registration of the application, a secret JWT claim - authenticated - is set to false.  You can set this to true at the back-end in your login handler method, eg:

   session.authenticated = true;
   session.timeout = 3600; // increase the session / JWT timeout to 1 hour

That's pretty much all there is to creating QEWD JWT Applications

Example

I've included a worked example of a simple QEWD JWT application within the QEWD repository - look in the /examples/jwt directory, eg:

https://github.com/robtweed/qewd/tree/master/example/jwt

The browser-side example code is in index.html - for ease of demonstration, all the application's JavaScript logic is inline within this file

An example startup file (qewd-jwt.js) can be found in:

https://github.com/robtweed/qewd/tree/master/example/jwt/startup_file

This startup file also shows you how to configure a micros-service - see my next posting for details.

The back-end module for the example application can be found in:

https://github.com/robtweed/qewd/tree/master/example/jwt/node_modules

You'll see that the handler function logic is very similar to a standard QEWD application.


Conclusion


That's about it, in terms of how and why to use JWTs for your QEWD applications.  JWTs are a pretty hot topic these days, and are very much seen as the modern way forward for applications.  Once again, QEWD makes it simple for you to use them - I've done all the hard work for you!

The one drawback I can see for JWTs is if you need to use large volumes of state information - conveying a lot of state data between the browser and back-end on every request and response would quickly become very inefficient.  If you're in that situation, you have 2 choices:

- revert to using standard back-end database-based QEWD sessions
- store your own pointer in the JWT and use that to point to the user's state data in a database of your choosing (eg the one you've already attached to QEWD - Cache, GT.M or Redis).  The downside of this is that you'll need to figure out the maintenance and garbage-collection logic for such a database record.

But for most applications, you probably don't need that much state data, in which case JWTs are an interesting alternative to the older server-side techniques for authentication and session management.

Have fun with JWTs and QEWD!

Rob 

