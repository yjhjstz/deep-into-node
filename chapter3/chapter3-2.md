
## The Magic of Yield
The introduction of Generators in ES6 has greatly changed the way JavaScript programmers view iterators and provided a new way to solve `callback hell`.

### Generators
The iterator pattern is a commonly used design pattern, but when the iteration rules are complex, maintaining the state within the iterator can be cumbersome. This is where generators come in. What are generators?
> Generators: a better way to build Iterators.

With the help of the `yield` keyword, we can implement the Fibonacci sequence more elegantly.

```js
function* fibonacci() {
  let a = 0, b = 1;

  while(true) {
    yield a;
    [a, b] = [b, a + b];
  }
}
```

### Yield and Asynchronous Operations
`yield` can pause the execution flow, which provides the possibility of changing the execution flow. This is similar to Python's coroutine.

The reason why generators can be used to control code flow is that they use `yield` to switch the execution paths of two or more generators. This switching is at the statement level, not the function call level. Its essence is CPS transformation.

After `yield`, the current call actually ends, and the control has actually been transferred to the function that called the `next` method of the generator externally, accompanied by a change in state. So if the external function does not continue to call the `next` method, the function where `yield` is located is equivalent to stopping at `yield`. So to complete the asynchronous operation and continue the function execution, just call the `next` method of the generator at the appropriate place, just like the function is executing after pausing.

### V8 Implementation

#### Parse Phase
The processing of generator functions and the `yield` keyword is in `parser.cc`. Let's take a look at the AST parsing function: `Parser::ParseEagerFunctionBody()`

```c++
3928 ZoneList<Statement*>* Parser::ParseEagerFunctionBody(
3929     const AstRawString* function_name, int pos, Variable* fvar,
3930     Token::Value fvar_init_op, FunctionKind kind, bool* ok) {
3931     .....
3954   // For generators, allocate and yield an iterator on function entry.
3955   if (IsGeneratorFunction(kind)) {
3956     ZoneList<Expression*>* arguments =
3957         new(zone()) ZoneList<Expression*>(0, zone());
3958     CallRuntime* allocation = factory()->NewCallRuntime(
3959         ast_value_factory()->empty_string(),
3960         Runtime::FunctionForId(Runtime::kCreateJSGeneratorObject), arguments,
3961         pos);
3962     VariableProxy* init_proxy = factory()->NewVariableProxy(
3963         function_state_->generator_object_variable());
3964     Assignment* assignment = factory()->NewAssignment(
3965         Token::INIT_VAR, init_proxy, allocation, RelocInfo::kNoPosition);
3966     VariableProxy* get_proxy = factory()->NewVariableProxy(
3967         function_state_->generator_object_variable());
3968     Yield* yield = factory()->NewYield(
3969         get_proxy, assignment, Yield::kInitial, RelocInfo::kNoPosition);
3970     body->Add(factory()->NewExpressionStatement(
3971         yield, RelocInfo::kNoPosition), zone());
3972   }
3973 
3974   ParseStatementList(body, Token::RBRACE, false, NULL, CHECK_OK);
3975 
3976   if (IsGeneratorFunction(kind)) {
3977     VariableProxy* get_proxy = factory()->NewVariableProxy(
3978         function_state_->generator_object_variable());
3979     Expression* undefined =
3980         factory()->NewUndefinedLiteral(RelocInfo::kNoPosition);
3981     Yield* yield = factory()->NewYield(get_proxy, undefined, Yield::kFinal,
3982                                        RelocInfo::kNoPosition);
3983     body->Add(factory()->NewExpressionStatement(
3984         yield, RelocInfo::kNoPosition), zone());
3985   }
3986    ...

```
L3955 determines whether it is a generator function. `ParseStatementList` parses the function body. Note that a generator function is also a function, and in V8, it is also represented by `JSFunction`.

In the two if function bodies, `Yield::kInitial` and `Yield::kFinal` two Yield AST nodes are created.

Yield states are:

```c++
enum Kind {
    kInitial,  // The initial yield that returns the unboxed generator object.
```

### Codegen phase
The machine code generation (x64 platform) mainly focuses on `runtime-generator.cc` and `full-codegen-x64.cc`.

`runtime-generator.cc` provides stub code segments such as `Create`, `Suspend`, `Resume`, `Close`, etc.,

which are used by full-codegen for inline use to generate assembly code.

Let's take a look at `RUNTIME_FUNCTION(Runtime_CreateJSGeneratorObject)`,

```c++
 14 RUNTIME_FUNCTION(Runtime_CreateJSGeneratorObject) {
 15   HandleScope scope(isolate);
 16   DCHECK(args.length() == 0);
 17 
 18   JavaScriptFrameIterator it(isolate);
 19   JavaScriptFrame* frame = it.frame();
 20   Handle<JSFunction> function(frame->function());
 21   RUNTIME_ASSERT(function->shared()->is_generator());
 22 
 23   Handle<JSGeneratorObject> generator;
 24   if (frame->IsConstructor()) {
 25     generator = handle(JSGeneratorObject::cast(frame->receiver()));
 26   } else {
 27     generator = isolate->factory()->NewJSGeneratorObject(function);
 28   }
 29   generator->set_function(*function);
 30   generator->set_context(Context::cast(frame->context()));
 31   generator->set_receiver(frame->receiver());
 32   generator->set_continuation(0);
 33   generator->set_operand_stack(isolate->heap()->empty_fixed_array());
 34   generator->set_stack_handler_index(-1);
 35 
 36   return *generator;
 37 }
```

The function creates a `JSGeneratorObject` object to store `JSFunction`, `Context`, and pc pointer based on the current Frame,
and sets the operand stack to empty.

After `yield`, the current execution environment is actually saved. L74 saves the current operand stack and saves it to the JSGeneratorObject object.

```c++
 40 RUNTIME_FUNCTION(Runtime_SuspendJSGeneratorObject) {
 41   HandleScope handle_scope(isolate);
 42   DCHECK(args.length() == 1);
 43   CONVERT_ARG_HANDLE_CHECKED(JSGeneratorObject, generator_object, 0);
 44 
 45   JavaScriptFrameIterator stack_iterator(isolate);
 46   JavaScriptFrame* frame = stack_iterator.frame();
 47   RUNTIME_ASSERT(frame->function()->shared()->is_generator());
 48   DCHECK_EQ(frame->function(), generator_object->function());
 49 
 50   // The caller should have saved the context and continuation already.
 51   DCHECK_EQ(generator_object->context(), Context::cast(frame->context()));
 52   DCHECK_LT(0, generator_object->continuation());
 53 
 54   // We expect there to be at least two values on the operand stack: the return
 55   // value of the yield expression, and the argument to this runtime call.
 56   // Neither of those should be saved.
 57   int operands_count = frame->ComputeOperandsCount();
 58   DCHECK_GE(operands_count, 2);
 59   operands_count -= 2;
 60 
 61   if (operands_count == 0) {
 62     // Although it's semantically harmless to call this function with an
 63     // operands_count of zero, it is also unnecessary.
 64     DCHECK_EQ(generator_object->operand_stack(),
 65               isolate->heap()->empty_fixed_array());
 66     DCHECK_EQ(generator_object->stack_handler_index(), -1);
 67     // If there are no operands on the stack, there shouldn't be a handler
 68     // active either.
 69     DCHECK(!frame->HasHandler());
 70   } else {
 71     int stack_handler_index = -1;
 72     Handle<FixedArray> operand_stack =
 73         isolate->factory()->NewFixedArray(operands_count);
 74     frame->SaveOperandStack(*operand_stack, &stack_handler_index);
 75     generator_object->set_operand_stack(*operand_stack);
 76     generator_object->set_stack_handler_index(stack_handler_index);
 77   }
 78 
 79   return isolate->heap()->undefined_value();
 80 }

```


L108 sets the PC offset of the current frame. L118 restores the operand stack, and L126-L130 returns the value based on the restored mode.


```c++
90 RUNTIME_FUNCTION(Runtime_ResumeJSGeneratorObject) {
 91   SealHandleScope shs(isolate);
 92   DCHECK(args.length() == 3);
 93   CONVERT_ARG_CHECKED(JSGeneratorObject, generator_object, 0);
 94   CONVERT_ARG_CHECKED(Object, value, 1);
 95   CONVERT_SMI_ARG_CHECKED(resume_mode_int, 2);
 96   JavaScriptFrameIterator stack_iterator(isolate);
 97   JavaScriptFrame* frame = stack_iterator.frame();
 98 
 99   DCHECK_EQ(frame->function(), generator_object->function());
100   DCHECK(frame->function()->is_compiled());
101 
102   STATIC_ASSERT(JSGeneratorObject::kGeneratorExecuting < 0);
103   STATIC_ASSERT(JSGeneratorObject::kGeneratorClosed == 0);
104 
105   Address pc = generator_object->function()->code()->instruction_start();
106   int offset = generator_object->continuation();
107   DCHECK(offset > 0);
108   frame->set_pc(pc + offset);
109   ...
113   generator_object->set_continuation(JSGeneratorObject::kGeneratorExecuting);
114 
115   FixedArray* operand_stack = generator_object->operand_stack();
116   int operands_count = operand_stack->length();
117   if (operands_count != 0) {
118     frame->RestoreOperandStack(operand_stack,
119                                generator_object->stack_handler_index());
120     generator_object->set_operand_stack(isolate->heap()->empty_fixed_array());
121     generator_object->set_stack_handler_index(-1);
122   }
123 
124   JSGeneratorObject::ResumeMode resume_mode =
125       static_cast<JSGeneratorObject::ResumeMode>(resume_mode_int);
126   switch (resume_mode) {
127     case JSGeneratorObject::NEXT:
128       return value;
129     case JSGeneratorObject::THROW:
130       return isolate->Throw(value);
131   }
132   ...
133 }
```
这边我们关注下 args 参数， args[0]是`JSGeneratorObject` 对象`generator_object`， args[1]是Object 对象
`value`, 也就是 `next` 的返回值，args[2]是表示 resume 模式的值。

对应的我们看到 `FullCodeGenerator::EmitGeneratorResume()` 中的这几行代码：
```c++
2296   __ Push(rbx);
2297   __ Push(result_register());
2298   __ Push(Smi::FromInt(resume_mode));
2299   __ CallRuntime(Runtime::kResumeJSGeneratorObject, 3);
```

L2297从 result 寄存器中取出 value, L2299调用 `RUNTIME_FUNCTION(Runtime_ResumeJSGeneratorObject)`。

这样，从 yield value 到 g.next() 取出 value, 相信大家有了一个大概的认知了。

### 延伸
我们看到node.js依托 v8层面实现了协程，有兴趣的同学可以关心下 fibjs, 它是用 C库实现了协程，遇到异步调用就 "yield" 放弃 CPU，
交由协程调度，也解决了 callback hell 的问题。
本质思想上两种方案没本质区别：
* Generator是利用yield特殊关键字来暂停执行，而fibers是利用Fiber.yield()暂停
* Generator是利用函数返回的Generator句柄来控制函数的继续执行，而fibers是在异步回调中利用Fiber.current.run()继续执行。

### 参考
* http://en.wikipedia.org/wiki/Continuation-passing_style
* https://zh.wikipedia.org/zh-cn/协程
* fibjs https://github.com/xicilion/fibjs
