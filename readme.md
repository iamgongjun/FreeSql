# FreeSql

打造 .NETCore 最方便的orm，dbfirst codefirst混合使用，codefirst模式下的开发阶段，建好实体不用执行任何，就能创建表和修改字段，dbfirst模式下提供api+模板，自定义生成代码，作者提供了3种模板。

* [《CodeFirst 快速开发文档》](Docs/codefirst.md)

* [《DbFirst 快速开发文档》](Docs/dbfirst.md)

# 快速开始
```csharp
var connstr = "Data Source=127.0.0.1;Port=3306;User ID=root;Password=root;" + 
    "Initial Catalog=cccddd;Charset=utf8;SslMode=none;Max pool size=10";

IFreeSql fsql = new FreeSql.FreeSqlBuilder()
    .UseConnectionString(FreeSql.DataType.MySql, connstr)
    .UseSlave("connectionString1", "connectionString2") //使用从数据库，支持多个

    .UseLogger(null) //使用日志，不指定默认输出控制台 ILogger
    .UseCache(null) //使用缓存，不指定默认使用内存 IDistributedCache

    .UseAutoSyncStructure(true) //自动同步实体结构到数据库
    .UseSyncStructureToLower(true) //转小写同步结构
    .Build();
```

# 实体
```csharp
[Table(Name = "tb_topic")]
class Topic {
    [Column(IsIdentity = true)]
    public int Id { get; set; }
    public int Clicks { get; set; }
    public string Title { get; set; }
    public DateTime CreateTime { get; set; }

    public int TypeId { get; set; }
    public TopicType Type { get; set; } //导航属性
}
class TopicType {
    [Column(IsIdentity = true)]
    public int Id { get; set; }
    public string Name { get; set; }

    public int ClassId { get; set; }
    public TopicTypeClass Class { get; set; } //导航属性
}
class TopicTypeClass {
    public int Id { get; set; }
    public string Name { get; set; }
}
```

# Part1: 查询

```csharp
List<Topic> t1 = fsql.Select<Topic>()
    .Where(a => a.Id > 0)
    .ToList();

//返回普通字段 + 导航对象 Type 的数据
List<Topic> t2 = fsql.Select<Topic>()
    .LeftJoin(a => a.Type.Id == a.TypeId)
    .ToList();

//返回一个字段
List<int> t3 = fsql.Select<Topic>()
    .Where(a => a.Id > 0)
    .ToList(a => a.Id);

//返回匿名类型
List<匿名类型> t4 = fsql.Select<Topic>()
    .Where(a => a.Id > 0)
    .ToList(a => new { a.Id, a.Title });

//返回元组
List<(int, string)> t5 = fsql.Select<Topic>()
    .Where(a => a.Id > 0)
    .ToList<(int, string)>("id, title");
```
### 联表之一：使用导航属性
```csharp
sql = fsql.Select<Topic>()
    .LeftJoin(a => a.Type.Id == a.TypeId)
    .ToSql();

sql = fsql.Select<Topic>()
    .LeftJoin(a => a.Type.Id == a.TypeId)
    .LeftJoin(a => a.Type.Class.Id == a.Type.ClassId)
    .ToSql();
```
### 联表之二：无导航属性
```csharp
sql = fsql.Select<Topic>()
    .LeftJoin<TopicType>((a, b) => b.Id == a.TypeId)
    .ToSql();

sql = fsql.Select<Topic>()
    .LeftJoin<TopicType>((a, b) => b.Id == a.TypeId)
    .LeftJoin<TopicTypeClass>((a, c) => c.Id == a.Type.ClassId)
    .ToSql();
```
### 联表之三：b, c 条件怎么设？试试这种！
```csharp
sql = fsql.Select<Topic>()
    .From<TopicType, TopicTypeClass>((s, b, c) => s
    .LeftJoin(a => a.TypeId == b.Id)
    .LeftJoin(a => b.ClassId == c.Id))
    .ToSql();
```
### 联表之四：原生SQL联表
```csharp
sql = fsql.Select<Topic>()
    .LeftJoin("TopicType b on b.Id = a.TypeId and b.Name = ?bname", new { bname = "xxx" })
    .ToSql();
```

# Part2: 添加
```csharp
var items = new List<Topic>();
for (var a = 0; a < 10; a++) items.Add(new Topic { Title = $"newtitle{a}", Clicks = a * 100 });

var t1 = fsql.Insert<Topic>().AppendData(items.First()).ToSql();
//INSERT INTO `tb_topic`(`Clicks`, `Title`, `CreateTime`) VALUES(?Clicks0, ?Title0, ?CreateTime0)
```
### 批量插入
```csharp
var t2 = fsql.Insert<Topic>().AppendData(items).ToSql();
//INSERT INTO `tb_topic`(`Clicks`, `Title`, `CreateTime`) VALUES(?Clicks0, ?Title0, ?CreateTime0), (?Clicks1, ?Title1, ?CreateTime1), (?Clicks2, ?Title2, ?CreateTime2), (?Clicks3, ?Title3, ?CreateTime3), (?Clicks4, ?Title4, ?CreateTime4), (?Clicks5, ?Title5, ?CreateTime5), (?Clicks6, ?Title6, ?CreateTime6), (?Clicks7, ?Title7, ?CreateTime7), (?Clicks8, ?Title8, ?CreateTime8), (?Clicks9, ?Title9, ?CreateTime9)
```
### 插入指定的列
```csharp
var t3 = fsql.Insert<Topic>().AppendData(items).InsertColumns(a => a.Title).ToSql();
//INSERT INTO `tb_topic`(`Title`) VALUES(?Title0), (?Title1), (?Title2), (?Title3), (?Title4), (?Title5), (?Title6), (?Title7), (?Title8), (?Title9)

var t4 = fsql.Insert<Topic>().AppendData(items).InsertColumns(a => new { a.Title, a.Clicks }).ToSql();
//INSERT INTO `tb_topic`(`Clicks`, `Title`) VALUES(?Clicks0, ?Title0), (?Clicks1, ?Title1), (?Clicks2, ?Title2), (?Clicks3, ?Title3), (?Clicks4, ?Title4), (?Clicks5, ?Title5), (?Clicks6, ?Title6), (?Clicks7, ?Title7), (?Clicks8, ?Title8), (?Clicks9, ?Title9)
```
### 忽略列
```csharp
var t5 = fsql.Insert<Topic>().AppendData(items).IgnoreColumns(a => a.CreateTime).ToSql();
//INSERT INTO `tb_topic`(`Clicks`, `Title`) VALUES(?Clicks0, ?Title0), (?Clicks1, ?Title1), (?Clicks2, ?Title2), (?Clicks3, ?Title3), (?Clicks4, ?Title4), (?Clicks5, ?Title5), (?Clicks6, ?Title6), (?Clicks7, ?Title7), (?Clicks8, ?Title8), (?Clicks9, ?Title9)

var t6 = fsql.Insert<Topic>().AppendData(items).IgnoreColumns(a => new { a.Title, a.CreateTime }).ToSql();
///INSERT INTO `tb_topic`(`Clicks`) VALUES(?Clicks0), (?Clicks1), (?Clicks2), (?Clicks3), (?Clicks4), (?Clicks5), (?Clicks6), (?Clicks7), (?Clicks8), (?Clicks9)
```
### 执行命令
| 方法 | 返回值 | 描述 |
| - | - | - |
| ExecuteAffrows | long | 执行SQL语句，返回影响的行数 |
| ExecuteIdentity | long | 执行SQL语句，返回自增值 |
| ExecuteInserted | List\<Topic\> | 执行SQL语句，返回插入后的记录 |

# Part3: 修改
### 更新指定列
```csharp
var t1 = fsql.Update<Topic>(1).Set(a => a.CreateTime, DateTime.Now).ToSql();
//UPDATE `tb_topic` SET `CreateTime` = '2018-12-08 00:04:59' WHERE (`Id` = 1)
```
### 更新指定列，累加
```csharp
var t2 = fsql.Update<Topic>(1).Set(a => a.Clicks + 1).ToSql();
//UPDATE `tb_topic` SET `Clicks` = ifnull(`Clicks`,0) + 1 WHERE (`Id` = 1)
```
### 保存实体
```csharp
var item = new Topic { Id = 1, Title = "newtitle" };
var t3 = fsql.Update<Topic>().SetSource(item).ToSql();
//UPDATE `tb_topic` SET `Clicks` = ?p_0, `Title` = ?p_1, `CreateTime` = ?p_2 WHERE (`Id` = 1)
```
### 保存实体，忽略一些列
```csharp
var t4 = fsql.Update<Topic>().SetSource(item)
    .IgnoreColumns(a => a.Clicks).ToSql();
//UPDATE `tb_topic` SET `Title` = ?p_0, `CreateTime` = ?p_1 WHERE (`Id` = 1)

var t5 = fsql.Update<Topic>().SetSource(item)
    .IgnoreColumns(a => new { a.Clicks, a.CreateTime }).ToSql();
//UPDATE `tb_topic` SET `Title` = ?p_0 WHERE (`Id` = 1)
```
### 批量保存
```csharp
var items = new List<Topic>();
for (var a = 0; a < 10; a++)
    items.Add(new Topic { Id = a + 1, Title = $"newtitle{a}", Clicks = a * 100 });

var t6 = fsql.Update<Topic>().SetSource(items).ToSql();
//UPDATE `tb_topic` SET `Clicks` = CASE `Id` WHEN 1 THEN ?p_0 WHEN 2 THEN ?p_1 WHEN 3 THEN ?p_2 WHEN 4 THEN ?p_3 WHEN 5 THEN ?p_4 WHEN 6 THEN ?p_5 WHEN 7 THEN ?p_6 WHEN 8 THEN ?p_7 WHEN 9 THEN ?p_8 WHEN 10 THEN ?p_9 END, `Title` = CASE `Id` WHEN 1 THEN ?p_10 WHEN 2 THEN ?p_11 WHEN 3 THEN ?p_12 WHEN 4 THEN ?p_13 WHEN 5 THEN ?p_14 WHEN 6 THEN ?p_15 WHEN 7 THEN ?p_16 WHEN 8 THEN ?p_17 WHEN 9 THEN ?p_18 WHEN 10 THEN ?p_19 END, `CreateTime` = CASE `Id` WHEN 1 THEN ?p_20 WHEN 2 THEN ?p_21 WHEN 3 THEN ?p_22 WHEN 4 THEN ?p_23 WHEN 5 THEN ?p_24 WHEN 6 THEN ?p_25 WHEN 7 THEN ?p_26 WHEN 8 THEN ?p_27 WHEN 9 THEN ?p_28 WHEN 10 THEN ?p_29 END WHERE (`Id` IN (1,2,3,4,5,6,7,8,9,10))
```
### 批量保存，忽略一些列
```csharp
var t7 = fsql.Update<Topic>().SetSource(items)
    .IgnoreColumns(a => new { a.Clicks, a.CreateTime }).ToSql();
//UPDATE `tb_topic` SET `Title` = CASE `Id` WHEN 1 THEN ?p_0 WHEN 2 THEN ?p_1 WHEN 3 THEN ?p_2 WHEN 4 THEN ?p_3 WHEN 5 THEN ?p_4 WHEN 6 THEN ?p_5 WHEN 7 THEN ?p_6 WHEN 8 THEN ?p_7 WHEN 9 THEN ?p_8 WHEN 10 THEN ?p_9 END WHERE (`Id` IN (1,2,3,4,5,6,7,8,9,10))
```
### 批量更新指定列
```csharp
var t8 = fsql.Update<Topic>().SetSource(items).Set(a => a.CreateTime, DateTime.Now).ToSql();
//UPDATE `tb_topic` SET `CreateTime` = ?p_0 WHERE (`Id` IN (1,2,3,4,5,6,7,8,9,10))
```
### 更新条件
```csharp
fsql.Update<Topic>(object dywhere)
```
dywhere 支持
* 主键值
* new[] { 主键值1, 主键值2 }
* Topic对象
* new[] { Topic对象1, Topic对象2 }
* new { id = 1 }
```csharp
var t9 = fsql.Update<Topic>().Set(a => a.Title, "新标题").Where(a => a.Id == 1).ToSql();
//UPDATE `tb_topic` SET `Title` = '新标题' WHERE (Id = 1)
```
### 自定义SQL
```csharp
var t10 = fsql.Update<Topic>().SetRaw("Title = {0}", "新标题").Where("Id = {0}", 1).ToSql();
//UPDATE `tb_topic` SET Title = '新标题' WHERE (Id = 1)
//sql语法条件，参数使用 {0}，与 string.Format 保持一致，无须加单引号，错误的用法：'{0}'
```
### 执行命令
| 方法 | 返回值 | 参数 | 描述 |
| - | - | - | - |
| ExecuteAffrows | long | | 执行SQL语句，返回影响的行数 |
| ExecuteUpdated | List\<T1\> | | 执行SQL语句，返回更新后的记录 |

# Part4: 删除
详情查看：[《Delete 删除数据》](Docs/delete.md)

# Part4: 表达式函数
详情查看：[《Expression 表达式函数》](Docs/expression.md)


# 更多文档整理中。。。

## 贡献者名单

