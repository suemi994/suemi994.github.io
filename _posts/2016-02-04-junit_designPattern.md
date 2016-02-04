---
layout: post
title: Junit源码阅读(六)之Junit中的设计模式
category: 源码阅读
tags: junit
date: 2016-02-04
---
{% include JB/setup %}


* 目录
{:toc}

---

### 前言

在这次的博客中我们将着重于Junit的许多集成性功能来讨论Junit中的种种设计模式。可以说Junit的实现本身就是GOF设计原则的范例教本，下面就让我们开始吧。

### 装饰器模式

装饰器模式是为了在原有功能上加入新功能，在Junit中绝对属于使用最频繁架构中最核心的模式，Runner、Filter、Rule等都是通过装饰器模式来完成扩展的。下就以Filter的实现机制来说明。

首先从命令行中解析出来的FilterSpec生成各种Filter，如下：

~~~java
	private Request applyFilterSpecs(Request request) {
        try {
            for (String filterSpec : filterSpecs) {
                Filter filter = FilterFactories.createFilterFromFilterSpec(
                        request, filterSpec);
                request = request.filterWith(filter);
            }
            return request;
        } catch (FilterNotCreatedException e) {
            return errorReport(e);
        }
    }
~~~

request的Filter方法将会返回一个新的Request，不过是Request的子类FilterRequest，它依然保留原有的request，只不过在返回Runner的时候，再采用Filter过滤，如下：

~~~java
public final class FilterRequest extends Request {
    private final Request request;
   
    private final Filter fFilter;

    
    public FilterRequest(Request request, Filter filter) {
        this.request = request;
        this.fFilter = filter;
    }

    @Override
    public Runner getRunner() {
        try {
            Runner runner = request.getRunner();
            fFilter.apply(runner);
            return runner;
        } catch (NoTestsRemainException e) {
            return new ErrorReportingRunner(Filter.class, new Exception(String
                    .format("No tests found matching %s from %s", fFilter
                            .describe(), request.toString())));
        }
    }
}
~~~

Filter的apply方法会调用runner自身实现的filter方法并以自己作为参数，以ParentRunner为例，我们给出filter方法的一般实现。

~~~java
public void filter(Filter filter) throws NoTestsRemainException {
        synchronized (childrenLock) {
            List<T> children = new ArrayList<T>(getFilteredChildren());
            for (Iterator<T> iter = children.iterator(); iter.hasNext(); ) {
                T each = iter.next();
                if (shouldRun(filter, each)) {
                    try {
                        filter.apply(each);
                    } catch (NoTestsRemainException e) {
                        iter.remove();
                    }
                } else {
                    iter.remove();
                }
            }
            filteredChildren = Collections.unmodifiableCollection(children);
            if (filteredChildren.isEmpty()) {
                throw new NoTestsRemainException();
            }
        }
    }

~~~

显然非原子的Runner通过维护一个filteredChildren列表来提供它所有通过过滤的child，每次有新的filter作用到之上，它都需要更新该列表。对于原子的测试，它会提供出它的description，由Filter实现的shouldRun方法来判断是否会被过滤，这也是filteredChildren列表更新的原理。

当我们先后有多个Filter时，可以不停地包装已有的FilterRequest，每个FilterRequest在getRunnere时都会先调用其内部的Request，然后执行相同的附加操作，也即更新内部Request返回的Runner的filteredChildren列表。使用实现同一接口继承同一父类的各个过滤器相互嵌套就可以实现一个过滤链。


### 工厂模式

工厂模式也在Junit中被大量使用，主要用来生产各类Rule、Filter等。我们依然以Filter机制为例来介绍一次抽象工厂的使用。FilterFactory是一个工厂接口如下：

~~~java
public interface FilterFactory {
    
    Filter createFilter(FilterFactoryParams params) throws FilterNotCreatedException;

    
    @SuppressWarnings("serial")
    class FilterNotCreatedException extends Exception {
        public FilterNotCreatedException(Exception exception) {
            super(exception.getMessage(), exception);
        }
    }
}
~~~

FilterFactories提供了各种由参数来生成不同工厂的方法，同时又使用生成的工厂来生成Filter，可以说在广义上这就是一个抽血工厂模式，不过FilterFactories完成了生产产品的过程，又集成了提供各类工厂的方法。下面给出代码大家感受一下：

~~~java
class FilterFactories {
    
    public static Filter createFilterFromFilterSpec(Request request, String filterSpec)
            throws FilterFactory.FilterNotCreatedException {
        Description topLevelDescription = request.getRunner().getDescription();
        String[] tuple;

        if (filterSpec.contains("=")) {
            tuple = filterSpec.split("=", 2);
        } else {
            tuple = new String[]{ filterSpec, "" };
        }

        return createFilter(tuple[0], new FilterFactoryParams(topLevelDescription, tuple[1]));
    }

    
    public static Filter createFilter(String filterFactoryFqcn, FilterFactoryParams params)
            throws FilterFactory.FilterNotCreatedException {
        FilterFactory filterFactory = createFilterFactory(filterFactoryFqcn);

        return filterFactory.createFilter(params);
    }

    
    public static Filter createFilter(Class<? extends FilterFactory> filterFactoryClass, FilterFactoryParams params)
            throws FilterFactory.FilterNotCreatedException {
        FilterFactory filterFactory = createFilterFactory(filterFactoryClass);

        return filterFactory.createFilter(params);
    }

    static FilterFactory createFilterFactory(String filterFactoryFqcn) throws FilterNotCreatedException {
        Class<? extends FilterFactory> filterFactoryClass;

        try {
            filterFactoryClass = Classes.getClass(filterFactoryFqcn).asSubclass(FilterFactory.class);
        } catch (Exception e) {
            throw new FilterNotCreatedException(e);
        }

        return createFilterFactory(filterFactoryClass);
    }

    static FilterFactory createFilterFactory(Class<? extends FilterFactory> filterFactoryClass)
            throws FilterNotCreatedException {
        try {
            return filterFactoryClass.getConstructor().newInstance();
        } catch (Exception e) {
            throw new FilterNotCreatedException(e);
        }
    }
}

~~~



### 组合模式

在Junit中组合模式主要用于管理Runner和Description的组合。由于测试的需求十分多样，有时需要测试单个方法，有时需要测试单个类的所有方法，有时又需要测试多个类的组合方法，所以对于测试的表达必须是强大的。组合模式显然符合这个要求，根部的runner会调动子节点的run，向上只暴露出根节点的run方法。由于之前几篇中对这一部分论述较多，此处就不再赘述了。

### 观察者模式

最典型的就是Notifier和Listener，当测试开始、结束、出现错误时，Notifier将通知它管理的Listener执行相应的操作，但有趣之处就在于为了能够处理在通知过程中出现的异常，Notifer使用了一个内部类SafeNotifier，所有的对应事件（测试开始等）覆写SafeNotifier里的notifyListener函数，在其中写调用Listener的具体哪一个函数。相关的详细内容请参考上一篇博客，下面给出RunNotifier的源码(删去部分内容)。

~~~java
public class RunNotifier {
    private final List<RunListener> listeners = new CopyOnWriteArrayList<RunListener>();
    private volatile boolean pleaseStop = false;

    public void addListener(RunListener listener) {
        if (listener == null) {
            throw new NullPointerException("Cannot add a null listener");
        }
        listeners.add(wrapIfNotThreadSafe(listener));
    }

    
    public void removeListener(RunListener listener) {
        if (listener == null) {
            throw new NullPointerException("Cannot remove a null listener");
        }
        listeners.remove(wrapIfNotThreadSafe(listener));
    }

    RunListener wrapIfNotThreadSafe(RunListener listener) {
        return listener.getClass().isAnnotationPresent(RunListener.ThreadSafe.class) ?
                listener : new SynchronizedRunListener(listener, this);
    }


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
    
    public void fireTestFinished(final Description description) {
        new SafeNotifier() {
            @Override
            protected void notifyListener(RunListener each) throws Exception {
                each.testFinished(description);
            }
        }.run();
    }

    
}

~~~

### 职责链模式

可以参考之前的第三篇博客，Junit的Validator机制需要在三个层次——类、方法和域上进行验证，采用了职责链模式。








