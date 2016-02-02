---
layout: post
title: Junit源码阅读(一)
category: 源码阅读
tags: junit
date: 2016-02-02
---
{% include JB/setup %}


* 目录
{:toc}

---

### Junit的工程结构

从上图可以清楚的看出Junit大致分为几个版块，接下来一一简略介绍这些版块的作用。

![Junit工程结构](/assets/img/20160202-struct.png)

- runner:定义了Junit模型中的许多基本概念，只要是一些虚类和接口，是整个Junit工程的基石
- runners:提供了从注解中使用反射完成测试用例执行的实现
- interval:提供了在Runner中许多虚类的默认实现，包括各类RunnerBuilder如基于注解提供Builder、各类Matchers、各类Request如类测试Request、方法测试Request。
- rules:用于用户自定义扩展，规定对于不同statement的用户自定义行为
- matchers:主要用于AssertThat方法，在4.4之后废弃
- validator:从类、方法和域多个方面检查测试样例
- experimental:一些测试特性


### Runner中的基本概念

- Runner:以Notifier为参数允许测试样例，运行过程中的Notifier负责监视测试过程
- Request:负责记录测试样例的Description信息，同事负责提供对应的Runner。可以通过Computer结合指定或默认的RunnerBuilder来直接为一系列测试类统一提供Runner
- Description:描述测试样例，使用Composite模式，组合多个样例
- Result:记录异常和失败，内置一个Listener来实现与测试过程的同步，测试完成时count自增，有样例失败则加入Failure列表
- Failure:将断言失败和抛出异常综合在同一个框架下，同时提供了Description的信息
- Listener:监视测试过程,典型的观察者模式
- Notifier:管理一系列Listener，保证线程安全
- Filter:指定条件，只运行符合条件的测试样例，可以动态添加，为每次测试增加了灵活性


### Runner中的依赖关系

![Runner中的依赖示意图](/assets/img/20160202-diagram.png)

JunitCore负责提供给用户统一的交互，从命令行运行测试样例。Notifier是一个虚类，子类需要实现如何通知Listener的方法，负责管理Listener集合，内部内置了一个静态的SafeNotifier，该类提供了一个run方法，来简单依次通知所有Listener，它用来实现在测试开始和失败出现的时候通知所有Listener。

Description是对测试样例的建模，用来组合多个测试样例，是Runner中的核心内容。

Filter也是一个虚类，子类应该实现shouldRun方法来决定对于Description是否运行。同时依然实现一个静态方法来提供什么都不过滤，以及一个判断原子描述是否等于期望描述的过滤器，对于非原子描述若其子描述均不等于期望描述则滤掉。如下列代码所示：

~~~java
/**
     * Returns a {@code Filter} that only runs the single method described by
     * {@code desiredDescription}
     */
    public static Filter matchMethodDescription(final Description desiredDescription) {
        return new Filter() {
            @Override
            public boolean shouldRun(Description description) {
                if (description.isTest()) {
                    return desiredDescription.equals(description);
                }

                // explicitly check if any children want to run
                for (Description each : description.getChildren()) {
                    if (shouldRun(each)) {
                        return true;
                    }
                }
                return false;
            }

            @Override
            public String describe() {
                return String.format("Method %s", desiredDescription.getDisplayName());
            }
        };
    }

~~~

Failure组合了Description和Throwable，为运行时异常和断言错误屏蔽了不一致的方面，可以向上提供错误信息和样例信息。

Request负责提供具体的Runner来run对应的测试样例，同时是Filter作用的主体，对于Filter，会返回一个新的FilterRequest，代码如下：

~~~java
package org.junit.internal.requests;

import org.junit.internal.runners.ErrorReportingRunner;
import org.junit.runner.Request;
import org.junit.runner.Runner;
import org.junit.runner.manipulation.Filter;
import org.junit.runner.manipulation.NoTestsRemainException;

/**
 * A filtered {@link Request}.
 */
public final class FilterRequest extends Request {
    private final Request request;
    /*
     * We have to use the f prefix, because IntelliJ's JUnit4IdeaTestRunner uses
     * reflection to access this field. See
     * https://github.com/junit-team/junit/issues/960
     */
    private final Filter fFilter;

    /**
     * Creates a filtered Request
     *
     * @param request a {@link Request} describing your Tests
     * @param filter {@link Filter} to apply to the Tests described in
     * <code>request</code>
     */
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



### Runner中的JunitCore

![JunitCore运行的时序图](/assets/img/20160202-timeline.png)

JUnitCore使用外观模式（facade），对外提供一致的界面，同时支持运行JUnit 4或JUnit 3.8.x用例，通过命令行执行用例.首先它使用jUnitCommandLineParseResult解析外部参数，将默认的TextListener加入内置的Notifier。它所运行的Runner也是由jUnitCommandLineParseResult提供的，先行通过测试filter掉不需要的样例，最后调用Runner的run方法。



