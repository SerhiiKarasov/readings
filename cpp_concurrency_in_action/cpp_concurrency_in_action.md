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
* performance
### separation of concerns 
* split ui from logic 
### performance
* process data by chunks
### platform-specific facilities
* native_handle()
### hello world
```
#include <iostream>
#include <thread>
void print_hello()
{
    std::cout << "Hello world" << std::endl;
}
int main()
{
    std::thread t(print_hello);
    t.join();
}
```
# Managing threads
### std::thread
* can be launched, can be one to wait it to finish, or one to have it in background
### argument
* requires smth executable
```
void do_some_work();
std::thread t(do_some_work);

// vs

class background_task
{
    public: 
    void operator() () const
    {
        do_smth1();
        do_smth2();
    }
};
background_task f;
std::tread my_thread(f);

```
* be aware
```
// declares function t that takes a single parameter(pointer to function taking zero arguments, return type background_task)
// returns and std::thread object, instead of launching a new thread
std::thread t(background_task());
```
