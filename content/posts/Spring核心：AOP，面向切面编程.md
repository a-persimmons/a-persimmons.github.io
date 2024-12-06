---
author: ["柿子"]
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
date: {{ .Date }}
summary: ""
tags: [""]
typora-root-url: ../../static
typora-copy-images-to: ../../static/images
---

# 5. Spring 核心：AOP，面向切面编程

> 本文内容是逐段翻译[spring-framework 1.1.x 官方文档第5章](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html)的内容。为什么不是1.0？因为1.1.x才开始有官方文档，之前的版本只有api文档。
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

matches(Method, Class) 方法用于测试该快捷方式是否与目标类上的给定方法相匹配。该评估可在创建 AOP 代理时执行，以避免每次调用方法时都要进行测试。如果给定方法的 2 参数匹配方法返回 true，且 MethodMatcher 的 isRuntime() 方法返回 true，那么每次方法调用时都会调用 3 参数匹配方法。这样，在执行目标建议之前，点切口就能立即查看传递给方法调用的参数。

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

RegexpMethodPointcut 的方便子类 RegexpMethodPointcutAdvisor 也允许我们引用建议。(请记住，Advice 可以是拦截器、before advice、throws advice 等）这简化了布线，因为一个 bean 既是快捷方式，又是顾问，如下所示：

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

RegexpMethodPointcutAdvisor 可用于任何建议类型。

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

在 Spring 1.0 RC2 及以上版本中，您可以在任何建议类型中使用自定义点切入。

### 5.2.5. [自定义切点](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3077)

由于 Spring 中的切点是 Java 类，而不是语言特性（如 AspectJ），因此可以声明自定义切点，无论是静态还是动态切点。不过，Spring 并不支持用 AspectJ 语法编码的复杂点切分表达式。不过，Spring 中的自定义快捷方式可以任意复杂。

Spring 的后续版本可能会支持 JAC 提供的 "语义指向"：例如，"改变目标对象中实例变量的所有方法"。

## 5.3. [Spring 中的通知类型](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3084)

现在让我们看看 Spring AOP 是如何处理建议的。

### 5.3.1. [通知生命周期](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3089)

Spring 通知可以在所有建议对象中共享，也可以为每个建议对象独有。这相当于按类或按实例提供通知。

最常用的是按类通知。它适用于通用建议，如事务顾问。这些通知不依赖于被代理对象的状态，也不添加新的状态；它们只是作用于方法和参数。

每实例通知适用于引介，以支持混合体。在这种情况下，建议会为代理对象添加状态。

在同一个 AOP 代理中，可以混合使用共享通知和按实例通知。

### 5.3.2. [Spring 中的通知类型](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-introduction-advice-types)

Spring 提供了几种开箱即用的通知类型，并可扩展以支持任意通知类型。让我们来看看基本概念和标准通知类型。

#### 5.3.2.1. [Around 通知的拦截](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3111)

Spring 中最基本的通知类型是围绕通知的拦截。

Spring 符合 AOP 联盟关于使用方法拦截的周围建议的接口。实现周围建议的方法拦截器应实现以下接口：

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

MethodInterceptors 提供了与其他 AOP 联盟兼容的 AOP 实现的互操作性。本节余下部分讨论的其他建议类型以 Spring 特有的方式实现了常见的 AOP 概念。虽然使用最具体的建议类型有其优势，但如果您可能希望在其他 AOP 框架中运行该方面，请坚持围绕建议使用 MethodInterceptor。请注意，目前各框架之间还不能互操作，而且 AOP 联盟目前也没有定义点切接口。

#### 5.3.2.2. [Before 通知](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3148)

更简单的通知类型是`Before 通知`。它不需要 MethodInvocation 对象，因为它只会在进入方法之前被调用。

`Before 通知`的主要优点是不需要调用 proceed() 方法，因此也就不会在无意中导致拦截器链无法继续。

MethodBeforeAdvice 接口如下所示。(*Spring框架的API设计理论上支持在字段操作之前提供“前置通知”（field before advice），也就是说，可以在字段被访问或修改之前执行一些自定义的逻辑。然而，通常情况下，Spring框架主要用于处理方法的拦截（field interception），而不是字段的拦截。因此，尽管技术上Spring的API允许这种设计，但实际上Spring框架不太可能实现这种字段级别的前置通知功能。简而言之，Spring的API设计上是支持字段级别的前置通知的，但实际上Spring更倾向于处理方法级别的拦截，并且不太可能去实现字段级别的拦截功能*）。

```java
public interface MethodBeforeAdvice extends BeforeAdvice {

    void before(Method m, Object[] args, Object target) throws Throwable;
}
```

请注意，返回类型是 void。`Before 通知`可以在连接点执行前插入自定义行为，但不能更改返回值。如果 before 建议抛出异常，将中止拦截器链的进一步执行。异常会沿着拦截器链向上传播。如果该异常未被选中，或在被调用方法的签名上，它将直接传递给客户端；否则，它将被 AOP 代理封装在一个未被选中的异常中。

在 Spring 中，before 建议是一个示例，它会统计所有正常返回的方法：

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

如果连接点抛出异常，则在连接点返回后调用 Throws 建议。Spring 提供类型化的抛出建议。请注意，这意味着 org.springframework.aop.ThrowsAdvice 接口不包含任何方法：它是一个标签接口，用于标识给定对象实现了一个或多个类型化抛出提示方法。这些方法应为

```java
afterThrowing([Method], [args], [target], subclassOfThrowable) 
```

只需要最后一个参数。因此，根据建议方法是否对方法和参数感兴趣，可以有一到四个参数。以下是抛出建议的示例。

如果抛出 RemoteException（包括子类），将调用此建议：

```java
public  class RemoteThrowsAdvice implements ThrowsAdvice {

    public void afterThrowing(RemoteException ex) throws Throwable {
        // Do something with remote exception
    }
}
```

如果抛出 ServletException，将调用以下建议。与上述建议不同的是，它声明了 4 个参数，因此可以访问调用的方法、方法参数和目标对象：

```java
public static class ServletThrowsAdviceWithArguments implements ThrowsAdvice {

    public void afterThrowing(Method m, Object[] args, Object target, ServletException ex) {
        // Do something will all arguments
    }
}
```

最后一个示例说明了如何在处理 RemoteException 和 ServletException 的单个类中使用这两种方法。在一个类中可以组合任意数量的抛出建议方法。

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

Spring 中的返回后建议必须实现`org.springframework.aop.AfterReturningAdvice`接口，如下所示：

```java
public interface AfterReturningAdvice extends Advice {

    void afterReturning(Object returnValue, Method m, Object[] args, Object target) 
            throws Throwable;
}
```

返回后通知可以访问返回值（不能修改）、调用方法、方法参数和目标。

下面的返回建议会统计所有未抛出异常的成功方法调用次数：

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

返回建议后，可与任何切点一起使用。

#### 5.3.2.5. [Introduction 通知](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3242)

Spring 将引介通知视为一种特殊的拦截通知。

引介需要一个`IntroductionAdvisor`和一个`IntroductionInterceptor`，实现下面的接口：

```java
public interface IntroductionInterceptor extends MethodInterceptor {

    boolean implementsInterface(Class intf);
}
```

从 AOP 联盟 `MethodInterceptor` 接口继承的 `invoke()` 方法必须实现引介：也就是说，如果被调用的方法位于`引介接口`上，则引介拦截器负责处理方法调用--它不能调用 `proceed()`。

`引介通知`不能与任何点切点一起使用，因为它只适用于类而非方法级别。只有`InterceptionIntroductionAdvisor`才能使用引介建议，它有以下方法：

```java
public interface InterceptionIntroductionAdvisor extends InterceptionAdvisor {

    ClassFilter getClassFilter();

    IntroductionInterceptor getIntroductionInterceptor();

    Class[] getInterfaces();
}
```

没有 `MethodMatcher`，因此也没有与导入建议相关的 Pointcut。只有类过滤才符合逻辑。

`getInterfaces()` 方法返回该顾问引介的接口。

让我们看看 Spring 测试套件中的一个简单示例。假设我们要为一个或多个对象引介以下接口：

```java
public interface Lockable {
    void lock();
    void unlock();
    boolean locked();
}
```

这说明了一个 mixin。我们希望能够将建议对象转换为 Lockable，无论其类型如何，并调用锁定和解锁方法。如果我们调用 lock() 方法，我们希望所有设置器方法都抛出 LockedException。这样，我们就可以添加一个方面，提供使对象不可变的能力，而对象对此一无所知：这就是 AOP 的一个很好的例子。

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

我们可以非常简单地应用这个顾问：它不需要任何配置。(不过，配置是必要的：没有 `IntroductionAdvisor` 就无法使用 `IntroductionInterceptor`）。与导入程序一样，该顾问必须按实例设置，因为它是有状态的。我们需要为每个建议对象创建不同的 `LockMixinAdvisor` 实例，因此也需要为每个建议对象创建不同的 `LockMixin` 实例。顾问包含被建议对象的部分状态。

我们可以使用 `Advised.addAdvisor()` 方法或（推荐方法）XML 配置，像其他顾问一样，以编程方式应用该顾问。下面讨论的所有代理创建选择，包括 "自动代理创建器"，都能正确处理介绍和有状态的混合体。

## 5.4. [Spring 中的 Advisor](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3395)

在 Spring 中，`Advisor`是是对切面的一种模块化封装。`Advisor`通常包含**一个通知**和**一个切点**。

`org.springframework.aop.support.DefaultPointcutAdvisor` 是最常用的顾问类。例如，它可与 `MethodInterceptor`、`BeforeAdvice` 或 `ThrowsAdvice` 一起使用。

在同一个 AOP 代理中，可以混合使用 Spring 中的`Advisor`和通知类型。例如，您可以在一个代理配置中使用`环绕通知`、``抛出通知``和`在通知之前`的拦截：Spring 会自动创建必要的创建拦截器链。

## 5.5. [使用 ProxyFactoryBean 创建 AOP 代理](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-pfb)



### 5.5.1[. 基础](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-pfb-1)

### 5.5.2. [JavaBean 属性](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-pfb-2)

### 5.5.3. [代理接口](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3534)

### 5.5.4. [代理类](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3601)

## 5.6. [方便的代理创建](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-tfb)

### 5.6.1. [TransactionProxyFactoryBean](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3647)

### 5.6.2. [EJB 代理](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3728)

## 5.7. [简明的代理定义](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-concise-proxy)

## 5.8. [使用 ProxyFactory 以编程方式创建 AOP 代理](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-prog)

## 5.9. [操作建议对象](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3794)

## 5.10. [使用 “autoproxy” 工具](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-autoproxy)

### 5.10.1. [自动代理 bean 定义](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-autoproxy-choices)

#### 5.10.1.1. [BeanNameAutoProxyCreator](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3884)

#### 5.10.1.2. [DefaultAdvisorAutoProxyCreator](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3907)

#### 5.10.1.3. [AbstractAdvisorAutoProxyCreator](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e3955)

### 5.10.2. [使用元数据驱动的自动代理](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-autoproxy-metadata)

## 5.11. [使用 TargetSources](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-targetsource)

### 5.11.1. [热插拔目标源](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-ts-swap)

### 5.11.2. [池化目标源](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-ts-pool)

### 5.11.3. [原型“目标源](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-ts-prototype)

## 5.12[. 定义新的 Advice 类型](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#aop-extensibility)

## 5.13. [延伸阅读和资源](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e4201)

## 5.14. [路线图](https://docs.spring.io/spring-framework/docs/1.1.x/reference/aop.html#d0e4223)