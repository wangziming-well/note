Spring规定了三个主要接口来规范Spring事务管理的框架：

* `PlatformTransactionManager`：界定事务边界
* `TransactionDefinition`：负责定义事务相关属性，如隔离级别、传播行为等
* `TransactionStatus`：负责的事务状态

# TransactionDefinition

`TransactionDefinition`定义如下:

~~~java
public interface TransactionDefinition {
	int PROPAGATION_REQUIRED = 0;
	int PROPAGATION_SUPPORTS = 1;
	int PROPAGATION_MANDATORY = 2;
	int PROPAGATION_REQUIRES_NEW = 3;
	int PROPAGATION_NOT_SUPPORTED = 4;
	int PROPAGATION_NEVER = 5;
	int PROPAGATION_NESTED = 6;
	int ISOLATION_DEFAULT = -1;
	int ISOLATION_READ_UNCOMMITTED = 1;
	int ISOLATION_READ_COMMITTED = 2;
	int ISOLATION_REPEATABLE_READ = 4;
	int ISOLATION_SERIALIZABLE = 8;
	int TIMEOUT_DEFAULT = -1;

	default int getPropagationBehavior() {
		return PROPAGATION_REQUIRED;
	}

	default int getIsolationLevel() {
		return ISOLATION_DEFAULT;
	}

	default int getTimeout() {
		return TIMEOUT_DEFAULT;
	}

	default boolean isReadOnly() {
		return false;
	}


	default String getName() {
		return null;
	}

	static TransactionDefinition withDefaults() {
		return StaticTransactionDefinition.INSTANCE;
	}
}
~~~

## 事务属性

`TransactionDefinition`主要定义了可以指定的事务属性：

* 事务隔离级别

* 事务的传播行为

* 是否为只读事务:isReadOnly()

* 事务的超时时间:TIMEOUT_DEFAULT

## 继承体系

![TransactionAttribute](https://gitee.com/wangziming707/note-pic/raw/master/img/TransactionAttribute.png)

`TransactionDefinition`的实现类根据使用场景可以划分为两类：

* 适用于编程式事务场景:`DefaultTransactionDefiniton`
* 适用于声明式事务场景:`TransactionAttribute`

### 编程式事务场景

* `DefaultTransactionDefiniton`是`TransactionDefiniton`的默认实现类，它提供了各事务属性的默认值，并可以通过它的setter方法改变这些默认值

* `TransactionTemplate`是Spring提供的进行编程式事务管理的模板方法类，它直接继承了`DefaultTransactionDefiniton`

### 声明式事务场景

* `TransactionAttribute`是继承自`TransactionDefinition`的接口定义，主要面向使用Spring AOP进行声明式事务管理的场合。它自己定义了一个rollbackOn方法：

  ```java
  boolean rollbackOn(Throwable ex);
  ```

  可以指定在抛出哪些异常的情况下可以回滚事务。

* `DefaultTransactionAttribute`是`TransactionAttribute`接口的默认实现类，并同时继承了`DefaultTransactionDefiniton`

* `RuleBasedTransactionAttribute`：允许我们同时指定多个回滚规则

  内部维护了一个`List<RollbackRuleAttribute> rollbackRules;`

  它在调用rollbackOn方法时，传入的异常类型会先和rollbackRules中的回滚规则进行匹配，再决定是否要回滚事务。

* `DelegatingTransactionAttribute `是抽象类，它的目的就是为了被子类化。

# TransactionStatus

TransactionStatus表示整个事务处理过程中的事务状态，我们更多的在编程式事务中使用它

它的继承体系如下：

![TransactionStatus](https://gitee.com/wangziming707/note-pic/raw/master/img/TransactionStatus.png)

* SavepointManager:为Savepoint的支持提供抽象
* TransactionStatus：
  * 继承了SavepointManager，所以也获得了管理Savepoint的能力，从而支持创建内部嵌套事务。
  * 提供了相应的方法查询事务的状态
  * 通过`setRollbackOnly()`方法标记当前事务以使其回滚
* AbstractTransactionStatus：是TransactionStatus的抽象类实现，为其他实现子类提供一些公共设施
* DefaultTransactionStatus：是Spring事务框架内部使用的主要TransactionStatus实现类
* SimpleTransactionStatus：没有使用，仅测试用

# PlatformTransactionManager

PlatformTransactionManager是Spring事务抽象框架的核心组件，通过它来开启事务和提交回滚事务以界定事务边界。

它的定义如下:

~~~java
public interface PlatformTransactionManager extends TransactionManager {

	TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
			throws TransactionException;

	void commit(TransactionStatus status) throws TransactionException;

	void rollback(TransactionStatus status) throws TransactionException;

}
~~~

## 继承体系

PlatformTransactionManager的整个抽象体系基于策略模式，由PlatformTransactionManager对事务界定进行统一抽象，而具体的界定策略的实现由具体的实现类。它的实现类可以划分为面向局部事务和分布式事务两个分支，而它的具体实现类基本会先继承AbstractPlatformTransactionManager抽象类，间接继承PlatformTransactionManager

## 概念

* transaction object :承载了当前事务的必要信息，PlatformTransactionManager实现类可以根据transaction object所提供的信息来决定如何处理当前事务
* TransactionSynchronization：是可以注册到事务处理过程中的回调接口，像是事务处理的事件监听器，当事务发生回滚，提交等事件时，会调用TransactionSynchronization上的一些方法来执行相应的回调逻辑
* TransactionSynchronizationManager:通过它来管理和持有TransactionSynchronization、当前事务状态以及具体的事务资源。

## AbstractPlatformTransactionManager

AbstractPlatformTransactionManager作为PlatformTransactionManager实现类的抽象父类，以模板方法的方式封装了固定的事务处理逻辑，它替各个子类实现了固定的事务内部处理逻辑：

* 判断是否存在当前事务，根据判断结果执行不同的处理逻辑
* 结合当前是否存在当前事务的情况，根据TransactionDefinition中指定的传播行为的不同语义执行后续逻辑
* 根据情况挂起或者恢复事务
* 提交事务之前检查readOnly字段是否被设置，如果是的话，以事务的回滚代替事务的提交；
* 在事务回滚的情况下，清理并恢复事务状态
* 如果事务的Synchronization处于active状态，在事务处理的指定节点触发注册的Synchronization回调接口

这些固定的事务内部处理逻辑大都体现在以下几个主要的模板方法中：

~~~java
public final TransactionStatus getTransaction( TransactionDefinition definition)
public final void rollback(TransactionStatus status) ;
public final void commit(TransactionStatus status);

protected final SuspendedResourcesHolder suspend(Object transaction)
protected final void resume( Object transaction,  SuspendedResourcesHolder resourcesHolder)
~~~

### getTransaction()

该方法的主要目的是开启一个事务，在这之前需要判断之前是否存在事务。根据情况根据传播行为的具体语义处理事务状态。

该方法的主要处理逻辑如下：

~~~java
@Override
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
    throws TransactionException {
	//检查参数合法性，如果definition参数为空，则创建一个默认的TransactionDefinition实例以提供默认的事务定义数据
    TransactionDefinition def = (definition != null ? definition : TransactionDefinition.withDefaults());
	//获取transaction object
    //获取的transaction object类型由具体的实现类来决定
	// doGetTransaction()方法是需要子类来实现的abstract
    Object transaction = doGetTransaction();
    
    boolean debugEnabled = logger.isDebugEnabled();
	//根据获取的transaction object判断是否存在当前事务；isExistingTransaction()方法由实现类实现，判断当前是否存在事务
    if (isExistingTransaction(transaction)) {
        //如果isExistingTransaction()方法返回true，即存在事务，调用handleExistingTransaction()方法统一处理存在当前事务的情况
        return handleExistingTransaction(def, transaction, debugEnabled);
    }
    //接下来处理不存在当前事务的情况

    if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", def.getTimeout());
    }
	//如果definition定义的传播行为是PROPAGATION_MANDATORY,将抛出异常
    if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
            "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    //如果definition定义的传播行为是PROPAGATION_REQUIRED、PROPAGATION_REQUIRES_NEW或者PROPAGATION_NESTED，将开启新的事务
    else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
             def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
             def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        //传入null的suspend(),将可能有的Synchronization挂起放到一边，因为这与新事务无关
        SuspendedResourcesHolder suspendedResources = suspend(null);
        if (debugEnabled) {
            logger.debug("Creating new transaction with name [" + def.getName() + "]: " + def);
        }
        //创建新的事务并返回
        try {
            return startTransaction(def, transaction, debugEnabled, suspendedResources);
        }
        catch (RuntimeException | Error ex) {
            resume(null, suspendedResources);
            throw ex;
        }
    }
    //其他情况，将创建空的事务并返回
    else {
        if (def.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
            logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                        "isolation level will effectively be ignored: " + def);
        }
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
    }
}
~~~

~~~java
//如果当前存在事务，将调用该方法处理传播行为
private TransactionStatus handleExistingTransaction(
    TransactionDefinition definition, Object transaction, boolean debugEnabled)
    throws TransactionException {
	//如果definition定义的传播行为是PROPAGATION_NEVER，将抛出异常并退出
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException(
            "Existing transaction found for transaction marked with propagation 'never'");
    }
    
	//如果definition定义的传播行为是PROPAGATION_NOT_SUPPORTED，则挂起当前事务，并返回默认的TransactionStatus
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
        if (debugEnabled) {
            logger.debug("Suspending current transaction");
        }
        Object suspendedResources = suspend(transaction);
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(
            definition, null, false, newSynchronization, debugEnabled, suspendedResources);
    }
    
	//如果definition定义的传播行为是PROPAGATION_REQUIRES_NEW，则挂起当前事务，并开启一个新的事务返回
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
        if (debugEnabled) {
            logger.debug("Suspending current transaction, creating new transaction with name [" +
                         definition.getName() + "]");
        }
        //挂起当前事务
        SuspendedResourcesHolder suspendedResources = suspend(transaction);
        try {
            //开启一个新的事务
            return startTransaction(definition, transaction, debugEnabled, suspendedResources);
        }
        catch (RuntimeException | Error beginEx) {
            resumeAfterBeginException(transaction, suspendedResources, beginEx);
            throw beginEx;
        }
    }
	///如果definition定义的传播行为是PROPAGATION_NESTED,会根据情况创建嵌套事务
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        //如果不允许创建嵌套事务，则抛出异常
        if (!isNestedTransactionAllowed()) {
            throw new NestedTransactionNotSupportedException(
                "Transaction manager does not allow nested transactions by default - " +
                "specify 'nestedTransactionAllowed' property with value 'true'");
        }
        if (debugEnabled) {
            logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
        }
        //如果实现类是使用Savepoint创建事务，则使用TransactionStatus创建相应的Savepoint并返回
        if (useSavepointForNestedTransaction()) {
            DefaultTransactionStatus status =
                prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
            status.createAndHoldSavepoint();
            return status;
        }
        //否则，通过嵌套调用doBegin创建嵌套事务，通常只针对JTA事务
        else {
            return startTransaction(definition, transaction, debugEnabled, null);
        }
    }
    if (debugEnabled) {
        logger.debug("Participating in existing transaction");
    }
    //进行当前事务和definition属性的一致性校验
    if (isValidateExistingTransaction()) {
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
            Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
            if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
                Constants isoConstants = DefaultTransactionDefinition.constants;
                throw new IllegalTransactionStateException("Participating transaction with definition [" +definition + "] specifies isolation level which is incompatible with existing transaction: " +(currentIsolationLevel != null ?isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :"(unknown)"));
            }
        }
        if (!definition.isReadOnly()) {
            if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
                throw new IllegalTransactionStateException("Participating transaction with definition [" +definition + "] is not marked as read-only but existing transaction is");
            }
        }
    }
    //对于其他的传播行为，比如PROPAGATION_REQUIRE和PROPAGATION_SUPORTS,PROPAGATION_MANDATORY将直接构建TransactionStatus并返回
    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}
~~~

### rollback()

~~~java
public final void rollback(TransactionStatus status) throws TransactionException {
    //如果事务已经完成，抛出异常
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException(
                "Transaction is already completed - do not call commit or rollback more than once per transaction");
    }

    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    //调用processRollback，执行回滚
    processRollback(defStatus, false);
}
~~~



~~~java
//执行具体的回滚操作
private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
    try {
        boolean unexpectedRollback = unexpected;

        try {
            //触发TransactionSynchronization的beforeCompletion()回调方法
            triggerBeforeCompletion(status);
			//如果有savepoint，则回滚到上一个保存点并释放savepoint
            if (status.hasSavepoint()) {
                if (status.isDebug()) {
                    logger.debug("Rolling back transaction to savepoint");
                }
                status.rollbackToHeldSavepoint();
            }
            //如果是新事务，则调用方法doRollback进行回滚，该方法由子类实现
            else if (status.isNewTransaction()) {
                if (status.isDebug()) {
                    logger.debug("Initiating transaction rollback");
                }
                doRollback(status);
            }
            else {
                // 如果当前存在事务
                if (status.hasTransaction()) {
                    //如果事务属性是只能归滚的，将调用doSetRollbackOnly方法进行回滚，该方法有具体实现类实现
                    if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
                        if (status.isDebug()) {
                            logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
                        }
                        doSetRollbackOnly(status);
                    }
                    else {
                        if (status.isDebug()) {
                            logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
                        }
                    }
                }
                else {
                    logger.debug("Should roll back transaction but cannot - no transaction available");
                }
                // Unexpected rollback only matters here if we're asked to fail early
                if (!isFailEarlyOnGlobalRollbackOnly()) {
                    unexpectedRollback = false;
                }
            }
        }
        catch (RuntimeException | Error ex) {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            throw ex;
        }
		//触发TransactionSynchronization的afterCompletion()回调方法
        triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
        if (unexpectedRollback) {
            throw new UnexpectedRollbackException(
                    "Transaction rolled back because it has been marked as rollback-only");
        }
    }
    finally {
        cleanupAfterCompletion(status);
    }
}
~~~

### commit()

~~~java
public final void commit(TransactionStatus status) throws TransactionException {
    //如果事务已经完成，报错
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException(
                "Transaction is already completed - do not call commit or rollback more than once per transaction");
    }

    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    //如果事务是只能回滚的，将执行回滚
    if (defStatus.isLocalRollbackOnly()) {
        if (defStatus.isDebug()) {
            logger.debug("Transactional code has requested rollback");
        }
        processRollback(defStatus, false);
        return;
    }
    if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
        if (defStatus.isDebug()) {
            logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
        }
        processRollback(defStatus, true);
        return;
    }
	//调用processCommit()执行回滚操作
    processCommit(defStatus);
}
~~~



~~~java
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    try {
        boolean beforeCompletionInvoked = false;

        try {
            boolean unexpectedRollback = false;
            //为提交做准备，由具体实现类实现该方法
            prepareForCommit(status);
            //调用TransactionSynchronization的beforeCommit()回调
            triggerBeforeCommit(status);
            //调用TransactionSynchronization的beforeCompetion()回调
            triggerBeforeCompletion(status);
            beforeCompletionInvoked = true;
			//如果由savepoint，释放事务持有的savepoint
            if (status.hasSavepoint()) {
                if (status.isDebug()) {
                    logger.debug("Releasing transaction savepoint");
                }
                unexpectedRollback = status.isGlobalRollbackOnly();
                status.releaseHeldSavepoint();
            }
            //如果是新事务，调用doCommit()执行提交，由具体实现类实现该方法
            else if (status.isNewTransaction()) {
                if (status.isDebug()) {
                    logger.debug("Initiating transaction commit");
                }
                unexpectedRollback = status.isGlobalRollbackOnly();
                doCommit(status);
            }
            else if (isFailEarlyOnGlobalRollbackOnly()) {
                unexpectedRollback = status.isGlobalRollbackOnly();
            }

            if (unexpectedRollback) {
                throw new UnexpectedRollbackException(
                        "Transaction silently rolled back because it has been marked as rollback-only");
            }
        }
        catch (UnexpectedRollbackException ex) {

            triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
            throw ex;
        }
        catch (TransactionException ex) {
            if (isRollbackOnCommitFailure()) {
                doRollbackOnCommitException(status, ex);
            }
            else {
                //调用TransactionSynchronization的afterCompetion()回调
                triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            }
            throw ex;
        }
        catch (RuntimeException | Error ex) {
            if (!beforeCompletionInvoked) {
                triggerBeforeCompletion(status);
            }
            doRollbackOnCommitException(status, ex);
            throw ex;
        }

        try {
            triggerAfterCommit(status);
        }
        finally {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
        }

    }
    finally {
        //完成后执行清理
        cleanupAfterCompletion(status);
    }
}
~~~

