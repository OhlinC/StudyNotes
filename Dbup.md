# Dbup

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

### Dbup的扫描以及记录

DbUp会扫描指定目录下的 SQL 脚本文件，并将每个脚本文件解析为一个单独的 SQL 命令。DbUp默认支持以下文件名模式：

- `*.sql`：所有扩展名为 `.sql` 的文件都会被视为 SQL 脚本文件。
- `*.*.sql`：以 `.sql` 为扩展名且文件名包含第二个点号的文件（例如 `001.create_table.sql`）也会被视为 SQL 脚本文件。  

![2](https://img-blog.csdnimg.cn/50e981a33fae4d28b6789e9dfa58c646.png)

在 DbUp 执行升级时，它会依次执行已解析的 SQL 命令，并将每个脚本的版本号和执行结果记录到数据库中，以便后续检查和管理。如果某个脚本执行失败，DbUp会回滚整个升级过程并记录错误信息。

**Dbup在底层执行过程中怎么筛选已执行文件**:

在DbUp执行过程中，已经执行的脚本会被记录到数据库的特定表（默认为'SchemaVersions'表）中，以便在下一次运行时能够避免重复执行。

当DbUp运行时，它首先查询数据库中存储的所有已执行脚本的版本号，然后从指定的目录中获取所有可供执行的脚本列表。接下来，DbUp会对这两个列表进行比较，只有那些版本号大于数据库中最后一个已执行的脚本版本号的脚本才会被执行。这意味着DbUp只会执行尚未在当前数据库实例中执行的脚本，并跳过那些已经执行过的脚本文件。

### 通过upgradeEngine.PerformUpgrade()函数进行执行数据库操作

##### 在执行前首先对执行文件进行过滤:

```csharp
var allScripts = GetDiscoveredScriptsAsEnumerable();
var executedScriptNames = new HashSet<string>(configuration.Journal.
GetExecutedScripts());

var sorted = allScripts.OrderBy(s => s.SqlScriptOptions.RunGroupOrder)
.ThenBy(s => s.Name, configuration.ScriptNameComparer);
var filtered = configuration.ScriptFilter.Filter(sorted, executedScriptNames, configuration.ScriptNameComparer);
return filtered.ToList();
```

首先，通过 `GetDiscoveredScriptsAsEnumerable()` 方法获取所有可用的脚本列表，返回值类型为 `IEnumerable<SqlScript>`。这里的 `SqlScript` 表示一个 SQL 脚本对象，包含了脚本名称、版本号、运行顺序等信息，以及一个方法可以执行该脚本。

然后，通过 `configuration.Journal.GetExecutedScripts()` 方法获取已经执行过的脚本列表，并将其保存到 `executedScriptNames` 变量中。这里的 `Journal` 是一个日志对象，用于记录已经执行的脚本，避免重复执行。

接下来，通过 LINQ 的 `OrderBy()` 和 `ThenBy()` 方法对脚本进行排序，按照 `RunGroupOrder` 属性和脚本名称排序。其中，`RunGroupOrder` 表示脚本所属的运行组，具有相同运行组的脚本会被一起执行。

然后，调用 `configuration.ScriptFilter.Filter()` 方法对脚本进行筛选。这里的 `ScriptFilter` 是一个筛选器对象，用于根据脚本名称和执行状态（已执行/未执行）进行筛选。这个方法接受三个参数：待筛选的脚本列表、已经执行过的脚本列表和脚本名称比较器。筛选结果以 `IEnumerable<SqlScript>` 的形式返回。

最后，将筛选结果转换成 `List<SqlScript>` 类型并返回。这个列表中的脚本会按照顺序依次执行，直到全部执行完毕或者发生错误。
