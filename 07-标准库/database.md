#一 Go的数据库接口

#### 1.1 接口介绍

Go官方没有提供数据库驱动，而是为开发数据库驱动定义了一些标准接口，开发者可以根据定义的接口来开发相应的数据库驱动，这样做的好处是：框架迁移极其方便。  

Go数据库标准包位于以下两个包中：
- database/sql：提供了保证SQL或类SQL数据库的泛用接口
- database/sql/driver：定义了应被数据库驱动实现的接口，这些接口会被sql包使用

#### 1.2 sql.Register
sql.Register位于database/sql，用来注册数据库驱动当第三方开发者开发数据库驱动时，都会实现init函数，在init里面会调用这个`Register(name string, driver driver.Driver)`完成本驱动的注册。

我们来看一下mysql、sqlite3的驱动里面都是怎么调用的：
```Go

//https://github.com/mattn/go-sqlite3驱动
func init() {
	sql.Register("sqlite3", &SQLiteDriver{})
}

//https://github.com/mikespook/mymysql驱动
// Driver automatically registered in database/sql
var d = Driver{proto: "tcp", raddr: "127.0.0.1:3306"}
func init() {
	Register("SET NAMES utf8")
	sql.Register("mymysql", &d)
}
```
我们看到第三方数据库驱动都是通过调用这个函数来注册自己的数据库驱动名称以及相应的driver实现。在database/sql内部通过一个map来存储用户定义的相应驱动。
```Go

var drivers = make(map[string]driver.Driver)

drivers[name] = driver
```
因此通过database/sql的注册函数可以同时注册多个数据库驱动，只要不重复。

>在我们使用database/sql接口和第三方库的时候经常看到如下:

>		import (
>			"database/sql"
>		 	_ "github.com/mattn/go-sqlite3"
>		)

>新手都会被这个`_`所迷惑，其实这个就是Go设计的巧妙之处，我们在变量赋值的时候经常看到这个符号，它是用来忽略变量赋值的占位符，那么包引入用到这个符号也是相似的作用，这儿使用`_`的意思是引入后面的包名而不直接使用这个包中定义的函数，变量等资源。

>我们在2.3节流程和函数一节中介绍过init函数的初始化过程，包在引入的时候会自动调用包的init函数以完成对包的初始化。因此，我们引入上面的数据库驱动包之后会自动去调用init函数，然后在init函数里面注册这个数据库驱动，这样我们就可以在接下来的代码中直接使用这个数据库驱动了。

#### 1.3 driver.Driver
Driver是一个数据库驱动的接口，他定义了一个method： Open(name string)，这个方法返回一个数据库的Conn接口。
```Go

type Driver interface {
	Open(name string) (Conn, error)
}
```
返回的Conn只能用来进行一次goroutine的操作，也就是说不能把这个Conn应用于Go的多个goroutine里面。如下代码会出现错误
```Go

...
go goroutineA (Conn)  //执行查询操作
go goroutineB (Conn)  //执行插入操作
...
```
上面这样的代码可能会使Go不知道某个操作究竟是由哪个goroutine发起的,从而导致数据混乱，比如可能会把goroutineA里面执行的查询操作的结果返回给goroutineB从而使B错误地把此结果当成自己执行的插入数据。

第三方驱动都会定义这个函数，它会解析name参数来获取相关数据库的连接信息，解析完成后，它将使用此信息来初始化一个Conn并返回它。

#### 1.4 driver.Conn
Conn是一个数据库连接的接口定义，他定义了一系列方法，这个Conn只能应用在一个goroutine里面，不能使用在多个goroutine里面，详情请参考上面的说明。
```Go

type Conn interface {
	Prepare(query string) (Stmt, error)
	Close() error
	Begin() (Tx, error)
}
```
Prepare函数返回与当前连接相关的执行Sql语句的准备状态，可以进行查询、删除等操作。

Close函数关闭当前的连接，执行释放连接拥有的资源等清理工作。因为驱动实现了database/sql里面建议的conn pool，所以你不用再去实现缓存conn之类的，这样会容易引起问题。

Begin函数返回一个代表事务处理的Tx，通过它你可以进行查询,更新等操作，或者对事务进行回滚、递交。

#### 1.5 driver.Stmt
Stmt是一种准备好的状态，和Conn相关联，而且只能应用于一个goroutine中，不能应用于多个goroutine。
```Go

type Stmt interface {
	Close() error
	NumInput() int
	Exec(args []Value) (Result, error)
	Query(args []Value) (Rows, error)
}
```
Close函数关闭当前的链接状态，但是如果当前正在执行query，query还是有效返回rows数据。

NumInput函数返回当前预留参数的个数，当返回>=0时数据库驱动就会智能检查调用者的参数。当数据库驱动包不知道预留参数的时候，返回-1。

Exec函数执行Prepare准备好的sql，传入参数执行update/insert等操作，返回Result数据

Query函数执行Prepare准备好的sql，传入需要的参数执行select操作，返回Rows结果集


#### 1.6 driver.Tx
事务处理一般就两个过程，递交或者回滚。数据库驱动里面也只需要实现这两个函数就可以
```Go

type Tx interface {
	Commit() error
	Rollback() error
}
```
这两个函数一个用来递交一个事务，一个用来回滚事务。

#### 1.7 driver.Execer
这是一个Conn可选择实现的接口
```Go

type Execer interface {
	Exec(query string, args []Value) (Result, error)
}
```
如果这个接口没有定义，那么在调用DB.Exec,就会首先调用Prepare返回Stmt，然后执行Stmt的Exec，然后关闭Stmt。

#### 1.8 driver.Result
这个是执行Update/Insert等操作返回的结果接口定义
```Go

type Result interface {
	LastInsertId() (int64, error)
	RowsAffected() (int64, error)
}
```
LastInsertId函数返回由数据库执行插入操作得到的自增ID号。

RowsAffected函数返回query操作影响的数据条目数。

#### 1.9 driver.Rows
Rows是执行查询返回的结果集接口定义
```Go

type Rows interface {
	Columns() []string
	Close() error
	Next(dest []Value) error
}
```
Columns函数返回查询数据库表的字段信息，这个返回的slice和sql查询的字段一一对应，而不是返回整个表的所有字段。

Close函数用来关闭Rows迭代器。

Next函数用来返回下一条数据，把数据赋值给dest。dest里面的元素必须是driver.Value的值除了string，返回的数据里面所有的string都必须要转换成[]byte。如果最后没数据了，Next函数最后返回io.EOF。


#### 1.10 driver.RowsAffected
RowsAffected其实就是一个int64的别名，但是他实现了Result接口，用来底层实现Result的表示方式
```Go

type RowsAffected int64

func (RowsAffected) LastInsertId() (int64, error)

func (v RowsAffected) RowsAffected() (int64, error)
```
#### 1.11 driver.Value
Value其实就是一个空接口，他可以容纳任何的数据
```Go

type Value interface{}
```
drive的Value是驱动必须能够操作的Value，Value要么是nil，要么是下面的任意一种
```Go

int64
float64
bool
[]byte
string   [*]除了Rows.Next返回的不能是string.
time.Time
```
#### 1.12 driver.ValueConverter
ValueConverter接口定义了如何把一个普通的值转化成driver.Value的接口
```Go

type ValueConverter interface {
	ConvertValue(v interface{}) (Value, error)
}
```
在开发的数据库驱动包里面实现这个接口的函数在很多地方会使用到，这个ValueConverter有很多好处：

- 转化driver.value到数据库表相应的字段，例如int64的数据如何转化成数据库表uint16字段
- 把数据库查询结果转化成driver.Value值
- 在scan函数里面如何把driver.Value值转化成用户定义的值

#### 1.13 driver.Valuer
Valuer接口定义了返回一个driver.Value的方式
```Go

type Valuer interface {
	Value() (Value, error)
}
```
很多类型都实现了这个Value方法，用来自身与driver.Value的转化。

通过上面的讲解，你应该对于驱动的开发有了一个基本的了解，一个驱动只要实现了这些接口就能完成增删查改等基本操作了，剩下的就是与相应的数据库进行数据交互等细节问题了，在此不再赘述。

#### 1.14 database/sql
database/sql在database/sql/driver提供的接口基础上定义了一些更高阶的方法，用以简化数据库操作,同时内部还建议性地实现一个conn pool。
```Go

type DB struct {
	driver 	 driver.Driver
	dsn    	 string
	mu       sync.Mutex // protects freeConn and closed
	freeConn []driver.Conn
	closed   bool
}
```
我们可以看到Open函数返回的是DB对象，里面有一个freeConn，它就是那个简易的连接池。它的实现相当简单或者说简陋，就是当执行`db.prepare` -> `db.prepareDC`的时候会`defer dc.releaseConn`，然后调用`db.putConn`，也就是把这个连接放入连接池，每次调用`db.conn`的时候会先判断freeConn的长度是否大于0，大于0说明有可以复用的conn，直接拿出来用就是了，如果不大于0，则创建一个conn，然后再返回之。
