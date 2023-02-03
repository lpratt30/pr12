
# Multi-Threaded Client-Server 
## Project 1, Grad. Intro to OS   						
**Luke Pratt** - 02/03/2023

**Overview** 
In addition to the transfer of data between being pervasive in daily life, it is particularly relevant to operating systems. The OS is responsible for blocking the operations and memory a given process can access at a given time, while processes may need access to the same data or need to transfer outputs between each other. Multi-threaded client-server communication is therefore a natural way to start studies in OS. 

The goal of this project is to build a client and server to request and write data (client) or to serve those requests (server). TCP as well as an HTTP-like protocol are used. It starts from creating a server-client connection that can transfer a single "hello world". It then progresses to transferring files. After the successful transfer of files, an API similar to libcurl's easy interface is implemented, which provides convenient functionality for a user to abstract the client-server connection and data transfer process. Finally, it converts that same application to be multi-threaded to execute multiple file transfer requests simultaneously. This project is written in C in Linux Ubuntu 20.04 using VisualStudios. Debugging is done with logging statements and GDB, while Valgrind is used to troubleshoot leaks.  

## Warmups
**Client-Server Connection** 

The warmups for this project follow [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/html/). Any time a connection is required in this project, it follows Beej's guidance.  While the guide is followed closely, the juicy bits are amalgamated together. 

The process to prepare a server for accepting connections is as follows:
 - specify a hostname, in this case NULL or "localhost" for a computer to check its own information
 - Initialize an addrinfo structure (hints) specifying terms of the connection, such as transmission type (TCP) and IP version
 - Initialize an addrinfo structure that holds information about the addresses you are trying to connect to
 - Call getaddrinfo(hostname, port_str, &hints,&servinfo); to modify servinfo into a linked list of available addreses
  - Iterate through the list of addresses and create a connection socket for the first available address on the port 
  -  listen() on that socket for incoming connections. Then, while(1) forever to accept client connections and serve

The process for a client to connect to a running server is as follows:

 - Specify the hostname (I.e www.google.com or "localhost" for your own computer) and port number in string format
 - Initialize the same addrinfo structures just as the server does and call getaddrinfo(hostname, port_str, &hints,&servinfo);
  - Iterate through the list of addresses and create a connection socket for the first available address on the port 

![alt text](https://github.com/lpratt30/pr12/blob/main/yarrr.PNG)
  
The purpose of a socket is to act as an interface for communication, such as send()ing and recv()ing data. send() and recv() are both calls that allow you to specify the amount of data you would like sent or received across a socket interface, but neither makes any guarantees that much data will send or receive. The only guarantee made is that the data will arrive in order if TCP is specified on the socket. Therefore, anytime these are used they need a while(bytes_done < bytes_needed) loop. 

Once a socket is created, the client may then connect to it and send() or recv() data. However, these are both blocking, so if something goes wrong or malicious, the client/server can get hung. SO_SNDTIMEO and SO_RCVTIMEO socket options may be set to timeout the socket after x seconds or microseconds of unresponded send and recv calls. Here, 1 second is arbitrarily used. Additionally, in this project, a limit to the number of allowed fail sends/recvs is used. This was to allow for some level of robustness of design without risking losing too much performance to errored connections.

Another socket option used is SO_REUSEADDR. Normally, a connection to the same port will be blocked for some time to ensure there is no more transmission happening. Using this option allows instant reuse of an address- great for testing.  

A critical choice when working with sockets is how a closed connection may be handled. Normally, the server should never close the client's connection for them, as the server has no prior knowledge of the client's API or if the client has recv() all data. Here, the shutdown() function is called by the server. This function closes down only the server's connection to the socket and allows for any remaining data to be transmitted. The client knows the connection is over when it recvs a message containing 0 bytes. 

A final note is that two internet protocols are used, IPv4 and IPv6. IPv4 is the oldest and most commonly used while IPv6 was created with the idea that one day we may run out of IP addresses on v4. The issue is that IPv4 and IPv6 follow different data structures. However, as detailed by Beej, code can be written to be agnostic to IP address type, particularly by specifying AF_UNSPEC when passing hints into getaddrinfo. 







## Part I

Part I of this project is creating a library to handle file transfer requests with an API similar to libcurl's "easy" interface. The file transfer request follow an HTTP-like protocol. Namely, gfclient.c and gfserver.c are written to meet the requirements outlined by gfclient.h and gfserver.h specifications. 

Further challenges of Part I involve error handling. What if the server disconnects while transferring data or its header? 
What if the server never sends the data it said it would, but maintains the connection? Likewise problems exist 
originating from the client. Additionally, any flavor of incorrectly formatted request/response header may exist.

What good is a server or client that can only do one thing at once? Part II of this project takes that same interface, uses mutex to make it thread safe, and transforms into a multi-threaded application to handle multiple requests simultaneously. 


Your README file is your opportunity to demonstrate to us that you understand the project.  Ideally, this
should be a guide that someone not familiar with the project could pick up and read and understand
what you did, and why you did it.

Specifically, we will evaluate your submission based upon:

- Your project design.  Pictures are particularly helpful here.
- Your explanation of the trade-offs that you considered, the choices you made, and _why_ you made those choices.
- A description of the flow of control within your submission. Pictures are helpful here.
- How you implemented your code. This should be a high level description, not a rehash of your code.
- How you _tested_ your code.  Remember, Bonnie isn't a test suite.  Tell us about the tests you developed.
  Explain any tests you _used_ but did not develop.
- References: this should be every bit of code that you used but did not write.  If you copy code from
  one part of the project to another, we expect you to document it. If you _read_ anything that helped you
  in your work, tell us what it was.  If you referred to someone else's code, you should tell us here.
  Thus, this would include articles on Stack Overflow, any repositories you referenced on github.com, any
  books you used, any manual pages you consulted.


In addition, you have an opportunity to earn extra credit.  To do so, we want to see something that
adds value to the project.  Observing that there is a problem or issue _is not enough_.  We want
something that is easily actioned.  Examples of this include:

- Suggestions for additional tests, along with an explanation of _how_ you envision it being tested
- Suggested revisions to the instructions, code, comments, etc.  It's not enough to say "I found
  this confusing" - we want you to tell us what you would have said _instead_.

While we do award extra credit, we do so sparingly.
