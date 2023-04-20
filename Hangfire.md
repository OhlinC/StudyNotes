# Hangfire

## 一.Hangfire的概念

Hangfire是一种用于在.NET应用程序中实现延迟任务和后台作业的开源框架。它提供了一个简单易用的方法来执行异步、长时间运行的任务，例如发送电子邮件、生成报告、处理数据等。Hangfire具有许多强大的特性，包括灵活的调度器、持久化存储、重试机制、故障转移、Web界面等，可以帮助开发人员更好地管理和监控后台作业。Hangfire支持多种存储后端，例如SQL Server、Redis、MongoDB等，同时还提供了针对ASP.NET Core和ASP.NET MVC的特定集成方式。

Hangfire 允许您以一种非常简单但可靠的方式在请求处理管道之外启动方法调用。这些方法调用在*后台线程*中执行，称为*后台作业*。

<img title="" src="https://docs.hangfire.io/en/latest/_images/hangfire-workflow.png" alt="Hangfire 工作流程" width="641">

## 二.BackgroundJob类

BackgroundJob是Hangfire库中的一个类，用于创建和管理后台任务。它允许我们在ASP.NET应用程序中快速轻松地将长时间运行的任务移至后台处理，而不必影响用户体验或阻塞Web服务器上的请求。使用BackgroundJob，我们可以将需要执行的代码封装在一个方法中，并将其传递给Hangfire，以便在后台异步执行。这个类提供了一系列的方法来管理后台任务，如添加、删除、重新安排等。

包含方法：

1. Enqueue：将一个方法加入到队列中等待执行。

2. Schedule：按照指定的时间安排任务的执行。

3. Continuations：将一个方法与另一个方法关联，当第一个方法完成时，自动触发第二个方法。

4. Delete：删除一个已经在队列中等待或正在执行的任务。

5. Requeue：重新将一个失败或者被取消的任务加入到队列中等待执行。

6. UpdateExpirationTime：修改一个任务的过期时间。

7. GetJobParameter：获取任务中指定名称的参数值。

8. SetJobParameter：设置任务中指定名称的参数值。

9. Retry：重新尝试执行一个失败的任务。

10. Perform：手动执行某个任务。

这些方法可以通过 BackgroundJob 类的静态方法来调用。这些方法提供了对任务的操作和控制，让我们可以更加灵活地使用 Hangfire 处理后台任务。

## 三.集成至项目

导入依赖：

```
install package Hangfire.Core
install package Hangfire.AspNetCore
install package Hangfire.MySqlStorage
```

在Start.up中配置：

```
app.UseHangfireDashboard();//配置后台仪表盘
app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
            //将Hangfire仪表板添加到应用程序的路由中
            endpoints.MapHangfireDashboard();
        });
```

我们将通过一个扩展类用于hangfire注册服务集合，：

![Hangfire配置类](https://img-blog.csdnimg.cn/60e5a9a180ca40519561caf725aa5700.png)

通过`services.AddHangfire()` 将创建一个 Hangfire实例添加到服务集合中。然后指定表生成前缀为”hangfire_“具体生成表后为：

![hangfire生成表](https://img-blog.csdnimg.cn/773b58ea3b9044418b790afa1ae049f1.png)

通过此配置后我们就可以在应用程序通过BackgroundJob类进行后台任务管理了。

声明一个简单的后台任务

```
BackgroundJob.Enqueue(() => Console.WriteLine("Hello, world!"));
```

当执行过后可以在hangfire仪表盘中看到任务执行过后的状态：
![hangfire仪表盘执行后](https://img-blog.csdnimg.cn/ff7bac561f0c4d94892e47ae0bffea9d.png)

虽然hangfire在所有组件关闭后会自动关闭hangfire 但是还是推荐使用.Dispose方法做一个关闭标记。
