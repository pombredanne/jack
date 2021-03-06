jack
====

Java bytecode to C++ transpiler to ahead-of-time compile Java code. 
This can be useful if you want to target operating systems that don't 
allow setting the executable flag on memory pages.

## Goals
Jack tries to provide a reasonable runtime environment for Java 
applications. The following features should be supported:

- Java 1.5 language compatibility (no annotations)
- garbage collection
- reflection (with limitations, see below)
- threads
- exceptions (with limitations, see below)
- JNI (with limitations, see below)
- alternative native code interface

## Non-Goals
- loading bytecode at runtime
- full JRE, say good bye to all those EE classes
- Java memory model for the most part (could potentially honor volatile fields/vars)
- anything else that's not listed under goals

## Code Overview
Jack's code is found in the jack/ folder and is organized as follows.

### Packages
- ```com.badlogic.jack```: main package, containing the main entry point, ```Jack```
- ```com.badlogic.jack.info```: classes that store additional information for classes, methods and fields.
- ```com.badlogic.jack.generators```: classes that generate parts of the c++ output.

### Flow
- ```Jack.java``` is the main entry point. It generates header and implementation files from a directory full of ```.class``` files.
- As a first step metadata (aka info) is collected for all classes, methods and fields. See the ```com.badlogic.jack.info``` package
- Based on that info, header files are generated for each class, see ```com.badlogic.jack.generators.HeaderGenerator```
- Next, implementation files are generated for each class, see ```com.badlogic.jack.generators.ImplementationGenerator```. The process goes like this:
  - Methods are emitted, via ```com.badlogic.jack.generators.MethodGenerator```, which in turn uses a series of other generators. During this phase additional information is added to the info classes, such as string literals.
  - ```<clinit>``` is emitted, via ```com.badlogic.jack.generators.ClinitGenerator```, which emits static field initialization codes, string literal initialization code and the original ```<clinit>``` method body of the class if present.
  - Header and static fields sections are emitted, via ```com.badlogic.jack.generators.StaticsGenerator```.
- Finally, runtime startup functions and reflection information is generated, via ```com.badlogic.jack.generators.RuntimeGenerator```.

### Runtime Library
Jack contains a minimal runtime library in runtime/jack-kernel. It heavily borrows from Avian VM. The runtime/jack-tests folder contains
a few basic tests that check language feature implementation and the runtime library. Neither of them links to the normal JRE library 
since they themselve implement "All The Things".

### Execution
Currently Jack is tested through compiling and running parts of the runtime library and tests. The simplest way to do this
works as follows:

- Import all three projects (jack/, runtime/jack-kernel/, runtime/jack-tests) into Eclipse. This will put ```.class``` files for both library implementations in their respective bin/ folder.
- Excecute ```Jack```, passing in three parameters specifying:
  - the directory the ```.class``` files reside in (e.g. ```../runtime/jack-kernel/bin;../runtime/jack-tests/bin```, to include both the runtime library and the tests)
  - the directory the ```.java``` files reside in (e.g. ```../runtime/jack-kernel/src;../runtime/jack-tests/src```). This is used to add Java source file lines to the C++ files for easier debugging.
  - the output directory (use ```native/classes``` for now).
- Once Jack is done, fire up Visual C++ 2010 (Express) or Xcode and load the corresponding project (```vs10```, ```xcode```).
- Delete all the files in the ```classes``` filter/group and reimport everything from ```native/classes```. You'll have to redo this step everytime you add a new Java source file to the projects you translate to C++.
- Open up ```native/jack.cpp```, mess around with the code to test classes or features, and compile, run and debug it.
- If you change a runtime library or test Java file, execute ```Jack``` again, and wait for Visual Studio/XCode to pick up the changes, then recompile.
- If you add or remove a file, you'll have to reimport all .cpp files from the ```native/classes``` folder.

This process is meant for developing Jack itself. End-users will get a nicer way to transpile and run their stuff.

# Done
- class hierarchy translation
- method body translation
- GC (Boehm GC for now)
- initial classpath implementation based on Avian's classpath (misses JNI parts)
- reflection: fix instanceof, what was i thinking :p
- reflection: class descriptors for arrays
- custom native code bridge (add @DirectNative to class or method, implement missing native methods directly in C++, see jack/native/vm/java_lang.cpp for implementations of Object and Class native methods.
- partial recompilation, means we only touch .h/.cpp files that actually changed. brings down c++ compile times enormously.

# TODO 
(in order, search for FIXME <task> in the code)
- array covariance, e.g. Object[] arr = new String[10]. Damn it.
- exceptions: set signal handlers, use setjmp, finally is thankfully 
handled by javac
- reflection: rest of class/constructor/method/field descriptors
- reflection: method#invoke, newInstance, newArray, might get away with 
not using libffi
- jni: minimally viable product :p
- threads
- add the rest of Avian's classpath + JNI implementations.
- add unit tests, get some from Avian, see if OpenJDK has anything 
useful.

## Translation
The following describes how the Java bytecode is translated to C++.

### Class hierarchies
Java class hierarchies are directly mapped to C++ class hierarchies. 
That way i don't have to implement my own vtable abstraction. Some 
funkiness is involved to cope with bridge methods and covariant return 
types, which usually turn up if you have a concrete generic class, e.g.
```List<String>#get(0)```. Covariance itself is supported by C++, however, 
the covariant return type needs to be complete (fully defined). Forward
declarations do not work, instead the full class declaration of a 
covariant return type is included in the header the class that has the
return type covariant method. This might lead to cyclic dependencies in
some ill-conceived cases.

The full Java class hierarchy is retained in the C++ class hierarchy.

Fields and methods are added as is, with prefixes (f_ for fields, m_ 
for methods).

### Strings
We use the String implementation by Avian, which includes UTF8
handling. Strings are hence stored as 16-bit character sequences.
String literals are constructed in a funky way, you don't want
to know how...

### Class initialization
The JVM usually calls ```MyClass#<clinit>``` the first time that class is 
referenced at runtime (see the Java language specs and JVM specs for 
more info). In an ahead-of-time compiled environment this would be to
expensive as tons of methods would have to be augmented with checks 
whether ```<clinit>``` was already called.

Jack initializes all classes at start up, in an order that guarantees 
that guarantees that all classes a class depends on are initialized 
before the class itself is initialized.

### Types
All java primitive types are mapped to equivalent C++ types, see 
vm/types.h (to be extended for more platforms, looking at you long 
long). Reference types are always represented as pointers, e.g. a field
of type ```java.lang.Object``` would translate to ```java_lang_Object*```.

The array type is special and implemented in C++, subclassing
```java.lang.Object``` (see vm/array.h). Multidimensional arrays are just
nested C++ Array instances. The transpiler will replace array access
with calls to equivalent methods/operators of the C++ Array class.

### Reflection
Reflection data is created before class initialization. It includes
class, constructor, method and field descriptors. The descriptors are
actually modelled via Java class instances, see classpath/src/java/lang/reflect
and classpath/src/java/lang/Class. 

Only a minimal set of reflection capabilities will be exposed. These 
are modelled after Avian VMs classpath (but don't use Avian's 
implementation, just the APIs which are a subset
of the JRE reflection classes). Using Java class instances has the
benefit that all the pesky reflection code can be written in Java
instead of C++, not including creating new class/array instances
and invoking methods.

These three operations are performed in native code. The reflection
classes have native methods which implement that functionality.

Special care needs to be taken for array class descriptors. The 
strategy would be to collect all array types in the entire program
and generate class descriptors for all of those. A concrete Array
implementation would then refer to the precompiled class descriptor.
This might blow up with arrays created via reflection if the array
is never used (e.g. assigned to a variable with an array type), but
instead treated and stored as an object. Very unlikely, but could 
happen.

Primitive types also have class descriptors, those need to be
generated and stored somewhere as well. I guess it makes sense
to keep them at the same place as the array class descriptors.

### Method body translation
Jack uses Soot, a Java bytecode analysis framework. It parses
Java bytecode and translates it into 3-address code representation
called Jimple which is like an infinite register machine representation.
This has benefits over the Java bytecode stack machine representation. 
Stack machines are load/store heavy. 3-address code as generated
by Soot is also load/store heavy, but can be expressed with a series
of local variables. Those can in turn be easily optimized to use
registers most of the time by any capable C++ compiler. It also
makes it easier for me to directly pass arguments to methods. On 
top of that, many loophole and more complex optimizations can be
implemented directly with Soot, e.g. elimination of many load/stores
(which the c++ compiler already does for us most of the time).

Another benefit of using Soot and it's 3-address code representation
is the fact that there are only 15 different statement types (plus
a few dozen value types). That's a lot more managable than the
200 something JVM instructions. 

The 3-address code can be translated to C++ almost directly. See
all the nasty Compiler#translateXXX methods. 

If the original Java source is available, Jack will add it as 
comments to the C++ source code on a per statement basis. This
makes following and debugging the C++ translation a bit easier.

### Exceptions
Are a strange beast. Two strategies are available:

- setjmp/longjmp: for each try/catch block of a method, a jump
address is pushed onto a thread local stack of jump addresses. When
an exception is thrown, the top of the jump address stack is poped,
and longjmp is called to jump to this address. Execution will continue
in the catch handler which checks if the exception type can be handled.
if it can, the exception handler code is executed. Otherwise the next
jump address is popped from the stack. If no fitting exception handler
can be found, the app will terminate. Local variables of a method
containing a try/catch block must be declared as volatile for this to 
work, which can potentially hurt performance.
- C++ exceptions: originally not considered as they lack finally blocks.
Have to reinvestigate due to a funky finding, see next paragraph.

In both cases we don't have to care about finally blocks. Javac will
inline those in the exception handlers as well as before any return
statements of a method. This makes our life considerably easier.

C++ automatic stack unwinding would be a nice to have instead of the
confusing mess that is setjmp/longjmp. Setjmp/longjmp seem to have
better performance characteristics (not counting the volatile locals).

I guess i'll try both :)

### Threading & Monitors
Threading itself should be straight forward, using whatever non-standard Windows 
supplies as well as Posix on any sane OS. The GC and exceptions
have to be considered here. It looks like Boehm GC is up for the task,
and making jump buffers thread local is trivial should i chose to use
setjmp/longjmp.

Thread interruption will be interesting, especially with regards to
interruptable i/o methods. This is an area where more research is
needed, Avian should give me some hints.

Monitors are a different beast. Every class instance in Java is
potentially a monitor. The mutex has to be stored somewhere, potentially
bloating objects. Here's a nice starting point http://static.usenix.org/event/jvm01/full_papers/dice/dice.pdf

Finally, locks need to be released in case of exceptions. Javac
makes our live a bit easier again by automatically generating
try/catch blocks around synchronized statements/methods:

    public void test() {
       synchronized(this) {
          test2();
       }
	   }

And the corresponding Jimple representation:

```
   (soot.jimple.JimpleBody)     public void test()
    {
        jack.Synchronized r0, r2;
        java.lang.Throwable $r3;

        r0 := @this: jack.Synchronized;
        r2 = r0;
        entermonitor r0;

     label0:
        virtualinvoke r0.<jack.Synchronized: void test2()>();
        exitmonitor r2;

     label1:
        goto label5;

     label2:
        $r3 := @caughtexception;

     label3:
        exitmonitor r2;

     label4:
        throw $r3;

     label5:
        return;

        catch java.lang.Throwable from label0 to label1 with label2;
        catch java.lang.Throwable from label3 to label4 with label2;
    }
```

### JNI
Given full reflection data and a non-moving GC, it should be almost
easy to implement at least the most useful parts of the JNI (lock primitive
arrays, get direct buffer addresses, fetch method ids and invoke them).

Native Java methods need a way to find and invoke their C/JNI counter part. 
Here's how that could be done naively:

- for each native method of a Java class, add a corresponding function pointer
  field in the C++ class.
- for each native method, generate a C++ body in the transpiler that:
  - makes sure any locks are obtained if the method is synchronized
  - checks if the function pointer field is set
    - if no, get the address via libdl or whatever Win32 provides
    - if yes, fetch the address from the field
  - invoke the JNI function

The arguments to the JNI function can be the direct pointers to any Java class
instance. We just hide them behind suitable definitions of jstring, jarray, jobject
and so on. JNIEnv has to be implemented at least to some degree so we can
pass it in as well. A Java class that implements most of JNIEnv might actually work
using the reflection data from other classes and one or two native methods.

Care has to be taken if exceptions are thrown in the native code, or if Java code
is called from the JNI code that itself throws an exception. Research this.

As long as the GC is non-moving we should be mostly safe when passing Java objects
directly. Threading might be a fun cause for heisenbugs in that case. Research this.

Besides a minimal JNI implementation, it would be nice to interface with C/C++
more directly. Figure out a nice way to do that.

### Garbage Collection
Out of laziness Jack uses the Boehm GC for now. Mono uses/used it, so it's good 
enough for this as well. All classes derrive from gc to override the new operator.
Primitive arrays are instantiated via GC_MALLOC_ATOMIC, anything else is allocated
via GC_MALLOC. Since the Boehm GC is non-moving we can be a bit lazy on the JNI side
of things.
