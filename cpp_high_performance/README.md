# book C++ High Performance

# why c++?
* Zero-cost abstractions 
* kind of more comfortable than c
``` cpp
struct string_elem_t { const char* str_; string_elem_t* next_; };
int num_hamlet(string_elem_t* books) {
  const char* hamlet = "Hamlet";
  int n = 0;
  string_elem_t* b; 
  for (b = books; b != 0; b = b->next_)
    if (strcmp(b->str_, hamlet) == 0)
      ++n;
  return n;
}
```
``` cpp
int num_hamlet(const std::list<std::string>& books) {
  return std::count(books.begin(), books.end(), "Hamlet");
}
```  
even though both listings should be compiled to the same code
* may have pointers stuff hidden in implementation
* portability

* c++ problems: long compilation, lack of libs
## Error handling and resource acquisition
* how to handle use case when exception lead to half created object:
```
struct Leaf { /* ... */ };
struct Branch { /* ... */ };

class OakTree {
public:
  auto& operator=(const OakTree& other) {
    leafs_ = other.leafs_;
// If copying the branches throws, only the leafs has been copied and the OakTree is left in an invalid state
    branches_ = other.branches_; 
    *this;
  }
  std::vector<Leaf> leafs_;
  std::vector<Branch> branches_;
};

auto oaktree_func() {
  auto tree0 = OakTree{std::vector<Leaf>{1000}, std::vector<Branch>{100}};
  auto tree1 = OakTree{std::vector<Leaf>{50}, std::vector<Branch>{5}} 
  try {
    tree0 = tree1;
  } 
  catch(const std::exception& e) {
 // tree0 might be broken
    save_to_disk({tree0, tree1});
  }
}
```  
we need to use copy-and-swap technique
```
class OakTree {
public:
  auto& operator=(const OakTree& other) {
 // First create local copies without modifying the OakTree objects. Copying may throw, but this OakTree will still be in a valid state
    auto leafs = other.leafs_;       
    auto branches = other.branches_;

  // No exceptions thrown, we can now safely modify the state of this object by non-throwing swap
    std::swap(leads_, leafs);
    std::swap(branches_, branches);
    return *this;
  }
  std::vector<Leaf> leafs_;
  std::vector<Branch> branches_;
};
```
# Chapter 2. Modern C++ Concepts
## auto
* Automatic type deduction with the auto keyword 
```
const int& cref() const {
    return m_; 
  }
 vs 
 auto& cref() const {
    return m_; 
  }
```
examples  
return by value  
``` cpp
auto val() const                // a) auto, deduced type
auto val() const -> int         // b) auto with type
int val() const                 // c) explicite type
```  
return by const ref  
```
auto& cref() const              // a) auto, deduced type
auto cref() const -> const int& // b) auto, trailing type
const int& cref() const         // c) explicite type
```
return by mutable reference
```cpp
auto& mref()                    // a) auto, deduced type
auto mref() -> int&             // b) auto, trailing type
int& mref()                     // c) explicite type
```
* With copy elision guarantees introduced in C++17, the statement auto x = Foo{} is identical to Foo x{}.
* use auto& only when need mutable reference
* auto&& is called forwarding reference, can bind to everything, it will extend lifetime of temporary
* better always to use const auto& (auto & and auto for mutable, auto&& for forwarding)
## The lambda function
* in c++14 enhanced with polymorphic capabilities
* instead of lambdas there may be used classes with operator() overloaded
* lambda may be seen as  
 - class with only one member function  
 - the capture block is a combination of class's members and its constructor  
 - each lambda has its own unique type  
  e.g.  
  ``` cpp
auto th = 3;
auto is_above = [th](int v) {
 return v > th;
};
auto test = is_above(5);
// test equals true

vs 

auto th = 3;
class IsAbove {
public:
 IsAbove(int th) : th{th} {}
// The only member function
 auto operator()(int v)const{
   return v > th;
 }
private:
int th{}; // Members
};
auto is_above = IsAbove{th};
auto test = is_above(5);
// test equals true
  ```
 * lambda keyword mutable , works like inverse for const keyword(by default lambda is const)
 * capture all in scope
 ``` cpp
 [=] //by value
 [&] //by ref
 ```
 * capture all in scope
 ``` cpp
 [this]
 [*this]
 [=]
 [&]
 ```
 * mixed capture
 ```
 [=, &c]
 [&, c]
 ```
 * assigning c function pointers to lambda 
 ``` cpp
 external void download_webpage(const char* url, void (*callback)(int, const char));
 
auto func() 
{
  auto lambda = +[](int result, const char* str) {};
  download_webpage("http://www.packt.com", lambda);
}
```
### lambdas and std::function
* std function signature is
```cpp
std::function<return type (parameter0, parameter1...)>
e.g. for void and argumentless functions
``` cpp
auto func = std::function<void(void)>{};
```
```
e.g.
```cpp
auto func = std::function<bool(int, std::string)>{}; 
```
* assigning lambas to std::function
* std::function can hold any lambda and can be reassigned in runtime
* what is captured by lambda doesn't influence signature, both lambdas with and without captures can be assigned to std::function
```cpp
auto func = std::function<void(int)>{}; 
func = [](int v) { std::cout << v; }; 
func = [forty_two](int v) { std::cout << (v + forty_two); }; 
```

* performance of std::function is worthier than lambda's. Compiler can inline the lambda(std::function cannot be inlined)
* if lambda function with captures is assigned to std::function, std::function will store them on heap(usually)
* invoking std::function requires more operations that lambda
* the polymorphic lambda is a lambda accepting auto as parameter
 ```  cpp
 auto v = 3;
 auto lambda = [v](auto v0, auto v1){ return v + v0*v1; };
```
in case if we want instead of lambda a class
``` cpp
class Lambda {
public:
  Lambda(int v) : v_{v} {}
  template <typename T0, typename T1>
  auto operator()(T0 v0, T1 v1) const { return v_ + v0*v1; }
private:
  int v_{};
};
auto v = 3;
auto lambda = Lambda{v};
```
### const proppagation
* example of possiblity to change in const method
``` cpp
class Foo {
public:
  Foo(int* ptr) : ptr_{ptr} {} 
  auto set_ptr_val(int v) const { 
*ptr_ = v; // Compiles despite function being declared const!
  }
private:
  int* ptr_{};
};

auto main() -> int {
  const auto foo = Foo{};
  foo.set_ptr_val(42);
}
```
* there is an experimental solution
```
namespace exp = std::experimental;
class Foo { 
  auto set_ptr_val(int v) const { 
    *ptr_ = v; // Will not compile, const is propagated
  }
  exp::propagate_const<int*> ptr_ = nullptr; 
}; 
```
## Move semantics 

* rule of three: copy constructor, copy assignment, destructor
``` cpp
class Buffer { 
public: 
  // Constructor 
  Buffer(const std::initializer_list<float>& values)  
  : size_{values.size()} { 
    ptr_ = new float[values.size()]; 
    std::copy(values.begin(), values.end(), ptr_); 
  } 
  // 1. Copy constructor 
  Buffer(const Buffer& other) : size_{other.size_} { 
    ptr_ = new float[size_]; 
    std::copy(other.ptr_, other.ptr_ + size_, ptr_); 
  } 
  // 2. Copy assignment 
  auto& operator=(const Buffer& other) {
    delete [] ptr_;
    ptr_ = new float[other.size_];
    size_ = other.size_;
    std::copy(other.ptr_, other.ptr_ + size_, ptr_);
    return *this;
  } 
  // 3. Destructor 
  ~Buffer() { 
    delete [] ptr_; // Note, it is valid to delete a nullptr 
    ptr_ = nullptr;
  } 


private: 
  size_t size_{0}; 
  float* ptr_{nullptr};
};
```
* rule of five = rule of three + move constructor and move assignment
``` cpp
class Buffer { 
  ... 
  Buffer(Buffer&& other) noexcept 
  : ptr_{other.ptr_}
  , size_{other.size_} { 
    other.ptr_ = nullptr;
    other.size_ = 0;
  }
  auto& operator=(Buffer&& other) noexcept {
    ptr_ = other.ptr_;
    size_ = other.size_;
    other.ptr_ = nullptr;
    other.size_ = 0;
    return *this;
  }
  ...
};
``
when the compiler detects that we perform what seems to be a copy, such as returning a Buffer from a function, but the copied-from value isn't used anymore, it will utilize the no-throw move-constructor/move-assignment instead of copying.  
* if move operations are not marked noexcept STL and algos would use simple copy operations
* moved from object should be able to execute it's destructor or reassing to a new value
* how to get an r-value(it is returned by function or we could use std::move)
``` cpp
class Bird { 
public: 
  Bird() {} 
  auto set_song(const std::string& s) { song_ = s; } 
  auto set_song(std::string&& s) { song_ = std::move(s); } 
  std::string song_; 
}; 
auto bird = Bird{};
//move assgined as std::move is used
auto cuckoo_b = std::string{"I'm a Cuckoo"};
bird.set_song(std::move(cuckoo_b));
//move assigned as string comes from function
auto make_roast_song() { return std::string{"I'm a Roast"}; } 
bird.set_song(make_roast_song());  
```
* if you do not declare any custom copy-constructor/copy-assignment or destructor, the move-constructors/move-assignments are implicitly declared
* Writing your classes so that they do not require any explicitly written copy-constructor, copy-assignment, move-constructor, move-assignment, or destructor is often referred to as the rule of zero
* better not declare an empty destructor, as it prevents the compiler from implementing certain optimizations. 
* A common pitfall - moving non-resources
``` cpp
class TowerList { 
public: 
  TowerList() : max_height_idx_{1}, tower_heights_{25.0f, 44.0f, 12.0f} {}
  auto get_max_tower_height() const { 
    return max_height_idx_ >= 0 ? 
      tower_heights_[max_height_idx_] : 0.0f; 
  } 
  std::vector<float> tower_heights_{}; 
  int max_height_idx_{-1}; 
};

auto a = TowerList{}; 
auto b = std::move(a); 
auto max_height = a.get_max_tower_height(); //it will fail here as vector is empty, but max_height is not
``` 
the solution is to implement move constructr and assignment by swapping  
```cpp
TowerList(TowerList&& tl) noexcept { 
  std::swap(tower_heights_, tl.tower_heights_); 
  std::swap(max_height_idx_, tl.max_height_idx_); 
} 
auto& operator=(TowerList&& tl) noexcept { 
  std::swap(tower_heights_, tl.tower_heights_); 
  std::swap(max_height_idx_, tl.max_height_idx_); 
  return *this; 
}
``` 
* you can add && to member function, just like a const control word
``` cpp
struct Foo { 
  auto func() && {} //it is executed only on r-values
}; 
```
## Representing optional values with std::optional
* part of C++17, is a wrapper to any tyoe where the wrapped type can be both initialized and uninitialized
* std::optional is a stack-allocated container with a max size of one.
* boost equivalent is boost::optional
* Before the introduction of std::optional, there was no clear way to define functions which may not return a defined value, such as the intersection point of two line segments
``` cpp
// Prerequisite
class Point {...}; class Line {...};  
external auto is_intersecting(Line a, Line b) -> bool {...}
external auto get_intersection(Line a, Line b) -> Point {...}

auto get_intersection_point(const Line& a, const Line& b) 
-> std::optional<Point> {
  return is_intersection(a, b) ?
    std::make_optional(get_intersection(a, b)):
    std::optional<Point>{};
}

// Prerequisite
auto line0 = Line{...}; 
auto line1 = Line{...};
external auto set_magic_point(Point p);

// Get optional intersection
auto intersection = get_intersection_point(line0, line1);
if(intersection.has_value()) {
// std::optional throws an exception if intersection is empty
  set_magic_point(*intersection);
}
```
* The object held by a  std::optional is always stack allocated, and the memory overhead of a std::optional<T> compared to T is  the size of one bool (usually one byte), plus possible padding.
* Two empty optional's are considered equal.
* An empty optional is considered less than a non-empty.
  ##  Representing dynamic values with std::any 
  * Just like std::optional, std::any can store an optional single value, but with the difference that it can store any type at runtime, just like a dynamically typed language. 

``` cpp
// Initialize an empty std::any
auto a = std::any{}; 
// Put a string in it
a = std::string{"something"}; 
// Return a reference to the withheld string
auto& str_ref = std::any_cast<std::string&>(a); 
// Copy the withheld string
auto str_copy = std::any_cast<std::string>(a); 
// Put a float in the 'any' and read it back
a = 135.246f; 
auto flt = std::any_cast<float>(a); 
// Copy the any object
auto b = a; 
auto is_same = 
  (a.type() == b.type()) && 
  (std::any_cast<float>(a) == std::any_cast<float>(b)); 
// a equals b as they contain the same type and value
```
  
 * check what type is withheld
``` cpp
template <typename T> 
auto is_withheld_type(const std::any& a) -> bool { 
  return typeid(T) == a.type(); 
} 

auto a = std::any{32.0};
auto is_int = is_withheld_type<int>(a); 
// is_int is false, 'a' contains a double
  ```
 * std::any heap-allocates its withheld value (although implementers are encouraged to store small objects inside of the any)
 * The Boost equivalent of std::any, boost::any, provides a fast version of std::any_cast called boost::any_cast_unsafe which can be utilized if you know which type is contained. In contrast to std::any_cast, using a boost::any_cast_unsafe with the wrong type will result in undefined behavior instead of a thrown exception.

# Chapter 3. Measuring Performance
## Asymptotic complexity and big O notation
* asymptotic computational complexity—that is, analyzing how the running time or memory consumption grows when the size of the input increases
* In order to find out how algorithm behaves when passed different sizes of the array, we would like to analyze the running time of this algorithm as a function of its input size
* If f(n) is a function that specifies the running time of an algorithm with input size n, we say that f(n) is O(g(n)) if there is a constant k such that f(n) ≤ k * g(n).

| f(n)       | Name                    | n = 10    | n = 50        | n = 1000     |
|------------|-------------------------|-----------|---------------|--------------|
| O(1)       | Constant                | 0.001 sec | 0.001 sec     | 0.001 sec    |
| O(log(n))  | Logarithmic             | 0.003 sec | 0.006 sec     | 0.01 sec     |
| O(n)       | Linear                  | 0.01 sec  | 0.05 sec      | 1 sec        |
| O(n log n) | Linearithmic or n log n | 0.03 sec  | 0.3 sec       | 10 sec       |
| O(n^2)     | Quadratic               | 0.1 sec   | 2.5 sec       | 16.7 minutes |
| O(2^n)     | Exponential             | 1 sec     | 357 centuries | Inf          |

### Amortized time complexity
* If an algorithm runs in constant amortized time, it means that it will run in O(1) in almost all cases, except a very few where it will perform worse.
* in case of push_back if in container there is some root -> operation takes constant time, i.e. doesn't depend on the size of data. If there is no room for element resize would require max n operations, i.e. Linear O(n)
* Amortized running time is used for analyzing a sequence of operations rather than a single one. 

### How to measure
* define a goal( ux response time - 100 ms,  graphics - 60 FPS (i.e. 16 ms), audio  128 sample buffer 44.1 hHz(i.e. 3 ms))
* measure 
* find the bottleneck
* make an educated guess to increase performace(lookup table, cache, less allocations)
* optimize
* evaluate
* refactor

### Performace properties
* Latency/response time (e.g. time between request/response)
* Throughput - number of transactions, operations, requests processed per time unit
* i/o bound or cpu bound. A task is said to be CPU bound if it would run faster if the CPU were faster. it's said to be I/O bound if it would run faster by making the I/O faster.
* Power consumption
* Aggregating data 

### performance testing - best practices
* measure not only functionality regression but performance too, i.e. start measuring performance from the project start
* plot your data, to see patterns, outliers
* use real data for performance testing, not some minimal size input

### profilers 
* analyze execution of the program and output a statistical summary, a profile, how often function was call, output call graph
* two types of profilers: sampling and instrumentation profiler(and actually mixed - example gprof)
* gprof https://www.thegeekstuff.com/2012/08/gprof-tutorial/
* Instrumentation profiler: means inserting code into the program to show how frequently each function is called. impacts performance, may prevent compiler optimization. 
* hand made instr profiler:
```cpp
class ScopedTimer { 
public: 
  using ClockType = std::chrono::steady_clock; 

  ScopedTimer(const char* func) : 
    function_{func},  
    start_{ClockType::now()} { }
 
  ScopedTimer(const ScopedTimer&) = delete; 
  ScopedTimer(ScopedTimer&&) = delete; 
  auto& operator=(const ScopedTimer&) -> ScopedTimer& = delete; 
  auto& operator=(ScopedTimer&&) -> ScopedTimer& = delete; 

  ~ScopedTimer() {
    using namespace std::chrono;
    auto stop = ClockType::now(); 
    auto duration = (stop - start_); 
    auto ms = duration_cast<milliseconds>(duration).count();  
    std::cout << ms << " ms " << function_ <<  '\n'; 
  } 
 
private: 
  const char* function_{}; 
  const ClockType::time_point start_{}; 
}; 

#if USE_TIMER 
  #define MEASURE_FUNCTION() ScopedTimer timer{__func__} 
#else 
  #define MEASURE_FUNCTION() 
#endif 

auto some_function() { 
  MEASURE_FUNCTION(); 
  ... 
} 

```
* Sampling profiles looks in program state at equal intervals(e.g. 10ms), less impact on the program, all optimization should be in place. Problem is inaccuracy and statistical approach(because of sampling some function can be active during each sample collection, but only during it)


### memory hierarchy
* currently caches l1,l2 are per core accessible, l3 is shared between all of the cores. 
* get info on hw
```
sysctl -a hw
```
* hw.memsize all main memory(ram)
* hw.cachelinesize, cache line size that is fetched when system access some byte. The various caches between the CPU and main memory keep track of 64 byte blocks instead of individual bytes.
* hw.l1icachesize is the size of the L1 instruction cache. This is a 32 KB cache dedicated to store instructions that have been recently used by the CPU. 
* hw.l1dcachesize is also 32 KB and is dedicated for data as opposed to instructions.
* accessing data that is already in cache is faster and it is known as temporal locality
* accessing data that is located near some other data you are using increases the likelihood that data is already in cache too. This is known spatial locality. 
* constant wiping out the cache lines leads to bad performance. it is called cache trashing. 
* example why cache matters
```cpp
constexpr auto kL1CacheCapacity = 32768; // The L1 Data cache size 
constexpr auto kSize = kL1CacheCapacity / sizeof(int); 
using MatrixType = std::array<std::array<int, kSize>, kSize>; 

auto cache_thrashing(MatrixType& matrix) { 
  auto counter = 0;
  for (auto i = 0; i < kSize; ++i) {
    for (auto j = 0; j < kSize; ++j) {
      matrix[i][j] = counter++;
    }
  }
}  
```
this is quick, but what if
```cpp
matrix[j][i] = counter++; 
```
that will slow down, as this block is not in cache

## STL containers
## sequence containers
 * sequence containers keep elements in the order you specify when add them. 
 * they are:
 * std::array
 * std::vector
 * std::deque
 * std::basic_string
 * std::list
 * std::forward_list
 
* How to choose proper containers: number of elements, usage pattern(how often is add, read, traversal, delete, rearrange), do you need to sort?
### std::vector
* std::vector, elements are guaranteed to be stored contiguously. It has size(elements that are already in) and capacity(allocated space)
* objects are moved only when they have nothrow move constructor
```cpp
Person(Person&& other) { // Will be copied 
   ...  
} 
Person(Person&& other) noexcept { // Will be moved 
   ...  
} 
```
as std::move_if_noexcept is used
* how to check if type is move-constructable
```cpp
static_assert(std::is_nothrow_move_constructible<Person>::value, "") 
```
* adding of newly created objects to vector can be done via emplace_back, as it creates object in place
* adding of newly created objects to vector can be done via push_back, as creates an object and then copy/move it to the vector
* vector's capacity would be increased when: object is added to vector and capacity == size, calling reserve(), calling shrink_to_fit()
### std::array
* statically sized, manages elements on stack(not heap), size is a template argument that is specified on compile time. 
* passing to the function
```cpp
auto a = std::array<int, 16>{}; 
auto f(const std::array<int, 16>& input) { 
  ... 
} 
f(a);  // Does not compile, f requires an int array of size 1024
```
### std::deque
* double ended queue
* when you need to add elements to the end and beginning of the sequence(faster than vector is such case)
* is based on fixed-sized arrays -> can access by index
* not stored in the contiguous memory
###  std::forward_list
* single linked list
* optimized for very short lists
* when list is empty it only occupies one word
* memory is not contiguous
### std::list
* double linked list
* when list is empty it only occupies one word
* memory is not contiguous
### std::basic_string
* std::string is a typedef for std::basic_string<char>
* after c++17 is guaranteed to lay in contiguous memory
* small-size optimization -> no dynamic allocations for small amount of objects
  
## Associative containers
* The associative containers place their elements based on the element itself. For example, it's not possible to add an element at the back or front in an associative container. Instead, the elements are added in a way that makes it possible to find the element without the need to scan the entire container. 
### categories of associative containers
* Ordered associative containers: are based on trees. Elements are sorted by < operator. All functions add, delete, find are O(log n), as it is tree based. They are: std::set, std::map, std::multiset, std::multimap. Self balanced tree are having height as O(log n)

* Unordered associative containers: are based on hast tables. Elements are compared by == operator and way to compute hash value. Adding, deleting, finding are O(1). They are: std::unordered_set, std::unordered_map, std::unordered_multiset, std::unordered_multimap. During hash calculations there may be hash collision. Stl propose fix called separate chaining.


* Separate chaining solves the problem of two unequal elements ending up at the same index. Instead of just storing the elements directly in the array, the array is a sequence of lists. Each lists can contain multiple elements, that is, all elements that are hashed to the same index. Good hashing algo distributes hashes among each list equally. 

* Usage of boost hashing algo
``` cpp
auto person_hash = [](const Person& person) { 
  auto seed = size_t{0};
  boost::hash_combine(seed, person.name()); 
  boost::hash_combine(seed, person.age()); 
  return seed;
};
```
* Hash policy. The average number of elements per bucket is called the load_factor. There is max_load_factor, when it reached number of buckets is increased and all elements would be rehashed. It is possible to trigger rehash manually. 

## Container adaptors
* they represent abstract data structures that can be implemented by the underlying sequence container.
* stack (LIFO supports push and pop -> can use vector, list, deque etc*(need back(), push_back(), pop_back()))
* queue
* priority_queue

### Priority queue
* offers constant time lookup of the element with the hightes priority. Priority is based on less operator. Insert and delete in logarithmic time. Is a partialy ordered structure. 


* too big objects is a problem, as they may not be fully in cache
* elements in incontigous memory may be a problem as traversing via them will require cache trashing
* big elements better to split, create other structures and refer to them via pointer/reference. 

## Parallel arrays
* shrink elements on smaller types , instead of pointer we store the smaller structures in separate arrays of equal size. Full object is a sum of elements from those arrays available at the following index.
* adv: when we need only info from some array -> good
* when we need to have a full object we are trashing cache.
* vector of bools is actually a bit array. Count here is very optimized. std::vector<bool> may be deprecated in favor of the fixed size std::bitset and a new dynamic_bitset(now only in boost)

# A deeper look at iterators
* iterator incorporates the following functionality: are we out of the sequence(is_end() -> bool), retrieve value from current position(read() -> T), step to the next position(step_fwd() -> void). Possible step back and write, step an auxiliary number of elements. May even write and step or read and step. 
*    If an algorithm count the number of occurrences of a value, it requires is_end(), read() and step_fwd()
*    If an algorithm fill a container with a value, it requires is_end(), write(), step_fwd()
*    A binary search algorithm on a sorted range requires step() and read()
*    An algorithm which rearrange the elements requires read(), write(), step_fwd() and step_bwd() 

### categories of iterators
* forward_iterator
* bidirectional_iterator
* random_access_iterator
* contigious_iterator(used when underlying data is string, vector, array, valarray)
  
As well as terators compliant with the functions we denoted read_step_fwd()and write_step_fwd: are called input_iterator and output_iterator

### Pointer mimicking syntax
* step_fwd usage, ++it
* step_bwd usage, --it
* step(n) usage, it+=n
* read() usage, auto value = *it;
* write(T val), *it = T{};
* algorithm built upon iterators will also work with regular C pointers.
* it is possible to use boost lib iterator facade. 

### Iterators as generators
* iterator to not need to point to some predefined data, but could generate the data on the fly
```cpp
class IntIterator {
public:
  IntIterator(int v) : v_{v} {}
  auto operator==(const IntIterator& it)const{ return v_ == it.v_; }
  auto operator!=(const IntIterator& it)const{ return !(*this==it); }
  auto& operator*() const { return v_; }
  auto& operator++() { ++v_; return *this; }
private:
  int v_{};
};

auto first = IntIterator{12}; // Start at 12
auto last = IntIterator{16};  // Stop when equal to 16
for(auto it = first; it != last; ++it) {
  std::cout << (*it) << " ";
}
// Prints 12 13 14 15
```

### Iterator traits
* stl differentiates iterators by their traits:
* iterator_category
* difference_type, type used to store the distance betgween two iterators
* value_type, the value returned by dereferenced iterator
* reference, the type used for referencing the value type
* pointer the type of pointer used for pointing to the value_type
```cpp
class IntIterator {
public:
  ...
  using difference_type = int;
  using value_type = int;
  using reference = int&;
  using pointer = int*;
  using iterator_category = std::forward_iterator_tag;
  ...
}
```
* Note that example uses the C++17 way of defining iterator traits for a custom iterator. Prior to C++17, defining iterator traits was more complex, as it required overloading the std::iterator_traits class or inheriting std::iterator (which is now deprecated).
* In order to read properties of an iterator, the STL class std::iterator_traits shall be utilized, not the raw iterator type.Correct: using Category = std::iterator_traits<Iterator>::iterator_categoryIncorrect: using Category = Iterator::iterator_category;
* to write iterator_distance - if random access find difference between indeces, if not calculate steps

```cpp 
template <typename Iterator>
auto iterator_distance(Iterator a, Iterator b) { 
  using Traits = typename std::iterator_traits<Iterator>;
  using Category = typename Traits::iterator_category;
  using Difference = typename Traits::difference_type;
  constexpr auto is_random_access = 
    std::is_same_v<Category, std::random_access_iterator_tag>;
if constexpr(is_random_access) {
    return b - a;
  }
  else {
    auto steps = Difference{};
    while(a != b) { ++steps; ++a; }
    return steps;
  }
}
```
* extending IntIteralt to be bidirtectional : 
```cpp
  using iterator_category = std::bidirectional_iterator_tag;
  auto& operator--() { --value_; return *this; }
```

# Chapter 6. STL Algorithms and Beyond
* algo in stl do operates only on iterator(not containers)
* basically iterator can be considered as a pointer
* generic algo is can work with any kind of containers:
```cpp
template <typename Iterator, typename T>
auto contains(Iterator begin, Iterator end, const T& v) {
  for (auto it = begin; it != end; ++it) {
    if (*it == v) {
      return true;
    }
  }
  return false;
}
```

* a new container can also use all the algorithms if it exposes the iterators. 
* Algorithms do not change the size of the container. For example, std::remove() or std::unique() does not actually remove elements from a container. Rather, it rearranges the elements so that the removed elements are placed at the back. 
* Algorithms that write data to an output iterator, such as std::copy() or std::transform(), requires already allocated data reserved for the output.
* Instead of preallocated memory it is possible to use insert_*iterator
```cpp
auto dst_vec = std::back_inserter(squared_vec);
auto dst_set = std::inserter(squared_set, squared_set.end());
```
* If you are operating on std::vector and know the expected size of the resulting container, you can use the reserve() member function before executing the algorithm in order to avoid unnecessary allocations. Otherwise, the vector will reallocate a new chunk of memory several times during the algorithm.
* comparison, an algorithm relies on the fundamental == and < operators, as in the case of an integer. To be able to use your own classes with algorithms, operator==  and operator< must either be provided by the class or as an argument to the algorithm.
* custome comparator with find_if
```cpp
auto x = std::find_if(names.begin(), names.end(), 
  [target_sz](const auto& v){ return v.size() == target_sz;}
);
```
* good idea is to build a namespace (named preds or something similar) of general-purpose predicates to make the code more readable.
```cpp
auto less_by_size = [](const auto& a, const auto& b){
  return a.size() < b.size();
};
auto equal_by_size = [](auto size){
  return [size](const auto& v){ return size == v.size(); };
};
```
* Algorithms require move operators not to throw
* looking for duplicates in array 1) compare each with each O(n^2) 2) copy array, sort, run an check adjacent elements
* C-style memcpy(), memmove(), memcmp(), memset()  are equal in performance to std::copy(), std::equal() , and std::fill()
* STL algorithms deliver performance. STL algorithms provide safety; even simpler algorithms may have corner cases which is easy to overlook. STL algorithms are future-proof; algorithms can be replaced by a more suitable algorithm if one wants to take advantage of SIMD extensions, parallelism, or even the GPU at a later stage. Good docs. 
* example, need to move first n elements from front to back using for loop
 ```cpp
  template <typename Container>
  auto move_n_elements_to_back(Container& c, size_t n) {
  // Copy the first n elements to the end of the container
  for(auto it = c.begin(); it != std::next(c.begin(), n); ++it) {c.emplace_back(std::move(*it));}
  // Erase the copied elements from front of container
  c.erase(c.begin(), std::next(c.begin(), n));
}
```
it has a problem that if container reallocates duiring iteration due to push_back(), it is no longer valid -> exception
* safe for loop(bad performance)
```cpp
template <typename Container>
auto move_n_elements_to_back(Container& c, size_t n) {
  for(size_t i = 0; i < n; ++i) {
    auto value = *std::next(c.begin(), i);
    c.emplace_back(std::move(value));
  }
  c.erase(c.begin(), std::next(c.begin(), n));
}
```
slower because of std::next uses std::list::iterator -> list is used. 
* best example based on std
```cpp
template <typename Container>
auto move_n_elements_to_back(Container& c, size_t n) {
  auto new_begin = std::next(c.begin(), n);
  std::rotate(c.begin(), new_begin, c.end());
}
```
* no exceptions, can work on static-sized containers, performance is O(n)
* on x86 compare with zero is faster then with other numbers(diff instruction is used)
* If you want to calculate the median of a range, you require the value in the middle of the sorted range. which can be done by std::partial_sort() or nth_element()
* sort 
```cpp
sort(r.begin(), r.end());
```
* Find median 
```cpp
auto middle = r.begin() + r.size() / 2;
nth_element(r.begin(), middle, r.end());
```
* Find the values as if fully ordered From left_idx to right_idx. List unordered.
```cpp
auto left_it = r.begin() + left_idx;
auto right_it = r.begin() + right_idx;
nth_element(r.begin(), left_it,r.end());
nth_element(left_it, right_it, r.end());
```
* Find the values as if fully ordered From left_idx to right_idx List ordered
```cpp
auto left_it = r.begin() + left_idx;
auto right_it = r.begin() + right_idx;
nth_element(r.begin(), left_it, r.end());
partial_sort(left_it, right_it, r.end());
```

## The future of STL and the ranges library
* limitations of the stl. iterator and algorithm concepts in STL lack composability.
* example, need to find the archery max skill(it would require to copy all with archery ability and then sort)
```cpp
enum class EAbility { Fencing, Archery };
class Warrior {
public:
  EAbility ability_{};
  int level_{};
  std::string name_{};
};

auto warriors = std::vector<Warrior> {
  Warrior{EAbility::Fencing, 12, "Zorro"},
  Warrior{EAbility::Archery, 10, "Legolas"},
  Warrior{EAbility::Archery, 7, "Link"}
};

auto is_archer = [](Warrior w){return w.ability_==EAbility::Archery;};
auto compare_level = [](Warrior a,Warrior b){return a.level_<b.level_;};

auto get_max_archor_level(const std::vector<Warrior>& warriors) {
  auto archery = std::vector<Warrior>{};
// The warrior list needs to be copied in order
// to filter to archery only
  std::copy_if(
    warriors.begin(),
    warriors.end(),
    std::back_inserter(archery),
    is_archer
  );
  auto max_level_it = std::max_element(
    archery.begin(),
    archery.end(),
    compare_level
  );
  return *max_level_it;
}

auto max_archor_level = get_max_archor_level(warriors);

```
* same problem can be resolved with ranger lib
```cpp
namespace rv = ranges::view;
auto get_max_archor_level(const std::vector<Warrior>&warriors)
{
  auto archer_levels = warriors | rv::filter(is_archer) | rv::transform([](const auto& w) { return w.level_; });
  auto max_level_it = ranges::max_element(archer_levels);
  return *max_level_it;
}
```
* ranges vs iterators. STL algorithm library has quite a verbose syntax, where every algorithm requires a pair of iterators as parameters. The ranges library has overloaded these functions but by taking a range as a parameter instead of a pair of iterators:
```cpp
ranges::sort(a);
ranges::count(a, 12);

std::sort(a.begin(), a.end());
std::count(a.begin(), a.end(), 12);
```
* main feature with the ranges library is the introduction of  views. Views in the range library are lazily evaluated iterations over a range. technically they are only iterators with built in logic, but syntactically, they provide a very pleasant syntax for many common operations.
*  possibility it offers of creating a view that can iterate over several containers as if they were a single list
```cpp
auto list_of_lists = std::vector<std::vector<int>> {
  {1, 2},
  {3, 4, 5},
  {5},
  {4, 3, 2, 1}
};
auto flattened_view = rv::join(list_of_lists);
for(auto v: flattened_view) 
  std::cout << v << ", ";
// Output: 1, 2, 3, 4, 5, 5, 4, 3, 2, 1,

auto max_value = *ranges::max_element(flattened_view);
// max_value is 5
```
* combining and piping 
```cpp
namespace rv = ranges::view;
auto numbers = std::vector<int>{1,2,3,4,5,6,7};
auto odd_squares_view = numbers
  | rv::transform([](auto v){ return v * v; })
  | rv::filter([](auto v){ return (v % 2) == 1; });

// odd_squared_view evaluates to "1, 9, 25, 49"
```
* ranges library consists of three types of operations: algorithms, actions, views
* diff between actions and views:
```cpp
// Prerequisite
auto is_odd = [](int v){ return (v % 2) == 1; }
auto square = [](int v){ return v*v; }
auto get() { return std::vector<int>{1,2,3,4}; }
using ra = ranges::action;
using rv = ranges::view;
// Print odd squares using actions...
for(auto v: get() | ra::remove_if(is_odd) | ra::transform(square)) {
 std::cout << v << " "; 
}

// Print odd squares using a view...
for(auto v: get() | rv::remove_if(is_odd) | rv::transform(square)) { 
  std::cout << v << " "; 
}
```
* Actions work like standard STL algorithms, but instead of using iterators as input/output, they take containers and return new modified containers.
* Unlike STL algorithms, the actions of ranges modify the size of the container, that is, ranges::action::remove_if() returns a container where the elements have been erased from the container, whereas std::remove_if(...) simply returns an iterator
* In order to mutate a container using actions you must choose one of the following:1) provide the container as an r-value 2) initiate the action with |= instead of | which mutates the container inplace.
* example of initialization
```cpp
namespace ra = ranges::action;
auto get(){return std::vector<int>{1,3,5,7};}
auto above_5 = [](auto v){ return v >= 5; };
```
* example of r-value getting
```cpp
auto vals = get() | ra::remove_if(above_5);
```
* example of constructed r-value using move
```cpp
auto vals = get();
vals = std::move(vals) | ra::remove_if(above_5);
```
* example with mutate operator
```cpp
auto vals = get();
vals |= ra::remove_if(above_5);
``` 
* Views return a proxy view, which, when iterated, looks like a mutated container.
* As actions transforms the container, they cannot transform the type, whereas views can transform to a view of any type.
```cpp
namespace ra = ranges::action;
namespace rv = ranges::view;
auto get_numbers() { return std::vector<int>{1,3,5,7}; }
// An action cannot transform the type
auto strings = get_numbers() | ra::transform([](auto v){ 
  return std::to_string(v); // Does not compile
});
// A view can transform the type 
auto ints = get_numbers();
auto str_view = ints | rv::transform([](auto v){ 
  return std::to_string(v);
});
```
* To mimic the behavior of std::transform() which can transform data to a container with a different value type, the view can be converted to a container. This can be performed either by using the functions ranges::to_vector, ranges::to_list etc, or by simply assigning the view to the range of choice.
```cpp
namespace rv = ranges::view;
auto a = std::list<int>{2,4};
std::list<std::string>b = a
 | rv::transform([](auto v){
   return std::to_string(v);
 });
//or
namespace rv = ranges::view;
auto a = std::list<int>{2,4};
auto b = a | rv::transform([](auto v){return std::to_string(v);}) | ranges::to_list;
```
### Algorithms
* The algorithms in the ranges library that do not return a view or a mutated container are simply referred to as algorithms. Examples of such algorithms are ranges::count, ranges::any_of, and so on. They work exactly as non-mutating algorithms in STL, with the exception that they use a range as input instead of a pair of iterators.
* cannot be chained pipe
```cpp
auto cars = std::vector<std::string>{"volvo","saab","trabant"};
// Using the STL library
auto num_volvos_a = std::count(cars.begin(), cars.end(), "volvo");
// Using the ranges library
auto num_volvos_b = ranges::count(cars, "volvo");
```
# Memory management
* isolation of memory is performed by virtual memory(each process has its own virtual address space)
* addresses in virtual address space are mapped by OS and MMU(part of processor)
* OS has may backup unused at the moment part of process to backup on the disk. The areas used for backups are called swap space, swap file, pagefile. It makes possible for processes to have virtual memory more than physical address space. 
* the most common way to implement virtual memory is to divide address space in fixed sized blocks called memory pages. 
* when process accesses memory at virtual address OS checks if the memory page is backed by physical memory. If the memory page is not mapped in the main memory, a hardware exception occurs and page is loaded from disk into memory. This type of hardware exception is called a page fault. This is not an error(just a way to interrupt loading of data until it is mapped). This operation takes more time than reading already present data.
* when there are no available page frames(memory page backed by physical memory) - new page frame to be evicted
* if the page to be evicted is dirty(differs from what was loaded from disk) it need to be written to disk before it can be replaced. This mechanism is called pagin. If memory is not modified -> page simply is evicted. 
* Not all OS that have virtual memory support paging(iOS). Virtual pages are never stored to disk, only clean pages can be evicted from memory. If main memory is full-> kill processes.
## Process memory
* stack, heap, static storage, thread local storage.
* usually(deps on arch) heap starts on low addressesof virtual memory, and grows upward, stack vice verse. 

### Stack 
* contiguous memory
* fixed size, if exceeds -> crash
* memory is not fragmented
* allocation fast
* each thread in program has its own stack
* ulimit -> to change or show stack size

### Heap
* is shared between threads
* memory may be fragmented(allocated 1kn, 2kb, 1kb, 2kb, deallocate all 1kb chunks, now when need to add 2kb object need to add smwh else, not at the place the 1kb were residing)

### Objects in memory
* new allocates and constructs
```cpp
auto user = new User{"John"};
```
* allocation and construction may be separatedm - placement new
```cpp
auto memory = std::malloc(sizeof(User));
auto user = new(memory) User("john);
...
user->~User(); //unfortunately need to be called explicitly
std::free(memory);
```
* c++17 introduces utility functions in <memory> for constructing and destructing objects without allocation deallocation of memory.
  * instead of placement new, we could use std::uninitialized_, instead of destructor call std::destroy_at()
  ```cpp
auto memory = std::malloc(sizeof(User));
auto user_ptr = reinterpret_cast<User*>(memory);
std::uninitialized_fill_n(user_ptr, 1, User{"john"});
std::destroy_at(user_ptr);
std::free(memory);
```
* when overloading operator new, need to overload also delete
  
### Memory allignment
* CPU reads memory into registers one word per time. Word size is 64bits or bits.
* alignof to find alignment
*  alignof(int) ->4 means address for int is a multiple of 4
* new and malloc guarantee that memory is alligned to the specified type
* <cstddef> header provides us with a type called std::max_align_t, whose alignment requirement is at least as strict as all the scalar types
  ```cpp
  auto *p = new char{};
  auto address = reinterpret_cast<std::uintptr_t>(p);
  auto max_alignment = alignof(std::max_align_t);
  std::cout<< (address % max_alignment) << std::endl;
 ```
* between two chars there would be distance alignof(std::max_align_t)

### Padding
```cpp
class Document { 
  bool is_cached_{}; 
  double rank_{}; 
  int id_{}; 
};
std::cout << sizeof(Document) << '\n'; // Possible output is 24
vs
class Document { 
  bool is_cached_{}; 
  char padding1[7]; // Invisible padding inserted by compiler 
  double rank_{}; 
  int id_{}; 
  char padding2[4]; // Invisible padding inserted by compiler 
}; 
```
* kinda rule, biggest at the beginning.
* better place data that is frequently used together

### Memory ownership
* smart pointers help to specify the ownership
* static and global are owned by program, and will be destroyed when program terminates
* Resource Acquisition Is Initialization, where the lifetime of a resource is controlled by the lifetime of an object.
* Unique ownership expresses that I, and only I, own the object. When I'm done using it, I will delete it.
* Shared ownership expresses that I own the object along with others. When no one needs the object anymore, it will be deleted.
* Weak ownership expresses that I'll use the object if it exists, but don't keep it alive just for me.
* Whenever we try to use the weak pointer, we need to convert it to a shared pointer first using the member function lock()

### Small size optimization
* may be done using union, example string


### Custom memory management
* custom memory allocator may be used for debugging, sandboxing, performance
* two important notations are: arena and memory pool
* arena is a block of contiguous memory including a strategy for handling out parts of that memory and reclaiming it later on. This could technically also be called an allocator. 
* several strategies about allocations/deallocations: 
  *   single-threaded, no need to protect data using synchronization primitives(locks, mutexes, atomics)
  *   fixed-size allocations, if arena hands out memory blocks of fixed size, easy to do, efficient and without memory fragmentation using free list
  *   limited lifetime, it is possible to reclaim all the new memory at once, and no need to clean up and claim continuously. 
* short_alloc, stack_alloc  - allocator on stack for small objects  
```cpp
template <size_t N> 
class Arena { 
  static constexpr size_t alignment = alignof(std::max_align_t); 
public: 
  Arena() noexcept : ptr_(buffer_) {} 
  Arena(const Arena&) = delete; 
  Arena& operator=(const Arena&) = delete; 
 
  auto reset() noexcept { ptr_ = buffer_; } 
  static constexpr auto size() noexcept { return N; } 
  auto used() const noexcept {  
    return static_cast<size_t>(ptr_ - buffer_); 
  } 
  auto allocate(size_t n) -> char*; 
  auto deallocate(char* p, size_t n) noexcept -> void; 
   
private: 
  static auto align_up(size_t n) noexcept -> size_t { 
    return (n + (alignment-1)) & ~(alignment-1); 
  } 
  auto pointer_in_buffer(const char* p) const noexcept -> bool { 
    return buffer_ <= p && p <= buffer_ + N; 
  } 
  alignas(alignment) char buffer_[N]; 
  char* ptr_{}; 
};

template<size_t N> 
auto Arena<N>::allocate(size_t n) -> char* { 
  const auto aligned_n = align_up(n); 
  const auto available_bytes =  
  static_cast<decltype(aligned_n)>(buffer_ + N - ptr_); 
  if (available_bytes >= aligned_n) { 
    char* r = ptr_; 
    ptr_ += aligned_n; 
    return r; 
  } 
  return static_cast<char*>(::operator new(n)); 
} 

template<size_t N> 
auto Arena<N>::deallocate(char* p, size_t n) noexcept -> void { 
  if (pointer_in_buffer(p)) { 
    n = align_up(n); 
    if (p + n == ptr_) { 
      ptr_ = p; 
    } 
  } 
  else { 
    ::operator delete(p);
  }
} 
auto user_arena = Arena<1024>{}; 
 
class User { 
public: 
  auto operator new(size_t size) -> void* { 
    return user_arena.allocate(size); 
  } 
  auto operator delete(void* p) -> void { 
    user_arena.deallocate(static_cast<char*>(p), sizeof(User)); 
  } 
  auto operator new[](size_t size) -> void* { 
    return user_arena.allocate(size); 
  } 
  auto operator delete[](void* p, size_t size) -> void { 
    user_arena.deallocate(static_cast<char*>(p), size); 
  } 
private:
  int id_{};
}; 
 
auto main() -> int { 
  // No dynamic memory is allocated when we create the users 
  auto user1 = new User{}; 
  delete user1; 
 
  auto users = new User[10]; 
  delete [] users; 
 
  auto user2 = std::make_unique<User>(); 
  return 0; 
}

```
* when using std collection we may provide custom allocator
* minimal impl
```cpp
template<typename T> 
struct Alloc {  
  using value_type = T; 
  Alloc(); 
  template<typename U> Alloc(const Alloc<U>&); 
  T* allocate(size_t n); 
  auto deallocate(T*, size_t) const noexcept -> void; 
}; 
template<typename T> 
auto operator==(const Alloc<T>&, const Alloc<T>&) -> bool;   
template<typename T> 
auto operator!=(const Alloc<T>&, const Alloc<T>&) -> bool; 
```

# Chapter 8. Metaprogramming and Compile-Time Evaluation
* Metaprogramming, is code that transforms itself into regular C++ code
* function template
```cpp
// pow_n accepts any number type
template <typename T> 
auto pow_n(const T& v, int n) { 
  auto product = T{1}; 
  for(int i = 0; i < n; ++i) { 
    product *= v; 
  }
  return product; 
}
```
* class template
```cpp
// Rectangle can be of any type 
template <typename T> 
class Rectangle { 
public: 
  Rectangle(T x, T y, T w, T h) : x_{x}, y_{y}, w_{w}, h_{h} {} 
  auto area() const { return w_ * h_; } 
private:
  T x_{}, y_{}, w_{}, h_{}; 
}; 

auto rectf = Rectangle<float>{2.0f, 2.0f, 4.0f, 4.0f}; 
```
* using integers as template parameters
```cpp
template <int N, typename T> 
auto const_pow_n(const T& v) { 
  static_assert(N >= 0); // Only works for positive numbers 
  auto product = T{1}; 
  for(int i = 0; i < N; ++i) { 
    product *= v; 
  }
  return product; 
}

// The compiler generates a function which squares the value
auto x2 = const_pow_n<float, 2>(4.0f); 
// The compiler generates a function which cubes the value
auto x3 = const_pow_n<float, 3>(4.0f);
```
* to check smth on compile-time -> static assert

### Type traits
* information about the types at compile time
* type traits 2 categories: 
  *   type traits that return info about a type as a boolean (are ending on \_v  C++17, previously it was ::value)
  *   type traits that return a new type ( are enging on \_t)
* example of \_v or ::value
```cpp
auto is_float = std::is_floating_point<A>::value; 
auto same_type = std::is_same<uint8_t, unsigned char>;
auto is_float_or_double = std::is_floating_point_v<decltype(flt)>;
class Parent {};
class Child : public Parent {};
class Infant {};
auto same_type = std::is_base_of<Child, Parent>;
auto same_type = std::is_base_of<Infant, Parent>;
```
* type_traits are from lib <type_traits>
* receiving type of variable via decltype
```cpp
auto sign_func = [](const auto& v) -> int { 
  using ReferenceType = decltype(v); 
  using ValueType = std::remove_reference_t<ReferenceType>; 
}; 
or 
  using IteratorType = decltype(r.begin()); 
  using ReferenceType = decltype(*IteratorType()); 
```

* std::enable_if_t type trait is used for function overloading when dealing with a template function. Can for example block calling of function with integer. It's syntax:
    * It is used as a return type
    * The first templated parameter is the condition
    * The second parameter is the returned value if the condition is fulfilled
```cpp
template <typename T>
auto interpolate(T left, T right, T power) -> std::enable_if_t<std::is_floating_point_v<T>, T>
{ 
  return left * ( 1 - power) + right * power;
}
//If this function is called with a non-floating point type, the code will not compile, as the function only exists for floating points.
```

* introspecting class members with sdt::is_detected. It is in <experimental/type_traits>, and exists in std::experimental. Used to detect if a class contains a particular member functions, member typedefs or member variables
```cpp
#include <experimental/type_traits> 

struct Octopus { 
  auto mess_with_arms() {} 
  int has_arms{};
}; 

template <typename T> 
using can_mess_with_arms = decltype(&T::mess_with_arms); 

auto tester() { 
  static_assert(std::experimental::is_detected<can_mess_with_arms, Octopus>::value, "")
  static_assert(std::experimental::is_detected<has_arms, Octopus>::value, "")
}
```
* combine is_detected and enable_if_t
```cpp
template<typename T> using has_to_string = decltype(&T::to_string);
template<typename T> using has_name_member = decltype(T::name_);

// Print the to_string() function if it exists in class
template <typename T,
bool HasToString = exp::is_detected<has_to_string,T>::value,
bool HasNameMember = exp::is_detected<has_name_member,T>::value>
auto print(const T& v)-> std::enable_if_t<HasToString && !HasNameMember>
{
  std::cout<< v.to_string() << std::endl;
}
```
### constexpr keyword
* tells compiler that function(?) is going to be evaluated at compile time if all the conditions for compile time evaluation are fullfilled. 
* A constexpr function has a few restrictions; it is not allowed to do the following in C++17(in C++11 allowed to contain a single return statement, requiring the programmer to resort to recursion for more advanced constexpr functions):
  *   Allocate memory on the heap
  *   Throw exceptions
  *   Handle local static variables
  *   Handle thread_local variables
  *   Call any function, which, in itself, is not a constexpr.

```cpp
constexpr auto sum(int x, int y, int z) {return x + y + z;}
//if to call (i.e. known values in compile time)
const auto value = sum(3, 4, 5)
//is overrided by following generated code
const auto value = 12;
```
### Verify compile-time computation using std::integral_constant
* std::integral_constant may be used to verify if constexpr is evaluated at compile time.  The integral constant is a template class that takes an integer type and an integer value as template parameters. 
*  If the compiler cannot evaluate the integer value at compile time, it won't compile. The value of the class is then accessed via the static field value of std::integral_constant.
```cpp
const auto ksum = std::integral_constant<int, sum(1,2,3)>; 

auto func() -> void { 
  // This compiles as the value of sum is evaluated at compile time 
  const auto sum_compile_time = std::integral_constant<int,sum(1,2,3)>; 
  int x, y, z; 
  std::cin >> x >> y >> z; 
  // Line below will not compile, the compiler cannot determine which value 
  // the integral constant has at compile time 
  const auto sum_runtime = std::integral_constant<int, sum(x, y, z)>; 
} 
```
* if constexpr
```cpp
template <typename Animal> 
auto speak(const Animal& a) { 
  if constexpr (std::is_same_v<Animal, Bear>) { a.roar(); } 
  else if constexpr (std::is_same_v<Animal, Duck>) { a.quack(); } 
}
```

### Heterogeneous containers
* containers that contain different types, std::pair, std::tuple
* The std::tuple container. cannot be changed in runtime, cannot add or remove elements
```cpp
auto tuple0 = std::tuple<int, std::string, bool>{}; 
auto tuple = std::make_tuple(42, std::string{"hi"}, true);
auto number = std::get<0>(tuple); 
auto str = std::get<1>(tuple); 
//only possible if the specified type is contained once in the tuple.
auto str = std::get<std::string>(tuple);
```
* type trait std::tuple_size_v<Tuple>  
```cpp
  template <typename Tuple, typename Functor, size_t Index = 0>
auto tuple_for_each(const Tuple& tpl, const Functor& f) -> void {
  constexpr auto tuple_size = std::tuple_size_v<Tuple>;
if constexpr(Index < tuple_size) {
    tuple_at<Index>(tpl, f);
    tuple_for_each<Tuple, Functor, Index+1>(tpl, f);
  }
}
auto tpl = std::make_tuple(1, true, std:.string{"Jedi"}); 
tuple_for_each(tpl, [](const auto& v){ std::cout << v << " "; }); 
```
* Structured bindings, multiple variables can be initialized at once using auto and a bracket initializer list. C++17
  ```cpp
  const auto& [name, id, kill_license] = make_bond(); 
  
  auto agents = { 
    std::make_tuple("James", 7, true), 
    std::make_tuple("Nikita", 108, false) 
  };
  for(auto&& [name, id, kill_license]: agents) { 
     std::cout << name << ", " << id << ", " << kill_license << '\n'; 
  } 
```
* If you want to return multiple arguments with named variables instead of tuple indices,
```cpp
auto make_bond() {
  struct Agent{std::string name_; int id_; bool kill_license_;}
  return Agent{"James", 7, true}; 
} 
auto b = make_bond(); 
std::cout << b.name_ << " " << b.id_ << " " < b.kill_license_ << '\n'; 
```
* The variadic template parameter pack. Syntax expanations:
    *   Ts is a list of types
    *   The <typename ...Ts&> function indicates that the function deals with a list
    *   The values... function expands the pack such that a comma is added between every type
```cpp
//was 
// Makes a string of by two arguments 
template <typename T0, typename T1> 
auto make_string(const T0& v0, const T1& v1) -> std::string { 
   return make_string(v0) + " " + make_string(v1); 
} 

template <typename ...Ts> 
auto make_string(const Ts& ...values) { 
  auto sstr = std::ostringstream{}; 
  // Create a tuple of the variadic parameter pack 
  auto tuple = std::tie(values...); 
  // Iterate the tuple 
  tuple_for_each(tuple, [&sstr](const auto& v){ sstr << v; }); 
  return sstr.str(); 
} 
```

* Dynamic-sized heterogenous containers. Using std::any as the base for a dynamic-size heterogenous container. Is in C++17,or boost::any. It has performance problems as runtime type checking
```cpp
auto container = std::vector<std::any>{42, "hi", true}; 

for(const auto& a: container) { 
  if(a.type() == typeid(int)) {  
    const auto& value = std::any_cast<int>(a); 
    std::cout << value; 
  } 
  else if(a.type() == typeid(const char*)) {  
    const auto& value = std::any_cast<const char*>(a); 
    std::cout << value; 
  } 
} 
```
### The std::variant
* trade off the ability to store any type in the container and, rather, concentrate on a fixed set of types declared at the container initialization
* The std::variant has two main advantages over std::any:
    * It does not store its contained type on the heap (unlike std::any)
    * It can be invoked with a polymorphic lambda, meaning you don't explicitly have to know its currently contained type 
* it is something like a union(same memory, several objects, only one at one moment of time)
```cpp
using VariantType = std::variant<int, std::string, bool>; 
auto v = VariantType{}; // The variant is empty 
v = 7; // v holds an int 
v = std::string{"Bjarne"}; // v holds a std::string, the integer is overwritten 
v = false; // v holds a bool, the std::string is overwritten
```
* visiting variants. 
```cpp
std::visit([](const auto& v){std::cout << v;},my_variant);
```
* The size of the variant is equal to the largest object type declared as its member

### Heterogenous container of variants
```cpp
using VariantType = std::variant<int, std::string, bool>; 
auto container = std::vector<VariantType>{}; 
container.push_back(false); 
container.pop_back();
std::reverse(container.begin(), container.end());
```
* check if variant holds the required type, std::holds_alternative<type>
```cpp
        return std::holds_alternative<bool>(v); 
```
* Global function std::get, can be used with std::tuple, std::pair, std::variant, std::array
  ```cpp
  std::get<Index>(container)
  or 
  std::get<Type>(container)
  ```
### Real world examples of metaprogramming 
  * example1, reflection - ability to inspect class without knowing about its content
  ```cpp
  class Town{
   public: 
    Town(size_t houses, size_t settlers, const std::string& name):
  houses_{houses}, 
  settlers_{settlers},
  name_{name}
  {}
  auto reflect () const {return std::tie(houses_, settlers_, name_);}
  private:
  size_t houses_{};
  size_t settlers_{};
  std::string name_{};
 };
 
 auto& operator<<(std::ostream& ostr, const Town& t) { 
 tuple_for_each(t.reflect(), [&ostr](const auto& m){ 
    ostr << m << " "; 
  }); 
  return ostr; 
}
```
* reflectiong is also available in Boost Hana, if to use macro. Reflection of public simple-type content of classes
*  every type which has the reflect() member function should also have operator==, operator< and the global std::stream& operator 
    *  class we will test has a reflect member function returning a tuple of references to its members
    *  the equality and less than comparison functions are enabled for all reflectable types
    *  the global std::ostream& operator<< is overloaded for reflectable types
```cpp
  auto shire = Town{100, 200, "Shire"}; 
  std::cout << shire; 
```
### Example 2 – Creating a generic safe cast function
* problems during cast
    * may lose a value if casting to lower bit type
    * may lose if cast from negative to unsigned
    * castring from a pointer to any other integer then uintptr_t
    * if casting from double to float
    * casting between pointers with a static_cast() may be undefined behaviour if pointers are not sharing common base class
* safe_cast() is intended to handle:
  * same type casting, actually returning input value
  * pointer to point
  * double to floating, double may be too large for a float number
  * arithmetic to arithmetic, cast back to check if nothing is lost
  * pointer to non-pointer - checks if destination is uintptr_t or intptr_t
  * all other cases -> fail
 ```cpp
template <typename T> constexpr auto make_false() { return false; }
template <typename Dst, typename Src> 
auto safe_cast(const Src& v) -> Dst{ 
  using namespace std;
  constexpr auto is_same_type = is_same_v<Src, Dst>;
  constexpr auto is_pointer_to_pointer =  
    is_pointer_v<Src> && is_pointer_v<Dst>; 
  constexpr auto is_float_to_float =  
    is_floating_point_v<Src> && is_floating_point_v<Dst>; 
  constexpr auto is_number_to_number =  
    is_arithmetic_v<Src> && is_arithmetic_v<Dst>; 
  constexpr auto is_intptr_to_ptr = (
    (is_same_v<uintptr_t,Src> || is_same_v<intptr_t,Src>)
    && is_pointer_v<To>; 
  constexpr auto is_ptr_to_intptr = 
    is_pointer_v<Src> &&
    (is_same_v<uintptr_t,Dst> || is_same_v<intptr_t,Dst>); 
 ```
 
 ### Example 3 – Hash strings at compile time
 * boost::hash_combine()
 * prehashed string:
 ```cpp
 class PrehashedString { 
public: 
  template <size_t N> 
  constexpr PrehashedString(const char(&str)[N]) 
  : hash_{hash_function(&str[0])} 
  , size_{N - 1} // The subtraction is to avoid null at end 
  , strptr_{&str[0]} 
  {} 
  auto operator==(const PrehashedString& s) const { 
    return 
      size_ == s.size_ && 
      std::equal(c_str(), c_str() + size_, s.c_str()); 
  } 
  auto operator!=(const PrehashedString& s) const {
    return !(*this == s); } 
  constexpr auto size()const{ return size_; } 
  constexpr auto get_hash()const{ return hash_; } 
  constexpr auto c_str()const->const char*{ return strptr_; } 
private: 
  size_t hash_{}; 
  size_t size_{}; 
  const char* strptr_{nullptr}; 
}; 
 
namespace std { 
template <> 
struct hash<PrehashedString> { 
  constexpr auto operator()(const PrehashedString& s) const { 
    return s.get_hash(); 
  } 
}; 
}
 
 const auto& hash_fn = std::hash<PrehashedString>{}; 
 const auto& str = PrehashedString("abc"); 
 std::cout <<  hash_fn(str); 
 ```


# Chapter 9. Proxy Objects and Lazy Evaluation
* Lazy evaluation is a technique used to postpone an operation until its result is really required. The opposite, where operations are performed right away, is called eager evaluation
* sometime eager evaluation may be undesired if the calculation would be never called
```cpp
struct Audio {};
audo load_audio(const std::string& path) ->Audio {...};

class AudioLibrary
{
  public: 
  auto get_eager(std::string id, const &Audio& otherwise)const
  {
    return map_.count(id) ? map.at(id) : otherwise;
  }
  audo get_lazy(std::string id, std::function<Audio> otherwise)const
  {
    return map_.count(id) ? map.at(id) : otherwise();
  }
  private:
  std::map<std::string, Audio> map_{};
  
  auto library = AudioLibrary{};
  auto red_fox_sound = library.get_eager(
  "red_fox", 
  load_audio("default_fox.wav")//function would be executed, even if returned value is not used
);

auto library = AudioLibrary{};
auto red_fox_sound = library.get_lazy(
  "red_fox", 
  [](){ return load_audio("default_fox.wav"); }
);

}
```

### Proxy objects
* are internal library objects that aren't intended to be visible for the user of the library. 
* Their task is to postpone operations until required and to collect the data of an expression until it can be evaluated and optimized. 



# Chapter 10. Concurrency
* Concurrency and parallelism. A program is said to run concurrently if it has multiple individual control flows running during overlapping time periods. In C++, each individual control flow is represented by a thread. The threads may or may not execute at the exact same time, though. If they do, they are said to execute in parallel. 
* Time slicing, concurrent threads on single cpu. Threads execution on cpu is interruopted by context switching.
* Shared memory. Threads created in the same process share the same virtual memory. 
* each thread has its stack. Nice way to save your data from other threads. 
* TLS thread local storage, place to store variables that are global in context of a thread, but are not shared between threads. 
* Everything else is shared by default: that is, dynamic memory allocated on the heap, global variables, and static local variables
* how to avoid data races: 
      * use atomic variables
      * use mutually exclusive lock
      * if possible use const, constr exps
      * create new immutable objects instead of mutating existing objects, then swap(to protect with mutex, or atomic operation)
      
 * deadlock - two threads are waiting for each other to release their lovs. 
      
 * An asynchronous task, will return the control back to the caller immediately and instead perform its work concurrently.
 * thread library, get thread id
 ```cpp
 std::cout << std::this_thread::get_id();
  ```
  * make thread sleep
  ```cpp
  std::this_thread::sleep_for(std::chrono::seconds{1}); 
  ```
  * 
```cpp
  auto print() { 
  std::this_thread::sleep_for(std::chrono::seconds{1}); 
  std::cout << "Thread ID: "<<  std::this_thread::get_id() << '\n'; 
} 
 
auto main() -> int { 
  auto t1 = std::thread{print}; 
  t1.join(); 
  std::cout << "Thread ID: "<<  std::this_thread::get_id() << '\n'; 
} 
```
* std::thread object is destructed, it must have been joined or detached or it will cause the program to call std::terminate(), which by default will call std::abort() if you haven't installed a custom std::terminate_handler.
* join() function is blocking—it waits until the thread has finished running
* detach() function is not blocking
* how many hardware threads:
```cpp
  std::cout << std::thread::hardware_concurrency() << '\n'; 
```
* Two instances of std::thread cannot be associated with the same underlying OS thread.
* It is possible to query a std::thread object in what state it is by using the std::thread::joinable property.
* A thread is not joinable if it has been:
    * default constructe, (i.e. has nothing to execute)
    * moved from (its associated thread has been transferred to another std::thread object)
    * detached by a detach()
    * joined by join()
  * when thread is destructed it must be no longer joinable(otherwise -> termination of programm)
 * avoiding dead lock:
 ```cpp
 struct Account { 
  Account() {} 
  int balance_ = 0; 
  std::mutex m_{}; 
}; 
 
void transfer_money(Account& from, Account& to, int amount) { 
   auto lock1 = std::unique_lock<std::mutex>{from.m_, std::defer_lock}; 
   auto lock2 = std::unique_lock<std::mutex>{to.m_, std::defer_lock}; 
   // Lock both unique_locks at the same time 
   std::lock(lock1, lock2); 
   from.balance_ -= amount; 
   to.balance_ += amount; 
} 

or 
void transfer(bank_account &from, bank_account &to, int amount)
{
      std::lock(from.m, to.m);
    // make sure both already-locked mutexes are unlocked at the end of scope
    std::lock_guard<std::mutex> lock1(from.m, std::adopt_lock);
    std::lock_guard<std::mutex> lock2(to.m, std::adopt_lock);
    from.balance -= amount;
    to.balance += amount;
}
 ```
 * useful functions  from <mutex>
      * defer_lock_t 	do not acquire ownership of the mutex
      * try_to_lock_t 	try to acquire ownership of the mutex without blocking
      * adopt_lock_t 	assume the calling thread already has ownership of the mutex
* condition variables, makes it possible tfor threads to wait until some specific condition has been met. Can be used for communication with other threads.
  * common pattern is to have one or many threads that are waiting for data to be consumed somehow. Consumers. Another group of threads are then responsible for producing data that is ready to be consumed. Producers. 
  * Producer and consumer patter can be implmeneted via condition variable. Combination std::condition_variable, std::unique_lock
  
```cpp
auto cv = std::condition_variable{}; 
auto q = std::queue<int>{}; 
auto mtx = std::mutex{}; // Protects the shared queue 
constexpr int done = -1; // Special value to signal that we are done 
 
void print_ints() { 
  auto i = int{0}; 
  while (i != done) { 
    { 
      auto lock = std::unique_lock<std::mutex>{mtx}; 
      while (q.empty())
        cv.wait(lock); // The lock is released while waiting 

      i = q.front(); 
      q.pop(); 
    } 
    if (i != done) { 
      std::cout << "Got: " << i << '\n'; 
    } 
  } 
} 
 
auto generate_ints() { 
  for (auto i : {1, 2, 3, done}) { 
    std::this_thread::sleep_for(std::chrono::seconds(1)); 
    { 
      std::lock_guard<std::mutex> lock(mtx); 
      q.push(i); 
    } 
    cv.notify_one(); 
  } 
} 
  
auto main() -> int { 
   auto producer = std::thread{generate_ints}; 
   auto consumer = std::thread{print_ints}; 
  
   producer.join(); 
   consumer.join(); 
} 
  ```
*  it is possible for the consumer to be awoken from its wait even though the producer thread did not signal - spurious wakeup.  always check the condition in a while-loop.

### Returning data and handling errors
* in <future> header there are class templates that help writing concurrent code without global variables and locks, and can communicate exceptions between threads for handling errors. 
*  futures and promises, which repesent two sides of a value. 
```cpp
  auto divide(int a, int b, std::promise<int>& p) { 
  if (b == 0) { 
    auto e = std::runtime_error{"Divide by zero exception"}; 
    p.set_exception(std::make_exception_ptr(e)); 
  } 
  else { 
    const auto result = a / b; 
    p.set_value(result); 
  } 
} 
 
auto main() -> int { 
   auto p = std::promise<int>{}; 
   std::thread(divide, 45, 5, std::ref(p)).detach(); 
     
   auto f = p.get_future(); 
   try { 
     const auto& result = f.get(); // Blocks until ready 
     std::cout << "Result: " << result << '\n'; 
   } 
   catch (const std::exception& e) { 
     std::cout << "Caught exception: " << e.what() << '\n'; 
   } 
} 
```
### Tasks
* help to automatically set up futures and promises
* std::packaged_task from <future>, std::packaged_task is itself a callable object that can be moved to the std::thread object
```cpp
auto divide(int a, int b) -> int { // No need to pass a promise ref here! 
  if (b == 0) { 
    throw std::runtime_error{"Divide by zero exception"}; 
  } 
  return a / b; 
} 
 
auto main() -> int { 
  auto task = std::packaged_task<decltype(divide)>{divide}; 
  auto f = task.get_future(); 
  std::thread{std::move(task), 45, 5}.detach(); 
     
  // The code below is unchanged from the previous example 
  try { 
    const auto& result = f.get(); // Blocks until ready 
    std::cout << "Result: " << result << '\n'; 
  } 
  catch (const std::exception& e) { 
    std::cout << "Caught exception: " << e.what() << '\n'; 
  } 
  return 0; 
} 
```
* async() - replaces packaged_task and thread part of code in our cpp impl, but that is a move from thread-based programming to task-based model. 
```cpp
auto main() -> int { 
  auto future = std::async(divide, 45, 5); 
  try { 
    const auto& result = future.get(); 
    std::cout << "Result: " << result << '\n'; 
  } 
  catch (const std::exception& e) { 
    std::cout << "Caught exception: " << e.what() << '\n'; 
  } 
} 
```
### atomics in c++.
* An atomic is a variable that can be safely muted from different threads. 
* if atomics do not use locks to protect underlying data, it is called lock-fee. 
* example
```cpp
std::atomic<int> counter; 
 
auto increment_counter(int n) { 
  for (int i = 0; i < n; ++i) 
    ++counter; // Safe, counter is now an atomic<int> 
} 
```
* std::atomic_int is typedefed to std::atomic<int>
  * it is possible to wrap custom type in atomic, as long as it is trivially copied(i.e. should have no pointers to dynamic memory, virtual functions, etc)
 ```cpp
  struct Point { 
  int y{}; 
  int x{}; 
}; 
 
auto p = std::atomic<Point>{};       // OK: Point is trivially copyable 
auto s = std::atomic<std::string>{}; // Error: cannot be trivially copied 
```

* shared_ptr in multithreading
```cpp 
//to restore in memory
auto p1 = std::make_shared<int>(int{42}); 
```
* it creates int on heap and a reference counted pointer to int., when make_shared is used, control block is created next to int. 
    * i.e. three entities are created
    * the actual shared_ptr
    *  the control block
    * the int
* now in another thread to make the following: 
```cpp
auto p2 = p1; 
```
* control block need to be mutated and it happens that in shared ptr's control block  is already covered by atomics,
* still to make shared pointer we need to overload the atomic type for it
```cpp
// Thread T1 calls this function
auto f1() { 
  auto new_p = std::make_shared<int>(std::rand());
  // ... 
  std::atomic_store(&p, new_p);
} 
 
// Thread T2 calls this function
auto f2() { 
  auto local_p = std::shared_ptr<int>{std::atomic_load(&p)}; 
  // Use local_p... 
} 
```
### C++ memory model
*  https://herbsutter.com/2013/02/11/atomic-weapons-the-c-memory-model-and-modern-hardware/
* use mutexes to create acquire memory fence that stop compiler in optimization/reordering of the memory
### Lock-free programming
*  concepts related to lock-free programming, such as Compare-And-Swap (CAS) and the ABA-problem 
*  Lock-free queues can be used for one-way communication with threads that cannot use locks to synchronize access to shared data. 
```cpp
template <class T, size_t N> 
class LockFreeQueue { 
public: 
  LockFreeQueue() : read_pos_{0}, write_pos_{0}, size_{0} { 
    assert(size_.is_lock_free()); 
  } 
  
  auto size() const { return size_.load(); } 
     
  // Writer thread 
  auto push(const T& t) { 
    if (size_.load() >= N) { 
      throw std::overflow_error("Queue is full"); 
    } 
    buffer_[write_pos_] = t; 
    write_pos_ = (write_pos_ + 1) % N; 
    size_.fetch_add(1); 
  } 
     
  // Reader thread 
  auto& front() const { 
    auto s = size_.load(); 
    if (s == 0) { 
      throw std::underflow_error("Queue is empty"); 
    } 
    return buffer_[read_pos_]; 
  } 
     
  // Reader thread 
  auto pop() { 
    if (size_.load() == 0) { 
      throw std::underflow_error("Queue is empty"); 
    } 
    read_pos_ = (read_pos_ + 1) % N; 
    size_.fetch_sub(1); 
  } 
private:   
  std::array<T, N> buffer_{};  // Used by both threads 
  std::atomic<size_t> size_{}; // Used by both threads 
  size_t read_pos_ = 0;        // Used by reader thread 
  size_t write_pos_ = 0;       // Used by writer thread 
};
```
### performance
* Avoid contention
    * sometimes parallel algo may be even slower than a single threaded.
    * locks and atomics are adding overhead and switching off the optimization from the compiler
* Avoid blocking operations
    * if function performs i/o takes more than a few milliseconds -> to implement asynchronously
 * number of threads/cpu core
    * at some point more threads may add even slow down, for intensive tasks that spent much time on waiting there may be used a lot of threads. 
    * for cpu bound tasks, threads >= cpu numbers. controlling number of threads may be done via thread pool
 * thread priorities
    * A thread with high priority is likely to be scheduled more often than threads with lower priorities. 
    * There is currently no way of setting the priority on a thread with the current C++ thread APIs. However, by using std::thread::native_handle, one can get a handle to the underlying operating system thread and use native APIs for setting priorities.
    * priority inversion. when a thread with high priority is waiting to acquire a lock that is currently held by a low priority thread.
 * Thread affinity.  
    * this is a request to the scheduler that some threads should be executed on a particular core if possible, to minimize cache misses(caching).
    * It is not possible to set thread affinity in a portable way with the current C++ APIs, but most platforms support some way of setting an affinity mask on a thread
```cpp
    #include <pthreads> // Non-portable header
auto set_affinity(const std::thread& t, int cpu) {
  cpu_set_t cpuset;
  CPU_ZERO(&cpuset);
  CPU_SET(cpu, &cpuset);
  pthread_t native_thread = t.native_handle(); 
  pthread_set_affinity(native_thread, sizeof(cpu_set_t), &cpuset); 
} 
```
* False sharing
    * when two threads use some data (that is not logically shared between the threads) but happen to be located in the same cache line, if the two threads are executing on different cores and constantly update the variable that resides on the shared cache line. 
    * solution to this problem is to pad each element in the array so that two adjacent elements cannot reside on the same cache line.
    * Since C++17, there is a portable way of doing this using the std::hardware_destructive_interference_size constant defined in <new> in combination with the alignas specifier. 
```cpp
  struct alignas(std::hardware_destructive_interference_size) Element {
   int counter_{};
}; 
 
auto elements = std::vector<Element>(num_threads); 
  //The elements in the vector are now guaranteed to reside on separate cache lines.
```
# Chapter 11. Parallel STL
* how to use the computer's graphical processing unit for computationally heavy tasks.
* Boost Compute library, which exposes the GPU via an interface that resembles the STL
* A parallel algorithm equivalent of a sequential algorithm is algorithmically slower than the sequential. Its benefits come from the ability to spread the algorithms onto several processing units. 
 * how to check if it makes sense to parallelize the task:     A: The time it takes to execute sequentially at one CPU core
    B: The time it takes to execute in parallel, multiplied by the number of cores. If A == B -> parallelizing is good.
 * For example, std::transform() is trivial to parallelize in the sense that each element is processed completely independent of every other. 
 ### Naive implementation std::transform()
    * Divide the elements into chunks corresponding to the number of cores in the computer
    * Execute each chunk in a separate task in parallel
    * Wait for all tasks to finish
 * example
  ```cpp
  template <typename SrcIt, typename DstIt, typename Func>
auto par_transform_naive(SrcIt first, SrcIt last, DstIt dst, Func f) {
  auto n = static_cast<size_t>(std::distance(first, last));
  auto num_tasks = std::max(std::thread::hardware_concurrency(), 1);
  auto chunk_sz = std::max(n / num_tasks, 1);
  auto futures = std::vector<std::future<void>>{};
  futures.reserve(num_tasks); // Invoke each chunk on a separate
// task, to be executed in parallel
  for (size_t task_idx = 0; task_idx < num_tasks; ++task_idx) {
    auto start_idx = chunk_sz * task_idx;
    auto stop_idx = std::min(chunk_sz * (task_idx + 1), n);
    auto fut = std::async([first, dst, start_idx, stop_idx, &f](){
      std::transform(first+start_idx, first+stop_idx, dst+start_idx, f);
    });
    futures.emplace_back(std::move(fut));
  }  
// Wait for each task to finish
  for (auto& fut : futures) { fut.wait();}
}
  ```
### Divide and conquer parallel transform()
* The input range is divided into two ranges; if the input range is smaller than a specified threshold, the range is processed, or else the range is split into two parts:  
    * One part is recursively processed at the calling thread
    * One part is branched to another task recursively processed at that task
* A range is divided recursively for parallel processing
```cpp
template <typename SrcIt, typename DstIt, typename Func> 
auto par_transform(SrcIt first,SrcIt last,DstIt dst,Func f,size_t chunk_sz) { 
  const auto n = static_cast<size_t>(std::distance(first, last));
  if (n <= chunk_sz) { 
    std::transform(first, last, dst, f); 
    return; 
  } 
  const auto src_middle = std::next(first, n/2); 
  // Branch of first part to another task 
  auto future = std::async([=, &func]{ 
    par_transform(first, src_middle, dst, f, chunk_sz); 
  }); 
  // Recursively handle the second part 
  const auto dst_middle = std::next(dst, n/2); 
  par_transform(src_middle, last, dst_middle, f, chunk_sz); 
  future.wait(); 
} 
```
*  generic implementation would be wise to use a large number of tasks rather than trying to figure out the correct number of tasks based on the number of CPU cores on the machine. 
### Implementing parallel std::count_if
```cpp
template <typename It, typename Pred> 
auto par_count_if(It first, It last, Pred pred, size_t chunk_sz) { 
  auto n = static_cast<size_t>(std::distance(first, last)); 
  if (n <= chunk_sz) 
    return std::count_if(first, last, pred);
  auto middle = std::next(first, n/2); 
  auto future = std::async([=, &pred]{ 
    return par_count_if(first, middle, pred, chunk_sz); 
  }); 
  auto num = par_count_if(middle, last, pred, chunk_sz); 
  return num + future.get(); 
} 
```
### Implementing parallel std::copy_if
```cpp
template <typename SrcIt, typename DstIt, typename Pred> 
auto copy_if(SrcIt first, SrcIt last, DstIt dst, Pred pred) { 
  for(auto it = first; it != last; ++it) { 
    if( pred(*it) ) { 
      *dst = *it;
++dst;
    }
  }
  return dst;
} 
```
ERROR -> writing to the same index
* Approach one – Use a synchronized write position
    * very slow with easy function, many calls, because of cache being trashed
    * using an atomic size_t and the fetch_add() member function. Whenever a thread tries to write a new element, it fetches the current index and adds one atomically, thus each value is written to a unique index.
  ```cpp
  template <typename SrcIt, typename DstIt, typename Pred>
auto _inner_par_copy_if_sync(
  SrcIt first,
  SrcIt last,
  DstIt dst,
std::atomic_size_t& dst_idx,
  Pred pred,
  size_t chunk_sz
) -> void {
  auto n = std::distance(first, last);
  if (n <= chunk_sz) {
    std::for_each(first, last, [&](const auto& v) {
      if (pred(v)) {
auto write_idx = dst_idx.fetch_add(1);
*std::next(dst, write_idx) = v;
      }
    });
    return;
  }
  auto middle = std::next(first, n / 2);
  auto future = std::async(
    [first, middle, dst, chunk_sz, &pred, &dst_idx] {
      return _inner_par_copy_if_sync(
first, middle, dst, dst_idx, pred, chunk_sz
      );
    });
_inner_par_copy_if_sync(middle, last, dst, dst_idx, pred, chunk_sz);
  future.wait();
}
 //outer function
template <typename SrcIt, typename DstIt, typename Pred>
auto par_copy_if_sync(SrcIt first,SrcIt last,DstIt dst,Pred p,size_t chunk_sz){
auto dst_write_idx = std::atomic_size_t{ 0 };
  _inner_par_copy_if_sync(first, last, dst, dst_write_idx, p, 
    chunk_sz);
  return std::next(dst, dst_write_idx);
}
  ```
 * Approach two – Split algorithm into two parts
      * Part one – Copy elements in parallel into the destination range
  ```cpp
  template <typename SrcIt, typename DstIt, typename Pred> 
auto par_copy_if_split(SrcIt first,SrcIt last,DstIt dst,Pred pred,size_t chunk_sz){ 
  // Part #1: Perform conditional copy in parallel 
  auto n = static_cast<size_t>(std::distance(first, last)); 
  using CopiedRange = std::pair<DstIt, DstIt>; 
  using FutureType = std::future< CopiedRange >;
  auto futures = std::vector<FutureType>{};
  futures.reserve(n / chunk_sz);
  for (size_t start_idx = 0; start_idx < n; start_idx += chunk_sz) {
    auto stop_idx = std::min(start_idx + chunk_sz, n);
    auto future = std::async([=, &pred] { 
      auto dst_first = dst + start_idx; 
      auto dst_last = std::copy_if(first + start_idx, first + stop_idx, 
        dst_first, pred); 
      return std::make_pair(dst_first, dst_last);
    });
    futures.emplace_back(std::move(future));
  } 
  // To be continued...
  ```
* Part two – Move the sparse range sequentially into a continuous range
```cpp
  // ...continued from above...// Part #2: Perform merge of resulting sparse range sequentially 
  auto new_end = futures.front().get().second; 
  for(auto it = std::next(futures.begin()); it != futures.end(); ++it) { 
    auto chunk_rng = it->get(); 
    new_end=std::move(chunk_rng.first, chunk_rng.second, new_end);
  } 
  return new_end; 
} // end of par_copy_if_split
```
### Parallel STL
* C++17, the STL library has been extended with parallel versions of most, but not all, algorithms. 
* sequentialve version
  ```cpp
  auto loopy_coaster = *std::find(
  roller_coasters.begin(),
  roller_coasters.end(), 
  "loopy"
);
```
* parallel version
  ```cpp
 auto loopy_coaster = *std::find(
 std::execution::par, 
  roller_coasters.begin(),
  roller_coasters.end(),
  "loopy"
); 
  ```
### Execution policies. 
* The execution policy informs the algorithms of how they are allowed to parallelize the algorithm; there are three default execution policies included in the STL parallel extensions.
* In the future, there will probably be libraries extending these policies for certain hardware and conditions
* The execution policies are defined in the header <execution> and reside in the namespace std::execution.
* Sequenced policy. std::execution::seq, makes the algorithm execute sequentially with no parallelism, just as the algorithm would execute if invoked without any execution policy at all. makes sense when working on small collections
 ```cpp
 auto find_largest(const std::vector<int>& v) { 
  auto threshold = 2048; 
  return v.size() < threshold ? 
    *std::max_element(std::execution::seq, v.begin(), v.end()) : 
    *std::max_element(std::execution::par, v.begin(), v.end());
}
```
* Parallel policy,std::execution::par , can be considered the standard execution policy for parallel algorithms. it handles exceptions, meaning that if an exception is thrown during the execution of the algorithm, the exception will be thrown out back on the main thread and the algorithm will break at an unspecified position
```cpp
auto inv_numbers(const std::vector<float>& c, std::vector<float>& out) { 
  out.resize(c.size(), -1.0f);
  auto inversef = [](float denominator){ 
    if(denominator != 0.0f) { return 1.0f/denominator; }
    else throw std::runtime_error{"Division by zero};}
  }; 
  auto p = std::execution::par;
  std::transform(p, c.begin(), c.end(), out.begin(), inversef); 
} 
 
auto test_inverse_numbers() { 
  auto numbers = std::vector<float>{3.0f, 4.0f, 0.0f, 8.0f, 2.0f}; 
  auto inversed = std::vector<float>{}; 
  try { 
    inv_numbers(numbers, inversed); 
  } 
  catch (const std::exception& e) { 
    std::cout << "Exception thrown, " << e.what() << '\n'; 
  } 
  for(auto v: inversed) { std::cout << v << ", "; } 
}                              
```                            
* Parallel unsequenced policy. std::execution::par_unseq , executes the algorithm in parallel like the parallel policy, but with the addition that it may also vectorize the loop using, for example, SIMD instructions if plausible. In addition to the vectorization, it has stricter conditions for the predicates than std::execution::par:(Predicates may not throw, doing so will cause undefined behavior or an instant crash,Predicates may not use a mutex for synchronization, doing so might cause a deadlock )
```cpp
  //example with possible deadlock
auto trees = std::vector<std::string>{"Pine", "Birch", "Oak"};
auto m = std::mutex{};
auto p = std::execution::par_unseq;
std::for_each(p, trees.begin(), trees.end(), [&m](const auto& t){
  auto guard = std::lock_guard<std::mutex>{m};
  std::cout << t << '\n';
});
```
* In other words, when using the std::execution::par_unseq policy you must make sure that the predicate does not throw or acquire a lock.
 
 ### Parallel modifications of algorithm
 * Most algorithms in STL are available as parallel versions straight out the box, but there are some noteworthy changes to std::accumulate and std::for_each, as their original requirements required in-order execution.
 * std::accumulate and std::reduce. The std::accumulate algorithm cannot be parallelized as it requires to be executed in order of the elements, which is not possible to parallelize. Instead, a new algorithm called std::reduce has been added, which works just like std::accumulate with the exception that it is executed un-ordered.
```cpp
auto mice = std::vector<std::string>{"Mickey", "Minnie", "Jerry"}; 
auto acc = std::accumulate(mice.begin(), mice.end(), {}); 
std::cout << acc << '\n'; 
// Prints "MickeyMinnieJerry"
auto red = std::reduce(mice.begin(), mice.end(), {}); 
std::cout << red << '\n'; 
// Possible output "MinnieJerryMickey" or "MickeyMinnieJerry" etc
```
* std::transform_reduce it transforms a range of elements as std::transform and then applies a functor. This accumulates them out of order, like std::reduce:
```cpp
auto mice = std::vector<std::string>{"Mickey","Minnie","Jerry"}; 
auto num_chars = std::transform_reduce( 
  mice.begin(),  
  mice.end(),  
  size_t{0}, 
  [](const std::string& m) { return m.size(); }, // Transform 
  [](size_t a, size_t b) { return a + b; }       // Reduce 
); 
// num_chars is 17
```
* std::for_each is mainly used for applying a functor to a range of elements, it is quite similar to std::transform() although it only processes elements, like this:
```cpp
auto peruvians = std::vector<std::string>{
  "Mario", "Claudio", "Sofia", "Gaston", "Alberto"}; 
std::for_each(peruvians.begin(), peruvians.end(), [](std::string& s) {
  s.resize(1); 
}); 
// Peruvians is now {"M", "C", "S", "G", "A"} 
```
It actually also returns the functor passed into it, which means it can be used like this:
```cpp
auto result_func = std::for_each( 
  peruvians.begin(),  
  peruvians.end(), 
  [all_names = std::string{}](const std::string& name) mutable { 
    all_names += name + " "; 
    return all_names; 
  } 
); 
auto all_names = result_func(""); 
// all_names is now "Mario Claudio Sofia Gaston Alberto ";
```
* Parallelizing an index-based for-loop
```cpp
template <typename Policy, typename Index, typename F>
auto parallel_for(Policy p, Index first, Index last, F f) {
  auto r = make_linear_range<Index>(first, last, last);
  std::for_each(std::move(p), r.begin(), r.end(), std::move(f));
}
```
### Executing STL algorithms on the GPU
* The main API for programming the GPU is OpenGL, although similar functionality is available in DirectX as well.
* At the beginning, there were two types of shader programs, vertex shaders( transform world coordinates into screen space coordinates,) and fragment shaders( lighting calculations/texture lookups just before writing a pixel to the screen).
* Over time, more shader-type programs were introduced and shaders gained more and more low-level options, such as reading/writing raw values from buffers instead of color values from textures.
### Boost Compute
* Boost Compute is well written, vendor independent, and contains almost all STL algorithms. 
* Basic concepts of Boost Compute
    * Device, the equivalent of the actual GPU on which the operations will be executed
    * Context, the context could be considered the gate to the device
    * Queue, a command queue on which you push operations, which are then executed asynchronously via the GPU driver
* GPUs in many cases have their own exclusive memory (although it often uses the standard RAM), all containers handled by Boost Compute must be copied to Boost Compute's designated containers before processing, and back to standard containers for further processing by the CPU.
* OpenCL is the underlying framework used by the Boost Compute library
* OpenCL is maintained by the Khronos group, which also maintains OpenGL
*  OpenCL program uses a C99 syntax that it executes on the GPU (just like the OpenGL shaders). The OpenCL shader is passed from your C++ application as a string containing its source code and compiled by the OpenCL when the application executes.
* boost compute
```cpp
#include <boost/compute.hpp> 
auto main() -> int { 
  // Initialize Boost Compute and OpenCL 
  namespace bc = boost::compute;
  auto device = bc::system::default_device();  
  auto context = bc::context(device);
  auto command_queue = bc::command_queue(context, device);
}
```
* example of calculation of sum of areas of circles, using cpu
```cpp
auto sum_circle_areas_cpu() { 
  constexpr auto n = 1024; 
  auto circles = make_circles(n); 
  auto areas = std::vector<float>(n);
  std::transform(circles.begin(), circles.end(), areas.begin(), 
    circle_area_cpu); 
  auto plus = std::plus<float>{};
  auto area = std::reduce(areas.begin(), areas.end(), 0.0f, plus);
  std::cout << area << '\n'; 
}
```
* how to do the same with gpu
    * inform boost compute about circle struct(This adaption is only necessary if we want to use a custom struct; standard data types such as floats, integers, and so on can be used directly. )
    ```cpp
    BOOST_COMPUTE_ADAPT_STRUCT(Circle, Circle, (x, y, r)); 
    ```
    * implement opencl equivalent of the circle_area_cpu()
    ```cpp
    namespace bc = boost::compute; 
    auto src_code = std::string_view{ 
      "float circle_area_gpu(Circle c) { "
      "  float pi = 3.14f;               "
      "  return c.r * c.r * pi;          "
      "}                                 "
    }; 

    auto circle_area_gpu = bc::make_function_from_source<float(Circle)> ( 
      "circle_area_gpu", src_code.data() 
    );
    ```
    * copy data to and from gpu
* Note that circle_area_gpu() and boost::compute::plus<float> are compiled by the OpenCL driver at runtime, although the binary can be stored for future use.  
* we are using strings for the OpenCL source code. In order to get a little bit more readability, Boost Compute comes with a convenience macro called BOOST_COMPUTE_FUNCTION, which makes strings out of the source code parameter. 
```cpp
  // make_function_from_source
  namespace bc = boost::compute; 
auto circle_area_gpu = 
 bc::make_function_from_source
  <float(Circle)> (
   "circle_area_gpu",
"float circle_area_gpu(Circle c){"
"  float pi = 3.14f;             "
"  return c.r * c.r * pi;        "
"}                               "
);
    
 //vs
 //MACRO BOOST_COMPUTE_FUNCTION
 BOOST_COMPUTE_FUNCTION(
  float, // Return type
  circle_area_gpu, // Name
  (Circle c), // Arg
  {
float pi = 3.14f;
    return c.r * c.r * pi;
  }
);
```
* Implementing the transform-reduction algorithm on the GPU. When implementing the actual transformation, we need to copy the data back and forth. The data structures housed at the GPU are prefixed with gpu_, and data structures housed at the CPU are prefixed with cpu_.Boost Compute has been enough to provide a compute::plus<float> functor equivalent of std::plus, which we use when the areas are reduced:
```cpp
namespace bc = boost::compute; 
auto circle_areas_gpu(bc::context& context, bc::command_queue& q) { 
  // Create a bunch of random circles and copy to the GPU
  const auto n = 1024; 
  auto cpu_circles = make_circles(n); 
  auto gpu_circles = bc::vector<Circle>(n, context);
  bc::copy(cpu_circles.begin(), cpu_circles.end(), gpu_circles.begin(), q); 
  // Transform the circles into their individual areas 
  auto gpu_areas = bc::vector<float>(n, context); 
  bc::transform( 
    gpu_circles.begin(), 
    gpu_circles.end(), 
    gpu_areas.begin(), 
    circle_area_gpu, 
    q 
  ); 
  // Accumulate the circle areas,
// Note that we are writing to a GPU vector of size 1 
  auto gpu_area = bc::vector<float>(1, context); 
  bc::reduce(gpu_areas.begin(), gpu_areas.end(), gpu_area.begin(), q); 
  // Copy the accumulated area back to the cpu 
  auto cpu_area = float{}; 
  bc::copy(gpu_area.begin(), gpu_area.end(), &cpu_area, q); 
  std::cout << cpu_area << '\n'; 
} 
```
* Using predicates with Boost Compute
```cpp
//cpu predicate 
auto less_r_cpu = [](Circle a,Circle b){
  return a.r < b.r;
};
//vs
//gpu predicate
BOOST_COMPUTE_FUNCTION(
  bool, // Return type
  less_r_gpu, // Function Name
  (Circle a, Circle b), // Args
{return a.r < b.r; } // Code
);

//usage
  bc::sort(circles.begin(), circles.end(), less_r_gpu, q);
```
* Box filter. Using a custom kernel in Boost Compute(almost bare OpenCL, rather than Boost Compute.). which applies a box filter of size r to a gray scale image.
```cpp
//cpp box filter
auto box_filter = [](
  int x, 
  int y,
  const auto& src,
  auto& odst, 
  int w,
  int r
) {
 float sum = 0.0f;
 for (int yp=y-r; yp<=y+r; ++yp) { 
  for (int xp=x-r; xp<=(x+r);++xp){
   sum += src[yp * w + xp];
  }
 } 
 float n = ((r*2 + 1) * (r*2 + 1));
 float average = sum / n;
 odst[y*w + x] = average;
};

//vs opencl box filter
auto src_code = std::string_view{
"kernel void box_filter(              "
"  global const float* src,           "
"  global float* odst,                "
"  int w,                             "
"  int r                              "
") {                                  "
"  int x = get_global_id(0);          "
"  int y = get_global_id(1);          "
"  float sum = 0.0f;                  "
"  for (int yp=y-r; yp<=y+r; ++yp){   "
"    for (int xp=x-r; xp<=x+r; ++xp){ "
"      sum += src[yp*w+xp];           "
"    }                                "
"  }                                  "
"  float n=(float)((r*2+1)*(r*2+1));  "
"  float average = sum / n;           "
"  odst[y*w+x] = average;             "
"}                                    "
};
namespace bc = boost::compute;
auto p=bc::program::create_with_source(
  src_code.data(), context
);
p.build();
auto kernel = bc::kernel{ 
  p, "box_filter"
};
```
* usage of boxfilter
```cpp
//cpu
auto box_filter_test_cpu(
 int w,
 int h,
 int r
) {
 using array_t = std::array<size_t,2>;
// Create std vectors
 auto src = std::vector<float>(w*h);
 std::iota(src.begin(),src.end(),0.f);
 auto dst = std::vector<float>(w*h);
 std::fill(res.begin(),res.end(),0.f);
// Make offset and elements
 auto offset = array_t{r,r};
 auto elems = array_t{w-r-r, h-r-r};
// Invoke filter on CPU
 for (int x=0; x < elems[0]; ++x){
   for (int y=0; y < elems[1]; ++y){
     auto xp = x + offset[0];
     auto yp = y + offset[1];
     box_filter(xp,yp,src,dst,w,r);
   }
 }
 return dst;
}
//vs gpu
namespace bc = boost::compute;
auto box_filter_test_gpu(
 int w,
 int h,
 int r,
 bc::context& ctx,
 bc::command_queue& q,
 bc::kernel& kernel
) {
 using array_t = std::array<size_t, 2>;
// Create vectors for GPU
 auto src=bc::vector<float>(w*h, ctx);
 bc::iota(src.begin(),src.end(),0.f,q);
 auto dst=bc::vector<float>(w*h, ctx);
 bc::fill(dst.begin(),dst.end(),0.f,q);
// Make offset and elements
 auto offset = array_t{r,r};
 auto elems = array_t{w-r-r, h-r-r};
// Invoke filter on GPU
 kernel.set_arg(0, src);
 kernel.set_arg(1, dst);
 kernel.set_arg(2, w);
 kernel.set_arg(3, r);
 q.enqueue_nd_range_kernel(
    kernel,
    2,
    offset.data(),
    elems.data(),
    nullptr
 );
// Copy back to cpu
 auto dst_cpu=std::vector<float>(w*h);
 bc::copy(
   dst.begin(),
   dst.end(),
   dst_cpu.begin(),
   q
 );
 return dst_cpu;
}
```
*  GPUs are generally harder to debug than a regular C++ program
```cpp
auto test_kernel(bc::context& ctx, bc::command_queue& q, bc::kernel& k) {
  auto flt_eq = [](float a, float b) {
     auto epsilon = 0.00001f;
     return std::abs(a - b) <= epsilon;
  }; 
  auto cpu = box_filter_test_cpu(2000, 1000, 2); 
  auto gpu = box_filter_test_gpu(2000, 1000, 2, ctx, q, k); 
  auto is_equal = cpu == dst; 
  auto is_almost_equal = std::equal(
    cpu.begin(), cpu.end(), gpu.begin(), flt_eq
  ); 
  std::cout  
    << "is_equal: " << is_equal << '\n' 
    << "is_almost_equal: " << is_float_equal << '\n' 
} 
```
* a computation time in the range of 30x faster on a standard GPU compared to a standard CPU
