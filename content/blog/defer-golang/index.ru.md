+++
title = "defer в Go: правила и ловушки"
date = 2026-06-30
draft = false
slug = "defer-golang"
description = "defer в Go: три правила и LIFO, перехват паники через recover и типичные ловушки, defer в цикле, value receiver, потерянная ошибка Close и os.Exit."
summary = "Оператор defer прост на поверхности, но скрывает нюансы. Разбираем три правила, работу с panic и recover и типичные ловушки."
tags = ["Go"]
+++

Когда я начинал изучать Go, одна конструкция показалась мне необычной.
Это оператор `defer`. Несмотря на кажущуюся простоту, он скрывает достаточно нюансов, чтобы удивить даже опытного
разработчика.

## Что такое defer и зачем он нужен

`defer` позволяет отложить вызов функции до выхода из текущей функции. Неважно, как мы выходим: через `return`,
по завершении тела или при панике. Отложенная функция всё равно будет вызвана. Удобно для очистки ресурсов.

Где пригодится `defer`:

- Очистка ресурсов (закрытие файлов, соединений с БД, HTTP response body)
- Синхронизация (разблокировка мьютексов, ожидание WaitGroup)
- Обработка ошибок и паник
- Принудительная запись данных из буфера
- Измерение времени выполнения функций для метрик

## Три правила defer

1. Аргументы отложенной функции вычисляются в момент выполнения оператора `defer`, а не при фактическом вызове.

```go
func main() {
    value := "old"
    defer fmt.Println(value)
    value = "new"
}

// Output: old
```

Если нужно, чтобы `defer` увидел актуальное значение, используем замыкание. Оно захватывает переменную по ссылке:

```go
func main() {
    value := "old"
    defer func () {
        fmt.Println(value)
    }()
    value = "new"
}

// Output: new
```

2. Отложенные функции вызываются в порядке LIFO (Last In First Out).

```go
func main() {
    for i := range 10 {
        defer fmt.Print(i)
    }
}

// Output: 9876543210
```

3. Отложенные функции могут читать и изменять именованные возвращаемые значения функции.

```go
func main() {
    value := getValue()
    fmt.Println(value)
}

func getValue() (value string) {
    defer func () {
        value = "new" // 2. value = "new"
    }()

    return "old" // 1. value = "old"
}
// 3. return "new"

// Output: new
```

Как это работает: `return "old"` сначала присваивает `value = "old"`, затем выполняются defer-функции (которые могут изменить `value`), и только потом функция реально возвращает значение.

## defer, panic и recover

Перехватить панику в Go можно только через `defer`. Когда функция вызывает `panic`, нормальное выполнение останавливается. Go начинает раскрутку стека: поднимается по вызовам вверх и на каждом уровне выполняет все defer-функции в LIFO-порядке. Если ни одна из них не вызвала `recover`, программа завершается с выводом стектрейса.

```go
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recovered:", r)
        }
    }()

    panic("something went wrong")
}

// Output: recovered: something went wrong
```

Важная деталь: `recover` работает только при прямом вызове из defer-функции. Кажется логичным вынести обработку в хелпер, но это не сработает:

```go
func myRecover() {
    if r := recover(); r != nil {
        fmt.Println("recovered:", r)
    }
}

func main() {
    defer myRecover() // panic не будет перехвачен
    panic("crash")
}

// Output:
// panic: crash
//
// goroutine 1 [running]:
// ...
```

Ещё одна тонкость: паника распространяется только по стеку текущей горутины. Если горутина паникует без `recover`, падает вся программа, а не только она.

```go
func main() {
    go func() {
        panic("crash") // убьёт всю программу
    }()

    time.Sleep(time.Second)
}

// Output: panic: crash
```

## Ловушки и подводные камни

### Игнорирование ошибки

```go
func writeData(filename string, data []byte) error {
    f, err := os.Create(filename)
    if err != nil {
        return err
    }
    defer f.Close()

    _, err = f.Write(data)
    return err
}
```

Выглядит правильно, но есть подвох. Если `f.Close()` вернёт ошибку, она потеряется. Чтобы её не потерять, делаем возвращаемую ошибку именованной и оборачиваем в `defer`. Благодаря третьему правилу defer-функция может прочитать `err` и даже изменить её.

```go
func writeData(filename string, data []byte) (err error) {
    f, err := os.Create(filename)
    if err != nil {
        return err
    }
    defer func() {
        err = errors.Join(err, f.Close())
    }()

    _, err = f.Write(data)
    return err
}
```

Альтернатива: использовать `f.Sync()` перед `return` для записи, а `defer f.Close()` оставить только для закрытия дескриптора.

```go
func writeData(filename string, data []byte) error {
    f, err := os.Create(filename)
    if err != nil {
        return err
    }
    defer f.Close()

    _, err = f.Write(data)
    if err != nil {
        return err
    }

    return f.Sync()
}
```

`errors.Join` ловит все ошибки, но требует именованных возвращаемых значений. `f.Sync()` проще: не нужны ни замыкания,
ни именованные значения. Разделение обязанностей: `Sync()` отвечает за данные, `Close()` за дескриптор.

### defer внутри цикла

`defer` работает на уровне функции, не блока. В цикле это создаёт проблему:

```go
func concatFiles(filenames ...string) (string, error) {
    var result string
    for _, filename := range filenames {
        f, err := os.Open(filename)
        if err != nil {
            return "", err
        }
        defer f.Close() // все файлы закроются только при выходе из функции!

        data, err := io.ReadAll(f)
        if err != nil {
            return "", err
        }
        result += string(data)
    }
    return result, nil
}
```

`defer` откладывает вызов `f.Close()` до выхода из функции `concatFiles`. Значит, открытые дескрипторы накапливаются, и
все файлы закроются только в самом конце. Передадим слайс из 1000 `filenames`, и одновременно откроется 1000
дескрипторов. А ОС ограничивает их количество на процесс, обычно 1024 по умолчанию. На тысячу файлов может просто не хватить.

Как это исправить:
- Вынести чтение файла в отдельную функцию
- Не использовать `defer`, вызывать `f.Close()` явно
- Использовать анонимную функцию с немедленным вызовом (IIFE)

Вот вариант с отдельной функцией:

```go
func concatFiles(filenames ...string) (string, error) {
    var result string
    for _, filename := range filenames {
        data, err := readFile(filename)
        if err != nil {
            return "", err
        }
        result += string(data)
    }
    return result, nil
}

func readFile(filename string) ([]byte, error) {
    f, err := os.Open(filename)
    if err != nil {
        return nil, err
    }
    defer f.Close()

    return io.ReadAll(f)
}
```

### Value receiver в defer

Первое правило говорит: аргументы вычисляются в момент `defer`. Но receiver метода тоже аргумент, хотя по виду и не скажешь. Если метод принимает value receiver, struct копируется при регистрации `defer`:

```go
type Data struct{ a int }
func (d Data) print() { fmt.Println(d.a) }

func main() {
    d := Data{a: 10}
    defer d.print() // d копируется, выведет 10
    d.a = 20
}

// Output: 10
```

Исправляем pointer receiver'ом:

```go
func (d *Data) print() { fmt.Println(d.a) }

// Output: 20
```

### os.Exit()

Отложенная функция не вызывается, если программа завершается с помощью `os.Exit()`. Это может ударить по graceful
shutdown. Если в `main()` настроены defer-функции для закрытия соединений или записи данных из буфера, а завершение идёт
через `os.Exit()` (или `log.Fatal`, который вызывает `os.Exit(1)` под капотом), то ничего из этого не выполнится.

```go
func main() {
    defer fmt.Println("deferred func")
    os.Exit(0)
}

// Ничего не выводит
```

## Заключение

`defer` из тех механизмов Go, что просты на поверхности, но требуют понимания нюансов. Запомните три вещи: аргументы вычисляются при регистрации, а не при вызове; в циклах `defer` копит ресурсы до выхода из функции; ошибку `Close()` нельзя игнорировать при записи. Тогда `defer` станет надёжным инструментом, а не источником сюрпризов.
