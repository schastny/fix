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
