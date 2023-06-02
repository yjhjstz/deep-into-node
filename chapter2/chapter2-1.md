
## C++ and JS interaction
This chapter mainly talks about how to implement JS calling C++ through V8. JS calling C++ is divided into JS calling C++ functions (global) and calling C++ classes.


### Data and templates
Because there are significant differences between C++ native data types and JavaScript data types, V8 provides the Value class, which is used from JavaScript to C++ and from C++ to JavaScrpt. For example:
```c++
Handle<Value> Add(const Arguments& args){
   int a = args[0]->Uint32Value(); 
   int b = args[1]->Uint32Value(); 

   return Integer::New(a+b); 
 }
```
Integer is a subclass of Value.

In V8, there are two template classes (not C++ template classes):
- ObjectTemplate
- FunctionTemplate
These two template classes are used to define JavaScript objects and JavaScript functions. We will encounter instances of template classes in subsequent sections. By using ObjectTemplate, C++ objects can be exposed to the script environment. Similarly, FunctionTemplate is used to expose C++ functions to the script environment for use by scripts.


### JS uses C++ variables
Sharing variables between JavaScript and V8 is actually very easy. The basic template is as follows:
```c++
static char sname[512] = {0}; 

 static Handle<Value> NameGetter(Local<String> name, const AccessorInfo& info) {
    return String::New((char*)&sname,strlen((char*)&sname)); 
 } 

 static void NameSetter(Local<String> name, Local<Value> value, const AccessorInfo& info) {
   Local<String> str = value->ToString(); 
   str->WriteAscii((char*)&sname); 
 }
```
After defining NameGetter and NameSetter, register them on global in the main function:
```c++
 // Create a template for the global object. 
 Handle<ObjectTemplate> global = ObjectTemplate::New(); 
 //public the name variable to script 
 global->SetAccessor(String::New("name"), NameGetter, NameSetter); 

```

### JS calls C++ functions
Calling C++ functions in JavaScript is the most common way to script. By using C++ functions, the ability of JavaScript scripts can be greatly enhanced, such as file reading and writing, network/database access, graphics/image processing, etc., similar to the jni technology of JAVA.

In C++ code, define a function prototype as follows:
```c++
 Handle<Value> func(const Arguments& args){//return something}
```
Then, expose it to the script:
`global->Set(String::New("func"),FunctionTemplate::New(func));`

### JS uses C++ classes
If analyzed from an object-oriented perspective, the most reasonable way is to expose C++ classes to JavaScript. This can greatly increase the number of built-in objects in JavaScript, thereby using the host language as little as possible and making greater use of the flexibility and scalability of dynamic languages. In fact, C++ has many concepts, complex content, and a steeper learning curve than JavaScript. The best application scenario is: the flexibility of scripting languages and the efficiency of system languages such as C/C++. Using the V8 engine, it is very convenient to "wrap" C++ classes into resources that can be used by JavaScript.

Here we give a relatively simple example, define a Person class, and then wrap and expose this class to JavaScript scripts, create a Person class object in the script, and use the methods of the Person object.
First, we define the Person class in C++:
```c++
class Person { 
 private: 
   unsigned int age; 
   char name[512]; 

 public: 
   Person(unsigned int age, char *name) {
     this->age = age; 
     strncpy(this->name, name, sizeof(this->name)); 
   } 

   unsigned int getAge() {
     return this->age;
   } 

   void setAge(unsigned int nage) {
     this->age = nage;
   } 

   char *getName() {
     return this->name;
   } 

   void setName(char *nname) {
     strncpy(this->name, nname, sizeof(this->name));
   } 
 };
```
The structure of the Person class is very simple, only two fields age and name, and each has its own getter/setter. Then we define the wrapper of the constructor:
```c++
Handle<Value> PersonConstructor(const Arguments& args){
   Handle<Object> object = args.This(); 
   HandleScope handle_scope; 
   int age = args[0]->Uint32Value(); 

   String::Utf8Value str(args[1]); 
   char* name = ToCString(str); 

   Person *person = new Person(age, name); 
   object->SetInternalField(0, External::New(person)); 
   return object; 
 }
```
From the function prototype, it can be seen that the wrapper of the constructor is consistent with the wrapper of the function in the previous section, because the constructor is also a function in V8's view. It should be noted that after obtaining the parameters from args and converting them to the appropriate type, we call the actual constructor of the Person class based on this parameter and set it in the internal field of the object. Immediately afterwards, we need to wrap the getter/setter of the Person class:
```c++
 Handle<Value> PersonGetAge(const Arguments& args){
   Local<Object> self = args.Holder(); 
   Local<External> wrap = Local<External>::Cast(self->GetInternalField(0)); 

   void *ptr = wrap->Value(); 

   return Integer::New(static_cast<Person*>(ptr)->getAge()); 
 } 

 Handle<Value> PersonSetAge(const Arguments& args) {
   Local<Object
