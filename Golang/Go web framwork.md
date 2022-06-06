###### tags: `Go`
# Go web framwork

## start go project

```
go mod init go_project
go mod tidy
```

hello world (main.go)

```
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	r.GET("/", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"data": "hello world"})
	})

	r.Run()
}

```

## Build and run in windows

```
go build main.go && ./main.exe
```

## test

```
go test
```

## some problem for tidy with version 1.17

```
go mod tidy -go=1.17
```

## component

1. framwork
    * fiber
    * gin
    * Echo
    * go-zero
2. ORM
    * GORM
    * Sqlx
    * xorm
    * golang-migrate/migrate
3. websocket
    * gorilla/websocket
    * gobwas
    * fiber
4. schedule
    * cron
    * go-co-op/gocron
5. job queue
    * buffered channel (goroutine)
    * gocraft/work
    * benmanns/goworker
    * streadway/amqp (rabbitmq client)
    * confluentinc/confluent-kafka-go (kafka)
    * segmentio/kafka-go (kafka)
    * NATS
    * hibiken/asynq
6. driver
    * go-redis (redis client)
    * gomodule/redigo (redis client)
7. doc(OpenApi)
    * swagger-api/swagger-codegen
    * go-swagger/go-swagger
    * swaggo/swag
    * grpc-ecosystem/grpc-gateway
8. dotenv
    * os
    * godotenv
    * viper
9. DI (dependency injection)
    * wire
10. log
    * logrus
    * zap
    * zerolog
11. template
    * pagoda (base Echo)

### framwork benchmark

* https://github.com/the-benchmarker/web-frameworks
* https://blog.golang.org/using-go-modules
* https://dev.to/techschoolguru/load-config-from-file-environment-variables-in-golang-with-viper-2j2d

### Gin concept

project components

* dotenv(godotenv)
* custom logger(log, gin log)
* custom recovery
* gracefully shutdown
* db connection
* redis connection

request middleware flow

```
Request -> Route Parser -> Middleware -> Route Handler -> Middleware -> Response
```

### dotenv (viper)

```
package configs

import "github.com/spf13/viper"

type Config struct {
	DB_Driver   string `mapstructure:"DB_DRIVER"`
	DB_HOST     string `mapstructure:"DB_HOST"`
	DB_PORT     string `mapstructure:"DB_PORT"`
	DB_USERNAME string `mapstructure:"DB_USERNAME"`
	DB_PASSWORD string `mapstructure:"DB_PASSWORD"`
	DB_NAME     string `mapstructure:"DB_NAME"`
	DB_SSL      string `mapstructure:"DB_SSL"`
	DB_Source   string `mapstructure:"DB_SOURCE"`
}

// var DB_Driver string
// var DB_HOST string
// var DB_PORT string
// var DB_USERNAME string
// var DB_PASSWORD string
// var DB_NAME string
// var DB_SSL string
// var DB_Source string

func LoadConfig(path string) (config Config, err error) {
	viper.AddConfigPath(path)
	viper.SetConfigName("app")
	viper.SetConfigType("env")

	viper.AutomaticEnv()

	err = viper.ReadInConfig()
	if err != nil {
		return
	}

	// DB_Driver = viper.GetString("DB_DRIVER")
	// DB_HOST = viper.GetString("DB_HOST")
	// DB_PORT = viper.GetString("DB_PORT")
	// DB_USERNAME = viper.GetString("DB_USERNAME")
	// DB_PASSWORD = viper.GetString("DB_PASSWORD")
	// DB_NAME = viper.GetString("DB_NAME")
	// DB_SSL = viper.GetString("DB_SSL")
	// DB_Source = viper.GetString("DB_SOURCE")

	err = viper.Unmarshal(&config)
	return
}

```

### dotenv (godotenv)

基本用法

```
	err := godotenv.Load()
	if err != nil {
		panic(err)
	}
	gin_mode := os.Getenv("GIN_MODE")
```

Config 用法 config.go

```
type Config struct {
	Env       string
	GinMode   string
	Port      string
	SecretKey string
	DB        *DBConfig
	RedisDB   *RedisConfig
}

type DBConfig struct {
	Host     string
	Port     string
	Username string
	Password string
	Name     string
	Charset  string
}

type RedisConfig struct {
	Host     string
	Port     string
	Password string
}

var configSet *Config

func LoadEnv() {
	err := godotenv.Load()
	if err != nil {
		panic(err)
	}
	configSet = &Config{
		Env:       os.Getenv("ENV"),
		GinMode:   os.Getenv("GIN_MODE"),
		Port:      os.Getenv("PORT"),
		SecretKey: os.Getenv("SECRET_KEY"),
		DB: &DBConfig{
			Host:     os.Getenv("DB_HOST"),
			Port:     os.Getenv("DB_PORT"),
			Username: os.Getenv("DB_USERNAME"),
			Password: os.Getenv("DB_PASSWORD"),
			Name:     os.Getenv("DB_NAME"),
			Charset:  "utf8mb4",
		},
		RedisDB: &RedisConfig{
			Host:     os.Getenv("REDIS_HOST"),
			Port:     os.Getenv("REDIS_PORT"),
			Password: os.Getenv("REDIS_PASSWORD"),
		},
	}
}

func GetConfig() *Config {
	return configSet
}
```


### gorm db connection

```
package models

import (
	"go-gin-site/configs"
	"log"
	"time"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

var DB *gorm.DB

func ConnectDataBase() {
	config, err := configs.LoadConfig(".")
	if err != nil {
		log.Fatal("cannot load config:", err)
	}
	// var dsn = config.DB_USERNAME + ":" + config.DB_PASSWORD + "@tcp(" + config.DB_HOST + ":" + config.DB_PORT + ")/" + config.DB_NAME + "?charset=utf8mb4&parseTime=True&loc=Local"
	// dsn := configs.DB_USERNAME + ":" + configs.DB_PASSWORD + "@tcp(" + configs.DB_HOST + ":" + configs.DB_PORT + ")/" + configs.DB_NAME + "?charset=utf8mb4&parseTime=True&loc=Local"
	var dsn = config.DB_Source
	// fmt.Printf(dsn)
	database, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})

	if err != nil {
		panic("Failed to connect to database!")
	}
	dbpool, err := database.DB()
	if err != nil {
		panic(err)
	}

	dbpool.SetConnMaxLifetime(time.Duration(3600) * time.Second) // 每條連線的存活時間
	dbpool.SetMaxOpenConns(100)                                  // 最大連線數
	dbpool.SetMaxIdleConns(10)                                   // 最大閒置連線數
	database.AutoMigrate(&Book{})

	DB = database
}

```

### golang-migrate/migrate

* [download binary file](https://github.com/golang-migrate/migrate/releases)

create migration file

```
./migrate create -ext sql -dir db/migrations -seq create_users
```

up file (DDL by database)

```
BEGIN;
CREATE TABLE IF NOT EXISTS users (
    id INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
    username VARCHAR(255) NOT NULL,
    account VARCHAR(255) NOT NULL,
    password VARCHAR(255) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT current_timestamp(),
    updated_at DATETIME NOT NULL DEFAULT current_timestamp(),
    PRIMARY KEY (id),
    UNIQUE INDEX `account` (`account`),
);
COMMIT;
```

down file (DDL)

```
DROP TABLE IF EXISTS users;
```

mysql connection

```
mysql://user:password@host:port/dbname?sslmode=disable
```

migrate up

```
migrate -database ${DATABASE_URL} -path db/migrations up 1
```

migrate down

```
migrate -database ${DATABASE_URL} -path db/migrations down 1
```

### bcrypt for password

```
package crypto

import (
	"golang.org/x/crypto/bcrypt"
)

// 暗号(Hash)化
func PasswordEncrypt(password string) (string, error) {
	hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	return string(hash), err
}

// 暗号(Hash)と入力された平パスワードの比較
func CompareHashAndPassword(hash, password string) error {
	return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
}

```

### Gin validation

use point to bind data must have key and null string
使用指標宣告取得資料, 必須有Key且可為空字串

```
type MyModel struct {
    MyField *int `json:"my_field" binding:"required"`
}

func MyFunction(c *gin.Context) {
    var myModel MyModel;
    err := c.shouldBind(&myModel);

    // the rest of the logic 
    // - if you do want to use the int value, you have to de-reference it, like so:

    var myIntValue = *myModel.MyField
}
```


## Reference

* Effective Go
    * https://go.dev/doc/effective_go
* How to Write Go Code
    * https://go.dev/doc/code
