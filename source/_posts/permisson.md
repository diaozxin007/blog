---
title: 如何利用 Spring Hibernate 高级特性设计实现一个权限系统
date: 2019-05-11 21:23:41
tags: [java,权限]
---

![keepout](//img.xilidou.com/img/alex-pudov-588871-unsplash%202.jpg?x-oss-process=image/resize,p_40)

我们的业务系统使用了一段时间后，用户的角色类型越来越多，这时候不同类型的用户可以使用不同功能，看见不同数据的需求就变得越来越迫切。
如何设计一个可扩展,且易于接入的权限系统.就显得相当重要了。结合之前我实现的的权限系统，今天就来和大家探讨一下我对权限系统的理解。

这篇文章会从权限系统业务设计，技术架构，关键代码几个方面，详细的阐述权限系统的实现。

<!--more-->

## 背景

权限系统是一个系统的基础功能，但是作为创业公司，秉承着快比完美更重要原则，老系统的权限系统都是硬编码在代码或者写在到配置文件中的。随着业务的发展，如此简陋的权限系统就显得捉襟见肘了。开发一套新的，强大的权限系统就提上了日程。

这里有两个重点：

* 业务系统已经运行一段时间积累了可观的代码和接口了，新的权限系统权在设计之初的一个要求就是，尽量减少权限系统对原有业务代码的入侵。（为了达成这个目的，我们会大量的使用 spring、springboot、jpa  以及 hibernate  的高级特性）
* 系统要易于使用，可以由业务方自行进行配置。

## 需求

权限系统需要支持功能权限和数据权限。

### 功能权限

所谓功能权限，就是指，拥有某种角色的用户，只能看到某些功能，并使用它。实现功能权限就简化为：

* 页面元素如何根据不同用户进行渲染
* API 的访问权限如何根据不同的用户进行管理

### 数据权限

所谓数据权限是指，数据是隔离的，用户能看到的数据，是经过控制的，用户只能看到拥有权限的某些数据。

比如，某个地区的 leader 可以查看并操作这个地区的所有员工负责的订单数据，但是员工就只能操作和查看自己负责的的订单数据。

对于数据权限，我们需要考虑的问题就抽象为，

1. 数据的归属问题：数据产生以后归属于谁？
2. 确定了数据的归属，根据某些配置，就能确定谁可以查看归属于谁的数据。

## 业务设计

经过上面的分析，我们可以抽象出以下几个实体：

### 功能权限

* 用户
* 角色
* 功能
* 页面元素
* API 信息

我们知道，对于一某个功能来说，它是由若干的前端元素和后端 API 组成的。

比如“合同审核” 这个功能就包括了，“查看按钮”、“审核按钮” 等前端元素。

涉及的 api 就可能包含了 `contract` 的 `get` 和 `patch` 两个 Restful 风格的接口。

抽象出来就是：在权限系统中若干前端元素和后端 API 组成了一个功能。

具体的关系，就是如下图：

![permission-er](//img.xilidou.com/img/permissin-er.png)

### 数据权限

具体每个系统的数据权限的实现有所不同，我们这里实现的数据权限是依赖于公司的组织架构实现的，所有涉及到的实体如下：

* 用户
* 数据权限关系
* 部门
* 数据拥有者
* 具体数据（订单，合同）

这里需要说明一下，要接入数据权限，首先需要梳理数据的归属问题，数据归属于谁？或者准确的来说，数据属于哪个数据拥有者，这个数据拥有者属于哪个部门。通过这个关联关系我们就可以明确，这个数据属于哪个部门。

对于数据的使用用户，来说，就需要查询，这个用户可以查看某个模块的某个部门的数据。

这里需要说明的是，不同的系统的数据权限需要具体分析，我们系统的数据权限是建立在公司的组织架构上的。

本质就是：

* 数据归属于某个数据拥有者
* 用户能够看到该数据拥有者的数据

具体的关系图如下：

![date-permission](//img.xilidou.com/img/datepermission.png)

注意，实际上用户和数据拥有者都是同一个实体 User 表示，只是为了表述方便进行了区分。

## 实现的技术难点

### Mysql 中树的储存

可以看出来，我们的功能和组织架构都是典型的树形结构。

我们最常见的场景如下

* 查询某个功能，及其所有子功能。
* 查询某个部门，及其所有子部门的所属员工。

抽象以后就是查询树的某个节点，和他的所有子节点。

为了便于查询，我们可以增加两个冗余字段，一个是 `parent_id` ,还有一个是 `path`。

* parent_id 很好理解，就是父节点的 id；
* path 指的是，这个节点，路径上的 id 的。使用'.'进行分隔的一个字符串。 比如

```shell
            A
           / \
          B   C
         /\   /\
        D  E F  G
                /\
               H  I
```

对于 D 的 path 就是 `(A.id).(B.id).` 这要的好处的就是通过 `sql` 的 `like` 的语句就能快速的查询出某个节点的子节点。

比如要获取节点 C 的所有子节点:

```slq
Select * from user where path like (A.id).(C.id).%
```

一次查询可以获取所有子节点，是一种查询友好的设计。如果需要我们可以为 `path` 字段增加索引，根据索引的左值定律，这样的 like 查询是可以走索引的。提升查询效率。

### 快速的自动的获取 API 信息

我们知道 `Spirng mvc` 在启动的时候会扫描被 `@RequestMapping` 注解标记的方法，并把数据放在 `RequestMappingHandlerMapping` 中。所以我们可以这样：

```java

@Componet
public class ApiScanSerivce{

    @Autoired
    private RequestMappingHandlerMapping requestMapping;

    @PostConstruct
    public void update(){

        Map<RequestMappingInfo,HandlerMethed> handlerMethods = requestMapping.getHandlerMethods();
        for(Map.Entry RequestMappinInfo,HandlerMethod) entry: handlerMethods.entrySet(){
            // 处理 API 上传的相关逻辑
            updateApiInfo();
        }

    }

}

```

获取项目的所有 http 接口。这样我们就可以遍历处理项目的接口数据。

### 描述一个 API

```java
public class ApiInfo{

    private Long id;
    private String uri; // api 的 uri
    private String method; //请求的 method：eg： get、 post、 patch。
    private String project; // 这组 api 属于哪一个 web 工程。
    private String signature; //方法的签名
    private Intger status; // api 状态
    private Intger whiteList; // 是否是白名单 api 如果是就不需过滤

}
```

其中方法的签名生成的算法伪代码:

```java
signature = className + "#" + methodName +"(" + parameterTypeList+")"
```

### 用户的权限数据

首先我们定义的用户权限数据如下：

```java
@Data
@ToString
public class UserPermisson{

    //用户可以看到的前端元素的列表
    private List<Long> pageElementIdList;

    //用户可以使用的 API 列表
    private List<String> apiSignatureList;

    //用户不同模块的数据权限 的 map。map 的 key 是模块名称，value 是这个能够看到数据属于那些用户的列表
    private Map<String,List<Long>> dataAccessMap；
}
```

### 利用 Spring 特性实现功能权限

对于如何使用 Spring 实现方法拦截，很自然的就像到了使用拦截器来实现。考虑到我们这个权限的组件是一个通用组件，所以就可以写一个抽象类，暴露出`getUid(HttpServletRequest requset)` 用户获取使用系统的 `userId`,以及 `onPermission(String msg)`留给业务方自己实现，没有权限以后的动作。

```java
public abstract class PermissonAbstractInterceptor extends HandlerInterceptorAdapter{

    protected abstarct long getUid(HttpServletRequest requset);

    protected abstract onPermession(String str) throws Exception;

    @Override
    public boolean preHandler(HttpServletRequest request,HttoServletResponse respponse,Object handler) throws Excption{
        // 获取用户的 uid
        long uid = getUid(request);

        // 根据用户 获取用户相关的 权限对象
        UserPermisson userPermission = getUserPermissonByUid(uid);

        if(inandler instanceof HanderMethod){
            //获取请求方的签名
            String methodSignerture = getMethodSignerture(handler);

            if(!userPermisson.getApiSignatureList().contains(methodSignerture)){

                onPermession("该用户没有权限");

            }
        }

    }

}
```

以上的代码只是提供一个思路。不是真实的代码实现。

所以接入方就只需要继承这个抽象方法，并实现对应的方法，如果你使用的是 Springboot 的，只需要把实现的拦截器注册到拦截器里面就可以使用了：

```java
@Configuration
public class MyWebAppConfigurer extends WebMvcConfigurerAdapter {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(permissionInterceptor);
        super.addInterceptors(registry);
    }

}
```

### 利用 Hibrenate 特性实现数据权限

通过上面的代码可以看出来，功能权限的实现，基本做到了没有侵入代码。对于数据权限的实现的原则还是尽量少的减少代码的入侵。

我们默认代码使用 Java 经典的 Controller、Service、Dao 三层架构。 主要使用的技术 Spring Aop、Jpa 的 filter，基本的实现思路如下图：

![date permission](//img.xilidou.com/img/jpa-filter.png)

基本的思路如下：

1. 用户登录以后，获取用户的数据权限相关信息。
2. 把相关信息权限系统放入 ThreadLocal 中。
3. 在 Dao 层中，从 ThreadLocal 中获取权限相关的权限数据。
4. 在 filter 中填充权限相关数据。
5. 从 Hibernate 上下文中取出 Session。
6. 在 Session 上添加相关 filter。

通过图片我们可以看出，我们基本不需要对 Controller、Service、Dao 进行修改，只需要按需实现对应模块的 filter。

看到这里你可能觉得"嚯~~",还有这种操作？我们就看看代码是怎么具体实现的吧。

1. 首先需要在 Entity 上写一个 Filter,假设我们写的是订单模块。

```java
@Entity
@Table(name = "order")
@Data
@ToString
@FilterDef(name = "orderOwnerFilter", parameters = {@ParamDef name= "ownerIds",type = "long"})
@Filters({@Filter name= "orderOwnerFiler", condition = "ownder in (:ownerIds)"})
public class order{
    private Long id;
    private Long ownerId;
    //其他参数省略
}
```

2. 写个注解

```java
@Retention(RetentinPolicy.RUNTIME)
@Taget(ElementType.METHOD)
public @interface OrderFilter{
}
```

3. 编写一个切面用于处理 Session、datePermission、和 Filter

```java
@Component
@Aspect
public class OrderFilterAdvice{
    @PersistenceContext
    private EntityManager entityManager;
    @Around("annotation(OrderFilter)")
    pblict Object doProcess (ProceedingJoinPoint joinPonit) throws ThrowableP{
        try{
            //从上下文里面获取 owerId，这个 Id 在 web 中就已经存好了
            List<Long> ownerIds = getListFromThreadLocal();
            //获取查询中的 session
            Session session = entityManager.unwrap(Session.class);
            // 在 session 中加入 filter
            Filter filter = unwrap.enableFilter("orderOwnerFilter");
            // filter 中加入数据
            filter.setParameterList("ownerIds"，ownerIds)
            //执行 被拦截的方法
            return join.proceed();
        }catch(Throwable e){
            log.error();
        }finally{
            // 最后 disable filter
           entityManager.unwrap(Session.class).disbaleFilter("orderOwnerFilter");
        }
    }
}
```

    这个拦截器，拦截被打了 `@OrderFilter` 的方法。

### 易于接入

为了方便接入项目，我们可以将涉及到的整套代码封装为一个 `springboot-starter` 这样使用者只需要引入对应的 starter 就能够接入权限系统。  

## 总结

权限系统随着业务的发展，是从可以没有逐渐变成为非常重要的模块。往往需要接入权限系统的时候，系统已经成熟的运行了一段时间了。大量的接口，负责的业务，为权限系统的接入提高了难度。同时权限系统又是看似通用，但是定制的点又不少的系统。

设计套权限系统的初衷就是，不需要大量修改代码，业务方就可方便简单的接入。
具体实现代码的时候，我们充分利用了面向切面的编程思想。同时大量的使用了 `Spring`、`Hibrenate`框架的高级特性，保证的代码的灵活，以及横向扩展的能力。

看完文章如果你发现有疑问，或者更好的实现方法，欢迎留言与我讨论。