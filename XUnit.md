# XUnit

## 1.单元测试

单元测试是指对软件中最小可测试单元进行的测试，通常是一个函数或一个模块。单元测试旨在验证每个代码单元的行为是否符合预期，以便及早发现和修复缺陷，提高代码质量。

单元测试有以下特点：

- 每个测试用例都针对一个具体的代码单元。
- 在运行测试之前，应使用各种技术（如mock）隔离被测代码单元与其它代码单元的相互作用。
- 测试应该是自动化的。
- 单元测试应该尽可能地快速运行，以便在开发过程中经常运行。

单元测试实施方法：

- 选择适当的单元测试框架（如JUnit、NUnit等）
- 编写测试用例和断言
- 运行测试并检查结果
- 根据测试结果修改代码并重新运行测试，直到测试全部通过为止

## 2.集成测试

集成测试是指将多个代码单元组合在一起进行测试，以验证它们在整个系统中的协作是否正常。集成测试通常在单元测试完成之后进行。

集成测试有以下特点：

- 测试用例涉及多个代码单元。
- 通常需要手动测试，但也可自动化。
- 集成测试应该尽可能早地进行。

集成测试实施方法：

- 确定集成测试的范围和策略。
- 使用各种技术（如mock）隔离测试单元之间的相互作用。
- 执行测试并记录结果。
- 根据测试结果修改代码并重新运行测试，直到测试全部通过为止。

## 3.系统测试

系统测试是指对整个软件系统进行的测试，以验证其是否符合用户需求和规格说明书。系统测试通常在所有其他测试完成之后进行。

系统测试有以下特点：

- 测试用例覆盖整个系统。
- 需要手动测试。
- 系统测试应该在软件发布之前进行。

系统测试实施方法：

- 确定系统测试的范围和策略。
- 执行黑盒测试（只关注输入和输出，不考虑内部实现）和白盒测试（检查软件内部结构）。
- 记录测试结果并汇总缺陷。
- 修复缺陷并重新运行测试，直到测试全部通过为止。

## 4.XUnit

编写第一个测试程序：

```csharp
using Xunit;

namespace MyFirstUnitTests
{
    public class UnitTest1
    {
        [Fact]
        public void PassingTest()
        {
            Assert.Equal(4, Add(2, 2));
        }

        [Fact]
        public void FailingTest()
        {
            Assert.Equal(5, Add(2, 2));
        }

        int Add(int x, int y)
        {
            return x + y;
        }
    }
}
```

运行后可拿到：

![10](https://img-blog.csdnimg.cn/4b4fb21f0ed14eeba5b67b3536784bc2.png)

你可能想知道为什么你的第一个单元测试使用命名的属性 `[Fact]`而不是具有更传统名称（如 Test）的属性。xUnit.net 支持两种主要类型的单元测试：事实和理论`(Fact｜Theory)`。在描述事实和理论之间的差异时，我们喜欢说：

> **事实**是永远正确的测试。他们测试不变的条件。
> 
> **理论**是仅适用于一组特定数据的测试。

让我们为现有事实添加一个理论（包括一些不良数据，因此我们可以看到它失败了）：

```csharp
[Theory]
[InlineData(3)]
[InlineData(5)]
[InlineData(6)]
public void MyFirstTheory(int value)
{
    Assert.True(IsOdd(value));
}

bool IsOdd(int value)
{
    return value % 2 == 1;
}
```

![11](https://img-blog.csdnimg.cn/80eb822ab2024f82b3e60834092f2f32.png)

## 5.集成在框架测试

引入依赖：

```
install package Xunit
install package Shouldly
```

配置依赖注入容器：

![13](https://img-blog.csdnimg.cn/9cd339690eec47aba98dd0349f696647.png)

配置TestUtilBase对容器的bean进行统一的生命周期管理

![12](https://img-blog.csdnimg.cn/832b42eeea58414fb1b1ef97c152b773.png)

在类中定义重载run以及RunWithUnitOfWork方法，根据不同的请求方式以及参数定制不同的请求接口方法，视情况而定：

![17](https://img-blog.csdnimg.cn/7658d62d301b47a9a52db062294f0043.png)

把各个组件注册到容器当中

![14](https://img-blog.csdnimg.cn/8c0941611ba744649049d864baaa31b9.png)

使用Autofac库注册把module注册到TestBase测试类当中，进行测试模块的组件。

注册模拟用户登陆请求上下文的测试：

```csharp
containerBuilder.RegisterInstance(new TestCurrentUser()).As<ICurrentUser>();
containerBuilder.RegisterInstance(Substitute.For<IMemoryCache>()).AsImplementedInterfaces();
containerBuilder.RegisterInstance(Substitute.For<IHttpContextAccessor>()).AsImplementedInterfaces();
```

最后用_testTopi为appsetting.json文件生成一个以此为后缀的配置文件，数据环境由_dataName生成一个模拟数据库环境（模拟环境并无真实环境数据，仅包含数据库结构）

定义一个测试集类，每一个测试类和测试方法都必须属于一个测试集合，而且同一个测试集合内的所有测试会在相同的测试上下文中执行。因此，使用 [Collection("...")] 注解可以将多个相关的测试组织在同一个测试集合中，从而方便进行共享状态的测试。

Demo：

![16](https://img-blog.csdnimg.cn/ff768eb70e8a48f7b0e47a58afc770f6.png)

在所有测试完成时关闭数据库：

![15](https://img-blog.csdnimg.cn/f20ba520d614474abc1c4d9da1945169.png)


