## Macro-level, System Design

- Design complexity:
	[Brooks](http://worrydream.com/refs/Brooks-NoSilverBullet.pdf) observed that much software complexity is essential, but some is also accidental

	**Essential complexity:**

	> The essence of a software entity is a construct of interlocking concepts: data sets, relationships among data items, algorithms, and invocations of functions. This essence is abstract, in that the conceptual construct is the same under many different representations. It is nonetheless highly precise and richly detailed.
	>
	> I believe the hard part of building software to be the specification, design, and testing of this conceptual construct, not the labor of representing it and testing the fidelity of the representation
	>
	> If this is true, building software will always be hard. There is inherently no silver bullet
	>
	> - Fred Brooks, The Mythical Man Month

	Essential complexity is a property of the *problem you want to solve*, and is independent of the developer, team, or technology.

	**Accidental complexity:**
	This can include system acrobatics motivated by working on the given system architecture and abstraction (e.g. using the wrong tools), complexity added due to [technical debt](https://en.wikipedia.org/wiki/Technical_debt), etc...
	Writing in high level languages raises the level of abstraction (e.g. garbage collection), thus removes the programmer's consideration for "unnecessary" details...until performance or predictability become part of the specification.
	"Hacking in" features, or maintaining backwards compatibility in spite of new architectures also significantly add to the accidental complexity of the system.

	When the system abstractions work *against* your goals, there is a *semantic gap* between what the system architecture makes easy or prioritizes, and what you need from the system.

    - [accidental vs. essential](https://en.wikipedia.org/wiki/No_Silver_Bullet) complexity
	- there are [many](https://pressupinc.com/blog/2014/05/root-causes-software-complexity/) [takes](https://medium.com/background-thread/accidental-and-essential-complexity-programming-word-of-the-day-b4db4d2600d4)
	- an [argument](https://danluu.com/essential-complexity/) that technology *does* decrease accidental complexity, e.g. displaying data with `ggplot` includes downsampling, rendering, image output, etc...

### Complexity Management

How do we design systems to manage complexity?
Code has a tenancy to become pretty tightly coupled.
Adding a new function codifies a number of assumptions for the data-structures and functions accessed.
When this happens on the large scale, logic becomes quite intertwined, thus any change impacts more code.
This makes any future change much more difficult.
Yet we have massive systems such as kernels, browsers, and video games.
How is this?

- *Abstraction* -
	We don't write high-level code that uses low-level mechanisms.
	Your application isn't programming interrupt handlers.
	Web applications don't program layout engines and bitmap creation, instead focusing on DOM manipulation.
	Abstraction is the foundation of computer science, and is particularly important in system design.
	Key components of abstraction include:

	1. providing abstract interfaces for resource/object access, and
	2. mechanisms to *contain the entropy* expressed by more permissive interfaces that we use to provide a higher-level, more constrained functionality.

	Each of these is inter-related, and essential.
	We wish to take the chaos of the system on which the abstraction is implemented, and organize it into a representation that is *less capable*, but *more useful* (for specific goals).

	*Example:*
	A car with steering wheels for each wheel.
	This is very powerful, but very easy to use incorrectly.
	Abstraction over this system can be used to tie the front wheel's steering together, and locking the back wheels.
	We provide an interface of a *single steering wheel* to actuate the car.
	We contain the complexity of the underlying system, while providing an abstraction that is more useful in many situations (unskilled drivers).
- *Modularity* -
	Abstraction is important, but *structuring the abstractions* with respect to each other is also necessary.
	Modularity requires that abstractions are implemented with unfettered access to the functionality in their module, but that inter-module dependencies are restricted to module interfaces.
	This requires the encapsulation and information hiding of the details for how the abstraction is provided.
	Inter-module *interfaces* provide important abstractions that tie modules together.
	Modules have the effect of constraining complexity by disallowing unrestrained code access outside of modules, and focuses a lot of design on inter-module abstractions.
	Much of the power of modularity comes from the ability to *replace different modules that share the same interface*, thus alter system behavior without writing additional code.
	The [Liskov substitution principal](https://en.wikipedia.org/wiki/Liskov_substitution_principle) provides an intuition as to when this is possible, but we've internalized this notion as *polymorphism*.
- *Layering and Hierarchy* -
	In systems that are large enough that the complexity of inter-module interactions becomes intractable, strategies for organizing modules are required.
	If there are $N$ modules, there are a potential $N^2$ interactions between them.
	For a large $N$, this can be challenging.
	A change in one module can impact all $N-1$ other modules, which makes changes quite challenging.

	Empirical studies have shown that complexity in complex subsystems is dependent on the *coupling* between components (inter-module connections).
	For example, some have found a correlation between some [coupling](https://www.hbs.edu/faculty/Publication%20Files/18-031_78569c7d-fc0c-4e1e-bd79-143bc507112a.pdf) [metrics](https://link.springer.com/chapter/10.1007/978-3-319-62105-0_4) in Chrome and security issues.

	The simplest way to constrain this complexity is to use a *layered* architecture in which modules can only leverage other modules at their level, and at one lower level.
	They can provide services to modules at the next higher level.
	This can be quite limiting, but constraints interactions to modules at the same level (on average $N/L$ modules for $L$ levels), and those at the next lower level.
	This creates $N/L + I$ interactions where $I << N/L$ is the number of modules at the next lower level that have public interfaces to the higher level.

	Compilers are typically implemented as a sequence of *passes* that happen on the core code representation data-structures.
	They tend to go from high-level representations of the code, down to more and more detailed.
	A similar example is the *Composite* linker that parses through many components, performing a set of [transitions](https://github.com/gwsystems/composite/blob/loader/src/composer/src/main.rs#L50-L68) that represent layers of computation, taking the system from general states to move specific, and finally, executable states.
	Note that pipelined-composition of modules (as shown here, and similar to pipes in UNIX) are similar to layered system construction as they restrict the interactions between modules, but they focus mainly on the processing of data between modules.

	The [*total ordering*](https://en.wikipedia.org/wiki/Total_order) on modules can be quite limiting.
	For example, they often require that modules be created at a level that simply pass a request from a higher-level downward to a lower level.
	As such, some systems instead use a hierarchical organization for modules.
	Systems form a *tree* of modules in which, like a layered system, they only communicate with modules in the same "branch" of the hierarchy, and with the interface of the "parent" in the hierarchy.
	This is a *very* common structuring technique and is used in monolithic systems, VM-based systems, and even distributed systems such as DNS.

	Hierarchical module organization is still limiting.
	You cannot communicate easily between branches, and you still can only communicate one level below you.
	So we can think about modules as generally interacting with each other in a structured directed acyclic graph ([DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph)).
	Question: why a DAG instead of a general graph?
	We'll discuss this case quite a bit, so I won't go into more details here.

**Importance of Interfaces.**
One important implication of a module- and interface-based structuring of system is that the interfaces are very important.

1. The interfaces encode the core of the abstractions in the system -- the usability of your abstractions will be tangibly experienced through your interface.
2. Specifications are stated in terms of the interface -- they encode the semantics (behavior) of the system at a high-level.
3. Multiple modules can implement the same interface, and many modules will depend on the same interface, thus any changes to the interface impact a potentially large amount of functionality -- it is hard to update interfaces.
4. Interfaces encode many of the assumptions that implementations are allowed to make, and how complex the error checking will need to be -- thus the simplicity of implementations and of the error handling in clients is heavily impacted by the interface.
5. Unit tests should be written to the *interfaces* -- the automated means of helping you maintain some consistency across changes (made by many people in a team) are coupled with interfaces.

When designing a system, the highest priority is to get the interfaces right^[In our own research, we've made the error again and again to work with hacked-together interfaces under the pressure of deadlines. In the medium term, this has wasted a lot of developer effort due to confusions and accidental complexity due to the weaknesses of the interfaces.].
If you have strong interfaces, implementations can be weak, and replaced.
If you're interfaces are not good, the system will collapse under complexity and confusion.

A number of other factors can combat complexity.
They all fall into a simple motto:

**Keep it simple.**
Instances of this include the following:

- *Avoid over-generalization.*
	Attempting to solve all problems runs the risk of generally solving none.
	This can apply to the implementation: you don't always need to make it generic across all future changes.
	A simple solution, to a well-stated problem, is often sufficient.

	Over-generalization can also apply an interface.
	Locks that enable recursive (nested) locking are more general than those that do not, but there are [many](http://www.fieryrobot.com/blog/2008/10/14/recursive-locks-will-kill-you/) [arguments](http://www.zaval.org/resources/library/butenhof1.html) [against](https://blog.stephencleary.com/2013/04/recursive-re-entrant-locks.html) [them](https://inessential.com/2013/09/24/recursive_locks), simplicity included.
- *Avoid optimization in general*.
	Optimization usually involves rewriting code-paths in ways that make them less terse, and less direct.
	Optimization almost always makes code more difficult to read, more difficult to debug, and more challenging to modify.
	Systems require optimization, but this can usually be done in targeted parts of the system.

**Include unit tests**.
This may be a strange item to discuss at this point, but it is quite important.
Unit tests are most useful (and near essential) on the interfaces between levels, and for public module interfaces.
When you *change* a system, it is immensely valuable to have a *baseline* for correct functionality.
A unit testing suite enables more aggressive design, as changing a module can leverage automatic checking of the intended semantics (assuming you have a decent test suite).
It also constrains complexity as module interfaces are better documented with use-cases, and auto-checked for some approximation of correctness.

### System Interface Design Considerations

When designing system interfaces, you must often consider more factors than if you're simply implementing a library, or a module in an application.
These are all motivated by system's responsibility to manage system resources and multiplex them.
The general question include how you should *name* system resources, how you can *access those resources*, and how you interact with *concurrent* resources.

- *Naming.*
	When modules are separated, they require structured means of communicating with each other.
	Where information hiding is taken very seriously, and where isolation is required, passing pointers to data-structures is not appropriate.
	Thus the question of *naming* becomes very important: how do we name the resources/objects that are passed into the interface on the client side, and manipulated on the server?

    - Text-based hierarchical namespaces (UNIX, URL/REST, D-Bus, plan9)
	- Per-principal hierarchical namespaces (containers, plan9)
	- Isolation via namespace hiding (containers, plan9, composite)
	- General routing facilities via programmable namespaces (D-Bus, plan9, systemd)
	- Serializable means of addressing in namespaces, and interacting (9P, HTTP)
	- names often include a service *and* an object/API selector

	When a client *references* an object by name, it is associated in the namespace with the object by *binding* the name to the object.
	This might be done using OS-provided abstractions such as file descriptor tables and the Virtual File System (VFS) layer in UNIX to dynamically *bind* a file descriptor to a *file* named using the hierarchical namespace.
	This form of binding often uses "handles", or opaque references (integers) that somehow *map* to the objects (e.g. through the file descriptor table).
	Alternatively, binding might be done *statically* during *linking* to tie function calls, and data accesses to their implementations.
	This form of binding often uses "references".
	Techniques such as lazy binding are somewhere in-between in that they use [Procedure Linkage Tables](https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html) (PLT) to dynamically bind client and library code (for dynamically loadable libraries,`.dll`s or `.so`s).
	A level of indirection in binding is powerful as it enables the replaceability of the module's implementation.

	Naming is the *foundation of isolation*.
	If a client cannot *name a resource*, then it *cannot access or modify that resource*.
	Lets go over a couple of examples:

	- Containers are a means in an existing monolithic OS to provide a custom user-level personality to a set of principals.
		As such, the file system, process id, network (and on and on) namespaces can be made *disjoint* across different principals.
		This effectively enabling those principals are isolated from each other (assuming the kernel itself is not compromised), and enables different OS and programming environments for each principal.
	- Capability-based systems associate a single namespace with each protection domain such the only accessible resources are those defined by a flat function $f(\text{capid}) \to \text{res}$ where $\text{capid} \in \mathbb{N}_0$ (the system call on such systems is essentially a wrapper around $f$).
		This simple namespace limits all resources available to a protection domain (to $R = \cup_{\text{id} \in \mathbb{N}_0} f(\text{id})$).
		Note that this looks a lot like a *file descriptor table*, but is used for *all* naming of resources.
		Two protection domains $p_i$ and $p_j$ (with corresponding capability lookups using $f_0, R_0$ and $f_1, R_1$)  are completely isolated from each other (modulo CPU time) if and only if $R_0 \cap R_1 = \emptyset$.
		One of the benefit of capability-based systems is the simplicity in understanding resource access availability in the system.
- Uniform means of *interacting with named objects*.

	- VFS, 9P, [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)

		| Op     | VFS    | [9P](http://man.cat-v.org/plan_9/5/intro) | SQL    | HTTP           |
		| ---    | ---    | ---                                       | ---    | ---            |
		| create | creat  | create                                    | INSERT | PUT            |
		| delete | unlink | remove                                    | DELETE | DELETE         |
		| read   | read   | read                                      | SELECT | GET            |
		| update | write  | write                                     | UPDATE | PUT[^RESTPOST] |

	    [^RESTPOST]: Note that `POST` is closer to a method invocation to perform some operation with semantics that go beyond "update please".

	    Note that UNIX and 9P focus on optimizing for iterative reads, and repetitive updates, but also uses `open` and `close` (`clunk` in 9P) to maintain a record of continuing read an update operations.
		This simplifies a number of things, including race conditions between accessing data in an object and deleting that object (the equivalent problem in HTTP is *paging* while deleting), and doing expensive access control checks once on `open` to avoid having to do them on read/update operations.

- *Data representation* for each object.

	- Untyped streams of bytes: data passed as text streams (pipes, files, etc...)
	- Typed functions: language ecosystems, [dbus](https://pythonhosted.org/txdbus/dbus_overview.html), [protobuf](https://developers.google.com/protocol-buffers)s, [capnproto](https://capnproto.org/)

- *Action and data aggregation for optimization.*
	Action aggregation across communication (move computation to data) moves some set of actions to where the data is to avoid communication in between each action.
	Adds complexity into the system by enabling aggregate operations to execute in the system service.

	- Database [stored procedures](https://en.wikipedia.org/wiki/Stored_procedure).
	- OpenGL [display lists](https://en.wikipedia.org/wiki/Display_list) (or command buffers/command lists).
	- Linux dynamically uploadable procedures via [ebpf](https://ebpf.io/).
	- The common use of [lua](http://www.lua.org/) as a configuration and policy engine embedded in C++ games.
	- [Graphql](https://graphql.org/)[[2]](https://en.wikipedia.org/wiki/GraphQL) vs [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) w/ [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations.
	- HTTP 1.0 vs. [pipelining in HTTP/2](https://en.wikipedia.org/wiki/HTTP_pipelining).

	Data aggregation across communication is most often see as the batching of data.
	When we print (e.g. with `printf`), we don't usually call `putc` on each character.
	Instead, batch up data to be printed out, and only call `write` to output all of the data at once when a newline, `\n` is printed.
	This generally *trades higher throughput for increased latency* by delaying data movement, thus amortizing data movement costs.

	- async communication via pipes with finite buffering to batch sends/receives
	- buffering/*batching* of service requests (printing!) -- downsides WRT throughput vs. latency & `fflush(...)`

- *Concurrency.*
	We must consider asynchrony and multiplexing in our interactions with named resources/objects: how can interactions hide the latency of communication (e.g. network/disk latency), and compensate for non-deterministic concurrency

	The main challenges here are that if we have a blocking API, we're forcing clients to use multiple threads, and to couple their thread's execute to our blocking and wakeup logic.
	If we have solely non-blocking APIs, then we're forcing the client into polling all resources (which wastes CPU).
	The solution is to provide *event multiplexing* functionality.
	These are APIs that enable a thread to block waiting for *any one of the resources that might have an event*, and to return when *any of them have an event*.
	Conceptually, we pass in a set of resource handles, and the function only returns when any of them have events, which are returned as a set of events.
	As such, the client can iterate through the set, and process each of the resources (e.g. by `read`ing data from them that we now know is available).
	POSIX defines `poll` and `select` to do event multiplexing, and Linux/FreeBSD extend that with `epoll`/`kqueues`.
	A key challenge when you have many resources provided by many different modules is how they can coordinate to multiplex their events.

	- blocking considered generally dangerous -- ties communicating principals together in execution (how?)
	- asynchronous functions complicate APIs as they require event notification functions such as `select`, `poll`, and `epoll` (idempotentcy & edge vs. level triggering)
	- [go](https://golang.org) is a programming language and environment at encourages using synchronous and asynchronous channels, along with lightweight computations (goroutines) to handle concurrency.
	- [libuv](https://libuv.org/) is a library that attempts to use an *event-based* programming model (versus a *thread-based* programming model) based on *callbacks*.
		It is the framework underlying `node.js`, and mimics the closure-based programming model common in javascript.
- *Parallelization.*
	For the most part, you want to be able to hide any parallelism behind an interface.
	However, it is often the case that the parallelism constraints span interfaces.
	For example, the specification of a function should publicise if the API is concurrency/parallelism-safe, or if locks need to be taken before calling the function.

	- Partition/split - data parallelism
	- Stream - pipelined parallelism
	- Commutativity indicates that an API *can* be implemented in a way that it can scale across increasing numbers of cores, but not necessary that it *is* implemented that way.
	- Details later!

An additional important aspect of system design is an understanding of reality.
There is a common trade-off between *perfection and shipping*.
An important question is what features and aspects of design don't need to be perfect?
Where can the design be weak?
If you don't have time to prevent all accidental complexity, try and engineer where you're OK with it sneaking in.

Taking this consideration further, some harsh reality: "[worse](https://en.wikipedia.org/wiki/Worse_is_better) is [better](https://dreamsongs.com/WorseIsBetter.html)".
The best technology doesn't always win.
The first mover effect matters.

### Interface Design Properties

Different functions in system interfaces have many high-level properties.
It is important to consider many of these factors when reading or implementing an interface.
Some of the most important properties that you should consider when assessing, and designing interfaces include *composability*, *state management*, and *commutativity*.

- *Composability.*
	Are there traps in interactions between *different functions* in the API?
	How about between this API and others?
	There are two key questions related to composability:

	1. If I call *f(...)*, is it possible for it to fail -- despite its specification giving us confidence it should work -- given the context of previous abstractions I've used up till now?
	2. If so, how can the abstraction I'm trying to use, be impacted by the context?
		Is it possible to control the impact of the non-composability, and how much accidental complexity does it add?

	An example (and more to come in the next sections): Locks are generally *not composable*.
	Lets consider a few functions that modify bank accounts:

	```c
	fn dec(struct account *a, int amnt) {
		lock_take(&a->l);
		if (a->balance < amnt) {
			lock_release(&a->l);
			return -1;
		}

		a->balance -= amnt;
		lock_release(&a->l);
		return 0;
	}

	fn inc(struct account *a, int amnt) {
		lock_take(&a->l);
		a->balance += amnt;
		lock_release(&a->l);
	}
	```

	These two functions work on their own, but when composed together, they don't!
	If we want to transfer money from one account to another, we have something like this:

	```c
	// transfer between accounts
	dec(a, 10);
	inc(b, 10);
	```

	The problem here, is if we're preempted between `dec` and `inc`, the total amount of money in the system is incorrect!

	```c
	fn transfer(struct account *from, *to, int amnt) {
		lock_take(&from->l);
		lock_take(&to->l);
		to->balance += amnt;
		from->balance -= amnt;
		lock_release(&to->l);
		lock_release(&from->l);
	}
	```

	Now we transfer by simply:

	```c
	transfer(a, b);
	```
	...but now the problem is that we have potential deadlock (if another thread does `transfer(b, a)`)!
	To make this work, you likely either use a global block that protects both accounts, or ensure that you take the locks in the same order, regardless the argument order (e.g. by always taking the lock with the lower address first).

	We see here that locks fundamentally don't compose: functions might work independently, but where a critical section is necessary to spans multiple calls, independent function's don't compose.
	Thus, you might see "aggregate functions" in APIs simply to span critical sections across the constituent conceptual operations.

	Some examples:

	- [`fork`](https://www.microsoft.com/en-us/research/uploads/prod/2019/04/fork-hotos19.pdf) vs. `posix_spawn` and `CreateProcessA`
	<!-- - The massive complexity Signals w/ re-entrancy & locks vs. blocking & event notification -->
	- Global data and state can cause a function to not compose with itself (e.g. `strtok` vs. `strtok_r`)!
	- Abstractions that have global and non-deterministic effects: interrupts vs. polling.
	- Locks (no enforced nesting) + blocking vs. lock-free/transactional memory, and the general challenge of [programming with locks](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/beautiful.pdf).
	- Multithreading vs. data-structures fundamentally don't compose!!
		However, sometimes a lack of composability is worth it, but demands additional abstractions.

- *[Separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) (SoC)* and *[Orthogonality](http://www.catb.org/~esr/writings/taoup/html/ch04s02.html#orthogonality).*
	When attempting to figure out how to break down the interfaces of the system into different "clusters" that are implemented by modules, the SoC and orthogonality are guides.
	Different functions in an interface -- and, indeed, different interfaces -- should each perform a specific function that does not overlap with others.
	This lack of overlapping functionality, and unintended modifications, produces an orthogonal design.

	> Do one thing well.
	> - Doug McIlroy

	This quote is not about simplicity, instead about orthogonality.
	Orthogonality in code can be seen as avoiding code redundancy: a single function is responsible for some functionality.
	This follows from the "don't repeat yourself" rule.

	Imagine a system that replaces the file system code in the kernel with a database (see an attempt in [WinFS](https://en.wikipedia.org/wiki/WinFS), and a simpler hack in [joinFS](https://www2.seas.gwu.edu/~gparmer/publications/esa11.pdf)).
	However, for power users it enables them to directly use SQL for operations like aggregation that couldn't be easily leveraged through the FS interface.
	Now there are *two* interfaces the provide two subtlety different APIs to the same state.
	But the SQL interface can be used to do things that you cannot do in the FS, for example, to modify `..` and `.` to reference other directories.
	What does the FS code do when it finds these inconsistent states?
	Does the FS now need to include code to consider *all possible* configurations of the database?
	Does the SQL processor have to include code to consider only operations the FS can perform?
	This example demonstrates the danger of non-orthogonality: either the implementations of different modules need to become increasingly coupled -- which makes it impossible to replace any of them, and more difficult to debug them, or they remain separate and have undefined, surprising, or bad interactions in edge-cases.

	The ability to `mmap` a file, and access its contents using load/store instructions *and* to *also* access the file using `read`/`write`/`seek` clearly is not orthogonal.
	However, the benefit of being able to directly access file data using memory operations has enough benefit that `mmap` won out over orthogonality.

	The above lock example that required the addition of the `transfer` function to extend the span of a critical section demonstrates non-orthogonality.
	`transfer` is conceptually redundant with `inc` and `dec`.
	It demonstrates that sometimes you need to depart from orthogonality to satisfy niche situations, or provide optimizations.
	Others [have argued](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/beautiful.pdf) that locks should be replaced with abstractions such as software transactional memory (STM)^[Time has not replaced locks with STM. It is useful and powerful when used with functional data-structures, but much more challenging with general data-structures.].

	Another simple example: Linux has a huge variety of different ways to do IPC including signals, pipes, System V Shared Memory, mapped files, shared files, named pipes, UNIX domain sockets, TCP/UDP sockets, DBus, Binder (on Android), ...
	When you sit down and try and write a service, which should you choose?

	A high-level way to think about SoC and orthogonality is to ask: given the set of *effects* that a function or an interface has on encapsulated state, the extent to which two functions or two interfaces are coupled (thus not following SoC/orthogonality) is commensurate to how much those effects impact state in a way similar to the other.
	We want our interfaces to be orthogonal *and* composable.
	Orthogonal interfaces do separate, useful things, and composing them together leads to a system that is greater than the two parts.
	This is the core of how systems become more useful, and more powerful, once they hit users, and was one of the core underlying principles of UNIX.

- *[Separation of Mechanism and Policy](https://dl.acm.org/doi/10.1145/1067629.806531).*
	Policy is a decision about *how* to use a provided *mechanism*.
	How does your application use the mechanisms for accessing files in the file system to create its functionality?
	How does the shell use pipes, `dup`, and `fork` to create useful pipelines?
	Often policy is leveraging the composability of the mechanism's interface to create new behavior.

	The X-windows server provides primitive mechanisms around compositing and rasterizing (displaying overlapping windows), and relies on the clients to provide their own styling and ["chrome"](https://en.wikipedia.org/wiki/Graphical_user_interface#User_interface_and_interaction_design).
	This moves the policy of how to define the chrome to the applications.
	This has the massive benefit that defines the chrome, is the application which likely knows best how to use the chrome.

	Policies tend to become *mechanisms* at the next higher abstraction level, and are used by even higher level policies as such.
	At the lowest level, atomic instructions are mechanisms provided by the hardware.
	We define policies on top of that to define locks and semaphores.
	We use those locks and semaphores as mechanisms to define the policy of full condition variables.
	This emphasizes the power of this separation: policy is enabled to use the abstractions in specific ways, and those policies that provide useful abstractions can be further built on.

	What's the downside?

	> the cost of the mechanism-not-policy approach is that when the user can set policy, the user must set policy.
	> - Eric Raymond

- *State management.*
	Interface implementations might abstract away an immense amount of state (like a file system), or they might take a lighter touch.

	- *Liveness* -
		Who owns a reference, the caller or the callee?
		This is a concern mainly applicable in pass-by-reference (library) based APIs where resources are referenced with pointers.
		It is essential to understand if the client or the service are in charge of deallocating the resource, and when references to the resource are valid.
		This type of information is a part of all APIs, even those that are handle-based (e.g. file descriptors).
		The client owns a reference if it is responsible for deciding when to `free` the resource (in handle-based APIs, this might instead be something like a `close` or `unlink`).
		If the service owns the reference, it is in charge of storing and managing the lifetime of the resource.
		A few examples:

		1. A service that stores objects in some data-structure (DS), and enables them to be retrieved later.

			In the first case, the client owns the object reference.
			A client passes a reference to an object, the service stores the reference in the DS, and the client `free`s the object.
			This can be problematic.
			What if the service is a library, and the client `free`s the object while it is still referenced in the data-structure?
			The library will now have a [dangling reference](https://en.wikipedia.org/wiki/Dangling_pointer) to memory.
			If the service provides *handles* for the client to reference the object, this problem goes away.
			Instead of simply calling `free` to deallocate the object in memory, we must use an API provided by the service to deallocate the object.
			Examples of this in APIs are `close`, and `pthread_mutex_destroy`.

		    In the second case, the service owns the object, so it is responsible for deallocating it.
			The client can pass a reference to the object, but since it passes ownership, it is possible that the service deallocates it.
			Thus, it cannot be dereferenced by the client.
		2. A second service simply takes a string as an argument, and counts the number of words in it.
			In the first case, the client owns the string, thus the service can reference it during the function call, but not after.
			This is the typical case in many of the `string.h` functions.
			The function does some operation that extracts information from the string, and never accesses it again (unless it is passed in again as an argument).

			In the second case, the service is passed the ownership over the string when the API is called to the service.
			The service, thus, must `free` the string.
			This is *not possible* with a handle-based approach as the service (e.g. in the kernel) cannot `free` a string in user-level (the `malloc` implementation is a library in user-level).
			If the service is a library, then it can free the reference (a pointer to the string).
			But imagine how awkward this is: you ask a function to count the number of words in a string, and now you're not allowed to access the string anymore!
			Yikes!

		A third examples demonstrates how libraries often structure their allocation and deallocation structures to take advantage of the ability to statically (on stack or in global memory) allocate structures.

		```c
		// Caller owns r, useful when this is necessary:
		// statically allocated r structures
		int res_init(struct resource *r, int f, ...) {
			*r = (struct resource) { .field = f, ... };

		    return 0; // initialization might fail based on arguments
		}

		// Callee owns r
		struct resource *res_alloc(int f, ...) {
			r = malloc(sizeof(struct resource));
			if (r == NULL) return NULL;

			init(r, f, ...);

		    return r;
		}

		// Callee *maintains* ownership of r's memory
		void res_deinit(struct resource *r) {
			// do whatever we need to do to free the associated resources
			memset(r, 0, sizeof(struct resource));
		}
		// Callee passes ownership to this function for us to free
		void res_free(struct resource *r) {
			res_deinit(r);
			free(r);
		}
		```
		Most users of the API will use `res_alloc` and `res_free`.
		However, if the `struct resource` is allocated on the stack, or in global memory, then we need to have a means to initialize (and deinitialize) the resource that does *not* include memory allocation.
		Note the stark difference in ownership between the `init`/`deinit` calls, and the `alloc`/`free` calls.
	- *Thread safety* -
		Can a function be called by two threads safety?
		The word "safely" is doing a lot of work here.
		The data-structures (state) that backs the API might suffer from *race conditions*, which only happen non-deterministically, but are decidedly not safe.
		A decent mental model is that all thread safe APIs are using locks behind the wraps to prevent races.
		If a function is *not* thread-safe, then you must provide your own locks to protect access to the API.
	- *Reentrancy*^[The wikipedia page for this is not good, so I'm not linking to it. Take that.] -
		A function is reentrant if it can be called within a signal or interrupt.
		The dangerous case is this: we're executing in a thread, and call the function $f()$; a signal or interrupt is triggered which saves our current register context, and executes the handler (interrupt service routine, or signal handler) *in the context of the current thread*; and we call $f()$.
		If $f$ uses global data-structures, we might be in a precarious position where half-modified data in the first call is then re-modified by the *nested* call to $f$.
		Functions that can operate correctly in these circumstances are reentrant (we can re-enter them safely), and those that cannot are non-reentrant.

		It is so hard to make functions with significant complexity reentrant, that most modern signal handlers only do very simple things.
		For example, `malloc` is not reentrant, so you cannot dynamically allocate memory in a signal handler.
		A common design pattern is for the signal handler to only write a byte to a pipe, and return.
		The main code (outside of the signal) can detect that the pipe has data, and handle the event at that point.

		Reentrancy is often confused with thread safety, and it superficially has a lot of similarities: the lack of it only causes problems non-deterministically, and there is a feeling that they both are related to race-conditions.
		However, they are not the same.
		In fact, often, when a function is thread safe (and uses locks), it is almost never reentrant.
		If we call the function, take the lock, then get a signal which calls the function and tries and take the same lock, we have a very bad situation.
		This is why `malloc` is not reentrant.
		Note that this example shows that signals and locks do not compose.
	- *Stateless and immutable APIs* -
		Stateless, or *pure* functions are those in which output depends *only* on the input, and the input is left unchanged.
		Specification and testing of pure functions is often much simpler than for those that have state.
		Specification need only be in terms of the input arguments, and not on the state of the underlying system.
		Testing need only consider permutations on input arguments, and not also on the various permutations on hiddlen system state.

		That said, functions that are stateless are often not very useful in systems.
		If a function can be computed only on the inputs, then why call down to the system to perform the operation -- instead just implement the question in a library?

		We might consider functions that access only *logically immutable structures* as a relaxation on stateless functions.
		Their functionality depends on hidden state beyond the arguments passed in, but if that state is not modified, we can get some testing and specification benefits.
		An example of this is system calls that are made only on subsets of the file-system namespace that can be read, but not written.
		Another example, `getpid()`.
	- [*Idempotent APIs*](https://en.wikipedia.org/wiki/Idempotence#Computer_science_examples) -
		Idempotency is more realistic for system APIs.
		Multiple calls to the same API function from a client does not result in a change the return value nor the visible state of the system.
		Intuitively: Can we make the same call twice, and both times it has the same impact?

	    Idempotency is expected in many situations.
		For example, we expect that two executions of `$ ls` will print out the same values.
		Even `echo` is idempotent (with used with the same argument): `$ echo "hi!" > file` results in `file` holding the expected value; as does `$ echo "hi!" > file; echo "hi!" > file`.
		For webpages, we expect that two `GET` requests (to retrieve some resource on a webpage) will return the same value.
		Loads to the same address, and stores to the same address are, on their own idempotent.

		Note that idempotency is not typically seen to apply across different API calls (it does not apply to function compositions).
		`read` is not idemponent as every time we call it we get a non-overlapping section of the file (the "next" part of the file).
		However, we can write a `readfile` function that reads the entire contents of the file, and that function will be idempotent, so long as no other thread does a `read` on the backing file-descriptor (thus causing `readfile` to skip data).
		The [fetch and add (`faa`) instruction](https://en.wikipedia.org/wiki/Fetch-and-add) is also not idempotent when asked to add non-zero values as every time it is called it will return the (updated) value of the counter, then add to it.

		What are the *implications of idempotency* of a function in an API?
		Generally, idempotency is a strong indication that the backing data can be *cached*.
		`read`s on a pipe have very different properties than on a file.
		There is no way to retrieve the same data out of the pipe as once it is read once, it is gone.
		Thus pipe APIs aren't idempotent, thus pipe data cannot generally be cached.
	    We'll revisit idempotency when we discuss event management.

- *Commutative APIs* -
	Functions are commutative if they can be reordered and still result in a semantically identical result.
	We've seen this mathematically in that `a + b = b + a`.
	In a CS setting, examples are closer to

	```c
	fd1 = open("file1", ...);
	fd2 = open("file2", ...);
	```
	versus

	```c
	fd2 = open("file2", ...);
	fd1 = open("file1", ...);
	```
	How can we assess if these commute?
	The only observable effect of opening the file is the return values.
	So the question is, do these two options commute with respect to (WRT) the specification?
	You may be surprised to know that they *do not* on POSIX systems.
	The file descriptor returned from any system call that returns a new file descriptor must be the lowest numerical `fd` that is not currently in use.
	Thus the previous two operations do *not* commute as the functions are *guaranteed* to return different values depending on the order of their execution.
	Changed semantics that simply specify that the `fd` returned can be any unused `fd` make `open` commutative with respect to itself.
	This is somewhat confusing, but what matters the most here is commutativity with respect to the *specification*.

	API commutativity has a large implication on parallel scalability of the system, a correlation we'll discuss later.

- *[Simplicity](https://en.wikipedia.org/wiki/KISS_principle).*
	It is a trope to "keep your code simple", but it is very rarely executed.
	Avoiding unnecessary optimizations and generality helps, but simplicity can be encouraged in design.
	Simple code often comes from asking what assumptions you can engineer into your code.
	If you can statically allocate all data-structures, and avoid `malloc` (and, more importantly, `free`).
	If you can ensure that data-structures or data adhere to constraints (e.g. circular doubly linked lists can avoid conditionals), you can avoid superfluous edge-cases.
	Generally, one need be willing to *rewrite* your code and *redesign* when you find that there is a way to simplify logic.
	This is a higher-order optimization that should span your focus from the minutia of the code, up to module and inter-module design.

### Reading

- Chapters 1 & 2 in "Principles of Computer System Design: An Introduction" by Jerome H. Saltzer and Frans Kaashoek the latter chapters of which are [free online](https://ocw.mit.edu/resources/res-6-004-principles-of-computer-system-design-an-introduction-spring-2009/online-textbook/).
- [Lamport](https://www.dropbox.com/sh/4cex542zznbjh7b/AADM59pqAb9YBy4eeT1uw0t8a?dl=0)
- [The Art of Unix Programming](http://www.catb.org/~esr/writings/taoup/html/index.html)

## OS Design and Trade-offs

Necessary background concepts:

- Resources need to be controlled with policies, and those polices must be expressed in code.
- The kernel is only special as it has de-facto access to all resources.
- If the kernel provides a means for accessing resources to processes, and a subset of resources is given to a process, then that process has great power (over those resources), but great responsibility (to manage those resources well).
- If the system provides a means to *delegate* resources from one process to another, then access to the resource can be controlled *and extended* by the original resource holder; and if resource delegation can be delegated, then the ability to *manage* resources can be extended to other processes.
- In both cases, the question how to revoke access must be answered.
- Being concrete about resources, we can start by talking about those at the lowest-level:

	- CPU
	- Physical pages of memory
	- Interrupts
	- IPIs
	- Timers and timer interrupts
	- Devices (memory mapped) -- virtual access to specific physical addresses

	...or we might think about this at a higher-level

	- [system calls](https://www.youtube.com/watch?v=xHu7qI1gDPA&list=TLPQMjIwNTIwMjA1MAN_H1P-DA)
	- files
	- network sockets
	- pipes
	- shared memory
	- threads
	- processes
	- containers

- Case in point: We have $N$ pages of memory, who should be able to access them? The system must have a function $owner(n) \to \{ (p_0, va_i), (p_1, va_j), \ldots \}$, which is to say, that for frame $n \leq N$, it is mapped into a set of processes at specific virtual addresses. What in the system defines this function, $owner$?

At the highest level, OSes provide the following major functions:

1. sharing of hardware between different principals,
2. providing functionality, abstractions, and services for principals to harness, and
3. ensuring protection between principals.


### Example Design Issues

- Composability and tightening designs: recursive/[reentrant](https://en.wikipedia.org/wiki/Reentrant_mutex) locks/mutexes and why many [believe](http://www.fieryrobot.com/blog/2008/10/14/recursive-locks-will-kill-you/) they [should](http://www.zaval.org/resources/library/butenhof1.html) be [considered](https://blog.stephencleary.com/2013/04/recursive-re-entrant-locks.html) [harmful](https://inessential.com/2013/09/24/recursive_locks)

	- Code should, to the maximum extent possible, understand the state of the system necessary for it to properly execute.
		Not understanding if a lock is taken or not, and having that be dynamically determined is hiding important details from the programmer.
		Thus, it encourages the programmer & maintainer to no think hard about the locking structure of the system.
	- Generally holding a lock should be as short as possible.
		Recursive locks encourage you to have two separate understandings of the critical section length, conflated into the same code.
	- This encourages a notion of rentrancy: you're in a critical section, and start executing in a *new* critical section that accesses the same internal data-structures.
		This means that you need to understand that the critical section you're half way executing through can be interfered with by the recursive call.
		Thus, it doesn't prevent something that you don't want to happen.
	- Condition variables (`cv`s) interact horribly with this.
		If you wait on a `cv`, you release the lock...which might release it in an outer critical section not designed to be preempted via the `cv` API, or not release the lock at all (thus deadlocking)!
		Thus, these don't compose with condition variables.

	> Recursive mutexes are a hack. There's nothing wrong with using them, but they're a crutch. Got a broken leg or library? Fine, use the crutch. But at least be aware that you're using a crutch, and why...
	> - Dave Butenhof

	>  ...a thread tries to lock a lock that’s already, well, locked, you’ll deadlock. This is good. I like this, just as I like crashes. I know, you think I’m crazy, but if you deadlock on your own thread, then you know you just did something bad.
	> - [FieryRobot](http://www.fieryrobot.com/)

	> Recursive locks let you avoid design. If you're trying to avoid system design, just put the lock down, and step away.
   	> - Gabe

- Needing to weaken an abstraction for performance: [access times](https://en.wikipedia.org/wiki/Stat_%28system_call%29#Criticism_of_atime) on files (`relatime` as default in Linux, see `man mount`)
- Non-composable APIs: [file locks](https://gavv.github.io/articles/file-locks/) [considered](http://0pointer.de/blog/projects/locking.html) [harmful](https://www.samba.org/samba/news/articles/low_point/tale_two_stds_os2.html)

	- `fcntl(F_SET_LK...)`, `lockf(...)` (the POSIX options)-

		- locks associated with *process* thus don't help with multi-threaded access (first unlock, will unlock for all threads, and "retaking" will not count or prevent access), thus it must be paired with in-proc locks; the process-centric view is not "future proof" -- the intent was for the lock to have a "context", and instead of specify it as such, they chose the only current existing context, a process, which subsequently breaks when we now have threads
		- the lock is released on the `close `of *any* fd referencing that file, so, e.g. doesn't compose with `getpwuid` which accesses `/etc/passwd` -- no composability;
	- `flock` - does not enable record (byte range in file) blocking, all or nothing!
	- ...and these different APIs often don't interact! So do they actually provide utility?
	- All approaches: what about untrusted processes not releasing your file's lock? Permissions impact liveness! Not composable.


<!--
### Specification

- services as state & state transitions
- mathematical notation and description
- files and directories
- tighter or looser designs/specifications
- Hyrum's law, which states:

	> With a sufficient number of users of an API, it does not matter what you promise in the contract. All observable behaviors of your system will be depended on by somebody.
-->

### Layering and Structure

- Layering as a fundamental system structuring technique ([THE](https://dl.acm.org/doi/10.1145/363095.363143) OS)

	0. interrupts + scheduling
	1. memory allocation
	2. console
	3. I/O (paging/buffering)
	4. user programs
	5. "the user"

	Motivated by multiple desirable properties:

	- testing via layers
	- restricted reasoning about 1. current layer, and 2. the *abstraction* of lower layers (which have already been reasoned about).
	- isolation is cross cutting

	Limitations:

	- Debugging of lvl 0 & 1 without printing?
	- No memory allocation in scheduling?
	- Where is a "thread" structure implemented?
		Needed for scheduling, but also "associated" with memory allocations and I/O.

- [Microkernels](http://people.cs.uchicago.edu/~shanlu/teaching/33100_fa15/papers/nucleus.pdf) as a structuring technique
- Design via the separation of [mechanism and policy](https://dl.acm.org/doi/10.1145/1067629.806531)
- Concurrency-driven event notification

### Intuition: Vertical vs. Horizontal Isolation

We add isolation into systems to decouple different bodies of software (data and functionality).
We do this to

1. ensure that two bodies of software that should have no trust in each other, can only impact each other to a controlled extent (we are sharing a system's resources, so it is difficult to remove all impact), and
2. to concentrate privileged access to resources between different services and applications -- thus enabling a file system to have enhanced access to all file data, and to provide the necessary logic to *mediate* client access to that data.

An intuitive way to understand isolation properties is by considering both *vertical* and *horizontal* isolation.

**Vertical** isolation implies a logical *dependency* between a client, $p^c$, and a server $p^s$ that entails some amount of client trust in the server's logic.
Dependencies can take many forms.
These include:

- a *synchronous functional dependency* in which a client invokes a function provided by a server -- examples include system calls to the kernel from applications, hypercalls in hypervisors, and synchronous IPC to servers in $\mu$-kernels,
- a *resource dependency* whereby a resource is consumed by a, or on behalf of a client, while that resource is managed by a server -- examples include CPU time managed by a scheduler, and networking throughput consumed for a client.

Isolation of this type is quite complicated and must consider all of the bad things clients can try to do to servers, and all of the foreseeable bugs -- and their impact -- in servers.
We have to consider mundane abuse by passing all configurations of parameters (for functional dependencies), resource accounting/consumption attacks in which the client attempts to use resources in the server without them being properly accounted to that client, and denial-of-service attacks.
The focus is very much on isolating the server from the client (think: the kernel from applications), but also on providing a maximum isolation of the client from the server (i.e. limiting the *trust* the client must have in the server).

Properties of this isolation:

- It is often highly asymmetric: the client depends on the proper functioning of the server, but not vice-versa.
	Different technologies might change this trust relationship.
- As such, the focus is on ensuring that server memory and data-structures are not accessible from clients.
- It is not uncommon for the server execution to occur within the scheduling context of the client (e.g. a kernel system call is accounted to the client requesting that service).
	This does require some amount of trust as a server that goes into an infinite loop does expend the client's cycles.
- It is very important that the server maintains control over how it begins execution, and maintains that control (i.e. system calls trap *only* to kernel instructions configured by the kernel itself).
- The key is that the client and server (by their very nature) interact in controlled ways.

**Horizontal** isolation involves any two clients which each both depend on sets of servers.
The strength of horizontal isolation depends on the strength of the vertical isolation for each server.
For example, if a client can compromise one of these servers through a functional dependency, all other clients of that server might suffer decreased or errant service.
A focus of horizontal isolation is on strong inter-client isolation.
The focus of the system is often of implementing the shared server in such a way that it can provide its service to all of its clients correctly, while also ensuring that client's have limited interference on each other.

Properties of this isolation:

- The main goal is to simply prevent interactions between horizontally isolated clients, by default.

**Isolation intuition examples.**

- Monolithic OSes provide vertical isolation (via dual-mode hardware) of the kernel from user-level applications, and require strong trust of applications on the kernel.
	They focus on horizontal isolation of processes from each other.
- Virtual Machines focus on vertical isolation of the hypervisor, but prioritize horizontal isolation of the majority of the system's software in VMs.
	Within the VM is almost always a monolithic system.
- Containers modify a monolithic system but actually *don't change the structural isolation of the system*.
	This points to the fact that this model of thinking about isolation is not sufficient.
- $\mu$-kernels must be *designed* to discriminate between horizontal and vertical isolation (see [Figure 1](http://www.minix3.org/docs/login-2010.pdf) for an example).
	Clearly, applications are isolated horizontally, but services that they leverage have components of being (of course) vertically isolated from clients, but *also* horizontally isolated as they might leverage other system services.

### Detailed Isolation Model

Horizontal and vertical isolation provide some intuition about what types of isolation are desirable.
However, more fidelity is necessary to capture more complicated relationships.
When you need to describe something with high specificity, you kind of have to math it.

- $p_i \in P$ is one of the protection domains in the system.
- $d^{\text{op}}(p_i) = \{d_{j \neq i}, \ldots\}$ is the set of protection domains that $p_i$ depends on for dependencies of operation type $\text{op}$.
- Dependency types (not an exhaustive list) including $\text{op} \in \{ \text{sync, async, strm, cpu, mem, boot}\}$ which are (respectively)

	- *synchronous function invocation* from $p_i$ to $p_{\text{sync}} \in d^{\text{sync}}(p_i)$,
	- *asynchronous request and response* from with request made from $p_i$ to $p_{\text{async}} \in d^{\text{async}}(p_i)$, and the an eventual reply from $p_{async}$,
	- *data is streamed* from $p_{\text{strm}} \in d^{\text{strm}}(p_i)$ to $p_i$ (this odd ordering is simply saying that the "dependency" of the consumer on the producer),
	- *management of resources* by $p_{\text{cpu|mem}} \in d^{\text{cpu|mem}}(p_i)$ for $p_i$ -- these are schedulers, or memory managers, and
	- *a booter* $p_{\text{boot}} \in d^{\text{boot}}(p_i)$ that is responsible for creating the $p_i$ process

- $o(p_j) = \{o_x, \ldots\}$ is the set of abstract objects *exported* to clients from service $p_j$ and accessible through synchronous function invocations.
	These objects might include files, pages of memory, network sockets, IPC end-points, etc... and $p_j$ is the system service managing those abstractions.
- $n(p_i, p_j) = \{o_x, \ldots\} \subseteq o(p_j)$ is the set of objects accessible via synchronous invocations from $p_i$ and provided by $p_j$ (where as we'll see soon, $p_j \in d^{\text{sync}}(p_i)$).

Note that the dependency relation is *transitive* for a subset of the dependency types, $\{\text{mem, cpu, strm}\}$ such that $\forall_{o \in \text{\{mem, cpu, strm\}}} p_j \in d^o(p_i) \wedge p_k \in d^o(p_j) \to p_k \in d^o(p_i)$.
All protection domains in the system that provide memory and CPU time must be depended on continuously to provide those services.
For streaming dependencies, all "upstream" protection domains must continue to do their job for a process to do useful work.

The rest of the dependency relations are more local, thus are not transitive (including $\{sync, async, boot\}$).
If $p_c$ can make a synchronous invocation to $p_s$, it does *not* mean that $p_c$ can  make synchronous invocations to $p_{s'}$, despite $p_{s'} \in d^{\text{sync}}(p_s)$.
Note that this is *essential* as it supports information hiding and separation of privileges -- a user application should *not* be able to ask a driver to directly write to disk; it *must* go through the file system to validate the request!
Similar arguments can be made for $\text{async}$ and $\text{boot}$.

Lets revisit the examples:

- Monolithic system is simply a set of processes $A = P \textbackslash p_{\text{kern}}$ (note that $A\textbackslash a$ means the set resulting from removing $a$ from set $A$), and the kernel $p_{\text{kern}}$, where $\forall_{p_i \in A, o \in \text{op}} d^o(p_i) = p_{\text{kern}}$.
	This shows the centrality and concentration of trust in the large kernel.
- VMs simply add a new layer to this: a set of VMs, $V$, with each VM, $V_x \in V$, consisting of a set of processes and kernel, $V_x = \{p_{i,x}, \ldots\} \cup p_{\text{kern}_x}$, with similar trust relationships $\forall_{V_x \in V, p_{i,x} \in V_x, o \in \text{op}} p_{\text{kern},x} \in d^o(p_{i,x})$.
	Each VM kernel and application has specific dependencies on the hypervisor, $p_{\text{hv}}$: $\forall_{V_x \in V, o \in \text{op}} d^o(p_{\text{kern},x}) = \{p_{\text{hv}}\}$ and $\forall_{V_x \in V, p_{i \neq \text{kern},x},  o \in \{\text{cpu, mem}\}} p_{\text{hv}} \in d^o(p_i)$.
	It is necessary for proper (horizontal) inter-application protection that applications themselves cannot directly make requests of the hypervisor, and instead must go through their guest kernel.
	This structure also demonstrates that for applications in *different* VMs to interact they must do so by making requests through their kernel, and through the hypervisor.
	As such, VMs form a highly *hierarchical* structure.
- $\mu$-kernels are complicated here, and I'll delay their discussion till we talk about component-based systems.
	The key insight is that the protection domains in the system define if we are using vertical or horizontal isolation by controlling the inter-protection-domain relations, and which protection domains are in charge of what resources.

### Questions

The OS must include services that provide functionality to multiple application principals.
This raises more questions than you'd likely expect:

- **Q1.** Do we consider protection in spite of compromises and faults in OS services?
- **Q2.** What failures do we try and recover from, and which do we consider terminal?
- **Q3.** The implementation of a service requires committing some resources (CPU processing in the service, and memory `malloc`ed).
	How do we provide protection between principals, given this?

- **Q4.**
	What is the *core* of many of the issues in the [comparison between POSIX and Win32](https://www.samba.org/samba/news/articles/low_point/tale_two_stds_os2.html)?
- **Q5.**
	Imagine we create a `int close_unlink(int fd, char *path)` that combines close and unlink.
	Is this useful?
	When?
	How would you go about assessing the utility of the call?
	When could it fail?
- **Q6.**
	API question for locks: do you

	1. publicize that a failure (`assert`) will happen if you recursively take a non-recursive lock,
	2. deadlock on your own contention, or
	3. do you return an error value?

- **Q7.**
	Do you implement malloc that is thread safe, or not?
	How about signal-safe?
	Or do you publish that it is not thread safe, and rely on the user to use it safely (e.g. with a lock)?

How we think about system design matters (i.e. our mental model for the system design).
We might think of a "flat" system as a microkernel, that is often visualized with servers all at user-level, on the same "level".
The emphasis is "servers" aren't special, and are user-level protection domains, just the same.
On the other hand, we think of layered system structures with a total order on software subsystem dependencies, like THE.
All systems must be layered to some degree if they use dual-mode hardware (the kernel layered below the rest).
However, this mental model is overly-simplistic.

**Q8.**
List the differences between two different relationships between protection domains, $p_0$ and $p_1$:

1. $p_0$ uses IPC to harness some functionality in $p_1$, thus $p_1$ provides a *service* to $p_1$ which is a client, and
2. $p_0$ and $p_1$ are siblings (or "peers") in that they both use IPC (or generally some communication mechanism) to harness the functionality of some services, with some intersection of those harnessed services, $\{p_2, p_3, \ldots, p_n\}$.

Start with some examples of both structures in systems your familiar with.
Specifically focus on where attacks are, where "*trust*" is required (and trust in what dimension -- from full trust, "they can crash me at any time, and can access all of my secrets", to more nuanced trust, "they can access none of my resources, but IPC is synchronous, thus I trust them to return in a predictable amount of time").
