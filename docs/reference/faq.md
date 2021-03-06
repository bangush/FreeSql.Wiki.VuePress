# 常见问题

### 1、如何监视 SQL？

方法一：UseMonitorCommand + UseNoneCommandParameter
```csharp
static IFreeSql fsql = new FreeSql.FreeSqlBuilder()
  .UseConnectionString(FreeSql.DataType.MySql, "...")
  .UseMonitorCommand(cmd => Console.WriteLine($"线程：{cmd.CommandText}\r\n"))
  .UseNoneCommandParameter(true)
  .Build();
```

方法二：Aop.CurdBefore/CurdAfter
```csharp
fsql.Aop.CurdAfter += (s, e) =>
{
  if (e.ElapsedMilliseconds > 200)
    Console.WriteLine($"线程：{e.Sql}\r\n")
};
```
---

### 2、MySql Enum 映射

默认情况 c# 枚举会映射为 MySql Enum 类型，如果想映射为 int 在 FreeSqlBuilder Build 之后执行以下 Aop 统一处理：
```csharp
fsql.Aop.ConfigEntityProperty += (s, e) => {
  if (e.Property.PropertyType.IsEnum)
    e.ModifyResult.MapType = typeof(int);
};
```
---

### 3、多个 IFreeSql 实例，如何注入使用？

[https://github.com/dotnetcore/FreeSql/issues/44](https://github.com/dotnetcore/FreeSql/issues/44)

---

### 4、TransactionalAttribute + UnitOfWorkManager 事务传播

[https://github.com/dotnetcore/FreeSql/issues/289](https://github.com/dotnetcore/FreeSql/issues/289)

---

### 5、怎么执行 SQL 返回实体列表？

```csharp
//直接查询
fsql.Ado.Query<T>(sql);

//嵌套一层做二次查询
fsql.Select<T>().WithSql(sql).Page(1, 10).ToList();
```

---

### 6、错误：【主库】状态不可用，等待后台检查程序恢复方可使用。xxx

一般是数据库连接失败，才会出现，请检查程序与数据库之间的网络。具体按 xxx 给出的提示进行排查。

---


### 7、错误：【主库】对象池已释放，无法访问。

原因一：手工调用了 fsql.Dispose，之后仍然使用它

原因二：使用了 IdleBus 管理 IFreeSql，错误的方式如下：

- a) 不要构建了 IFreeSql 再丢去注册

```c#
var fsql = new FreeSqlBulder()...Build();
ib.Register("key01", () => fsql); //错了，错了，错了

ib.Register("key01", () => new FreeSqlBulder()...Build()); //正确
```

- b) 尽量每次都使用 ib.Get 获得 IFreeSql 对象(避免存对象引用)，IdleBus 内部超时释机制一时触发，再使用引用对象，就会报这个报错

原因三：检查项目的系统事件，是否在异常之前触发

```c#
AppDomain.CurrentDomain.ProcessExit += (s1, e1) =>
{
  //记录日志
};
Console.CancelKeyPress += (s1, e1) =>
{
  //记录日志  
};
```

如果确定问题，可以在 FreeSqlBuilder 构建对象的时候 UseExitAutoDisposePool(false) 关闭这个机制

---

### 8、错误：ObjectPool.Get 获取超时（10秒）。

原因一：UnitOfWork 使用未释放，请保证程序内使用 UnitOfWork 的地方会执行 Dispose

原因二：Max Pool Size 设置过小，程序访问量过高

监视 fsql.Ado.MasterPool.Statistics，它的值：Pool: 5/100, Get wait: 0, GetAsync await: 0

```
5 为可用连接数，值为0后开始排队
100 为当前最大连接数
Get await 为同步方法获取连接的排队数量（超过10秒就会报错）
GetAsync await 为异步方法获取连接的排队数量
```
  
监视 FreeSql.UnitOfWork.DebugBeingUsed 这个静态字典，存储正在使用事务的工作单元

注意：尽量不要使用 fsql.Ado.MasterPool.Get() 或 GetAsync() 方法，否则请检查姿势。

---