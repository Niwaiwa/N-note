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

## component

1. framwork
    * fiber
    * gin
2. ORM
    * GORM
    * Sqlx
3. websocket
    * gorilla/websocket
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

### framwork benchmark

* https://github.com/the-benchmarker/web-frameworks
* https://blog.golang.org/using-go-modules
* https://dev.to/techschoolguru/load-config-from-file-environment-variables-in-golang-with-viper-2j2d

### dotenv

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

## Reference

* Effective Go
    * https://go.dev/doc/effective_go
* How to Write Go Code
    * https://go.dev/doc/code
