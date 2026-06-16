# Java Virtual Threads - Staff Engineer Guide

## Overview

Virtual Threads, introduced by Project Loom, are lightweight threads managed by the JVM.

Traditional Java concurrency follows:

```text
1 Java Thread = 1 OS Thread
```

Virtual Threads change this to:

```text
Many Virtual Threads
          │
          ▼
     JVM Scheduler
          │
          ▼
     Few OS Threads
```

The goal is to enable highly concurrent applications using familiar blocking code.

---

# Why Traditional Threads Do Not Scale

Consider a service that performs:

* Product lookup
* Inventory lookup
* Recommendation lookup

Traditional execution:

```text
Request
   │
   ├── OS Thread #101
   │      │
   │      └── Waiting on Product Service
   │
   ├── OS Thread #102
   │      │
   │      └── Waiting on Inventory Service
   │
   └── OS Thread #103
          │
          └── Waiting on Recommendation Service
```

During network waits:

* CPU is idle
* OS thread remains allocated
* Scheduler must continue tracking the thread
* Memory remains reserved

As request volume grows, the application requires more OS threads.

---

# Virtual Thread Architecture

Virtual Threads are scheduled by the JVM.

```text
Virtual Thread #1
Virtual Thread #2
Virtual Thread #3
Virtual Thread #4
Virtual Thread #5
         │
         ▼
    JVM Scheduler
         │
         ▼
  Carrier Threads
   (OS Threads)
```

Carrier Threads are normal OS threads.

Virtual Threads execute on Carrier Threads whenever they are runnable.

---

# Lifecycle of a Virtual Thread

## Running

```text
Virtual Thread
      │
      ▼
Carrier Thread
      │
      ▼
CPU
```

---

## Blocking Operation

```text
Virtual Thread
      │
      ▼
HTTP Call / Database Call
      │
      ▼
Waiting
```

The JVM performs:

```text
Park Virtual Thread
      │
      ▼
Release Carrier Thread
      │
      ▼
Carrier Executes Other Work
```

---

## Resumption

```text
Response Arrives
      │
      ▼
Virtual Thread Ready
      │
      ▼
Mount on Carrier Thread
      │
      ▼
Continue Execution
```

---

# End-to-End Flow

```text
submit()
   │
   ▼
Create Virtual Thread
   │
   ▼
Assign Carrier Thread
   │
   ▼
Execute Task
   │
   ▼
Network Wait
   │
   ▼
Virtual Thread Parked
   │
   ▼
Carrier Thread Released
   │
   ▼
Carrier Executes Other Work
   │
   ▼
Network Response
   │
   ▼
Virtual Thread Resumed
   │
   ▼
Task Completed
```

---

# Example

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {

    Future<String> product =
            executor.submit(() -> fetchProduct());

    Future<String> inventory =
            executor.submit(() -> fetchInventory());

    Future<String> recommendations =
            executor.submit(() -> fetchRecommendations());

    System.out.println(product.get());
    System.out.println(inventory.get());
    System.out.println(recommendations.get());
}
```

---

# Mental Model

Traditional Threads:

```text
Task
 │
 ▼
Java Thread
 │
 ▼
OS Thread
 │
 ▼
CPU
```

Virtual Threads:

```text
Task
 │
 ▼
Virtual Thread
 │
 ▼
JVM Scheduler
 │
 ▼
Carrier Thread
 │
 ▼
CPU
```

---

# Key Benefits

## Massive Concurrency

Applications can create millions of Virtual Threads.

```text
1,000,000 Virtual Threads
          │
          ▼
 Hundreds of Carrier Threads
```

---

## Simpler Code

Use blocking code:

```java
String product = client.fetchProduct();
```

instead of complex reactive pipelines.

---

## Better Resource Utilization

Waiting operations no longer monopolize OS threads.

```text
Waiting Work
      │
      ▼
Virtual Thread Parked
      │
      ▼
Carrier Thread Reused
```

---

# Key Takeaway

Virtual Threads make blocking I/O scalable by separating application concurrency from operating-system threads.

The JVM parks waiting Virtual Threads, releases Carrier Threads, and resumes execution when results become available.

This enables:

* Simpler code
* Massive concurrency
* Lower thread-management overhead
* Better CPU utilization
