# 用aop和注解重写bot


## ？前言

之前就有个想法把bot用spring重写一下。了解完一些基础的东西后终于能开写了。

## ？基于注解加反射

### 推

这一点是看了[`诺天的博客`](https://nuotian.furry.pro/blog/archives/244)。

### 思路

- 1. 写个自定义注解用以标记方法

- 2. 利用spring自带的`SpringUtil.getApplicationContext().getBeansWithAnnotation(Controller.class)`来获取所有被@Controller标记的类的bean。

     【注： 这里的SpringUtil是你自己的写的任意实现了`ApplicationContextAware`接口的类的名字。并且注解类型必须是Spring里的注解（@Controller,@Component等，不然获取不到bean

- 3. 利用反射的相关方法获取所有方法。再做个遍历筛选出带有自定义注解的方法加入一个列表。

- 4. 监听器中调用这个列表进行逻辑判断，确定什么时候调用并传入方法需要的实参

- 5. 这样发消息的时候就能实现自动判断并执行

### 补充

博客里写的已经很详细了。但是总感觉有些功能还可以再补充一下：

- 逻辑判断和多种权限：

  一：[诺天](https://nuotian.furry.pro/blog/)写的消息的逻辑判断是放在注解中，再和bot接收到的消息进行对比来实现的。但是这样

  逻辑一复杂就会让监听器太冗杂。

  二：一个注解只能让一个方法用于Group或Friend，没法多个权限

  解决方案：

  - (一)让注解可重复，增加注解元素

    @Event注解：

    ```java
    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.RUNTIME)
    @Repeatable(Events.class) // 元注解，使该注解能重复。@Events是含有 返回Event[] 的方法的注解
    public @interface Event
    {
        EventEnum eventType() default EventEnum.NONE; // EventEnum是个枚举类
        boolean authority() default false; // 第一层判断,是否开启这个方法
        boolean judgeByMethod() default false; // 复杂判断时开启这个
        String message() default ""; // 第二层判断,触发条件
    }
    ```

    @Events注解：

    ```java
    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface Events
    {
        Event[] value();
    }
    ```

    >  注意@Events注解必须写@Target，且元素类型是@Event的子集

  - (二)除了message的判断，再在反射时把`MessageChain`传进去。

    - 复杂的方法把`judgeByMethod`设为true，让方法利用`MessageChain`获得消息进行逻辑处理
    - 简单的方法把`judgeByMethod`设为false，利用`message`注解元素在监听器中进行equals判断

  - 结果：利用`authority`和`message`来灵活分配权限(虽然也不灵活就是了)；`judgeByMethod`实现判断权限转移

## ？基于注解和切面编程(Aop)

因为Aop是根据反射开发的，于是我想能不能用Aop写一个。

###  思路

利用Aop对被指定注解标记的方法进行增强。在方法执行前进行一个判断。

### 代码

Aop部分：

```java
类上标注@Aspect注解

// 切入点 实际切入的方法
// 注意是对Events切入，因为多个Event注解其实在获取的时候会转换成@Events({@Event(xx), @Event(xx)})
@PointCut("@annotation(com.yanfang.animabotaop.annotations.Events)")
public void judgeMethodPointCut() {} // 切入方法？

// 环绕通知
// && @annotation(events)是传入参数给通知方法的一种手段，可以省去声明
// around方法参数名要和@Around中一致
// ProceedingJoinPoint 是环绕通知特有的，是JoinPoint的子接口，其他通知为JoinPoint。
// 该参数的功能很强大
@Around("judgeMethodPointCut() && @annotation(events)")
public void around(@NotNull ProceedingJoinPoint jpt, @NotNull Events events) throws Throwable
{
    for (Event eventAnnotation : events.value())
    {
        // 获取实参值（注意是“值”
        Object[] argValues = jpt.getArgs();
        // 获取传入的event，这里默认所有方法第一个参数都是MessageEvent
        MessageEvent event = (MessageEvent) argValues[0];
        // 进行逻辑判断，这里和用反射时一样
        if (eventAnnotation.authority())
            {
                System.out.println("已授权");
                // 有复杂逻辑就将judgeByMethod设为true， 交给方法进行复杂逻辑处理
                if (eventAnnotation.judgeByMethod() || (eventAnnotation.message().equals(event.getMessage().contentToString())))
                {
                    if (event.getSubject() instanceof Group && eventAnnotation.eventType() == EventEnum.GROUPEVENT)
                    {
                        joinPoint.proceed(argValues);
                    }
                    else if (event.getSubject() instanceof Friend && eventAnnotation.eventType() == EventEnum.MASTEREVENT)
                    {
                        joinPoint.proceed(argValues);
                    }
                    else if (event.getSubject() instanceof Member && eventAnnotation.eventType() == EventEnum.TEMPEVENT)
                    {
                        joinPoint.proceed(argValues);
                    }
                }
            }
    }
}
```



监听器方法方面：

```java
@EventHandler
public ListeningStatus onMessage(@NotNull MessageEvent event) throws Exception
{
    @NotNull Test test = SpringUtil.getBean(Test.class);
    test.test(event);
    return ListeningStatus.LISTENING;

}
```

测试类：

```java
@Controller
public class Test
{
    @Events(@Event(eventType = EventEnum.MASTEREVENT, authority = true, message = "test"))
    public void test(MessageEvent event)
    {
        if (event == null) System.out.println("event 为空");
        else event.getSubject().sendMessage("get");
    }
}
```

结果：

authority为true时发送了get， 为false时没发送get。说明执行了环绕通知

### 坑

- 注意Aop和反射不同的一点是：反射可以帮助传入实参；而Aop只能获取方法被传入的实参进行处理，而不能传入参数（毕竟Aop的原衷是不影响方法本身的情况下进行对方法的增强。
  - 而实参在监听器方法里传入
- 这里不能通过获取所有Event注解， 再分别进行处理，而需要获取@Events，再通过它的value()元素获取所有的@Event注解 [即使表面上可以在方法上多写几个@Event注解。但实际是`@Events({@Event(...), @Event(...)})`,但是用反射好像可以直接获取多个@Event注解进行遍历处理
- 注意通知的方法`around()`传参不能直接传入，而是需要在`@Around`中传入。否则会`报空`。（其实是获取方法上的@Events注解，然后传进around()方法

## ？比较与总结

总的来说，各有优缺。

- 反射：先进行判断，然后再根据判断结果来决定是否执行方法
- Aop：要先调用方法（注意还并没有实际执行），进而会触发通知，用通知中的方法进行逻辑判断



-  反射代码较多，但是处理上更灵活，不必先调用方法
-  Aop要先调用方法，才会触发通知，但是代码量更少，更简洁
