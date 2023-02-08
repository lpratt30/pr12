
# Multi-Threaded Client-Server 
 Project 1, Grad. Intro to OS   						
**Luke Pratt** - 02/03/2023

**Overview**    
    
While the transfer of data is pervasive in daily life, it is particularly relevant to operating systems. The OS is responsible for blocking the operations and memory a given process can access at a given time, while processes may need access to the same data or need to transfer outputs between each other. Multi-threaded client-server communication is therefore a natural way to start studies in OS. 

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
  -  listen() on that socket for up to n pending incoming connections. Then, while(1) forever to accept client connections and serve

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


## Library perspective

The connection process for both client and server is done in a function that does as described in the warmup in a dedicated function that returns the socket descriptor. This kept the main working methods much shorter.  

get file server: 
 - Creates a socket to listen for connections at a specified port 
 - Waits to receive a connection and subseqent request for data transfer from a client 
 - Creates a context data structure containing information specific to this request
 - Parses the request for header to determine if valid request 
 - If valid request, calls the user provided handler to go to the file path and transfer data across
	 - Has functions gfs_send and gfs_send_header which are called by users handler. These transfer data as described in the warmup
	 - gfs_send_header sends an HTTP like header containing the status of the request and the file size in bytes if request status is OK
 - If invalid request, respond with GF INVALID
 - Shutdown() connection with client and cleanup memory 
 - Continue serving new requests 


get file client: 
 - Receives a file path from a user desired for download 
 - Checks for available connections on a specified port and host 
 - Creates an HTTP like request header and sends it to the server 
 - Waits to recv() the response header and data from the server
 - Receives more or less data than the size of a response header 
 - Parses the response to check status, if invalid response header, or if potentially valid partially received response header
	 - If partially valid, continue to listen for some attempts and timeout limit for more data 
	 - If confirmed invalid, set request status and exit 
 - Extract file length if valid full response header
 - Point to the location in buffer the byte after the size of the response and calls the user write_func
 - Continues to recv() data until either the server stops ending data or all expected data is received 
 - If server stops early, sets status as GF_ERROR
 - sets status to GF_OK and returns 0 to gf user to indicate the work is succsesfully, else returns negative integer and has otherwise approiately set GF status 

## Testing & Debugging Part I

One critical error was that the files contained the exact number of bytes after being transferred as they should, but the file corrupted. The command cmp file1 file2 showed that the files were off at the very first bytes. To visualize, a .txt file was sent, and shown to be corrupted by ~90 bytes. The corruption wasn't continuous, some invalid characters existed in the middle of the file. Buffers were confirmed to have no off-by-one errors or other. A binary gfclient and gfserver was downloaded from the interops thread on Piazza (Matthew Borland) and tested against these implementations. The files did not corrupt for binary gfserver, so the client was known to be the issue. 

The code functioned as it should when exactly the response size in bytes was recv() in one call, but no more no less. This didn't track because the parser for the response header was doing byte level memory comparisons between the buffer and possible response header format, only up to the length of received bytes. Why would a byte-level parser function incorrectly when parsing from a buffer that was written to (with the correct buffer offset updated by the number of bytes received in prior calls) once, but not multiple times? Or once, but with byte level comparisons only at the start of the buffer? The stranger thing yet is the files didn't always corrupt, they had about a 15% chance to not corrupt. This random chance of succsess pointed that it may be memory management between succsessive calls, but this was tested to not be the case by ensuring all buffers were zero'ed and all vars initalized for each request. The chars being passed into the parser was manually printed before being passed to show it was passing the expected scheme, format, and file size.

After going line by line with GDB, the issue turned out to involve the manner void* buffers were being parsed from. Because we can't know what type of data will be transmitted ahead of time, all the buffers used were void* buffer[BUFSIZE]. However, when the buffer was being passed into the parsing function, it resulted in undefined behavior when doing memcmp of bytes in the buffer to possible header formats represented by strings (not including comparisons to null terminator). 

It was then tested against recvs() of 1 bytes at a time, response_header bytes, and 8192 bytes. It was found to work in all cases. This parser (and corresponding 1 second socket timeout option that allows n fail recv() attempts) is robust as it can handle recvs() that have a lot of latency and randomness. After having a robust connection system and response parse parser, server responses other than GF_OK were tested to check for correct assignment of GF_VALUE and return values which was relatively trivial. 

## Part II

Part II of this project takes the same interface, uses POSIX and boss-worker with conditional signaling, and transforms into a multi-threaded application to handle multiple requests simultaneously. 

The gfserver and gfclient both have the boss set to a higher priority than the workers using this stack overflow [post](https://stackoverflow.com/questions/27558768/setting-a-thread-priority-to-high-c) as a reference. This is so that boss has highest priority to add new requests into the que such that performance isn't bottlenecked by the boss fighting for que mutex from workers trying to receive work orders. 

All workers wait on a conditional variable that is the number of work orders to be done to be positive. The serve runs indefinitely. For the client to end processing, the number of items is set to be negative to process. This results in less lines of code (I.e cleaner more maintainable code) and slightly more computationally efficient to process than using the poision-pill method. 

Part II was greatly assisted by this [diagram](https://docs.google.com/drawings/d/1a2LPUBv9a3GvrrGzoDu2EY4779-tJzMJ7Sz2ArcxWFU/edit) which is provided in the assignment specification. It details the flow of the API. Lecture P2L3 served as a guide for thread creation and conditional/mutex handling. Additionally, questions asked and answered by discussions in Piazza and Slack by students and TAs, were a great resource for understanding Part II.


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
