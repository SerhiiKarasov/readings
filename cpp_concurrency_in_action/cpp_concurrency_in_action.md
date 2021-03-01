# Hello, world of concurrency in C++
### task switching on single processor system
### concurrency with multiple processes
 * divide application into multiple, separate single-threaded processes that are run at the same time. 
 * Problem is here to communicate between processes - OS sets some restrictions, so it is slow and complicated. 
 * From another hand it is easier to write concurrent code with processes. You can then move some processes to other machine.
 * c++ standard is does not provide about inter-process communication. hence it is platform specific
### concurrency with multiple threads
* because of shared address space and lack of protection of data between threads makes overhead of communication smaller(then in case of processes)
* harder to write safe code
* c++ standard is only about inter-thread communication
### why concurrency
* separation of concerns
* performace
### separation of concerns 
* split ui from logic 
### performance
*