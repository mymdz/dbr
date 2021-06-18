gocraft/dbr provides additions to Go's database/sql for super fast performance and convenience.

## Отличия от оригинальной библиотеки

* Добавлена поддержка clickhouse (аналогично mailru/dbr, но от актуальной версии оригинала с исправленными багами)
* Реализована возможность автоматически повторять неудачные запросы

### Clickhouse

Для подключения к Clickhouse рекомендую библиотеку github.com/mailru/go-clickhouse. Пример использования:

```go
import (
    "fmt"
    _ "github.com/lib/pq"
    "github.com/mailru/go-clickhouse"
    _ "github.com/mailru/go-clickhouse"
    "github.com/mymdz/dbr"
    "github.com/sirupsen/logrus"
)

func GetClickHouseDSN(dbConfig *config.Db) string {
	clickConf := clickhouse.Config{
		User:     dbConfig.Username,
		Password: dbConfig.Password,
		Scheme:   "http",
		Host:     fmt.Sprintf("%s:%d", dbConfig.Host, dbConfig.Port),
		Database: dbConfig.DbName,
	}

	return clickConf.FormatDSN()
}

func main() {
    connect, err := dbr.Open(
        "clickhouse",
        connector.GetClickHouseDSN(dbConfig),
        createDBEventReceiver(true), // ниже подробнее об этом
        )
    if err != nil {
        logrus.Error("Failed to connect to ClickHouse: ", err)
        return err
    }
}
```

Исходная библиотека позволяет логировать все запросы, улетающие в базу, ловить ошибки и следить за временем выполнения зпросов. 
Для этого необходимо реализовать интерфейс EventReceiver (для ошибок и таймингов) и TracingEventReceiver для логирования всех запросов до начала выполнения.
Ниже пример реализации:

```go
// NullEventReceiver is a sentinel EventReceiver; use it if the caller doesn't supply one
type DatabaseEventReceiver struct {
	logQueries bool
	PromErr    prometheus.Counter
	PromTiming prometheus.Histogram
}

func createDBEventReceiver(logQueries bool) *DatabaseEventReceiver {
	var erc = &DatabaseEventReceiver{
		PromErr: prometheus.NewCounter(prometheus.CounterOpts{
			Name:      "failed_queries",
			Help:      "How long it took to process the request",
		}),
		PromTiming: prometheus.NewHistogram(
			prometheus.HistogramOpts{
				Name:      "query_time",
				Help:      "How long it took to process the request",
				Buckets:   []float64{.00005, .0001, .0005, .001, .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
			},
		),

		logQueries: logQueries,
	}

	prometheus.MustRegister(erc.PromErr)
	prometheus.MustRegister(erc.PromTiming)

	return erc
}

// Event receives a simple notification when various events occur
func (n *DatabaseEventReceiver) Event(eventName string) {}

// EventKv receives a notification when various events occur along with
// optional key/value data
func (n *DatabaseEventReceiver) EventKv(eventName string, kvs map[string]string) {}

// EventErr receives a notification of an error if one occurs
func (n *DatabaseEventReceiver) EventErr(eventName string, err error) error { return err }

// EventErrKv receives a notification of an error if one occurs along with
// optional key/value data
func (n *DatabaseEventReceiver) EventErrKv(eventName string, err error, kvs map[string]string) error {
    n.PromErr.Add(1)
    retries, ok := kvs["retries_left"]
    if !ok {
        retries = "0"
    }
    logrus.Errorf("Database query failure (will be retried %s times): %s", retries, kvs["sql"])
    return err
}

// Timing receives the time an event took to happen
func (n *DatabaseEventReceiver) Timing(eventName string, nanoseconds int64) {}

// TimingKv receives the time an event took to happen along with optional key/value data
func (n *DatabaseEventReceiver) TimingKv(eventName string, nanoseconds int64, kvs map[string]string) {
	n.PromTiming.Observe(time.Duration(nanoseconds).Seconds())
}

func (n *DatabaseEventReceiver) SpanStart(ctx context.Context, eventName, query string) context.Context {
	if n.logQueries {
		logrus.Infof(query)
	}
	return ctx
}

func (n *DatabaseEventReceiver) SpanError(ctx context.Context, err error) {}

func (n *DatabaseEventReceiver) SpanFinish(ctx context.Context) {}
```

### Повторение запросов

При ошибках можно автоматически повторять запрос с какой-то задержкой. Пример:
```go
connect.SetRetryConfig(dbr.RetryConfig{
    InitialDelay: 10 * time.Millisecond, // первоначальная задержка перед повторным запросом
    Retries:      2, // количество повторов
    ProgressiveFactor: 1, // Если например 2, то задержка при дополнительных попытках будет 10, 20, 40, 80...
})
```
RetryConfig задается для подключения, но можно выставить свой RetryConfig для отдельного запроса:
```go
connect.Select("name").
    From("users").
    Where(dbr.Eq("user_id", user.Id)).
    WithRetryConfig(dbr.RetryConfig{ Retries: 2 }).
    Load(&userName)
```


# Документация оригинальной библиотеки
```
$ go get -u github.com/gocraft/dbr/v2
```

```go
import "github.com/gocraft/dbr/v2"
```

## Driver support

* MySQL
* PostgreSQL
* SQLite3
* MsSQL

## Examples

See [godoc](https://godoc.org/github.com/gocraft/dbr) for more examples.

### Open connections

```go
// create a connection (e.g. "postgres", "mysql", or "sqlite3")
conn, _ := Open("postgres", "...", nil)
conn.SetMaxOpenConns(10)

// create a session for each business unit of execution (e.g. a web request or goworkers job)
sess := conn.NewSession(nil)

// create a tx from sessions
sess.Begin()
```

### Create and use Tx

```go
sess := mysqlSession
tx, err := sess.Begin()
if err != nil {
	return
}
defer tx.RollbackUnlessCommitted()

// do stuff...

tx.Commit()
```

### SelectStmt loads data into structs

```go
// columns are mapped by tag then by field
type Suggestion struct {
	ID	int64		// id, will be autoloaded by last insert id
	Title	NullString	`db:"subject"`	// subjects are called titles now
	Url	string		`db:"-"`	// ignored
	secret	string		// ignored
}

// By default gocraft/dbr converts CamelCase property names to snake_case column_names.
// You can override this with struct tags, just like with JSON tags.
// This is especially helpful while migrating from legacy systems.
var suggestions []Suggestion
sess := mysqlSession
sess.Select("*").From("suggestions").Load(&suggestions)
```

### SelectStmt with where-value interpolation

```go
// database/sql uses prepared statements, which means each argument
// in an IN clause needs its own question mark.
// gocraft/dbr, on the other hand, handles interpolation itself
// so that you can easily use a single question mark paired with a
// dynamically sized slice.

sess := mysqlSession
ids := []int64{1, 2, 3, 4, 5}
sess.Select("*").From("suggestions").Where("id IN ?", ids)
```

### SelectStmt with joins

```go
sess := mysqlSession
sess.Select("*").From("suggestions").
	Join("subdomains", "suggestions.subdomain_id = subdomains.id")

sess.Select("*").From("suggestions").
	LeftJoin("subdomains", "suggestions.subdomain_id = subdomains.id")

// join multiple tables
sess.Select("*").From("suggestions").
	Join("subdomains", "suggestions.subdomain_id = subdomains.id").
	Join("accounts", "subdomains.accounts_id = accounts.id")
```

### SelectStmt with raw SQL

```go
SelectBySql("SELECT `title`, `body` FROM `suggestions` ORDER BY `id` ASC LIMIT 10")
```

### InsertStmt adds data from struct

```go
type Suggestion struct {
	ID		int64
	Title		NullString
	CreatedAt	time.Time
}
sugg := &Suggestion{
	Title:		NewNullString("Gopher"),
	CreatedAt:	time.Now(),
}
sess := mysqlSession
sess.InsertInto("suggestions").
	Columns("title").
	Record(&sugg).
	Exec()

// id is set automatically
fmt.Println(sugg.ID)
```

### InsertStmt adds data from value

```go
sess := mysqlSession
sess.InsertInto("suggestions").
	Pair("title", "Gopher").
	Pair("body", "I love go.")
```


## Benchmark (2018-05-11)

```
BenchmarkLoadValues/sqlx_10-8         	    5000	    407318 ns/op	    3913 B/op	     164 allocs/op
BenchmarkLoadValues/dbr_10-8          	    5000	    372940 ns/op	    3874 B/op	     123 allocs/op
BenchmarkLoadValues/sqlx_100-8        	    2000	    584197 ns/op	   30195 B/op	    1428 allocs/op
BenchmarkLoadValues/dbr_100-8         	    3000	    558852 ns/op	   22965 B/op	     937 allocs/op
BenchmarkLoadValues/sqlx_1000-8       	    1000	   2319101 ns/op	  289339 B/op	   14031 allocs/op
BenchmarkLoadValues/dbr_1000-8        	    1000	   2310441 ns/op	  210092 B/op	    9040 allocs/op
BenchmarkLoadValues/sqlx_10000-8      	     100	  17004716 ns/op	 3193997 B/op	  140043 allocs/op
BenchmarkLoadValues/dbr_10000-8       	     100	  16150062 ns/op	 2394698 B/op	   90051 allocs/op
BenchmarkLoadValues/sqlx_100000-8     	      10	 170068209 ns/op	31679944 B/op	 1400053 allocs/op
BenchmarkLoadValues/dbr_100000-8      	      10	 147202536 ns/op	23680625 B/op	  900061 allocs/op
```

## Thanks & Authors
Inspiration from these excellent libraries:
* [sqlx](https://github.com/jmoiron/sqlx) - various useful tools and utils for interacting with database/sql.
* [Squirrel](https://github.com/lann/squirrel) - simple fluent query builder.

Authors:
* Jonathan Novak -- [https://github.com/cypriss](https://github.com/cypriss)
* Tai-Lin Chu -- [https://github.com/taylorchu](https://github.com/taylorchu)
* Sponsored by [UserVoice](https://eng.uservoice.com)

Contributors:
* Paul Bergeron -- [https://github.com/dinedal](https://github.com/dinedal) - SQLite dialect

## License
[![FOSSA Status](https://app.fossa.io/api/projects/git%2Bgithub.com%2Fgocraft%2Fdbr.svg?type=large)](https://app.fossa.io/projects/git%2Bgithub.com%2Fgocraft%2Fdbr?ref=badge_large)
