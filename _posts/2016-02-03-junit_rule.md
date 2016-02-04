---
layout: post
title: Junit源码阅读(四)之自定义扩展
category: 源码阅读
tags: junit
date: 2016-02-03
---
{% include JB/setup %}


* 目录
{:toc}

---

### 前言

上次的博客中我们着重介绍了Junit的Validator机制，这次我们将聚焦到自定义扩展Rule上来。在很多情形下我们需要在测试过程中加入一些自定义的动作，这些就需要对statement进行包装，Junit为此提供了以TestRule接口和RunRules为基础的Rule扩展机制。

### 基本模型

首选TestRule由注解@ClassRule来指定，下面我们先给出TestRule的定义。

~~~java
public interface TestRule {
    /**
     * Modifies the method-running {@link Statement} to implement this
     * test-running rule.
     *
     * @param base The {@link Statement} to be modified
     * @param description A {@link Description} of the test implemented in {@code base}
     * @return a new statement, which may be the same as {@code base},
     *         a wrapper around {@code base}, or a completely new Statement.
     */
    Statement apply(Statement base, Description description);
}

~~~

代码中的注释十分清楚，TestRule提供结合Description为原Statement附加功能转变为新的Statement的apply方法。RunRules则是一系列TestRule作用后得到的Statement，如下：

~~~java
public class RunRules extends Statement {
    private final Statement statement;

    public RunRules(Statement base, Iterable<TestRule> rules, Description description) {
        statement = applyAll(base, rules, description);
    }

    @Override
    public void evaluate() throws Throwable {
        statement.evaluate();
    }

    private static Statement applyAll(Statement result, Iterable<TestRule> rules,
            Description description) {
        for (TestRule each : rules) {
            result = each.apply(result, description);
        }
        return result;
    }
}

~~~


### 原理解释

那么RunRules又是如何在我们的测试运行过程中被转化的呢？还记得在第二篇博客中我们提到了在classBlock方法中statement会被withBeforeClasses等装饰，同样此处它也被withClassRules装饰。首先由testClass返回带@ClassRule注解的对应值，分别由getAnnotatedFieldValues和getAnnotatedMethodValues方法提供。之后我们将这些值转化为TestRule对象，然后将这个TestRule列表和原有的statement结合返回RunRules。

~~~java
	private Statement withClassRules(Statement statement) {
        List<TestRule> classRules = classRules();
        return classRules.isEmpty() ? statement :
                new RunRules(statement, classRules, getDescription());
    }
~~~

### TimeOut示例

接下来我们以超时扩展为示例来看一看一个扩展是如何起作用的。

~~~java
public class Timeout implements TestRule {
    private final long timeout;
    private final TimeUnit timeUnit;
    private final boolean lookForStuckThread;

    
    public static Builder builder() {
        return new Builder();
    }

    
    @Deprecated
    public Timeout(int millis) {
        this(millis, TimeUnit.MILLISECONDS);
    }

    
    public Timeout(long timeout, TimeUnit timeUnit) {
        this.timeout = timeout;
        this.timeUnit = timeUnit;
        lookForStuckThread = false;
    }

    
    protected Timeout(Builder builder) {
        timeout = builder.getTimeout();
        timeUnit = builder.getTimeUnit();
        lookForStuckThread = builder.getLookingForStuckThread();
    }

    
    public static Timeout millis(long millis) {
        return new Timeout(millis, TimeUnit.MILLISECONDS);
    }

    
    public static Timeout seconds(long seconds) {
        return new Timeout(seconds, TimeUnit.SECONDS);
    }

    
    protected final long getTimeout(TimeUnit unit) {
        return unit.convert(timeout, timeUnit);
    }

    
    protected final boolean getLookingForStuckThread() {
        return lookForStuckThread;
    }

    
    protected Statement createFailOnTimeoutStatement(
            Statement statement) throws Exception {
        return FailOnTimeout.builder()
            .withTimeout(timeout, timeUnit)
            .withLookingForStuckThread(lookForStuckThread)
            .build(statement);
    }

    public Statement apply(Statement base, Description description) {
        try {
            return createFailOnTimeoutStatement(base);
        } catch (final Exception e) {
            return new Statement() {
                @Override public void evaluate() throws Throwable {
                    throw new RuntimeException("Invalid parameters for Timeout", e);
                }
            };
        }
    }

    
    public static class Builder {
        private boolean lookForStuckThread = false;
        private long timeout = 0;
        private TimeUnit timeUnit = TimeUnit.SECONDS;

        protected Builder() {
        }

        
        public Builder withTimeout(long timeout, TimeUnit unit) {
            this.timeout = timeout;
            this.timeUnit = unit;
            return this;
        }

        protected long getTimeout() {
            return timeout;
        }

        protected TimeUnit getTimeUnit()  {
            return timeUnit;
        }

        
        public Builder withLookingForStuckThread(boolean enable) {
            this.lookForStuckThread = enable;
            return this;
        }

        protected boolean getLookingForStuckThread() {
            return lookForStuckThread;
        }


        /**
         * Builds a {@link Timeout} instance using the values in this builder.,
         */
        public Timeout build() {
            return new Timeout(this);
        }
    }
}

~~~

我们可以看到上述最核心的就是createFailOnTimeoutStatement方法，它直接返回了一个FailOnTimeout，并且用它内建的Builder初始化。下面我们仅仅给出FailOnTimeout内部的域以及一些核心方法。

~~~java
public class FailOnTimeout extends Statement {
    private final Statement originalStatement;
    private final TimeUnit timeUnit;
    private final long timeout;
    private final boolean lookForStuckThread;

    private FailOnTimeout(Builder builder, Statement statement) {
        originalStatement = statement;
        timeout = builder.timeout;
        timeUnit = builder.unit;
        lookForStuckThread = builder.lookForStuckThread;
    }


    public static class Builder {
        private boolean lookForStuckThread = false;
        private long timeout = 0;
        private TimeUnit unit = TimeUnit.SECONDS;

        private Builder() {
        }

        public FailOnTimeout build(Statement statement) {
            if (statement == null) {
                throw new NullPointerException("statement cannot be null");
            }
            return new FailOnTimeout(this, statement);
        }
    }


    @Override
    public void evaluate() throws Throwable {
        CallableStatement callable = new CallableStatement();
        FutureTask<Throwable> task = new FutureTask<Throwable>(callable);
        ThreadGroup threadGroup = new ThreadGroup("FailOnTimeoutGroup");
        Thread thread = new Thread(threadGroup, task, "Time-limited test");
        thread.setDaemon(true);
        thread.start();
        callable.awaitStarted();
        Throwable throwable = getResult(task, thread);
        if (throwable != null) {
            throw throwable;
        }
    }

    private Throwable getResult(FutureTask<Throwable> task, Thread thread) {
        try {
            if (timeout > 0) {
                return task.get(timeout, timeUnit);
            } else {
                return task.get();
            }
        } catch (InterruptedException e) {
            return e; // caller will re-throw; no need to call Thread.interrupt()
        } catch (ExecutionException e) {
            // test failed; have caller re-throw the exception thrown by the test
            return e.getCause();
        } catch (TimeoutException e) {
            return createTimeoutException(thread);
        }
    }

    private class CallableStatement implements Callable<Throwable> {
        private final CountDownLatch startLatch = new CountDownLatch(1);

        public Throwable call() throws Exception {
            try {
                startLatch.countDown();
                originalStatement.evaluate();
            } catch (Exception e) {
                throw e;
            } catch (Throwable e) {
                return e;
            }
            return null;
        }

        public void awaitStarted() throws InterruptedException {
            startLatch.await();
        }
    }

}
~~~

可以看出它通过内置的Builder类来配置参数，通过CallableStatement和FutureTask启动新线程来运行真实的测试样例，并使用CountDownLatch来让父进程等待。实际的超时判断则借助了FutureTask的getResult，如果规定时间未返回结果就抛出超时异常。