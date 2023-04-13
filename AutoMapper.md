# AutoMapper

## 一. 什么是自动映射器

AutoMapper 是一个对象-对象映射器。对象-对象映射的工作原理是将一种类型的输入对象转换为另一种类型的输出对象。AutoMapper 之所以有趣，是因为它提供了一些有趣的约定，以消除弄清楚如何将类型 A 映射到类型 B 的繁琐工作。只要类型 B 遵循 AutoMapper 的既定约定，映射两种类型所需的配置几乎为零。

## 二. 为什么使用AutoMapper

AutoMapper是一个对象映射工具，它可以帮助我们将一个对象的属性值自动映射到另一个对象上。在实际开发中，我们经常会遇到需要将数据从一个对象转换到另一个对象的情况，这时候使用 AutoMapper 可以减少手动编写转换代码的工作量，提高开发效率。

具体来说，使用 AutoMapper 有以下好处：

1. 减少重复的转换代码：如果我们需要对多个不同的对象进行相同的转换操作，那么我们不需要为每个对象都编写一遍相同的转换代码，而是可以使用 AutoMapper 提供的配置文件来统一管理。

2. 简化代码结构：使用 AutoMapper 可以让代码更加简洁易懂，避免出现大量繁琐的属性赋值操作。

3. 易于维护和修改：当需要修改转换规则时，只需要修改 AutoMapper 的映射配置文件即可，而不需要在代码中逐个修改转换操作。

## 三. 部署AutoMapper

**在项目中引入**：

```
Install-Package AutoMapper
```

**配置依赖注入：**

AutoMapper通常与依赖注入一起使用，要将AutoMapper集成到ASP.NET Core中。

![4](https://img-blog.csdnimg.cn/5e1b2094cdc84dbb9d6fbc309c163505.png)

**定义对象规则：**

我们可以通过继承 Profile 类并实现其中的 `CreateMap` 方法来定义对象之间的映射规则。

`CreateMap `方法有两个参数，第一个参数是源类型，第二个参数是目标类型。

![3](https://img-blog.csdnimg.cn/65b5313ed5f54a26bb796be0d2534a72.png)

**在应用程序中应用：**

接下来，我们就可以使用`.ProjectTo`方法进行对象映射了。假设有一个名为 `UserQuestionDto` 的类，我们希望将 `UserQuestion` 类型的数据映射到 `UserQuestionDto` 类型，可以按照如下方式使用 **.ProjectTo**方法：

![5](https://img-blog.csdnimg.cn/90f85e950eb74269824766d803e0c6e6.png)

上述代码中，_context.userQuestions 是一个 IQueryable<UserQuestion> 对象，`.ProjectTo<UserQuestionDto>(_mapper.ConfigurationProvider)` 将其映射成 IQueryable<UserQuestionDto>，然后通过`.ToList()`方法将结果转换成 List<UserQuestionDto> 对象。

需要注意的是，`.ProjectTo` 方法需要使用 AutoMapper 配置提供程序来指定映射细节。这个配置提供程序可以通过调用 `_mapper.ConfigurationProvider` 来获得。此外，`.ProjectTo` 方法还支持一些高级选项，例如投影到聚合类型、从多个源对象映射等功能。

也可以使用Mapper实例：

```
_mapper.Map<CreatePeopleDto>(result)
```

将result的对象属性映射到CreatePeopleDto中。需要注意的是，这行代码的精确行为取决于所使用的AutoMapper实例Mapper的对象规则配置。这个配置决定了源对象和目标对象之间的属性如何映射，包括任何自定义映射或命名约定。

```
_mapper.Map(updateQuestion, userQuestion);
```

这段代码的目的是使用 `updateQuestion` 对象提供的新值来更新 `userQuestion` 对象。映射库会确保只复制指定的属性，并确保类型匹配。


