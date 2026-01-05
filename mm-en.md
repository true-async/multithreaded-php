# Multithreading in PHP: Looking to the Future

## Why this article?

Within [RFC TrueAsync 1.7](https://wiki.php.net/rfc/true_async) the question arises: how will the proposed RFC interact with possible future changes in PHP core? Having at least a sense of where PHP might go is key to designing the language well for many years. That is why this article exists.

The [TrueAsync](https://github.com/true-async/) project is not just PHP core changes for async, but also other research needed to answer questions like:

* How far can PHP move toward multithreading?
* Are there fundamental limits?
* What core changes might be needed to make multithreading real?
* Which language abstractions could be implemented?

I did not try to make this text an exhaustive review of every aspect of multithreading in PHP, nor to make it technically exact in every detail or understandable to a broad audience. Still, I hope it will be useful to many PHP developers and set a direction for further discussion.

## History

Several years ago, when we needed to add high-volume telemetry to a PHP application, I said it was impossible. But after seeing `Swoole`’s architecture, I wanted to test that claim. Could we create an API that generates and processes a large amount of data without slowing down the client?

We built an optimized `OpenTelemetry` for PHP that wrote data in parts, collecting it into large chunks for sending to an intermediate telemetry server. Data was compressed, and `JSON` structures were serialized with `MessagePack`.

The main hypothesis was: if we use single-threaded coroutines, we can build telemetry gradually and periodically send it to the server on a timer or when a size threshold is reached. The code should be fast because there is no cross-thread interaction. Is that true?

The experiment showed telemetry cut API throughput in half. The hypothesis failed. But why? At a _conceptual_ level everything seemed logical. `Swoole` already made PHP functions non-blocking, so coroutines should have been efficient. Somewhere we made a mistake.

In the second version telemetry was collected only during one request and immediately dropped into a job process that aggregated, compressed, and sent telemetry to the server. This solution performed much better. But that should not be the case! Or should it? Data between processes was sent through a `pipe`; it was serialized on one side and deserialized on the other. Even if the pipe lives in memory, OS calls are expensive.

Later we found the reason: there was a lot of telemetry, so compression consumed significant CPU time relative to handling API requests. Thus, even though `Swoole` coroutines were efficient for I/O, they did not help with CPU-intensive tasks.

This case is one of many showing that single-threaded coroutines do not solve every problem. It also shows how multithreading can complement coroutines, creating a toolkit for a broad range of problems.

## Single-threaded + offload

Offloading CPU-intensive work to a separate process was not a “new invention.” It is part of a broader model that independently appeared in different languages and frameworks and is known as **Single-threaded + offload**.

Imagine one person quickly sorting letters (thousands per hour), while heavy parcels are loaded and driven away by other employees in trucks. What happens if the sorter starts hauling parcels? The letter queue grows to the ceiling.

The `Single-threaded + offload` model splits tasks into two types:

1. **I/O-bound tasks** — reading files, network calls, database access. Most of the time the process waits for the outside world. Thousands of such operations fit into one thread via concurrent async (`coroutines`, `await`).

2. **CPU-bound tasks** — compression, encryption, parsing, computation. Here the CPU runs at full capacity, and concurrency alone is not enough — you need more cores.

The model separates these tasks **physically**: the main thread (`Event Loop`) handles only `I/O`, and `CPU` tasks are sent to separate threads or processes (`Workers`).

**Node.js** was famous for its single-threaded `Event Loop`, great for network apps. But when developers tried to process images or compress video right in the request handler, the server turned into a pumpkin. The fix came with `Worker Threads` — separate threads for CPU-heavy operations.

**Python** went a similar route. Along with `asyncio` the language got a strong tool for I/O-bound code, but the built-in GIL (Global Interpreter Lock) prevented real CPU parallelism inside one process (that issue is already addressed at the time of writing). For blocking operations `loop.run_in_executor()` and `asyncio.to_thread()` (since Python 3.9) appeared to offload heavy work to a thread pool or process pool. The event loop stays responsive, and computation runs in parallel.

**PHP/Swoole** is built on the same architecture: `Request Workers` handle HTTP requests with coroutines, and `Task Workers` do heavy computation. Communication via `UnixSocket` or `pipe` lets one process handle ~100k operations per second.

### Advantages of the model

**1. Resource efficiency**

A single-threaded event loop can serve thousands of concurrent I/O operations with minimal overhead. Switching between coroutine tasks is cheaper than OS-level thread context switches. `CPU-bound` tasks get real parallelism on multiple cores — each worker loads its own core without bothering others.

**2. Simplicity for development**

Code in the event loop needs no mutexes, semaphores, or other joys of multithreaded programming. In a single-threaded model exactly one task runs at a time, so **race conditions** are impossible. `Workers` run in parallel, but if you follow `Shared Nothing`, sync issues do not arise.

The difference in complexity between multithreaded code and single-threaded async code is huge. Modern languages and frameworks unsurprisingly aim for single-threaded async, not classic multithreading.

**3. Simpler compiler/runtime**

Async functions in a single-threaded model are much simpler for compiler and runtime. A good language with multithreading needs its own codegen pipeline. PHP has a serious constraint: part of the code is written in C. That prevents efficient bytecode-level optimizations for threading, memory management, parameter passing. Go’s design is famously complex — proprietary stack, sophisticated GC — all needed for efficient goroutines and channels. We will get to PHP’s GC later, so no relaxing yet!

**4. Manual load distribution**

Developers can consciously split load between request-handling code and worker-pool code. Manual control lets you squeeze the theoretical maximum from hardware. On the other hand, that is also a downside.

### Disadvantages of the model

**1. Manual load distribution**

Manual distribution is a double-edged spear. A developer can optimize for specific tasks, or misjudge what belongs in I/O code and what in workers. The I/O code might get overloaded with heavy work, degrading responsiveness and increasing latency.

The model requires PHP developers to be sufficiently skilled, or to rely on skilled framework authors and their prepared solutions.

**2. Not for every task**

`Single-threaded + offload` is great for web servers, APIs, microservices where the main load is I/O with databases, filesystems, and network calls. But for workloads that need intense computation every step — scientific computing, rendering, ML — the model may be less effective. Full multithreading fits better there.

You might say: we can live with that! We are ready! But is PHP itself ready to be multithreaded?

## Is PHP ready for multithreading?

While working on `TrueAsync`, one of the toughest discussions was “why PHP has no async.” Explaining why PHP is not ready for multithreading may hit similar difficulties. First, though, let’s talk about multithreading. Why do we need it — or better: why do we **NOT** need it?

> Multithreading is not needed for parallel execution of code.

The idea that multithreading is required for parallelism got rooted in programmers’ minds long ago, just like the idea that black holes “suck in” matter rooted in pop culture.

Parallel execution is handled just fine by processes, which are isolated from each other (since the 80386 architecture). Processes can talk via `IPC`, and their completion is tracked with signals (OS events). So why do we need threads?

To answer honestly we would have to go back in time and invite the people who made the decisions so they could explain them: Edsger Dijkstra, Fernando Corbató, Barbara Liskov, Richard Rashid — we could have a great talk show. But even if they agreed, we might still fail to get a straight answer.

It would be incorrect to say:

> Threads are needed so parallel code can share memory without extra tools.

Processes can share memory too, but you need to map a segment into the address space (an extra tool). Threads share **all memory** by default. Literally, if there is a variable `x` accessible in thread `A`, it is available at the same address in thread `B` with no tricks... But no! Multiple threads **cannot** safely work with a shared variable without extra tools.

A more honest statement would be:

> Threads are needed to pass memory between tasks without extra overhead.

If threads use memory to pass messages such that only one thread at a time has guaranteed access to a given memory region, that is maximally efficient in both memory and CPU. Threads deliberately avoid memory that would be shared. This model is called `Shared Nothing`.

Threads are for efficient data transfer between tasks. Just as true as “black holes do not suck.”

## PHP memory model

How does PHP handle memory? A simplified abstract model:
1. Code
2. Data
3. PHP VM state

The ability to share PHP code between threads already exists (solved with PHP JIT). The other components are tightly coupled and cannot be torn apart so easily. For example, PHP uses a global `object_store` holding references to all created objects. PHP’s memory manager is built to work with objects from a single PHP VM and is not oriented toward multithreading. The PHP **Garbage Collector** cannot work with data from different threads and even requires a full VM stop because it directly modifies object `refcount`.

So PHP is a strictly single-threaded model with a stop-the-world GC.

### Moving PHP VM between threads

PHP uses **Thread-Local Storage (TLS)** to keep the VM state per thread. This is critical for isolation between threads in ZTS (Zend Thread Safety) mode.

In modern builds of PHP, a “static” variable per the C11 standard `__thread` (or `__declspec(thread)` in MSVC) is used to get a pointer to the VM state. The speed is very high; on `x86_64` it boils down to reading an address at an offset from the base in register `FS` or `GS`.

```asm
      ; offset - a constant offset computed at compile time
      ; fs - base address of the memory segment
      mov rax, QWORD PTR fs:offset
```

Because `FS/GS` is unique per thread (enforced by the OS), reading from it always returns the correct VM state pointer.

Being able to move VM state between threads can help implement features like Go-like coroutines or actors. Modern VMs pass context through custom codegen, sending VM state via CPU registers. For PHP that trick is impossible because under the hood it uses C functions, and in C there is no way to pass an implicit context parameter to every function. Moving PHP VM state between threads will cost some percentage of total performance.

But what if we could move only a small part of the VM state needed to execute code? For example, `PHP Fiber` copies part of the pointers to global structures (`zend_executor_globals`) when switching.

What if we conceptually split PHP VM into two large parts:
1. `PHP VM` shared. Classes, functions, constants, ini directives, executable code.
2. `PHP VM` movable. The portion of the VM that needs to move.

![PHP VM shared vs private](diagrams/tls-globals-structure.svg)

Some structures could be marked `shared`, others `movable`; even `Executor Globals` could be split into shared and movable parts to enable efficient VM state movement between threads. Extension global structures would not lose performance due to extra indirection because they already use it.

The problem arises with structures related to code compilation because PHP is dynamic via `include/require`, `eval`, and autoloading. These features make it hard to split VM state into shared and movable parts efficiently. If we solve that, PHP could move part of the VM state between threads with minimal overhead.

## Passing objects between threads

What would PHP need to change to pass objects between threads safely? How could this work?

Let’s tackle it at the language level. Say we have a `SomeObject` instance in `$obj` and need to send it to another thread. Is that possible?

```php
$obj = new SomeObject();

$thread = new Thread(function () use ($obj) {
    echo $obj->someMethod();
});

$thread->join();
```

Since `SomeObject` belongs only to `$obj`, we could safely move its address from one thread to another. `$obj` in the main thread would be destroyed:

```php
$obj = new SomeObject();

$thread = new Thread(function () use ($obj) {
    echo $obj->someMethod();
});

// $obj is undefined here

$thread->join();
```

The code above is a 100% analog of move semantics that recently appeared in C++ and exists in Rust and other languages. This way of passing memory between threads offers:
1. Safety. Only one thread owns the object.
2. No overhead for copying or serialization.

To make the behavior predictable and readable by static analyzers, we should add special move syntax, e.g.:

```php
$obj = new SomeObject();

// consume $obj signals moving the object
$thread = new Thread(function () use (consume $obj) {
    echo $obj->someMethod();
});

// $obj is undefined here. Error should be reported here in PHP9.
echo $obj;
```

Looks great, right?

However, moving objects with `refcount = 1` has issues. Consider a category tree:

```php
$electronics = new CategoryNode('Electronics');

$categoriesTree = new Tree();
$categoriesTree->addToPath('/products/electronics', $electronics);
$categoriesTree->addToPath('/popular/electronics', $electronics);  // same category!
```

`$electronics` is in the tree twice (`refcount = 2`). What happens if we move `$categoriesTree` to another thread?

To move safely, we must guarantee that all objects in the graph have no external references:

```php
$node = new CategoryNode('Electronics');
$categoriesTree = new Tree();
$categoriesTree->addToPath('/products/electronics', $node);

$favourites = [$node];  // external reference!

$thread = new Thread(function () use ($categoriesTree) {
    // $categoriesTree moved
});

// $favourites[0] now points to memory in another thread
// Dangling pointer!
```

Safe movement would require:

1. **Full graph traversal** — check all nested objects.
2. **Refcount checks** — for every object in the graph.
3. **Identity preservation** — duplicates inside the graph must stay duplicates.

We can design algorithms for this; call it `deep copy`. A simple implementation could look like:

```php
// Deep copy pseudocode
// Source graph in thread A
$node = new Node('A');        // addr: 0x1000
$tree->left = $node;          // addr: 0x1000
$tree->right = $node;         // addr: 0x1000 (same reference)

// Deep copy into thread B (pseudocode with MM)
$copied_map = [];  // hash table: addr_source -> addr_target

function deepCopyToThread(object $obj, Thread $target_thread_mm) 
{
    $source_addr = get_object_address($obj);

    if (isset($copied_map[$source_addr])) {
        return $copied_map[$source_addr];  // already copied!
    }

    // Allocate memory in the other thread's MM
    $new_addr = $target_thread_mm->allocate(sizeof($obj));
    $copied_map[$source_addr] = $new_addr;

    // Copy object data
    memcpy($new_addr, $source_addr, sizeof($obj));

    // Traverse properties
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

**Time complexity of deep copy**: `O(N + E)`, where `N` is the number of objects and `E` is the number of references.
**Space complexity**: `O(N)` — hash table + new objects + recursion stack.

Compared to serialization this can be a faster copy because it avoids converting to/from a transport format, but the gain depends on the data shape and graph size. You can also build a hybrid: move data with `refcount = 1`, and run `deep copy` for everything else.

Result:
1. PHP developers do not need to think about how objects are passed to another thread.
2. Best case: memory is moved (`refcount = 1`).
3. Worst case: memory is copied with `deep copy` while preserving identity (`refcount > 1`).

Looks good:
* minimal PHP syntax changes
* gradual changes possible
* multithreading becomes available

But at the core level not everything is rosy. To make object movement real, PHP needs some memory-management mechanism across threads. Right now that is impossible.

## Multithreaded PHP Memory Manager

PHP’s memory manager is similar to modern allocators like `jemalloc` or `tcmalloc`. The difference: it lacks a correct algorithm for freeing memory from another thread.

Consider:
* An object is created in thread `A`.
* It is passed to thread `B` via move (as-is).
* In `B`, the object is no longer needed and should be freed.

Each PHP thread has its own `Memory Manager (MM)`. When `B` tries to free memory allocated in `A`, there is a problem. `B`’s MM knows nothing about `A`’s memory, and freeing it would cause an error. Direct access from `B` to `A`’s MM structures is a bad idea because it requires synchronization. Modern high-performance multithreaded allocators solve this with deferred free.

The general idea of `deferred free`:
1. `B`’s MM sees an unknown pointer.
2. It finds which MM owns the pointer and sends a message to that MM’s queue saying the pointer can be freed.
3. `A`’s MM processes the queue and frees the pointers in its own context.

![Cross-thread deallocation](diagrams/cross-thread-free.svg)

Using modern lock-free structures, this algorithm has high throughput, allows different threads to free memory in parallel, and almost avoids locks.

A multithreaded PHP memory manager opens doors for other changes that were previously impossible.

## Shared objects

Being able to pass memory from one thread to another with minimal operations is great, but what if we could create objects that are designed up front for sharing across threads?

Many services can be built as immutable objects, so they should be shareable between processes, saving memory and speeding up worker startup.

Unfortunately `refcount` gets in the way because it effectively makes all PHP objects mutable! Can we avoid that?

### Proxy objects

The first approach is proxy objects that reference real objects stored in a shared memory pool accessible to all threads. The proxies hold only an identifier or pointer to the real object and methods to access its data. Downsides:
* increased access time to data/properties
* added complexity for `Reflection API` and type checks

On the other hand, PHP already has a strong mechanism for creating proxies. Proxy-shared objects are a fine option in some cases, e.g. for a counters table or a data table like Swoole/Table.

### Shared objects with GC_SHARE flag

PHP has a built-in mechanism for **immutable** elements via the `GC_IMMUTABLE` flag. It is used for:

- **Interned strings** (`IS_STR_INTERNED`) — string constants that live for the entire PHP process
- **Immutable arrays** (`IS_ARRAY_IMMUTABLE`) — e.g. `zend_empty_array`
- **Constants in opcache** — compiled code with constant data

`GC_IMMUTABLE` lets the engine **skip** refcount changes for such structures:

```c
// Zend/zend_types.h
// Function that increments refcount for zend_refcounted_h
static zend_always_inline void zend_gc_try_addref(zend_refcounted_h *p) {
    if (!(p->u.type_info & GC_IMMUTABLE)) {
        ZEND_RC_MOD_CHECK(p);
        ++p->refcount;
    }
}
```

A similar mechanism could support `SharedObjects`, e.g. a `GC_SHARE` flag.

Performance analysis shows that checking `GC_SHARE` adds **+34% overhead** to an isolated `refcount++` (microbenchmark). In real apps, where refcount work is a small fraction of total work, the impact should be barely noticeable:

- **Realistic operations** (arrays/objects): +3-9%
- **Real apps**: +0.05-0.5%

This solves half the problem; the other half is designing GC for such objects. Using atomic refcount is not ideal due to potential slowdown when many threads hit the same object. A deferred free algorithm is likely a better fit.

### Region-based memory

Region-based memory is popular now for memory management in web-oriented languages.

The idea: allocate memory for a specific task or thread in separate regions that can be freed wholesale (or almost) when no longer needed. That avoids per-object management complexity and simplifies GC.

For example, `PHP MM` could guarantee that objects are created in a region tied to a specific PHP object. The region’s lifetime equals the object’s lifetime.

When the object is destroyed, the whole region can be freed without traversing children. If such an object needs to be “moved” to another thread, you can avoid deep copy.

PHP VM has issues implementing region-based memory — e.g. the global object list, opcode caching. But the chances of an efficient implementation are not zero and warrant further research.

A working region-based memory algorithm opens the door to implementing actors — special objects with isolated memory.

Actors are the most convenient, powerful, and safe tool for multithreaded programming.

## Coroutines and Threads together

From a coroutine’s point of view, `Thread` is an `Awaitable` object. A coroutine can await a `Thread` result without blocking other coroutines. So one thread can host many coroutines waiting for heavy tasks. The thread serving them keeps responding quickly to new requests because awaiting a `Thread` does not block the event loop.

```php
use Async\await;
use Async\Thread;

$thread = new Thread(function() {
    // hardware-bound task here
    return 42;
});

$result = await($thread); // Coroutine pauses here until the Thread finishes
```

This approach can implement a chat scenario with CPU-intensive tasks and simple business logic.

![Thread + Coroutine architecture](diagrams/chat-sequence.svg)

The diagram shows an example architecture. The app has two thread pools: request-handling threads with concurrent multitasking, and worker threads for CPU-intensive tasks. A coroutine processes a request, can fully pause while a worker does the heavy task, and then continues.

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

The coroutine code is sequential and reads like normal code where `ThreadPool::enqueue` would call the callback in the same thread. The DTO crosses threads, and the resulting string is not copied twice in memory.

## Garbage Collector and stateful mode

Modernizing PHP’s memory manager is not the only change needed to improve the language for a multithreaded environment. Without an efficient GC, multithreaded PHP will suffer from performance issues and memory leaks from cycles.

PHP’s GC uses two algorithms: **reference counting** as the primary memory management mechanism and **Concurrent Cycle Collection** (Bacon-Rajan, 2001) for cycles. Reference counting increments/decrements on every assignment, making it unsafe for multithreading without synchronization. Atomics on every assignment would be huge overhead; without sync you get races and leaks. The cycle collector, though called “concurrent,” works only within a single thread and uses color marking (**PURPLE** → **GREY** → **WHITE/BLACK**) to find cycles, which is not thread-safe either.

The upside: the current GC implementation will work in a multithreaded setting because it is separated from the memory manager and does not depend on where memory was allocated.

But if PHP wants a multithreaded era of stateful apps, the GC must be adapted to:
1. Run **in parallel** in a separate thread without affecting business code.
2. Free resources as quickly as possible.
3. Provide extra tools for leak detection, logging, and telemetry (especially for long-running apps!).

The cycle collector can be modified to work in a multithreaded environment by processing references in a separate thread, improving overall responsiveness. That may be enough for a start!

## Actors

`ThreadPool` and passing objects between threads are useful but demand attention, skill, and effort from developers. There is a better abstraction for multithreaded programming that hides thread/memory complexity and fits business logic perfectly: **actors**.

Actors are a model of concurrent parallel programming where the basic unit of computation is an **actor**.

Each actor:
- Has its own isolated state
- Processes messages sequentially
- Interacts with other actors only via messages
- May run in a separate thread

You can think of an actor like an object, which makes it possible to use familiar OOP patterns in multithreaded PHP.

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
   $room->postMessage('Alice', 'Hello!');  // Runs in another thread, suspends the coroutine!
   $messages = $room->getMessages();       // Runs in another thread, suspends the coroutine!
   echo json_encode($messages);
});
```

`ChatRoom` objects are special. Their data and PHP VM state are localized to move easily between threads. Each method runs in its own thread, but at any moment only one thread can execute methods of a given actor.

Semantically, the base `Actor` class defines how PHP VM and the memory manager work so that `ChatRoom` objects can safely run in separate threads. The class type “stores” not only info about methods and properties but also how the MM and GC should operate for such objects. Similar approaches exist in other languages: Rust, C++. The upside: no syntax changes and it fits OOP philosophy.

The example looks like plain sequential code running inside a coroutine. But since `postMessage` and `getMessages` run in another thread, they are not executed directly. The coroutine sends a **message** to the actor’s queue, goes into waiting, and resumes only when the actor runs the method in another thread and returns the result.

None of this conflicts with familiar PHP OOP: `Actor` overrides `__call`:

```php
class Actor 
{
    private $threadPool;

    public function __call(string $name, array $arguments): mixed
    {
        if(current_thread_id() === $this->threadPool->getThreadIdForActor($this)) {
            // Run method directly if we are in the same thread
            return $this->$name(...$arguments);
        }
    
        // Otherwise queue the call for the actor
        return $this->threadPool->enqueueActorMethod($this, $name, $arguments);
    }
}
```

`enqueueActorMethod` adds `postMessage` to the actor queue, subscribes to the result event, and calls `Async\suspend()` to pause the coroutine.

Actor code executes sequentially, solving race conditions and making multithreaded development transparent for the developer.

Parallelism is achieved because each `ChatRoom` actor can run in a separate thread:

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

`ChatRoom` instances can run in parallel in different threads because each actor has its own execution thread, unique PHP VM state, and memory.

Create **100 chat rooms**:

```php
use Async\Actor;

$rooms = [
    'general' => new ChatRoom('general'),
    'random'  => new ChatRoom('random'),
    'tech'    => new ChatRoom('tech'),
    // ... 97 more rooms
];

// Coroutine for handling requests
HttpServer::onRequest(function(Request $request, Response $response) use ($rooms) {
   // HTTP request handling
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

Each chat room processes messages sequentially and in parallel with other rooms.

Actors need no mutexes, locks, complex synchronization, or manual thread-pool interaction. They provide a ready high-level solution for parallelizing work.

If one chat room needs to send a message to another, it is possible because actors are `SharedObject` and can interact across threads:

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
    
### Actor internals

PHP VM guarantees that all objects inside an actor:
* either belong only to that actor and are allocated in its unique region
* or belong and were moved from other regions or threads
* or are another SharedObject or another actor

An actor either owns its region or works only with explicitly shared immutable objects — otherwise races remain.

The memory manager guarantees that all memory operations inside actor methods are automatically tied to it via the region directly associated with the actor.

Methods are executed via an `MPMC` message queue served by a `Scheduler`. The `Scheduler` allocates CPU time between actors, providing concurrent and parallel execution.

![Actor model architecture](diagrams/actor-message-flow.svg)

## Conclusion

All of this sounds great, but when will we see it for real? You ask.

The `Single-threaded + offload` model may appear in the near future because many components are ready. `TrueAsync`: single-threaded coroutines reached beta. An experimental multithreaded memory manager and `API` for creating threads are implemented.

Actors require more development time because they touch many parts of PHP core, and they remain a realistic goal for PHP 9, offering the market a safe multithreaded programming language.
