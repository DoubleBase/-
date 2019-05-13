## 基本简介

`BlockingQueue`是一个阻塞队列，当队列元素不为空时，可以检索元素，当队列不满的时候，可以添加元素。

`BlockingQueue`对队列元素的一些操作方法有4种处理方式：抛出异常、返回特殊值，阻塞，超时机制。如下表所示

|          | 抛出异常  | 返回特殊值 | 阻塞   | 超时               |
| :------- | --------- | ---------- | ------ | ------------------ |
| 添加元素 | add(e)    | offer(e)   | put(e) | offer(e,time,unit) |
| 移除元素 | remove()  | poll()     | take() | poll(e,time,unit)  |
| 查看元素 | element() | peek()     | 无     | 无                 |

- 添加元素

  - add：如果队列满，抛出`IllegalStateException`异常
  - put：如果队列满，则阻塞
  - offer：添加成功返回`true`，如果队列满，返回`false`

- 移除元素

  - remove：移除并返回头元素，如果队列空，抛出`NoSuchElementException`异常
  - take：移除并返回头元素，如果队列空，则阻塞
  - poll：移除并返回头元素，如果队列空，则返回`null`

- 查询元素

  - element：返回队列头元素，如果队列空，抛出`NoSuchElementException`异常
  - peek：返回队列头元素，如果队列空，则返回`null`

## 类图

![1557421040869](E:\知识笔记\多线程\阻塞队列\BlockingQueue.png)

继承Queue，元素具有先进先出（FIFO）特性，同时具有Collection的接口实现方法

## 特性

- **元素不能为null值**，当添加一个null元素时，会抛出一个空指针异常
- 有容量大小限制，在任何时候可能都有剩余容量，当队列满的时候，其他元素就无法再不阻塞的情况下进行添加，如果没有容量限制，阻塞队列总是返回Integer.MAX_VALUE值作为剩余容量
- 经常作为生产者-消费者队列，另外支持Collection接口，例如：使用remove(e)移除指定元素，但是这些操作效率不高，只是偶尔使用，例如当队列中的消息不需要处理时。
- BlockingQueue中的方法是**线程安全**的，所有方法在执行的时候，通过内部锁或其他并发控制实现原子性操作，批量的Collection操作 addAll，containsAll，retailAll，removeAll不一定是原子性操作，除非特殊处理
- 不支持任何close或shutdown操作来表示无法添加元素
- **Happens-Before**：将一个元素放入BlockingQueue的操作先行发生于另一个线程从BlockingQueue中获取这个元素的操作

- 

## 生产者-消费者

```java
class Producer implements Runnable {
   private final BlockingQueue queue;
   Producer(BlockingQueue q) { queue = q; }
   public void run() {
     try {
       while (true) { 
           queue.put(produce()); 
       }
     } catch (InterruptedException ex) { ... handle ...}
   }
   Object produce() { ... }
 }
```

```java
class Consumer implements Runnable {
   private final BlockingQueue queue;
   Consumer(BlockingQueue q) { queue = q; }
   public void run() {
     try {
       while (true) { consume(queue.take()); }
     } catch (InterruptedException ex) { ... handle ...}
   }
   void consume(Object x) { ... }
 }
```

```java
class Setup {
   void main() {
     BlockingQueue q = new SomeQueueImplementation();
     Producer p = new Producer(q);
     Consumer c1 = new Consumer(q);
     Consumer c2 = new Consumer(q);
     new Thread(p).start();
     new Thread(c1).start();
     new Thread(c2).start();
   }
 }
```

