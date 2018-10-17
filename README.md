# BlockingCollection
BlockingCollection is a C++11 thread safe collection class that provides the following features:
- Implementation of classic Producer/Consumer pattern (i.e. condition variable, mutex);   
- Concurrent adding and taking of items from multiple threads.
- Optional maximum capacity.
- Insertion and removal operations that block when collection is empty or full.
- Insertion and removal "try" operations that do not block or that block up to a specified period of time.
- Insertion and removal 'bulk" operations that allow more than one element to be added or taken at once. 
- Priority-based insertion and removal operations. 
- Encapsulates any collection type that satisfy the ProducerConsumerCollection requirement.
- [Minimizes](#performance-optimizations) sleeps, wake ups and lock contention by managing an active subset of producer and consumer threads.
- Pluggable condition variable and lock types.
- Range-based loop support.
## Bounding and Blocking Support
BlockingCollection<T> supports bounding and blocking. Bounding means that you can set the maximum capacity
of the collection. Bounding is important in certain scenarios because it enables you to control the maximum
size of the collection in memory, and it prevents the producing threads from moving too far ahead of the consuming threads. 
  
Multiple threads or tasks can add elements to the collection, and if the collection reaches its specified maximum capacity, the producing threads will block until an element is removed. Multiple consumers can remove elements, and if the collection becomes empty, the consuming threads will block until a producer adds an item. A producing thread can call the complete_adding method to indicate that no more elements will be added. Consumers monitor the is_completed property to know when the collection is empty and no more elements will be added. The following example shows a simple BlockingCollection with a bounded capacity of 100. A producer task adds items to the collection as long as some external condition is true, and then calls complete_adding. The consumer task takes items until the is_completed property is true.  
```C++
// A bounded collection. It can hold no more 
// than 100 items at once.
BlockingCollection<Data*> collection(100);

// a simple blocking consumer
std::thread consumer_thread([&collection]() {

  while (!collection.is_completed())
  {
      Data* data;
      
      // take will block if there is no data to be taken
      auto status = collection.take(data);
      
      if(status == BlockingCollectionStatus::Ok)
      {
          process(data);
      }
      
      // Status can also return BlockingCollectionStatus::Completed meaning take was called 
      // on a completed collection. Some other thread can call complete_adding after we pass 
      // the is_completed check but before we call take. In this example, we can ignore that
      // status since the loop will break on the next iteration.
  }
});

// a simple blocking producer
std::thread producer_thread([&collection]() {

  while (moreItemsToAdd)
  {
      Data* data = GetData(data);
      
      // blocks if collection.size() == collection.bounded_capacity()
      collection.add(data);
  }
  
  // let consumer know we are done
  collection.complete_adding();
});
```
## Timed Blocking Operations
In timed blocking try_add and try_take operations on bounded collections, the method tries to add or take an item. If an item is available it is placed into the variable that was passed in by reference, and the method returns Ok. If no item is retrieved after a specified time-out period the method returns TimedOut. The thread is then free to do some other useful work before trying again to access the collection.
```C++
  BlockingCollection<Data*> collection(100);
  
  Data *data;
  
  // if the collection is empty this method returns immediately
  auto status = collection.try_take(data);
  
  // if the collection is still empty after 1 sec this method returns immediately
  status = collection.try_take(data, std::chrono::milliseconds(1000));
  
  // in both case status will return BlockingCollectionStatus::TimedOut if 
  // try_take times out waiting for data to become available
```
## Bulk Operations
BlockingCollection's add and take operations are all thread safe. But it accomplishes this by using a mutex. To minimize mutex contention when adding or taking elements BlockingCollection supports bulk operations. It is usually much cheaper to acquire the mutex and then to add or take a whole batch of elements in one go, than it is to acquire and release the mutex for every add and take.
```C++
BlockingCollection<Data*> collection(100);

std::array<Data*, 20> arr;

size_t taken;

auto status = collection.try_take_bulk(arr, arr.size(), taken);

// try_take_bulk will update taken with actual number of items taken
```
## Specifying the Collection Type
When you create a BlockingCollection object, you can specify not only the bounded capacity but also the type of collection to use. 
For example, you could specify a ```QueueContainer<T>``` object for first in, first out (FIFO) behavior, or a ```StackContainer<T>``` object for last in, first out (LIFO) behavior. You can use any collection class that supports the ProducerConsumerCollection requirement. The default collection type for BlockingCollection is ```QueueContainer<T>```. The following code example shows how to create a BlockingCollection of strings that has a capacity of 1000 and uses a ```StackContainer<T>```
```C++
BlockingCollection<std::string, StackContainer<std::string>> stack(1000);
```
Type aliases are also available:
```C++
BlockingQueue<std::string> blocking_queue;
BlockingStack<std::string> blocking_stack;
```
## Priority based Insertion and Removal
PriorityBlockingCollection offers the same functionality found in BlockingCollection<T, PriorityQueue>.
But the add/try_add methods add items to the collection based on their priority - (0 is lowest priority).

FIFO order is maintained when items of the same priority are added consecutively. And the take/try_take methods
return the highest priority items in FIFO order.

In addition, PriorityBlockingCollection adds additional methods (i.e. take_prio/try_take_prio) for taking
the lowest priority items.

PriorityBlockingCollection's default priority comparer expects that the objects being compared have
overloaded < and > operators. If this is not the case then you can provide your own comparer implementation
like in the following example.
```C++
struct PriorityItem {
  PriorityItem(int priority) : Priority(priority) 
  {}
  
  int Priority;
};

class CustomComparer {
public:
  CustomComparer() {
  }

  int operator() (const PriorityItem &item, const PriorityItem &new_item) {
    if (item.Priority < new_item.Priority)
      return -1;
    else if (item.Priority > new_item.Priority)
      return 1;
    else
      return 0;
  }
};

using CustomPriorityContainer = PriorityContainer<PriorityItem, CustomComparer>;

PriorityBlockingCollection<PriorityItem, CustomPriorityContainer> collection;
```
## Range-based for loop Support
BlockingCollection provides an iterator that enables consumers to use ```for(auto item : collection) { ... }```to remove items until the collection is completed, which means it is empty and no more items will be added. For more information, see
```C++
BlockingCollection<Data> collection(100);

// a simple blocking consumer using range-base loop
std::thread consumer_thread([&collection]() {
        
    for(auto data : collection) {
        process(data);
    }    
});
```
## ProducerConsumerCollection Requirement  
In order for a container to be used with the BlockingCollection<T> it must meet the ProducerConsumerCollection requirement.
The ProducerConsumerCollection requires that all the following method signatures must be supported:
  
- size_type size()
- bool try_add(const value_type& element)
- bool try_add(value_type&& element)
- bool try_take(value_type& element)
- template <typename... Args> bool try_emplace(Args&&... args)

BlockingCollection currently supports three containers:

- QueueContainer
- StackContainer
- PriorityContainer

## Performance Optimizations
BlockingCollection can behave like most condition variable based collections. That is, it will by default issue a signal each time a element is added or taken from its underlying Container. But this approach leads to poor application scaling and performance. 

So in the interest of performance BlockingCollection can be configured to maintain a subset of active threads that are currently adding and taking elements. This is important because it allows BlockingCollection not to have to issue a signal each time an element is added or taken. Instead in the case of consumers it issues a signal only when an element is taken and there are no active consumers, or when the Container's element count starts to grow beyond a threshold level. And in the case of producers, BlockingCollection will issue a signal only when an element is added and there are no active producers or when the Container's available capacity starts to grow beyond a threshold level. 

In both cases, this approach greatly improves performance and makes it more predictable.

Two strategy classes are responsible for implementing the behavior just described.

1. NotEmptySignalStrategy
   - implements conditions under which a "not empty" condition variable should issue a signal
   
2. NotFullSignalStrategy
   - implements conditions under which a "not full" condition variable should issue a signal

### NotEmptySignalStrategy
This strategy will return true under two conditions.

1. All consumers are currently not active (i.e. waiting)
2. Number of active consumers < total consumers AND number of item in collection per active consumer is > threshold value
```C++
template<size_t ItemsPerThread = 16> struct NotEmptySignalStrategy {

    bool should_signal(size_t active_workers, size_t total_workers, size_t item_count, size_t /*capacity*/) const {
        return active_workers == 0 || (active_workers < total_workers && item_count / active_workers > ItemsPerThread);
    }
};
```
### NotFullSignalStrategy
This strategy will return true under two conditions.

1. All producers are currently not active (i.e. waiting)
2. Number of active producers < total producers AND current available capacity per active producer is > threshold value
```C++
template<size_t ItemsPerThread = 16> struct NotFullSignalStrategy {

    bool should_signal(size_t active_workers, size_t total_workers, size_t item_count, size_t capacity) const {
        return (active_workers == 0 || (active_workers < total_workers && (capacity - item_count) / active_workers > ItemsPerThread));
    }
};
```
Note that in both strategies the threshold value (i.e. ItemsPerThread) can be specified. And that completely new strategies can be used for both "no empty" and "not full" use cases by creating a new strategy that implements the required method signature.
```C++
bool should_signal(size_t active_workers, size_t total_workers, size_t item_count, size_t capacity)
```
### Attaching/Detaching Consumers & Producers
In order for BlockingCollection to maintain a subset of active threads it exposes attach_producer and attach_consumer member function. The calling thread can call either of those functions to attach itself to the BlockingCollection as either a producer or consumer respectively. Note that the thread should remember to detach itself in both cases.
```C++
BlockingCollection<Data> collection(100);
  
std::thread consumer_thread([&collection]() {

  collection.attach_consumer();
  
  int item;
  
  for(int i = 0; i < 10; i++)
    collection.take(item);
    
  collection.detach_consumer();
});
```
### Consumer & Producer Guards
In order to mitigate forgetting to attach or detach from a BlockingCollection the BlockingCollection Guard classes (i.e. ProducerGuard<T> and ConsumerGuard<T>) can be used for this purpose. Both Guard classes are a RAII-style mechanism for attaching a thread to the BlockingCollection and detaching it when the thread terminates. As well as in exception scenarios.
  
In the following examples, ConsumerGuard and ProducerGuard automatically attach and detach the std::threads to the BlockingCollection.
```C++
BlockingCollection<int> collection;
  
std::thread consumer_thread([&collection]() {

  ConsumerGuard<BlockingCollection<int>> Guard(collection);
  
  int item;
  
  for(int i = 0; i < 10; i++)
    collection.take(item);
});
```
```C++
std::thread producer_thread([&collection]() {

  ProducerGuard<BlockingCollection<int>> Guard(collection);

  for(int i = 0; i < 10; i++)
    collection.add(i+1);
});
```
## Pluggable Condition Variable and Lock Types
The BlockingCollection class by default will use std::condition_variable and std::mutex classes. But those two synchronization primitives can be overridden by specializing ConditionVarTraits.
```C++
template <typename ConditionVarType,  typename LockType>
struct ConditionVarTraits;

template <>
struct ConditionVarTraits<std::condition_variable, std::mutex>
{
    static void initialize(std::condition_variable& cond_var) {
    }

    static void signal(std::condition_variable& cond_var) {
        cond_var.notify_one();
    }

    static void broadcast(std::condition_variable& cond_var) {
        cond_var.notify_all();
    }

    static void wait(std::condition_variable& cond_var, std::unique_lock<std::mutex>& lock) {
        cond_var.wait(lock);
    }

    template<class Rep, class Period> static bool wait_for(std::condition_variable& cond_var, std::unique_lock<std::mutex>& lock, const std::chrono::duration<Rep, Period>& rel_time) {
        return std::cv_status::timeout == cond_var.wait_for(lock, rel_time);
    }
};
```
In the following example, ConditionVarTraits is specialized to use Win32 CONDITION_VARIABLE and SRW_LOCK.

Note that Win32's SRWLOCK synchronization primitive requires a wrapper class so that it can meet the BasicLockable requirements needed by std::unique_lock.
```C++
class WIN32_SRWLOCK {
public:
    WIN32_SRWLOCK() {
        InitializeSRWLock(&srw_);
    }

    void lock() {
        AcquireSRWLockExclusive(&srw_);
    }

    void unlock() {
        ReleaseSRWLockExclusive(&srw_);
    }

    SRWLOCK& native_handle() {
        return srw_;
    }
private:
    SRWLOCK srw_;
};

template <>
struct ConditionVarTraits<CONDITION_VARIABLE, WIN32_SRWLOCK>
{
    static void initialize(CONDITION_VARIABLE& cond_var) {
      InitializeConditionVariable(&cond_var);
    }

    static void signal(CONDITION_VARIABLE& cond_var) {
      WakeConditionVariable(&cond_var);
    }

    static void broadcast(CONDITION_VARIABLE& cond_var) {
      WakeAllConditionVariable(&cond_var);
    }

    static void wait(CONDITION_VARIABLE& cond_var, std::unique_lock<WIN32_SRWLOCK>& lock) {
      SleepConditionVariableSRW(&cond_var, &lock.mutex()->native_handle(), INFINITE, 0);
    }

    template<class Rep, class Period> static bool wait_for(CONDITION_VARIABLE& cond_var, std::unique_lock<WIN32_SRWLOCK>& lock, const std::chrono::duration<Rep, Period>& rel_time) {
    
      DWORD milliseconds = static_cast<DWORD>(rel_time.count());

      if (!SleepConditionVariableSRW(&cond_var, &lock.mutex()->native_handle(), milliseconds, 0)) {
        if (GetLastError() == ERROR_TIMEOUT)
          return true;
      }
      return false;
    }
};
```
## Condition Variable Generator
The BlockingCollection class uses the ConditionVariableGenerator template to generate the condition variable and lock types it will use. In addition, the template also generates the strategy classes.
```C++
template<typename ThreadContainerType, typename NotFullSignalStrategy, typename NotEmptySignalStrategy, typename ConditionVarType, typename LockType> 
struct ConditionVariableGenerator {

    using NotFullType = ConditionVariable<ThreadContainerType, NotFullSignalStrategy, ConditionVarType, LockType>;
    using NotEmptyType = ConditionVariable<ThreadContainerType, NotEmptySignalStrategy, ConditionVarType, LockType>;

    using lock_type = LockType;
};
```
By default, the BlockingCollection class will use the following ConditionVariableGenerator type alias.
```C++
using StdConditionVariableGenerator = ConditionVariableGenerator<ThreadContainer<std::thread::id>, NotFullSignalStrategy<16>, NotEmptySignalStrategy<16>, std::condition_variable, std::mutex>;
```
But it can easily be replaced by something else such as the following.
```C++
using Win32ConditionVariableGenerator = ConditionVariableGenerator<ThreadContainer<std::thread::id>, NotFullSignalStrategy<16>, NotEmptySignalStrategy<16>, CONDITION_VARIABLE, WIN32_SRWLOCK>;
```
A custom condition variable generator can be used like so:
```C++
BlockingCollection<int, QueueContainer<int>, Win32ConditionVariableGenerator> collection;
```
## License
BlockingCollection uses the GPLv3 license that is available [here](https://github.com/CodeExMachina/BlockingCollection/blob/master/LICENSE).
