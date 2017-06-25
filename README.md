# Practical Persistence in Go: Organising Database Access (RU) (Практическое использование в Go: Организация доступа к базам данных)

------

Оригинал: http://www.alexedwards.net/blog/organising-database-access

------

Несколько недель назад кто то создал общение на Reddit с просьбой:
>В контексте веб-приложения, что бы Вы использовали в качестве лучшей практики Go для доступа к базе данных в (HTTP или других) обработчиках? <blockquote></blockquote>

Ответы, которые он получил, были разнообразными и интересными. Некоторые люди посоветовали использовать dependency injection, некоторые поддержали идею использования простых глобальных переменных, другие предложили поместить указатель пула соединений в x/net/context.

Что касается меня? Я думаю что правильный ответ зависит от проекта. 

Какая общая структура и размер проекта? Какой подход используется вами для тестирования? Какое развитие получит в будущем? Все эти вещи и многое другое частично влияют на то, какой подход подойдет для вас.

*What's the overall structure and size of the project? What's your approach to testing? How is it likely to grow in the future?* All these things and more should play a part when you pick an approach to take.

В этом посте я рассматриваю четыре разных подхода к организации вашего кода и структурирование доступа к пулу подключений к базе данных.

#Глобальные переменные

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

```File: main.go
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
}```

```File: models/db.go
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
}```

```File: models/books.go
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
}```

Если вы запустите приложение и выполните запрос на /books вы должны получить ответ похожий на:
```$ curl -i localhost:3000/books
HTTP/1.1 200 OK
Content-Length: 205
Content-Type: text/plain; charset=utf-8

978-1503261969, Emma, Jayne Austen, £9.44
978-1505255607, The Time Machine, H. G. Wells, £5.99
978-1503379640, The Prince, Niccolò Machiavelli, £6.99```
