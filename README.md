
[![Build Status](https://travis-ci.org/francoispqt/onelog.svg?branch=master)](https://travis-ci.org/francoispqt/onelog)
[![codecov](https://codecov.io/gh/francoispqt/onelog/branch/master/graph/badge.svg)](https://codecov.io/gh/francoispqt/onelog)
[![Go Report Card](https://goreportcard.com/badge/github.com/francoispqt/onelog)](https://goreportcard.com/report/github.com/francoispqt/onelog)
[![Go doc](http://img.shields.io/badge/go-documentation-blue.svg?style=flat-square
)](https://godoc.org/github.com/francoispqt/onelog)
![MIT License](https://img.shields.io/badge/license-mit-blue.svg?style=flat-square)

# Onelog
Onelog is a dead simple but very efficient JSON logger. 
It is one of the fastest JSON logger out there and the fastest when logging extra fields. Also, it is one of the logger with the lowest allocation.

It gives more control over log levels enabled by using bitwise operation for setting levels on a logger.

It is also modular as you can add a custom hook, define level text values, level and message keys.

Go 1.9 is required as it uses a type alias over gojay.Encoder.

It is named onelog as a reference to zerolog and because it sounds like `One Love` song from Bob Marley :)

## Get Started

```bash 
go get github.com/francoispqt/onelog
```

Basic usage:

```go
import "github.com/francoispqt/onelog"

func main() {
    // create a new Logger
    // first argument is an io.Writer
    // second argument is the level, which is an integer
    logger := onelog.New(
        os.Stdout, 
        onelog.ALL, // shortcut for onelog.DEBUG|onelog.INFO|onelog.WARN|onelog.ERROR|onelog.FATAL,
    )
    logger.Info("hello world !") // {"level":"info","message":"hello world"}
}
```

## Levels

Levels are ints mapped to a string. The logger will check if level is enabled with an efficient bitwise &(AND), if disabled, it returns right away which makes onelog the fastest when running disabled logging with 0 allocs and less than 1ns/op. [See benchmarks](#benchmarks)

When creating a logger you must use the `|` operator with different levels to toggle bytes. 

Example if you want levels INFO and WARN:
```go
logger := onelog.New(
    os.Stdout, 
    onelog.INFO|onelog.WARN,
)
```

This allows you to have a logger with different levels, for example you can do: 
```go
var logger *onelog.Logger

func init() {
    // if we are in debug mode, enable DEBUG lvl
    if os.Getenv("DEBUG") != "" {
        logger = onelog.New(
            os.Stdout, 
            onelog.ALL, // shortcut for onelog.DEBUG|onelog.INFO|onelog.WARN|onelog.ERROR|onelog.FATAL
        )
        return
    }
    logger = onelog.New(
        os.Stdout, 
        onelog.INFO|onelog.WARN|onelog.ERROR|onelog.FATAL,
    )
}
```

Available levels:
- onelog.DEBUG
- onelog.INFO
- onelog.WARN
- onelog.ERROR
- onelog.FATAL

You can change their textual values by doing, do this only once at runtime as it is not thread safe: 
```go
onelog.LevelText(onelog.INFO, "INFO")
```

## Hook

You can define a hook which will be run for every log message. 

Example:
```go 
logger := onelog.New(
    os.Stdout, 
    onelog.ALL,
)
logger.Hook(func(e onelog.Entry) {
    e.String("time", time.Now().Format(time.RFC3339))
})
logger.Info("hello world !") // {"level":"info","message":"hello world","time":"2018-05-06T02:21:01+08:00"}
```

## Logging

### Without extra fields
Logging without extra fields is easy as:
```go 
logger := onelog.New(
    os.Stdout, 
    onelog.ALL,
)
logger.Debug("i'm not sure what's going on") // {"level":"debug","message":"i'm not sure what's going on"}
logger.Info("breaking news !") // {"level":"info","message":"breaking news !"}
logger.Warn("beware !") // {"level":"warn","message":"beware !"}
logger.Error("my printer is on fire") // {"level":"error","message":"my printer is on fire"}
logger.Fatal("oh my...") // {"level":"fatal","message":"oh my..."}
```

### With extra fields 
Logging with extra fields is quite simple, specially if you have used gojay:
```go 
logger := onelog.New(
    os.Stdout, 
    onelog.ALL,
)

logger.DebugWithFields("i'm not sure what's going on", func(e onelog.Entry) {
    e.String("string", "foobar")
    e.Int("int", 12345)
    e.Int64("int64", 12345)
    e.Float("float64", 0.15)
    e.Bool("bool", true)
    e.Error("err", errors.New("someError"))
    e.ObjectFunc("user", func() {
        e.String("name", "somename")
    })
}) 
// {"level":"debug","message":"i'm not sure what's going on","string":"foobar","int":12345,"int64":12345,"float64":0.15,"bool":true,"err":"someError","user":{"name":"somename"}}

logger.InfoWithFields("breaking news !", func(e onelog.Entry) {
    e.String("userID", "123455")
}) 
// {"level":"info","message":"breaking news !","userID":"123456"}

logger.WarnWithFields("beware !", func(e onelog.Entry) {
    e.String("userID", "123455")
}) 
// {"level":"warn","message":"beware !","userID":"123456"}

logger.ErrorWithFields("my printer is on fire", func(e onelog.Entry) {
    e.String("userID", "123455")
}) 
// {"level":"error","message":"my printer is on fire","userID":"123456"}

logger.FatalWithFields("oh my...", func(e onelog.Entry) {
    e.String("userID", "123455")
}) 
// {"level":"fatal","message":"oh my...","userID":"123456"}
```

## Accumulate context
You can create get a logger with some accumulated context that will be included on all logs created by this logger.

To do that, you must call the `With` method on a logger.
Internally it creates a copy of the current logger and returns it. 

Example: 
```go 
logger := onelog.New(
    os.Stdout, 
    onelog.ALL,
).With(func(e onelog.Entry) {
    e.String("userID", "123456")
})

logger.Info("user logged in") // {"level":"info","message":"user logged in","userID","123456"}

logger.Debug("wtf?") // {"level":"debug","message":"wtf?","userID","123456"}

logger.ErrorWithFields("Oops", func(e onelog.Entry) {
    e.String("error_code", "ROFL")
}) // {"level":"error","message":"oops","userID","123456","error_code":"ROFL"}
```

## Change levels txt values, message and/or level keys
You can change globally the levels values by calling the function: 
```go
onelog.LevelText(onelog.INFO, "INFO")
```

You can change the key of the message by calling the function: 
```go 
onelog.MsgKey("msg")
```

You can change the key of the level by calling the function: 
```go 
onelog.LevelKey("lvl")
```

Beware, these changes are global (affects all instances of the logger). Also, these function should be called only once at runtime to avoid any data race issue.

# Benchmarks

The benchmark data presented here is the one from Uber's benchmark suite where we added onelog. 

A pull request will be submitted to Zap to integrate onelog in the benchmarks.

Benchmarks are here: https://github.com/francoispqt/zap/tree/onelog-bench/benchmarks

## Disabled Logging
|             | ns/op | bytes/op     | allocs/op |
|-------------|-------|--------------|-----------|
| Zap         | 7.70  | 0            | 0         |
| zerolog     | 2.67  | 0            | 0         |
| logrus      | 12.1  | 16           | 1         |
| onelog      | 0.88  | 0            | 0         |

## Disabled with fields
|             | ns/op | bytes/op     | allocs/op |
|-------------|-------|--------------|-----------|
| Zap         | 208   | 768          | 5         |
| zerolog     | 68.7  | 128          | 4         |
| logrus      | 721   | 1493         | 12        |
| onelog      | 1.31  | 0            | 0         |

## Logging basic message
|             | ns/op | bytes/op     | allocs/op |
|-------------|-------|--------------|-----------|
| Zap         | 203   | 0            | 0         |
| zerolog     | 154   | 0            | 0         |
| logrus      | 1256  | 1554         | 24        |
| onelog      | 317   | 0            | 0         |

## Logging basic message and accumulated context
|             | ns/op | bytes/op     | allocs/op |
|-------------|-------|--------------|-----------|
| Zap         | 276   | 0            | 0         |
| zerolog     | 164   | 0            | 0         |
| logrus      | 1256  | 1554         | 24        |
| onelog      | 333   | 0            | 0         |

## Logging message with extra fields
|             | ns/op | bytes/op     | allocs/op |
|-------------|-------|--------------|-----------|
| Zap         | 1764  | 770          | 5         |
| zerolog     | 1210  | 128          | 4         |
| logrus      | 13211 | 13584        | 129       |
| onelog      | 1047  | 128          | 4         |