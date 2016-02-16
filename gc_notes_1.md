# Understanding Java Garbage Collection
[Original](http://www.cubrid.org/blog/dev-platform/understanding-java-garbage-collection/)

Understanding how GC works can help you write much better Java applications.  
That means you have experience in developing applications of certain size. That means you completely understand the features of the application you have developed. 

## Stop-the-world 
Stop-the-world will occur no matter which GC algorithm you choose. Stop-the-world means that the JVM is stopping the application from running to execute a GC. When stop-the-world occurs, every thread except for the threads needed for the GC will stop their tasks. The interrupted tasks will resume only after the GC task has completed.  
GC tuning often means reducing this stop-the-world time.

## Generational Garbage Collection 

Java does not explicitly specify a memory and remove it in the program code. Some people sets the relevant object to null or use System.gc() method to remove the memory explicitly. 

*Setting it to null is not a big deal, but calling System.gc() method will affect the system performance drastically, and must not be carried out.** 

In Java, as the developer does not explicitly remove the memory in the program code, the garbage collector finds the unnecessary (garbage) objects and removes them. This garbage collector was created based on the following two preconditions.
* Most objects soon become unreachable.
* References from old objects to young objects only exist in small numbers.
