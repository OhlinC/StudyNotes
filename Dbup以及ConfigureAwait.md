# 1.Dbup

DbUp是一个开源的数据库部署工具，可以用于自动化数据库升级和版本控制。它支持多种数据库类型（如SQL Server、MySQL、PostgreSQL等），并提供了简单易用的命令行工具和API，方便开发人员集成到其CI/CD系统中。DbUp的主要功能包括解析并执行数据库脚本、管理数据库版本号、记录升级行为以及实现回滚等操作。

### 项目中部署：

**引入依赖：**

```dotnet
Install-Package DbUp
```

**配置Dbup：**

```dotnet
public void Run()
 {
     var upgrader = DeployChanges.To.MySqlDatabase("连接的数据库字符串")
                .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly())
                .LogToConsole()
                .Build();

        var result = upgrader.PerformUpgrade();

        if (!result.Successful)
        {
            Console.ForegroundColor = ConsoleColor.Red;
            Console.WriteLine(result.Error);
            Console.ResetColor();
            return -1;
        }

        Console.ForegroundColor = ConsoleColor.Green;
        Console.WriteLine("Success!");
        Console.ResetColor();
        return 0;
  }
```

**加入执行脚本：**

![1](https://img-blog.csdnimg.cn/d7fcc3a85fd746a5991a19feec7e5cdc.png)

**应用程序启动前加载：**

```dotnet
new DbRunner("数据源").Run()
```



### 补足：

- 入口点是`DeployChanges.To`
- 获取将要执行的脚本`GetScriptsToExecute()`
- 获取已执行的脚本`GetExecutedScripts()`
- 检查是否需要升级`IsUpgradeRequired()`
- 为什么任何新的迁移脚本创建版本记录而不能执行它们`MarkAsExecuted`
- 尝试连接到数据库`TryConnect()`
- 执行数据库升级`PerformUpgrade()`
- 日志脚本输出`LogScriptOutput()`

# async/await的ConfigureAwait

## 1.什么是同步上下文

同步上下文（SynchronizationContext）是一个抽象类，用于协调多个线程的执行。同步上下文通常用于将异步方法中的回调函数切换回主线程执行。

在使用异步编程模型时，可能会遇到需要执行长时间操作的情况，这时候就可以使用异步方法来实现。异步方法将长时间操作放入另外一个线程中执行，而当前线程则可以继续执行其他操作。当异步方法完成后，它会通过回调函数将结果返回给主线程。

但是，在一些情况下，异步方法的回调函数需要在主线程中执行，比如更新 UI 元素或者操作共享资源。这时候就可以使用同步上下文来实现。同步上下文提供了一个 Post 方法，可以将回调函数加入到同步队列中，以便在主线程中执行。当主线程空闲时，同步上下文会自动从同步队列中取出回调函数并执行。

C# 中的同步上下文有多种实现，比如 WindowsFormsSynchronizationContext、AspNetSynchronizationContext 等。每种同步上下文都提供了不同的机制来协调多个线程的执行，开发者可以根据具体情况选择合适的同步上下文来使用。

Demo：

```dotnet
// 将请求发送到服务器并等待响应
HttpResponseMessage response = await Task.Run(() => client.PostAsync("http://example.com/api", content));

// 在同步上下文中处理响应
SynchronizationContext.Current.Post(_ =>
{
    if (response.IsSuccessStatusCode)
    {
        // 处理成功响应
    }
    else
    {
        // 处理错误响应
    }
}, null);
```

这段代码是一个使用异步/await模式的HTTP POST请求示例。它使用HttpClient类向指定的URL发送POST请求，并等待响应。

在这个例子中，我们使用了Task.Run方法来将操作放到后台线程中，以避免阻塞UI线程。当服务器返回响应时，我们使用同步上下文（SynchronizationContext）来确保处理响应的代码运行在UI线程上下文中，从而避免跨线程访问UI元素引发的异常。

如果响应状态码表明请求成功，我们可以执行一些成功处理逻辑；否则，我们可以根据错误类型执行相应的错误处理逻辑。

## 2.ConfigureAwait(false) 做什么？

在异步编程中，当需要执行长时间运行的操作时，通常会将代码封装在异步任务中。默认情况下，当任务完成时，其结果将被传输回调用线程的同步上下文，以便可以更新 GUI 界面等操作。但是，在某些情况下，如果不需要返回结果到同步上下文，则可以使用ConfigureAwait(false) 来告知编译器不需要在任务完成后切换回同步上下文。这可以提高性能，因为避免了不必要的上下文切换和线程阻塞。

需要注意的是，如果在异步任务中使用ConfigureAwait(false)，则应确保任务不会依赖于同步上下文，并且不会引发线程死锁或竞争条件等问题。

## 3.为什么要使用 ConfigureAwait(false)？

使用ConfigureAwait(false)可以提高异步代码的性能并减少线程阻塞，特别是在执行大量异步操作时。如果您正在编写一个需要执行多个异步操作的方法，并且这些操作不依赖于同步上下文，则使用ConfigureAwait(false) 可以避免重复地切换线程上下文，从而减少了 CPU 开销和内存开销。

另外，如果您在 ASP.NET中编写异步代码，则应尽可能使用ConfigureAwait(false)。因为 ASP.NET 框架会维护一个同步上下文，并在每个请求上下文中都有一个线程池，同时使用异步代码和同步上下文可能导致死锁和性能问题。

总之，在大部分情况下，如果您确定异步操作不需要返回到调用方的同步上下文中，那么使用ConfigureAwait(false) 是一种很好的选择。

## 4.为什么要使用ConfigureAwait(true)?

没关就是开了（没有写false就会接着使用同步上下文）
