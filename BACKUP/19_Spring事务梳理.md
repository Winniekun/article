# [Spring事务梳理](https://github.com/Winniekun/article/issues/19)

**<font style="color:rgb(34, 34, 34);">事务管理</font>**<font style="color:rgb(34, 34, 34);">，一个被说烂的也被看烂的话题，但是自己还是行程结构化的理解，所以抽个周末好好梳理了下，本文会从设计角度，一步步的剖析 Spring 事务管理的设计思路。</font>

<font style="color:rgb(34, 34, 34);">首先先抛出几个问题：</font>

+ **Spring 为什么要设计事务管理器？**
+ **传播行为到底是怎么实现的？**
+ **为什么 REQUIRES_NEW 可以做到“独立事务”？**
+ **AOP 自动挡事务与 TransactionTemplate 手动挡，本质区别是什么？**
+ **异常一定导致回滚吗？回滚策略是怎么决定的？**

<font style="color:rgb(34, 34, 34);">这篇文章不讲八股，不贴源码堆砌，尽量从设计者的视角理解拆解Spring的事务模型。</font>

# 为什么要事务管理
先看看如果没有事务管理器的话，如果想让多个操作（方法/类）处在一个事务里应该怎么做：
```java
// MethodA:
 public void methodA() {
     Connection connection = getOrCreateTransaction();
     try {
         connection.prepareStatement("INSERT INTO demo(name) VALUES ('A')").executeUpdate();
         // methodB 使用相同事务
         methodB();
         commit(tx);
     } catch (Exception e) {
         rollback(connection);
     } finally {
         cleanup(connection);
     }
 }

// MethodB:
public void methodB(Connection connection){
         connection.prepareStatement("INSERT INTO demo(name) VALUES ('B')").executeUpdate();
}
```
每次都要获取connection，可以再优化下代码，把connection放入ThreadLocal中做成上下文，然后将获取连接、提交、回滚操作封装成相关的方法
```java
// ThreadLocal 用于保存当前线程的事务资源
    static ThreadLocal<TransactionStatus> currentTx = new ThreadLocal<>();

    public static void main(String[] args) {
        ManualTransactionDemo demo = new ManualTransactionDemo();

        System.out.println("=== 场景 A：methodA -> methodB（共用同一事务）===");
        demo.methodA();

        System.out.println("\n=== 场景 B：methodA -> methodBNewTx（methodB 开启新事务）===");
        demo.methodA_NewTx();
    }

    //-------------------------------
    // 方法 A：使用当前事务
    //-------------------------------
    public void methodA() {
        TransactionStatus tx = getOrCreateTransaction();
        Connection connection = tx.connection;

        try {
            System.out.println("[methodA] 执行 SQL A");
            connection.prepareStatement("INSERT INTO demo(name) VALUES ('A')").executeUpdate();

            // methodB 使用相同事务
            methodB();

            commit(tx);
        } catch (Exception e) {
            rollback(tx);
        } finally {
            cleanup(tx);
        }
    }

    //-------------------------------
    // 方法 B：使用当前事务
    //-------------------------------
    public void methodB() {
        TransactionStatus tx = getCurrentTransaction();
        Connection connection = tx.connection;

        try {
            System.out.println("[methodB] 执行 SQL B");
            connection.prepareStatement("INSERT INTO demo(name) VALUES ('B')").executeUpdate();
        } catch (Exception e) {
            setRollbackOnly(tx);
        }
    }

    //-------------------------------
    // 方法 A：调用 methodB 但 methodB 是新事务（模拟 REQUIRES_NEW）
    //-------------------------------
    public void methodA_NewTx() {
        TransactionStatus tx = getOrCreateTransaction();
        Connection connection = tx.connection;

        try {
            System.out.println("[methodA_NewTx] 执行 SQL A");
            connection.prepareStatement("INSERT INTO demo(name) VALUES ('A2')").executeUpdate();

            // methodB 独立新事务
            methodB_REQUIRES_NEW();

            commit(tx);
        } catch (Exception e) {
            rollback(tx);
        } finally {
            cleanup(tx);
        }
    }

    //-------------------------------
    // 方法 B：开启独立事务（模拟 Propagation.REQUIRES_NEW）
    //-------------------------------
    public void methodB_REQUIRES_NEW() {
        // 挂起当前事务
        TransactionStatus suspended = suspend();

        // 新事务
        TransactionStatus newTx = getOrCreateTransaction();

        try {
            System.out.println("[methodB_REQUIRES_NEW] 执行 SQL B2");
            newTx.connection.prepareStatement("INSERT INTO demo(name) VALUES ('B2')").executeUpdate();

            commit(newTx);
        } catch (Exception e) {
            rollback(newTx);
        } finally {
            cleanup(newTx);

            // 恢复旧事务
            resume(suspended);
        }
    }

    //=====================================================
    // 事务管理核心（模拟 Spring）
    //=====================================================

    /**
     * 获取当前事务，没有则创建
     */
    private TransactionStatus getOrCreateTransaction() {
        TransactionStatus tx = currentTx.get();
        if (tx == null) {
            tx = new TransactionStatus(acquireConnection(), true);
            currentTx.set(tx);
        }
        return tx;
    }

    private TransactionStatus getCurrentTransaction() {
        return currentTx.get();
    }

    /**
     * 挂起当前事务
     */
    private TransactionStatus suspend() {
        TransactionStatus tx = currentTx.get();
        currentTx.remove();
        return tx;
    }

    /**
     * 恢复挂起事务
     */
    private void resume(TransactionStatus tx) {
        if (tx != null) {
            currentTx.set(tx);
        }
    }

    /**
     * 提交事务
     */
    private void commit(TransactionStatus tx) {
        if (tx.newTransaction && !tx.rollbackOnly) {
            try {
                System.out.println("[TX] commit");
                tx.connection.commit();
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        } else {
            System.out.println("[TX] 非新事务或设置为 rollbackOnly，跳过 commit");
        }
    }

    /**
     * 回滚事务
     */
    private void rollback(TransactionStatus tx) {
        if (tx.newTransaction) {
            try {
                System.out.println("[TX] rollback");
                tx.connection.rollback();
            } catch (SQLException e) {
                System.err.println("Rollback failed:" + e.getMessage());
            }
        } else {
            System.out.println("[TX] 非新事务，只标记 rollbackOnly");
            tx.rollbackOnly = true;
        }
    }

    /**
     * 清理资源
     */
    private void cleanup(TransactionStatus tx) {
        if (tx.newTransaction) {
            releaseConnection(tx.connection);
            currentTx.remove();
        }
    }

    private void setRollbackOnly(TransactionStatus tx) {
        tx.rollbackOnly = true;
    }

    //=====================================================
    // JDBC 原始连接操作（模拟文章）
    //=====================================================

    private Connection acquireConnection() {
        try {
            Connection conn =
                DriverManager.getConnection("jdbc:mysql://localhost:3306/user?useSSL=false&characterEncoding=utf8",
                    "root", "11111");
            conn.setAutoCommit(false);
            System.out.println("[TX] acquire new connection");
            return conn;
        } catch (Exception e) {
            throw new RuntimeException("Cannot acquire connection", e);
        }
    }

    private void releaseConnection(Connection connection) {
        try {
            System.out.println("[TX] release connection");
            connection.close();
        } catch (SQLException e) {
            System.err.println("Release failed: " + e.getMessage());
        }
    }

    //=====================================================
    // TransactionStatus：模仿 Spring
    //=====================================================
    static class TransactionStatus {
        Connection connection;
        boolean newTransaction;
        boolean rollbackOnly = false;

        public TransactionStatus(Connection conn, boolean newTx) {
            this.connection = conn;
            this.newTransaction = newTx;
        }
    }
```
但是太麻烦了，现在看起来好点了，不过我有一个新的需求：想让 methodB 独立一个新事务，单独提交和回滚，不影响 methodA，这……可就有点难搞了，ThreadLocal 中已经绑定了一个 Connection，再新事务的话就不好办了。那如果再复杂点呢，methodB 中需要调用 methodC，methodC 也需要一个独立事务……

而且，每次 bind/unbind 的操作也有点太傻了，万一哪个方法忘了写 unbind ，最后来一个连接泄露那不是完蛋了，好在 Spring 提供了事务管理器，帮我们解决了这一系列痛点。



# Spring事务管理解决了什么问题
<font style="color:rgb(34, 34, 34);">Spring 提供的事务管理可以帮我们管理事务相关的资源，比如 JDBC 的 Connection、Mybatis 的 SqlSession等。如刚才例子中的 Connection 绑定到 ThreadLocal 来解决共享一个事务的这种方式，Spring 事务管理就已经帮我们做好了。除了管理事务相关资源外，还可以帮我们处理复杂场景下的嵌套事务，比如前面说到的 methodB/methodC 独立事务。</font>  
嵌套事务：

```sql
public void tranansactionMethodA() {
  transactionMethodB();

}

public void transactionMethodB() {
  transactionMethodC()
}

public void transactionMethodC() {

}
```

在方法链中，多个方法都使用事务进行管理如tranansactionMethodA → transactionMethodB → transactionMethodC，就是嵌套事务。

## 事务传播行为
<font style="color:rgb(34, 34, 34);">在嵌套事务中，子方法使用当前事务、还是开启新的事物亦或者是不使用事务，对于子方法的事务处理策略，在 Spring 的事务管理中，</font>**<font style="color:rgb(34, 34, 34);">这个子方法的事务处理策略叫做事务传播行为（Propogation Behavior）</font>**<font style="color:rgb(34, 34, 34);">。</font>

<font style="color:rgb(34, 34, 34);">主要的行为共分为三种，具体的细分行为可自行搜索。</font>

| 分类 | 典型行为 | 说明 |
| --- | --- | --- |
| 使用当前事务 | REQUIRED | 默认策略 |
| 不使用当前事务，新建一个 | REQUIRES_NEW | 子方法必须独立事务 |
| 不使用事务 | NOT_SUPPORTED | 完全关闭事务 |


### 举例
```java
@Transactional
public void methodA() {
    String sql = "insert into employees (name, department_id, salary) value (?, ?, ?)";
    String name = UUID.randomUUID().toString() + "-A";
    Random random = new Random();
    int randomInt = random.nextInt(10000) + 1;
    jdbcTemplate.update(sql, name, 1, randomInt);
    springTransactionB.methodBDefault();
}


@Transactional()
public void methodBDefault() {
    String sql = "insert into employees (name, department_id, salary) value (?, ?, ?)";
    String name = UUID.randomUUID().toString() + "-B";
    Random random = new Random();
    int randomInt = random.nextInt(10000) + 1;
    jdbcTemplate.update(sql, name, 1, randomInt);
}


```



## 支持不同的事务配置
除了可以处理嵌套事务的传播行为，还可以为每个事务使用不同的配置，譬如不同的隔离级别：

```java
@Transactional
public void methodADiffTransactionLevel() {
    String sql = "insert into employees (name, department_id, salary) value (?, ?, ?)";
    String name = UUID.randomUUID().toString() + "-AA-TransactionLevel";
    Random random = new Random();
    int randomInt = random.nextInt(10000) + 1;
    jdbcTemplate.update(sql, name, 1, randomInt);
    springTransactionB.methodBDiffTransactionLevel();
}

// 事务A还未提交，事务B即可读取出来
@Transactional(propagation = Propagation.REQUIRES_NEW, isolation = Isolation.READ_UNCOMMITTED)
public void methodBDiffTransactionLevel() {
    String sql = "select * from employees";
    List<Map<String, Object>> list = jdbcTemplate.queryForList(sql);
    System.out.println(list);
}


```

![](https://cdn.nlark.com/yuque/0/2025/png/2099170/1763898186744-bfa1ba09-0cc9-4a80-8813-c81e1a8bbd54.png)

## <font style="color:rgb(34, 34, 34);">总结</font>
<font style="color:rgb(34, 34, 34);">好了，现在已经了解了 Spring 事务管理的所有核心功能，来总结一下这些核心功能点：</font>

1. <font style="color:rgb(34, 34, 34);">连接/资源管理 - 无需手动获取资源、共享资源、释放资源</font>
2. <font style="color:rgb(34, 34, 34);">嵌套事务的支持 - 支持嵌套事务中使用不同的资源策略、回滚策略</font>
3. <font style="color:rgb(34, 34, 34);">每个事务/连接使用不同的配置</font>



# 事务管理器的设计
通过前面的一些列介绍，可以直观的感觉出来，事务管理核心的工作：提交+回滚。关于传播等都是基于这两个做的拓展，所以Spring中，将事务管理的核心功能抽象为一个**事务管理器（Transaction Manager），**基于事务管理器，拓展出了各种各样的玩儿法。

```sql
public interface PlatformTransactionManager extends TransactionManager {

    // 获取事务资源
    TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException;
    
    // 提交
    void commit(TransactionStatus status) throws TransactionException;

    // 回滚
    void rollback(TransactionStatus status) throws TransactionException;
}

```

通过源码可以看出接口共提供了三个功能：

1. **<font style="color:rgb(34, 34, 34);">获取事务资源（getTransaction）</font>**<font style="color:rgb(34, 34, 34);">：</font>
    1. <font style="color:rgb(34, 34, 34);">资源可以是任意的，比如jdbc connection/hibernate mybatis session之类，然后绑定并存储</font>
2. **<font style="color:rgb(34, 34, 34);">提交事务（commit）：</font>**
    1. <font style="color:rgb(34, 34, 34);"> 提交指定的事务资源</font>
3. **<font style="color:rgb(34, 34, 34);">回滚事务（rollback）：</font>**
    1. <font style="color:rgb(34, 34, 34);"> 回滚指定的事务资源</font>

## 事务定义-TransactionDefinition
<font style="color:rgb(34, 34, 34);">还记得上面的 @Transactional 注解吗，里面定义了传播行为、隔离级别、回滚策略、只读之类的属性，这个就是一次事务操作的定义。在获取事务资源时，需要根据这个事务的定义来进行不同的配置：</font>

1. <font style="color:rgb(34, 34, 34);">比如配置了使用新事务，那么在获取事务资源时就需要创建一个新的，而不是已有的</font>
2. <font style="color:rgb(34, 34, 34);">比如配置了隔离级别，那么在首次创建资源（Connection）时，就需要给 Connection 设置 propagation</font>
3. <font style="color:rgb(34, 34, 34);">比如配置了只读属性，那么在首次创建资源（Connection）时，就需要给 Connection 设置 readOnly</font>

<font style="color:rgb(34, 34, 34);">为什么要单独用一个 TransactionDefinition 来存储事务定义，直接用注解的属性不行吗？当然可以，但注解的事务管理只是 Spring 提供的自动挡，还有适合老司机的手动挡事务管理（后面会介绍）；手动挡可用不了注解，所以单独建一个事务定义的模型，这样就可以实现通用。</font>

## 事务状态-TransactionStatus
<font style="color:rgb(34, 34, 34);">那既然嵌套事务下，每个子方法的事务可能不同，所以还得有一个子方法事务的状态 -TransactionStatus，用来存储当前事务的一些数据和状态，比如事务资源（Connection）、回滚状态等。</font>

## 获取事务资源
<font style="color:rgb(34, 34, 34);">事务管理器的第一步，就是根据事务定义来获取/创建资源了，这一步最麻烦的是要区分传播行为，不同传播行为下的逻辑不太一样。“默认的传播行为下，使用当前事务”，怎么算有当前事务呢？把事务资源存起来嘛，只要已经存在那就是有当前事务，直接获取已存储的事务资源就行。文中开头的例子也演示了，如果想让多个方法无感的使用同一个事务，可以用 ThreadLocal 存储起来，简单粗暴。  
</font><font style="color:rgb(34, 34, 34);">Spring 也是这么做的，不过它实现的更复杂一些，抽象了一层</font>**<font style="color:rgb(34, 34, 34);">事务资源同步管理器 - TransactionSynchronizationManager（本文后面会简称 TxSyncMgr）</font>**<font style="color:rgb(34, 34, 34);">，在这个同步管理器里使用 ThreadLocal 存储了事务资源。</font>

```java
public abstract class TransactionSynchronizationManager {

	private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<>("Transactional resources");

	private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
			new NamedThreadLocal<>("Transaction synchronizations");

	private static final ThreadLocal<String> currentTransactionName =
			new NamedThreadLocal<>("Current transaction name");

	private static final ThreadLocal<Boolean> currentTransactionReadOnly =
			new NamedThreadLocal<>("Current transaction read-only status");

	private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
			new NamedThreadLocal<>("Current transaction isolation level");

	private static final ThreadLocal<Boolean> actualTransactionActive =
			new NamedThreadLocal<>("Actual transaction active");

  ...
}
```

  
<font style="color:rgb(34, 34, 34);">剩下的就是根据不同传播行为，执行不同的策略了，分类之后只有 3 个条件分支：</font>

1. <font style="color:rgb(34, 34, 34);">当前有事务 - 根据不同传播行为处理不同</font>
2. <font style="color:rgb(34, 34, 34);">当前没事务，但需要开启新事务</font>
3. <font style="color:rgb(34, 34, 34);">彻底不用事务 - 这个很少用</font>

```java
public final TransactionStatus getTransaction(TransactionDefinition definition) {
    //创建事务资源 - 比如 Connection
    Object transaction = doGetTransaction();
    
    if (isExistingTransaction(transaction)) {
        // 处理当前已有事务的场景
        return handleExistingTransaction(def, transaction, debugEnabled);
    }else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED){
        
        // 开启新事务
    	return startTransaction(def, transaction, debugEnabled, suspendedResources);
    }else {
    	// 彻底不用事务
    }
    
    // ...
}
```

### 分支一
> **<font style="color:rgb(34, 34, 34);">当前有事务 - 根据不同传播行为处理不同</font>**<font style="color:rgb(34, 34, 34);">，这个就稍微有点麻烦了。因为有子方法独立事务的需求，可是 TransactionSynchronizationManager 却只能存一个事务资源。</font>`
>

#### <font style="color:rgb(34, 34, 34);">挂起（Suspend）和恢复（Resume）</font>
<font style="color:rgb(34, 34, 34);">Spring 采用了一种挂起(Suspend) - 恢复(Resume)的设计来解决这个嵌套资源处理的问题。当子方法需要独立事务时，就将当前事务挂起，从 TxSyncMgr 中移除当前事务资源，创建新事务的状态时，将挂起的事务资源保存至新的事务状态 TransactionStatus 中；在子方法结束时，只需要再从子方法的事务状态中，再次拿出挂起的事务资源，重新绑定至 TxSyncMgr 即可完成恢复的操作。整个挂起 - 恢复的流程，如下图所示：</font>

> **<font style="color:rgb(34, 34, 34);">注意：挂起操作是在获取事务资源这一步做的，而恢复的操作是在子方法结束时（提交或者回滚）中进行的。</font>**
>

![](https://cdn.nlark.com/yuque/0/2025/png/2099170/1763903759555-61f86782-48b6-4c4e-b7f6-0fed6edb90c9.png)

<font style="color:rgb(34, 34, 34);">这样一来，每个 TransactionStatus 都会保存挂起的前置事务资源，如果方法调用链很长，每次都是新事务的话，那这个 TransactionStatus 看起来就会像一个链表：</font>  
![](https://cdn.nlark.com/yuque/0/2025/png/2099170/1763903894636-74308732-2b92-4c7b-8123-fa33aeb3f419.png)

### 分支二
> **<font style="color:rgb(34, 34, 34);">分支 2 - 当前没事务，但需要开启新事务</font>**<font style="color:rgb(34, 34, 34);">，这个逻辑相对简单一些。只需要新建事务资源，然后绑定到 ThreadLocal 即可：</font>
>

```java
private TransactionStatus startTransaction(TransactionDefinition definition, Object transaction,
			boolean debugEnabled, @Nullable SuspendedResourcesHolder suspendedResources) {
	// 创建事务
	DefaultTransactionStatus status = newTransactionStatus(
			definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
	// 开启事务
    doBegin(transaction, definition);
    // 然后将事务资源绑定到事务资源管理器 TransactionSynchronizationManager
	prepareSynchronization(status, definition);
	return status;
}
```





## <font style="color:rgb(34, 34, 34);">自动挡与手动挡</font>
### <font style="color:rgb(34, 34, 34);">自动挡</font>
<font style="color:rgb(34, 34, 34);">XML/@Transactional 两种基于 AOP 的注解管理，其入口类是 TransactionInterceptor，是一个 AOP 的 Interceptor，负责调用事务管理器来实现事务管理。  
</font>

<font style="color:rgb(34, 34, 34);">因为核心功能都在事务管理器里实现，所以这个 AOP Interceptor 很简单，只是调用一下事务管理器，核心（伪）代码如下：</font>

```java
public Object invoke(MethodInvocation invocation) throws Throwable {
    
    // 获取事务资源
	Object transaction = transactionManager.getTransaction(txAttr);    
    Object retVal;
    
    try {
        // 执行业务代码
    	retVal = invocation.proceedWithInvocation();
        
        // 提交事务
        transactionManager.commit(txStatus);
    } catch (Throwable ex){
        // 先判断异常回滚策略，然后调用事务管理器的 rollback
    	rollbackOn(ex, txStatus);
    } 
}
```

<font style="color:rgb(34, 34, 34);">并且 AOP 这种自动挡的事务管理还增加了一个回滚策略的玩法，这个是手动挡 TransactionTemplate 所没有的，但这个功能并不在事务管理器中，只是 AOP 版事务的一个增强。</font>

### <font style="color:rgb(34, 34, 34);">手动挡</font>
`<font style="color:rgb(193, 121, 139);background-color:rgb(249, 242, 244);">TransactionTemplate</font>`<font style="color:rgb(34, 34, 34);"> </font><font style="color:rgb(34, 34, 34);">这个是手动挡的事务管理，虽然没有注解的方便，但是好在灵活，异常/回滚啥的都可以自己控制。  
</font>

<font style="color:rgb(34, 34, 34);">所以这个实现更简单，连异常回滚策略都没有，特殊的回滚方式还要自己设置（默认是任何异常都会回滚），核心（伪）代码如下：</font>

```java
public <T> T execute(TransactionCallback<T> action) throws TransactionException {
	
    // 获取事务资源
    TransactionStatus status = this.transactionManager.getTransaction(this);
    T result;
    try {
        
        // 执行 callback 业务代码
        result = action.doInTransaction(status);
    }
    catch (Throwable ex) {
        
        // 调用事务管理器的 rollback
        rollbackOnException(status, ex);
    }
    
    提交事务
    this.transactionManager.commit(status);
	}
}
```

### <font style="color:rgb(34, 34, 34);">为什么有这么方便的自动挡，还要手动挡？</font>
<font style="color:rgb(34, 34, 34);">因为手动挡更灵活啊，想怎么玩就怎么玩，比如我可以在一个方法中，执行多个数据库操作，但使用不同的事务资源：</font>

```java
Integer rows = new TransactionTemplate((PlatformTransactionManager) transactionManager,
                                       new DefaultTransactionDefinition(TransactionDefinition.ISOLATION_READ_UNCOMMITTED))
    .execute(new TransactionCallback<Integer>() {
        @Override
        public Integer doInTransaction(TransactionStatus status) {
			// update 0
            int rows0 = jdbcTemplate.update(...);
            
            // update 1
            int rows1 = jdbcTemplate.update(...);
            return rows0 + rows1;
        }
    });

Integer rows2 = new TransactionTemplate((PlatformTransactionManager) transactionManager,
                                        new DefaultTransactionDefinition(TransactionDefinition.ISOLATION_READ_UNCOMMITTED))
    .execute(new TransactionCallback<Integer>() {
        @Override
        public Integer doInTransaction(TransactionStatus status) {
            
            // update 2
            int rows2 = jdbcTemplate.update(...);
            return rows2;
        }
    });
```

<font style="color:rgb(34, 34, 34);">在上面这个例子里，通过 TransactionTemplate 我们可以精确的控制 update0/update1 使用同一个事务资源和隔离级别，而 update2 单独使用一个事务资源，并且不需要新建类加注解的方式。</font>

### <font style="color:rgb(34, 34, 34);">手自一体可以吗？</font>
<font style="color:rgb(34, 34, 34);">当然可以，只要我们使用的是同一个事务管理器的实例，因为绑定资源到同步资源管理器这个操作是在事务管理器中进行的。  
</font>

<font style="color:rgb(34, 34, 34);">AOP 版本的事务管理里，同样可以使用手动挡的事务管理继续操作，而且还可以使用同一个事务资源 。  
</font>

<font style="color:rgb(34, 34, 34);">比如下面这段代码，update1/update2 仍然在一个事务内，并且 update2 的 callback 结束后并不会提交事务，事务最终会在 methodA 结束时，TransactionInterceptor 中才会提交</font>

```java
@Transactional
public void methodA(){
    
    // update 1
	jdbcTemplate.update(...);
    new TransactionTemplate((PlatformTransactionManager) transactionManager,
                                        new DefaultTransactionDefinition(TransactionDefinition.ISOLATION_READ_UNCOMMITTED))
    .execute(new TransactionCallback<Integer>() {
        @Override
        public Integer doInTransaction(TransactionStatus status) {
            
            // update 2
            int rows2 = jdbcTemplate.update(...);
            return rows2;
        }
    });
   
}
```



# 总结
<font style="color:rgb(34, 34, 34);">Spring 的事务管理，其核心是一个抽象的事务管理器，XML/@Transactional/TransactionTemplate 几种方式都是基于这个事务管理器的，三中方式的核心实现区别并不大，只是入口不同而已。</font>
<img width="1140" height="950" alt="Image" src="https://github.com/user-attachments/assets/eaab1ef9-75b3-4f22-9fc7-c79417aa7c96" />

)

