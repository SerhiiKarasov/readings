# 1.Hello, world of concurrency in C++
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
# 2.Managing threads
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
    std::thread t(update_data_for_widget, w, data);//thread constructor will just create an internal copy
    display_status();
    t.join();//the data from this thread was not updated
    process_widget_data(data);
}
```
the correct solution for the problem is using local copy instead of reference, is to wrap it with std::ref
```
void update_data_for_widget(widget_id w, widget_data& data);
std::thread t(update_data_for_widget, w, std::ref(data));//thread constructor will just create an internal copy
```
* passing a member function point as the function
```
class X
{
    public: 
    void do_lengthy_work();
};
X my_x;
std::thread t(&X::do_lengthy_work, &my_work);
```
this code will invoke my_x.do_lengthy_work() on the new thread, because the address of my_x is supplied as the object pointer. In order to add some argument to the method just add third argument in the thread constructor


### transferring ownership of a thread
* use case: function creates a background thread but passes back ownership of the new thread to the calling function rather than waiting for it to complete.
* use std::move of std::thread, thread is only moveable, not copyable

### hand-made scoped_guard for the thread
```
class scoped_thread
{
    std::thread t;
public:
    explicit scoped_thread(std::thread t):
    t(std::move(t))
    {
        if(!t.joinable())
        {
            throw std::logic_error("No thread");
        }
    }
    ~scoped_thread()
    {
        t.join();
    }
    scoped_thread(scoped_thread const &)=- delete;
    scoped_thread& operator=(scoped_thread const &)=- delete;
};
 struct func();
 void f(
     int some_local_state;
     scoped_thread t(std::thread (func(some_local_state)));
     do_something_in_current_thread;
 )

```

# 3.sharing data between threads
* at some point of time of work on the data structures the invariant can be broken, it's better another threads do not access to such intermediate data.
* e.g. delete a node in a list 1) identify node to delete, 2) change link in previous node to point to the next one 3) change link in next node to point to the previous node 4) delete the node. At point 2-3 we have broken invariants.
* race condition is anything where outcome depends on the relative ordering of execution of operations on two or more threads.
### avoiding race conditions
* wrap your code with a protection mechanism to ensure that only the thread actually performing a modification can see the intermediate stats where the invariants are broken.
* lock-free programming, modification of the data structure/invariant design so that modifications are done as a series of indivisible changes.
* transaction, modifications of the data structure are performed withing one operation called transaction. There may be transaction log and one step commit. Software transactional memory(STM). C++ doesn't support it, at least in C++20.

### protecting data with mutexes
* the thread library ensures that once the one thread locked a specific mutex, all other threads that try to lock the same mutex have to wait for the unlock.
* mutexes can cause deadlock
* the syntax is: create instance of std::mutex, and call it's member function lock(), at the end call unlock()
* the RAII solution is to use std::lock_guard
```
#include <list>
#include <mutex>
#include <algorithm>

std::list<int> some_list;
std::mutex some_mutex;
void add_to_list(int value)
{
    std::lock_guard<std::mutex> guard(some_mutex);
    some_list.push_back(new_value);
}

bool list_contains(int value_to_find)
{
    std::lock_guard<std::mutex> guard(some_mutex);
    return std::find(some_list.begin(), some_list.end(), value_to_find) != some_list.end();
}
```
the usage of the same mutex makes list_contains and add_to_list mutually exclusive.
* the previous example will not work properly if the list would be passed via reference/pointer. Any code that has an access to that pointer or reference can access/modify the protected data without locking the mutex.

### structuring code for protecting shared data
* check that none of the member functions return a pointer or reference to the protected data
* check that none of the member functions are passing pointers or references to the 'third party' functions.
* bad use case, it is possible to have written access to the protection data which is not mutually exclusive
```
classe some_data
{
private:
    int a;
    std::string b;
public:
    void do_something();
};

class data_wrapper
{
private:
    some_data data;
    std::mutex m;
public:
    template<typename Function>
    void process_data(Function func)
    {
        std::lock_guard<std::mutex> l(m);
        func(data);
    }
};

some_data *unprotected;
void malicious_function(some_data& protected_data)
{
    unprotected = &protected_data;
}
data_wrapper x;
void foo()
{
    x.process_data(malicious_function);
    unprotected->do_something();
}
```
### spotting race conditions inherent in interfaces
* even having all methods covered with mutex it would not help with the deleting of the node in double linked list, as generally you need to modify 3 nodes. it is necessary to cover with mutex the whole list.
* example with stack, we can call push(), pop(), top(), empty(), size();
```
template<typename T, typename Container=std::deque<T> >
class stack
{
public:
    explicit stack(const Container&);
    explicit stack(Container&& = Container());
    template <class Alloc> explicit stack(const Alloc&);
    template <class Alloc> stack(const Container&, const Alloc&);
    template <class Alloc> stack(Container&&, const Alloc&);
    template <class Alloc> stack(stack&&, const Alloc&);
    
    bool empty() const;
    site_t size() const;
    T& top();
    T const& top() const;
    void push(T const&);
    void push(T&&);
    void pop();
    void swap(stack&&);
};
```
the problem is that size(), empty() cannot be trusted. When they are called they are ok, but once they've returned other threads are free to access the stack and might push(), pop() before the thread that needed size() info is starting its work.
* in particular if stack instance is not shared, it's safe to check for empty() and then call top(), or top() and then pop()
```
stack<int> s;
if(!s.empty())
{
    int const value = s.top();//calling top on empty container is UB
    s.pop();
    do_something(value);
}
```
* the problem is related to the interface, need to change the interface design
* option 1: pass in a reference. requires constructing object of stack, may be resource hungry. requires type to be assignable. 
```
std::vector<int> result;
some_stack.pop(result);
```
* option 2: require a no-throw copy constructor, or move constructor. Use std::is_nothrow_copy_constructible, std::is_nothrow_move_constructible.
* option 3: return a point to the popped item.pointers can be copied without throwing. but requires a means of managing the memory allocated and adds some overhead for simple types like int. Good to use std::shared_ptr, is destroyed once the last pointer is destroyed. 
* option 4: option1 + option2 or option1 + option3
* example of thread safe stack data structure
```

#include <exception>
#include <memory>
#include <mutex>
#include <stack>

struct empty_stack: std::exception
{
    const char* what() const throw();
};

template <typename T>
class threadsafe_stack
{
private: 
    std::stack<T> data;
    mutable std::mutex m;
    
public: 
    threadsafe_stack(){};
    threadsafe_stack(const threadsafe_stack& other)
    {
        std::lock_guard<std::mutex> lock(other.m);
        data=other.data;//copy performed in constructor body
    }
    
    threadsafe_stack &operator = (const threadsafe_stack&) = delete; //assignment operator is deleted

    std::shared_ptr<T> pop()
    {
        std::lock_guard<std::mutex> lock(other.m);
        if(data.empty()) throw empty_stack();//try for emptry before assignment
        std::shared_ptr<T> const res(std::make_shared<T>(data.top()));//allocate return value before modifying the stack
        data.pop();
        return res;
    }
    
    void push(T new_value)
    {
        std::lock_guard<std::mutex> lock(m);
        data.push(new_value);
    }
    
    void pop(T& value)
    {
        std::lock_guard<std::mutex> lock(other.m);
        if(data.empty()) throw empty_stack();//try for emptry before assignment
        value=data.top();
        data.pop();
    }
    bool empty() const
    {
         std::lock_guard<std::mutex> lock(other.m);
         return data.empty();
    }
};
```
* the stack is not assignable, no assignment operator and no swap().
* the stack is copyable
* the problem with locking schemes, sometimes you need more than one mutex.but that may lead to deadlocks.

###  deadlock: the problem and solution
* deadlock example: there are two mutexes. Both mutexes need to be locked to perform the operation. There are two threads, each of them hold one mutex only. Both of the threads are waiting another to unlock its mutex. 
* the common advice is: always lock the two mutexes in the same order. Sometimes it is straightforward, when both mutexes are serving different purposes. Sometimes it is not, when mutexes are protecting a separate instance of the same class. 
* solution for taking care of pair of mutexes is to use std::lock. It can lock two or more mutexes at once without a risk of deadlock.
```
class some_big_object;
void swap(some_big_object& lhs, some_big_object& rhs);

class X
{
private: 
    some_big_object some_detail;
    std::mutex m;
public: 
    X(some_big_object const& sd): some_detail(sd){}
    friend void swap(X& lhs, X& rhs)
    {
        if(&lhs==&rhs)//attempting to acquire lock on a std::mutex that is already held by the same thread is UB
            return; // std::recursive_mutex can be locked by the same thread multiple times
        std::lock(lhs.m, rhs.m);//locking of both mutexes, if the exception would be thrown by any of the mutexes locking, the already locked one would be released.
        std::lock_guard<std::mutex> lock_a(lhs.m, std::adopt_lock);//adopt_lock means that the mutexes are already locked and guard should just adopt the ownership of the existing lock, rather then attempt to lock the mutex in the constructor
        std::lock_guard<std::mutex> lock_b(rhs.m, std::adopt_lock);
        swap(lhs.some_detail, rhs.some_detail);
    }
};
```
* the std::lock doesn't help when they are locked separatly
* the most frequent reason of the deadlock is related to locks. But it is not the only reason.
* it is possible to deadlock with two threads without any explicit lock by calling join() on the std::thread for the other. In that case neither thread can progress as it wits for the another thread.
* the solution is to implement the idea, don't wait for a thread if it may wait for your thread
### avoiding nested deadlocks
* don't acquire lock if you already hold one, if you need several locks use std::lock.

### avoiding calling user supplied code while holding a lock
* that code can acquire locks, hence we may have a problem with nested deadlocks

### acquire locks in a fixed order
* if you need to acquire several locks, so then use std::locks to preserve the order of locking
* in some cases it is not so straightforward, e.g. working on list. When you need to access the list, threads must acquire three nodes 1) the one to delete, 2) node before 3) node after.  But if two threads are traversing same list in reverse orders - we can have a deadlock, cause the nodes maybe locked in different orders. thread 1 will lock N and N+1, while thread 2 will lock N+1 and N. The possible solution in this example to define the order of traversal.

### use a lock hierarchy
* it is a particular case of defining lock ordering.
* the idea is that you divide the application into layers and identify all the mutexes that may be locked in any given layer. Code will not be permitted to lock the mutex that is already locked in a lower layer. You can check it in a runtime by assigning layer numbers to each mutex and keeping record of which mutex are locked by each thread.
* example:
```
hierarchical_mutex high_level_mutex(10000);// 10k is high level prio
hierarchical_mutex low_level_mutex(5000); //5k is low level prio

int do_low_level_stuff();

int low_level_func()
{
    std::lock_guard<hierarchical_mutex> lk(low_level_mutex);
    return do_low_level_stuff();
}


void do_high_level_stuff(int some_param);

int high_level_func()
{
    std::lock_guard<hierarchical_mutex> lk(high_level_mutex);
    do_high_level_stuff(low_level_func());
}

void thread_a()//is ok, from the pov of hierarchy
{
    high_level_func();
}

hierarchical_mutex other_mutex(100); // is ultra low level prio

void do_other_stuff();

void other_stuff()
{
    high_level_func(); //call high prio function? but the prio of other_stuff is very low. It's wrong.!!!
    // the hierarchical_mutex will throw exception
    do_other_stuff();
}
void thread_b() // disregards the rule
{
    std::lock_guard<hierarchical_mutex> lk(other_mutex);
    other_stuff();
}
```
* deadlock between hierarchical_mutex are impossible, because the mutexes enforce ordering
* you can't hold two locks at the same time if they are not of the same level.
* for std::lock_guard<> usability the user defined hierarchical_mutex should implement lock(), unlock(), try_lock();
* try_lock() if the lock is hold by another thread will return false and will not wait for the another thread to unlock the mutex.
```
class hierarchical_mutex
{
    std::mutex internal_mutex;
    unsigned long const hierarchy_value;
    unsigned long previous_hierarchy_value;
    static thread_local unsigned long this_thread_hierarchy_value;//this is important to have this value that represents the current thread value hierarchy
    
    void check_for_hierarchy_violation()
    {
        if(this_thread_hierarchy_value <= hierarchy_value)//passes successfully for the first time because this_value is ULONG_MAX
        {
            throw std::logic_error("mutex hierarchy was violated");
        }
    }
    
    void update_hierarchy_value()
    {
        previous_hierarchy_value=this_thread_hierarchy_value;
        this_thread_hierarchy_value=hierarchy_value;
    }
public:
    explicit hierarchical_mutex(unsigned long value):
        hierarchy_value(value), 
        previous_hierarchy_value(0)
    {}
    
    void lock()
    {
        check_for_hierarchy_violation();
        internal_mutex.lock();// with this check, lock delegates to the internal mutex for the actual locking
        update_hierarchy_value();
    }
    
    void unlock()
    {
        this_thread_hierarchy_value=previous_hierarchy_value;
        internal_mutex.unlock();
    }
    
    bool try_unlock()
    {
        check_for_internal_violation();
        if(!internal_mutex.try_lock())
            return false;
        update_hierarchy_value();
        return true;
    }
};

thread_local unsigned long hierarchical_mutex::this_thread_hierachical_value(ULONG_MAX); //hierarchy is initialized to max value, so initially any mutex can be locked
//every thread has its own copy, cause it is thread_local

```
### std::unique_lock
* more flexible substitute for std::lock_guard() is std::unique_lock, which is RAII as well
