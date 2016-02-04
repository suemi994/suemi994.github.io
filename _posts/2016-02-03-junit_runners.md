---
layout: post
title: Junit源码阅读(二)之样例运行的机制
category: 源码阅读
tags: junit
date: 2016-02-03
---
{% include JB/setup %}


* 目录
{:toc}

---

### 前言

在上次的博客中我们提到了最终由Runner以Notifier为参数执行测试样例，但并没有解释到底测试方法是如何被运行起来的，一些诸如RunWith、RunAfter之类的特性又到底是如何实现的呢。这次我们就集中深入Runner的运行机制来探究样例是如何被运行的。

### 包装注解信息——FrameWorkMember

首先我们需要把注解等用户配置信息收集起来并attach到对应的方法、类和属性上，为了在之后的代码中能够方便的取到这些信息，我们要包装原有的类、方法和域，分别如下。

#### TestClass

TestClass包含原有的clazz信息，并且维护了两个Map来管理它所包含的方法与属性，每个map的键是注解，而值是标上注解的FrameWorkMethod或FrameWorkField。同时TestClass还默认内置两个Comparator来排序自己所包含的方法和属性。

下面给出如何构造一个TestClass的代码。

~~~java
public TestClass(Class<?> clazz) {
        this.clazz = clazz;
        if (clazz != null && clazz.getConstructors().length > 1) {
            throw new IllegalArgumentException(
                    "Test class can only have one constructor");
        }

        Map<Class<? extends Annotation>, List<FrameworkMethod>> methodsForAnnotations =
                new LinkedHashMap<Class<? extends Annotation>, List<FrameworkMethod>>();
        Map<Class<? extends Annotation>, List<FrameworkField>> fieldsForAnnotations =
                new LinkedHashMap<Class<? extends Annotation>, List<FrameworkField>>();

        scanAnnotatedMembers(methodsForAnnotations, fieldsForAnnotations);

        this.methodsForAnnotations = makeDeeplyUnmodifiable(methodsForAnnotations);
        this.fieldsForAnnotations = makeDeeplyUnmodifiable(fieldsForAnnotations);
    }

    protected void scanAnnotatedMembers(Map<Class<? extends Annotation>, List<FrameworkMethod>> methodsForAnnotations, Map<Class<? extends Annotation>, List<FrameworkField>> fieldsForAnnotations) {
        for (Class<?> eachClass : getSuperClasses(clazz)) {
            for (Method eachMethod : MethodSorter.getDeclaredMethods(eachClass)) {
                addToAnnotationLists(new FrameworkMethod(eachMethod), methodsForAnnotations);
            }
            // ensuring fields are sorted to make sure that entries are inserted
            // and read from fieldForAnnotations in a deterministic order
            for (Field eachField : getSortedDeclaredFields(eachClass)) {
                addToAnnotationLists(new FrameworkField(eachField), fieldsForAnnotations);
            }
        }
    }
~~~

TestClass的主要功能就是向Runner提供clazz信息以及附带的注解信息，上文的addToAnnotationLists将对应member加入该annotation映射的member列表。下面给一个TestClass的方法列表截图，大家可以感受一下。

![TestClass方法列表](/assets/img/20160203-testClass.png)

#### FrameWorkMethod

我们先给出它的父类FrameWorkMember的定义

~~~java
public abstract class FrameworkMember<T extends FrameworkMember<T>> implements
        Annotatable {
    abstract boolean isShadowedBy(T otherMember);

    boolean isShadowedBy(List<T> members) {
        for (T each : members) {
            if (isShadowedBy(each)) {
                return true;
            }
        }
        return false;
    }

    protected abstract int getModifiers();

    /**
     * Returns true if this member is static, false if not.
     */
    public boolean isStatic() {
        return Modifier.isStatic(getModifiers());
    }

    /**
     * Returns true if this member is public, false if not.
     */
    public boolean isPublic() {
        return Modifier.isPublic(getModifiers());
    }

    public abstract String getName();

    public abstract Class<?> getType();

    public abstract Class<?> getDeclaringClass();
}

~~~

FrameWorkMethod包装了方法信息以及方法相关的注解以及一些基本的验证方法比如validatePublicVoid和是否被其他FrameWorkMethod覆盖的判断方法，除父类要求外它主要提供的信息如下：

- Annotations
- Method
- ReturnType
- ParameterTypes

#### FrameWorkField

同FrameWorkMethod差不多，FrameWorkField和它继承自同一父类，较为简单，此处就不再详细介绍了。

### 真正的执行单元——Statement

Statement是最小的执行单元，诸如RunAfter、RunWith等功能均是通过嵌套Statement来实现的，这是一个典型的装饰器模式以增加新功能。下面我们先给出Statement的定义，再给出一个嵌套的例子。

~~~java
public abstract class Statement {
    /**
     * Run the action, throwing a {@code Throwable} if anything goes wrong.
     */
    public abstract void evaluate() throws Throwable;
}
~~~

 下面以RunAfter的实现为例来说明：

 ~~~java
 public class RunAfters extends Statement {
    private final Statement next;

    private final Object target;

    private final List<FrameworkMethod> afters;

    public RunAfters(Statement next, List<FrameworkMethod> afters, Object target) {
        this.next = next;
        this.afters = afters;
        this.target = target;
    }

    @Override
    public void evaluate() throws Throwable {
        List<Throwable> errors = new ArrayList<Throwable>();
        try {
            next.evaluate();
        } catch (Throwable e) {
            errors.add(e);
        } finally {
            for (FrameworkMethod each : afters) {
                try {
                    each.invokeExplosively(target);
                } catch (Throwable e) {
                    errors.add(e);
                }
            }
        }
        MultipleFailureException.assertEmpty(errors);
    }
}
 ~~~

 可以看出新的Statement执行时会先执行旧有的Statement，再将附加上的一系列方法以target为参数运行。


### 组合方法测试的Runner实现——BlockJunitClassRunner

Junit使用虚类ParentRunner来管理复合的Runner，使用composite模式，而BlockJunitClassRunner是ParentRunner的一个子类，主要负责同一测试类多个方法的组合测试，也就是最常用的情形。我们首先还是聚焦在如何运行测试样例上。

首先看ParentRunner如何实现run方法

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
~~~

这里有个classBlock方法用来提供真正运行的Statement，下面我们看一看

~~~java
protected Statement classBlock(final RunNotifier notifier) {
        Statement statement = childrenInvoker(notifier);
        if (!areAllChildrenIgnored()) {
            statement = withBeforeClasses(statement);
            statement = withAfterClasses(statement);
            statement = withClassRules(statement);
        }
        return statement;
    }
~~~

这个过程就是先通过反射获得初始Statement，然后附加上RunBefore、RunAfter、用户自定义Rule，我们来看一下初始Statement是如何生成的。

其过程是先取得所有通过过滤器的Childeren，再使用内置的调度器来分别按顺序调用runChild方法，下面我们给出BlockJunit4ClassRunner的runChild方法

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

这里面最重要的就是RunLeaf也就是原子测试方法以及如何为单个方法生成的Statement——methodBlock，我们在下面分别给出。

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

RunLeaf的逻辑并不难，先通知Notifier测试开始，再直接调用statement的evaluate方法，最后通知Notifier测试结束。我们再来看看statement是如何生成的。

~~~java
protected Statement methodBlock(final FrameworkMethod method) {
        Object test;
        try {
            test = new ReflectiveCallable() {
                @Override
                protected Object runReflectiveCall() throws Throwable {
                    return createTest(method);
                }
            }.run();
        } catch (Throwable e) {
            return new Fail(e);
        }

        Statement statement = methodInvoker(method, test);
        statement = possiblyExpectingExceptions(method, test, statement);
        statement = withPotentialTimeout(method, test, statement);
        statement = withBefores(method, test, statement);
        statement = withAfters(method, test, statement);
        statement = withRules(method, test, statement);
        return statement;
    }

~~~

上述代码的逻辑还是比较复杂的，这里简单概述一下，首先构造测试类的实例，然后为对应method构造statement的子类InvokeMethod，然后调用FrameWorkMethod的反射运行方法，如下：

~~~java
public class InvokeMethod extends Statement {
    private final FrameworkMethod testMethod;
    private final Object target;

    public InvokeMethod(FrameworkMethod testMethod, Object target) {
        this.testMethod = testMethod;
        this.target = target;
    }

    @Override
    public void evaluate() throws Throwable {
        testMethod.invokeExplosively(target);
    }
}
~~~


### 组合类测试的Runner实现——Suite

Suite是对于ParentRunner的另一子类实现，主要用于多个测试类的情形。Suite自己维护一个runner列表，实现了getChilderen方法，其层次是在上文中提到的runChildren里，这一部分需要取出children节点然后调用runChild方法。我们着重考察suite和BlockJunit4ClassRunner在getChildren和runChild方法上的区别。Suite通过用户传入的runnerBuilder为每个类单独建立runner作为children返回，而后者则返回带Test注解的FrameWorkMethod列表。使用getChildren拿到的runner直接运行run方法。下面我们给出RunnerBuilder是如何为一系列测试类提供一系列对应的Runner，说来也简单，就是使用为单个类建立Runner的方法为每个测试类建立最后组成一个集合。但是此处需要防止递归——this builder will throw an exception if it is requested for another runner for {@code parent} before this call completes（说实话这段如何防止递归我也没看懂，有看懂的兄弟求教）。对于Suite而言，一般就是它维护一个BlockJUnit4ClassRunner列表。

~~~java
public abstract class RunnerBuilder {
    private final Set<Class<?>> parents = new HashSet<Class<?>>();

    /**
     * Override to calculate the correct runner for a test class at runtime.
     *
     * @param testClass class to be run
     * @return a Runner
     * @throws Throwable if a runner cannot be constructed
     */
    public abstract Runner runnerForClass(Class<?> testClass) throws Throwable;

    /**
     * Always returns a runner, even if it is just one that prints an error instead of running tests.
     *
     * @param testClass class to be run
     * @return a Runner
     */
    public Runner safeRunnerForClass(Class<?> testClass) {
        try {
            return runnerForClass(testClass);
        } catch (Throwable e) {
            return new ErrorReportingRunner(testClass, e);
        }
    }

    Class<?> addParent(Class<?> parent) throws InitializationError {
        if (!parents.add(parent)) {
            throw new InitializationError(String.format("class '%s' (possibly indirectly) contains itself as a SuiteClass", parent.getName()));
        }
        return parent;
    }

    void removeParent(Class<?> klass) {
        parents.remove(klass);
    }

    /**
     * Constructs and returns a list of Runners, one for each child class in
     * {@code children}.  Care is taken to avoid infinite recursion:
     * this builder will throw an exception if it is requested for another
     * runner for {@code parent} before this call completes.
     */
    public List<Runner> runners(Class<?> parent, Class<?>[] children)
            throws InitializationError {
        addParent(parent);

        try {
            return runners(children);
        } finally {
            removeParent(parent);
        }
    }

    public List<Runner> runners(Class<?> parent, List<Class<?>> children)
            throws InitializationError {
        return runners(parent, children.toArray(new Class<?>[0]));
    }

    private List<Runner> runners(Class<?>[] children) {
        List<Runner> runners = new ArrayList<Runner>();
        for (Class<?> each : children) {
            Runner childRunner = safeRunnerForClass(each);
            if (childRunner != null) {
                runners.add(childRunner);
            }
        }
        return runners;
    }
}

~~~

### 在注解中加上参数

#### BlockJUnit4ClassRunnerWithParameters

Junit使用BlockJUnit4ClassRunnerWithParameters继承BlockJUnit4ClassRunner来完成对于组合方法的带参数测试。它覆写了createTest方法和对构造器和域的验证方法。

~~~java
@Override
    public Object createTest() throws Exception {
        InjectionType injectionType = getInjectionType();
        switch (injectionType) {
            case CONSTRUCTOR:
                return createTestUsingConstructorInjection();
            case FIELD:
                return createTestUsingFieldInjection();
            default:
                throw new IllegalStateException("The injection type "
                        + injectionType + " is not supported.");
        }
    }
~~~

 
#### Parameterized

Parameterized继承了Suite，用来完成对多个类的组合测试的带参数版本。它提供三大注解——Parameters、Parameter、UseParametersRunnerFactory，前两者是用来指定参数的，后者用来指定对于每一个测试类如何生成Runner的工厂,默认工厂返回BlockJUnit4ClassRunnerWithParameters。我们下面给出内置的工厂类如何创建runner的代码。

~~~java
		private List<Runner> createRunnersForParameters(
                Iterable<Object> allParameters, String namePattern,
                ParametersRunnerFactory runnerFactory) throws Exception {
            try {
                List<TestWithParameters> tests = createTestsForParameters(
                        allParameters, namePattern);
                List<Runner> runners = new ArrayList<Runner>();
                for (TestWithParameters test : tests) {
                    runners.add(runnerFactory
                            .createRunnerForTestWithParameters(test));
                }
                return runners;
            } catch (ClassCastException e) {
                throw parametersMethodReturnedWrongType();
            }
        }

        private List<Runner> createRunners() throws Throwable {
            Parameters parameters = getParametersMethod().getAnnotation(
                    Parameters.class);
            return Collections.unmodifiableList(createRunnersForParameters(
                    allParameters(), parameters.name(),
                    getParametersRunnerFactory()));
        }
~~~


