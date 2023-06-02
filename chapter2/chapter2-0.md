
## V8 concept
### Architecture diagram
![](e09d7b330d9e754f7ff1282a1af55295.png)

The execution process of the JS engine is roughly: source code -> abstract syntax tree -> bytecode -> JIT -> native code.

V8 more directly converts the abstract syntax tree into native code through JIT technology, abandoning some performance optimizations that can be performed in the bytecode stage, but ensuring execution speed.
After V8 generates native code, it also collects some information through Profiler to optimize native code. Although there are fewer performance optimizations in the bytecode generation stage,
it greatly reduces the conversion time.

> PS: TurboFan will gradually replace Crankshaft

Before using the v8 engine, let's first understand a few basic concepts: handle, scope, and context environment (which can be simply understood as the runtime environment).

### Isolate
> An isolate is a VM instance with its own heap. It represents an isolated instance of the V8 engine.
> V8 isolates have completely separate states. Objects from one isolate must not be used in other isolates.

An Isolate is an independent virtual machine corresponding to one or more threads. However, it can only be entered by one thread at a time. All Isolates are completely isolated from each other and cannot have any shared resources. If an Isolate is not explicitly created, a default Isolate will be created automatically.

The concepts of Context, Scope, and Handle mentioned later are all within an Isolate, as shown in the figure below:
![](Context.png)

### Handle concept
In V8, memory allocation is performed in the V8 Heap, and JavaScript values and objects are also stored in the V8 Heap. This Heap is maintained independently by V8, and objects that lose references will be GCed and can be reallocated to other objects. The Handle is a reference to an object in the Heap. In order to manage memory allocation in V8, GC needs to track all objects in V8. Since objects are referenced by Handle, GC needs to manage Handle so that GC can know the reference status of an object in the Heap. When the Handle reference of an object changes, GC can recycle or move the object. Therefore, in V8 programming, a Handle must be used to reference an object, rather than directly obtaining a reference to an object through C++. Directly referencing an object through C++ will make the object unmanageable by V8.

Handle is divided into Local and Persistent.

From the literal meaning, Local is local, and it is also managed by HandleScope.
Persistent, similar to global, is not managed by HandleScope, and its scope can extend to different functions, while Local is local and has a smaller scope.
Persistent Handle objects need to be used in pairs with Persistent::New and Persistent::Dispose, similar to new and delete in C++.

Persistent::MakeWeak can be used to weaken a Persistent Handle. If the only reference Handle of an object is a Persistent, you can use the MakeWeak method to weaken the reference. This method can trigger GC to recycle the referenced object.

### Scope
Conceptually, a scope can be seen as a container for a handle. There can be many handles (that is, many v8 engine-related objects) in a scope, and the objects pointed to by the handles can be released one by one separately, but many times (when you really start writing business code), releasing handles one by one is too cumbersome. Instead, you can release a scope, and all handles contained in this scope will be released uniformly.

There are several scopes in v8.h: HandleScope, Context::Scope.

HandleScope is used to manage Handle, while Context::Scope is only used to manage Context objects.

The code looks like this:
```c++
  // Handles in this function will be managed by handleScope
  HandleScope handleScope;
  // Create a js execution environment Context
  Handle<Context> context = Context::New();
  Context::Scope contextScope(context);
  // Other code
```
In general, a HandleScope is placed at the beginning of a function, so that the Handles in this function do not need to worry about releasing resources.
Context::Scope only does: call context->Enter() in the constructor, and call context->Leave() in the destructor.


### Context concept
Conceptually, this context environment can also be understood as a runtime environment. When executing a javascript script, there are always some environment variables or global functions. If we want to embed v8 engine in our own c++ code, we naturally hope to provide some c++-written functions or modules for others to call directly from the script, so that the power of javascript will be reflected. We can write global functions or classes in c++, so that others can call them directly through javascript, which invisibly extends the functionality of javascript.

Contexts can be nested, that is, when the current function has a Context, if there is another Context when calling another function, javascript in the called function is based on the nearest Context, and when exiting this function, it returns to the original Context.

We can "import" different global variables and functions into different Contexts without affecting each other. It is said that the original purpose of designing Context was to allow the browser to have an independent javascript execution environment for each iframe when parsing HTML, that is, each iframe corresponds to a Context.

#### Different execution contexts in the same scope

![](c3ad9f4a15cb36af631932a52dec3e96.png)

### Relationship
![](1354452360_3578.png)


