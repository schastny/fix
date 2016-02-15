# JVM Internals
[Original](http://www.cubrid.org/blog/dev-platform/understanding-jvm-internals/)

Every developer who uses Java knows that Java bytecode runs in a JRE (Java Runtime Environment). 
The most important element of the JRE is Java Virtual Machine (JVM), which analyzes and executes Java byte code.
 
## Virtual Machine
The JRE is composed of the Java API and the JVM. 
The role of the JVM is to read the Java application through the Class Loader and execute it along with the Java API.

The features of JVM are as follows:
* **Stack-based virtual machine**   
  The most popular computer architectures such as Intel x86 Architecture and ARM Architecture run based on a register. 
  However, JVM runs based on a stack.
* **Symbolic reference**   
  All types (class and interface) except for primitive data types are referred to through symbolic reference, 
  instead of through explicit memory address-based reference. 
* **Garbage collection**  
  A class instance is explicitly created by the user code and automatically destroyed by garbage collection.
* **Guarantees platform independence by clearly defining the primitive data type**   
  A traditional language such as C/C++ has different int type size according to the platform. 
  The JVM clearly defines the primitive data type to maintain its compatibility and guarantee platform independence.
* **Network byte order**   
  The Java class file uses the network byte order. 
  To maintain platform independence between the little endian used by Intel x86 Architecture 
  and the big endian used by the RISC Series Architecture, a fixed byte order must be kept. 
  Therefore, JVM uses the network byte order, which is used for network transfer. 
  The network byte order is the big endian.
  
## Java bytecode
To implement WORA (write once, run anywhere), the JVM uses *Java bytecode*, 
a middle-language between Java (user language) and the machine language. 
This Java bytecode is the smallest unit that deploys the Java code.

The class file itself is a binary file that cannot be understood by a human. 
To manage this file, JVM vendors provide javap, the disassembler. 
The result of using *javap* is called Java assembly. 

## OpCode (operation code) 
There are four OpCodes that invoke a method in the Java Bytecode.
* **invokeinterface** - Invokes an interface method
* **invokespecial** - Invokes an initializer, private method, or superclass method
* **invokestatic** - Invokes static methods
* **invokevirtual** - Invokes instance methods

See also [Opcodes list by Function](http://homepages.inf.ed.ac.uk/kwxm/JVM/codeByFn.html).

```
#>javap -c App
Compiled from "App.java"
public class App extends java.lang.Object{
public App();
  Code:
   0:   aload_0
   1:   invokespecial   #1; //Method java/lang/Object."<init>":()V
   4:   return

public static void main(java.lang.String[]);
  Code:
   0:   getstatic       #2; //Field java/lang/System.out:Ljava/io/PrintStream;
   3:   ldc     #3; //String Hi!
   5:   invokevirtual   #4; //Method java/io/PrintStream.println:(Ljava/lang/String;)V
   8:   return

}
```
In the disassembled result above, what does the number in front of the code mean?  
It is *the byte number*. Perhaps this is the reason why the code executed by the JVM is called Java "Byte"code. 

## Type expression in java bytecode
In the Java Bytecode, the class instance is expressed as "L;" and void is expressed as "V".   

| Java Bytecode | Type | Description |
| --- | --- | --- |
| B | byte | signed byte|
| C | char | Unicode character|
| D | double | double-precision floating-point value|
| F | float | single-precision floating-point value|
| I | int | integer|
| J | long | long integer|
| L\<classname\> | reference | an instance of class \<classname\> |
| S |short | signed short |
| Z |boolean | true or false |
| [ |reference | one array dimension |

## Examples of java bytecode expressions
| Java Code | Java Bytecode Expression |
| --- | --- |
| `double d[][][];` | `[[[D` |
| `Object myMethod(int I, double d, Thread t)` | `(IDLjava/lang/Thread;)Ljava/lang/Object;`
For more details, see "4.3 Descriptors" in "The JVM Spec". 
For various Java Bytecode instruction sets, see "6. The JVM Instruction Set" in "The JVM Spec".

## Class File Structure and Limitations
The 65535 byte limit is one of the JVM limitations, and stipulates that the size of one method cannot be more than 65535 bytes.

A class file consists of a single ClassFile structure (JVM Spec, 4.1. The ClassFile Structure):
```
ClassFile {
    u4 magic;
    u2 minor_version;
    u2 major_version;
    u2 constant_pool_count;
    cp_info constant_pool[constant_pool_count-1];
    u2 access_flags;
    u2 this_class;
    u2 super_class;
    u2 interfaces_count;
    u2 interfaces[interfaces_count];
    u2 fields_count;
    field_info fields[fields_count];
    u2 methods_count;
    method_info methods[methods_count];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

```
xldn4541vdap [shchasta] #>hexdump -C App.class
00000000  ca fe ba be 00 00 00 32  00 1d 0a 00 06 00 0f 09  |▒▒▒▒...2........|
00000010  00 10 00 11 08 00 12 0a  00 13 00 14 07 00 15 07  |................|
00000020  00 16 01 00 06 3c 69 6e  69 74 3e 01 00 03 28 29  |.....<init>...()|
```

With this value, see the class file format.
* **magic**: The first 4 bytes of the class file are the magic number. This is a pre-specified value to distinguish the Java class file. As shown in the Hex Editor above, the value is always 0xCAFEBABE (**feca beba**). In short, when the first 4 bytes of a file is 0xCAFEBABE, it can be regarded as the Java class file. This is a kind of "witty" magic number related to the name "Java".
* **minor_version, major_version:** The next 4 bytes indicate the class version. As the class file is 0x00000032, the class version is 50.0. The version of a class file compiled by JDK 1.6 is 50.0, and the version of a class file compiled by JDK 1.5 is 49.0. The JVM must maintain backward compatibility with class files compiled in a lower version than itself. On the other hand, when a upper-version class file is executed in the lower-version JVM, *java.lang.UnsupportedClassVersionError* occurs.
* **constant_pool_count, constant_pool[]:** Next to the version, the class-type constant pool information is described. This is the information included in the Runtime Constant Pool area, which will be explained later. While loading the class file, the JVM includes the constant_pool information in the Runtime Constant Pool area of the method area. As the constant_pool_count of the App.class file is 0x001d, you can see that the constant_pool has (29-1) indexes, 28 indexes.
* **access_flags:** This is the flag that shows the modifier information of a class; in other words, it shows public, final, abstract or whether or not to interface.
* **this_class, super_class:** The index in the constant_pool for the class corresponding to this and super, respectively.
* **interfaces_count, interfaces[]:** The index in the the constant_pool for the number of interfaces implemented by the class and each interface.
* **fields_count, fields[]:** The number of fields and the field information of the class. The field information includes the field name, type information, modifier, and index in the constant_pool.
* **methods_count, methods[]:** The number of methods in a class and the methods information of the class. The methods information includes the methods name, type and number of the parameters, return type, modifier, index in the constant_pool, execution code of the method, and exception information.
* **attributes_count, attributes[]:** The attribute_info structure has various attributes. For field_info or method_info, attribute_info is used.

The javap program briefly shows the class file format in a form that users can read. When App.class is analyzed using the "javap -verbose" option, the following contents are printed.

```
#>javap -verbose App
Compiled from "App.java"
public class App extends java.lang.Object
  SourceFile: "App.java"
  minor version: 0
  major version: 50
  Constant pool:
const #1 = Method       #6.#15; //  java/lang/Object."<init>":()V
const #2 = Field        #16.#17;        //  java/lang/System.out:Ljava/io/PrintStream;
const #3 = String       #18;    //  Hi!
const #4 = Method       #19.#20;        //  java/io/PrintStream.println:(Ljava/lang/String;)V
const #5 = class        #21;    //  App
const #6 = class        #22;    //  java/lang/Object
const #7 = Asciz        <init>;
const #8 = Asciz        ()V;
const #9 = Asciz        Code;
const #10 = Asciz       LineNumberTable;
const #11 = Asciz       main;
const #12 = Asciz       ([Ljava/lang/String;)V;
const #13 = Asciz       SourceFile;
const #14 = Asciz       App.java;
const #15 = NameAndType #7:#8;//  "<init>":()V
const #16 = class       #23;    //  java/lang/System
const #17 = NameAndType #24:#25;//  out:Ljava/io/PrintStream;
const #18 = Asciz       Hi!;
const #19 = class       #26;    //  java/io/PrintStream
const #20 = NameAndType #27:#28;//  println:(Ljava/lang/String;)V
const #21 = Asciz       App;
const #22 = Asciz       java/lang/Object;
const #23 = Asciz       java/lang/System;
const #24 = Asciz       out;
const #25 = Asciz       Ljava/io/PrintStream;;
const #26 = Asciz       java/io/PrintStream;
const #27 = Asciz       println;
const #28 = Asciz       (Ljava/lang/String;)V;

{
public App();
  Code:
   Stack=1, Locals=1, Args_size=1
   0:   aload_0
   1:   invokespecial   #1; //Method java/lang/Object."<init>":()V
   4:   return
  LineNumberTable:
   line 1: 0

public static void main(java.lang.String[]);
  Code:
   Stack=2, Locals=1, Args_size=1
   0:   getstatic       #2; //Field java/lang/System.out:Ljava/io/PrintStream;
   3:   ldc     #3; //String Hi!
   5:   invokevirtual   #4; //Method java/io/PrintStream.println:(Ljava/lang/String;)V
   8:   return
  LineNumberTable:
   line 4: 0
   line 5: 8

}
```

## JVM Structure
>
Java Source (.java)   
↓  
(javac)   
↓  
Java Byte Code (.class)  
↓   
JVM
* Class Loader
* Execution Engine
* Runtime Data Areas

A class loader loads the compiled Java Bytecode to the Runtime Data Areas, and the execution engine executes the Java Bytecode.

### Class Loader

Java provides a dynamic load feature; it loads and links the class when it refers to a class for the first time at runtime, not compile time. JVM's class loader executes the dynamic load. The features of Java class loader are as follows:  
* **Hierarchical Structure:** Class loaders in Java are organized into a hierarchy with a parent-child relationship. The Bootstrap Class Loader is the parent of all class loaders.
* **Delegation mode:** Based on the hierarchical structure, load is delegated between class loaders. When a class is loaded, the parent class loader is checked to determine whether or not the class is in the parent class loader. If the upper class loader has the class, the class is used. If not, the class loader requested for loading loads the class.
* **Visibility limit:** A child class loader can find the class in the parent class loader; however, a parent class loader cannot find the class in the child class loader.
* **Unload is not allowed:** A class loader can load a class but cannot unload it. Instead of unloading, the current class loader can be deleted, and a new class loader can be created.

Each class loader has its namespace that stores the loaded classes. When a class loader loads a class, it searches the class based on *FQCN (Fully Qualified Class Name)* stored in the namespace to check whether or not the class has been already loaded. Even if the class has an identical FQCN but a different namespace, it is regarded as a different class. A different namespace means that the class has been loaded by another class loader.

### The class loader delegation model
>
Bootstrap class loader  
↑  
Extension class loader  
↑  
System class loader  
↑  
User-defined class loader   
↑  
User-defined class loader  

![class-loader-delegation-model.png](http://www.cubrid.org/files/attach/images/220547/468/290/class-loader-delegation-model.png)

When a class loader is requested for class load, it checks whether or not the class exists in the class loader cache, the parent class loader, and itself, in the order listed. In short, it checks whether or not the class has been loaded in the class loader cache. If not, it checks the parent class loader. If the class is not found in the bootstrap class loader, the requested class loader searches for the class in the file system.
* **Bootstrap class loader:** This is created when running the JVM. It loads Java APIs, including object classes. Unlike other class loaders, it is implemented in native code instead of Java.
* **Extension class loader:** It loads the extension classes excluding the basic Java APIs. It also loads various security extension functions.
* **System class loader:** If the bootstrap class loader and the extension class loader load the JVM components, the system class loader loads the application classes. It loads the class in the $CLASSPATH specified by the user.
* **User-defined class loader:** This is a class loader that an application user directly creates on the code.

Frameworks such as Web application server (WAS) use it to make Web applications and enterprise applications run independently. In other words, this guarantees the independence of applications through class loader delegation model. Such a WAS class loader structure uses a hierarchical structure that is slightly different for each WAS vendor.

If a class loader finds an unloaded class, the class is loaded and linked by following the process illustrated below.

### Class Load Stage

![class-load-stage](http://www.cubrid.org/files/attach/images/220547/468/290/class-load-stage.png)

* **Loading:** A class is obtained from a file and loaded to the JVM memory.
* **Verifying:** Check whether or not the read class is configured as described in the Java Language Specification and JVM specifications. This is the most complicated test process of the class load processes, and takes the longest time. Most cases of the JVM TCK test cases are to test whether or not a verification error occurs by loading wrong classes.
* **Preparing:** Prepare a data structure that assigns the memory required by classes and indicates the fields, methods, and interfaces defined in the class.
* **Resolving:** Change all symbolic references in the constant pool of the class to direct references.
* **Initializing:** Initialize the class variables to proper values. Execute the static initializers and initialize the static fields to the configured values.

### Runtime Data Areas
![runtime-data-access-configuration.png](http://www.cubrid.org/files/attach/images/220547/468/290/runtime-data-access-configuration.png)

Runtime Data Areas are the memory areas assigned when the JVM program runs on the OS.   
**Are created per one thread:**
* PC Register 
* JVM Stack
* Native Method Stack  

**Shared by all threads:**
* Heap
* Method Area
* Runtime Constant Pool

**PC register:** PC (Program Counter) register exists per thread, and is created when the thread starts. PC register has the address of a JVM instruction being executed now.  
**JVM stack:** JVM stack exists per a thread, and is created when the thread starts. It is a stack that saves the struct (Stack Frame). The JVM just pushes or pops the stack frame to the JVM stack. If any exception occurs, each line of the stack trace shown as a method such as printStackTrace() expresses one stack frame.  
![jvm-stack-configuration.png](http://www.cubrid.org/files/attach/images/220547/468/290/jvm-stack-configuration.png)

* *Stack frame:* One stack frame is created whenever a method is executed in the JVM, and the stack frame is added to the JVM stack of the thread. When the method is ended, the stack frame is removed.  
Each stack frame has the reference for Local Variable Array, Operand Stack, and runtime Constant Pool of a class where the method being executed belongs.  
The size of Local Variable Array and Operand Stack is determined while compiling. Therefore, the size of stack frame is fixed according to the method.  
* *Local variable array:* It has an index starting from 0.  
0 is the reference of a class instance where the method belongs.  
From 1, the parameters sent to the method are saved. After the method parameters, the local variables of the method are saved.
* *Operand stack:* An actual workspace of a method. Each method exchanges data between the Operand stack and the Local Variable Array, and pushes or pops other method invoke results.  
The necessary size of the Operand Stack space can be determined during compiling. Therefore, the size of the Operand Stack can also be determined during compiling.

**Native method stack:** A stack for native code written in a language other than Java. In other words, it is a stack used to execute C/C++ codes invoked through JNI (Java Native Interface). According to the language, a C stack or C++ stack is created.  
**Method area (PermGen)** The method area is shared by all threads, created when the JVM starts. It stores runtime constant pool, field and method information, static variable, and method bytecode for each of the classes and interfaces read by the JVM. The method area can be implemented in various formats by JVM vendor. Oracle Hotspot JVM calls it *Permanent Area or Permanent Generation (PermGen)*.   
The garbage collection for the method area is optional for each JVM vendor.  
**Runtime constant pool:** An area that corresponds to the constant_pool table in the class file format. This area is included in the method area; however, it plays the most core role in JVM operation. Therefore, the JVM specification separately describes its importance. As well as the constant of each class and interface, it contains all references for methods and fields.  
In short, when a method or field is referred to, the JVM searches the actual address of the method or field on the memory by using the runtime constant pool.  
**Heap:** A space that stores instances or objects, and is a target of garbage collection. This space is most frequently mentioned when discussing issues such as JVM performance. JVM vendors can determine how to configure the heap or not to collect garbage.

### Going back to the disassembled bytecode we discussed previously
```
public void add(java.lang.String);
  Code:
   0:   aload_0
   1:   getfield        #15; //Field admin:Lcom/nhn/user/UserAdmin;
   4:   aload_1
   5:   invokevirtual   #23; //Method com/nhn/user/UserAdmin.addUser:(Ljava/lang/String;)Lcom/nhn/user/User;
   8:   pop
   9:   return
```

Comparing the disassembled code (OpCode) and the assembly code of the x86 architecture, the two have a similar format. However, the difference is that Java Bytecode does not write *register name*, *memory addressor*, or *offset* on the Operand.   
As described before, the JVM uses *stack*, not *registers*. And it uses *index numbers* such as 15 and 23 instead of memory addresses since it manages the memory by itself.  
The 15 and 23 are the indexes of the constant pool of the current class (here, UserService class). In short, the JVM creates a constant pool for each class, and the pool stores the reference of the actual target.

Each row of the disassembled code is interpreted as follows.
* **aload_0:** Add the #0 index of the local variable array to the Operand stack. The #0 index of the local variable array is always this, the reference for the current class instance.
* **getfield #15:** In the current class constant pool, add the #15 index to the Operand stack. UserAdmin admin field is added. Since the admin field is a class instance, a reference is added.
* **aload_1:** Add the #1 index of the local variable array to the Operand stack. From the #1 index of the local variable array, it is a method parameter. Therefore, the reference of String userName sent while invoking add() is added.
* **invokevirtual #23:** Invoke the method corresponding to the #23 index in the current class constant pool. At this time, the reference added by using getfield and the parameter added by using aload_1 are sent to the method to invoke. When the method invocation is completed, add the return value to the Operand stack.
* **pop:** Pop the return value of invoking by using invokevirtual from the Operand stack. You can see that the code compiled by the previous library has no return value. In short, the previous has no return value, so there was no need to pop the return value from the stack.
* **return:** Complete the method.

#### Example of Java Bytecode Loaded on Runtime Data Areas
The following figure will help you understand the explanation.  
![example-of-java-bytecode-loaded-on-runtime-data-areas.png](http://www.cubrid.org/files/attach/images/220547/468/290/example-of-java-bytecode-loaded-on-runtime-data-areas.png)

For reference, in this method, no local variable array has been changed. So the figure above displays the changes in Operand stack only. However, in most cases, local variable array is also changed. Data transfer between the local variable array and the Operand stack is made by using a lot of load instructions (aload, iload) and store instructions (astore, istore). 

In this figure, we have checked the brief description of the runtime constant pool and the JVM stack. When the JVM runs, each class instance will be assigned to the heap, and class information including User, UserAdmin, UserService, and String will be stored in the method area.

### Execution Engine
The bytecode that is assigned to the runtime data areas in the JVM via class loader is executed by the execution engine. The execution engine reads the Java Bytecode in the unit of instruction. It is like a CPU executing the machine command one by one. Each command of the bytecode consists of a 1-byte OpCode and additional Operand. The execution engine gets one OpCode and execute task with the Operand, and then executes the next OpCode.

But the Java Bytecode is written in a language that a human can understand, rather than in the language that the machine directly executes. Therefore, the execution engine must change the bytecode *to the language that can be executed by the machine in the JVM.* 

The bytecode can be changed to the suitable language in one of two ways:
- Interpreter
- JIT (Just-In-Time) compiler

**Interpreter:** Reads, interprets and executes the bytecode instructions one by one. As it interprets and executes instructions one by one, it can quickly interpret one bytecode, but slowly executes the interpreted result. This is the disadvantage of the interpret language. The 'language' called Bytecode basically runs like an interpreter.  
**JIT (Just-In-Time) compiler:** The JIT compiler has been introduced to compensate for the disadvantages of the interpreter. The execution engine runs as an interpreter first, and at the appropriate time, the JIT compiler compiles the entire bytecode to change it to native code. After that, the execution engine no longer interprets the method, but directly executes using native code. Execution in native code is much faster than interpreting instructions one by one. The compiled code can be executed quickly since the native code is stored in the cache.
 
However, it takes more time for JIT compiler to compile the code than for the interpreter to interpret the code one by one. Therefore, if the code is to be executed just once, it is better to interpret it instead of compiling. *Therefore, the JVMs that use the JIT compiler internally check how frequently the method is executed and compile the method only when the frequency is higher than a certain level.*  
![java-compiler-and-jit-compiler.png](http://www.cubrid.org/files/attach/images/220547/468/290/java-compiler-and-jit-compiler.png)

How the execution engine runs is not defined in the JVM specifications. Therefore, JVM vendors improve their execution engines using various techniques, and introduce various types of JIT compilers. 

Most JIT compilers run as shown in the figure below:  
![jit-compiler.png](http://www.cubrid.org/files/attach/images/220547/468/290/jit-compiler.png)

The JIT compiler converts the bytecode to an intermediate-level expression, IR (Intermediate Representation), to execute optimization, and then converts the expression to native code.

**Oracle Hotspot VM** uses a JIT compiler called **Hotspot Compiler**. It is called Hotspot because Hotspot Compiler searches the 'Hotspot' that requires compiling with the highest priority through profiling, and then it compiles the hotspot to native code.   
If the method that has the bytecode compiled is no longer frequently invoked, in other words, if the method is not the hotspot any more, the Hotspot VM removes the native code from the cache and runs in interpreter mode.  
The Hotspot VM is divided into the *Server VM* and the *Client VM*, and the two VMs use different JIT compilers.  

![hotspot-client-vm-and-server-vm.png](http://www.cubrid.org/files/attach/images/220547/468/290/hotspot-client-vm-and-server-vm.png)  
The client VM and the server VM use an identical runtime; however, they use different JIT compilers, as shown in the above figure. The client VM and the server VM use an identical runtime, however, they use different JIT compilers as shown in the above figure. Advanced Dynamic Optimizing Compiler used by the server VM uses more complex and diverse performance optimization techniques.

**IBM JVM** has introduced **AOT (Ahead-Of-Time) Compiler** from IBM JDK 6 as well as the JIT compiler. This means that many JVMs share the native code compiled through the shared cache. In short, the code that has been already compiled through the AOT compiler can be used by another JVM without compiling. In addition, IBM JVM provides a fast way of execution by pre-compiling code to JXE (Java EXecutable) file format using the AOT compiler.  

Most Java performance improvement is accomplished by improving the execution engine. As well as the JIT compiler, various optimization techniques are being introduced so the JVM performance can be continuously improved. The biggest difference between the initial JVM and the latest JVM is the execution engine.

Hotspot compiler has been introduced to Oracle Hotspot VM from version 1.3, and JIT compiler has been introduced to Dalvik VM from Android 2.2.

## The Java Virtual Machine Specification, Java SE 7 Edition
On 28th July, 2011, Oracle released Java SE 7 and updated the JVM Spec to Java SE 7 version. After releasing "The Java Virtual Machine Specification, Second Edition" in 1999, it took 12 years for Oracle to release the updated version. The updated version includes various changes and modifications accumulated over 12 years, and describes more clear specifications. In addition, it reflects the contents included in "The Java Language Specification, Java SE 7 Edition" released with Java SE 7. The major changes can be summarized as follows:
* Generics introduced from Java SE 5.0, supporting variable argument method
* Bytecode verification process technique changed since Java SE 6
* Added **invokedynamic** instruction and related class file formats for supporting dynamic type languages
* Deleted the description of the concept of the Java language itself and referred reader to the Java language specifications
* Deleted the description on Java Thread and Lock, and transferred these to the Java language specifications

The biggest change of these is the addition of *invokedynamic* instruction. *This means that a change was made in the JVM internal instruction sets, as the JVM started to support dynamic type languages of which type is not fixed, such as script languages, as well as Java language from Java SE 7.* The OpCode 186 which had not been used previously has been assigned to the new instruction, invokedynamic, and new contents have been added to the class file format to support the invokedynamic.

The version of the class file created by the Java compiler of Java SE 7 is **51.0**. The version of Java SE 6 is **50.0**.  
Much of the class file format has been changed. Therefore, class files with version 51.0 cannot be executed in the Java SE 6 JVM. 

Despite these various changes, *the 65535 byte limit of the Java method has not been removed*. Unless the JVM class file format is innovatively changed, it may not be removed in the future.

For reference, Oracle Java SE 7 VM supports **G1**, the new garbage collection; however, it is limited to the Oracle JVM, so JVM itself does not limit any garbage collection type. Therefore, the JVM specifications do not describe that.

### String in switch Statements
Java SE 7 adds various grammars and features. However, compared to the various changes in language of Java SE 7, **there are not so many changes in the JVM. So, how can the new features of the Java SE 7 be implemented?**  

We will see how String in switch Statements (a function to add a string to a switch() statement as a comparison) has been implemented in Java SE 7 by disassembling it.

For example, the following code has been written.

```
// SwitchTest
public class SwitchTest {
    public int doSwitch(String str) {
        switch (str) {
        case "abc":        return 1;
        case "123":        return 2;
        default:           return 0;
        }
    }
}
```

Since it is a new function of Java SE 7, it cannot be compiled using the Java compiler for Java SE 6 or lower versions. Compile it using the javac of Java SE 7. The following screen is the compiling result printed by using javap –c.

```
$>javap -c SwitchTest.classCompiled from "SwitchTest.java"
public class SwitchTest {
  public SwitchTest();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return  public int doSwitch(java.lang.String);
    Code:
       0: aload_1
       1: astore_2
       2: iconst_m1
       3: istore_3
       4: aload_2
       5: invokevirtual #2                  // Method java/lang/String.hashCode:()I
       8: lookupswitch  { // 2
                 48690: 50
                 96354: 36
               default: 61
          }
      36: aload_2
      37: ldc           #3                  // String abc
      39: invokevirtual #4                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      42: ifeq          61
      45: iconst_0
      46: istore_3
      47: goto          61
      50: aload_2
      51: ldc           #5                  // String 123
      53: invokevirtual #4                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      56: ifeq          61
      59: iconst_1
      60: istore_3
      61: iload_3
      62: lookupswitch  { // 2
                     0: 88
                     1: 90
               default: 92
          }
      88: iconst_1
      89: ireturn
      90: iconst_2
      91: ireturn
      92: iconst_0
      93: ireturn
```

A significantly longer bytecode than the Java source code has been created. First, you can see that lookupswitch instruction has been used for switch() statement in Java bytecode. However, two lookupswitch instructions have been used, not the one lookupswitch instruction. When disassembling the case in which int has been added to switch() statement, only one lookupswitch instruction has been used. This means that the switch() statement has been divided into two statements to process the string. See the annotation of the #5, #39, and #53 byte instructions to see how the switch() statement has processed the string.

In the #5 and #8 byte, first, hashCode() method has been executed and switch(int) has been executed by using the result of executing hashCode() method. In the braces of the lookupswitch instruction, branch is made to the different location according to the hashCode result value. String "abc" is hashCode result value 96354, and is moved to #36 byte. String "123" is hashCode result value 48690, and is moved to #50 byte.

In the #36, #37, #39, and #42 bytes, you can see that the value of the str variable received as an argument is compared using the String "abc" and the equals() method. If the results are identical, '0' is inserted to the #3 index of the local variable array, and the string is moved to the #61 byte.

In this way, in the #50, #51, #53, and #56 bytes, you can see that the value of the str variable received as an argument is compared by using the String "123" and the equals() method. If the results are identical, '1' is inserted to the #3 index of the local variable array and the string is moved to the #61 byte.

In the #61 and #62 bytes, the value of the #3 index of the local variable array, i.e., '0', '1', or any other value, is lookupswitched and branched.

In other words, in Java code, the value of the str variable received as the switch() argument is compared using the hashCode() method and the equals() method. With the result int value, switch() is executed.

In this result, the compiled bytecode is not different from the previous JVM specifications. The new feature of Java SE 7, String in switch is processed by the Java compiler, not by the JVM itself. In this way, other new features of Java SE 7 will also be processed by the Java compiler.

## Conclusion
I don't think that we need to review how Java has been developed to use Java well. So many Java developers develop great applications and libraries without understanding JVM deeply. However, if you understand JVM, you will understand Java more, and it will be helpful to solve the problems like the case we have reviewed here.

Besides the description mentioned here, the JVM has various features and technologies. The JVM specifications provide a flexible specification for JVM vendors to provide more advanced performance so that various technologies can be applied by the vendor. 

In particular, **garbage collection** is the technique used by most languages that provides usability similar to that of a VM, the latest and state-of-the-art technique in its performance.
