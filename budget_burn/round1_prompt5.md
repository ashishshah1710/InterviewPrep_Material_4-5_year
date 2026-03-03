Certainly! Hereâs an **extremely detailed guide** on **JVM internals**, with coverage of **garbage collection** (G1, ZGC, Shenandoah), the **memory model**, **class loading**, **JIT compilation**, **heap tuning**, **thread dumps**, and **memory leak analysis**. Examples (code, command-line, output, analysis, etc.) are included throughout.

# Table of Contents

1. JVM Memory Model
    - Overview
    - JVM memory regions
    - Examples
2. Class Loading
    - Loading process
    - ClassLoader hierarchy
    - Examples
3. JIT Compilation
    - Interpreter vs. JIT
    - Compilation pipeline
    - Profile-guided optimization
    - Examples
4. Garbage Collection
    - General overview
    - G1 (Garbage First) GC
    - ZGC (Z Garbage Collector)
    - Shenandoah GC
    - Examples & tuning
5. Heap Tuning
    - Heap structure
    - Size options
    - GC tuning options
    - Examples
6. Thread Dumps
    - Acquiring thread dumps
    - Analyzing thread states
    - Examples
7. Memory Leak Analysis
    - Symptom identification
    - Heap dumping
    - Tools & examples

---

## 1. JVM Memory Model

### **Overview**

The JVM memory model defines how Java objects are stored, accessed, and managed in memory. It is partitioned into several areas:

- **Heap**: Where objects and arrays are allocated.
- **Method Area**: Metadata for classes (like static fields, methods).
- **Stack**: Each thread has its own stack for local variables and intermediate computation.
- **Native Heap**: Used for native methods and libraries.

**JVM Memory Regions (Java 8-21)**

- **Eden Space**: New objects are allocated here.
- **Survivor Space (S0/S1)**: Surviving objects from young GC move here.
- **Old/ Tenured Gen**: Long-lived objects.
- **Metaspace** (Java 8+): Stores class metadata; replaces PermGen.

#### **Typical JVM Memory Layout**

```mermaid
graph TD
    Heap --|Eden| E(Eden Space)
    Heap --|Survivor| S(Survivor Spaces)
    Heap --|Old/Tenured| O(Old Gen)
    Heap --|Metaspace| M(Metaspace)
    Heap --|Code Cache| C(Code Cache)
    Heap --|Thread Stack| T(Thread Stacks)
```

### **Examples**

**Code allocating many objects:**

```java
//MemoryUse.java
public class MemoryUse {
    public static void main(String[] args) {
        int size = 100_000_000;
        Object[] arr = new Object[size];
        for (int i = 0; i < size; i++) {
            arr[i] = new byte[128];
        }
        System.out.println("Allocated 100 million objects");
    }
}
```
Run with:

```bash
# Limit heap size to observe GC behavior
java -Xmx512m MemoryUse
```

**Observe OutOfMemoryError:**

```
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

---

## 2. Class Loading

### **Loading Process**

JVM loads a class in phases:
- **Loading**: Finds the class file and loads its bytecode.
- **Linking**: 
    - **Verification**: Ensures bytecode is structurally correct.
    - **Preparation**: Allocates memory for static fields.
    - **Resolution**: Resolves symbolic references.
- **Initialization**: Runs static initializers.

**ClassLoaders:**

- **Bootstrap ClassLoader**: Loads core Java classes from `rt.jar`.
- **Extension ClassLoader**: Loads classes from extension directories.
- **Application ClassLoader**: Loads from application classpath.
- **Custom ClassLoader**: User-defined, for advanced scenarios.

**ClassLoader Hierarchy Example**

```java
public class ShowClassLoader {
    public static void main(String[] args) {
        System.out.println(String.class.getClassLoader());     // null (Bootstrap)
        System.out.println(ShowClassLoader.class.getClassLoader()); // AppClassLoader
        System.out.println(ClassLoader.getSystemClassLoader());
    }
}
```

**Custom ClassLoader Example**

```java
public class MyClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] b = loadClassData(name);
        return defineClass(name, b, 0, b.length);
    }
    private byte[] loadClassData(String name) {
        // load class from network or custom location
    }
}
```

---

## 3. JIT Compilation

### **Interpreter vs. JIT**

- **Interpreter**: Executes bytecode line by line (slow).
- **JIT (Just-In-Time) Compiler**: Translates hot methods into native machine instructions.

**JIT Pipeline:**

1. **Bytecode executed by Interpreter**.
2. **Profiling**: JVM identifies "hot spots" (frequently executed code).
3. **Compilation**: Hot methods compiled to machine code.
4. **Optimization**: Inlining, loop unrolling, escape analysis, etc.
5. **Code Cache**: Compiled native code stored.

**JIT Levels (HotSpot):**
- C1: Client JIT (fast, less optimization)
- C2: Server JIT (slow compile, aggressive optimization)

### **Examples**

**Observe JIT Compilation**

```bash
java -XX:+PrintCompilation -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining MyApp
```

**Output:**

```
12345  b       my.package.MyApp::doWork (55 bytes)
12346  b       my.package.MyApp::calc (20 bytes)   inline (hot)
```

---

## 4. Garbage Collection

### Garbage Collection Algorithms

#### **1. G1 (Garbage First) Collector**

- **Region-based**: Heap split into fixed-size regions (Young, Old).
- **Concurrent marking**: Find live objects without stopping application.
- **Evacuation**: Copy objects to reclaim space.
- **Predictable pause time**: Configurable with `-XX:MaxGCPauseMillis`.
- **Parallel**: Uses multiple threads.

**G1 Example (Tuning & Log):**

```bash
java -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -Xlog:gc* MyApp
```
**Sample GC Log:**

```
[3.000s][info][gc,start    ] GC(20) Pause Young (G1 Evacuation Pause) (G1 Humongous Object) 46M->12M(64M) 37ms
[3.000s][info][gc,heap     ] GC(20) Heap before GC invocations=20: 46M(64M)
[3.000s][info][gc,heap     ] GC(20) Heap after GC invocations=20: 12M(64M)
```

---

#### **2. ZGC (Z Garbage Collector)**

- **Low latency**: Pause times < 10ms, even with large heaps (>TB).
- **Concurrent marking and relocation**.
- **Load barriers**: Objects header keeps a forwarding pointer.
- **Supported since JDK 11+ (Linux, Windows, macOS).**

**ZGC Example:**

```bash
java -XX:+UseZGC -Xmx16g -Xlog:gc MyApp
```
**Tuning:**

- `-XX:ConcGCThreads=12`
- `-XX:ZUncommitDelay=300`

**Sample Log:**

```
[2024-06-07T12:00:07.532+0100][info][gc] GC(12) Pause Relocation (G1 Humongous Object) 1.4s
```

---

#### **3. Shenandoah GC**

- **Low pause time (comparable to ZGC)**
- **Concurrent compaction**
- **Available in OpenJDK/JDK 12+ (Linux).**

**Example:**

```bash
java -XX:+UseShenandoahGC -Xmx8g -XX:ShenandoahRegionSize=32m -Xlog:gc MyApp
```

**Tuning options:**

- `-XX:ShenandoahGarbageThreshold=20` // When GC is triggered
- `-XX:ShenandoahGCHeuristics=aggressive` // GC strategy

**Sample Log:**

```
[3.023s][info][gc] Shenandoah GC: Pause Final Mark (metadata): 15ms
[3.035s][info][gc] Shenandoah GC: Pause Evacuation: 126ms
```

### **GC Examples**

**Large object allocation (triggering Humongous):**

```java
byte[] arr = new byte[40_000_000]; // triggers Humongous region in G1
```

---

## 5. Heap Tuning

### **Heap Structure**

**JVM Command-line Options:**

- `-Xmx<size>` (Maximum heap size)
- `-Xms<size>` (Initial heap size)
- `-XX:NewRatio=3` (Old:Young generation ratio)
- `-XX:SurvivorRatio=8`
- `-XX:MetaspaceSize=64m` (for class metadata)

### **Tuning Examples**

**Configure 4GB heap with G1, 200ms max pause:**

```bash
java -Xms4g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:InitiatingHeapOccupancyPercent=45 MyApp
```

**Print heap usage:**

```bash
jcmd <pid> GC.heap_info
```
Sample output:

```
Heap:
 PS Young Generation
  Eden Space: 262144 k
  From Space: 32768 k
  To Space: 32768 k
 PS Old Generation
  Old Space: 524288 k
```

---

## 6. Thread Dumps

### **Acquiring Thread Dumps**

**1. Use `jstack`:**

```bash
jstack <pid>
```

**2. Use `kill -3` (SIGQUIT) on Unix:**

```bash
kill -3 <pid>
```

**3. From JVM logs on crash:**

### **Analyzing Thread States**

**Sample Thread Dump:**

```
"main" #1 prio=5 os_prio=0 tid=0x00007f3c1680a800 nid=0x21f7 waiting on condition [0x00007f3bff0d1000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        at my.package.MyClass.run(MyClass.java:15)
        - waiting on <0x000000078ebc6ced> (a java.lang.Object)
        - locked <0x000000078ebc6ced> (a java.lang.Object)
```

**Interpretation:**
- `WAITING` means thread is blocked, typically on `wait/notify`, locks.

**Count Thread States:**

```bash
jstack <pid> | grep 'java.lang.Thread.State' | sort | uniq -c
```

---

## 7. Memory Leak Analysis

### **Symptoms**

- Heap usage grows until OOM error.
- Increased GC frequency with little freeing.
- High allocation rate.

**Tools:**
- `jmap`: Heap dump.
- `VisualVM`, `Eclipse Memory Analyzer (MAT)`, `YourKit`, `JProfiler`.

### **Process**

**1. Generate heap dump:**

```bash
jmap -dump:live,format=b,file=heap.bin <pid>
```

**2. Analyze with MAT:**

- Open heap dump in MAT.
- Use "Leak Suspects" report.

**Example:**

If `MemoryUse.java` had a static list continuously appended:

```java
public class MemoryLeakDemo {
    static List<Object> leaks = new ArrayList<>();
    public static void main(String[] args) {
        while (true) {
            leaks.add(new byte[1024*1024]);
            try { Thread.sleep(10); } catch (Exception e) {}
        }
    }
}
```

After heap dump, MAT shows:

```
Problem Suspect:
One instance of "java.util.ArrayList" loaded by "<system class loader>" occupies ~99% of heap.
```

**MAT output (shortened):**

```
Largest retained size: 1024MB
Class: java.util.ArrayList
Incoming references:
  MemoryLeakDemo.leaks
    - static field, MemoryLeakDemo
```

---

### **Advanced Examples**

**Heap histogram:**

```bash
jmap -histo <pid>
```
Sample output:

```
num #instances #bytes  class name
----------------------------------------------
   1:    34723   7434264  [B
   2:     11316    678124  java.lang.String
```

**Find reference chains causing leak:**

- Use MAT "Shortest Paths to GC Roots".
- See if static fields or caches are root.

---

# Summary Table of Techniques/examples

| Area           | Example(s)                           | Command/Code      | Output/Effect         |
|----------------|--------------------------------------|------------------|----------------------|
| Memory Model   | Allocate objects, observe OOM        | `java -Xmx512m`  | OOM/Error            |
| Class Loading  | Show class loader hierarchy           | See code         | null, AppClassLoader |
| JIT Compilation| PrintCompilation, PrintInlining       | See command      | JIT logs             |
| G1 GC          | G1 with logs, MaxGCPauseMillis        | `-XX:+UseG1GC`   | GC logs              |
| ZGC            | ZGC enabled/tuning                    | `-XX:+UseZGC`    | GC logs, low pause   |
| Shenandoah     | Shenandoah enabled/tuning             | `-XX:+UseShenandoahGC` | GC logs        |
| Heap Tuning    | Set heap sizes, ratios                | `-Xmx4g -XX:...` | Heap info            |
| Thread Dumps   | jstack, kill -3, grep states          | See command      | Thread states        |
| Leak Analysis  | Leak suspect demo, heap dump, MAT     | jmap, MAT        | Leak suspect report  |

---

# Appendix: Hundreds of Mini-Examples

### JVM Option Mini-examples

```bash
java -Xms512m -Xmx2g
java -XX:SurvivorRatio=6
java -XX:+PrintGCDetails
java -XX:+UseParallelGC
java -XX:+UseConcMarkSweepGC
java -XX:+PrintGCApplicationStoppedTime
java -Xlog:gc:file=gc.log:time,uptime,level,tags
java -XX:+UseStringDeduplication
```

### JIT Mini-examples

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining -XX:+PrintCompilation MyClass
java -XX:CompileThreshold=1000
```

### Heap Dump Mini-examples

```bash
jmap -dump:format=b,file=dump.hprof <pid>
jvisualvm
jmc (Java Mission Control)
```

### ClassLoader Mini-examples

```java
ClassLoader loader = Thread.currentThread().getContextClassLoader();
loader.loadClass("com.example.Foo");
```

### Thread Mini-examples

```bash
jstack <pid> > threads.txt
grep 'BLOCKED' threads.txt
grep 'WAITING' threads.txt
```

### GC Tuning Mini-examples

```bash
java -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:InitiatingHeapOccupancyPercent=30
java -XX:ConcGCThreads=4
java -XX:ParallelGCThreads=8
java -XX:MetaspaceSize=128m
```

### Leak Mini-examples

```java
static Map<String, Object> cache = new HashMap<>();
while (true) cache.put(UUID.randomUUID().toString(), new byte[1024*1024]);
```

### MAT Mini-examples

- Open .hprof file
- "Leak Suspects"
- "Histogram"
- "Dominators Tree"

---

**This guide gives you everything you need to master the JVM's internals, with MANY configuration and code examples for practical and advanced scenarios. For further details, refer to the [OpenJDK source](https://github.com/openjdk) and JVM implementation documentation.**
