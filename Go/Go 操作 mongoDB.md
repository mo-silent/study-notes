# Go 操作 mongoDB
> Author mogd 2022-05-06
> Update mogd 2022-05-09
> Adage `Take action and dive in head first.`

## 一、连接数据库

安装

```go
go get go.mongodb.org/mongo-driver/mongo
```

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





## 错误记录

### 连接 Replica Set出现错误

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

### vscode 提示 primitive.E 

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