+++
title = "defer in Go: rules and pitfalls"
date = 2026-06-30
draft = false
slug = "defer-golang"
description = "How defer works in Go: the three rules and LIFO, recovering from panics, and the common pitfalls, defer in a loop, value receivers, the lost Close error, and os.Exit."
summary = "defer looks simple on the surface but hides subtleties. We walk through the three rules, panic and recover, and the common pitfalls."
tags = ["Go"]
+++

When I was first learning Go, one construct struck me as unusual: the `defer` statement.
Despite its apparent simplicity, it hides enough subtleties to surprise even an experienced
developer.

## What defer is and why you need it

`defer` postpones a function call until the surrounding function returns. It doesn't matter how we
leave it: via `return`, by reaching the end of the body, or through a panic. The deferred function
runs regardless. Handy for cleaning up resources.

Where `defer` is useful:

- Releasing resources (closing files, database connections, HTTP response bodies)
- Synchronization (unlocking mutexes, waiting on a WaitGroup)
- Handling errors and panics
- Flushing buffered data
- Measuring function execution time for metrics

## The three rules of defer

1. The arguments of a deferred call are evaluated when the `defer` statement runs, not when the call actually happens.

```go
func main() {
    value := "old"
    defer fmt.Println(value)
    value = "new"
}

// Output: old
```

If you want `defer` to see the current value, use a closure. It captures the variable by reference:

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

2. Deferred functions run in LIFO order (last in, first out).

```go
func main() {
    for i := range 10 {
        defer fmt.Print(i)
    }
}

// Output: 9876543210
```

3. Deferred functions can read and modify the function's named return values.

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

How it works: `return "old"` first assigns `value = "old"`, then the deferred functions run (and may change `value`), and only then does the function actually return.

## defer, panic, and recover

In Go you can only recover from a panic through `defer`. When a function calls `panic`, normal execution stops. Go begins unwinding the stack: it walks up the call chain and, at each level, runs all deferred functions in LIFO order. If none of them calls `recover`, the program terminates and prints a stack trace.

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

An important detail: `recover` only works when called directly from a deferred function. Moving the handling into a helper seems reasonable, but it won't work:

```go
func myRecover() {
    if r := recover(); r != nil {
        fmt.Println("recovered:", r)
    }
}

func main() {
    defer myRecover() // panic will NOT be recovered
    panic("crash")
}

// Output:
// panic: crash
//
// goroutine 1 [running]:
// ...
```

Another subtlety: a panic propagates only up the current goroutine's stack. If a goroutine panics without recovering, the whole program crashes, not just that goroutine.

```go
func main() {
    go func() {
        panic("crash") // kills the entire program
    }()

    time.Sleep(time.Second)
}

// Output: panic: crash
```

## Pitfalls and gotchas

### Ignoring the error

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

Looks correct, but there's a catch. If `f.Close()` returns an error, it's lost. To avoid losing it, make the return error named and wrap the close in a `defer`. Thanks to the third rule, the deferred function can read `err` and even change it.

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

An alternative: call `f.Sync()` before returning to flush the data, and leave the deferred `f.Close()` to just close the descriptor.

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

`errors.Join` catches every error but requires named return values. `f.Sync()` is simpler: no closures
and no named values. It also separates concerns: `Sync()` is responsible for the data, `Close()` for the descriptor.

### defer inside a loop

`defer` works at the function level, not the block level. Inside a loop, that creates a problem:

```go
func concatFiles(filenames ...string) (string, error) {
    var result string
    for _, filename := range filenames {
        f, err := os.Open(filename)
        if err != nil {
            return "", err
        }
        defer f.Close() // all files close only when the function returns!

        data, err := io.ReadAll(f)
        if err != nil {
            return "", err
        }
        result += string(data)
    }
    return result, nil
}
```

`defer` postpones each `f.Close()` until `concatFiles` returns. That means open descriptors pile up, and
every file is closed only at the very end. Pass a slice of 1000 `filenames` and you'll have 1000
descriptors open at once. The OS caps the number per process, usually 1024 by default. For a thousand files there may simply not be enough.

How to fix it:
- Extract the file reading into a separate function
- Skip `defer` and call `f.Close()` explicitly
- Use an immediately invoked anonymous function (IIFE)

Here's the version with a separate function:

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

### Value receiver in defer

The first rule says arguments are evaluated at the moment of `defer`. But a method's receiver is an argument too, even though it doesn't look like one. If the method takes a value receiver, the struct is copied when the `defer` is registered:

```go
type Data struct{ a int }
func (d Data) print() { fmt.Println(d.a) }

func main() {
    d := Data{a: 10}
    defer d.print() // d is copied, prints 10
    d.a = 20
}

// Output: 10
```

Fix it with a pointer receiver:

```go
func (d *Data) print() { fmt.Println(d.a) }

// Output: 20
```

### os.Exit()

A deferred function does not run if the program exits via `os.Exit()`. This can hurt graceful
shutdown. If `main()` has deferred functions that close connections or flush buffers, but you exit
through `os.Exit()` (or `log.Fatal`, which calls `os.Exit(1)` under the hood), none of them will run.

```go
func main() {
    defer fmt.Println("deferred func")
    os.Exit(0)
}

// Prints nothing
```

## Conclusion

`defer` is one of those Go mechanisms that are simple on the surface but require understanding the nuances. Remember three things: arguments are evaluated at registration, not at the call; inside loops `defer` accumulates resources until the function returns; and you can't ignore the `Close()` error when writing. Then `defer` becomes a reliable tool rather than a source of surprises.
