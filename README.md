
# Multi-Threaded Client-Server 
 Project 1, Grad. Intro to OS   						
**Luke Pratt** - 02/03/2023

**Overview**    
    
While the transfer of data is pervasive in daily life, it is particularly relevant to operating systems. The OS is responsible for blocking the operations and memory a given process can access at a given time, while processes may need access to the same data or need to transfer outputs between each other. Multi-threaded client-server communication is therefore a natural way to start studies in OS. 

The goal of this project is to build a client and server to request and write data (client) or to serve those requests (server). TCP as well as an HTTP-like protocol are used. It starts from creating a server-client connection that can transfer a single "hello world". It then progresses to transferring files. After the successful transfer of files, an API similar to libcurl's easy interface is implemented, which provides convenient functionality for a user to abstract the client-server connection and data transfer process. Finally, it converts that same application to be multi-threaded to execute multiple file transfer requests simultaneously.

This project is written in C in Linux Ubuntu 20.04 using VisualStudios. Debugging is done with logging statements and GDB, while Valgrind is used to troubleshoot leaks.  

## Warmups
**Client-Server Connection** 

The warmups for this project follow [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/html/). Any time a connection is required in this project, it follows Beej's guidance.  While the guide is followed closely, the juicy bits are amalgamated together. 

Critical bits are how to appropriately use the address structure and getaddrinfo calls to be IP version agnostic. 

A key point in the warmups for the file transferthat sticks throughout the entire project is that no calls involving transfer of information by operating system guarantee any amount of data being transferred/received/written. They only guarentee the information arrives in order if using TCP. Every recv, send, read, write, all must be placed into a while(work_done < work_needed) throughtout the project. 


## Part I

Part I of this project is creating a library to handle file transfer requests with an API similar to libcurl's "easy" interface. The file transfer request follow an HTTP-like protocol. Namely, gfclient.c and gfserver.c are written to meet the requirements outlined by gfclient.h and gfserver.h specifications.

Every time a server receives a request or the client receives a response header, there are 3 possibilities. 

A: A full, valid header has been received 
B: A partial, but so-far valid header has been received 
C: A full or partial invalid header has been received

No assumptions are made anyhwere about what sorts of data will arrive and when. There are many combinations of invalid headers, so heuristics like tokenization ended up making things more complicated and error prone. Additionally, the data isn't string data, so string functions may not be use. All header parsing is done with memory comparisons at the byte level, starting exhaustively from the start of the header buffer each time. 

To avoid hung as well as accomodate latency, the server and client both have socketoptions set that will time out after 1 second has passed. Then, they will also allow for up to n failed recvs/sends before taking the request to be failed and aborting. 

## Testing & Debugging Part I

One critical error was that the files contained the exact number of bytes after being transferred as they should, but the file corrupted. The command cmp file1 file2 showed that the files were off at the very first bytes. To visualize, a .txt file was sent, and shown to be corrupted by ~90 bytes. The corruption wasn't continuous, some invalid characters existed in the middle of the file. Buffers were confirmed to have no off-by-one errors or other. A binary gfclient and gfserver was downloaded from the interops thread on Piazza (Matthew Borland) and tested against these implementations. The files did not corrupt for binary gfserver, so the client was known to be the issue. 

The code functioned as it should when exactly the response size in bytes was recv() in one call, but no more no less. This didn't track because the parser for the response header was doing byte level memory comparisons between the buffer and possible response header format, only up to the length of received bytes. Why would a byte-level parser function incorrectly when parsing from a buffer that was written to (with the correct buffer offset updated by the number of bytes received in prior calls) once, but not multiple times? Or once, but with byte level comparisons only at the start of the buffer? The stranger thing yet is the files didn't always corrupt, they had about a 15% chance to not corrupt. This random chance of succsess pointed that it may be memory management between succsessive calls, but this was tested to not be the case by ensuring all buffers were zero'ed and all vars initalized for each request. The chars being passed into the parser was manually printed before being passed to show it was passing the expected scheme, format, and file size.

After going line by line with GDB, the issue turned out to involve the manner void* buffers were being parsed from. Because we can't know what type of data will be transmitted ahead of time, all the buffers used were void* buffer[BUFSIZE]. However, when the buffer was being passed into the parsing function, it resulted in undefined behavior when doing memcmp of bytes in the buffer to possible header formats represented by strings (not including comparisons to null terminator). 

It was then tested against recvs() of 1 bytes at a time, response_header bytes, and 8192 bytes. It was found to work in all cases. This parser (and corresponding 1 second socket timeout option that allows n fail recv() attempts) is robust as it can handle recvs() that have a lot of latency and randomness. After having a robust connection system and response parse parser, server responses other than GF_OK were tested to check for correct assignment of GF_VALUE and return values which was relatively trivial. 

## Part II

Part II of this project takes the same interface, uses POSIX and boss-worker with conditional signaling, and transforms into a multi-threaded application to handle multiple requests simultaneously. 

The server for Part II was the more involved of the 2. In the server, a boss-thread acts as a handler for the server, accepting requests from clients and placing them into a que. The boss is the server, while the workers execute the requests. Every time the boss placed a new request into the que, it also sent a signal, so one of the workers would know to check for a request. Because the server runs indefinitely, all threads were in a while loop with no exit conditions aside from fatal errors. The boss was joined to the main thread to keep the server's operation as blocking. 

The client for Part II also used boss-worker. This time, the main thread was the boss thread. The more pure form boss-worker implementation would have the boss queing up all requests, and then either controlling a flag or enquing a poison pill. However, creating and enquing the requests requires some amount of work, and in the interest of efficieny the boss should be doing as little as possible. Additionally, honoring this convention requires more code (I.e less readable, more error prone, and less maintainable). 

A more efficient approach was to keep a global flag of the number of requests. The worker threads were created one-by-one and a signal was sent after each thread was created, such that the workers did not even need to wait for all of the workers to be created to start. Once the number of remaining requests received is 0, a worker signals to another worker, releases the lock, and exits, such that all workers will leave one-by-one. 

The request process required a content_get call, and it wasn't clear if this was thread-safe or not. It was assumed to not be and was placed within locked operations by the workers, but performance could be improved if it is thread-safe and placing it after the lock. 

## Testing & Debugging Part II

Testing was done locally with a multi-threaded file transfer. First, transfer was done with 1 single worker thread to test basic functionality. 

Testing and debugging for part II was mainly regarding issues of memory ownership across thread stacks and the convention of what information is declared or intialized and where. GDB and logging statements were used to track what information was being contained within pointers-to-pointers-of-structs as they were passed around.  Binaries from the interops Piazza post provided confidence in isolating functionality. 
