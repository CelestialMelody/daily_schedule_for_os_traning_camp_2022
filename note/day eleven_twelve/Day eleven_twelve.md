# Day eleven/twelve

- **仅**实现 `FnOnce`，该类型的闭包会拿走**被捕获变量**的所有权

  ```rust
  fn fn_once<F>(func: F)
  where
      F: FnOnce(usize) -> bool + Copy // 闭包 + 实现了Copy特征
  ```

  强制闭包取得捕获变量的所有权，可以在参数列表前添加 `move` 关键字，这种用法通常用于闭包的生命周期大于捕获变量的生命周期时，例如将闭包返回或移入其他线程

- `move` 本身强调的是**闭包如何捕获变量**，实际上使用了 `move` 的闭包依然可能实现了 `Fn` 或 `FnMut` 特征
- **一个闭包实现了哪种 Fn 特征取决于该闭包如何使用被捕获的变量，而不是取决于闭包如何捕获它们**。
- 所有的闭包都自动实现了 `FnOnce` 特征，因此任何一个闭包都至少可以被调用一次
- 没有移出所捕获变量的所有权的闭包自动实现了 `FnMut` 特征
- 不需要对捕获变量进行改变的闭包自动实现了 `Fn` 特征

三个特征的简化版源码

```rust
pub trait Fn<Args> : FnMut<Args> {
    extern "rust-call" fn call(&self, args: Args) -> Self::Output;
}

pub trait FnMut<Args> : FnOnce<Args> {
    extern "rust-call" fn call_mut(&mut self, args: Args) -> Self::Output;
}

pub trait FnOnce<Args> {
    type Output;

    extern "rust-call" fn call_once(self, args: Args) -> Self::Output;
}
```

从特征约束能看出来 `Fn` 的前提是实现 `FnMut`，`FnMut` 的前提是实现 `FnOnce`，因此要实现 `Fn` 就要同时实现 `FnMut` 和 `FnOnce`

闭包作为函数返回值

Rust 要求函数的参数和返回类型，必须有固定的内存大小，例如 `i32` 就是 4 个字节，引用类型是 8 个字节，总之，绝大部分类型都有固定的大小，但是不包括特征，因为特征类似接口，对于编译器来说，无法知道它后面藏的真实类型是什么，因为也无法得知具体的大小。

对于返回特征与闭包，编译器提示我们加一个 `impl` 关键字，返回不同类型，可以用特征对象



---

迭代器

- 惰性初始化
- 迭代器实现了 `Iterator` 特征，最主要的就是实现其中的 `next` 方法
- `next` 方法对**迭代器的遍历是消耗性的**，每次消耗它一个元素，最终迭代器中将没有任何元素，只能返回 `None`



into_iter, iter, iter_mut

- `into_iter` 会夺走所有权
- `iter` 是借用
- `iter_mut` 是可变借用



- `Iterator` 是迭代器特征，只有实现了它才能称为迭代器，才能调用 `next`

- `IntoIterator` 强调的是某一个类型如果实现了该特征，它可以通过 `into_iter`，`iter` 等方法变成一个迭代器



消费者与适配器

- 消费者是迭代器上的方法，依赖 `next` 方法来消费元素

- 迭代器是原来数据的借用

  

- 消费性适配器

  - 迭代器上的某个方法，在其内部调用了 `next` 方法，消费掉迭代器，然后返回一个值
  - 会**拿走迭代器所有权**，**并不意味着也是源数据的借用**

- 迭代器适配器

  - 迭代器适配器，会返回一个新的迭代器，这是实现链式方法调用的关键
  - 与消费者适配器不同，迭代器适配器是惰性的，意味着你**需要一个消费者适配器来收尾，最终将迭代器转换成一个具体的值**

- collect

  - 消费者适配器，使用它可以将一个迭代器中的元素收集到指定类型中
  - 在消费时要指定类型

- 闭包作为适配器参数

  - 迭代器适配器，用于对迭代器中的每个值进行过滤。 它使用闭包作为参数，该闭包的参数是来自迭代器中的值



---

#### [并发和并行](https://course.rs/advance/concurrency-with-threads/concurrency-parallelism.html)

Erlang 之父 Joe Armstrong图片解释：

- **并发(Concurrent)** 是多个队列使用同一个咖啡机，然后两个队列轮换着使用（未必是 1:1 轮换，也可能是其它轮换规则），最终每个人都能接到咖啡
- **并行(Parallel)** 是每个队列都拥有一个咖啡机，最终也是每个人都能接到咖啡，但是效率更高，因为同时可以有两个人在接咖啡
- 还可以对比下串行：只有一个队列且仅使用一台咖啡机，前面哪个人接咖啡时突然发呆了几分钟，后面的人就只能等他结束才能继续接。
- 并发也存在这个问题啊，前面的人发呆了几分钟不接咖啡怎么办？很简单，另外一个队列的人把他推开就行了（



**并发和并行都是对“多任务”处理的描述，其中并发是轮流处理，而并行是同时处理**。

单核心并发

- 操作系统的多线程，正是操作系统多线程 + CPU 核心，才实现了现代化的多任务操作系统。
- 在 OS 级别，多线程负责管理我们的任务队列，你可以简单认为一个线程管理着一个任务队列，然后线程之间还能根据空闲度进行任务调度。我们的程序只会跟 OS 线程打交道，并不关心 CPU 到底有多少个核心，真正关心的只是 OS，当线程把任务交给 CPU 核心去执行时，如果只有一个 CPU 核心，那么它就只能同时处理一个任务。
- 并发的关键在于：**快速轮换处理不同的任务**，给用户带来所有任务同时在运行的假象

多核心并行

- 当 CPU 核心增多到 `N` 时，那么同一时间就能有 `N` 个任务被处理，那么我们的并行度就是 `N`，相应的处理效率也变成了单核心的 `N` 倍（实际情况并没有这么高）

多核心并发

- 当核心增多到 `N` 时，操作系统同时在进行的任务肯定远不止 `N` 个，这些任务将被放入 `M` 个线程队列中，接着交给 `N` 个 CPU 核心去执行，最后实现了 `M:N` 的处理模型，在这种情况下，**并发跟并行时同时在发生的，所有用户任务从表面来看都在并发的运行，其实实际上，同一时刻只有 `N` 个任务能被同时并行的处理**。



- 如果某个系统支持两个或者多个动作的**同时存在**，那么这个系统就是一个并发系统。如果某个系统支持两个或者多个动作**同时执行**，那么这个系统就是一个并行系统。并发系统与并行系统这两个定义之间的关键差异在于 **“存在”** 这个词
- **并行”概念是“并发”概念的一个子集**；并行一定是并发，并发只有在多核时才可能并行



编程语言的并发模型

- 由于操作系统提供了创建线程的 API，因此部分语言会直接调用该 API 来创建线程，因此最终程序内的线程数和该程序占用的操作系统线程数相等，一般称之为**1:1 线程模型**，例如 Rust
- 还有些语言在内部实现了自己的线程模型（绿色线程、协程），程序内部的 M 个线程最后会以某种映射方式使用 N 个操作系统线程去运行，因此称之为**M:N 线程模型**，其中 M 和 N 并没有特定的彼此限制关系。一个典型的代表就是 Go 语言
- 还有些语言使用了 Actor 模型，基于消息传递进行并发，例如 Erlang 语言

- 每一种模型都有其优缺点及选择上的权衡，而 Rust 在设计时考虑的权衡就是运行时(Runtime)。出于 Rust 的系统级使用场景，且要保证调用 C 时的极致性能，它最终选择了尽量小的运行时实现
- 运行时是那些会被打包到所有程序可执行文件中的代码
- 根据每个语言的设计权衡，运行时虽然有大有小（例如 Go 语言由于实现了协程和 GC，运行时相对就会更大一些），但是除了汇编之外，每个语言都拥有它。小运行时的其中一个好处在于最终编译出的可执行文件会相对较小，同时也让该语言更容易被其它语言引入使用
- 而绿色线程/协程的实现会显著增大运行时的大小，因此 Rust 只在标准库中提供了 `1:1` 的线程模型，如果你愿意牺牲一些性能来换取更精确的线程控制以及更小的线程上下文切换成本，那么可以选择 Rust 中的 `M:N` 模型，这些模型由三方库提供了实现，例如大名鼎鼎的 [tokio](https://github.com/tokio-rs/tokio)



---

#### 多线程

风险

- 竞态条件(race conditions)，多个线程以非一致性的顺序同时访问数据资源
- 死锁(deadlocks)，两个线程都想使用某个资源，但是又都在等待对方释放资源后才能使用，结果最终都无法继续执行
- 一些因为多线程导致的很隐晦的 BUG，难以复现和解决



使用 `thread::spawn` 可以创建线程

- 线程内部的代码使用闭包来执行
- `main` 线程一旦结束，程序就立刻结束，因此需要保持它的存活，直到其它子线程完成自己的任务



- `thread::sleep` 会让当前线程休眠指定的时间，随后其它线程会被调度运行，因此就算你的电脑只有一个 CPU 核心，该程序也会表现的如同多 CPU 核心一般，这就是并发！

- 在线程闭包中使用 move



Rust 中线程是如何结束的

- 线程的代码执行完，线程就会自动结束。

- 如果线程中的代码不会执行完呢？那么情况可以分为两种进行讨论：

  - 线程的任务是一个循环 IO 读取，任务流程类似：IO 阻塞，等待读取新的数据 -> 读到数据，处理完成 -> 继续阻塞等待 ··· -> 收到 socket 关闭的信号 -> 结束线程，在此过程中，绝大部分时间线程都处于阻塞的状态，因此虽然看上去是循环，CPU 占用其实很小，也是网络服务中最最常见的模型

  - 线程的任务是一个循环，里面没有任何阻塞，包括休眠这种操作也没有，此时 CPU 很不幸的会被跑满，而且你如果没有设置终止条件，该线程将持续跑满一个 CPU 核心，并且不会被终止，直到 `main` 线程的结束



[多线程的性能](https://course.rs/advance/concurrency-with-threads/thread.html#%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%9A%84%E6%80%A7%E8%83%BD)



`std::sync::Barrier;` 	`join().unwrap()`	`std::sync::{Arc, Mutex, Condvar};` 	`notify_one()` 	`std::sync::Once` 	`call_once()` 

线程屏障(Barrier)：在 Rust 中，可以使用 Barrier 让多个线程都执行到某个点后，才继续一起往后执行

线程局部变量(Thread Local Variable)

用条件控制线程的挂起和执行

只被调用一次的函数



多发送者，单接收者

- 通道`std::sync::mpsc`，其中`mpsc`是*multiple producer, single consumer*的缩写，代表了该通道支持多个发送者，但是只支持唯一的接收者。

  ```rust
  let (tx, rx) = mpsc::channel();
  ```

  - `tx`,`rx`对应发送者和接收者，它们的类型由编译器自动推导: `tx.send(1)`发送了整数，因此它们分别是`mpsc::Sender<i32>`和`mpsc::Receiver<i32>`类型，需要注意，由于内部是泛型实现，一旦类型被推导确定，该通道就只能传递对应类型的值
  - 接收消息的操作`rx.recv()`会阻塞当前线程，直到读取到值，或者通道被关闭
  - 需要使用`move`将`tx`的所有权转移到子线程的闭包中



不阻塞的 try_recv 方法



使用多发送者（以两个子线程为例）

- 多了一个对发送者的克隆`let tx1 = tx.clone();`，然后一个子线程拿走`tx`的所有权，另一个子线程拿走`tx1`的所有权

- 需要所有的发送者都被`drop`掉后，接收者`rx`才会收到错误，进而跳出`for`循环，最终结束主线程
- 这里虽然用了`clone`但是并不会影响性能，因为它并不在热点代码路径中，仅仅会被执行一次
- 由于两个子线程谁先创建完成是未知的，因此哪条消息先发送也是未知的，最终主线程的输出顺序也不确定



消息顺序

- 消息顺序仅仅是因为线程创建引起的，并不代表通道中的消息是无序的，对于通道而言，消息的发送顺序和接收顺序是一致的，满足`FIFO`原则(先进先出)



异步通道`mpsc::channel`

- 无论接收者是否正在接收消息，消息发送者在发送消息时都不会阻塞

同步通道`mpsc::sync_channel`

- 同步通道**发送消息是阻塞的，只有在消息被接收后才解除阻塞**

- 消息缓存：指定同步通道的消息缓存条数
  - 缓存条数为0时，同步通道-阻塞通道，类似于go的阻塞通道（make时，len取值0）
  - 设定为`N`时，发送者就可以无阻塞的往通道中发送`N`条消息，当消息缓冲队列满了后，新的消息发送将被阻塞(如果没有接收者消费缓冲队列中的消息，那么第`N+1`条消息就将触发发送阻塞)

使用异步消息虽然能非常高效且不会造成发送线程的阻塞，但是存在消息未及时消费，最终内存过大的问题。在实际项目中，可以考虑使用一个带缓冲值的同步通道来避免这种风险。



通道关闭

- 所有发送者被drop或者所有接收者被drop后，通道会自动关闭
- 在编译期实现的，完全没有运行期性能损耗



join

- `main` 线程若是结束，则所有子线程都将被终止

- 通过调用 `handle.join`，可以让当前线程阻塞，直到它等待的子线程的结束

- 如果希望等待子线程结束后，再结束 `main` 线程，你需要使用创建线程时返回的句柄的 `join` 方法。





传输多种类型的数据——枚举+match



```rust
use std::sync::mpsc;
fn main() {

    use std::thread;

    let (send, recv) = mpsc::channel();
    let num_threads = 3;
    for i in 0..num_threads {
        let thread_send = send.clone();
        thread::spawn(move || {
            thread_send.send(i).unwrap();
            println!("thread {:?} finished", i);
        });
    }

    // send没有drop 下面的循环不会结束 recv会一直接收信号
    // 在这里drop send...
    // drop(send);

    for x in recv {
        println!("Got: {}", x);
    }
    println!("finished iterating");
}
```

以上代码看起来非常正常，但是运行后主线程会一直阻塞，最后一行打印输出也不会被执行

原因在于： 子线程拿走的是复制后的`send`的所有权，这些拷贝会在子线程结束后被`drop`，因此无需担心，但是`send`本身却直到`main`函数的结束才会被`drop`

通道关闭的两个条件：发送者全部`drop`或接收者被`drop`，要结束`for`循环显然是要求发送者全部`drop`，但是由于`send`自身没有被`drop`，会导致该循环永远无法结束，最终主线程会一直阻塞。

解决办法很简单，`drop`掉`send`即可：在代码中的注释下面添加一行`drop(send);`。



复习与补充

>枚举内存对齐：
>
>Rust 会按照枚举中占用内存最大的那个成员进行内存对齐，这意味着就算你传输的是枚举中占用内存最小的成员，它占用的内存依然和最大的成员相同, 因此会造成内存上的浪费。

>Rust 的所有权规则：
>
>- 若值的类型实现了`Copy`特征，则直接复制一份该值，然后传输过去
>- 若值没有实现`Copy`，则它的所有权会被转移给接收端，在发送端继续使用该值将报错

---

#### 线程同步

共享内存与消息传递（通信）

- 共享内存可以说是同步的灵魂，因为消息传递的底层实际上也是通过共享内存来实现，两者的区别如下：

  - 共享内存相对消息传递能节省多次内存拷贝的成本

  - 共享内存的实现简洁的多

  - 共享内存的锁竞争更多

- 消息传递适用的场景很多——几个主要的使用场景:

  - 需要可靠和简单的(简单不等于简洁)实现时
  - 需要模拟现实世界，例如用消息去通知某个目标执行相应的操作时

  - 需要一个任务处理流水线(管道)时，等等

- 使用共享内存(并发原语)的场景往往就比较简单粗暴：需要简洁的实现以及更高的性能时。

- 消息传递类似一个单所有权的系统：一个值同时只能有一个所有者，如果另一个线程需要该值的所有权，需要将所有权通过消息传递进行转移。而共享内存类似于一个多所有权的系统：多个线程可以同时访问同一个值。



互斥锁 Mutex

- `Mutex`让多个线程并发的访问同一个值变成了排队访问：同一时间，只允许一个线程`A`访问该值，其它线程需要等待`A`访问完成后才能继续
- `Mutex<T>`是一个智能指针，准确的说是`m.lock()`返回一个智能指针`MutexGuard<T>`:
  - 它实现了`Deref`特征，会被自动解引用后获得一个引用类型，该引用指向`Mutex`内部的数据
  - 它还实现了`Drop`特征，在超出作用域后，自动释放锁，以便其它线程能继续获取锁

- 使用方法`m.lock()`向`m`申请一个锁, 该方法会**阻塞当前线程，直到获取到锁**，因此当多个线程同时访问该数据时，只有一个线程能获取到锁，其它线程只能阻塞着等待，这样就保证了数据能被安全的修改
- **`m.lock()`方法也有可能报错**，例如当前正在持有锁的线程`panic`了。在这种情况下，其它线程不可能再获得锁，因此`lock`方法会返回一个错误

- 在使用数据前必须先获取锁
- 在数据使用完成后，必须**及时**的释放锁（drop 或者 使用花括号）



内部可变性

- `Rc<T>`和`RefCell<T>`的结合，可以实现单线程的内部可变性。

- `Mutex<T>`可以支持修改内部数据，当结合`Arc<T>`一起使用时，可以实现多线程的内部可变性



> 忘记释放锁是经常发生的，导致很多用户都热衷于使用消息传递的方式来实现同步，例如 Go 语言直接把`channel`内置在语言特性中，甚至还有无锁的语言，例如`erlang`，完全使用`Actor`模型，依赖消息传递来完成共享和同步。幸好 Rust 的类型系统、所有权机制、智能指针等可以很好的帮助我们减轻使用锁时的负担。



**死锁**(deadlock)

单线程死锁

- 另一个锁还未被释放时去申请新的锁（对同一数据封锁两次）

多线程死锁

- 当我们拥有两个锁，且两个线程各自使用了其中一个锁，然后试图去访问另一个锁时，就可能发生死锁

  ```rust
  // 可能死锁
  use std::{sync::{Mutex, MutexGuard}, thread};
  use std::thread::sleep;
  use std::time::Duration;
  
  use lazy_static::lazy_static;
  lazy_static! {
      static ref MUTEX1: Mutex<i64> = Mutex::new(0);
      static ref MUTEX2: Mutex<i64> = Mutex::new(0);
  }
  
  fn main() {
      // 存放子线程的句柄
      let mut children = vec![];
      for i_thread in 0..2 {
          children.push(thread::spawn(move || {
              for _ in 0..1 {
                  // 线程1
                  if i_thread % 2 == 0 {
                      // 锁住MUTEX1
                      let guard: MutexGuard<i64> = MUTEX1.lock().unwrap();
  
                      println!("线程 {} 锁住了MUTEX1，接着准备去锁MUTEX2 !", i_thread);
  
                      // 当前线程睡眠一小会儿，等待线程2锁住MUTEX2
                      sleep(Duration::from_millis(10));
  
                      // 去锁MUTEX2
                      let guard = MUTEX2.lock().unwrap();
                  // 线程2
                  } else {
                      // 锁住MUTEX2
                      let _guard = MUTEX2.lock().unwrap();
  
                      println!("线程 {} 锁住了MUTEX2, 准备去锁MUTEX1", i_thread);
  
                      let _guard = MUTEX1.lock().unwrap();
                  }
              }
          }));
      }
  
      // 等子线程完成
      for child in children {
          let _ = child.join();
      }
  
      println!("死锁没有发生");
  }
  
  ```

  死锁在这段代码中不是必然发生的，这是由于子线程的初始化顺序和执行速度并不确定，我们无法确定哪个线程中的锁先被执行，因此也无法确定两个线程对锁的具体使用顺序。

- 当一个操作试图锁住两个资源，然后两个线程各自获取其中一个锁，并试图获取另一个锁时，就会造成死锁
- 死锁发生的必然条件：线程 1 锁住了`MUTEX1`并且线程`2`锁住了`MUTEX2`，然后线程 1 试图去访问`MUTEX2`，同时线程`2`试图去访问`MUTEX1`，就会死锁。 因为线程 2 需要等待线程 1 释放`MUTEX1`后，才会释放`MUTEX2`，而与此同时，线程 1 需要等待线程 2 释放`MUTEX2`后才能释放`MUTEX1`，这种情况造成了两个线程都无法释放对方需要的锁，最终死锁。
- 那么为何某些时候，死锁不会发生？原因很简单，线程 2 在线程 1 锁`MUTEX1`之前，就已经全部执行完了，随之线程 2 的`MUTEX2`和`MUTEX1`被全部释放，线程 1 对锁的获取将不再有竞争者。 同理，线程 1 若全部被执行完，那线程 2 也不会被锁，因此我们在线程 1 中间加一个睡眠，增加死锁发生的概率。如果你在线程 2 中同样的位置也增加一个睡眠，那死锁将必然发生!



try_lock

- 与`lock`方法不同，`try_lock`会**尝试**去获取一次锁，如果无法获取会返回一个错误，因此**不会发生阻塞**

  ```rust
  use std::{sync::{Mutex, MutexGuard}, thread};
  use std::thread::sleep;
  use std::time::Duration;
  
  use lazy_static::lazy_static;
  lazy_static! {
      static ref MUTEX1: Mutex<i64> = Mutex::new(0);
      static ref MUTEX2: Mutex<i64> = Mutex::new(0);
  }
  
  fn main() {
      // 存放子线程的句柄
      let mut children = vec![];
      for i_thread in 0..2 {
          children.push(thread::spawn(move || {
              for _ in 0..1 {
                  // 线程1
                  if i_thread % 2 == 0 {
                      // 锁住MUTEX1
                      let guard: MutexGuard<i64> = MUTEX1.lock().unwrap();
  
                      println!("线程 {} 锁住了MUTEX1，接着准备去锁MUTEX2 !", i_thread);
  
                      // 当前线程睡眠一小会儿，等待线程2锁住MUTEX2
                      sleep(Duration::from_millis(10));
  
                      // 去锁MUTEX2
                      let guard = MUTEX2.try_lock();
                      println!("线程1获取MUTEX2锁的结果: {:?}",guard);
                  // 线程2
                  } else {
                      // 锁住MUTEX2
                      let _guard = MUTEX2.lock().unwrap();
  
                      println!("线程 {} 锁住了MUTEX2, 准备去锁MUTEX1", i_thread);
                      sleep(Duration::from_millis(10));
                      let guard = MUTEX1.try_lock();
                      println!("线程2获取MUTEX1锁的结果: {:?}",guard);
                  }
              }
          }));
      }
  
      // 等子线程完成
      for child in children {
          let _ = child.join();
      }
  
      println!("死锁没有发生");
  }
  
  ```

  使用了之前必定会死锁的代码，但是将`lock`替换成`try_lock`，这段代码将不会再有死锁发生



>在 Rust 标准库中，使用`try_xxx`都会尝试进行一次操作，如果无法完成，就立即返回，不会发生阻塞。例如消息传递章节中的`try_recv`以及本章节中的`try_lock`



读写锁 RwLock

- 需要大量的并发读，`Mutex`就无法满足需求了
- 同时允许多个读，但最多只能有一个写（ps：与[引用作用域](https://course.rs/basic/ownership/borrowing.html#%E5%8F%AF%E5%8F%98%E5%BC%95%E7%94%A8%E4%B8%8E%E4%B8%8D%E5%8F%AF%E5%8F%98%E5%BC%95%E7%94%A8%E4%B8%8D%E8%83%BD%E5%90%8C%E6%97%B6%E5%AD%98%E5%9C%A8)**不一样**）
- 读和写不能同时存在
- 读可以使用`read`、`try_read`，写`write`、`try_write`, 在实际项目中，`try_xxx`会安全的多

- 读和写不能同时发生，如果使用`try_xxx`解决，就必须做大量的错误处理和失败重试机制
- 当读多写少时，写操作可能会因为一直无法获得锁导致连续多次失败([writer starvation](https://stackoverflow.com/questions/2190090/how-to-prevent-writer-starvation-in-a-read-write-lock-in-pthreads))
- RwLock 其实是操作系统提供的，实现原理要比`Mutex`复杂的多，因此单就锁的性能而言，比不上原生实现的`Mutex`



- 追求高并发读取时，使用`RwLock`，因为`Mutex`一次只允许一个线程去读取
- 如果要保证写操作的成功性，使用`Mutex`
- 不知道哪个合适，统一使用`Mutex`
- `RwLock`虽然看上去貌似提供了高并发读取的能力，但这个不能说明它的性能比`Mutex`高，事实上`Mutex`性能要好不少，后者**唯一的问题也仅仅在于不能并发读取**

- 一个常见的、错误的使用`RwLock`的场景就是使用`HashMap`进行简单读写，因为`HashMap`的读和写都非常快，`RwLock`的复杂实现和相对低的性能反而会导致整体性能的降低，因此一般来说更适合使用`Mutex`。
- 如果你要使用`RwLock`要确保满足以下两个条件：**并发读，且需要对读到的资源进行"长时间"的操作**，`HashMap`也许满足了并发读的需求，但是往往并不能满足后者："长时间"的操作



用条件变量(Condvar)控制线程的同步

- `notify_one`在此 condvar 上唤醒一个阻塞的线程。如果此条件变量上有阻塞的线程，则它将从其调用中唤醒到 [`wait`](https://rustwiki.org/zh-CN/std/sync/struct.Condvar.html#method.wait) 或 [`wait_timeout`](https://rustwiki.org/zh-CN/std/sync/struct.Condvar.html#method.wait_timeout)。 不会以任何方式缓冲对 `notify_one` 的调用