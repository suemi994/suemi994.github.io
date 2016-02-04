---
layout: post
title: Junit源码阅读(五)
category: 源码阅读
tags: junit
date: 2016-02-06
---
{% include JB/setup %}


* 目录
{:toc}

---

### 前言

尽管在第二次博客中我们讲述了Runner的运行机制，但是许多其他特性比如Filter是如何与运行流程结合却并不清楚。这次我们来回顾整理一下Junit的执行流程，给出各种特性生效的机理，并分析一些代码中精妙的地方。

### Junit的执行流程

JUnitCore的RunMain方法，使用jUnitCommandLineParseResult解析参数并生成Request。

~~~java
Result runMain(JUnitSystem system, String... args) {
        system.out().println("JUnit version " + Version.id());

        JUnitCommandLineParseResult jUnitCommandLineParseResult = JUnitCommandLineParseResult.parse(args);

        RunListener listener = new TextListener(system);
        addListener(listener);

        return run(jUnitCommandLineParseResult.createRequest(defaultComputer()));
    }
~~~

在jUnitCommandLineParseResult的createRequest方法中，调用Request的classes方法生成Request，并对生成的Request进行过滤

~~~java
public Request createRequest(Computer computer) {
        if (parserErrors.isEmpty()) {
            Request request = Request.classes(
                    computer, classes.toArray(new Class<?>[classes.size()]));
            return applyFilterSpecs(request);
        } else {
            return errorReport(new InitializationError(parserErrors));
        }
    }
~~~

接下来我们就要进入核心部分了，先提出以下几个问题：

- 如何为单个类生成Request 
- Filter的实现机制
- 对于错误如何把它纳入以Request为初始并最终使用Runner的run这一套机制中

我们先回答第一个问题，其他问题我们会在之后慢慢解答：

接下来先给出classes方法的代码

~~~java
public static Request classes(Computer computer, Class<?>... classes) {
        try {
            AllDefaultPossibilitiesBuilder builder = new AllDefaultPossibilitiesBuilder(true);
            Runner suite = computer.getSuite(builder, classes);
            return runner(suite);
        } catch (InitializationError e) {
            return runner(new ErrorReportingRunner(e, classes));
        }
    }
~~~

问题再次被拆分为三个：

- 生成一个Builder
- 从Builder导出一个Runner
- 从Runner生成一个Request

AllDefaultPossibilitiesBuilder的逻辑是先后使用ignoredBuilder、annotatedBuilder、suiteMethodBuilder、junit3Builder、junit4Builder来生成Runner直到有一个成功则返回，然后getSuite方法将返回的Runner变为Suite。

Computer的作用是包装从builder生成Runner的逻辑，提供两种方案——生成SingleClassRunner和Suite。我们在这里提一下生成Suite的逻辑，就是使用该builder不停为多个测试类生成对应的Runner并放置到Suite的Runner列表中，Suite其他的初始化过程依从其父类，此处就不详述。

最后从Runner生成Request也异常简单，也就是实现其getRunner方法返回该Runner。现在我们把注意力投放到builder如何导出Runner，以JUnit4Builder为例，下面给出代码：

~~~java
 public class JUnit4Builder extends RunnerBuilder {
    @Override
    public Runner runnerForClass(Class<?> testClass) throws Throwable {
        return new BlockJUnit4ClassRunner(testClass);
    }
}
~~~

可以看出它其实就是直接生成了一个BlockJUnit4ClassRunner，下面我们关注该Runner的构造过程。

~~~java
	protected ParentRunner(Class<?> testClass) throws InitializationError {
        this.testClass = createTestClass(testClass);
        validate();
    }

    protected TestClass createTestClass(Class<?> testClass) {
        return new TestClass(testClass);
    }
~~~

可以看出构造SingleClassRunne的过程就是解析生成TestClass的过程，我们在第二篇博客里已经详细讲解过了，此处就不再赘述了。可能许多读者看到这儿会很困惑，之前一直强调的描述测试样例的Description到底是在哪里生成的呢？其实是在Runner的run过程里生成的，如下：

~~~java
@Override
    public void run(final RunNotifier notifier) {
        EachTestNotifier testNotifier = new EachTestNotifier(notifier,
                getDescription());
        try {
            Statement statement = classBlock(notifier);
            statement.evaluate();
        } catch (AssumptionViolatedException e) {
            testNotifier.addFailedAssumption(e);
        } catch (StoppedByUserException e) {
            throw e;
        } catch (Throwable e) {
            testNotifier.addFailure(e);
        }
    }
    @Override
    public Description getDescription() {
        Description description = Description.createSuiteDescription(getName(),
                getRunnerAnnotations());
        for (T child : getFilteredChildren()) {
            description.addChild(describeChild(child));
        }
        return description;
    }

~~~

可以明显地得知Description是依据Runner来建立的，它的结构和Runner应该保持一致，它是Runner的附属品，用来供与Runner交互的Notifier获得信息。


### Junit的Failure机制

下面我们主要关注在运行之后Result的生成和Failure的处理。Result是Notifier通过fireTestRunFinished(result)生成的，我们来看一看它具体做了什么。

~~~java
	private abstract class SafeNotifier {
        private final List<RunListener> currentListeners;

        SafeNotifier() {
            this(listeners);
        }

        SafeNotifier(List<RunListener> currentListeners) {
            this.currentListeners = currentListeners;
        }

        void run() {
            int capacity = currentListeners.size();
            List<RunListener> safeListeners = new ArrayList<RunListener>(capacity);
            List<Failure> failures = new ArrayList<Failure>(capacity);
            for (RunListener listener : currentListeners) {
                try {
                    notifyListener(listener);
                    safeListeners.add(listener);
                } catch (Exception e) {
                    failures.add(new Failure(Description.TEST_MECHANISM, e));
                }
            }
            fireTestFailures(safeListeners, failures);
        }

        abstract protected void notifyListener(RunListener each) throws Exception;
    }

    public void fireTestRunFinished(final Result result) {
        new SafeNotifier() {
            @Override
            protected void notifyListener(RunListener each) throws Exception {
                each.testRunFinished(result);
            }
        }.run();
    }
    public void fireTestAssumptionFailed(final Failure failure) {
        new SafeNotifier() {
            @Override
            protected void notifyListener(RunListener each) throws Exception {
                each.testAssumptionFailure(failure);
            }
        }.run();
    }
     private void fireTestFailures(List<RunListener> listeners,
            final List<Failure> failures) {
        if (!failures.isEmpty()) {
            new SafeNotifier(listeners) {
                @Override
                protected void notifyListener(RunListener listener) throws Exception {
                    for (Failure each : failures) {
                        listener.testFailure(each);
                    }
                }
            }.run();
        }
    }
~~~

简单概括一下，就是调用它所管理的Listener的testRunFinished来处理Result，处理完毕之后就把该Listener标记为安全的，如果处理过程中出现异常，则将该异常加入failures列表，全部通知完毕后再次通知所有安全的Listener处理之前所有的Failure。这里还有一个问题，那就是Failure到底是怎样在运行时生成的。由于断言机制，所有的断言失败都会抛出对应的异常，对于断言异常，请看下文：

~~~java
@Override
    protected void runChild(final FrameworkMethod method, RunNotifier notifier) {
        Description description = describeChild(method);
        if (isIgnored(method)) {
            notifier.fireTestIgnored(description);
        } else {
            Statement statement;
            try {
                statement = methodBlock(method);
            }
            catch (Throwable ex) {
                statement = new Fail(ex);
            }
            runLeaf(statement, description, notifier);
        }
    }
~~~

Fail会直接抛出原有的异常，再次调用RunLeaf

~~~java
	protected final void runLeaf(Statement statement, Description description,
            RunNotifier notifier) {
        EachTestNotifier eachNotifier = new EachTestNotifier(notifier, description);
        eachNotifier.fireTestStarted();
        try {
            statement.evaluate();
        } catch (AssumptionViolatedException e) {
            eachNotifier.addFailedAssumption(e);
        } catch (Throwable e) {
            eachNotifier.addFailure(e);
        } finally {
            eachNotifier.fireTestFinished();
        }
    }
~~~



最后转入addFailure处理，而addFailure会最终通知Notifier中的各个Listener处理
。那JunitCore到底使用了怎样的Listener，它又执行了怎样的操作。

~~~java
public Result run(Runner runner) {
        Result result = new Result();
        RunListener listener = result.createListener();
        notifier.addFirstListener(listener);
        try {
            notifier.fireTestRunStarted(runner.getDescription());
            runner.run(notifier);
            notifier.fireTestRunFinished(result);
        } finally {
            removeListener(listener);
        }
        return result;
    }
~~~

注意JunitCore的Run方法中加入了一个Result中内置的Listener，其定义如下

~~~java
@RunListener.ThreadSafe
    private class Listener extends RunListener {
        @Override
        public void testRunStarted(Description description) throws Exception {
            startTime.set(System.currentTimeMillis());
        }

        @Override
        public void testRunFinished(Result result) throws Exception {
            long endTime = System.currentTimeMillis();
            runTime.addAndGet(endTime - startTime.get());
        }

        @Override
        public void testFinished(Description description) throws Exception {
            count.getAndIncrement();
        }

        @Override
        public void testFailure(Failure failure) throws Exception {
            failures.add(failure);
        }

        @Override
        public void testIgnored(Description description) throws Exception {
            ignoreCount.getAndIncrement();
        }

        @Override
        public void testAssumptionFailure(Failure failure) {
            // do nothing: same as passing (for 4.5; may change in 4.6)
        }
    }

~~~

可以看出该Listener在testFailure中完成了添加Failure的动作，看到这里我简直激动莫名，使用一个内置的Listener子类来避免显式为Result添加Failure，而是依然把这些操作集中在Notifier——Listener的观察者体系里，可谓精妙绝伦！

同时还需注意在runMain方法中加入了一个TextListener来完成打印结果的工作。


### Junit的初始化错误处理

上面我们讲述了Junit如何处理样例中的断言错误以及运行时错误，但是当初始化Request的过程中一旦发生异常依然需要继续运行并提示测试失败，这又是怎么实现的呢？

我们回顾之前createRequest方法中的errorReport，代码如下：

~~~java
	public static Request errorReport(Class<?> klass, Throwable cause) {
        return runner(new ErrorReportingRunner(klass, cause));
    }
~~~

我们来看一下这个ErrorReportingRunner的实现

~~~java
public class ErrorReportingRunner extends Runner {
    private final List<Throwable> causes;

    private final String classNames;

    public ErrorReportingRunner(Class<?> testClass, Throwable cause) {
        this(cause, new Class<?>[] { testClass });
    }
    
    public ErrorReportingRunner(Throwable cause, Class<?>... testClasses) {
        if (testClasses == null || testClasses.length == 0) {
            throw new NullPointerException("Test classes cannot be null or empty");
        }
        for (Class<?> testClass : testClasses) {
            if (testClass == null) {
                throw new NullPointerException("Test class cannot be null");
            }
        }
        classNames = getClassNames(testClasses);
        causes = getCauses(cause);
    }
    
    @Override
    public Description getDescription() {
        Description description = Description.createSuiteDescription(classNames);
        for (Throwable each : causes) {
            description.addChild(describeCause(each));
        }
        return description;
    }

    @Override
    public void run(RunNotifier notifier) {
        for (Throwable each : causes) {
            runCause(each, notifier);
        }
    }

    private String getClassNames(Class<?>... testClasses) {
        final StringBuilder builder = new StringBuilder();
        for (Class<?> testClass : testClasses) {
            if (builder.length() != 0) {
                builder.append(", ");
            }
            builder.append(testClass.getName());
        }
        return builder.toString();
    }

    @SuppressWarnings("deprecation")
    private List<Throwable> getCauses(Throwable cause) {
        if (cause instanceof InvocationTargetException) {
            return getCauses(cause.getCause());
        }
        if (cause instanceof InitializationError) {
            return ((InitializationError) cause).getCauses();
        }
        if (cause instanceof org.junit.internal.runners.InitializationError) {
            return ((org.junit.internal.runners.InitializationError) cause)
                    .getCauses();
        }
        return Arrays.asList(cause);
    }

    private Description describeCause(Throwable child) {
        return Description.createTestDescription(classNames, "initializationError");
    }

    private void runCause(Throwable child, RunNotifier notifier) {
        Description description = describeCause(child);
        notifier.fireTestStarted(description);
        notifier.fireTestFailure(new Failure(description, child));
        notifier.fireTestFinished(description);
    }
}

~~~

上面的大致逻辑依然是先生成Description（因为没有到ParentRunner的run那一步，Description没有生成，故需要在此生成），然后移交给Notifier处理

