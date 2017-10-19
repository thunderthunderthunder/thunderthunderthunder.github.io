---
tite: 深入理解Condition条件队列
description: 
date: 2017-10-16
categories:
 - java并发
tags:
 - concurrent
 - 并发
---

## 前言
Condition是AQS的一个高级特性，jdk中有很多工具使用了这个特性，比如BlockingQueue，ReentrantReadWriteLock等等。
这篇文章将详细的解析它的源码。

## await流程
这个方法会释放当前线程占有的资源，并将自己添加到Condition的等待队列中。为什么是Condition的等待队列呢？
因为AQS也有自身的等待队列，AQS和Condition是两个不同的等待队列。
Condition等待队列中的线程需要被signal或signalAll这个两个方法唤醒。
下面看一下await方法：
```java
public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            //重要：线程被signal，或被中断，都可以走出这个循环
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                //如果线程是被中断的，则直接break
                //checkInterruptWhileWaiting这个方法如果返回0，则表示是被正常唤醒的（signal或signalAll）
                //如果返回THROW_IE，则表示为在线程被正常唤醒之前就中断了线程，需要抛出中断异常
                //如果返回REINTERRUPT，则表示为在线程被正常唤醒之后中断了线程，需要重新自我中断
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            //重新去竞争锁，acquireQueued方法会返回线程在获取锁的等待过程中是否被中断过
            //如果被中断过，则返回true，将interruptMode设置为REINTERRUPT，表示需要重新自我中断
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            //过滤掉被取消掉结点
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            //根据线程在整个过程中是否被中断，中断的时机，来回放中断
            //如果是在被正常唤醒之前中断，则抛出中断异常，否则设置中断标示位就可以了
            //这也说明了await这个方法是可以响应中断的
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```
其中addConditionWaiter这个方法是将持有当前线程的结点添加到Condition的等待队列中。
```java
private Node addConditionWaiter() {
            Node t = lastWaiter;
            if (t != null && t.waitStatus != Node.CONDITION) {
                //过滤掉被取消掉结点
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            //为当前线程new一个节点，且节点状态为CONDITION
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;//将当前结点设置为等待队列的对头
            else
                t.nextWaiter = node;
            lastWaiter = node;//将等待队列的对尾指向新加入的结点（就是持有当前线程的结点）
            return node;//返回持有当前线程的结点
        }
```

```java
//这个方法是完全释放当前线程占有的资源
final int fullyRelease(Node node) {
        boolean failed = true; //是否释放成功标示
        try {
            int savedState = getState(); //获取当前的state值
            if (release(savedState)) { //释放state，该方法在ASQ中已经解析过
                failed = false; //成功释放
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            //如果释放失败，则将结点状态改为取消
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```
如上所说，Condition.await()释放资源，唤醒的是AQS的等待队列，而不是Condition自身的等待队列，Condition自身的等待队列只能靠signal，signalAll来唤醒。
下面这个方法返回true则说明该node在AQS的队列中，可以获取资源，返回false则说明node在Condition队列中，等待被signal。
```java
final boolean isOnSyncQueue(Node node) {
        //如果结点的waitStatus为CONDITION，或者结点的前置结点为null，则表明还在Condition自身的等待队列中，需要被signal
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        //在AQS的等待队列中，有资格去获取锁
        if (node.next != null) // If has successor, it must be on queue
            return true;
        return findNodeFromTail(node);
    }
```
## signal流程
直接看主要的三个方法，doSignal是唤醒一个结点，doSignalAll是唤醒所有在Condition队列中的结点。
这个两个方法都调用了transferForSignal方法来真正唤醒结点。
```java
private void doSignal(Node first) {
            do {
                //等待队列的头节点出队
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }

private void doSignalAll(Node first) {
        lastWaiter = firstWaiter = null;
        //循环将唤醒所有等待队列中的结点
        do {
            Node next = first.nextWaiter;
            first.nextWaiter = null;
            transferForSignal(first);
            first = next;
        } while (first != null);
    }
```
```java
final boolean transferForSignal(Node node) {
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
        //将该结点放到AQS的队列的对尾，并返回它的前置结点
        //从中可以发现，AQS等待队列中等待的结点prev肯定不为空，head为当前持有锁的线程所在的结点
        Node p = enq(node);
        int ws = p.waitStatus; //获取前置结点的状态
        //如果前置结点状态为取消，或者将前置结点的状态设置为通知后继结点（SIGNAL）失败（失败是可能在这一瞬间前置结点被取消了），则直接唤醒当前结点
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```
