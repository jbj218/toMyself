# Java基础

## Static

~~~java
public class SimpleLock {
    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int acquires) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int releases) {
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }
    }

    private final Sync sync = new Sync();

    public void lock() {
        sync.acquire(1);
    }

    public void unlock() {
        sync.release(1);
    }
}

~~~

> Static不是代表只有一份吗？那静态内部类怎么说呢？首先单纯的一个类是不可以被static修饰的，但静态内部类可以，而且它本身还是个类，那我new这个静态内部类的时候，可以有多个对象吗？那不是和static冲突了吗？

### 静态内部类

**静态内部类的 `static`，和静态变量的 `static`，意思不一样。**

#### 先拆清楚：两个 `static` 各管什么

| 写法                    | static 修饰谁  | static 的含义                                  |
| :---------------------- | :------------- | :--------------------------------------------- |
| `static int count;`     | 修饰**字段**   | 这个字段属于**类**，全局共享一份               |
| `static class Inner {}` | 修饰**内部类** | 这个内部类**不持有外部类引用**，是个"独立的类" |

**关键：`static` 修饰类的时候，不是"这个类只有一份实例"，而是"这个类不依赖外部类实例"。**

#### 静态内部类的 static 到底啥意思

对比两种内部类：

```java
public class Outer {
    // 非静态内部类
    class Inner1 {
        // 隐式持有 Outer.this
    }

    // 静态内部类
    static class Inner2 {
        // 不持有 Outer.this，独立
    }
}
```

- **非静态内部类**：每个 `Inner1` 实例都偷偷拿着一个 `Outer` 实例的引用，所以 `new Inner1()` 必须先有 `Outer` 实例
- **静态内部类**：`Inner2` **不持有** `Outer` 实例引用，就是个普通类，只是"写在 Outer 里面"而已

所以静态内部类的 `static` 表达的是：**"我不需要外部类实例就能存在"**，跟"只能有一个实例"没有任何关系。

#### 那能 new 多个对象吗

**当然可以，而且和 static 完全不冲突。**

```java
Outer.Inner2 a = new Outer.Inner2();
Outer.Inner2 b = new Outer.Inner2();
Outer.Inner2 c = new Outer.Inner2();
// a、b、c 是三个独立对象，完全合法
```

`static` 在这里**没有限制实例数量**。它只是说"这个类不依赖外部类实例"，至于你 new 几个，随便。

#### 那"静态"到底体现在哪

体现在**创建方式**和**依赖关系**上：

java

```
// 非静态内部类：必须先有外部实例
Outer outer = new Outer();
Outer.Inner1 i = outer.new Inner1();   // 语法特殊，依附外部实例

// 静态内部类：直接 new，不需要外部实例
Outer.Inner2 i = new Outer.Inner2();   // 跟普通类一样
```



- 非静态内部类：`outer.new Inner1()`，**依附于一个外部实例**
- 静态内部类：`new Outer.Inner2()`，**独立创建**

**"静态" = "不依附外部实例"，不是"只有一个实例"。**

#### 为什么你会觉得冲突

你的直觉是：

> "static 不是表示只有一份吗？那 new 多个不就冲突了？"

这个直觉来自**静态变量**：

```java
static int count;   // 所有实例共享这一份 count
```

但静态变量是"**字段**"，`static` 修饰字段时表示"字段属于类，共享一份"。

而 `static` 修饰**类**时，含义变了——表示"**这个类不持有外部类引用**"。

**同一个关键字 `static`，修饰不同东西，含义不同。**

#### 顺带：什么时候用静态内部类

因为静态内部类**不持有外部引用**，所以：

- **省内存**：不像非静态内部类那样多一个 `this$0` 指针
- **避免内存泄漏**：不会因为内部类实例活着，导致外部类回收不掉
- **语义清晰**：如果内部类不需要访问外部实例成员，就该用 static

这也是为什么 `AQS` 的 `Sync`、`ReentrantLock` 的 `Sync` 都用 `static` —— 它们不需要外部锁实例的引用。

#### 一句话

> **静态内部类的 `static` 不是"只有一份实例"，而是"不持有外部类引用、可独立创建"。** 所以你当然能 `new` 多个静态内部类对象，和 `static` 完全不冲突。冲突感来自把"修饰字段的 static（共享一份）"和"修饰类的 static（不依附外部实例）"混为一谈了。

