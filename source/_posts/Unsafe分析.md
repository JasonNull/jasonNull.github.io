---
title: Unsafe分析
date: 2018-06-23 18:26:15
tags:
- java
- 并发
- 源码
categories:
- java
- 并发
toc: true
---
Unsafe类是一个拥有大量native方法的一个类，里面都是比较底层的接口
# get与put
主要是操作一个无法访问的属性值。
```java
// 获取类内的属性值
public native int getInt(Object var1, long var2);
// 将类偏移量为var2的地方值设置为var4
public native void putInt(Object var1, long var2, int var4);
```
<!-- more -->
# memory
主要是对直接内存的操作，被`DirectByteBuffer`调用
```java
// 分配内存
public native long allocateMemory(long var1);
// 在原地址var1重新分配大小为var3的内存
public native long reallocateMemory(long var1, long var3);
// 在var1对象的var2地址的长度为var4的一块内存的值初始化为var6
public native void setMemory(Object var1, long var2, long var4, byte var6);
public void setMemory(long var1, long var3, byte var5) {
    this.setMemory((Object)null, var1, var3, var5);
}
// 复制内存
public native void copyMemory(Object var1, long var2, Object var4, long var5, long var7);
public void copyMemory(long var1, long var3, long var5) {
    this.copyMemory((Object)null, var1, (Object)null, var3, var5);
}
// 释放内存
public native void freeMemory(long var1);
```

# offset
获取了offset主要是为了cas操作
```java
// 用于获取各个不同类型属性的偏移量
public native long staticFieldOffset(Field var1);
public native long objectFieldOffset(Field var1);
public native Object staticFieldBase(Field var1);
public native int arrayBaseOffset(Class<?> var1);
public native int arrayIndexScale(Class<?> var1);
```

# CAS
主要是为了无锁数据结构使用，以及atomic类使用，实现调用了处理器的**lock，cmpxchg指令**。
```java
// 第二个参数是偏移量
public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);
```
```c
// 用于添加lock指令
// 说一下lock指令，1、确保对内存的读-改-写操作原子执行 2、禁止该指令与前后的读写指令重排序 3、把写缓冲区中的所有数据刷新到内存中
// 在Pentium处理器之前，lock操作之间锁住内存总线的
// 但是在Pentium4处理器之后，intel做了一个优化
// 如果lock后面的指令需要访问的内存已经完整地在一个缓存行之中，则将转为锁住缓存行
// 但是如果不在缓存行之中，则会锁住整个内存总线。
#define LOCK_IF_MP(mp) __asm cmp mp, 0  \
                       __asm je L0      \
                       __asm _emit 0xF0 \
                       __asm L0:
inline jint     Atomic::cmpxchg(jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  // 判断当前处理器是不是多核处理器
  int mp = os::is_MP();
  __asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    // 将eax与原值作比较，如果相同则将ecx的值，写入edx指向的地址
    // 多核处理器上，指令变成如此，lock cmpxchg dword ptr [edx], ecx 
    LOCK_IF_MP(mp)
    cmpxchg dword ptr [edx], ecx 
  }
}
```

# volatile, ordered
**putOrderedInt的原理是使用了storeStore屏障（sfence），性能比storeLoad快20%，但是不保证立马可见**
```java
public native Object getObjectVolatile(Object var1, long var2);
public native void putObjectVolatile(Object var1, long var2, Object var4);
public native int getIntVolatile(Object var1, long var2);
public native void putIntVolatile(Object var1, long var2, int var4);
public native boolean getBooleanVolatile(Object var1, long var2);
public native void putBooleanVolatile(Object var1, long var2, boolean var4);
public native byte getByteVolatile(Object var1, long var2);
public native void putByteVolatile(Object var1, long var2, byte var4);
public native short getShortVolatile(Object var1, long var2);
public native void putShortVolatile(Object var1, long var2, short var4);
public native char getCharVolatile(Object var1, long var2);
public native void putCharVolatile(Object var1, long var2, char var4);
public native long getLongVolatile(Object var1, long var2);
public native void putLongVolatile(Object var1, long var2, long var4);
public native float getFloatVolatile(Object var1, long var2);
public native void putFloatVolatile(Object var1, long var2, float var4);
public native double getDoubleVolatile(Object var1, long var2);
public native void putDoubleVolatile(Object var1, long var2, double var4);
public native void putOrderedObject(Object var1, long var2, Object var4);
public native void putOrderedInt(Object var1, long var2, int var4);
public native void putOrderedLong(Object var1, long var2, long var4);
```
# park
其实主要还是每个线程拥有一个Parker实例
调用Unsafe.park的时候，会调用当前线程Parker实例的对应方法
```java
public native void unpark(Object var1);
public native void park(boolean var1, long var2);

class Parker : public os::PlatformParker {
private:
  volatile int _counter ;
  Parker * FreeNext ;
  JavaThread * AssociatedWith ; // Current association

public:
  Parker() : PlatformParker() {
    _counter       = 0 ;
    FreeNext       = NULL ;
    AssociatedWith = NULL ;
  }
protected:
  ~Parker() { ShouldNotReachHere(); }
public:
  // For simplicity of interface with Java, all forms of park (indefinite,
  // relative, and absolute) are multiplexed into one call.
  void park(bool isAbsolute, jlong time);
  void unpark();

  // Lifecycle operators
  static Parker * Allocate (JavaThread * t) ;
  static void Release (Parker * e) ;
private:
  static Parker * volatile FreeList ;
  static volatile int ListLock ;

};

void Parker::park(bool isAbsolute, jlong time) {
  // 代表被unpark过
  if (Atomic::xchg(0, &_counter) > 0) return;

  Thread* thread = Thread::current();
  assert(thread->is_Java_thread(), "Must be JavaThread");
  JavaThread *jt = (JavaThread *)thread;

  if (Thread::is_interrupted(thread, false)) {
    return;
  }

  timespec absTime;
  if (time < 0 || (isAbsolute && time == 0) ) { // don't wait at all
    return;
  }
  if (time > 0) {
    unpackTime(&absTime, isAbsolute, time);
  }

  ThreadBlockInVM tbivm(jt);
  if (Thread::is_interrupted(thread, false) || pthread_mutex_trylock(_mutex) != 0) {
    return;
  }

  int status ;
  if (_counter > 0)  { // no wait needed
    _counter = 0;
    status = pthread_mutex_unlock(_mutex);
    assert (status == 0, "invariant") ;

    // 插入一个完全的屏障
    OrderAccess::fence();
    return;
  }

  OSThreadWaitState osts(thread->osthread(), false /* not Object.wait() */);
  jt->set_suspend_equivalent();
  assert(_cur_index == -1, "invariant");

  if (time == 0) {
    _cur_index = REL_INDEX; // arbitrary choice when not timed
    // 等待被释放，不知道为什么要使用这个不同的信号量
    status = pthread_cond_wait (&_cond[_cur_index], _mutex) ;
  } else {
    _cur_index = isAbsolute ? ABS_INDEX : REL_INDEX;
    // 等待被释放
    status = os::Linux::safe_cond_timedwait (&_cond[_cur_index], _mutex, &absTime) ;
    if (status != 0 && WorkAroundNPTLTimedWaitHang) {
      pthread_cond_destroy (&_cond[_cur_index]) ;
      pthread_cond_init    (&_cond[_cur_index], isAbsolute ? NULL : os::Linux::condAttr());
    }
  }
  _cur_index = -1;
  assert_status(status == 0 || status == EINTR ||
                status == ETIME || status == ETIMEDOUT,
                status, "cond_timedwait");

#ifdef ASSERT
  pthread_sigmask(SIG_SETMASK, &oldsigs, NULL);
#endif

  _counter = 0 ;
  status = pthread_mutex_unlock(_mutex) ;
  assert_status(status == 0, status, "invariant") ;
  OrderAccess::fence();

  if (jt->handle_special_suspend_equivalent_condition()) {
    jt->java_suspend_self();
  }
}

void Parker::unpark() {
  int s, status ;
  status = pthread_mutex_lock(_mutex);
  assert (status == 0, "invariant") ;
  s = _counter;
  _counter = 1;
  if (s < 1) {
    // thread might be parked
    if (_cur_index != -1) {
      // thread is definitely parked
      if (WorkAroundNPTLTimedWaitHang) {
        status = pthread_cond_signal (&_cond[_cur_index]);
        assert (status == 0, "invariant");
        status = pthread_mutex_unlock(_mutex);
        assert (status == 0, "invariant");
      } else {
        status = pthread_mutex_unlock(_mutex);
        assert (status == 0, "invariant");
        status = pthread_cond_signal (&_cond[_cur_index]);
        assert (status == 0, "invariant");
      }
    } else {
      pthread_mutex_unlock(_mutex);
      assert (status == 0, "invariant") ;
    }
  } else {
    pthread_mutex_unlock(_mutex);
    assert (status == 0, "invariant") ;
  }
}
```