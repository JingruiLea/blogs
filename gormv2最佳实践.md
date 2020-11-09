![](RackMultipart20201109-4-1m54uho_html_85ec075b80692caa.gif)

# **追求极致**

# **--GormV2 API**

# **探究与最佳实践**

gorm是个好东西, 但是除了张金柱同学, 谁又能发挥它的最大实力, 给我们的增删改查注入更多动力.

本篇碎碎念, 希望讨论下gormV2中各种花里胡哨api的最佳实践, 也讨论下gorm的源码.

这篇文章通过看源码, 写测试, 问作者等方式, 总结了gorm常见的问题和文档中没有写清的行为.

**嫌太长不看的直接看总结****.**

## **总结**

1. 配置单数表名, 再也不用写TableName

1. 使用DB前先db.model, 万无一失

1. CreatedAt, UpdatedAt, mysql类型用datetime(3)(毫秒)

1. 支持db.Create创建和批量创建

1. BeforeCreate创建前自动填写id

AfterUpdate更新后自动创建record

1. Where用结构体查询, 自动忽略零值, 不用手打列名.

1. ErrRecordNotFound只会出现在Take, First,Last方法中

1. Find结构体时: 等同于Take, 但没有ErrRecordNotFound, RowsAffected为0或1

1. 行锁的写法DB.Clauses(clause.Locking{Strength: &quot;UPDATE&quot;})

1. Updates支持结构体更新, 自动忽略零值, 不用手打列名. 强行更新零值用Select

1. Update支持gorm.ExprSQL表达式更新

1. 事务直接用db.Transaction方法即可

## **从表名开始**

如何定义表名? gorm的默认表名策略是模型名字的复数, 比如:
```golang
 type User Struct{}
```

默认表名为users, 但是生产习惯常用单数表名, 而且加es的复数或者sheep单复同形容易让人迷惑.

当然, gorm实现了复杂的单数变复数逻辑, 我们可以从 github.com/jinzhu/inflection/inflections.go 一探究竟, 下面复习下单复同型/不可数单词, 以及特殊变复数的单词:
```go
var uncountableInflections = []string{"equipment", "information", "rice", "money", "species", "series", "fish", "sheep", "jeans", "police"}

var irregularInflections = IrregularSlice{
   {"person", "people"},
   {"man", "men"},
   {"child", "children"},
   {"sex", "sexes"},
   {"move", "moves"},
   {"mombie", "mombies"},
}
```

好吧, 写个增删改查还得过专八.

我们可以实现gorm.schema.scheme.go.Tabler.TableName()方法, 重写表名.
```golang
func (*User) TableName() string {return "user"}
```
OK!

除了这种奇奇怪怪的写法, 在gormV2中, 我们可以使用一个配置改为单数表名, 谢天谢地.
```golang
NamingStrategy: &schema.NamingStrategy{SingularTable: true}
```

schema.NamingStrategy实现了gorm.schema.naming.go.Namer接口, 我们也可以自己实现这个接口,替换gorm的命名策略, 接口简单易懂.

```go
type Namer interface {
   TableName(table string) string
   ColumnName(table, column string) string
   JoinTableName(joinTable string) string
   RelationshipFKName(Relationship) string
   CheckerName(table, column string) string
   IndexName(table, column string) string
}
```

## **列名**

列名无可非议, 简简单单的下划线模式. 而且gorm会自动将ID转换成id而不是i\_d, 那么, 就遵循golang的语言规范, 放心的在代码中使用ID吧~

让我们来看下这个特殊变小写数组.

**列名变小写****?**

gorm.schema.naming.go

```go
// https://github.com/golang/lint/blob/master/lint.go#L770
commonInitialisms         = []string{"API", "ASCII", "CPU", "CSS", "DNS", "EOF", "GUID", "HTML", "HTTP", "HTTPS", "ID", "IP", "JSON", "LHS", "QPS", "RAM", "RHS", "RPC", "SLA", "SMTP", "SSH", "TLS", "TTL", "UID", "UI", "UUID", "URI", "URL", "UTF8", "VM", "XML", "XSRF", "XSS"}
```

**CreatedAt, UpdatedAt**

更加有争议和难以理解的列名,应该是这两个.

**最原始的方法 ,mysql****自动添加****.**

如果想让mysql来做这件事, 可以这样写:

```go
create table user2
(
    ....
    create_time datetime(3) not null DEFAULT CURRENT_TIMESTAMP(3) comment '创建时间',
    update_time datetime(3) not null DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) comment '修改时间',
   ....
);
```

```golang
type User struct {
   ID        int64 `gorm:"autoIncrement"`
   CreatedAt time.Time `gorm:default`
   UpdatedAt time.Time `gorm:default`
}
```

加上gorm的default标签, 这个标签的意思是: gorm.Create时, 如果该列值为零, 使用建表时的default值, sql语句中也就没有这一列.

gorm.Updates如果Update一个map,就是强制更新. 如果Update一个结构体, 不赋值也就是用mysql的默认值, 如果赋值就直接Update, 这一点很巧妙.

gorm对默认值的处理:

gorm.callbacks.create.go:285

```golang
defaultValueFieldsHavingValue := map[*schema.Field][]interface{}{}
for _, field := range stmt.Schema.FieldsWithDefaultDBValue {
   if v, ok := selectColumns[field.DBName]; (ok && v) || (!ok && !restricted) {
      if v, isZero := field.ValueOf(rv); !isZero {
         if len(defaultValueFieldsHavingValue[field]) == 0 {
            defaultValueFieldsHavingValue[field] = make([]interface{}, stmt.ReflectValue.Len())
         }
         defaultValueFieldsHavingValue[field][i] = v
      }
   }
}
```

你要说我能看懂, 那我肯定看不懂, 但你要说看不懂, 我还知道它能干啥, 不错不错...

**更好的方法**** , gorm ****自动添加**

如果想让gorm来添加, sql就可以不用写default

```golang
type User struct {
   ID        int64 `gorm:"autoIncrement"`
   CreatedAt time.Time
   UpdatedAt time.Time
}
```

只要列名叫CreatedAt和UpdatedAt即可.

gorm.Create即可自动更新CreatedAt和UpdatedAt.

但是gorm.Updates时可要注意了, 如果没有使用db.Model指定model结构体, 就不能更新UpdatedAt.

所以如果这样:
```go
db.Table(tableName).Updates(map[string]interface{})
```
是无法更新更新时间的.

正确做法是:

```go
db.Model(&amp;User{}).Updates(map[string]interface{})
```
如果傲娇的你, 偏偏不喜欢这样的列名, 非要用CreateTime, 那好吧, 你可以这样: 不告诉你, 自己去查文档... [https://gorm.io/zh\_CN/docs/models.html](https://gorm.io/zh_CN/docs/models.html)

**小建议**** : **** 建议建表时使用****datetime(3)****毫秒类型 ****.**

## **Create**

[https://gorm.io/zh\_CN/docs/create.html](https://gorm.io/zh_CN/docs/create.html)

gormV2支持创建和批量创建, 你可以这样写:
```go
//创建
db.Create(&User{})

//批量创建
users := []*User{{},{},{}}
db.Create(&users)
```

## **Hooks**

我们可以灵活的使用hooks, 以减少重复代码在logic层对业务逻辑的侵入, 使代码更简洁. 钩子命名好像Vue啊! 哈哈哈


## **Where**

gorm中最常用的语句, gormv1中最常用的方法是:

```golang
db.Where("uid = ?", 1)
```

gormV2支持两种新的where, Struct和map条件, 本文介绍struct条件:

```go
db.Where(&User{phone:"123", Age:0})

//Select * from User where phone = 123
//Age没有被查询
```

结构体查询非常好用, 你不用手动写列名, 避免了因为列名写错而导致的错误.

**注意 当使用结构作为条件查询时，**** GORM **** 只会查询非零值字段。这意味着如果您的字段值为 **** 0 ****、****&#39;&#39; ****、**** false **** 或其他**[零值](https://tour.golang.org/basics/12)**，该字段不会被用于构建查询条件，例如：**

## **Find, Take, First, Last**

**ErrRecordNotFound**

这个东西可是实在太恶心了... 但是golang的反射又没法对ptr赋值为nil, 所以只能通过error返回. 这种处理方式也是可以理解的.
```go
err := db.Where(...).First(...).Error
// 检查 ErrRecordNotFound 错误
errors.Is(err, gorm.ErrRecordNotFound)
```

**注意**** : ****只有**** First,Last,Take ****方法会产生**** gorm.ErrRecordNotFound ****错误哦**

**Find**

为了躲避ErrRecordNotFound, 我们来看下这个可爱的Find方法.

Find方法可以接受两种参数, 一种是结构体指针, 一种是数组指针.

接受数组指针很好理解, 我们可以通过数组长度来判断是否查到数据:
```go
res := []*model.User{}
ret := db.Model(&model.User{}).Where(`user_id > ?`, 0).Find(res)
```

接受结构体指针时呢? 我们只能通过ret.RowsAffected == 0来判断是否查到数据, 所以ErrRecordNotFound还有点可爱?

**注意**** :db.Find(&amp;User{}).RowsAffected ****只会是**** 0 ****或**** 1**
```
res := &model.User{}
ret := db.Model(&model.User{}).Where(`user_id > ?`, 0).Find(res)
ret.RowsAffected
```

从源码来看, Find结构体和Find数组的差别如下:

gorm.scan.go:214

```go
switch(){
case reflect.Slice, reflect.Array:
    for initialized || rows.Next() {
    }
case reflect.Struct:
    if initialized || rows.Next() {
    }
}
```

OMG!仅仅是for和if的差别...

那么Find和Take有什么差别呢? 你猜的没错, 只是Take会返回ErrRecordNotFound

gorm.scan.go:239

```golang
if db.RowsAffected == 0 && db.Statement.RaiseErrorOnNotFound {
   db.AddError(ErrRecordNotFound)
}
//Find RaiseErrorOnNotFound=false
//Take RaiseErrorOnNotFound=true
```

好家伙! 我直接好家伙! 果然是大佬的代码...

**行锁🔒**

哪里的代码有version乐观锁和share锁, 我去参观下 T.T 只见过UPDATE锁.

gormV2和V1这里确实不一样, 写法如下:

```golang
DB.Clauses(clause.Locking{Strength: "UPDATE"}).Find(&users)
```
mysql行锁只在事务中有用.

## **Update**

gormV2的更新是我最喜欢的一部分, 非常的有趣...

**零值**

我们不再需要这样, 我讨厌的方式:

```go
var updateMap := make(map[string]interface{})
if phone != "" {
    updateMap["phone"] = phone
}
db.Updates(updateMap)
```

我们只需要这样:

```go
db.Updates(&User{Phone:phone})
```
Updates方法接受一个Model结构体, 如果更新的列为 &quot;&quot;,0,false,time.Time{} 就会不更新该列. (如果使用gorm-curd, 连数组长度都帮你判断了,真的是太好用了!)

那你要问, 如果我非要设置这个用户的phone为&quot;&quot;呢?

你可以这样写:

```go
db.Select("phone").Updates(&User{Phone:""})
//或者
db.Updates(map[string]interface{}{"phone":""})
```

又回去map了... 梅开二度...

**SQL**** 表达式**

如果需要商品库存 - 1, gormV2不再需要用事务取出, 再 -1 , 再存入...

可以这样:
```golang
DB.Model(&product).Where("goods_id = ?", 10086).Update("quantity", gorm.Expr("quantity - ?", 1))
```

## **事务**

**事务**

要在事务中执行一系列操作，一般流程如下, 这个事务写法真的是太棒了!

```golang
db.Transaction(func(tx *gorm.DB) error {
 // 在事务中执行一些 db 操作（从这里开始，您应该使用 'tx' 而不是 'db'）
 if err := tx.Create(&Animal{Name: "Giraffe"}).Error; err != nil {
 // 返回任何错误都会回滚事务
 return err
 }

 if err := tx.Create(&Animal{Name: "Lion"}).Error; err != nil {
 return err
 }

 // 返回 nil 提交事务
 return nil
})
```

**嵌套事务**

GORM 支持嵌套事务，可以回滚事务中的事务而不用担心外层事务回滚.