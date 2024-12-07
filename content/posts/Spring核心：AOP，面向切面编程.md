---
author: ["柿子"]
title: '5. Spring 核心：AOP，面向切面编程'
date: '2022-10-08 20:01:00'
summary: ""
tags: [""]
typora-root-url: ../../static
typora-copy-images-to: ../../static/images
---

> 本文内容是逐段翻译[Spring-Framework 1.1.x 官方文档的第5章](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html)的内容。通过在翻译阅读过程中去尝试感受当年作者的意境。为什么不是1.0？因为1.1.x才开始有官方文档，之前的版本只有api文档。
>
> *如果你想知道人为什么要这么搞，那么应该去看书/文档；如果你要知道让机器干了什么？那你应该看代码！——左耳朵耗子*

## 5.1. [概念](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-introduction)

面向方面编程（AOP）提供了另一种思考程序结构的方法，是对 OOP 的补充。OO 将应用程序分解为对象的层次结构，而 AOP 则将程序分解为方面或关注点。这样就能将事务管理等关注点模块化，否则这些关注点就会跨越多个对象。(这类问题通常被称为交叉问题）。

Spring 中使用了 AOP：

- 提供声明式企业服务，尤其是作为 EJB 声明式服务的替代。其中最重要的服务是声明式事务管理，它建立在 Spring 的事务抽象之上。
- 允许用户实现自定义 AOP，用 AOP 补充他们对 OOP 的使用。

因此，您可以将 Spring AOP 视为一种使能技术，它允许 Spring 在不使用 EJB 的情况下提供声明式事务管理；或者使用 Spring AOP 框架的全部功能来实现自定义方面。

如果你只对**通用声明式服务**或**其他预打包的声明式中间件服务**（如池化服务）感兴趣，就不需要直接使用 Spring AOP，也可以跳过本章的大部分内容。

### 5.1.1. [AOP 概念](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-introduction-defn)

首先，让我们定义一些 AOP 核心概念。这些术语并非 Spring 特有。遗憾的是，AOP 术语并不特别直观。不过，如果 Spring 使用自己的术语，那就更令人困惑了。

- **切面(Aspect)**：模块化的关注点，否则其实现可能会跨越多个对象。在 J2EE 应用程序中，事务管理就是横切关注点的一个很好的例子。方面使用 Spring 作为顾问或拦截器来实现。
- **连接点(Joinpoint)**：程序执行过程中的点，如方法调用或抛出的特定异常。在 Spring AOP 中，连接点总是方法调用。Spring 并未在显著位置使用连接点一词；连接点信息可通过传递给拦截器的 MethodInvocation 参数上的方法访问，并通过 org.springframework.aop.Pointcut 接口的实现进行评估。
- **通知(Advice)**：AOP 框架在特定连接点采取的行动。不同类型的通知包括 "around"、"before"、"throws" 通知。下面将讨论通知类型。包括 Spring 在内的许多 AOP 框架都将通知建模为拦截器，并在连接点 "around"维护一连串拦截器。
- **切点(Pointcut)**：一组连接点，指定何时应触发通知。AOP 框架必须允许开发人员指定切点：例如，使用正则表达式。
- **引介(Introduction)**：为通知类添加方法或字段。Spring 允许你为任何通知对象引介新的接口。例如，你可以使用导言让任何对象实现 IsModified 接口，以简化缓存。
- **目标对象(Target object)**：包含连接点的对象。也称为通知对象或代理对象。
- **AOP 代理(AOP proxy)**：由 AOP 框架创建的对象，包括通知。在 Spring 中，AOP 代理将是 JDK 动态代理或 CGLIB 代理。
- **编织(Weaving)**：组装各个方面以创建通知对象。这可以在编译时完成（例如使用 AspectJ 编译器），也可以在运行时完成。Spring 和其他纯 Java AOP 框架一样，在运行时执行编织。

不同的通知类型包括：

- **环绕通知(Around advice)**：围绕连接点（如方法调用）的通知。这是最强大的一种通知。环绕通知将在方法调用前后执行自定义行为。它们负责选择是继续执行连接点，还是通过返回自己的返回值或抛出异常来快捷执行。
- **前置通知(Before advice)**：在连接点之前执行的通知，但不能阻止执行流进入连接点（除非抛出异常）。
- **异常通知(Throws advice)**：在方法抛出异常时执行的通知。Spring 提供了强类型的抛出通知，因此您可以编写代码来捕获您感兴趣的异常（和子类），而无需从 Throwable 或 Exception 进行转换。
- **后置通知(After returning advice)**：在连接点正常完成后执行的通知：例如，如果方法返回时没有抛出异常。

环绕通知是最通用的通知。大多数基于拦截的 AOP 框架（如 Nanning Aspects）只提供围绕通知。

与 AspectJ 一样，Spring 也提供了各种通知类型，因此我们通知您使用功能最少的通知类型来实现所需的行为。例如，如果您只需要用一个方法的返回值更新缓存，那么您最好使用 after returning advice，而不是 around advice，尽管 around advice 也能实现相同的功能。

使用最特殊的通知类型可以提供更简单的编程模型，减少出错的可能性。例如，您不需要在用于环绕通知的 MethodInvocation 上调用 proceed() 方法，因此也不会调用失败。

切点概念是 AOP 的关键所在，它将 AOP 与提供拦截功能的旧技术区分开来。切点分使通知的目标不受 OO 层次结构的影响。例如，提供声明式事务管理的**环绕通知**可应用于跨越多个对象的一组方法。因此，快捷方式提供了 AOP 的结构元素。

### 5.1.2. [Spring AOP 的功能和目标](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-introduction-spring-defn)

Spring AOP 是用纯 Java 实现的。不需要特殊的编译过程。Spring AOP 无需控制类加载器的层次结构，因此适合在 J2EE 网络容器或应用服务器中使用。

Spring 目前支持方法调用的拦截。虽然可以在不破坏 Spring AOP 核心 API 的情况下添加对字段拦截的支持，但并未实现字段拦截。

字段拦截可以说违反了 OO 封装。我们认为这在应用程序开发中是不明智的。如果需要字段拦截，请考虑使用 AspectJ。

Spring 提供了表示指向和不同通知类型的类。Spring 使用 advisor 这个术语来表示代表一个切面的对象，包括通知和针对特定连接点的点切入。

不同的通知类型包括 MethodInterceptor（来自 AOP 联盟拦截 API）和 org.springframework.aop 包中定义的通知接口。所有通知都必须实现 org.aopalliance.aop.Advice 标签接口。开箱即支持的通知包括 MethodInterceptor ；ThrowsAdvice ；BeforeAdvice 和 AfterReturningAdvice。下面我们将详细讨论通知类型。

Spring 实现了 AOP 联盟拦截接口 (http://www.sourceforge.net/projects/aopalliance)。环绕通知必须实现 AOP 联盟 org.aopalliance.intercept.MethodInterceptor 接口。该接口的实现可在 Spring 或任何其他符合 AOP 联盟的实现中运行。目前，JAC 实现了 AOP 联盟接口，Nanning 和 Dynaop 也可能在 2004 年初实现。

Spring 的 AOP 方法与大多数其他 AOP 框架不同。其目的不是提供最完整的 AOP 实现（尽管 Spring AOP 的能力很强），而是提供 AOP 实现与 Spring IoC 之间的紧密集成，以帮助解决企业应用中的常见问题。

因此，举例来说，Spring 的 AOP 功能通常与 Spring IoC 容器结合使用。AOP 通知使用普通的 bean 定义语法指定（尽管这允许强大的 "自动代理" 功能）；通知和指向本身由 Spring IoC 管理：这是与其他 AOP 实现的一个重要区别。有些事情是 Spring AOP 无法轻松或高效完成的，例如向非常细粒度的对象提供通知。在这种情况下，AspectJ 可能是最佳选择。不过，根据我们的经验，Spring AOP 可以很好地解决 J2EE 应用程序中适合使用 AOP 的大多数问题。

Spring AOP 绝不会为了提供全面的 AOP 解决方案而与 AspectJ 或 AspectWerkz 竞争。我们认为，Spring 等基于代理的框架和 AspectJ 等全面的框架都很有价值，它们是互补的，而不是竞争的。因此，Spring 1.1 的首要任务是将 Spring AOP 和 IoC 与 AspectJ 无缝集成，以便在基于 Spring 的一致应用架构中满足 AOP 的所有用途。这种集成不会影响 Spring AOP API 或 AOP Alliance API；Spring AOP 将保持向后兼容。

### 5.1.3. [Spring 中的 AOP 代理](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-introduction-proxies)

Spring 默认将 J2SE 动态代理（JDK代理）用于 AOP 代理。这样就可以代理任何接口或接口集。

Spring 还可以使用 CGLIB 代理。这是代理类而非接口所必需的。如果业务对象没有实现接口，则默认使用 CGLIB。由于针对接口而非类编程是一种良好的做法，因此业务对象通常会实现一个或多个业务接口。

可以强制使用 CGLIB：我们将在下文讨论，并解释为什么要这样做。

在 Spring 1.0 之后，Spring 可能会提供更多类型的 AOP 代理，包括完全生成的类。这不会影响编程模型。

## 5.2. [Spring 中的切点（Pointcut）](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-introduction-pointcuts)

让我们看看 Spring 是如何处理关键的点切概念的。

### 5.2.1[. 概念](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e2905)

Spring 的快捷方式模型实现了与通知类型无关的快捷方式重用。使用相同的快捷方式可以针对不同的通知。

`org.springframework.aop.Pointcut`接口是中心接口，用于向特定类和方法提供通知。完整的接口如下所示：

```java
public interface Pointcut {

    ClassFilter getClassFilter();

    MethodMatcher getMethodMatcher();

}
```

将 Pointcut 接口拆分为两部分，可以重复使用类和方法匹配部分，以及细粒度的组合操作（例如与另一个方法匹配器执行 "union"）。

ClassFilter 接口用于将点切限制在一组给定的目标类上。如果 matches() 方法总是返回 true，那么所有目标类都将被匹配：

```java
public interface ClassFilter {

    boolean matches(Class clazz);
}
```

MethodMatcher 接口通常更为重要。完整的接口如下所示：

```java
public interface MethodMatcher {

    boolean matches(Method m, Class targetClass);

    boolean isRuntime();

    boolean matches(Method m, Class targetClass, Object[] args);
}
```

matches(Method, Class) 方法用于测试该快捷方式是否与目标类上的给定方法相匹配。该评估可在创建 AOP 代理时执行，以避免每次调用方法时都要进行测试。如果给定方法的 2 参数匹配方法返回 true，且 MethodMatcher 的 isRuntime() 方法返回 true，那么每次方法调用时都会调用 3 参数匹配方法。这样，在执行目标通知之前，点切口就能立即查看传递给方法调用的参数。

大多数方法匹配器都是静态的，这意味着它们的 isRuntime() 方法会返回 false。在这种情况下，3 参数匹配方法永远不会被调用。

如果可能的话，尽量让点切分成为静态的，让 AOP 框架在创建 AOP 代理时缓存点切分的评估结果。

### 5.2.2. [切点操作](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e2954)

Spring 支持点切分操作：尤其是联合和相交。

- 联合是指任一指向匹配的方法。

- 交集是指两个点切入相匹配的方法。

- 联合通常更有用。

可以使用`org.springframework.aop.support.Pointcuts`类中的静态方法，或使用同一软件包中的 ComposablePointcut 类来组成快捷方式。

### 5.2.3. [方便的切点实现](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e2979)

Spring 提供了几种方便的点切实现。有些可以开箱即用，有些则需要在特定于应用程序的点切入中进行子类化。

#### 5.2.3.1. [静态切点](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e2984)

静态快捷方式基于方法和目标类，不能考虑方法的参数。静态快捷方式对于大多数应用来说都是足够的，也是最好的。Spring 只需在首次调用方法时评估一次静态快捷方式即可：此后，每次调用方法时都无需再次评估快捷方式。

让我们来看看 Spring 包含的一些静态点切实现。

##### 5.2.3.1.1. [正则表达式关键字](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e2991)

正则表达式是实现特定静态快捷方式的一种显而易见的方法。org.springframework.aop.support.RegexpMethodPointcut 是一种使用 Perl 5 正则表达式语法的通用正则表达式快捷方式。

使用该类，您可以提供一系列模式字符串。如果其中任何一个匹配，则点切线的值将为 true（因此结果实际上就是这些点切线的组合）。

使用方法如下：

```java
<bean id="settersAndAbsquatulatePointcut" 
    class="org.springframework.aop.support.RegexpMethodPointcut">
    <property name="patterns">
        <list>
            <value>.*get.*</value>
            <value>.*absquatulate</value>
        </list>
    </property>
</bean>
```

RegexpMethodPointcut 的方便子类 RegexpMethodPointcutAdvisor 也允许我们引用通知。(请记住，Advice 可以是拦截器、before advice、throws advice 等）这简化了布线，因为一个 bean 既是快捷方式，又是顾问，如下所示：

```java
<bean id="settersAndAbsquatulateAdvisor" 
    class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
    <property name="advice">
        <ref local="beanNameOfAopAllianceInterceptor"/>
    </property>
    <property name="patterns">
        <list>
            <value>.*get.*</value>
            <value>.*absquatulate</value>
        </list>
    </property>
</bean>
```

RegexpMethodPointcutAdvisor 可用于任何通知类型。

RegexpMethodPointcut 类需要 Jakarta ORO 正则表达式包。

##### 5.2.3.1.2. [属性驱动的指向](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3023)

静态快捷方式的一个重要类型是元数据驱动快捷方式。它使用元数据属性的值：通常是源代码级元数据。

#### 5.2.3.2. [动态切点](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3031)

与静态快捷方式相比，动态快捷方式的评估成本更高。它们会考虑方法参数以及静态信息。这意味着每次调用方法时都必须对其进行评估；由于参数会发生变化，因此无法缓存评估结果。

主要的例子是控制流快捷方式。

##### 5.2.3.2.1. [控制流指向](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3044)

Spring 控制流快捷方式在概念上类似于 AspectJ cflow 快捷方式，但功能没那么强大。(目前还无法指定一个快捷方式在另一个快捷方式下面执行）。控制流快捷方式与当前的调用堆栈相匹配。例如，如果连接点被 `com.mycompany.web` 包中的方法或 `SomeCaller` 类调用，它就会触发。控制流快捷方式使用 `org.springframework.aop.support.ControlFlowPointcut` 类指定。

> 注意：在运行时对控制流指向进行评估的成本甚至要比其他动态指向高得多。在 Java 1.4 中，成本约为其他动态指向式的 5 倍；在 Java 1.3 中则超过 10 倍。

### 5.2.4. [切点的超类](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3064)

Spring 提供了有用的快捷方式超类，帮助你实现自己的快捷方式。

由于静态快捷方式最为有用，因此您可能会如下图所示对 StaticMethodMatcherPointcut 进行子类化。这只需要实现一个抽象方法（当然也可以覆盖其他方法来定制行为）：

```java
class TestStaticPointcut extends StaticMethodMatcherPointcut {

    public boolean matches(Method m, Class targetClass) {
        // return true if custom criteria match
    }
}
```

此外，还有用于动态插入点的超类。

在 Spring 1.0 RC2 及以上版本中，您可以在任何通知类型中使用自定义点切入。

### 5.2.5. [自定义切点](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3077)

由于 Spring 中的切点是 Java 类，而不是语言特性（如 AspectJ），因此可以声明自定义切点，无论是静态还是动态切点。不过，Spring 并不支持用 AspectJ 语法编码的复杂点切分表达式。不过，Spring 中的自定义快捷方式可以任意复杂。

Spring 的后续版本可能会支持 JAC 提供的 "语义指向"：例如，"改变目标对象中实例变量的所有方法"。

## 5.3. [Spring 中的通知类型](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3084)

现在让我们看看 Spring AOP 是如何处理建通知的。

### 5.3.1. [通知生命周期](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3089)

Spring 通知可以在所有通知对象中共享，也可以为每个通知对象独有。这相当于按类或按实例提供通知。

最常用的是按类通知。它适用于通用通知，如事务顾问。这些通知不依赖于被代理对象的状态，也不添加新的状态；它们只是作用于方法和参数。

每实例通知适用于引介，以支持混合体。在这种情况下，通知会为代理对象添加状态。

在同一个 AOP 代理中，可以混合使用共享通知和按实例通知。

### 5.3.2. [Spring 中的通知类型](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-introduction-advice-types)

Spring 提供了几种开箱即用的通知类型，并可扩展以支持任意通知类型。让我们来看看基本概念和标准通知类型。

#### 5.3.2.1. [Around 通知的拦截](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3111)

Spring 中最基本的通知类型是围绕通知的拦截。

Spring 符合 AOP 联盟关于使用方法拦截的`环绕通知`的接口。实现`环绕通知`的方法拦截器应实现以下接口：

```java
public interface MethodInterceptor extends Interceptor {
  
    Object invoke(MethodInvocation invocation) throws Throwable;
}
```

invoke() 方法的 MethodInvocation 参数公开了被调用的方法、目标连结点、AOP 代理以及方法的参数。invoke() 方法应返回调用的结果：连接点的返回值。

一个简单的 MethodInterceptor 实现如下：

```java
public class DebugInterceptor implements MethodInterceptor {

    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("Before: invocation=[" + invocation + "]");
        Object rval = invocation.proceed();
        System.out.println("Invocation returned");
        return rval;
    }
}
```

注意对 MethodInvocation 的 proceed() 方法的调用。该方法在拦截器链中向下进行，直到连接点。大多数拦截器都会调用此方法，并返回其返回值。然而，MethodInterceptor 与其他拦截器一样，可以返回不同的值或抛出异常，而不是调用 proceed 方法。但是，如果没有充分的理由，您不希望这样做！

MethodInterceptors 提供了与其他 AOP 联盟兼容的 AOP 实现的互操作性。本节余下部分讨论的其他通知类型以 Spring 特有的方式实现了常见的 AOP 概念。虽然使用最具体的通知类型有其优势，但如果您可能希望在其他 AOP 框架中运行该方面，请坚持`环绕通知`使用 MethodInterceptor。请注意，目前各框架之间还不能互操作，而且 AOP 联盟目前也没有定义点切接口。

#### 5.3.2.2. [Before 通知](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3148)

更简单的通知类型是`Before 通知`。它不需要 MethodInvocation 对象，因为它只会在进入方法之前被调用。

`Before 通知`的主要优点是不需要调用 proceed() 方法，因此也就不会在无意中导致拦截器链无法继续。

MethodBeforeAdvice 接口如下所示。(*Spring框架的API设计理论上支持在字段操作之前提供“前置通知”（field before advice），也就是说，可以在字段被访问或修改之前执行一些自定义的逻辑。然而，通常情况下，Spring框架主要用于处理方法的拦截（field interception），而不是字段的拦截。因此，尽管技术上Spring的API允许这种设计，但实际上Spring框架不太可能实现这种字段级别的前置通知功能。简而言之，Spring的API设计上是支持字段级别的前置通知的，但实际上Spring更倾向于处理方法级别的拦截，并且不太可能去实现字段级别的拦截功能*）。

```java
public interface MethodBeforeAdvice extends BeforeAdvice {

    void before(Method m, Object[] args, Object target) throws Throwable;
}
```

请注意，返回类型是 void。`Before 通知`可以在连接点执行前插入自定义行为，但不能更改返回值。如果 before 通知抛出异常，将中止拦截器链的进一步执行。异常会沿着拦截器链向上传播。如果该异常未被选中，或在被调用方法的签名上，它将直接传递给客户端；否则，它将被 AOP 代理封装在一个未被选中的异常中。

在 Spring 中，before 通知是一个示例，它会统计所有正常返回的方法：

```java
public class CountingBeforeAdvice implements MethodBeforeAdvice {
    private int count;
    public void before(Method m, Object[] args, Object target) throws Throwable {
        ++count;
    }

    public int getCount() { 
        return count; 
    }
}
```

`Before 通知`可用于任何快捷方式。

#### 5.3.2.3. [Throws 通知](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3182)

如果连接点抛出异常，则在连接点返回后调用 Throws 通知。Spring 提供类型化的抛出通知。请注意，这意味着 org.springframework.aop.ThrowsAdvice 接口不包含任何方法：它是一个标签接口，用于标识给定对象实现了一个或多个类型化抛出提示方法。这些方法应为

```java
afterThrowing([Method], [args], [target], subclassOfThrowable) 
```

只需要最后一个参数。因此，根据通知方法是否对方法和参数感兴趣，可以有一到四个参数。以下是抛出通知的示例。

如果抛出 RemoteException（包括子类），将调用此通知：

```java
public  class RemoteThrowsAdvice implements ThrowsAdvice {

    public void afterThrowing(RemoteException ex) throws Throwable {
        // Do something with remote exception
    }
}
```

如果抛出 ServletException，将调用以下通知。与上述通知不同的是，它声明了 4 个参数，因此可以访问调用的方法、方法参数和目标对象：

```java
public static class ServletThrowsAdviceWithArguments implements ThrowsAdvice {

    public void afterThrowing(Method m, Object[] args, Object target, ServletException ex) {
        // Do something will all arguments
    }
}
```

最后一个示例说明了如何在处理 RemoteException 和 ServletException 的单个类中使用这两种方法。在一个类中可以组合任意数量的抛出通知方法。

```java
public static class CombinedThrowsAdvice implements ThrowsAdvice {

    public void afterThrowing(RemoteException ex) throws Throwable {
        // Do something with remote exception
    }
 
    public void afterThrowing(Method m, Object[] args, Object target, ServletException ex) {
        // Do something will all arguments
    }
}
```

抛出通知可与任何切点一起使用。

#### 5.3.2.4. [After Returning 通知](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3222)

Spring 中的返回后通知必须实现`org.springframework.aop.AfterReturningAdvice`接口，如下所示：

```java
public interface AfterReturningAdvice extends Advice {

    void afterReturning(Object returnValue, Method m, Object[] args, Object target) 
            throws Throwable;
}
```

返回后通知可以访问返回值（不能修改）、调用方法、方法参数和目标。

下面的返回通知会统计所有未抛出异常的成功方法调用次数：

```java
public class CountingAfterReturningAdvice implements AfterReturningAdvice {
    private int count;

    public void afterReturning(Object returnValue, Method m, Object[] args, Object target) throws Throwable {
        ++count;
    }

    public int getCount() {
        return count;
    }
}
```

这个通知不会改变执行路径。如果出现异常，异常将被抛向拦截器链，而不是返回值。

返回通知后，可与任何切点一起使用。

#### 5.3.2.5. [Introduction 通知](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3242)

Spring 将引介通知视为一种特殊的拦截通知。

引介需要一个`IntroductionAdvisor`和一个`IntroductionInterceptor`，实现下面的接口：

```java
public interface IntroductionInterceptor extends MethodInterceptor {

    boolean implementsInterface(Class intf);
}
```

从 AOP 联盟 `MethodInterceptor` 接口继承的 `invoke()` 方法必须实现引介：也就是说，如果被调用的方法位于`引介接口`上，则引介拦截器负责处理方法调用--它不能调用 `proceed()`。

`引介通知`不能与任何点切点一起使用，因为它只适用于类而非方法级别。只有`InterceptionIntroductionAdvisor`才能使用引介通知，它有以下方法：

```java
public interface InterceptionIntroductionAdvisor extends InterceptionAdvisor {

    ClassFilter getClassFilter();

    IntroductionInterceptor getIntroductionInterceptor();

    Class[] getInterfaces();
}
```

没有 `MethodMatcher`，因此也没有与导入通知相关的 Pointcut。只有类过滤才符合逻辑。

`getInterfaces()` 方法返回该顾问引介的接口。

让我们看看 Spring 测试套件中的一个简单示例。假设我们要为一个或多个对象引介以下接口：

```java
public interface Lockable {
    void lock();
    void unlock();
    boolean locked();
}
```

这说明了一个 mixin。我们希望能够将通知对象转换为 Lockable，无论其类型如何，并调用锁定和解锁方法。如果我们调用 lock() 方法，我们希望所有设置器方法都抛出 LockedException。这样，我们就可以添加一个方面，提供使对象不可变的能力，而对象对此一无所知：这就是 AOP 的一个很好的例子。

首先，我们需要一个 `IntroductionInterceptor` 来完成繁重的工作。在这种情况下，我们扩展 `org.springframework.aop.support.DelegatingIntroductionInterceptor` 方便类。我们可以直接实现 `IntroductionInterceptor`，但在大多数情况下，使用 `DelegatingIntroductionInterceptor` 是最好的选择。

委托引介拦截器（`DelegatingIntroductionInterceptor`）旨在将引介委托给引介接口的实际实现，同时隐藏拦截的使用。可以使用构造函数参数将委托设置为任何对象；默认委托（使用无参数构造函数时）是 this。因此，在下面的示例中，委托是 `DelegatingIntroductionInterceptor` 的 LockMixin 子类。给定一个委托（默认情况下本身就是），`DelegatingIntroductionInterceptor` 实例会查找委托实现的所有接口（`IntroductionInterceptor` 除外），并支持针对其中任何接口的引介。LockMixin 等子类可以调用 `suppressInterflace(Class intf)` 方法来抑制不应暴露的接口。然而，无论引介拦截器准备支持多少接口，所使用的引介拦截器都将控制实际暴露的接口。引介的接口将隐藏目标程序对同一接口的任何实现。

因此，`LockMixin` 子类化了 `DelegatingIntroductionInterceptor` 并实现了 `Lockable` 本身。超类会自动识别出 `Lockable` 可用于支持引介，因此我们不需要指定这一点。我们可以用这种方法引介任意数量的接口。

请注意锁定实例变量的使用。这实际上是在目标对象的状态之外添加了额外的状态。

```java
public class LockMixin extends DelegatingIntroductionInterceptor 
    implements Lockable {

    private boolean locked;

    public void lock() {
        this.locked = true;
    }

    public void unlock() {
        this.locked = false;
    }

    public boolean locked() {
        return this.locked;
    }

    public Object invoke(MethodInvocation invocation) throws Throwable {
        if (locked() && invocation.getMethod().getName().indexOf("set") == 0)
            throw new LockedException();
        return super.invoke(invocation);
    }
}
```

通常没有必要重载 `invoke()` 方法：`DelegatingIntroductionInterceptor` 实现--如果方法被引介，则调用委托方法，否则向连接点前进--通常就足够了。在本例中，我们需要添加一个检查：如果处于锁定模式，则不能调用 `setter` 方法。

所需的引介顾问很简单。它所需要做的就是持有一个不同的 `LockMixin` 实例，并指定引介的接口--在本例中，只是 `Lockable`。更复杂的示例可能需要引用引介拦截器（将被定义为原型）：在这种情况下，没有与 `LockMixin` 相关的配置，因此我们只需使用 `new` 来创建它。

```java
public class LockMixinAdvisor extends DefaultIntroductionAdvisor {

    public LockMixinAdvisor() {
        super(new LockMixin(), Lockable.class);
    }
}
```

我们可以非常简单地应用这个顾问：它不需要任何配置。(不过，配置是必要的：没有 `IntroductionAdvisor` 就无法使用 `IntroductionInterceptor`）。与导入程序一样，该顾问必须按实例设置，因为它是有状态的。我们需要为每个通知对象创建不同的 `LockMixinAdvisor` 实例，因此也需要为每个通知对象创建不同的 `LockMixin` 实例。顾问包含被通知对象的部分状态。

我们可以使用 `Advised.addAdvisor()` 方法或（推荐方法）XML 配置，像其他顾问一样，以编程方式应用该顾问。下面讨论的所有代理创建选择，包括 "自动代理创建器"，都能正确处理介绍和有状态的混合体。

## 5.4. [Spring 中的 Advisor](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3395)

在 Spring 中，`Advisor`是对切面编程的一种模块化封装。`Advisor`通常包含**一个通知**和**一个切点**。

`org.springframework.aop.support.DefaultPointcutAdvisor` 是最常用的顾问类。例如，它可与 `MethodInterceptor`、`BeforeAdvice` 或 `ThrowsAdvice` 一起使用。

在同一个 AOP 代理中，可以混合使用 Spring 中的`Advisor`和通知类型。例如，您可以在一个代理配置中使用`环绕通知`、``抛出通知``和`在通知之前`的拦截：Spring 会自动创建必要的创建拦截器链。

## 5.5. [使用 ProxyFactoryBean 创建 AOP 代理](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-pfb)

如果您正在使用 Spring IoC 容器（`ApplicationContext` 或 `BeanFactory`）来处理业务对象，那么您应该使用 Spring 的 AOP `FactoryBean`。(请记住，工厂 Bean 引入了一层间接层，使其能够创建不同类型的对象）。

在 Spring 中创建 AOP 代理的基本方法是使用 `org.springframework.aop.framework.ProxyFactoryBean`。这可以完全控制将应用的指向和建议，以及它们的排序。不过，如果你不需要这样的控制，还有更简单的选择。

### 5.5.1[. 基础知识](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-pfb-1)

`ProxyFactoryBean`与其他 Spring `FactoryBean`实现一样，引入了一定程度的间接性。如果你定义了一个名称为 foo 的 `ProxyFactoryBean`，那么引用 foo 的对象看到的并不是`ProxyFactoryBean`实例本身，而是由`ProxyFactoryBean` 实现的`getObject()`方法创建的一个对象。该方法将创建一个封装目标对象的 AOP 代理。

使用 `ProxyFactoryBean` 或其他 IoC 感知类来创建 AOP 代理的一个最重要的好处是：这意味着通知和指向也可以由 IoC 来管理。这是一个强大的功能，可以实现其他 AOP 框架难以实现的某些方法。例如，通知本身可以引用应用程序对象（除目标对象外，任何 AOP 框架都应提供目标对象），从而受益于依赖注入提供的所有可插拔性。

### 5.5.2. [JavaBean 属性](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-pfb-2)

与 Spring 提供的大多数 `FactoryBean` 实现一样，`ProxyFactoryBean` 本身也是一个 JavaBean。其属性用于

- 指定要代理的目标
- 指定是否使用 CGLIB

一些关键属性继承自`org.springframework.aop.framework.ProxyConfig`：所有 AOP 代理工厂的超类。这些属性包括：

- proxyTargetClass：如果我们应该代理目标类而不是其接口，则为 true。如果为 true，我们就需要使用 CGLIB。
- optimize：是否对创建的代理进行积极优化。除非了解相关 AOP 代理如何处理优化，否则不要使用此设置。此设置目前仅用于 CGLIB 代理；对 JDK 动态代理（默认值）没有影响。
- frozen：配置代理工厂后，是否禁止更改通知。默认为 false。
- exposeProxy：是否应在 ThreadLocal 中公开当前代理，以便目标可以访问它。(如果目标需要获取代理，且 exposeProxy 为 true，则目标可以使用 `AopContext.currentProxy()` 方法。
- aopProxyFactory：AopProxyFactory 的实现。提供了一种自定义使用动态代理、CGLIB 或其他代理策略的方法。默认实现将适当选择动态代理或 CGLIB。应该不需要使用此属性；它的目的是允许在 Spring 1.1 中添加新的代理类型。

`ProxyFactoryBean` 特有的其他属性包括：

- proxyInterfaces：字符串接口名称数组。如果没有提供，将使用目标类的 CGLIB 代理
- interceptorNames：字符串数组，包含要应用的顾问、拦截器或其他建议名称。排序很重要。这些名称是当前工厂中的 Bean 名称，包括来自祖先工厂的 Bean 名称。
- singleton：无论 getObject() 方法被调用多少次，工厂是否都应返回一个对象。一些 FactoryBean 实现提供了这种方法。默认值为 true。如果要使用有状态的建议（例如，有状态的混合体），请使用原型建议，并将单例值设为 false。

### 5.5.3. [代理接口](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3534)

让我们来看一个 ProxyFactoryBean 运行的简单示例。该示例涉及：

- 将被代理的目标 Bean。这就是下面示例中的 "personTarget "Bean 定义。

- 通知器和拦截器用于提供建议。

- AOP 代理 Bean 定义指定了目标对象（personTarget Bean）和要代理的接口，以及要应用的通知。

```java
<bean id="personTarget" class="com.mycompany.PersonImpl">
  <property name="name"><value>Tony</value></property>
  <property name="age"><value>51</value></property>
</bean>

<bean id="myAdvisor" class="com.mycompany.MyAdvisor">
  <property name="someProperty"><value>Custom string property value</value></property>
</bean>

<bean id="debugInterceptor" class="org.springframework.aop.interceptor.DebugInterceptor">
</bean>

<bean id="person" 
  class="org.springframework.aop.framework.ProxyFactoryBean">
  <property name="proxyInterfaces"><value>com.mycompany.Person</value></property>

  <property name="target"><ref local="personTarget"/></property>
  <property name="interceptorNames">
      <list>
          <value>myAdvisor</value>
          <value>debugInterceptor</value>
      </list>
  </property>
</bean>
```

请注意：`interceptorNames` 属性包含一个字符串列表：当前工厂中`拦截器`或`通知器`的`bean`名称。可以使用通知器、拦截器、返回前、返回后和抛出通知对象。通知器的排序很重要。

您可能想知道为什么列表中不包含 `Bean` 引用？原因是如果 `ProxyFactoryBean` 的单例属性设置为 false，它就必须能够返回独立的代理实例。如果任何顾问本身就是一个原型，那么就需要返回一个独立的实例，因此必须能够从工厂获得原型的实例；仅持有引用是不够的。

上面的 "person "Bean 定义可以用来代替 "Person "的实现，如下所示：

```java
Person person = (Person) factory.getBean("person");
```

同一 IoC 上下文中的其他 Bean 可以表达对它的强类型依赖，就像对普通 Java 对象一样：

```java
<bean id="personUser" class="com.mycompany.PersonUser">
    <property name="person"><ref local="person" /></property>
</bean>
```

本例中的 PersonUser 类将公开 Person 类型的属性。就其本身而言，AOP 代理可以透明地代替 "真正" Person 的实现。不过，它的类将是一个动态代理类。我们可以将其转换为 Advised 接口（下文将讨论）。

使用匿名内部 Bean 可以隐藏目标和代理之间的区别，如下所示。只有 ProxyFactoryBean 的定义是不同的；包含建议只是为了完整：

```java
<bean id="myAdvisor" class="com.mycompany.MyAdvisor">
    <property name="someProperty"><value>Custom string property value</value></property>
</bean>

<bean id="debugInterceptor" class="org.springframework.aop.interceptor.DebugInterceptor">
</bean>

<bean id="person" 
    class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="proxyInterfaces"><value>com.mycompany.Person</value></property>

    <!-- Use inner bean, not local reference to target -->
    <property name="target">
        <bean class="com.mycompany.PersonImpl">
            <property name="name"><value>Tony</value></property>
            <property name="age"><value>51</value></property>
        </bean>
   </property>

    <property name="interceptorNames">
        <list>
            <value>myAdvisor</value>
            <value>debugInterceptor</value>
        </list>
    </property>
</bean>
```

这样做的好处是只有一个 `Person` 类型的对象：如果我们想防止应用程序上下文中的用户获取对未建议对象的引用，或者需要避免与 Spring IoC 自动布线产生任何歧义，这样做就很有用。可以说，`ProxyFactoryBean` 定义自成一体也是一个优点。不过，在某些时候，从工厂中获取未建议的目标可能实际上是一种优势：例如，在某些测试场景中。

### 5.5.4. [代理类](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3601)

如果需要代理一个类，而不是一个或多个接口，该怎么办？

想象一下，在我们上面的示例中，没有 Person 接口：我们需要建议一个名为 Person 的类，该类没有实现任何业务接口。在这种情况下，可以将 Spring 配置为使用 CGLIB 代理，而不是动态代理。只需将上述 ProxyFactoryBean 上的 proxyTargetClass 属性设置为 true 即可。虽然编程时最好使用接口而非类，但在处理遗留代码时，建议使用未实现接口的类的功能还是很有用的。(总的来说，Spring 并不具有规范性。虽然它让应用良好实践变得更容易，但也避免了强迫使用特定的方法）。

如果您愿意，您可以在任何情况下强制使用 CGLIB，即使您有接口。

CGLIB 代理的工作原理是在运行时生成目标类的子类。Spring 会对生成的子类进行配置，以便将方法调用委托给原始目标类：该子类用于实现装饰器模式，编织建议。

CGLIB 代理一般来说对用户是透明的。不过，也有一些问题需要考虑：

- 不能建议使用 `final` 方法，因为它们不能被覆盖。
- 您需要在类路径上安装 CGLIB 2 二进制文件；动态代理可通过 JDK

CGLIB 代理和动态代理的性能差别不大。从 Spring 1.0 开始，动态代理速度稍快。不过，未来可能会有所改变。在这种情况下，性能不应成为决定性的考虑因素。

## 5.6. [方便的代理创建](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-tfb)

通常情况下，我们并不需要 ProxyFactoryBean 的全部功能，因为我们只对其中一个方面感兴趣：例如，事务管理。

当我们想专注于某个特定方面时，可以使用许多方便的工厂来创建 AOP 代理。我们将在其他章节中讨论这些工厂，因此在此仅对其中的一些工厂进行简要介绍。

### 5.6.1. [TransactionProxyFactoryBean](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3647)

随 Spring 一起提供的 jPetStore 示例应用程序展示了 TransactionProxyFactoryBean 的使用。

TransactionProxyFactoryBean 是 ProxyConfig 的子类，因此基本配置与 ProxyFactoryBean 共享。(请参阅上面的 ProxyConfig 属性列表）。

下面这个来自 jPetStore 的示例说明了它是如何工作的。与代理工厂 Bean 一样，也有一个目标 Bean 定义。应根据代理工厂 Bean 定义（此处为 "petStore"）而不是目标 POJO（"petStoreTarget"）来表达依赖关系。

TransactionProxyFactoryBean 需要一个目标和有关 "事务属性 "的信息，指定哪些方法应该是事务性的，以及所需的传播和其他设置：

```java
<bean id="petStoreTarget" class="org.springframework.samples.jpetstore.domain.logic.PetStoreImpl">
    <property name="accountDao"><ref bean="accountDao"/></property>
    <!-- Other dependencies omitted -->
</bean>

<bean id="petStore" 
    class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
    <property name="transactionManager"><ref bean="transactionManager"/></property>
    <property name="target"><ref local="petStoreTarget"/></property>
    <property name="transactionAttributes">
        <props>
            <prop key="insert*">PROPAGATION_REQUIRED</prop>
            <prop key="update*">PROPAGATION_REQUIRED</prop>
            <prop key="*">PROPAGATION_REQUIRED,readOnly</prop>
        </props>
    </property>
</bean>
```

与 ProxyFactoryBean 一样，我们可能会选择使用内部 bean 来设置目标属性的值，而不是引用顶级目标 bean。

TransactionProxyFactoryBean 会自动创建事务顾问，包括基于事务属性的切入点，因此只建议使用事务方法。

TransactionProxyFactoryBean 允许使用 preInterceptors 和 postInterceptors 属性指定 "前 "和 "后 "建议。这些属性使用拦截器、其他建议或顾问的对象数组，以便将其放在事务拦截器之前或之后的拦截链中。可以使用 XML Bean 定义中的 `<list>` 元素填充这些属性，如下所示：

```java
<property name="preInterceptors">
    <list>
        <ref local="authorizationInterceptor"/>
        <ref local="notificationBeforeAdvice"/>
    </list>
</property>
<property name="postInterceptors">
    <list>
        <ref local="myAdvisor"/>
    </list>
</property>
```

这些属性可以添加到上述 "petStore "Bean 定义中。一种常见的用法是将事务性与声明安全性结合起来：这与 EJB 提供的方法类似。

由于使用的是实际实例引用，而不是 ProxyFactoryBean 中的 Bean 名称，前拦截器和后拦截器只能用于共享实例建议。因此，它们不能用于有状态建议：例如，在混合体中。这与 TransactionProxyFactoryBean 的目的是一致的。它提供了一种进行普通事务设置的简单方法。如果您需要更复杂、更个性化的 AOP，请考虑使用通用的 ProxyFactoryBean 或自动代理创建器（见下文）。

特别是如果我们将 Spring AOP 视为 EJB 的替代品，我们会发现大多数建议都相当通用，并使用共享实例模型。声明式事务管理和安全检查就是典型的例子。

TransactionProxyFactoryBean 依赖于通过其 transactionManager JavaBean 属性实现的 PlatformTransactionManager。这就允许基于 JTA、JDBC 或其他策略的可插拔事务实现。这与 Spring 事务抽象有关，而非 AOP。我们将在下一章讨论事务基础架构。

如果你只对声明式事务管理感兴趣，TransactionProxyFactoryBean 是一个很好的解决方案，而且比使用 ProxyFactoryBean 更简单。

### 5.6.2. [EJB 代理](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3728)

其他专用代理为 EJB 创建代理，使调用代码可以直接使用 EJB "业务方法 "接口。调用代码无需执行 JNDI 查找或使用 EJB 创建方法：大大提高了可读性和架构灵活性。

有关详细信息，请参阅本手册中有关 Spring EJB 服务的章节。

## 5.7. [精炼的代理定义](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-concise-proxy)

特别是在定义事务代理时，可能会出现许多类似的代理定义。使用父类和子类 bean 定义以及内部 bean 定义，可以使代理定义更加简洁明了。

首先要为代理创建一个父类、模板、Bean 定义：

```json
{ "parent": "parent", "template": "template", "beanDefinition": "bean definition" }

<bean id="txProxyTemplate" abstract="true"
        class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
    <property name="transactionManager"><ref local="transactionManager"/></ref></property>
    <property name="transactionAttributes">
      <props>
        <prop key="*">PROPAGATION_REQUIRED</prop>
      </props>
    </property>
</bean>
```

这将永远不会被实例化，因此实际上可能是不完整的。然后，需要创建的每个代理都只是一个子 Bean 定义，它将代理的目标封装为一个内部 Bean 定义，因为目标本身永远不会被使用。

```java
<bean id="myService" parent="txProxyTemplate">
    <property name="target">
      <bean class="org.springframework.samples.MyServiceImpl">
      </bean>
    </property>
</bean>
```

当然，也可以覆盖父模板的属性，比如本例中的事务传播设置：

```java
<bean id="mySpecialService" parent="txProxyTemplate">
    <property name="target">
      <bean class="org.springframework.samples.MySpecialServiceImpl">
      </bean>
    </property>
    <property name="transactionAttributes">
      <props>
        <prop key="get*">PROPAGATION_REQUIRED,readOnly</prop>
        <prop key="find*">PROPAGATION_REQUIRED,readOnly</prop>
        <prop key="load*">PROPAGATION_REQUIRED,readOnly</prop>
        <prop key="store*">PROPAGATION_REQUIRED</prop>
      </props>
    </property>
</bean>
```

请注意，在上面的示例中，我们已通过使用 abstract 属性将父 Bean 定义明确标记为抽象（如前所述），因此它实际上可能不会被实例化。应用程序上下文（而非简单的 Bean 工厂）默认情况下会预先实例化所有单例。因此，如果您有一个（父）Bean 定义，而您只打算将其用作模板，并且该定义指定了一个类，那么您必须确保将 abstract 属性设置为 true，否则应用程序上下文将实际尝试预实例化它，这一点非常重要（至少对于单例 Bean 而言）。

## 5.8. [使用 ProxyFactory 以编程方式创建 AOP 代理](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-prog)

使用 Spring 以编程方式创建 AOP 代理非常简单。这样，您就可以使用 Spring AOP，而无需依赖 Spring IoC。

下面的列表显示了为目标对象创建代理的过程，其中包含一个拦截器和一个顾问。目标对象实现的接口将自动被代理：

```java
ProxyFactory factory = new ProxyFactory(myBusinessInterfaceImpl);
factory.addInterceptor(myMethodInterceptor);
factory.addAdvisor(myAdvisor);
MyBusinessInterface tb = (MyBusinessInterface) factory.getProxy();
```

第一步是构建一个 org.springframework.aop.framework.ProxyFactory 类型的对象。您可以使用目标对象创建该对象（如上面的示例），也可以在另一个构造函数中指定要代理的接口。

您可以添加拦截器或顾问，并在代理服务器工厂的整个生命周期内操纵它们。如果添加了引入拦截器（IntroductionInterceptionAroundAdvisor），就可以使代理实现其他接口。

ProxyFactory 上还有一些方便的方法（继承自 AdvisedSupport），允许您添加其他建议类型，如 before 建议和 throws 建议。AdvisedSupport 是 ProxyFactory 和 ProxyFactoryBean 的超类。

在大多数应用程序中，将 AOP 代理创建与 IoC 框架集成是最佳做法。我们建议您使用 AOP 将配置从 Java 代码中外部化，这与一般情况下的做法相同。

## 5.9. [操作通知对象](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3794)

无论如何创建 AOP 代理，都可以使用 org.springframework.aop.framework.Advised 接口来操作它们。任何 AOP 代理都可以投向此接口，无论它实现了其他什么接口。该接口包括以下方法：

```java
Advisor[] getAdvisors();

void addAdvice(Advice advice) throws AopConfigException;

void addAdvice(int pos, Advice advice) throws AopConfigException;

void addAdvisor(Advisor advisor) throws AopConfigException;

void addAdvisor(int pos, Advisor advisor) throws AopConfigException;

int indexOf(Advisor advisor);

boolean removeAdvisor(Advisor advisor) throws AopConfigException;

void removeAdvisor(int index) throws AopConfigException;

boolean replaceAdvisor(Advisor a, Advisor b) throws AopConfigException;

boolean isFrozen();
```

getAdvisors() 方法将为添加到工厂的每个顾问、拦截器或其他建议类型返回一个顾问。如果您添加了一个顾问，在此索引处返回的顾问将是您添加的对象。如果添加了拦截器或其他建议类型，Spring 会将其封装在一个顾问中，并使用一个总是返回 true 的点切线。因此，如果您添加了一个 MethodInterceptor，那么此索引返回的顾问将是一个 DefaultPointcutAdvisor，它将返回您的 MethodInterceptor 和一个与所有类和方法相匹配的 pointcut。

addAdvisor() 方法可用于添加任何顾问。通常，持有快捷方式和建议的顾问是通用的 DefaultPointcutAdvisor，它可以与任何建议或快捷方式一起使用（但不能用于导入）。

默认情况下，即使代理已创建，也可以添加或删除顾问或拦截器。唯一的限制是无法添加或删除引入顾问，因为工厂中的现有代理不会显示接口的变化。(您可以从工厂获取一个新的代理来避免这个问题）。

将 AOP 代理转换为 Advised 接口并检查和操作其建议的一个简单示例：

```java
Advised advised = (Advised) myObject;
Advisor[] advisors = advised.getAdvisors();
int oldAdvisorCount = advisors.length;
System.out.println(oldAdvisorCount + " advisors");

// Add an advice like an interceptor without a pointcut
// Will match all proxied methods
// Can use for interceptors, before, after returning or throws advice
advised.addAdvice(new DebugInterceptor());

// Add selective advice using a pointcut
advised.addAdvisor(new DefaultPointcutAdvisor(mySpecialPointcut, myAdvice));

assertEquals("Added two advisors", oldAdvisorCount + 2, advised.getAdvisors().length);
```

在生产中修改业务对象的建议是否可取（不是双关语）值得商榷，尽管毫无疑问存在合理的使用情况。不过，它在开发过程中可能非常有用：例如，在测试中。我有时会发现，以拦截器或其他建议的形式添加测试代码，进入我要测试的方法调用内部，是非常有用的。(例如，建议可以进入为该方法创建的事务内部：例如，在标记事务回滚之前，运行 SQL 检查数据库是否已正确更新）。

根据创建代理的方式，通常可以设置冻结标记，在这种情况下，建议 isFrozen() 方法将返回 true，任何通过添加或删除来修改建议的尝试都将导致 AopConfigException 异常。冻结建议对象状态的功能在某些情况下非常有用：例如，防止调用代码删除安全拦截器。在 Spring 1.1 中，如果已知不需要在运行时修改建议，也可以使用该功能进行积极优化。

## 5.10. [使用 “autoproxy” 工具](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-autoproxy)

到目前为止，我们已经考虑过使用 ProxyFactoryBean 或类似的工厂 Bean 来显式创建 AOP 代理。

Spring 还允许我们使用 "自动代理 "Bean 定义，它可以自动代理选定的 Bean 定义。这是建立在 Spring "Bean 后置处理器 "基础架构之上的，它可以在容器加载时修改任何 Bean 定义。

在这种模式下，您需要在 XML Bean 定义文件中设置一些特殊的 Bean 定义，以配置自动代理基础架构。这样，您只需声明符合自动代理条件的目标即可：您无需使用 ProxyFactoryBean。

有两种方法可以做到这一点：

1. 使用自动代理创建器，在当前上下文中引用特定的 bean
2. 值得单独考虑的自动代理创建特例；由源级元数据属性驱动的自动代理创建

### 5.10.1. [自动代理 bean 定义](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-autoproxy-choices)

`org.springframework.aop.framework.autoproxy` 包提供了以下标准自动代理创建器。

#### 5.10.1.1. [BeanNameAutoProxyCreator](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3884)

BeanNameAutoProxyCreator 会自动为名称符合字面值或通配符的 Bean 创建 AOP 代理。

```java
<bean id="jdkBeanNameProxyCreator" 
    class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
    <property name="beanNames"><value>jdk*,onlyJdk</value></property>
    <property name="interceptorNames">
        <list>
            <value>myInterceptor</value>
        </list>
    </property>
</bean>
```

与 ProxyFactoryBean 一样，拦截器也有一个拦截器名称（interceptorNames）属性，而不是一个拦截器列表，以便为原型顾问提供正确的行为。已命名的 "拦截器 "可以是顾问或任何建议类型。

与一般的自动代理一样，使用 BeanNameAutoProxyCreator 的主要目的是将相同的配置一致地应用于多个对象，并尽量减少配置量。它是将声明式事务应用于多个对象的常用选择。

名称匹配的 Bean 定义（如上例中的 "jdkMyBean "和 "onlyJdk"）是目标类的普通旧 Bean 定义。BeanNameAutoProxyCreator 将自动创建一个 AOP 代理。相同的建议将应用于所有匹配的 Bean。请注意，如果使用的是建议（而不是上例中的拦截器），那么不同的豆可能会应用不同的指向。

#### 5.10.1.2. [DefaultAdvisorAutoProxyCreator](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3907)

DefaultAdvisorAutoProxyCreator 是一种更通用、功能更强大的自动代理创建器。它会在当前上下文中自动应用符合条件的顾问，而无需在自动代理顾问的 Bean 定义中包含特定的 Bean 名称。它与 BeanNameAutoProxyCreator 一样，都具有一致配置和避免重复的优点。

使用这一机制包括：

- 指定 DefaultAdvisorAutoProxyCreator Bean 定义
- 在相同或相关上下文中指定任意数量的顾问。请注意，这些顾问必须是顾问，而不仅仅是拦截器或其他建议。这样做是必要的，因为必须有一个评估点切入，以检查候选 bean 定义的每个建议是否合格。

DefaultAdvisorAutoProxyCreator 会自动评估每个顾问中包含的点切，以确定它应该对每个业务对象（例如示例中的 "businessObject1 "和 "businessObject2"）应用哪些建议（如果有的话）。

这意味着每个业务对象可以自动应用任意数量的顾问。如果顾问中没有任何点切与业务对象中的任何方法相匹配，则不会代理该对象。在为新业务对象添加 bean 定义时，如有必要，将自动代理这些 bean 定义。

一般来说，自动代理的优点是使调用者或依赖程序无法获得未经建议的对象。在此 ApplicationContext 上调用 getBean("businessObject1") 将返回一个 AOP 代理，而不是目标业务对象。(前面展示的 "内部 bean "习例也具有这种优势）。

```java
<bean id="autoProxyCreator"
    class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator">
</bean>

<bean id="txAdvisor"
    autowire="constructor"
    class="org.springframework.transaction.interceptor.TransactionAttributeSourceAdvisor">
    <property name="order"><value>1</value></property>
</bean>

<bean id="customAdvisor"
    class="com.mycompany.MyAdvisor">
</bean>

<bean id="businessObject1"
    class="com.mycompany.BusinessObject1">
    <!-- Properties omitted -->
</bean>

<bean id="businessObject2"
    class="com.mycompany.BusinessObject2">
</bean>
```

如果您想对许多业务对象一致地应用相同的建议，DefaultAdvisorAutoProxyCreator 将非常有用。一旦基础架构定义到位，您就可以简单地添加新的业务对象，而无需包含特定的代理配置。您还可以非常容易地添加其他方面，例如跟踪或性能监控方面，而只需对配置进行最小的更改。

DefaultAdvisorAutoProxyCreator 支持过滤（使用命名约定，以便只评估某些顾问，允许在同一工厂中使用多个不同配置的 AdvisorAutoProxyCreator）和排序。顾问可以实现 org.springframework.core.Ordered 接口，以确保在出现问题时能正确排序。上例中使用的 TransactionAttributeSourceAdvisor 有一个可配置的排序值；默认为无序。

#### 5.10.1.3. [AbstractAdvisorAutoProxyCreator](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3955)

这是 DefaultAdvisorAutoProxyCreator 的超类。如果顾问定义对框架 DefaultAdvisorAutoProxyCreator 的行为定制不足，您可以通过子类化该类来创建自己的自动代理创建器。

### 5.10.2. [使用元数据驱动的自动代理](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-autoproxy-metadata)

一种特别重要的自动代理是由元数据驱动的。这产生了一种类似于 .NET ServicedComponents 的编程模型。它不像 EJB 那样使用 XML 部署描述符，而是将事务管理和其他企业服务的配置保存在源代码级属性中。

在这种情况下，您可以将 DefaultAdvisorAutoProxyCreator 与能理解元数据属性的顾问结合使用。元数据的具体内容保存在候选顾问的点切部分，而不是自动代理创建类本身。

这实际上是 DefaultAdvisorAutoProxyCreator 的一个特例，但值得单独考虑。(元数据感知代码是在顾问中包含的指向代码中，而不是 AOP 框架本身）。

jPetStore 示例应用程序的 /attributes 目录显示了属性驱动自动代理的使用。在这种情况下，无需使用 TransactionProxyFactoryBean。只需在业务对象上定义事务属性就足够了，因为使用了元数据感知的指向切入。bean 定义包括 /WEB-INF/declarativeServices.xml 中的以下代码。请注意，这是通用代码，可以在 jPetStore 之外使用：

```java
<bean id="autoProxyCreator" 
    class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator">
</bean>

<bean id="transactionAttributeSource"
    class="org.springframework.transaction.interceptor.AttributesTransactionAttributeSource"
    autowire="constructor">
</bean>

<bean id="transactionInterceptor"
    class="org.springframework.transaction.interceptor.TransactionInterceptor"
    autowire="byType">
</bean>

<bean id="transactionAdvisor"
    class="org.springframework.transaction.interceptor.TransactionAttributeSourceAdvisor"
    autowire="constructor" >
</bean>

<bean id="attributes"
    class="org.springframework.metadata.commons.CommonsAttributes"
/>
```

`DefaultAdvisorAutoProxyCreator` Bean 定义--本例中称为 "autoProxyCreator"，但名称并不重要（甚至可以省略）--将拾取当前应用程序上下文中所有符合条件的指向。在这种情况下，`TransactionAttributeSourceAdvisor` 类型的 "transactionAdvisor" Bean 定义将适用于携带事务属性的类或方法。`TransactionAttributeSourceAdvisor` 通过构造函数依赖性依赖于 `TransactionInterceptor`。示例通过自动接线解决了这一问题。`AttributesTransactionAttributeSource` 依赖于 `org.springframework.metadata.Attributes` 接口的实现。在本片段中，"attributes" Bean 使用 Jakarta Commons Attributes API 获取属性信息，从而满足了这一要求。(应用程序代码必须已使用 Commons Attributes 编译任务编译）。

这里定义的 TransactionInterceptor 依赖于 PlatformTransactionManager 定义，该定义没有包含在本通用文件中（尽管可以包含），因为它将针对应用程序的事务要求（通常是 JTA，如本示例中的 JTA，或 Hibernate、JDO 或 JDBC）：

```java
<bean id="transactionManager" 
    class="org.springframework.transaction.jta.JtaTransactionManager"/>
```

如果您只需要声明式事务管理，那么使用这些通用 XML 定义将导致 Spring 自动代理所有带有事务属性的类或方法。您无需直接使用 AOP，编程模型与 .NET ServicedComponents 相似。
这种机制是可扩展的。可以根据自定义属性进行自动代理。您需要
- 定义自定义属性。
- 指定一个带有必要建议的顾问，包括一个因类或方法上存在自定义属性而触发的快捷方式。您也可以使用现有的建议，只需执行一个静态快捷方式，即可获取自定义属性。

这些顾问可以是每个建议类（例如混合类）所独有的：它们只需被定义为原型而非单例 Bean 定义。例如，如上图所示，Spring 测试套件中的 LockMixin 引入拦截器可与属性驱动的点切法结合使用，以实现 mixin 目标，如图所示。我们使用通用的 DefaultPointcutAdvisor，并使用 JavaBean 属性进行配置：

```java
<bean id="lockMixin"
    class="org.springframework.aop.LockMixin"
    singleton="false"
/>

<bean id="lockableAdvisor"
    class="org.springframework.aop.support.DefaultPointcutAdvisor"
    singleton="false"
>
    <property name="pointcut">
        <ref local="myAttributeAwarePointcut"/>
    </property>
    <property name="advice">
        <ref local="lockMixin"/>
    </property>
</bean>

<bean id="anyBean" class="anyclass" ...
```

如果属性感知点切匹配 anyBean 或其他 Bean 定义中的任何方法，就会应用 mixin。请注意，lockMixin 和 lockableAdvisor 定义都是原型。myAttributeAwarePointcut 可以是一个单例定义，因为它不保存单个建议对象的状态。

## 5.11. [使用 TargetSources](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-targetsource)

Spring 提供了 TargetSource 的概念，用 org.springframework.aop.TargetSource 接口表示。该接口负责返回实现连接点的 "目标对象"。每次 AOP 代理处理方法调用时，都会要求 TargetSource 实现提供一个目标实例。

使用 Spring AOP 的开发人员通常不需要直接使用 TargetSource，但它提供了支持池化、热插拔和其他复杂目标的强大手段。例如，池化 TargetSource 可以使用池管理实例，为每次调用返回不同的目标实例。

如果未指定 TargetSource，则会使用封装本地对象的默认实现。每次调用都会返回相同的目标（如您所料）。

让我们来看看 Spring 提供的标准目标源，以及如何使用它们。

使用自定义目标源时，目标通常需要是一个原型，而不是一个单例 Bean 定义。这样，Spring 就能在需要时创建一个新的目标实例。

### 5.11.1. [热插拔目标源](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-ts-swap)

org.springframework.aop.target.HotSwappableTargetSource 允许切换 AOP 代理的目标，同时允许调用者保留对它的引用。

更改目标源的目标会立即生效。HotSwappableTargetSource 是线程安全的。

您可以通过 HotSwappableTargetSource 上的 swap() 方法更改目标，具体方法如下：

```java
HotSwappableTargetSource swapper = 
    (HotSwappableTargetSource) beanFactory.getBean("swapper");
Object oldTarget = swapper.swap(newTarget);
```

所需的 XML 定义如下：
```java
<bean id="initialTarget" class="mycompany.OldTarget">
</bean>

<bean id="swapper" 
    class="org.springframework.aop.target.HotSwappableTargetSource">
    <constructor-arg><ref local="initialTarget"/></constructor-arg>
</bean>

<bean id="swappable" 
    class="org.springframework.aop.framework.ProxyFactoryBean"
>
    <property name="targetSource">
        <ref local="swapper"/>
    </property>
</bean>
```
上述 swap() 调用会更改可交换 Bean 的目标。持有该 bean 引用的客户端不会察觉到这一变化，但会立即开始访问新的目标。
虽然这个示例没有添加任何建议--使用 TargetSource 也不一定要添加建议--当然，任何 TargetSource 都可以与任意建议结合使用。

### 5.11.2. [池化 Target Sources](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-ts-pool)
使用池目标源可提供与无状态会话 EJB 类似的编程模型，即维护一个由相同实例组成的池，方法调用将指向池中的空闲对象。
Spring 池与 SLSB 池的一个重要区别是，Spring 池可以应用于任何 POJO。与一般的 Spring 一样，这种服务可以以非侵入的方式应用。
Spring 为 Jakarta Commons Pool 1.1 提供了开箱即用的支持，它提供了相当高效的池化实现。要使用此功能，您需要在应用程序的 classpath 中安装 commons-pool Jar。您也可以子类化 org.springframework.aop.target.AbstractPoolingTargetSource 以支持任何其他池 API。
配置示例如下：
```java
<bean id="businessObjectTarget" class="com.mycompany.MyBusinessObject" 
    singleton="false">
    ... properties omitted
</bean>

<bean id="poolTargetSource" 
    class="org.springframework.aop.target.CommonsPoolTargetSource">
    <property name="targetBeanName"><value>businessObjectTarget</value></property>
    <property name="maxSize"><value>25</value></property>
</bean>

<bean id="businessObject" 
    class="org.springframework.aop.framework.ProxyFactoryBean"
>
    <property name="targetSource"><ref local="poolTargetSource"/></property>
    <property name="interceptorNames"><value>myInterceptor</value></property>
</bean>
```
请注意，目标对象--即示例中的 "businessObjectTarget"--必须是一个原型。这允许 PoolingTargetSource 实现创建新的目标实例，以便在必要时扩大池。有关其属性的信息，请参阅 AbstractPoolingTargetSource 的 Javadoc 以及您希望使用的具体子类：maxSize 是最基本的属性，并且始终保证存在。
在这种情况下，"myInterceptor "是需要在同一 IoC 上下文中定义的拦截器的名称。不过，使用池化并不需要指定拦截器。如果你只想使用池化，而不需要其他建议，那就根本不要设置拦截器名称（interceptorNames）属性。
可以对 Spring 进行配置，使其能够将任何池对象投向 org.springframework.aop.target.PoolingConfig 接口，该接口通过一个介绍来公开有关池的配置和当前大小的信息。你需要像这样定义一个顾问：
```java
<bean id="poolConfigAdvisor" 
    class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
    <property name="targetObject"><ref local="poolTargetSource" /></property>
    <property name="targetMethod"><value>getPoolingConfigMixin</value></property>
</bean>
```
该顾问通过调用 AbstractPoolingTargetSource 类上的方便方法获得，因此使用了 MethodInvokingFactoryBean。该顾问的名称（此处为 "poolConfigAdvisor"）必须位于暴露池对象的 ProxyFactoryBean 中的拦截器名称列表中。
演员阵容如下：
```java
PoolingConfig conf = (PoolingConfig) beanFactory.getBean("businessObject");
System.out.println("Max pool size is " + conf.getMaxSize());
```
通常没有必要对无状态服务对象进行池化。我们认为不应将其作为默认选择，因为大多数无状态对象都具有天然的线程安全性，如果资源被缓存，实例池就会出现问题。
使用自动代理可以实现更简单的池化。可以设置任何自动代理创建者使用的 TargetSources。

### 5.11.3. [原型 Target Sources](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-ts-prototype)
设置 "原型" 目标源与池化目标源类似。在这种情况下，每次方法调用都会创建一个新的目标实例。虽然在现代 JVM 中创建一个新对象的成本并不高，但连接新对象（满足其 IoC 依赖关系）的成本可能会更高。因此，如果没有充分的理由，您不应该使用这种方法。
为此，您可以对上图中的 poolTargetSource 定义进行如下修改。(为了清晰起见，我还更改了名称）。
```java
<bean id="prototypeTargetSource" 
    class="org.springframework.aop.target.PrototypeTargetSource">
    <property name="targetBeanName"><value>businessObjectTarget</value></property>
</bean>
```
只有一个属性：目标 Bean 的名称。TargetSource 实现中使用继承来确保命名的一致性。与池化目标源一样，目标 Bean 必须是一个原型 Bean 定义。

## 5.12[. 定义新的 Advice 类型](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-extensibility)
Spring AOP 的设计具有可扩展性。虽然目前内部使用的是拦截实现策略，但除了拦截周围的建议、之前的建议、抛出的建议和返回建议之后的建议之外，还可以支持任意建议类型。
org.springframework.aop.framework.adapter 包是一个 SPI 包，允许在不更改核心框架的情况下添加新的自定义建议类型支持。对自定义建议类型的唯一限制是它必须实现 org.aopalliance.aop.Advice 标签接口。
请参阅 org.springframework.aop.framework.adapter 软件包的 Javadocs 了解更多信息

## 5.13. [延伸阅读和资源](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e4201)
我推荐 Ramnivas Laddad 所著的《AspectJ in Action》（Manning，2003 年）作为 AOP 的入门读物。
有关 Spring AOP 的更多示例，请参阅 Spring 示例应用程序：
- JPetStore 的默认配置说明了如何使用 TransactionProxyFactoryBean 进行声明式事务管理
- JPetStore 的 /attributes 目录说明了属性驱动的声明式事务管理的使用情况

如果您对 Spring AOP 更高级的功能感兴趣，请查看测试套件。测试覆盖率超过 90%，这说明了本文档未讨论的高级功能。

## 5.14. [路线图](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e4223)
Spring AOP 与 Spring 的其他部分一样，正在积极开发中。核心 API 非常稳定。与 Spring 的其他部分一样，AOP 框架也非常模块化，在保留基本设计的同时实现了扩展。我们计划在 Spring 1.1 版本中进行多项改进，以保持向后兼容性。这些改进包括
- 性能改进：AOP 代理的创建由工厂通过策略接口来处理。因此，我们可以支持更多的 AOPProxy 类型，而不会影响用户代码或核心实现。我们计划在 1.0.3 版本中对 CGLIB 代理进行重大性能优化，并在 Spring 1.1 中对运行时不会更改建议的情况进行进一步优化。这将大大减少 AOP 框架的开销。不过请注意，在正常使用中，AOP 框架的开销并不是问题。
- 更具表现力的快捷方式Spring 目前提供了一个表现力丰富的 Pointcut 接口，但我们可以通过添加更多的 Pointcut 实现来增加价值。我们正在考虑与 AspectJ 集成，以便在 Spring 配置文件中使用 AspectJ 的 Pointcut 表达式。如果您希望贡献有用的 Pointcut，请联系我们！

最重要的改进可能涉及与 AspectJ 的集成，这将与 AspectJ 社区合作完成。我们相信，这将在以下方面为 Spring 和 AspectJ 用户带来巨大的好处：
- 允许使用 Spring IoC 配置 AspectJ 方面。这有可能在适当的地方将 AspectJ 方面集成到应用程序中，就像将 Spring 方面集成到应用程序 IoC 上下文中一样。
- 允许在 Spring 配置中使用 AspectJ 的点切表达式来针对 Spring 建议。这比我们自己设计点切表达式语言有很大的好处；AspectJ 经过了深思熟虑，而且文档齐全。

这两种集成都应在 Spring 1.1 中提供。