# Implementation
This chapter describes some used mechanism in more detail. It also introduces some classes and methods, but it is not supposed to serve as a detailed and full documentation. The Doxygen software has been used to generate documentation so the complete overview of the code can be found in the attached HTML document. The program is implemented in the C++ language. This allows the use of standard C library functions. Also the functionality provided by the C++ STL is exploited. Some of its containers are used as a base for containers with synchronized access implemented in the framework. Some of the functionality from the C++11 standard is also used, so the use of the program is limited to computers with compiler that supports this standard.

## Networking
One of the most important issue is how to handle the networking. The chosen approach will be described in this section. As it has been said already, the program uses TCP for any kind of network communication. This is certainly good option when we want to handle data transfers, however, it can be considered unnecessarily demanding for simple tasks. However to preserve the implementation simple, we chose not to use UDP. At least the advantages of the use of TCP are exploited. To utilize the possibilities of the operating system, standard socket API is used. Information about addresses are stored in the \textit{sockaddr\_storage} structures which are suitable for storing both IPv4 and IPv6 addresses. This approach makes it possible to switch between both the versions easily. There are also some helper functions to work with address structures which are generic in the use of an address family. To provide easy manipulation with addresses, the structure \textit{MyAddr} was created which groups the related functionality together.

### Spawning connections
The ultimate class to handle the connections is the \textit{NetworkHandler} class. It provides all the necessary functionality. Each instance of the program binds to the given listening port and starts accepting connections. When the connection is accepted, new thread is spawned to handle the connection. When the program wants to make the connection, it provides structure referring to a given neighbor together with the commands to execute. to the \textit{NetworkHandler} instance.  The connection is then spawned and handled. Here we encounter the topic of commands. Each action consists of set of commands that implements it. Commands are instances of a given class. Each command has two parts. When one node initiates the connection, it provides the vector of commands to be executed. They are processed in the loop in the following manner: The command's \textit{execute} method is invoked. It communicates over the network. Firstly it sends name of the command to be invoked in the peer node. The peer's thread loops too. First it reads the name of the command and then it invokes it. In that moment there are command methods running in both nodes and they can communicate. When they end, the receiving node waits for another action while the initiating node spawns next command from the vector, if any is present. Otherwise it ends the connection. This mechanism is used to handle all the network communication. What happens in case of problems is described in the section about handling errors.

Incoming connections are always handled asynchronously. Nevertheless, the outgoing connections may be handled in the synchronous way. In can be essential sometimes. For example if the node needs to obtain some potential neighbors because his list is empty, it has to wait until the action ends, because if it just sent the request and continued, it would probably find the list still empty.

### Protocol
As it was said, the communication happens thanks to commands. Thread that handles the incoming connection is provided with respective file descriptor and address of the communicating node. First data that appears are considered as the listening port number of the node, so it can be identified in the neighbors list or added to the list of potential neighbors. Then the name of the command is sent, which is represented by the \textit{enum} type. If no error occurs, the confirmation is send and the appropriate command's method is invoked. After returning from the method, another command is read. If there are no data left, the connection is closed.

### Transferring the data
First problem every network application has to deal with considers the byte order. The POSIX sockets API provides set of functions to deal with it. Namely it's \textit{htons} and \textit{ntohs}, or \textit{htonl} and  \textit{ntohl} respectively. These functions convert the native representation of short (long) data types to the network byte order. The framework uses these function.

\subsection*{Base transfer functions}
Each of the following functions has basically two parts, sending and its receiving counterpart. Integers are stored as the \textit{int64\_t} type which ensures correct communication even between 32 and 64 bit nodes. Then they are sent using the mentioned converting functions. When the string is transfered, first is sent the length of the string, followed by the appropriate number of characters. Commands are sent as numbers, wrapper functions are used which converts between the \textit{enum} type and \textit{int32\_t} explicitly. Sending the structures containing the addresses is managed by another special function. It converts the address to strings and sends it in this form, followed by the port number. This allows to handle both IPv4 and IPv6 addresses, the format is recognized during the reversed conversion.

\subsection*{Transfering files}
The most delicate task is to transfer the files. The file is first check, and its size is determined. Then it is sent and then the function repeatedly reads part of the file to buffer and sends it until all file is processed. The count of sent bytes is compared with the actual file size in the end. The receiving side accepts the bytes and writes it to the file continuously, the file size is checked in the end. The data are first written to a temporary file and after the successful transfer it is renamed. This mechanism prevents inconsistency of the received files. Each file is referenced by the \hyperref[the-transferinfo-structure]{structure}. Among other things this structure stores counter of unsuccessful sent tries. If the counter exceeds given limit, the neighbor is treated as invalid and his state is set to non-free. The file is also checked, if it is not valid for some reason the whole process has to be aborted because there is no way how to fix one specific file.


### Handling errors
Unfortunately the network environment is quite error prone and all the action has uncertain results. Moreover, the communication can be interrupted at any time. Because of this it is important for the network application to be able to deal with different error situations.

\subsection*{Errors during the connection handling}
Almost all the functions indicates error by the negative return code. These codes are checked so the error can propagate. If some error happens during the control communication, the loop handling the connection simply brakes, so the connection is closed. The commands are invoked in the try-catch block, so if the data have been corrupted or the synchronization has been lost, the next invalid command name raises an exception and the error is handled. The loss of the synchronization may be detected thanks to the obligatory confirmation of every command.

\subsection*{Other errors}
If an error happens during the file transfer, the receiving side detects inconsistency thanks to checking of the file size, so the bad file can be removed. Generally, if any error is encountered during the communication, the corresponding execute method indicates it by its return value, so it can be propagated further. The signal handler also has to be set to cover situations when the connection is destroyed unexpectedly and the SIGPIPE is delivered.

## Structures' overview
This section describes two main structures used in the framework. They common sign is, that they inherit from the Listener class, so they have to implement the invoke method. This fact makes it possible to use them as \hyperref[periodic-actions]{periodic listeners}.

### The TransferInfo structure
This structure serves for referencing the chunk. It contains flags used for transfer, addresses (source and destination), information about the video and path locating the physical position of the file. This field is important because it makes it possible to reference the file. It is changed several time during the process; as the state of the chunk changes, it it located in various directories and it is important to keep the field actual. The structure also contains information which help to determine the encoding process as well as statistics describing the result. These are used to compute and update the quality of the neighbor. The structure is also equipped with a pair of methods that transfers it over the network, which simplifies its usage. Periodic invocation decreases the timer. It is used when the referenced chunk is waiting for return. For the first time the timer reaches zero the neighbor is checked, if it is alive. If yes, the timer is send one more time. If the neighbor doesn't respond or the timer reaches zero for the second time, the chunks is resent.

### The NeighborInfo structure
Instances of this structure are kept in the \hyperref[neighborstorage]{NeighborStorage class}. They represent the neighbor, that is its address and listening port. It also keeps the information about quality of the neighbor. The time from last check is stored too. Periodic invocation causes the timer to decrease and possibly contact the neighbor to refresh the state.

## Important classes

### The NeighborStorage class
It is class used for storing the \textit{NeighborInfo} structures. It provides several methods to maintain neighbors list while preserving synchronization. This is crucial, because the information about neighbor can change any time but it's desirable to keep our knowledge consistent. The class has one instance and helps to keep the information about neighbor at one place, so the manipulation can be controlled.

### The NetworkHandler class
This class is used to handle all the networking issues. It provides functionality to spawn the connections or contact neighbors. It also hold the list of potential neighbors. This list differs from the neighbors list, because besides address and port, which are necessary, no further info is stored about potential neighbors. Also, most of the other classes have a notion about this list. When the lack of neighbors appears, simply the function \textit{obtainNeighbors} is invoked, which uses the list internally. This class also handles adding of the new neighbors.

### The Data class
It is a singleton class which helps to keep all the data at one place while accessible from anywhere in the program. It holds the instances of the \textit{NeighborStorage}, \textit{State} and all the \hyperref[queues]{queues} used during the transfer and the encoding process. 

There are more significant classes such as TaskHandler and WindowPrinter, which will be discussed in the corresponding sections.

## Periodic actions
The framework uses a mechanism which invokes some actions periodically. Pros and cons of this approach are described in the respective sections, namely \hyperref[problems-alternatives-and-possible-improvements]{the alternatives}. To implement this mechanism, separate thread runs that loops and once after each time quantum it invokes methods of structures that inherits from the \textit{Listener} abstract class and are stored in the special queue. The time quantum is defined as a constant, so all timers actually express count of the quanta left. Obvious disadvantage of this approach is busy waiting that is used in the loop. Alternative approach could use a signal handler and setting an alarm. However, this could lead to not necessary asynchronous interrupts. Since the mutexes are used to avoid race conditions, a problem could occur if the signal interrupted some method holding a "bad" mutex.

## Queues
As it is described in the corresponding \hyperref[distribution-of-chunks]{section}, the references to chunks are stored in different queues, depending on the state in which they are. Since the whole process is non-deterministic, different conditions may occur and more than one thread could need to work with the queue at one moment. This means, that some way of synchronization has to be provided. Because of this, the textit{SynchronizedQueue} has been created. Basically it provides usual functionality that could be requested from the queue, but ensures avoiding race conditions because only one thread at a time can access the underlying data. This is implemented by the use of textit{mutex}. The \textit{pop} method also uses the conditional variable, so if there are no data that can be popped at the moment of invocation it blocks and waits for a signal. 

## Working with input and output
All the file operations are customized for use on the UNIX operating system. Nevertheless, there is possibility to implement respective functions to work on different operating systems easily. 

\subsection*{File operations}
Each chunk is stored in a separate file. For this reason several functions were implemented to allow easier work with the file system. These include functions for manipulation with the filenames, controlling files and working with directories. Also there is a generic function which spawns an external process. This function uses process forking, spawns the desired process and returns contents of its standard output and error streams. The function also accepts time after which it kills the process. The result of the process' run propagates in the function's return value so the caller can react accordingly. /this mechanism is used when working with video, namely splitting, encoding and joining it.

\subsection*{Working with the video}
The video processing is secured by the \textit{TaskHandler class}. When it is loaded, some useful properties are obtained thanks to the \textit{ffprobe} program. To allow easy processing, the output is in the JSON format which is then parsed with the help of the \textit{rapidjson} library. This library consists of header files only and is distributed as a part of the source code. It is available under the MIT license which makes it suitable for usage. These values can then be showed using the F6 key. More importantly, it is used to compute the number of chunks that will be created. Note, that the number of chunks can change slightly during the splitting process. It is caused by the fact, that the teoretically computed count does not consider the positions of key frames. So the chunks can actually have different sizes and thus their count differs. This fact also implies, that each chunks has got slightly different size. When the process starts, the instance of \textit{ffmpeg} process is spawned which splits the file. The resulting files are then stored in the special subdirectory of the working directory. For easy identification, each process is assigned a unique code determined by the time stamp. The chunks are then numbered in increasing order. The names are then stored in the referencing structures that are created at this point. Then the chunks are distributed as described in the \hyperref[distribution-of-chunks]{section} about distributing. Each processing neighbor encodes the chunks using the \textit{ffmpeg} and then sends it back. The information about the chunk, such as the level of the encoding quality are stored in the referencing structure. Once the chunks are collected, first a list of the files to join is created and then the \textit{ffmpeg} is used once more to join all the chunks to the output file.

An alternative approach to splitting the file was used during the development. Firstly the position of every split was computed. Then it was spawned one process per each chunk. The advantage was, that the distribution could begin after the first chunk was created and therefore some time was saved. However this approach turned out to be bad because of the existence of key frames. It does not allow to split the file at the arbitrary position so certain shift was observable in the resulting file.

\subsection*{User interaction}
To provide interaction with the user, the \textit{curses} library is used. User can provide input from the keyboard. There is set no delay of the input, so the buffering is disabled. This causes the need to handle the reading manually but also makes it possible to control the input and accept the commands immediately. The output is provided using the \textit{WindowPrinter} class. This class stores the queue of lines of output and provides functionality to add or remove some. Each record holds the line together with the information of the style how to print it. The screen is divided into four parts. Each part spans the whole width. There is a line displaying available commands at the top. The biggest portion of the vertical space belongs to two windows. The first one displays different information about processing, neighbors or file properties. The second window displays the status changes, notifications and potentially some debugging messages. The most bottom part shows a prompt when user input is required.

Because there is usually a lot of threads that can cause the screen to refresh (this means the particular \textit{WindowPrinter} instance is updated), it is important to allow only one graphical update at the moment, otherwise it could cause inconsistency of the graphical data. This is ensured by a mutex assigned to each \textit{WindowPrinter} instance.

## Synchronization
Because of the nondeterministic nature of the application, it is necessary to provide some kind of synchronization to ensure data and information consistency. This problem is solved by the use of mutex and conditional variable available in the C++ standard library. There are implementations of queue and map like structures which ensures serialized access. These classes uses containers from the STL and synchronization primitives mentioned above. These structures, namely the \textit{SynchronizedMap} and \textit{SynchronizedQueue} are used to store chunk references. Operations with the list of neighbors have to be synchronized too, because for example one thread could be working with the neighbor's reference while the other finds out that the neighbor is not responding and wants to remove it from the list.

Another area which have to deal with some race conditions is output which is displayed on the screen. The output is handled by the \textit{curses} library which provides practically raw access to the screen. This means, that if more threads try to work with the screen at one moment, there is high possibility that they compromise each other and nonsensical data are displayed at the output. Moreover, this situation can also possibly result in the segmentation fault. To avoid these situations, the data that are supposed to appear on the screen are stored in the respective instances of the \textit{WindowPrinter} and mutexes are used to allow only one thread to change the content of the storage or refresh the screen. This approach has a disadvantage, that each call of the routine that produces some output is possibly blocking

## Error detection and recovery
During the process, various types of errors can occur. Errors connected with the networking are discussed in the respective \hyperref[handling-errors]{section}. Here we will describe other errors that could possibly happen.

\subsection*{Neighbor failure}
If an unexpected failure of some node occurs, the other nodes which has it in their lists must react. If the communication is interrupted in the middle, there is no way how the other node can recognize the failure, so the command execution just fails. However this usually leads to repetition of the command which registers the failure. Generally the failure is noted when the try to establish a connection with the given neighbor fails. This can occur when processing a command, checking a neighbor or when the timer assigned with some chunk reaches zero. In every of these situations the same function is used, so the situation is  always treated in the same way. The unresponsive neighbor is removed from the list. If it has some chunks assigned, i.e. they has been sent to it already, they are queued for send to another neighbor. If it has chunks being processed by the current node, those chunks are trashed, because there will not be any us for them.

\subsection*{Chunk disappearance}
After the chunk is send to be processed, it is pushed into special queue where it is checked periodically. If the time is up, the respective neighbor is checked. If it is responding, the timeout is set once again. If it does not respond, or the timer reaches zero for the second time, the chunk is queued for send. Also, in case of the neighbor failure, the neighbor is removed. This can lead to a situation, when one chunk is being processed by two different neighbors at the same time. However, after it returns for the first time, it is put into the dedicated storage, so if it returns afterwards, it is simply rejected. This approach could be improved by introducing new action which would cancel the chunk's encoding process at the remote node. It was not implemented though, because it is not clear on which node the process should be canceled. Connected with this problem is the situation, when one node receives by accident one chunk more times. The files are checked and what is more, the transfer uses temporary files so the worst scenario involves wasteful encoding of the chunk for the second time. Another issue which has to be solved is how to set the timeout. When the neighbor is involved for the first time, we have no information about its performance, so the timeout is set respectively to the size of the chunk, default multiplication factor is used. When the neighbor has already quality factor assigned, it is used to compute the timeout. So the quality coefficient can be seen as time needed to encode and transfer some unit of data.

\subsection*{Other errors}
Errors can also be encountered during the manipulation with the video. Because all the video related problems are manipulated by the external programs, the mechanism is used, which can control the process. The approximate upper bounds are set for each task that is supposed to be executed and if the process' execution takes too long, the signal is send which terminates it, thus the error code is returned.