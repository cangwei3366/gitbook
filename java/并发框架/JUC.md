# JUC

````java

源码分析

内部类 Sync 

​```java
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        //获取锁
        abstract void lock();

        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
    	//非公平方式获取锁
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            //获取信号量状态
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    //设置当前线程独占
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {//可重入特性
                int nextc = c + acquires; // 增加重入次数
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
		//试图在共享模式下获取对象状态
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())//// 当前线程不为独占线程
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                // 已经释放，清空独占
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
		// 判断资源是否被当前线程占有
        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }
		// 新生一个条件
        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        // 返回独占线程 如果没有则返回null
        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }
		// 返回状态
        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }
		// 资源是否被占用
        final boolean isLocked() {
            return getState() != 0;
        }

        /**
         * Reconstitutes the instance from a stream (that is, deserializes it).
         */
    	//自定义反序列化逻辑
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }
​```

内部类NonfairSync 

​```java
//非公平锁 
static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                // 把当前线程设置独占了锁
                setExclusiveOwnerThread(Thread.currentThread());
            else// 以独占模式获取对象，忽略中断
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
​```

内部类FairSync 

​```java
//公平锁 
static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
    	//尝试公平加锁
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {//若状态为0
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {//尝试加锁
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {//同一线程可重入
                int nextc = c + acquires;
                if (nextc < 0) // 超过了int的表示范围
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
	public final boolean hasQueuedPredecessors() {
       //保证了不论是新的线程还是已经排队的线程都顺序使用锁
       //判断当前线程是否构建了节点，以及该节点是否位于head节点后的第一个节点，如果是才会去尝试cas获取锁
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
​```

````
