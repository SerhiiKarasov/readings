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
* be aware with the syntax
```
// declares function t that takes a single parameter(pointer to function taking zero arguments, return type background_task)
// returns and std::thread object, instead of launching a new thread
std::thread t(background_task());
```
correct versions 
```
std::thread t((background_task()));
std::thread t{(background_task())};
std::thread t([] (){
    do_something();
    do_something_else();
});
```
* thread join -> waiting for its end
* thread detach -> leave it run on its own
* you need to be sure that the thread is correctly joined/detached before object lifetime ends
```
#bad example

#include <iostream>
#include <thread>
using namespace std;
void do_something(int i)
{
    cout<<"Hello World" << ++i <<endl;
}

struct func
{
    int &i;
    func(int &i_):i(i_){}
    void operator() ()
    {
        for(unsigned j=0; j<1000000; ++j)
        {
            do_something(i);
        }
    }
};

void oops()
{
    int some_local_state = 0;
    func my_func(some_local_state); // variable is deleted on finish of the function
    std::thread my_thread(my_func);
    my_thread.detach();  // don't wait for thread finish
}//here thread may be still running


int main()
{
    cout<<"Hello World"<<endl;
    oops();
    cout<<"Hello World_2"<<endl;

    return 0;
}
```
* better to copy the data into the thread rather than sharing the data
* check if objects are not containing pointers and references 
* bad idea to create thread within the function that has access to local variables, unless thread finishes earlier than function.

### Waiting for the thread to complete
* join() is bruteforce, if you need more control over thread use condition variables, futures
* calling join() cleans up storage associated with thread so the std::thread object is no longer associated with the now finished thread. 
* calling join() makes object no longer joinable, 
* use joinable()
  
### Waiting in exceptional circumstances
* be sure that join() or detach() was called before thread object is destroyed
* problem is when exception between thread started and join() called
* make sense to cover exception case with join()
hardcore solution is
```
struct func();
void f()
{
    int some_local_state = 0; 
    func my_func(some_local_state);
    std::thread t(my_func);
    try
    {
        do_smth_in_current_thread();
    }
    catch(...)
    {
        t.join();
        throw();
    }
    t.join();
}
```
smart RAII solution is 
```
class thread_guard
{
    std::thread& t;
public:
    explicit thread_guard(std::thread& t_):t(t_){}
    ~thread_guard()
    {
        if(t.joinable())
            t.join();
    }
    thread_guard(thread_guard const&)=delete;
    thread_guard& operator=(thread_guard const&) = delete;    
};

struct func;

void f()
{
    int some_local_state = 0;
    func my_func(some_local_state);
    std::thread t(my_func);
    thread_guard g(t);

    do_something_in_current_thread();
}
```
### running thread in background
* calling detach() on the std::thread leaves thread to run in backgound, no direct means of communication and can't wait for thread to complete. 
```
std::thread t(do_background_task());
t.detach();
assert(!t.joinable());
```
* can't join() after detach()
* if std::thread is detached, it is impossible to obtain std::thread object
* detached threads are called daemon threads
* can't call detach from a std::thread with no associated thread of execution, same for join. I.e. only joinable() threads are detachable.
* example word editing app, each new document means new thread
```
void edit_document(std::string const &  filename)
{
    open_document_and_display_gui(filename);
    while(!done_editing())
    {
        user_command cmd = get_user_input();
        if(cmd.type == open_new_document)
        {
            std::string const new_name=get_filename_from_user();
            std::thread t(edit_document, new_name)
            t.detach();
        }
        else
        {
            process_user_input(cmd);
        }
    }
}
```
### passing arguments to a thread function
* even though  second parameter is std::string, it is passed as char const *, and is converted to std::string in context of new thread
```
void f(int i, std::string const & s);
std::thread t(f, 3, "3");
```
* risky moment about passing pointer to automatic variable
```
void f(int i, std::string const & s);
void oops(int some_param)
{
    char buffer[1024];
    sprintf(buffer, "%i", some_param);
    std::thread t(f, 3, buffer);//oops can be finished before the cast from char const * to std::string would be done -> undefined behaviour
    t.detach();
}
```
the correct solution for the problem is cast/dangling pointer, is to perform it before passing to function
```
void not_oops(int some_param)
{
    char buffer[1024];
    sprintf(buffer, "%i", some_param);
    std::thread t(f, 3, std::string(buffer));
    t.detach();
}
```
* vice verse, object is copied, but you wanted a pass via reference
```
void update_data_for_widget(widget_id w, widget_data& data);
void oops_again(widget_id w)
{
    widget_data data;
    std::thread t(update_data_for_widget, w, data);
    display_status();
    t.join();
    process_widget_data();
}
```