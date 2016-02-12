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
