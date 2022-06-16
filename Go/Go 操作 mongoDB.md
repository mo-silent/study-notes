# Go 操作 mongoDB
> Author mogd 2022-05-06
> Update mogd 2022-05-25
> Adage `Take action and dive in head first.`

## 一、连接数据库

安装

```go
go get go.mongodb.org/mongo-driver/mongo
```

### 1.1 代码解析
连接数据库，这里封装成一个函数，返回 `mongo.Client`

```go
// MongoClient Create a database connection
//
// return *mongo.Client
func MongoClient() *mongo.Client {
	credential := options.Credential{
		AuthMechanism: "SCRAM-SHA-1",
		Username:      "mogd",
		Password:      "admin",
	}
	// ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	// defer cancel()
	clientOpts := options.Client().ApplyURI(uri).SetAuth(credential)
	client, err := mongo.Connect(context.TODO(), clientOpts)
	if err != nil {
		panic(err)
	}
	return client
}
```

`mongo.Connect()`必须传递一个 `context` 和一个 `options.ClientOptions` 对象，`client options` 用于设置连接字符串，以及配置 `driver` 的设置，如 `write concern`, `socket timeout` 等等；`mongo.Connect()` 会返回一个 `*mongo.Client` 和一个 `error`，通过 `*mongo.Client` 来操作 `mongodb`

> [options包文档](https://pkg.go.dev/go.mongodb.org/mongo-driver/mongo/options)

`options.ClientOptions` 的调用顺序会导致连接信息的不同，例如在 `SetAuth` 之前调用 `ApplyURI`，则 `SetAuth` 的凭据覆盖连接字符串中的值，反之 `ApplyURI` 的值会覆盖 `SetAuth`

这里的代码中 `SetAuth` 传入了 `options.Credential` 结构体，用于传递 `mongodb` 的身份验证凭据，看一下 `options.Credential` 的定义
```go
type Credential struct {
	AuthMechanism           string
	AuthMechanismProperties map[string]string
	AuthSource              string
	Username                string
	Password                string
	PasswordSet             bool
}
``` 
`AuthMechanism`: 用于身份验证的机制，`SCRAM-SHA-256`、`SCRAM-SHA-1`、`MONGODB-CR`、`PLAIN、GSSAPI`、`MONGODB-X509` 和 `MONGODB-AWS`

`AuthMechanismProperties`: 用于为某些机制指定其他配置选项
- `SERVICE_NAME`: 用于 `GSSAPI` 身份验证的服务名称。默认为 `mongodb`
- `CANONICALIZE_HOST_NAME`: `true or false`，驱动程序将规范化主机名以进行 `GSSAPI` 身份验证
- `SERVICE_REALM`: `GSSAPI` 认证的服务领域
- `SERVICE_HOST`: 用于 `GSSAPI` 身份验证的主机名
- `AWS_SESSION_TOKEN`: 用于 `MONGODB-AWS` 身份验证的 AWS 令牌

`AuthSource`: 用于身份验证的数据库名称
`username`: 用户名
`Password`: 密码
`PasswordSet`: 对于 `GSSAPI`，如果指定了密码，则必须为 `true`，即使密码为空字符串，如果未指定密码，则为 `false`，表示应从正在运行的进程的上下文中获取密码。对于其他机制，该字段被忽略

**数据库关闭**
通过 `mongo.Connect()` 获取到数据库连接后，当应用程序不再使用连接，需要关闭连接，否则会一直占用数据库的连接数；使用 `client.Disconnect()` 关闭

```go
	defer func() {
		if err := Client.Disconnect(context.TODO()); err != nil {
			panic(err)
		}
	}()
```

这里使用 `defer` 关键字，即在函数退出前，执行数据库连接的关闭操作

### 1.2 官方直连完整示例

```go
import (
	"context"
	"log"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
	"go.mongodb.org/mongo-driver/mongo/readpref"
)

func main() {
	// Create a Client to a MongoDB server and use Ping to verify that the
	// server is running.

	clientOpts := options.Client().ApplyURI("mongodb://localhost:27017")
	client, err := mongo.Connect(context.TODO(), clientOpts)
	if err != nil {
		log.Fatal(err)
	}
	defer func() {
		if err = client.Disconnect(context.TODO()); err != nil {
			log.Fatal(err)
		}
	}()

	// Call Ping to verify that the deployment is up and the Client was
	// configured successfully. As mentioned in the Ping documentation, this
	// reduces application resiliency as the server may be temporarily
	// unavailable when Ping is called.
	if err = client.Ping(context.TODO(), readpref.Primary()); err != nil {
		log.Fatal(err)
	}
}
```

通过 `client.Ping()` 测试数据库的连通性

## 二、Database 操作

数据库的操作，这里是列出所有数据库和删除数据库

### 2.1 ListDatabases
```go
// ListMongoDatabase list all non-empty database
//
// return mongo.ListDatabasesResult
func ListMongoDatabase(client *mongo.Client) mongo.ListDatabasesResult {
	// Use a filter to select databases.
	result, err := client.ListDatabases(
		context.TODO(),
		bson.D{
			{Key: "empty", Value: false},
		})
	if err != nil {
		panic(err)
	}
	// for _, db := range result.Databases {
	// 	fmt.Println(db.Name)
	// }
	return result
}
```

列出数据库的方法是属于 `mongo.Client` 对象的，所以在调用时，使用的是 `client.ListDatabases()` 这个方法；该方法传入三个参数，`context` 、`filter` 接口和 `options.ListDatabasesOptions`
> `func (c *Client) ListDatabases(ctx context.Context, filter interface{}, opts ...*options.ListDatabasesOptions) (ListDatabasesResult, error)`
- `context`: 上下文，一般用 `context.TODO()`
- `filter`: 接口，主要是传入 `bson.D{}`
- `options.ListDatabasesOptions`: 这个的定义 `...` 可以多个该类型的参数

`client.ListDatabases` 方法返回一个 `mongo.ListDatabasesResult`，该结构体定义如下：
```go
type ListDatabasesResult struct {
	// A slice containing one DatabaseSpecification for each database matched by the operation's filter.
	Databases []DatabaseSpecification

	// The total size of the database files of the returned databases in bytes.
	// This will be the sum of the SizeOnDisk field for each specification in Databases.
	TotalSize int64
}
// Databases 结构体
type DatabaseSpecification struct {
	Name       string // The name of the database.
	SizeOnDisk int64  // The total size of the database files on disk in bytes.
	Empty      bool   // Specfies whether or not the database is empty.
}
```

`ListDatabasesResult` 结构体两个属性，一个是切片类型的 `DatabaseSpecification` 结构体，另一个就是总大小

`DatabaseSpecification` 记录了每个数据库的名称，在磁盘的总字节大小，以及是否为空

### 2.2 Drop Database

删除数据库，是通过 `Database` 对象的 `Drop` 方法，该方法很简单，只需要一个 `context` 参数，而 `Database` 对象，通过 `client.Database()` 方法获取

`client.Database()` 方法传入两个参数
> `func (c *Client) Database(name string, opts ...*options.DatabaseOptions) *Database`
- `name`: 数据库名称, `string` 类型
- `opts`: 用于配置数据库的选项， `...*options.DatabaseOptions` 类型

```go
db := Client.Database("test")
// DropMongoDatabase drop mongo database
//
// return string
func DropMongoDatabase(db *mongo.Database) string {
	err := db.Drop(context.TODO())
	if err != nil {
		return fmt.Sprintln("err: ", err)
	}
	return fmt.Sprintln("drop database success")
}
```

## 三、集合

`MongoDB` 集合的操作，这里是列出所有集合、删除某个特定集合和创建单个集合

### 3.1 ListCollection

```go
// MongoListCollection list mongodb all collection
//
// param db *mongo.Database "a handle for a database"
//
// return []bson.M
func MongoListCollection(db *mongo.Database) []bson.M {
	result, err := db.ListCollections(
		context.TODO(),
		bson.D{},
	)
	if err != nil {
		panic(err)
	}
	var results []bson.M
	if err := result.All(context.TODO(), &results); err != nil {
		log.Fatal(err)
		return []bson.M{bson.M{"err": err}}
	}
	// fmt.Println(results)
	return results
}
```

列出所有集合的操作，是调用 `mongo.Database` 对象的 `ListCollections`，这个方法跟前面的 `client.ListDatabases` 方法很像，前两个参数都一样，第三个参数变成了 `options.ListCollectionsOptions`，返回值是 `mongo.Cursor` 和 `error`

> `func (db *Database) ListCollections(ctx context.Context, filter interface{}, opts ...*options.ListCollectionsOptions) (*Cursor, error)`

### 3.2 DropCollection
```go
// MongoDropCollection drop mongodb collection
//
// param coll *mongo.Collection "a handle for a collection"
//
// return string
func MongoDropCollection(coll *mongo.Collection) string {
	err := coll.Drop(context.TODO())
	if err != nil {
		return fmt.Sprintln("err: ", err)
	}
	return "drop collection success"
}
```

删除集合操作，调用了 `mongo.Collection` 对象的 `Drop` 方法，跟数据库的删除函数可以说是一样的
> `func (coll *Collection) Drop(ctx context.Context) error`

`mongo.Collection` 对象是通过 `mongo.Database` 的 `Collection` 方法获取的
> `func (db *Database) Collection(name string, opts ...*options.CollectionOptions) *Collection`

### 3.3 CreateCollection
```go
// MongoCreateCollection create mongodb collection
//
// param db *mongo.Database "a handle for a database"
//
// return string
func MongoCreateCollection(db *mongo.Database) string {
	opts := options.CreateCollection().SetValidator(bson.M{})
	err := db.CreateCollection(context.TODO(), "users", opts)
	if err != nil {
		return fmt.Sprintln("err: ", err)
	}
	return "create collection success"
}
```

## 四、文档
### 4.1 ListCollection
```go
// MongoListCollection list mongodb all collection
//
// param db *mongo.Database "a handle for a database"
//
// return []bson.M
func MongoListCollection(db *mongo.Database) []bson.M {
	result, err := db.ListCollections(
		context.TODO(),
		bson.D{},
	)
	if err != nil {
		panic(err)
	}
	var results []bson.M
	if err := result.All(context.TODO(), &results); err != nil {
		log.Fatal(err)
		return []bson.M{bson.M{"err": err}}
	}
	// fmt.Println(results)
	return results
}
```

### 4.2 DropCollection
```go
// MongoDropCollection drop mongodb collection
//
// param coll *mongo.Collection "a handle for a collection"
//
// return string
func MongoDropCollection(coll *mongo.Collection) string {
	err := coll.Drop(context.TODO())
	if err != nil {
		return fmt.Sprintln("err: ", err)
	}
	return "drop collection success"
}
```
### 4.3 CreateCollection
```go
// MongoCreateCollection create mongodb collection
//
// param db *mongo.Database "a handle for a database"
//
// return string
func MongoCreateCollection(db *mongo.Database) string {
	opts := options.CreateCollection().SetValidator(bson.M{})
	err := db.CreateCollection(context.TODO(), "users", opts)
	if err != nil {
		return fmt.Sprintln("err: ", err)
	}
	return "create collection success"
}
```
 
## 五、错误记录

### 5.1 连接 Replica Set出现错误

错误如下：
```go
panic: server selection error: server selection timeout, current topology: { Type: ReplicaSetNoPrimary, Servers: [{ Addr: 127.0.0.1:27017, Type: Unknown, Last error: connection() error occurred during connection handshake: dial tcp 127.0.0.1:27017: connectex: No connection could be made because the target machine actively refused it. }, ] }
```

具体代码 ([官方示例](https://www.mongodb.com/docs/drivers/go/current/fundamentals/connection/#connection-uri))

```go
package main

import (
	"context"
	"fmt"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
	"go.mongodb.org/mongo-driver/mongo/readpref"
)

// Connection URI
const uri = "mongodb://user:pass@sample.host:27017/?maxPoolSize=20&w=majority"

func main() {
	// Create a new client and connect to the server
	client, err := mongo.Connect(context.TODO(), options.Client().ApplyURI(uri))

	if err != nil {
		panic(err)
	}
	defer func() {
		if err = client.Disconnect(context.TODO()); err != nil {
			panic(err)
		}
	}()

	// Ping the primary
	if err := client.Ping(context.TODO(), readpref.Primary()); err != nil {
		panic(err)
	}

	fmt.Println("Successfully connected and pinged.")
}
```

**解决方法**

在连接的 `uri` 后面加上 `connect=direct`

`const uri = "mongodb://user:pass@sample.host:27017/?maxPoolSize=20&w=majority&connect=direct"`

[问题解决链接](https://stackoverflow.com/questions/56393513/docker-and-mongo-go-driver-server-selection-error)

> In contrast to the `mongo-go-driver`, by default it would perform server discovery and attempt to connect as a replica set. If you would like to connect as a single node, then you need to specify `connect=direct` in the connection URI. See also [Example Connect Direct](https://godoc.org/go.mongodb.org/mongo-driver/mongo#example-Connect--Direct)

### 5.2 vscode 提示 primitive.E 

```go
go.mongodb.org/mongo-driver/bson/primitive.E composite literal uses unkeyed fields
```

是因为使用了 `bson.D` 的原因，不影响运行，是因为写法不规范

规范写法：
```go
bson.D{primitive.E{Key: "empty", Value: false}}
// 简写
bson.D{{Key: "empty", Value: false}}
```