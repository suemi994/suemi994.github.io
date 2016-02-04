---
layout: post
title: Junit源码阅读(三)之精致的Validator
category: 源码阅读
tags: junit
date: 2016-02-03
---
{% include JB/setup %}


* 目录
{:toc}

---

### 前言

在建立Runner的过程中，往往需要对当前的测试样例和注解进行验证，比如检查测试类是否含有非静态内部类，测试类是否是Public的。Junit的验证机制非常精致而优美，在本次博客中我们就主要来谈一谈Validator机制的实现。

### 指定一个验证器

首先我们可以使用注解来指定某一个用户自定义Validator来进行验证，下面给出AnnotationValidator的父类以及相应注解。

~~~java

@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface ValidateWith {
   Class<? extends AnnotationValidator> value();
}
~~~

~~~java

public abstract class AnnotationValidator {

    private static final List<Exception> NO_VALIDATION_ERRORS = emptyList();

    public List<Exception> validateAnnotatedClass(TestClass testClass) {
        return NO_VALIDATION_ERRORS;
    }

    public List<Exception> validateAnnotatedField(FrameworkField field) {
        return NO_VALIDATION_ERRORS;

    }

    public List<Exception> validateAnnotatedMethod(FrameworkMethod method) {
        return NO_VALIDATION_ERRORS;
    }
}
~~~



以上可以很清楚地看出ValidateWith注解会指定相关的用户自定义Validator。AnnotationValidator是真正执行验证操作的单元，至于这些Validator的Validate方法在何时调用，我们会在文章的最后一部分讲解。


### 对一个类进行验证

从上面的代码可以看出一个Validator需要实现三方面的验证——对类的验证、对方法的验证、对域的验证，Junit使用职责链模式来提供了三方面的验证。

首先在AnnotationsValidator中定义三个默认的Validator类，如下。

~~~java
  private static class ClassValidator extends AnnotatableValidator<TestClass> {
        @Override
        Iterable<TestClass> getAnnotatablesForTestClass(TestClass testClass) {
            return singletonList(testClass);
        }

        @Override
        List<Exception> validateAnnotatable(
                AnnotationValidator validator, TestClass testClass) {
            return validator.validateAnnotatedClass(testClass);
        }
    }

    private static class MethodValidator extends
            AnnotatableValidator<FrameworkMethod> {
        @Override
        Iterable<FrameworkMethod> getAnnotatablesForTestClass(
                TestClass testClass) {
            return testClass.getAnnotatedMethods();
        }

        @Override
        List<Exception> validateAnnotatable(
                AnnotationValidator validator, FrameworkMethod method) {
            return validator.validateAnnotatedMethod(method);
        }
    }

    private static class FieldValidator extends
            AnnotatableValidator<FrameworkField> {
        @Override
        Iterable<FrameworkField> getAnnotatablesForTestClass(TestClass testClass) {
            return testClass.getAnnotatedFields();
        }

        @Override
        List<Exception> validateAnnotatable(
                AnnotationValidator validator, FrameworkField field) {
            return validator.validateAnnotatedField(field);
        }
    }
~~~

然后依次调用这三种Validator

~~~java
private static final List<AnnotatableValidator<?>> VALIDATORS = Arrays.<AnnotatableValidator<?>>asList(
            new ClassValidator(), new MethodValidator(), new FieldValidator());

    /**
     * Validate all annotations of the specified test class that are be
     * annotated with {@link ValidateWith}.
     * 
     * @param testClass
     *            the {@link TestClass} that is validated.
     * @return the errors found by the validator.
     */
    public List<Exception> validateTestClass(TestClass testClass) {
        List<Exception> validationErrors= new ArrayList<Exception>();
        for (AnnotatableValidator<?> validator : VALIDATORS) {
            List<Exception> additionalErrors= validator
                    .validateTestClass(testClass);
            validationErrors.addAll(additionalErrors);
        }
        return validationErrors;
    }
~~~

我们可以看到ClassValidator等都继承自AnnotatableValidator，而且在真正验证的时候调用的是它们的validateTestClass方法。它们其实也并非验证的真正执行单元，它们首先找到相应TestClass的所有对应层面的注解，然后看这些注解是否是ValidateWith类型，是的话则由类的内置工厂来提供具体的AnnotationValidator。详细情况我们在下一小节中描述。



### 扩展与默认的结合——漂亮的工厂

首先我们给出AnnotationValidatorFactory的定义

~~~java
public class AnnotationValidatorFactory {
    private static final ConcurrentHashMap<ValidateWith, AnnotationValidator> VALIDATORS_FOR_ANNOTATION_TYPES =
            new ConcurrentHashMap<ValidateWith, AnnotationValidator>();

    public AnnotationValidator createAnnotationValidator(ValidateWith validateWithAnnotation) {
        AnnotationValidator validator = VALIDATORS_FOR_ANNOTATION_TYPES.get(validateWithAnnotation);
        if (validator != null) {
            return validator;
        }

        Class<? extends AnnotationValidator> clazz = validateWithAnnotation.value();
        try {
            AnnotationValidator annotationValidator = clazz.newInstance();
            VALIDATORS_FOR_ANNOTATION_TYPES.putIfAbsent(validateWithAnnotation, annotationValidator);
            return VALIDATORS_FOR_ANNOTATION_TYPES.get(validateWithAnnotation);
        } catch (Exception e) {
            throw new RuntimeException("Exception received when creating AnnotationValidator class " + clazz.getName(), e);
        }
    }

}
~~~

工厂通过一个线程安全的Map存储注解和对应的实际AnnotationValidator实例，而AnnotableValidator内置通过内置一个工厂来存储所有对应层级的验证器实例，并调用这些验证器对应层级的验证方法来返回可能的异常，我们下面贴出该内部类的代码：

~~~java
private static abstract class AnnotatableValidator<T extends Annotatable> {
        private static final AnnotationValidatorFactory ANNOTATION_VALIDATOR_FACTORY = new AnnotationValidatorFactory();

        abstract Iterable<T> getAnnotatablesForTestClass(TestClass testClass);

        abstract List<Exception> validateAnnotatable(
                AnnotationValidator validator, T annotatable);

        public List<Exception> validateTestClass(TestClass testClass) {
            List<Exception> validationErrors= new ArrayList<Exception>();
            for (T annotatable : getAnnotatablesForTestClass(testClass)) {
                List<Exception> additionalErrors= validateAnnotatable(annotatable);
                validationErrors.addAll(additionalErrors);
            }
            return validationErrors;
        }

        private List<Exception> validateAnnotatable(T annotatable) {
            List<Exception> validationErrors= new ArrayList<Exception>();
            for (Annotation annotation : annotatable.getAnnotations()) {
                Class<? extends Annotation> annotationType = annotation
                        .annotationType();
                ValidateWith validateWith = annotationType
                        .getAnnotation(ValidateWith.class);
                if (validateWith != null) {
                    AnnotationValidator annotationValidator = ANNOTATION_VALIDATOR_FACTORY
                            .createAnnotationValidator(validateWith);
                    List<Exception> errors= validateAnnotatable(
                            annotationValidator, annotatable);
                    validationErrors.addAll(errors);
                }
            }
            return validationErrors;
        }
    }

~~~


可以说这个带工厂的内部类是一处神来之笔，整个流程是AnnotationsValidator类的validateTestClass方法依次调用职责链中三个层级的AnnotatableValidator，它们先找出所有对应层次上的注解，再滤掉那些不是ValidateWith类型的注解，然后通过一个工厂来维护所有验证器实例，调用这些实例来真正验证。因为对于不同的TestClass，我们在一个层面上只用维护一个工厂。使用内部类，只暴露应该暴露的，扩展的用户只应扩展自定义的Validator，不应该在层次逻辑上进行扩展，不应该在整体验证的AnnotationsValidator之外再使用任何单独层次的AnnotatableValidator。


