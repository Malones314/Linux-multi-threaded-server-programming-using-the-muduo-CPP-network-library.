# Linux多线程服务端编程-使用muduo C++ 网络库
## 多线程编程

### 线程安全的对象生命期管理
可以使用boost库中的shared_ptr和weak_otr完美解决析对象析构时可能存在的race condition(竞态条件)
#### 析构函数遇到多线程的情况
1. 可能出现的几种race condition：

    1.1 即将析构一个对象时，如何知道是否有别的线程正在执行该对象的成员函数

    1.2 如何保证正在执行成员函数期间，对象不会在另一个线程里被析构

    1.3 在调用一个对象的成员函数之前，如何知道这个对象还活着，他的析构函数会不会正好执行到一半

这些问题可以通过使用```shared_ptr```来一劳永逸地解决。

2. 一个线程安全的class应该有以下三个条件：

    2.1 多个线程同时访问时，有正确的表现

    2.2 不论操作系统如何调度这些线程，不论这些线程的执行顺序如何交织

    2.3 调试端代码无需额外的同步或其他协调工作

根据这三个条件，```string```，```vector```，```map```等都不是线程安全的class，因为他们需要在外部加锁才能提供多个线程同时访问。
 
3. 对象的构造要做到线程安全，**唯一**的要求是在构造期间不要泄漏```this```指针，即：
    1. 不要在构造期间注册任何回调
    2. 不要在构造函数中把this传递给跨线程的对象
    3. 即使在构造函数的最后一行也不要泄漏```this```指针 （因为该类可能是基类，先于派生类构造，可能执行完最后一行后还会继续执行派生类的构造）

不要这么做：
```cpp
class Foo : public Observer{
  public:
    Foo( Observer* s){
      s->register_(this); 
    }

    virtual void update();
};
```
应当选择这种做法：
```cpp
class Foo : public Observer{
  public:
    Foo();
    virtual void update();

    void observe( Observer* s){
      s->register_(this);
    }
};
Foo* pFoo = new Foo;
Observer* s = getSubject();
pFoo->observe(s);   //二段式构造（构造函数+initialize()，有时是解决问题的好办法 ）。或者直接写 s->register_(pFoo);
```
4. 析构时很复杂
作为数据成员的mutex不能保护析构。因为析构函数会把mutex成员变量给析构了，所以无法使用mutex实现线程安全。
而且，对于基类对象，当调用到基类析构函数的时候，派生类对象的那部分已经析构好了，基类对象所拥有的mutexlock不能保护整个析构过程。

为了保证析构函数的线程安全性，可以采用以下几种方法：
```cpp
1. 使用互斥锁：在析构函数中使用互斥锁来保护共享资源，以确保同一时间只有一个线程可以访问该资源。
class Foo {
public:
  ~Foo() {
    std::lock_guard<std::mutex> lock(mutex_);
    // 释放资源
  }
private:
  std::mutex mutex_;
};
```
这段代码使用了C++11中的```std::lock_guard<std::mutex>```来保护共享资源```mutex_```，以确保同一时间只有一个线程可以访问该资源。```std::lock_guard```是一个RAII类，它在构造函数中获取锁，在析构函数中释放锁，从而保证了锁的正确使用。

如果你需要在多线程环境下访问共享资源，可以考虑使用```std::lock_guard```来保证线程安全性。

```std::lock_guard```的主要目的是简化互斥量的使用，确保在退出作用域时互斥量自动解锁，从而避免忘记手动解锁导致的死锁或资源泄漏。
当创建```std::lock_guard```对象时，它会自动锁定互斥量。一旦到达```std::lock_guard```对象的作用域结束，**无论是正常执行还是异常退出**，```std::lock_guard```对象的析构函数将自动被调用，释放互斥量的锁。这样确保了互斥量在合适的时候被解锁，避免了手动解锁的疏忽。

```cpp
2. 2. 使用原子操作：在析构函数中使用原子操作来保护共享资源，以确保同一时间只有一个线程可以修改该资源。
class Foo {
public:
  ~Foo() {
    std::atomic<bool>& flag = getFlag();
    while (flag.exchange(true)) {
      std::this_thread::yield();
    }
    // 释放资源
    flag.store(false);
  }
private:
  std::atomic<bool>& getFlag() {
    static std::atomic<bool> flag(false);
    return flag;
  }
};
```
```std::atomic```是C++标准库中提供的一个模板类，用于实现原子操作。它是为了在多线程环境下对共享数据进行安全访问而设计的。
```std::atomic```可以用于实现对特定类型的原子操作，例如读取、写入、交换和比较等。它保证了这些操作的原子性，即这些操作要么完全执行，要么完全不执行，不会发生中间状态的干扰。

```std::atomic```的主要特点和用途如下：

**1**.原子性操作：```std::atomic```提供了一组函数和操作符，可以对共享变量进行原子操作。这意味着多个线程可以同时对同一变量进行读写操作，而不会导致数据竞争或未定义的行为。

**2**.内存顺序：```std::atomic```还提供了一些内存顺序的控制机制，用于指定对共享变量的读写操作在多线程间的可见性和顺序性。可以使用不同的内存顺序来平衡性能和一致性需求。

**3**.支持的类型：```std::atomic```可以用于各种内置数据类型（如整数、指针）以及一些特定的自定义类型。可以通过模板参数来指定要操作的数据类型。

使用```std::atomic```的一般步骤如下：
**1**定义一个std::atomic对象，指定要进行原子操作的数据类型。
**2**使用提供的成员函数或操作符来执行原子操作，如加载（load）、存储（store）、交换（exchange）、比较并交换（compare_exchange_weak/compare_exchange_strong）等。

‘++’不是原子操作，‘++’被分为三步：
1. 将变量放入寄存器
2. 将寄存器中的值加一
3. 将寄存器中的值写回主存

#### 使用shared_ptr/weak_ptr解决问题
##### shared_ptr/weak_ptr 有几个关键点：
1. shared_ptr 控制对象的生命期。shared_ptr是强引用，只要有一个指向x对象的 shared_ptr 存在，该x对象就不会被析构。当最后一个指向对象x的最后一个 shared_ptr 析构或者reset()的时候，x保证被销毁。

2. weak_ptr 不控制对象的生命期，但知道对象是否还活着。如果对象还活着，可以提升(promote)为 shared_ptr，如果对象死了，提升失败，返回一个空的 shared_ptr。提升/lock()是线程安全的。

3. shared_ptr/weak_ptr 在主流平台是原子操作，没有用锁，性能好

4. shared_ptr/weak_ptr 安全级别与string和STL容器一样
即：
  4.1 一个shared_ptr对象实体可以被多个线程同时读取
  4.2 两个shared_ptr对象实体可以被两个线程同时写入（析构算“写”）
  4.3 如果要从多个线程读写同一个shared_ptr对象，要加锁。

### Qt中的线程安全

#### Qt中线程遇到死锁怎么办

**1** 避免死锁：设计良好的线程安全代码应该尽量避免死锁的发生。死锁通常是由于线程之间的循环等待导致的，因此可以通过合理地规划锁的获取顺序来避免死锁。

**2** 使用互斥锁的超时机制：可以在获取锁的过程中设置超时时间，如果在超时时间内无法获取到锁，可以选择放弃或重新尝试获取锁。例如，可以使用```QMutex::tryLock()```方法来尝试获取锁，并根据返回值判断是否成功获取锁。

**3** 使用读写锁（QReadWriteLock）：读写锁允许多个线程同时读取共享数据，但只允许一个线程写入数据。通过合理地使用读写锁，可以减少锁的竞争，降低死锁的可能性。

**4** 使用信号与槽机制代替直接操作：在多线程环境中，使用信号与槽机制可以避免直接在不同线程中操作共享数据，从而减少潜在的死锁问题。

**5** 使用Qt的事件循环：Qt的事件循环机制可以帮助处理线程间的通信和同步，避免了手动管理线程同步的复杂性。可以使用```QEventLoop```和自定义事件来实现线程间的安全通信。

**6** 调试工具：Qt提供了一些调试工具，如Qt Creator的调试器和QtConcurrent中的QFutureWatcher，可以帮助定位和解决线程死锁问题。这些工具可以帮助你跟踪线程的执行流程、查看线程状态以及分析线程间的交互。

##### tryLock
对于 ```QMutex::tryLock()```，它会尝试获取互斥锁并返回获取结果，但它并不会自动将锁置于上锁状态。如果```tryLock()```返回```true```，表示成功获取锁，此时你仍需要手动调用```lock()```或```unlock()```来进行上锁或解锁操作, 确保对共享资源的访问是线程安全的。

```cpp
QMutex mutex;
// 尝试获取互斥锁
if (mutex.tryLock()) {
  // 获取锁成功
  // 在这里执行需要线程安全的操作

  // 手动上锁
  mutex.lock();

  // 继续在上锁状态下执行其他操作

  // 释放锁
  mutex.unlock();
} else {
  // 获取锁失败
  // 处理无法获取锁的情况
  // ...
}
```
也可以使用QMutexLocker类，在构造函数中接受一个QMutex对象作为参数并锁定。

### tips
**1** 在某些操作系统和编译器的实现中，为了保证输出的正确性，标准输出函数可能会使用互斥锁（或其他同步机制）来确保一次只有一个线程能够访问控制台输出。这就导致了多个线程调用 printf 时可能会发生阻塞，因为每个线程需要等待其他线程释放锁才能继续执行输出。而qDebug作为控制台输出，没有这种情况。

#### join和detach
**1. join()** ：调用```join()函数```会**阻塞**当前线程，**直到被调用的线程完成执行**。当一个线程调用```join()函数```时，它会等待被调用的线程执行完成，然后继续执行。如果不调用```join()函数```而直接销毁了拥有线程的```std::thread对象```，程序会终止并抛出```std::terminate异常```。```join()函数```可以用于等待线程完成并获取其返回值（如果有）。

**2. detach()**  ：调用```detach()函数```会**分离**被调用的线程，使其**在后台独立运行**。分离的线程将继续运行，而调用```detach()```的线程可以继续执行其他任务，不再关注分离的线程的状态。当一个线程被分离后，它的资源将由操作系统自行管理。因此，不再可以使用```join()函数```等待分离的线程完成执行或获取其返回值。如果不调用```join()```或```detach()```函数而直接销毁拥有线程的```std::thread对象```，程序会终止并抛出```std::terminate异常```。

**3. join和detach对比** ```join()函数```用于等待线程执行完成并获取其返回值，而```detach()函数```将线程分离，使其在后台独立运行，不再与调用线程相关联。需要注意的是，在一个线程对象上只能调用一次```join()```或```detach()```，否则会抛出```std::system_error异常```。

**4. 两者分别何时使用** 
1. 使用```join()```：

  * 希望**等待线程完成执行并获取其结果时**，应该使用```join()```。通过调用```join()```函数，当前线程会被**阻塞**，直到被调用的线程执行完成。这对于需要等待线程完成后继续执行的情况非常有用，尤其是需要获取线程的返回值时。
  * 希望确保线程的资源得到正确释放时，应该使用```join()```。通过在主线程中调用```join()```，可以确保在销毁```std::thread```对象之前，被调用的线程已经执行完成，从而避免可能的资源泄漏。

2. 使用```detach()```：

  * 希望将线程与创建它的```std::thread```对象分离，使它在**后台独立运行**时，应该使用```detach()```。分离线程意味着主线程不再等待被调用的线程执行完成，而是让它在后台继续执行。
  * 不需要等待线程执行完成或获取其返回值时，可以使用```detach()```。这在某些情况下可以提高程序的效率，特别是当主线程不需要关注线程的状态，并且线程的生命周期可以独立于主线程时。
```cpp
#include <thread>
#include <iostream> 
using namespace std;
void detach_clock(){
  cout << "detach_running" << endl;
}
void join_clock(){
  cout << "join_running" << endl;
}
int main(){
  thread thread_detach( detach_clock);
  thread_detach.detach();
  cout << "main_running" << endl;
  thread thread_join( join_clock);
  thread_join.join();
  cout << "main_running" << endl;
  return 0;
}
//输出结果：
// main_running
// detach_running
// join_running
// main_running
```

#### 线程安全
##### 保证线程安全的几种常见方式
###### 1.互斥量

互斥量（Mutex）是最基本的线程同步机制，通过互斥量可以实现对共享资源的互斥访问。线程在访问共享资源之前，需要先获取互斥量的锁，进入临界区；访问完成后，释放互斥量的锁，允许其他线程进入临界区。互斥量可以保证在同一时刻只有一个线程访问共享资源，从而避免竞态条件（race condition）的发生。以下情况下推荐使用互斥量：

1.  共享资源的读写操作：当多个线程需要同时读取或修改同一个共享资源时，使用互斥量可以确保在同一时刻只有一个线程访问共享资源，避免数据竞争和不一致的情况。互斥量可以实现互斥访问，使得读操作和写操作可以正确地进行。
    
2.  临界区保护：当某段代码（临界区）需要被限制在同一时刻只能有一个线程执行时，可以使用互斥量来实现互斥访问。通过在进入临界区之前获取互斥量的锁，只有一个线程能够进入临界区执行代码，其他线程需要等待锁的释放。
    
3.  数据结构的一致性维护：当多个线程并发地访问和修改数据结构时，需要保证数据结构的一致性。互斥量可以用于对整个数据结构进行加锁，确保在修改过程中不会出现数据损坏或不一致的情况。
    
4.  共享资源的初始化和清理：在多线程环境中，对于共享资源的初始化和清理操作，需要使用互斥量来保护。例如，在多个线程中同时初始化一个全局变量，可以使用互斥量来确保只有一个线程执行初始化操作。
    

需要注意的是，互斥量的使用会引入一定的开销，因为每次访问共享资源都需要进行加锁和解锁操作。因此，在设计和实现中需要合理地选择互斥量的粒度和范围，避免不必要的锁竞争和性能瓶颈。此外，互斥量的正确使用也需要注意死锁（deadlock）和饥饿（starvation）等问题，避免出现线程阻塞或无法继续执行的情况。

###### 2.条件变量

条件变量（Condition Variable）：条件变量用于实现线程间的等待和通知机制，允许线程等待某个条件满足后再继续执行。
在条件变量的使用中，有等待端（wait side）和通知端（signal/broadcast side）两个角色。
条件变量通常与互斥量一起使用，通过互斥量保护共享数据的访问，并使用条件变量来进行线程间的等待和通知操作。条件变量可以用于实现生产者-消费者模型、线程池等多线程场景。以下情况下推荐使用条件变量：

1.  等待特定条件的发生：当线程需要等待某个特定条件的发生时，可以使用条件变量进行等待。线程可以通过条件变量的wait()方法进入等待状态，直到其他线程满足条件并通过条件变量的notify()方法通知等待线程。这种机制可以有效地避免线程轮询和空转，提高系统的性能和资源利用率。
    
2.  生产者-消费者模型：生产者-消费者模型中的生产者线程在生产数据之后，需要通知消费者线程进行消费。使用条件变量可以实现生产者线程等待缓冲区有空闲位置的条件，以及消费者线程等待缓冲区有数据可供消费的条件。当条件满足时，生产者线程通过条件变量的notify()方法通知消费者线程继续执行。
    
3.  线程间的顺序控制：有时候需要确保多个线程按照特定的顺序执行，条件变量可以用于控制线程的执行顺序。通过条件变量的wait()和notify()方法，可以让线程在特定的条件下等待和被唤醒，从而实现线程的有序执行。
    
4.  任务调度和线程池：在任务调度和线程池等场景中，使用条件变量可以控制线程的执行和等待。条件变量可以用于通知线程有新的任务可执行，或者等待所有任务执行完成。通过条件变量的等待和通知机制，可以更灵活地管理线程的调度和资源的分配。
    
需要注意的是，条件变量的使用需要配合互斥量（Mutex）一起使用，以保护共享数据的访问和修改。互斥量用于提供对共享数据的互斥访问，条件变量用于线程间的等待和通知。同时，正确使用条件变量也需要注意避免死锁和饥饿等问题，以及合理地设置等待条件和通知条件，确保线程的正确同步和协作。

条件变量只有一种正确的使用方式：
* 对于wait端：
1. 必须与mutex一起使用，该bool表达式的读写需受此mutex保护
2. 在mutex已经上锁的时候才能调用wait()
3. 把判断bool条件和wait()放到while循环中
```cpp
muduo::MutexLock mutex;
//创建了一个条件变量 cond，并将互斥锁 mutex 与之关联。
muduo::Condition cond(mutex); 
std::deque<int> queue;

int dequeue(){
  MutexLockGuard lock( mutex);
  //检查队列是否为空。如果队列为空，则调用 cond.wait() 进入等待状态。
  //导致当前线程释放互斥锁，并等待条件变量 cond 的通知。
  //一旦有其他线程调用了 cond.notify() 或 cond.notify_all() 
  //通知有可用数据时，当前线程将被唤醒并继续执行。
  while( queue.empty()){  
    cond.wait();  
  }
}
```
* 对于signal/broadcast端：
1. 不一定要在mutex已经上锁的时候调用signal
2. 在signal之前一般要修改bool表达式
3. 修改bool表达式通常需要使用mutex保护(至少full memory barrier)
4. signal和broadcast的区别：
  signal：用于表示资源可用
  broadcast：用于表示状态变化
> Full memory barrier（完全内存屏障），也称为全局内存屏障或总线锁定，是一种CPU指令或操作，用于强制保证在其前后的所有内存访问操作按照严格的顺序进行。它的作用是确保在内存屏障之前的所有读写操作在内存屏障之前完成，且在内存屏障之后的所有读写操作在内存屏障之后开始。
```cpp
void enqueue( int x){
  MutexLockGuard lock(mutex); //在当前作用域中自动获取互斥锁 mutex 
  queue.push_back(x); //这是一个临界区操作，通过获取互斥锁 mutex，
    //确保在同一时间只有一个线程能够修改队列。
  cond.notify();  //发送条件变量 cond 的通知。可以移出临界区之外
}
```
上面的dequeue()和enqueue()实现了一个简单的容量无限的BlockingQueue

###### 3.原子操作
原子操作（Atomic Operation）：原子操作是一种不可分割的操作，可以保证在多线程环境下的原子性。即确保一个操作在执行过程中不会被其他线程中断。原子操作可以用于操作共享变量，而无需使用互斥量或其他同步机制。C++11引入了原子操作的标准库，包括std::atomic和std::atomic\_flag等类型，可以在不使用互斥量的情况下实现线程安全的访问和修改。以下情况下推荐使用原子操作：

1.  简单的读取和写入操作：当对共享变量进行简单的读取和写入操作时，可以使用原子操作来保证线程安全。原子操作能够确保读取和写入操作的原子性，避免了数据竞争和不一致的情况。
    
2.  累加、累减等操作：对于一些常见的计数、统计等操作，原子操作提供了一种高效的方式来进行累加、累减等操作。原子操作能够确保这些操作的原子性，避免了并发情况下的竞态条件。
    
3.  标志位的设置和清除：在多线程环境中，经常需要设置和清除一些标志位来表示某种状态或事件。使用原子操作可以确保对标志位的设置和清除操作的原子性，避免了竞态条件和不一致的情况。
    
4.  避免互斥量的开销：相比于使用互斥量进行线程同步，原子操作具有更轻量级的实现。当只需要进行简单的原子操作而不需要复杂的互斥访问时，原子操作可以避免互斥量的开销，提高系统的性能。
    

需要注意的是，原子操作适用于对单个共享变量进行简单的操作，而不适用于对复杂数据结构或共享资源的操作。在使用原子操作时，需要注意选择适当的原子类型，如std::atomic、std::atomic\_flag等，并确保对共享变量的所有操作都使用原子操作进行。此外，原子操作的使用也需要注意内存模型和内存可见性的问题，以确保线程之间对共享变量的读写操作能够正确地同步和协作。

###### 4.读写锁
读写锁（Reader-Writer Lock）：读写锁是一种特殊的锁机制，允许多个线程同时读取共享资源，但只允许一个线程写入共享资源。读写锁适用于读操作频繁、写操作较少的场景，可以提高并发性能。C++11引入了std::shared\_mutex类型来实现读写锁。在多线程开发中用于解决读多写少的场景下的并发访问问题。以下情况下推荐使用读写锁：

1.  读操作频繁：当多个线程并发地进行读操作，并且读操作占据了大部分的操作时间，而写操作较少时，使用读写锁可以提高并发性能。读写锁允许多个线程同时读取共享资源，从而减少了读操作之间的互斥开销。
    
2.  写操作较少：当写操作相对较少且相对不频繁时，使用读写锁可以提高并发性能。读写锁允许只有一个线程进行写操作，确保写操作的原子性和一致性，而读操作可以并发进行，提高了系统的吞吐量。
    
3.  数据结构的读写分离：当共享数据结构同时被读取和写入时，可以使用读写锁来实现读写分离。读写锁允许多个线程同时进行读操作，而只允许一个线程进行写操作。通过读写锁的控制，可以提高系统的并发性和性能。
    
4.  读操作不影响写操作的一致性：使用读写锁要求读操作是幂等的，即多次读取相同的数据结果不变。如果读操作会影响写操作的一致性，那么使用读写锁可能会导致数据不一致的情况。
    

需要注意的是，读写锁的使用要根据具体的场景和需求来决定。读写锁的机制相对复杂，引入了额外的开销，因此在写操作较为频繁或读操作与写操作之间存在强依赖关系的情况下，使用读写锁可能不会带来性能的提升，甚至会导致更复杂的同步问题。因此，在选择使用读写锁时，需要综合考虑读写操作的比例、并发性能的要求以及数据一致性的需求，合理地选择合适的线程同步机制。

###### 5.并发容器
并发容器（Concurrent Containers）：并发容器是一种特殊的数据结构，设计用于在多线程环境下安全地操作共享数据，进行并发访问和修改。并发容器提供了内部的同步机制，可以保证多线程并发访问时的线程安全性。C++标准库中提供了一些并发容器，如std::vector、std::map、std::queue的并发版本。以下情况下推荐使用并发容器：

1.  并发访问和修改：当多个线程需要同时访问和修改同一个数据结构时，使用并发容器可以提供线程安全的操作。并发容器使用内部的同步机制来保证并发访问时的一致性和正确性，避免数据竞争和不一致的情况。
    
2.  高并发性能要求：并发容器经过特殊设计和优化，能够在高并发场景下提供良好的性能和吞吐量。相比于传统的线程安全容器，使用并发容器可以更好地利用多核处理器和并行计算资源，提高并发性能。
    
3.  简化线程同步：并发容器内部实现了线程同步机制，开发人员无需显式地使用互斥量、条件变量等同步原语进行手动同步。使用并发容器可以简化线程同步的代码，减少出错的机会，提高开发效率。
    
4.  适用于特定的并发场景：某些并发场景下，使用传统的线程安全容器可能无法满足需求，需要更高级的并发容器。例如，使用并发队列可以实现生产者-消费者模型，使用并发哈希表可以实现并发查找和插入等。
     
需要注意的是，并发容器并非适用于所有情况。在某些低并发的场景下，使用普通的线程安全容器或手动同步机制可能更简单、更高效。此外，使用并发容器也需要注意选择适当的并发容器类型，根据具体的需求和数据结构的特点选择合适的并发容器，以获得最佳的性能和线程安全性。
