# 扫描HangfireRemark

## 一.如何在api启动时执行hangfire任务扫描

在api启动时部署一个扫描接口子类的一个周期性任务并添加到hangfire周期性任务中：

```csharp
var recurringJobTypes = typeof(IRecurringJob).Assembly.GetTypes().Where(type => type.IsClass && typeof(IRecurringJob).IsAssignableFrom(type)).ToList();

        foreach (var type in recurringJobTypes)
        {
            var job = (IRecurringJob) app.ApplicationServices.GetRequiredService(type);

            if (string.IsNullOrEmpty(job.CronExpression))
            {
                Log.Error("Recurring Job Cron Expression Empty, {Job}", job.GetType().FullName);
                continue;
            }

            RecurringJob.AddOrUpdate<IJobSafeRunner>(job.JobId, r => r.Run(job.JobId, type), job.CronExpression, job.TimeZone);
        }
```

在`IjobSafeRunner`实现：

```csharp
private readonly ILifetimeScope _lifetimeScope;

    public JobSafeRunner(ILifetimeScope lifetimeScope)
    {
        _lifetimeScope = lifetimeScope;
    }

    public async Task Run(string jobId, Type jobType)
    {
        await using var newScope = _lifetimeScope.BeginLifetimeScope();

        var job = (IJob) newScope.Resolve(jobType);

        using (LogContext.PushProperty("JobId", job.JobId))
        {
            await job.Execute();
        }
    }
```

通过此配置可以在api启动时扫描项目中继承`IRecurringJob`接口的类添加到`HangfireRecuuring`当中去。

一般Cron表达式配置在appsettings.json文件中，并配置一个setting类去获取value方便项目的管理。

Demo：

```csharp
public class Myclass : IRecurringJob
{
    private readonly IMediator _mediator;

    private readonly TestCronExpressionSetting _cronExpressionSetting;

    public Myclass(
        IMediator mediator, TestCronExpressionSetting cronExpressionSetting)
    {
        _mediator = mediator;
        _cronExpressionSetting = cronExpressionSetting;
    }

    public async Task Execute()
    {
        //需要执行的方法
        await _mediator.SendAsync(new TestCommand(), CancellationToken.None);
    }

    public string JobId => nameof(添加到hangfire的任务名);

    public string CronExpression => _cronExpressionSetting.Value;
}
```

## 二.如何获取hangfire任务

### 1.获取Recurring任务

```csharp
JobStorage.Current.GetConnection().GetRecurringJobs().Select(x => x.LastJobId).ToList();
```

直接从JobStorage里获取到一个RecurringDto集合，这里我们只取JobIds。

### 2.获取Delayed任务

```csharp
JobStorage.Current.GetMonitoringApi()  
 .ScheduledJobs(0, (int)JobStorage.Current.GetMonitoringApi().ScheduledCount())  
 .ToList().Select(x => x.Key).ToList();
```

ScheduledJobs参数为起始下标至结尾下标，这里我们拿全部，通过这个方法拿到一个ScheduledJobDto的JobList集合**不用ToList转List的话会报异常**。

### 3.获取Succeeded任务

```csharp
JobStorage.Current.GetMonitoringApi()
            .SucceededJobs(0, (int)JobStorage.Current.GetMonitoringApi().SucceededListCount())
            .ToList().Select(x => x.Key).ToList();
```

其他的任务都是以`GetMonitoringApi`下的方法获取方式都差不多。

### 4.获取Job状态

JobStorage的获取状态有两种快捷方式：

* 通过`JobStorage.Current.GetMonitoringApi().JobDetails(jobid)`拿到JobDetailsDto其中类型为List<`StateHistoryDto`>的History成员包含着整个job的历史状态信息。

* 通过`JobStorage.Current.GetConnection().GetStateData(jobId)`拿到StateData类，其中的name成员为任务的状态，其中状态有：
  
  - Enqueued：表示已入队但尚未处理的作业数。
  - Fetched：表示已从队列中取出但尚未执行的作业数。
  - Processing：表示当前正在处理的作业数。
  - Succeeded：表示已成功执行的作业数。
  - Failed：表示执行失败的作业数。
  - Scheduled：表示尚未入队的调度作业数。
  - Deleted：表示已删除的作业数。
  - Awaiting：表示还未被处理的延迟任务数。
  - Recurring：表示已注册的重复任务数
  
  注意：通过GetStateData方法获取的任务不能是Recurring 任务。

## 三.如何在XUnit测试

建一个接口类代理执行

```csharp
private readonly Func<IRecurringJobManager> _recurringJobManagerFunc;
private readonly Func<IBackgroundJobClient> _backgroundJobClientFunc;

 public PostBoyBackgroundJobClient(
        Func<IRecurringJobManager> recurringJobManagerFunc,
        Func<IBackgroundJobClient> backgroundJobClientFunc)
    {
        _recurringJobManagerFunc = recurringJobManagerFunc;
        _backgroundJobClientFunc = backgroundJobClientFunc;

        _queue = new EnqueuedState("default");
    }

    public string Enqueue(Expression<Func<Task>> methodCall)
    {
        return _backgroundJobClientFunc()?.Create(methodCall, _queue);
    }

    public string Enqueue<T>(Expression<Func<T, Task>> methodCall)
    {
        return _backgroundJobClientFunc()?.Create(methodCall, _queue);
    }

    public string Schedule(Expression<Func<Task>> methodCall, TimeSpan delay)
    {
        return _backgroundJobClientFunc()?.Schedule(methodCall, delay);
    }

    public string Schedule(Expression<Func<Task>> methodCall, DateTimeOffset enqueueAt)
    {
        return _backgroundJobClientFunc()?.Schedule(methodCall, enqueueAt);
    }

    public string Schedule<T>(Expression<Func<T, Task>> methodCall, DateTimeOffset enqueueAt)
    {
        return _backgroundJobClientFunc()?.Schedule(methodCall, enqueueAt);
    }

    public string ContinueJobWith(string parentJobId, Expression<Func<Task>> methodCall)
    {
        return _backgroundJobClientFunc()?.ContinueJobWith(parentJobId, methodCall, _queue);
    }

    public string ContinueJobWith<T>(string parentJobId, Expression<Func<T, Task>> methodCall)
    {
        return _backgroundJobClientFunc()?.ContinueJobWith(parentJobId, methodCall, _queue);
    }

    public bool DeleteJob(string jobId)
    {
        return _backgroundJobClientFunc().Delete(jobId);
    }

    public void RemoveRecurringJobIfExists(string jobId)
    {
        _recurringJobManagerFunc()?.RemoveIfExists(jobId);
    }

    public void AddOrUpdateRecurringJob<T>(string recurringJobId, 
        Expression<Func<T, Task>> methodCall, 
        string cronExpression, 
        TimeZoneInfo timeZone = null,
        string queue = "default")
    {
        _recurringJobManagerFunc().AddOrUpdate(recurringJobId, methodCall, cronExpression, timeZone, queue);
    }
```

并重写里面的方法，在项目继承这个类添加hangfire方法。

在测试中通过继承这个接口在重写在测试的方法就可以mock出一个相应方法的hangfire测试：

```csharp
public class MockingBackgroundJobClient : IPostBoyBackgroundJobClient
{
    public static readonly List<string> TestJobs = new ();

    public static readonly List<string> TestStore = new ();

    private readonly IComponentContext _componentContext;

    public MockingBackgroundJobClient(IComponentContext componentContext)
    {
        _componentContext = componentContext;
    }
    
    public MockingBackgroundJobClient()
    {
    }

     public string Enqueue(Expression<Func<Task>> methodCall)
    {
        var func = methodCall.Compile();
        func().Wait();
        return string.Empty;
    }

    public string Enqueue<T>(Expression<Func<T, Task>> methodCall) where T : notnull
    {
        var mediator = _componentContext.Resolve<T>();
        var func = methodCall.Compile();
        func(mediator).Wait();
        return string.Empty;
    }

    public string Schedule(Expression<Func<Task>> methodCall, TimeSpan delay)
    {
        TestJobs.Add("add Delayed");
        return "";
    }

    public string Schedule(Expression<Func<Task>> methodCall, DateTimeOffset enqueueAt)
    {
        TestJobs.Add("add Delayed");
        return "";
    }

    public string Schedule<T>(Expression<Func<T, Task>> methodCall, DateTimeOffset enqueueAt)
    {
        TestJobs.Add("add Delayed");
        return "";
    }

    public string ContinueJobWith(string parentJobId, Expression<Func<Task>> methodCall)
    {
        return "";
    }

    public string ContinueJobWith<T>(string parentJobId, Expression<Func<T, Task>> methodCall)
    {
        return "";
    }

    public bool DeleteJob(string jobId)
    {
        return TestJobs.Count > 0 ? TestJobs.Remove(jobId) : default;
    }

    public void RemoveRecurringJobIfExists(string jobId)
    {
        TestStore.Add("Trigger remove");
    }
    
    public void AddOrUpdateRecurringJob<T>(string recurringJobId, Expression<Func<T, Task>> methodCall, string cronExpression,
        TimeZoneInfo timeZone = null, string queue = "default")
    {
        TestJobs.Add("Add Recurring");
    }
```

对于延期任务或者周期性任务可通过计算添加次数进行判断，但是队列任务需要模拟任务的执行过程并调用方法，所以我们mock应执行参数中的方法，在调用方法时注册Mock：

```csharp
await Run<IMediator>(async mediator => { await mediator.SendAsync(new TestCommand()); },
            builder =>
            {
                builder
                    .RegisterType<MockingBackgroundJobClient>()
                    .As<IPostBoyBackgroundJobClient>()
                    .InstancePerLifetimeScope();
            });
```

所以尽量把获取hangfire方法写在IPostBoyBackgroundJobClient的实现类中，方便Mock测试。
