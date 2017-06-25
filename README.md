# Practical Persistence in Go: Organising Database Access (RU) (Практическое использование в Go: Организация доступа к базам данных)

------

Оригинал: http://www.alexedwards.net/blog/organising-database-access

------

Несколько недель назад кто то создал [общение на Reddit](https://www.reddit.com/r/golang/comments/38hkor/go_best_practice_for_accessing_database_in/) с просьбой:
>В контексте веб-приложения, что бы Вы использовали в качестве лучшей практики Go для доступа к базе данных в (HTTP или других) обработчиках? <blockquote></blockquote>

Ответы, которые он получил, были разнообразными и интересными. Некоторые люди посоветовали использовать внедрение зависимости, некоторые поддержали идею использования простых глобальных переменных, другие предложили поместить указатель пула соединений в x/net/context.

Что касается меня? Я думаю что правильный ответ зависит от проекта. 

Какая общая структура и размер проекта? Какой подход используется вами для тестирования? Какое развитие получит в будущем? Все эти вещи и многое другое частично влияют на то, какой подход подойдет для вас.

*What's the overall structure and size of the project? What's your approach to testing? How is it likely to grow in the future?* All these things and more should play a part when you pick an approach to take.

В этом посте я рассматриваю четыре разных подхода к организации вашего кода и структурирование доступа к пулу подключений к базе данных.

# Глобальные переменные

Первый подход, который мы рассмотрим является общим и простым - возьмем указатель на пул соеденений к базе данных и положим в глобальную переменную.

Что бы код выглядил красивым и DRY (Don't Repeat Yourself - рус. не повторяйся), вы иногда можете комбинировать с функцией инициализации, которая будет устанавливать глобальный пул соединений из других пакетов и тестов. 

Мне нравятся конкретные примеры, давайте продолжим работу с базой данных онлайн магазина и кодом из моего предыдущего [поста](http://www.alexedwards.net/blog/practical-persistence-sql) . Мы рассмотрим создание простых приложений с MVC (Model View Controller) подобной структурой - с обработчиками HTTP в основном приложении и отдельным пакетом моделей содержащей глобавльные переменные для БД, функцию InitDB(), и нашу логику по базе данных. 

```
bookstore
├── main.go
└── models
	├── books.go
	└── db.go
```

File: main.go
```go
package main

import (
    "bookstore/models"
    "fmt"
    "net/http"
)

func main() {
    models.InitDB("postgres://user:pass@localhost/bookstore")

    http.HandleFunc("/books", booksIndex)
    http.ListenAndServe(":3000", nil)
}

func booksIndex(w http.ResponseWriter, r *http.Request) {
    if r.Method != "GET" {
        http.Error(w, http.StatusText(405), 405)
        return
    }
    bks, err := models.AllBooks()
    if err != nil {
        http.Error(w, http.StatusText(500), 500)
        return
    }
    for _, bk := range bks {
        fmt.Fprintf(w, "%s, %s, %s, £%.2f\n", bk.Isbn, bk.Title, bk.Author, bk.Price)
    }
}
```

File: models/db.go
```go
package models

import (
    "database/sql"
    _ "github.com/lib/pq"
    "log"
)

var db *sql.DB

func InitDB(dataSourceName string) {
    var err error
    db, err = sql.Open("postgres", dataSourceName)
    if err != nil {
        log.Panic(err)
    }

    if err = db.Ping(); err != nil {
        log.Panic(err)
    }
}
```

File: models/books.go
```go
package models

type Book struct {
    Isbn   string
    Title  string
    Author string
    Price  float32
}

func AllBooks() ([]*Book, error) {
    rows, err := db.Query("SELECT * FROM books")
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    bks := make([]*Book, 0)
    for rows.Next() {
        bk := new(Book)
        err := rows.Scan(&bk.Isbn, &bk.Title, &bk.Author, &bk.Price)
        if err != nil {
            return nil, err
        }
        bks = append(bks, bk)
    }
    if err = rows.Err(); err != nil {
        return nil, err
    }
    return bks, nil
}
```

Если вы запустите приложение и выполните запрос на /books вы должны получить ответ похожий на:
```bash
$ curl -i localhost:3000/books
HTTP/1.1 200 OK
Content-Length: 205
Content-Type: text/plain; charset=utf-8

978-1503261969, Emma, Jayne Austen, £9.44
978-1505255607, The Time Machine, H. G. Wells, £5.99
978-1503379640, The Prince, Niccolò Machiavelli, £6.99
```

Использование глобаьных переменных потенциально подходит, если:
* Вся логика по базам данных находится в одном пакете. 
* Ваше приложение достаточно маленькое, что бы отслеживание глобальных переменных в уме не создавало вам проблем.
* Подход к тестированию означает, что вы не нуждаетесь в проверке базы данных и не запускаете параллельные тесты.

В примере выше использование глобальных переменных замечательно подходит. Но что произойдет в более сложных приложениях, когда логика базы данных используется в нескольких пакетах?

Один из вариантов это вызывать InitDB несколько раз, но такой подход быстро может стать неуклюжим и я лично вижу это немного странным (легко забыть инициализировать пул коннектов и полученть панику вызова пустого указателя во время выполнения). Второй вариант это создание отдельного конфигурационного пакета с экспортируемой переменной БД и импортировать "yourproject/config" в каждый файл, где это необходимо. Если не понятно о чем идет речь, на всякий случай я добавил простой пример [пример](https://gist.github.com/alexedwards/8b4b0cd4495d7c3abadd). 

# Внедрение зависимости

Во втором подходе мы рассмотрим внедрение зависимости. В нашем примере, мы явно хотим передать указатель к пулу соединений в наши обработчики HTTP и затем в нашу логику базы данных.

В реальном мире приложения имеют вероятно дополнительный уровень (конкурентно безопасный) в котором находятся элементы к которым ваши обработчики имеют доступ. Такими могут быть указатель но логгер или кэш, а так же пул соединений с базой данных.

Для проектов, в которых все ваши обработчики находятся в одном пакете, аккуратный подход состоит в том, что бы все элементы находились в пользовательском типе Env:

```go
type Env struct {
    db *sql.DB
    logger *log.Logger
    templates *template.Template
}
```
... и затем определить ваши обработчики и методы там же, где и Env. Это обеспечивает чистую и характерный способ для создания пула соединений (и для других элементов) для ваших обработчиков. Полный пример:

File: main.go
```go
package main

import (
    "bookstore/models"
    "database/sql"
    "fmt"
    "log"
    "net/http"
)

type Env struct {
    db *sql.DB
}

func main() {
    db, err := models.NewDB("postgres://user:pass@localhost/bookstore")
    if err != nil {
        log.Panic(err)
    }
    env := &Env{db: db}

    http.HandleFunc("/books", env.booksIndex)
    http.ListenAndServe(":3000", nil)
}

func (env *Env) booksIndex(w http.ResponseWriter, r *http.Request) {
    if r.Method != "GET" {
        http.Error(w, http.StatusText(405), 405)
        return
    }
    bks, err := models.AllBooks(env.db)
    if err != nil {
        http.Error(w, http.StatusText(500), 500)
        return
    }
    for _, bk := range bks {
        fmt.Fprintf(w, "%s, %s, %s, £%.2f\n", bk.Isbn, bk.Title, bk.Author, bk.Price)
    }
}
```

File: models/db.go
```go
package models

import (
    "database/sql"
    _ "github.com/lib/pq"
)

func NewDB(dataSourceName string) (*sql.DB, error) {
    db, err := sql.Open("postgres", dataSourceName)
    if err != nil {
        return nil, err
    }
    if err = db.Ping(); err != nil {
        return nil, err
    }
    return db, nil
}
```

File: models/books.go
```go
package models

import "database/sql"

type Book struct {
    Isbn   string
    Title  string
    Author string
    Price  float32
}

func AllBooks(db *sql.DB) ([]*Book, error) {
    rows, err := db.Query("SELECT * FROM books")
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    bks := make([]*Book, 0)
    for rows.Next() {
        bk := new(Book)
        err := rows.Scan(&bk.Isbn, &bk.Title, &bk.Author, &bk.Price)
        if err != nil {
            return nil, err
        }
        bks = append(bks, bk)
    }
    if err = rows.Err(); err != nil {
        return nil, err
    }
    return bks, nil
}
```

# Или использовать замыкания...

Если вы не хотите определять ваши обработчики и методы в Env, альтернативным подходом будет использование логики обработчиков в замыкании и закрытие переменной Env следующим образом:

File: main.go
```go
package main

import (
    "bookstore/models"
    "database/sql"
    "fmt"
    "log"
    "net/http"
)

type Env struct {
    db *sql.DB
}

func main() {
    db, err := models.NewDB("postgres://user:pass@localhost/bookstore")
    if err != nil {
        log.Panic(err)
    }
    env := &Env{db: db}

    http.Handle("/books", booksIndex(env))
    http.ListenAndServe(":3000", nil)
}

func booksIndex(env *Env) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Method != "GET" {
            http.Error(w, http.StatusText(405), 405)
            return
        }
        bks, err := models.AllBooks(env.db)
        if err != nil {
            http.Error(w, http.StatusText(500), 500)
            return
        }
        for _, bk := range bks {
            fmt.Fprintf(w, "%s, %s, %s, £%.2f\n", bk.Isbn, bk.Title, bk.Author, bk.Price)
        }
    })
}
```

Внедрение зависимостей является хорошим подходом, когда:

* Все ваши обработчики находятся в одном пакете. 
* Все обработчики имеют один и тот же набор зависимостей.
* Подход к тестированию означает, что вы не нуждаетесь в проверке базы данных и не запускаете параллельные тесты.

Еще раз, вы можете использовать этот подход, если ваши обработчики и логика базы данных будут распределены по нескольким пакетам. Один из способов добиться этого, создать отдельный конфигурационный пакет, экспортируемый тип Env и закрытие конфигурацию. Один из способов использования Env в примере выше. А так же простой [пример](https://gist.github.com/alexedwards/5cd712192b4831058b21).

# Использование интерфейсов