# Multithreading in PHP: Looking to the Future

## Why this article?

Within [RFC TrueAsync 1.7](https://wiki.php.net/rfc/true_async) a question arises: how will the proposed RFC interact with possible future changes to PHP core? Being able to at least imagine how PHP might evolve is the key to designing the language well for many years ahead. That is why this article exists.

The [TrueAsync](https://github.com/true-async/) project is not just PHP core changes for async, but also other research needed to answer questions like:

* How far can PHP go toward multithreading?
* Are there fundamental limits?
* What core changes would be needed to make multithreading real?
* Which language abstractions could be implemented?

I have not tried to make this text an exhaustive review of every aspect of multithreading in PHP, nor to make it technically exact in all details or understandable to a broad audience. However, I hope the article will be useful to a wide range of PHP developers and set a direction for further discussion.

## History

When a few years ago we needed to add high-volume telemetry to a PHP application, I said it was impossible. But after bumping into `Swoole`’s architecture, it became interesting to test that claim. Could we build an API capable of generating and processing a large amount of data without slowing down the client interaction?

To that end we built an optimized version of `OpenTelemetry` for `PHP` that wrote data in parts, forming a larger chunk to send to an intermediate telemetry server. Compression was used, and `JSON` structures were serialized with `MessagePack`.

The main hypothesis was: if we use single-threaded coroutines, we can build telemetry gradually and then periodically send it to the server on a timer or when a size threshold is reached. The code would be fast because there would be no cross-thread interaction. Was that true?

The experiment showed telemetry slowed the API down by half. The hypothesis failed. But why? At a _conceptual_ level everything seemed logical. `Swoole` by then already made PHP functions non-blocking, so coroutines should have been efficient. Somewhere we made a mistake.

In the second version, telemetry data was collected only during one request and immediately dropped into a job process that aggregated, compressed, and sent telemetry to the server. This solution performed much better. But that should not be the case! Or should it? Data between processes was sent through a `pipe`; it was serialized on one side and deserialized on the other. Even if the pipe lives in memory, OS calls are expensive.

Later we found the reason: there was a lot of telemetry data, so compression consumed significant CPU time compared to handling API requests. Thus, even though `Swoole` coroutines were efficient for I/O, they did not help with CPU-intensive tasks.

This case is one of many showing that single-threaded coroutines do not solve every problem. It also demonstrates how multithreading can complement coroutines, forming a toolkit for solving a wide range of tasks.

## Single-threaded + offload

Offloading CPU-intensive work to a separate process was not a “new invention.” It is part of a broader model that independently appeared in various languages and frameworks and is known as **Single-threaded + offload**.

Imagine one person quickly sorting letters (thousands per hour), while heavy parcels are handled and shipped by other employees in trucks. What happens if the sorter starts hauling parcels? The pile of letters grows to the ceiling.

The `Single-threaded + offload` model separates tasks into two types:

1. **I/O-bound tasks** — reading files, network calls, database access. Most of the time the process waits for the outside world. Thousands of such operations fit into one thread via concurrent async (`coroutines`, `await`).

2. **CPU-bound tasks** — compression, encryption, parsing, computation. Here the CPU works at full power and concurrency will not help; you need real compute power, i.e., more CPU cores.

The model separates these tasks **physically**: the main thread (the `Event Loop`) handles only `I/O`, and `CPU` tasks are offloaded to separate threads or processes (`Workers`).

**Node.js** has always been famous for its single-threaded event loop, ideal for network apps. But when developers tried to process images or compress video right in the request handler, the server turned into a pumpkin. The solution came in the form of `Worker Threads` — separate threads for CPU-heavy work.

**Python** went a similar route. With `asyncio` the language gained a powerful tool for I/O-bound code, but the built-in GIL (Global Interpreter Lock) prevented real CPU parallelism within one process (this problem is solved as of the time of writing). So for blocking operations, `loop.run_in_executor()` and `asyncio.to_thread()` (since Python 3.9) appeared to offload heavy work to a `thread pool` or `process pool`. The event loop stays responsive while computations run in parallel.

**PHP/Swoole** is built on the same architecture: `Request Workers` handle HTTP requests with coroutines, while `Task Workers` tackle heavy computations. Interaction via `UnixSocket` or `pipe` lets one process handle ~100k operations per second.

### Advantages of the model

**1. Resource efficiency**

An `Event Loop` in a single thread can handle thousands of simultaneous I/O operations with minimal overhead. Switching between coroutine tasks is dirt cheap compared to OS-level thread context switches. And `CPU-bound` tasks get real parallelism on different cores — each worker pegs its core at 100% without disturbing neighbors.

**2. Simpler development**

The event loop code does not need mutexes, semaphores, and other “joys” of multithreading. In a single-threaded model, exactly one task runs at a time, so **race conditions** are impossible. Yes, workers run in parallel, but if you follow Shared Nothing, synchronization problems remain localized.

It is worth noting the gulf between the complexity of programming in a multithreaded environment versus a single-threaded concurrent one. Modern languages and frameworks have good reason to favor single-threaded async models over classic multithreading.

**3. Simpler compiler/language**

Async functions in a single-threaded model are much simpler for the compiler and runtime. Much simpler! A good language with multithreading needs its own codegen path. PHP has a major constraint here: part of its code is written in `C`. That prevents efficient bytecode-level optimizations, memory management, and parameter passing that account for multithreading. Go’s language design is not accidentally one of the most complex: proprietary stack, complex GC — all needed for efficient goroutines and channels. We will get back to PHP’s GC later, so do not relax yet!

**4. Manual load distribution**

Developers get a chance to consciously split load between request-handling logic and worker-pool logic. Manual control lets you squeeze the theoretical max out of hardware and code. On the flip side, it is also a drawback.

### Disadvantages of the model

**1. Manual load distribution**

Manual distribution is a double-edged sword. On one hand, a developer can optimize for specific tasks; on the other, they might misjudge what should go into I/O-bound code and what into workers. The result: I/O-bound code overloaded with heavy tasks, degrading server responsiveness and increasing latency.

The model demands that PHP developers be sufficiently skilled, or rely on other skilled developers.

**2. Not for every workload**

The `Single-threaded + offload` model is excellent for web servers, APIs, microservices where the main load is `I/O` operations with databases, filesystems, network calls. But for workloads that need heavy computation at every step — scientific computing, graphics rendering, machine learning — the model may be less effective. In such cases models with full multithreading, where each task can use all available resources, are a better fit.

You might say: we can live with that! We are ready! But is PHP itself ready to become multithreaded?

## Is PHP ready for multithreading?

During TrueAsync development one of the hardest debates was, “Why is there no async in PHP?” Trying to explain why PHP is not ready for multithreading might meet the same hurdles. But first, let us talk about multithreading. Why do we need it? Or better yet: why do we **NOT** need it?

> Multithreading is not needed for parallel code execution.

The idea that multithreading is required for parallel code is ingrained in developers’ minds, just like the idea that black holes “suck in” matter.

Processes handle parallel execution just fine while being isolated from each other (since the 80386 architecture). Processes can interact via `IPC`, and their completion can be tracked with signals (OS events). Why do we need threads, then?

To give an honest answer we would need to travel to the distant past and invite famous minds to explain their decisions firsthand. Edsger Dijkstra, Fernando Corbató, Barbara Liskov, Richard Rashid — I am sure we would have a great talk show. But even if they agreed to an interview, I fear we still would not extract a perfectly honest answer.

It would be incorrect to claim:

> Threads exist so parallel code can share memory without extra tools.

Processes can share memory too, but you need to map a memory region into the address space (an extra tool). Threads share **all memory** of the process by default. Literally, if there is a variable `x` accessible in thread `A`, it is at the same address in thread `B` without any tricks... But no! Multiple threads **cannot** work with a single variable without extra tools.

A more honest statement would be:

> Threads exist to transfer memory between tasks without extra tools.

If threads use memory to pass messages such that only one thread at a time is guaranteed access to a given memory region, that is maximally efficient for both memory and CPU. At the same time, threads deliberately avoid memory regions that would be shared. This model is known as `Shared Nothing`.

So, threads exist to efficiently transfer data between tasks. That is as true as “black holes do not suck.”

## PHP memory model

How does memory work in PHP? We can describe PHP’s abstract memory model like this:
1. Code
2. Data
3. PHP VM state

`PHP` code can already be shared between threads (that was solved with `PHP JIT`).

But the other components are tightly coupled. You cannot just tear them apart. For example, PHP uses a global `object_store` that holds references to all created objects. PHP’s memory manager is built to work with objects from one `PHP VM` and is not oriented toward multithreading. PHP’s **Garbage Collector** (GC) cannot operate on data across threads and even requires a full PHP VM stop because it directly modifies object `refcount`.

Thus PHP is a strictly single-threaded model with a stop-the-world GC.

### Moving PHP VM between threads

PHP uses **Thread-Local Storage (TLS)** to store VM state in each thread. This is a critically important mechanism that provides state isolation between threads in ZTS (Zend Thread Safety) mode.

To get a pointer to the VM state in modern PHP builds, a “static” variable is declared per the C11 standard `__thread` (or `__declspec(thread)` in MSVC). This operation is extremely fast and on `x86_64` boils down to reading from the `FS` or `GS` register.

```asm
      mov rax, QWORD PTR fs:offset
```

Because the `FS/GS` register is unique per thread (ensured by the OS), reading from it always returns the correct pointer to the VM state.

Being able to move VM state between threads could be useful for features like Go-like coroutines or Actors. Modern VMs provide context transfer thanks to custom assembly codegen, passing VM state through CPU registers. This trick is impossible for PHP because under the hood it uses `C` functions, and in `C` there is no way to declare a special context parameter implicitly passed to all functions. So moving PHP VM state between threads will cost some percentage of overall performance.

But what if we could move not all VM state, only the small part required to run code? For example, `PHP Fiber` copies part of the pointers to global structures (`zend_executor_globals`) when switching.

What if we can conceptually split PHP VM into two big parts:
1. PHP VM shared. Classes, functions, constants, ini directives, executable code.
2. PHP VM private. The part of the VM that must be unique.

![PHP VM shared vs private](diagrams/tls-globals-structure.svg)

Some of these structures can be marked `shared` and some `private`, and even `Executor Globals` can be divided into shared and private parts to enable efficient VM state transfer between threads. Global structures of extensions will not lose performance due to indirect access since they already use it.

The problem arises only with structures related to code compilation because PHP has dynamic features like `include/require`, `eval`, and class autoloading. These features prevent an efficient split of VM state into shared and private parts. But if we find a solution, PHP could move part of the VM state between threads with minimal overhead.

## Passing objects between threads

What needs to change in PHP to allow painless passing of objects between threads? How could such a feature be implemented?

Let us try to solve this at the language level. Suppose we have an object of class `SomeObject` in variable `$obj`, and we want to pass it to another thread. Is that possible?

```php
$obj = new SomeObject();

$thread = new Thread(function () use ($obj) {
    echo $obj->someMethod();
});

$thread->join();
```

Since the `SomeObject` instance belongs only to `$obj`, we could safely transfer its address from one thread to another. At the same time the `$obj` variable in the main thread would be destroyed:

```php
$obj = new SomeObject();

$thread = new Thread(function () use ($obj) {
    echo $obj->someMethod();
});

// $obj is undefined here

$thread->join();
```

The code above is essentially a 100% analogue of move semantics that appeared not long ago in C++, exists in Rust, and other languages. This way of passing memory between threads has the following advantages:
1. Safety. Only one thread owns the object.
2. No overhead for copying or serialization.

However, this approach has problems. Consider a tree of categories:

```php
$electronics = new CategoryNode('Electronics');

$categoriesTree = new Tree();
$categoriesTree->addToPath('/products/electronics', $electronics);
$categoriesTree->addToPath('/popular/electronics', $electronics);  // same category!
```

The `$electronics` object appears twice in the tree (`refcount = 2`). But what happens if we try to move `$categoriesTree` to another thread?

For safe movement we must guarantee that every object in the graph has no external references:

```php
$node = new CategoryNode('Electronics');
$categoriesTree = new Tree();
$categoriesTree->addToPath('/products/electronics', $node);

$favourites = [$node];  // external reference!

$thread = new Thread(function () use ($categoriesTree) {
    // $categoriesTree is moved
});

// $favourites[0] now points to memory in another thread
// Dangling pointer!
```

To ensure safe movement we would need:

1. **Full graph traversal** — check all nested objects.
2. **Refcount check** — for every object in the graph.
3. **Identity preservation** — duplicates inside the graph must remain duplicates.

We can imagine several variants of a `deep copy` algorithm. A simple implementation could look like this:

```php
// Deep copy pseudocode
// Source graph in thread A
$node = new Node('A');        // addr: 0x1000
$tree->left = $node;          // addr: 0x1000
$tree->right = $node;         // addr: 0x1000 (same reference)

// Deep copy to thread B (pseudocode with MM)
$copied_map = [];  // hash table: addr_source -> addr_target

function deepCopyToThread(object $obj, Thread $target_thread_mm) 
{
    $source_addr = get_object_address($obj);

    if (isset($copied_map[$source_addr])) {
        return $copied_map[$source_addr];  // already copied!
    }

    // Allocate memory in target thread MM
    $new_addr = $target_thread_mm->allocate(sizeof($obj));
    $copied_map[$source_addr] = $new_addr;

    // Copy object data
    memcpy($new_addr, $source_addr, sizeof($obj));

    // Recursively traverse properties
    foreach ($obj->properties as $prop) {
        if (is_object($prop)) {
            $new_prop_addr = deepCopyToThread($prop, $target_thread_mm);
            // Update pointer in the new object
            update_property($new_addr, $prop, $new_prop_addr);
        }
    }

    return $new_addr;
}

// Result in thread B:
// $newTree->left (addr: 0x2500) === $newTree->right (addr: 0x2500)
// Identity preserved!
```

**Deep copy complexity**: `O(N + E)`, where `N` is the number of objects, `E` is the number of references.
**Space complexity**: `O(N)` — hash table + new objects + recursion stack.

Compared to serialization, this is a faster form of copying because it does not require converting data to and from a transport format. In addition, you can build a hybrid algorithm that moves data with `refcount = 1` and runs `deep copy` for other objects.

We get the result:
1. A PHP developer does not need to think about how objects will be passed to another thread.
2. In the best case memory is moved (`refcount = 1`).
3. In the worst case memory is copied by the deep-copy algorithm with identity preservation (`refcount > 1`).

This sounds good because no language changes are required and multithreading becomes available.

However, at the PHP core level things are not so rosy. To make object movement a reality, PHP needs some mechanism to manage memory across threads. At the moment this is impossible.

## Multithreaded PHP Memory Manager

PHP’s memory manager is in many ways similar to modern allocators like `jemalloc` or `tcmalloc`. The difference is that it lacks an algorithm to correctly free memory from another thread.

Let us look closer:
* An object is created in thread `A`.
* The object is transferred to thread `B` via move (i.e., as is).
* In thread `B` the object is no longer needed and should be freed with `free`.

Each PHP thread has its own `Memory Manager (MM)`. When thread `B` tries to free memory allocated in thread `A`, a problem arises. The memory manager of `B` knows nothing about the memory of `A`, and trying to free it causes an error. Direct access to `A`’s MM structures from thread `B` is a bad idea because it requires synchronization. Modern high-performance multithreaded memory managers solve this with `deferred free`.

The general idea of the `deferred free` algorithm:
1. Thread `B`’s memory manager sees that the object address is unknown.
2. It finds which memory manager owns the pointer and posts a message to a queue that the pointer can be freed.
3. Thread `A`’s memory manager processes the queue and frees the pointers in its own context.

![Cross-thread deallocation](diagrams/cross-thread-free.svg)

Such an algorithm, built on modern lock-free data structures, has high throughput, lets different threads free memory in parallel, and almost does not require locks.

A multithreaded PHP memory manager opens the door for other changes that were previously impossible.

## Shared objects

Being able to pass memory from one thread to another with minimal operations is good, but what if we could create objects designed for sharing between threads from the start?

Many services can be built as immutable objects, and thus should be able to be honestly shared between processes, saving memory and speeding up worker initialization even more.

Unfortunately, `refcount` prevents implementing such objects because it effectively makes all PHP objects mutable! Can we work around this?

### Proxy objects

The first approach is to create proxy objects that refer to real objects stored in a shared memory pool accessible to all threads. Proxy objects would store only an identifier or pointer to the real object plus methods to access its properties and methods. Unfortunately this approach has drawbacks:
* It increases the time to access data and properties.
* It can increase complexity when working with the `Reflection API` and type computation.

On the other hand, PHP already has a mature mechanism for building proxy objects. In some cases, proxy-shared objects are an excellent option, e.g., for a counter table or a Swoole/Table-like data table.

### Shared objects with a GC_SHARE flag

PHP has a built-in mechanism for **immutable** elements via the `GC_IMMUTABLE` flag. It is used for:

- **Interned strings** (`IS_STR_INTERNED`) — string constants that live for the entire PHP run.
- **Immutable arrays** (`IS_ARRAY_IMMUTABLE`) — for example, `zend_empty_array`.
- **Constants in opcache** — compiled code with constant data.

The `GC_IMMUTABLE` flag lets the PHP engine **skip** refcount changes for such structures:

```c
// Zend/zend_types.h
static zend_always_inline void zend_gc_try_addref(zend_refcounted_h *p) {
    if (!(p->u.type_info & GC_IMMUTABLE)) {
        ZEND_RC_MOD_CHECK(p);
        ++p->refcount;
    }
}
```

A similar mechanism can be used to work correctly with `SharedObjects`, for example by defining a `GC_SHARE` flag.

Performance analysis shows that checking the `GC_SHARE` flag adds **+34% overhead** to an isolated `refcount++` operation. In real applications, where refcount operations are a small portion of overall work, the impact should be nearly negligible:

- **Realistic operations** (arrays/objects): +3-9%
- **Real applications**: +0.05-0.5%

This approach solves half the problem; the other half is building a garbage-collection mechanism for such an object. Using an `atomic refcount` is not ideal due to potential performance loss under heavy multi-thread access. A `deferred free` algorithm is likely a better fit.

### Region-based memory

`Region-based memory` is now a popular approach to memory management in languages aimed at the web.

The idea is to allocate memory for specific tasks or threads in separate regions that can be freed in full (or almost in full) when no longer needed. This avoids complexities of per-object memory management and simplifies GC work.

For example, `PHP MM` could guarantee creation of objects in a specific memory region tied to a PHP object. The region’s lifetime equals the object’s lifetime.

When the object is destroyed, the entire region can be freed at once without traversing child elements. And if such an object needs to be “moved” between threads, the `deep copy` step can be skipped.

PHP VM has challenges implementing region-based memory, such as the global object list and opcode caching. But the chances of an efficient implementation are not zero and need further research.

A working region-based memory algorithm enables implementing Actors — special objects with isolated memory.

Actors are the most convenient, powerful, and safe tool for multithreaded programming.

## Coroutines and Threads interaction

From a coroutine’s perspective, `Thread` is an `Awaitable` object. That means a coroutine can wait for a `Thread` result without blocking other coroutines. Many coroutines in one thread can be waiting for heavy tasks, while the servicing thread stays responsive because awaiting a `Thread` does not block the event loop.

```php
use Async\await;
use Async\Thread;

$thread = new Thread(function() {
    // hardware-bound task here
    return 42;
});

$result = await($thread); // Coroutine stops here until the Thread finishes
```

With this approach you can build a chat workflow containing CPU-intensive tasks and simple business logic.

![Thread + Coroutine architecture](diagrams/chat-sequence.svg)

The diagram shows an example architecture. The application has two thread pools: request-handling threads with concurrent multitasking and worker threads for `CPU-intensive` tasks. A coroutine handling one request can fully stop while waiting for a worker thread to do heavy work, then continue processing the request.

```php
use Async\await;
use Async\ThreadPool;

final readonly class ImageDto
{
    public function __construct(
    public int $width,
    public int $height,
    public string $text,
) {}
}

$pool = new ThreadPool(2);
$dto = new ImageDto(
    width: 200,
    height: 200,
    text: 'Hello TrueAsync!'
);

$image = $pool->enqueue(function (ImageDto $dto) {
    $img = imagecreatetruecolor($dto->width, $dto->height);

    $white = imagecolorallocate($img, 255, 255, 255);
    $black = imagecolorallocate($img, 0, 0, 0);

    imagefill($img, 0, 0, $white);
    imagestring($img, 5, 20, 90, $dto->text, $black);

    ob_start();
    imagepng($img);
    imagedestroy($img);
    return ob_get_clean();
}, $dto);

$response->setHeader('Content-Type', 'image/png');
$response->write($image);
$response->end();
```

The coroutine code looks sequential and reads as if `ThreadPool::enqueue` called the callback in the same thread. The DTO moves from one thread to another, and the resulting string is not copied twice.

## Garbage Collector and stateful mode

Upgrading PHP’s memory manager is not the only change needed to improve the language for a multithreaded environment. Without an efficient garbage collector, multithreaded PHP will suffer performance issues and memory leaks from GC cycles.

PHP’s garbage collector uses a combination of two algorithms: **reference counting** as the primary memory management mechanism and **Concurrent Cycle Collection** (Bacon-Rajan, 2001) to detect cyclic references. Reference counting increments and decrements on every assignment, which makes it unsafe in a multithreaded environment without synchronization. Atomic operations for each assignment would create enormous overhead, while lack of synchronization would cause race conditions and leaks. The `Cycle collector`, though called “concurrent,” operates only in one thread context and uses color marking (**PURPLE** → **GREY** → **WHITE/BLACK**) to find cycles, which is also not thread-safe.

The good news is that the current PHP GC implementation will work in a multithreaded environment because it is separate from the memory manager and does not depend on where memory was allocated.

But if PHP wants to enter the multithreaded era of stateful applications, the GC must be adapted to solve:
1. Work **in parallel** in a separate thread without affecting business code.
2. Free resources efficiently as fast as possible.
3. Provide extra tools for detecting/logging memory leaks and telemetry (especially important for long-running apps!).

The `Cycle Collection` algorithm can be adapted for multithreading by processing references in a separate thread, which would increase overall responsiveness of a PHP application. That might be enough to start!

## Actors

Of course, `ThreadPool` and the ability to pass objects between threads are useful, but they require developer attention, skill, and effort. There is a better abstraction for multithreaded programming that hides the complexity of threads and memory while fitting business logic perfectly: **Actors**.

Actors are a model of concurrent parallel programming in which the basic unit of computation is an **actor**.

Each actor:
- Has its own isolated state.
- Processes messages sequentially.
- Interacts with other actors only via messages.
- Can run in a separate thread.

You can think of an actor like an object, making it possible to use familiar OOP paradigms in multithreaded PHP programming.

Imagine a chat server with many rooms. Each room is a separate object.

```php
use Async\Actor;

class ChatRoom extends Actor
{
    private array $messages = [];
    private string $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    public function postMessage(string $user, string $text): void
    {
        $this->messages[] = [
            'user' => $user,
            'text' => $text,
            'time' => time()
        ];
    }

    public function getMessages(): array
    {
        return $this->messages;
    }
}

spawn(function() {
   $room = new ChatRoom('general');
   $room->postMessage('Alice', 'Hello!');  // Runs in another thread, blocks coroutine!
   $messages = $room->getMessages();       // Runs in another thread, blocks coroutine!
   echo json_encode($messages);
});
```

Objects of class `ChatRoom` are special. Their data and PHP VM state are localized so they can easily move between threads. Each method of `ChatRoom` runs separately in its own thread, but only one thread can execute methods of a given actor at a time.

From the language semantics perspective, the base `Actor` class defines how the PHP VM and memory manager should work so `ChatRoom` objects can safely run in separate threads. That is, the class type “stores” not only information about methods and properties but also how the memory manager and GC should behave for such objects. You can see a similar approach in languages like `Rust` and `C++`. The upside is that it requires no syntax changes and fits within existing OOP philosophy.

The example looks like normal sequential code inside a coroutine. But since `postMessage` and `getMessages` run in another thread, they are not invoked directly. The coroutine sends a **message** to the actor’s queue, moves to a waiting state, and resumes only when the actor executes the method in another thread and returns a result.

None of this contradicts familiar PHP OOP because the `Actor` class overrides the magic `__call` method:

```php
class Actor 
{
    private $threadPool;

    public function __call(string $name, array $arguments): mixed
    {
        if(current_thread_id() === $this->threadPool->getThreadIdForActor($this)) {
            // Run directly if we are in the same thread
            return $this->$name(...$arguments);
        }
    
        // Otherwise enqueue the call
        return $this->threadPool->enqueueActorMethod($this, $name, $arguments);
    }
}
```

`enqueueActorMethod` puts the `postMessage` call into the actor’s queue, subscribes to the result event, and calls `Async\suspend()` to stop the coroutine.

Actor code runs sequentially, solving data-race issues and making multithreaded development transparent for the programmer.

Parallelism is achieved because each `ChatRoom` actor can execute in a separate thread:

```php
spawn(function() {
   $room = new ChatRoom('room1');
   $room->postMessage('Alice', 'Hello!');
   $messages = $room->getMessages();
   echo json_encode($messages);
});

spawn(function() {
   $room = new ChatRoom('room2');
   $room->postMessage('Bob', 'Hi there!');
   $messages = $room->getMessages();
   echo json_encode($messages);
});
```

Code of different `ChatRoom` objects can run in parallel in different threads because each actor has its own execution thread, unique PHP VM state, and memory.

Let us create **100 chat rooms**:

```php
use Async\Actor;

$rooms = [
    'general' => new ChatRoom('general'),
    'random'  => new ChatRoom('random'),
    'tech'    => new ChatRoom('tech'),
    // ... another 97 rooms
];

// Coroutine to handle requests
HttpServer::onRequest(function(Request $request, Response $response) use ($rooms) {
   // Handle HTTP request
   $roomName = $request->getQueryParam('room');
   $room = $rooms[$roomName] ?? null;
   
   if (!$room) {
      $response->setStatus(404);
      $response->write('Room not found');
      $response->end();
      return;
   }
   
   // Calls look synchronous but run in another thread!
   $room->postMessage($request->getQueryParam('user'), $request->getQueryParam('text'));
   $messages = $room->getMessages();
   
   $response->setHeader('Content-Type',  'application/json');  
   $response->write(json_encode($messages));
   $response->end();
});
```

Each chat room processes messages sequentially yet in parallel relative to other rooms.

Actors require no mutexes, locks, complex synchronization, or manual interaction with the thread pool. They provide a ready high-level solution for parallelizing tasks.

If one chat room needs to send a message to another, that is possible because actors are `SharedObject`s and can interact across threads:

```php
class Rooms extends Actor
{
    private array $rooms = [];
    
    public function __construct(string ...$roomNames)
    {
       foreach ($roomNames as $name) {
           $this->rooms[$name] = new ChatRoom($name);
       }
    }
    
    public function broadcastMessage(string $fromRoom, string $user, string $text): void
    {
        foreach ($this->rooms as $name => $room) {
            if ($name !== $fromRoom) {
                // Non-blocking call
                $room->postMessageAsync($user, $text);
            }
        }
    }
}

spawn(function() {
   $rooms = new Rooms('general', 'room1', 'room2', 'room3');
   $rooms->broadcastMessage('general', 'Alice', 'Hello!');
});

```        
    
### Internal actor architecture

PHP VM guarantees that all objects inside an actor:
* either belong only to that actor and are allocated in its unique region
* or belong to and are moved from other regions or threads
* or are another SharedObject or another actor

The memory manager ensures that all memory operations inside actor methods are automatically bound to it through the memory region directly associated with the actor.

To execute methods, an `MPMC` message queue is used and unwound by a `Scheduler`. It is the scheduler that distributes CPU time among actors, providing concurrent and parallel execution.

![Actor model architecture](diagrams/actor-message-flow.svg)

## Conclusion

As of today, within the TrueAsync project we already have an experimental multithreaded memory manager and an API for creating threads. The amount of work needed to refactor PHP VM state to safely move its private part between threads looks realistic and doable.

We can confidently say that a safe and reliable multithreaded PHP is not a crazy dream but an achievable goal.
