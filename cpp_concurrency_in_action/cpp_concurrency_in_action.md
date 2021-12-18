# 1.Hello, world of concurrency in C++

### task switching on single processor system

### concurrency with multiple processes

- divide application into multiple, separate single-threaded processes that are run at the same time.
- Problem is here to communicate between processes - OS sets some restrictions, so it is slow and complicated.
- From another hand it is easier to write concurrent code with processes. You can then move some processes to other machine.
- c++ standard is does not provide about inter-process communication. hence it is platform specific

### concurrency with multiple threads

- because of shared address space and lack of protection of data between threads makes overhead of communication smaller(then in case of processes)
- harder to write safe code
- c++ standard is only about inter-thread communication

### why concurrency

- separation of concerns
- performance

### separation of concerns

- split ui from logic

### performance

- process data by chunks

### platform-specific facilities

- native_handle()

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

- can be launched, can be one to wait it to finish, or one to have it in background

### argument

- requires smth executable

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

- be aware with the syntax

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

- thread join -> waiting for its end
- thread detach -> leave it run on its own
- you need to be sure that the thread is correctly joined/detached before object lifetime ends

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

- better to copy the data into the thread rather than sharing the data
- check if objects are not containing pointers and references
- bad idea to create thread within the function that has access to local variables, unless thread finishes earlier than function.

### Waiting for the thread to complete

- join() is bruteforce, if you need more control over thread use condition variables, futures
- calling join() cleans up storage associated with thread so the std::thread object is no longer associated with the now finished thread.
- calling join() makes object no longer joinable,
- use joinable()

### Waiting in exceptional circumstances

- be sure that join() or detach() was called before thread object is destroyed
- problem is when exception between thread started and join() called
- make sense to cover exception case with join()
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

- calling detach() on the std::thread leaves thread to run in backgound, no direct means of communication and can't wait for thread to complete.

```
std::thread t(do_background_task());
t.detach();
assert(!t.joinable());
```

- can't join() after detach()
- if std::thread is detached, it is impossible to obtain std::thread object
- detached threads are called daemon threads
- can't call detach from a std::thread with no associated thread of execution, same for join. I.e. only joinable() threads are detachable.
- example word editing app, each new document means new thread

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

- even though second parameter is std::string, it is passed as char const \*, and is converted to std::string in context of new thread

```
void f(int i, std::string const & s);
std::thread t(f, 3, "3");
```

- risky moment about passing pointer to automatic variable

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

- vice verse, object is copied, but you wanted a pass via reference

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

- passing a member function point as the function

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

- use case: function creates a background thread but passes back ownership of the new thread to the calling function rather than waiting for it to complete.
- use std::move of std::thread, thread is only moveable, not copyable

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

- at some point of time of work on the data structures the invariant can be broken, it's better another threads do not access to such intermediate data.
- e.g. delete a node in a list 1) identify node to delete, 2) change link in previous node to point to the next one 3) change link in next node to point to the previous node 4) delete the node. At point 2-3 we have broken invariants.
- race condition is anything where outcome depends on the relative ordering of execution of operations on two or more threads.

### avoiding race conditions

- wrap your code with a protection mechanism to ensure that only the thread actually performing a modification can see the intermediate stats where the invariants are broken.
- lock-free programming, modification of the data structure/invariant design so that modifications are done as a series of indivisible changes.
- transaction, modifications of the data structure are performed withing one operation called transaction. There may be transaction log and one step commit. Software transactional memory(STM). C++ doesn't support it, at least in C++20.

### protecting data with mutexes

- the thread library ensures that once the one thread locked a specific mutex, all other threads that try to lock the same mutex have to wait for the unlock.
- mutexes can cause deadlock
- the syntax is: create instance of std::mutex, and call it's member function lock(), at the end call unlock()
- the RAII solution is to use std::lock_guard

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

- the previous example will not work properly if the list would be passed via reference/pointer. Any code that has an access to that pointer or reference can access/modify the protected data without locking the mutex.

### structuring code for protecting shared data

- check that none of the member functions return a pointer or reference to the protected data
- check that none of the member functions are passing pointers or references to the 'third party' functions.
- bad use case, it is possible to have written access to the protection data which is not mutually exclusive

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

- even having all methods covered with mutex it would not help with the deleting of the node in double linked list, as generally you need to modify 3 nodes. it is necessary to cover with mutex the whole list.
- example with stack, we can call push(), pop(), top(), empty(), size();

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

- in particular if stack instance is not shared, it's safe to check for empty() and then call top(), or top() and then pop()

```
stack<int> s;
if(!s.empty())
{
    int const value = s.top();//calling top on empty container is UB
    s.pop();
    do_something(value);
}
```

- the problem is related to the interface, need to change the interface design
- option 1: pass in a reference. requires constructing object of stack, may be resource hungry. requires type to be assignable.

```
std::vector<int> result;
some_stack.pop(result);
```

- option 2: require a no-throw copy constructor, or move constructor. Use std::is_nothrow_copy_constructible, std::is_nothrow_move_constructible.
- option 3: return a point to the popped item.pointers can be copied without throwing. but requires a means of managing the memory allocated and adds some overhead for simple types like int. Good to use std::shared_ptr, is destroyed once the last pointer is destroyed.
- option 4: option1 + option2 or option1 + option3
- example of thread safe stack data structure

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

- the stack is not assignable, no assignment operator and no swap().
- the stack is copyable
- the problem with locking schemes, sometimes you need more than one mutex.but that may lead to deadlocks.

### deadlock: the problem and solution

- deadlock example: there are two mutexes. Both mutexes need to be locked to perform the operation. There are two threads, each of them hold one mutex only. Both of the threads are waiting another to unlock its mutex.
- the common advice is: always lock the two mutexes in the same order. Sometimes it is straightforward, when both mutexes are serving different purposes. Sometimes it is not, when mutexes are protecting a separate instance of the same class.
- solution for taking care of pair of mutexes is to use std::lock. It can lock two or more mutexes at once without a risk of deadlock.

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

- the std::lock doesn't help when they are locked separatly
- the most frequent reason of the deadlock is related to locks. But it is not the only reason.
- it is possible to deadlock with two threads without any explicit lock by calling join() on the std::thread for the other. In that case neither thread can progress as it wits for the another thread.
- the solution is to implement the idea, don't wait for a thread if it may wait for your thread

### avoiding nested deadlocks

- don't acquire lock if you already hold one, if you need several locks use std::lock.

### avoiding calling user supplied code while holding a lock

- that code can acquire locks, hence we may have a problem with nested deadlocks

### acquire locks in a fixed order

- if you need to acquire several locks, so then use std::locks to preserve the order of locking
- in some cases it is not so straightforward, e.g. working on list. When you need to access the list, threads must acquire three nodes 1) the one to delete, 2) node before 3) node after. But if two threads are traversing same list in reverse orders - we can have a deadlock, cause the nodes maybe locked in different orders. thread 1 will lock N and N+1, while thread 2 will lock N+1 and N. The possible solution in this example to define the order of traversal.

### use a lock hierarchy

- it is a particular case of defining lock ordering.
- the idea is that you divide the application into layers and identify all the mutexes that may be locked in any given layer. Code will not be permitted to lock the mutex that is already locked in a lower layer. You can check it in a runtime by assigning layer numbers to each mutex and keeping record of which mutex are locked by each thread.
- example:

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

- deadlock between hierarchical_mutex are impossible, because the mutexes enforce ordering
- you can't hold two locks at the same time if they are not of the same level.
- for std::lock_guard<> usability the user defined hierarchical_mutex should implement lock(), unlock(), try_lock();
- try_lock() if the lock is hold by another thread will return false and will not wait for the another thread to unlock the mutex.

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

- more flexible substitute for std::lock_guard() is std::unique_lock, which is RAII as well
- doesn't always own the mutex it is associated with
- can take as a second argument std::adopt_lock to have the lock object manage the lock on a mutex
- can take as a second argument std::defer_lock to indicate that mutex should be unlocked on construction, the lock can be acquired later, by calling lock()

```
class some big_object;
void swap(some_big_object& lhs, some_big_object& rhs);

class X
{
    private:
        some_big_object some_detail;
        std::mutex m;
    public:
        X(some_big_object const& sd): some_detail(sd){}

        friend void swap(X& lhs, X &rhs)
        {
            if(&lhs == &rhs)
                return;

            std::unique_lock<std::mutex> lock_a(lhs.m, std::defer_lock);//std::defer_lock leaves mutexts unlocked
            std::unique_lock<std::mutex> lock_b(lhs.m, std::defer_lock);
            std::lock(lock a, lock b); // lock_a/b can be passed cause they provide lock(), try_lock(), unlock()
            swap(lhs.some_detail, rhs.some_detail);
        }
}
```

- calling lock() on lock_a, lock_b just calls the member function of the same name. And also they update the flag inside the std::unique_lock instance to indicate that the mutex is currently owned. If it is owned by the instance, the desctructor must call unlock(), and if the instance doesn't own the mutex there should be no call of unlock() in the destructor.
- there is a slight performance penalty of using the std::unique_lock over std:lock_guard

### transfering mutex ownership between scopes

- std::unique_lock do not have to own their mutexes, the ownership of a mutex can be transfered between instances by moving the instances around
- in some cases such transfer is automatic(return from function), in some cases explicitly via std::move
- if value is lvalue(real variable or reference, rvalue a temporary value) - ownership transfer is automatic if rvalue, or must be done explicitly if lvalue
- std::unique_lock is movable, but no copyable
- one of the usecase: function locks mutex and transfers ownership to the caller

```
std::unique_lock<std::mutex> get_lock()
{
    extern std::mutex some_mutex;
    std::unique_lock<std::mutex> lk(some_mutex);
    prepare_data();
    return lk;
}

void process_data()
{
    std::unique_lock<std::mutex> lk(get_lock());
    do_smth();
}

```

### lock granularity

- bad idea to hold lock on time consuming operations like IO.
- holding lock on a data for too long will cause bad performance

```
void get_and_process_data()
{
    std::unique_lock<std::mutex> my_lock(the_mutex);
    some_class data_to_process=get_next_data_chunk();
    my_lock.unlock();
    result_type result = process(data_to_process);
    my_lock.lock();
    write_result(data_to_process, result);
}
```

- In general, a lock should be held for only the minimum possible time needed to perform the required operations.
- This also means that time-consuming operations such as acquiring another lock (even if you know it won’t dead-lock) or waiting for I/O to complete shouldn’t be done under lock, unless it is required

```

class Y
{
private:
    int some_detail;
    mutable std::mutex m;

    int get_detail() const
    {
        std::lock_guard<std::mutex> lock_a(m);
        return some_detail;
    }


public:
    Y(int sd) : some_detail(sd){};

    friend bool operator==(Y const& lhs, Y const& rhs)
    {
        if (&lhs == &rhs)
        {
            return true;
        }
        int const lhs_value = lhs.get_detail();
        int const rhs_value = rhs.get_detail();
        return lhs_value==rhs_value;//as only one lock is hold at a time, we are comparing values that are not guaranteed to be unchanged in between.
    }
}
```

## alternative facilities for protecting shared data

- it's not only mutexes :)
- shared data needs protection only on initialization(e.g. is read only when created)

### protecting shared data on init

- lazy init (each operation that requires the resource first checks to see if it has been initialized and then initializes it before use if not)

```
std::shared_ptr<some_resource> resource_ptr;

void foo()
{
    if (!resource_ptr)
    {
        resource_ptr.reset(new some_resource);
    }
    resource_ptr->do_smth();
}
```

- if shared resource is safe for concurrent access, the only thing to redo is init, this solution is not optimal as it requires serialization of all threads

```
std::shared_ptr<some_resource> resource_ptr;
std::mutex resorce_mutex;

void foo()
{
    std::unique_lock<std::mutex> lk(resource_mutex); // all threads are serialized here
    if (!resource_ptr)
    {
        resource_ptr.reset(new some_resource);
    }
    lk.unlock(); // only init is protected
    resource_ptr->do_smth();
}
```

- one not good solution is Double-Checked Locking pattern. the pointer is first read without acquiring the lock B (in the code below), and the lock is acquired only if the pointer is NULL. The pointer is then checked again once the lock has been acquired c (hence the double-checked part) in case another thread has done the initialization between the first check and this thread acquiring the lock

```
void undefined_behaviour_with_double_checked_locking()
{
    if(!resource_ptr) //is not syncronized with the write done by other thread
    {
        std::lock_guard<std::mutex> lk(resource_mutex);
        {
            if(!resoource_ptr)
            {
                resource_ptr.reset(new some_resorce);
            }
        }
        resource_ptr->do_smth();
    }
}
```

- it contains a race conditiion the thread may see the pointer written by another thread but not see the instance of new some_resource, do_smth() maybe called on wrong data
- as a solution it was added in c++ std::once_flag, std::call_once, instead of locking the mutexes and explicitly checking the pointer, every threead can just use std::call_once, the pointer would be initialized by some thread when call_once returns. Call_once has a lower overhead that using mutexes explicitly.

```
std::shared_ptr<some_resource> resource_ptr;
std::once_flag resource_flag;

void init_resource()
{
    resource_ptr.reset(new some_resource);
}
void foo()
{
    std::call_once(resource_flag, init_resource); //inti is called only once
    resource_ptr->do_smth();
}
```

- thread safe initialization of a class memeber using std::call_once

```
class X
{
    private:
     connection_info connection_details;
     connection_handle connection;
     std::once_flag connection_init_flag;

     void open_connection()
     {
         connection - connection_manager.open(connection_details);
     }

public:
 X(connection_info const & connection_details_):
 connection_details(connection_details_)
 {}

 void send_data(data_packet const& data)
 {
     std::call_once(connection_init_flag, &X::open_connection, this);
     connection.send_data(data);
 }

 data_packet receive_data()
 {
     std::call_once(connection_init_flag, &X::open_connection, this);
     return connection.receive_data();
 }
}
```

- here init would be done either in send_data, or receive_data.
- before c++11, when local variable is static. The init of such variable is defined to occure the first time control passes through its declaration. before c++11, some threads may already started to use the static variable, but before the first thread finished working with it. Multiple threads may belive that they are first.
- in c++11 the init will be happended on one thread, and no other threads would proceeed until init is complete.
- alternative to std::call_once

```
class my_class;
my_class& get_my_class_intstance()
{
    static my_class instance;
    return instance; //the init is guaranteed to be thread safe
}
```

### protecting rarely updated data structures

- reader-writer mutex allows for two different kinds of usage: write by one thread, read by many.c++ out of box do not provide it. at least in c++11/14
- std::shared_mutex in c++17 or boost::shared_mutex

```
// may be used for locking in place of corresponding std::mutex
std::local_guard<std::shared_mutex>
std::unique_lock<std::shared_mutex>

//those threads that do not update data
std::shared_lock<std::shared_mutex>
```

- The only constraint is that if any thread has a shared lock, a thread that tries to acquire an exclusive lock will block until all other threads have relinquished their locks, and likewise if any thread has an exclusive lock, no other thread may acquire a shared or exclusive lock until the first thread has relinquished its lock.
- dns cache example(rare updatable structure

```
#include <map>
#include <string>
#include <mutex>
#include <shared_mutex>

class dns_entry;

class dns_cache
{
    std::map<std::string, dns_entry> entries;
    mutable std::shared_mutex entry_mutex;
public:
    dns_entry find_entry(std::string const & domain) const
    {
        std::shared_lock<std::shared_mutex> lk(entry_mutex);//aka read only access
        std::map<std::string, dns_entry> :: const_interator const it = entries.find(domain);
        return (it==entries.end())?dns_entry():it->second;
    }

    void update_or_add_entry(std::string const &domain, dns_entry const & dns_details)
    {
        std::lock_guard<std::shared_mutex> lk(entry_mutex; //aka write access
        entries[domain] = dns_details;
    }
};
```

### recursive locking

- With std::mutex, it’s an error for a thread to try to lock a mutex it already owns, and attempting to do so will result in undefined behavior.
- but that is possible with std::recursive_mutex
- It works just like std::mutex, except that you can acquire multiple locks on a single instance from the same thread.
- You must release all your locks before the mutex can be locked by another thread, so if you call lock() three times, you must also call unlock() three times. Correct use of std::lock_guard <std::recursive_mutex> and std::unique_lock<std::recursive_mutex> will handle this for you.
- usage is not recommended

# Synchronizing concurrent operations

- sometimes need not only to protect data, but to synchronize actions between threads
- for synchronization we are using condition variables and futures

### Waiting for an event or other condition

- if one thread is waiting for a second thread to complete a task, it has several options.
- First, it could just keep checking a flag in shared data (protected by a mutex) and have the second thread set the flag when it completes the task. This is wasteful on two counts: the thread consumes valuable processing time repeatedly checking the flag, and when the mutex is locked by the waiting thread, it can’t be locked by any other thread.
- A second option is to have the waiting thread sleep for small periods between the
  checks using the std::this_thread::sleep_for() function

```
bool flag;
std::mutex m;
void wait_for_flag()
{
    std::unique_lock<std::mutex> lk(m);
    while(!flag)
    {
        lk.unlock();
        std::this_thread::sleep_for(std;:chrono::milliseconds(100));
        lk.lock();
    }
}
```

it is not effective, too short sleep may need a lot of wakeups, too long sleep may notify about the thing finished too lately.
\*The third, and preferred, option is to use the facilities from the C++ Standard Library to wait for the event itself. The most basic mechanism is about waiting for an event to be triggered by another thread is the condition variable. When sme thread has determined that the conditiion is satisfied, it can then notify one or more of the threads waiting on the conditiion.

### Waiting for a condition with condition variables

- std::condition_variable, std::condition_variable_any from <condition_variable> library.
- std::condition_variable, works with mutex -> prefered solution.
- std::condition_variable_any, works with what is a mutex-like, gives some sort of flexibility
- example on condition_variable:

```
std::mutex mut;
std::queue<data_chunk> data_queue; // a queue to pass data between threads
std::conditiion_variable data_cond;


void data_preparation_thread()
{
    while(more_data_to_prepare)
    {
        data_chunk const data = prepare_data();
        std::lock_guard<std::mutex> lk(mut);
        data_queue.push(data);
        data_cond.notify_one();
    }
}

void data_processing_thread>()
{
    while(true)
    {
        std::unique_lock<std::mutex> lk(mut);
        data_cond.wait(
            lk, []{return !data_queue.empty();});
            data_chunk data=data_queue.front();
            data_queue.pop();
            lk.unlock();
            process(data);
            if(is_last_chunk(data))
            {
                break;
            }
    }
}
```

### Building a thread-safe queue with condition variables

- operations requireed for the queue. inspired by the std::string
- this is non thread safe

```
template <class T, classs Container = std::deque<T> >
class queue{
public:
    explicit queue(const Container&);
    explicit queue(Container&& = Container());
    template <class Alloc> explicit queue(const Alloc&);
    template <class Alloc> queue(const Container&, const Alloc&);
    template <class Alloc> queue(Container&&, const Alloc&);
    template <class Alloc> queue(queue&&, const Alloc&);
    void swap(queue& q);
    bool empty() const;
    size_type size() const;
    T& front();
    const T& front() const;
    T& back();
    const T& back() const;
    void push(const T& x);
    void push(T&& x);
    void pop();
    template <class... Args> void emplace(Args&&... args);
};
```

- here we have same stuff as some time before, front and pop should not be separate
- threadsafe queue example

```
template<typename T>
class threadsafe_queue
{
private:
    mutable std::mutex mut; //mutex locking is a mutating operation, so in order const method to lock, this mutex is marked as mutable
    std::queue<T> data_queue;
    std::condition_variable data_cond;
public:
    threadsafe_queue(){};

    threadsafe_queue(threadsafe_queue const& other)
    {
        std::lock_guard<std::mutex> lk(other.mut);
        data_queue=other.data_queue;
    }

    threadsafe_queue& operator= (const threadsafe_queue&) = delete;

    void push(T new_value)
    {
        std::lock_guard<std::mutex> lk(mut);
        data_queue.push(new_value);
        data_cond.notify_one();
    }

    bool try_pop(T& value)
    {
        std::lock_guard<std::mutex> lk(mut);
        if(data_queue.empty())
        {
            return false;
        }
        value=data_queue.front();
        data_queue.pop();
        return true;
    }

    std::shared_ptr<T> try_pop()
    {
        std::lock_guard<std::mutex> lk(mut);
        if(data_queue.empty())
        {
            return std::shared_ptr<T>();
        }
        std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
        data_queue.pop();
        return res;
    }

    void wait_and_pop(T& value);
    {
        std::unique_lock<std::mutex> lk(mut);
        data_cond.wait(lk,[this]{return !data_queue.empty();});
        value=data_queue.front();
        data_queue.pop();
    }

    std::shared_ptr<T> wait_and_pop()
    {
        std::unique_lock<std::mutex> lk(mut);
        data_cond.wait(lk,[this]{return !data_queue.empty();});
        std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
        data_queue.pop();
        return res;
    }

    bool empty() const
    {
        std::lock_guard<std::mutex> lk(mut);
        return data_queue.empty();
    }
};

// usage
threadsafe_queue<data_chunk> data_queue;
 void data_preparation_thread()
{
    while(more_data_to_prepare())
    {
        data_chunk const data=prepare_data();
        data_queue.push(data);
    }
}


void data_processing_thread()
{
    while(true)
    {
        data_chunk data;
        data_queue.wait_and_pop(data);
        process(data);
        if(is_last_chunk(data))
        {
            break;
        }
    }
}
```

- Condition variables are also useful where there’s more than one thread waiting for the same event.

### Waiting for one-off events with futures

- There are two sorts of futures in the C++ Standard Library, implemented as two class templates declared in the <future> library header
- unique futures (std::future<>) . An instance of std::future is the one and only instance that refers to its associated event.
- shared futures (std::shared_future<>). multiple instances of std::shared_future may refer to the same event.
- The std:future<void>,std::shared_future<void> template specializations should be used where there’s no associated data.

### Returning values from background tasks

- You use std::async to start an asynchronous task for which you don’t need the result right away. Rather than giving you back a std::thread object to wait on,
  std::async returns a std::future object, which will eventually hold the return value of the function. When you need the value, you just call get() on the future, and the thread blocks until the future is ready and then returns the value.

```
#include <future>
#include <iostream>

int find_the_answer_to_ltuae();
void do_other_stuff();
int main()
{
    std::future<int> the_answer = std::async(find_the_answer_to_ltuae);
    do_other_stuff();
    std::cout << "the answer is " << the_answer.get() << std::endl;
}
```

- By default, it’s up to the implementation whether std::async starts a new thread, or whether the task runs synchronously when the future is waited for.
  This can be controlled by the parameter std::lauch.
- the parameter can be std::launch::deferred to indicate the function to be deferred untill wait(), get() is called on the future
- std::launch::async to indicate that the function must be run on its own thread

```
auto f6=std::async(std::launch::async,Y(),1.2);
auto f7=std::async(std::launch::deferred,baz,std::ref(x));
auto f8=std::async(std::launch::deferred | std::launch::async,baz,std::ref(x));
auto f9=std::async(baz,std::ref(x));
f7.wait();
```

### Associating a task with a future

- std::packaged_task<> ties a future to a function or callable object. When the std:: packaged_task<> object is invoked, it calls the associated function or callable object and makes the future ready, with the return value stored as the associated data.

### PASSING TASKS BETWEEN THREADS

```
#include
#include
#include
#include
#include
<deque>
<mutex>
<future>
<thread>
<utility>
std::mutex m;
std::deque<std::packaged_task<void()> > tasks;
boolvoidgui_shutdown_message_received();
get_and_process_gui_message();

void gui_thread()
{
    while(!gui_shutdown_message_received())
    {
        get_and_process_gui_message();
        std::packaged_task<void()> task;
        {
            std::lock_guard<std::mutex> lk(m);
            if(tasks.empty())
                continue;
            task=std::move(tasks.front());
            tasks.pop_front();
        }
        task();
    }
}
```

- the GUI thread loops until a message has been received telling the GUI to shut down, repeatedly polling for GUI messages to handle, such as user clicks, and for tasks on the task queue. If there are no tasks on the queue, it loops again; otherwise, it extracts the task from the queue, releases the lock on the queue, and then runs the task g. The future associated with the task will then be made ready when the task completes.

### Making (std::)promises

- std::promise<T> provides a means of setting a value (of type T), which can later be read through an associated std::future<T> object.

```
#include <future>
void process_connections(connection_set& connections)
{
    while(!done(connections))
    {
        for(connection_iterator connection=connections.begin(),end=connections.end();connection!=end;++connection)
        {
        if(connection->has_incoming_data())
        {
            data_packet data=connection->incoming();
            std::promise<payload_type>& p= connection->get_promise(data.id); p.set_value(data.payload); }
            if(connection->has_outgoing_data())
            {
                outgoing_packet data=
                connection->top_of_outgoing_queue();
                connection->send(data.payload);
                data.promise.set_value(true);

            }
        }
    }
}
```

### Saving an exception for the future

```
double square_root(double x)
{
    if(x<0)
    {
        throw std::out_of_range(“x<0”);
    }
    return sqrt(x);
}

std::future<double> f=std::async(square_root,-1);
double y=f.get();
```

- if the function call invoked as part of std::async throws an exception, that exception is stored in the future in place of a stored value, the future becomes ready, and a call to get() rethrows that stored exception.
- Note: the standard leaves it unspecified whether it is the original exception object that’s rethrown or a copy;
- Naturally, std::promise provides the same facility, with an explicit function call. If you wish to store an exception rather than a value, you call the set_exception() member function rather than set_value().

```
extern std::promise<double> some_promise;
try
{
    some_promise.set_value(calculate_value());
}
catch(...)
{
    some_promise.set_exception(std::current_exception());
}
```

- This uses std::current_exception() to retrieve the thrown exception; the alternative here would be to use std::copy_exception() to store a new exception directly without throwing:

```
some_promise.set_exception(std::copy_exception(std::logic_error("foo ")));
```
* If you need to wait for the same event from more than one thread, you need to use std::shared_future

### Waiting from multiple threads
* If you access a single std::future object from multiple threads without additional synchronization, you have a data race and undefined behavior. 

### Waiting with a time limit
* There are two sorts of timeouts you may wish to specify: a duration-based timeout, where you wait for a specific amount of time (for example, 30 milliseconds), or an absolute timeout, where you wait until a specific point in time (for example, 17:30:15.045987023 UTC on November 30, 2011)
* std::condition_variable has two overloads of the wait_for() and two overloads of the wait_until()
* current time is: std::chrono::system_clock::now()
* If a clock ticks at a uniform rate (whether or not that rate matches the period) and can’t be adjusted, the clock is said to be a steady clock.
* The is_steady static data member of the clock class is true if the clock is steady and false otherwise.
* std::chrono::system_clock will not be steady, because the clock can be adjusted

### Durations
* Durations are the simplest part of the time support; they’re handled by the std::chrono::duration<> class template
* The first template parameter is the type of the representation (such as int, long, or double)
* the second is a fraction specifying how many seconds each unit of the duration represents
* Explicit conversions can be done with std::chrono::duration_cast<>
* example of wait basing on duration
```
std::future<int> f = std::async(some_task);
if(f.wait_for(std::chrono::milliseconds(35)) == std::future_status::ready )
{
    do_something_with(f.get());
}
```

### Time points
* The time point for a clock is represented by an instance of the std::chrono::time_point<> class template, which specifies which clock it refers to as the first template parameter and the units of measurement (a specialization of std::chrono::duration<>) as the second template parameter. 
* The value of a time point is the length of time (in multiples of the specified duration) since a specific point in time called the epoch of the clock.
* Typical epochs include 00:00 on January 1, 1970and the instant when the computer running the application booted up
* Although you can’t find out when the epoch is, you can get the time_since_epoch() for a given time_point.
* You can also subtract one time point from another that shares the same clock. The result is a duration specifying the length of time between the two time points. 
```
auto start  = std::chrono::high_resolution_clock::now();
do_something();
auto stop = std::chrono::high_resolution_clock::now();
std::cout << "do_something() took :" << std::chrono::duration<double, std::chrono::seconds> (stop - start).count() << " seconds." << std::endl;
```

###  Waiting for a condition variable with a timeout
```
#include <condition_variable>
#include <mutex>
#include <chrono>

std::condition_variable cv;
bool done;
std::mutex m;

bool wait_loop()
{
    auto const timeout = std::chrono::steady_clock::now() + std::chrono::milliseconds(500);
    std::unique_lock<std::mutex> lk(m);
    while(!done)
    {
        if(cv.wait_until(lk, timeout) == std::cv_status::timeout)
            break;
    }
    return done;

}
```
* This is the recommended way to wait for condition variables with a time limit, if you’re not passing a predicate to the wait. 

### Functions that accept timeouts
* table of functions

|class/namespace|functions|return value  |
|---|---|---|
|std::this_thread  | sleep_for(duration), sleep_untill(time_point))   | N/A  |
|std::condition_variable, std::condition_variable_any   | wait_for(lock, duration), wait_until(lock, time_point)   | std::cv_status::timeout, std::cv_status::no_timeout |
|   |wait_for(lock,duration,predicate)wait_until(lock,time_point,predicate)   | bool |
|std::timed_mutex,std::recursive_timed_mutex|try_lock_for(duration)try_lock_until(time_point)|bool|
|std::unique_lock<TimedLockable>|unique_lock(lockable,duration)unique_lock(lockable,time_point)|   |
|   |try_lock_for(duration)try_lock_until(time_point)|bool—true if the lock was acquired, false otherwise|
|std::future<ValueType> orstd::shared_future<ValueType>|wait_for(duration)wait_until(time_point)|std::future_status::timeout if the wait timedout, std::future_status::ready if the future is ready|

* The simplest use for a timeout is to add a delay to the processing of a particular thread, so that it doesn’t take processing time away from other threads when it has nothing to do
*  std::this_thread::sleep_for() and std::this_thread::sleep_until()
*  std::timed_mutex supports timeouts
*  std::recursive_timed_mutex supports timeouts
*  std::try_lock_for() and try_lock_until() 
