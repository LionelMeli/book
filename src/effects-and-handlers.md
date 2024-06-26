# Effects and Handlers

> **Warning:** Effects handlers are a technology preview and subject to change.

> **Warning:** Effects handlers are an experimental feature. Do not use them in
> production.

## Getting Started with User-Defined Effects and Handlers

### Non-Resumable Effects: Exceptions

We can use effects and handlers to implement exceptions. For example:

```flix
eff Throw {
    pub def throw(): Void
}

def divide(x: Int32, y: Int32): Int32 \ Throw = 
    if (y == 0) {
        do Throw.throw()
    } else {
        x / y
    }

def main(): Unit \ IO = 
    try {
        println(divide(3, 2));
        println(divide(3, 0))
    } with Throw {
        def throw(_k) = println("Oops: Division by Zero!")
    }
```

Here we declare the effect `Throw` and use it inside the `divide` function. In
`main` we perform two divisions. The first succeeds and prints `1`. The second
fails and the error message is printed. The continuation `_k` is unused (and in
fact cannot be used because it requires an argument of type `Void`). The `main`
function has the `IO` effect since we use `println` in the handler, but it does
_not_ have the `Throw` effect since that has been handled.

> **Note:** `Void` is an empty (uninhabited) type built-in to Flix. The `Void`
> type, in combination with an effect operation, can be used everywhere a normal
> type is required. But notably a function, e.g. a continuation, which requires
> an argument of type `Void` cannot be invoked. 

### Resumable Effects

We can also implement resumable effects. For example:

```flix
eff Ask {
    pub def ask(): String
}

eff Say {
    pub def say(s: String): Unit
}

def greeting(): Unit \ {Ask, Say} = 
    let name = do Ask.ask();
    do Say.say("Hello Mr. ${name}")

def main(): Unit \ IO = 
    try {
        try greeting() with Ask {
            def ask(k) = k("Bond, James Bond")
        }
    } with Say {
        def say(s, k) = { println(s); k() }
    }
```

Here we declare two effects: `Ask` and `Say`. We use both effects in `greeting`.
In `main` we call `greeting` and handle each effect. We handle the `Ask` effect
by always resuming the continuation with the string `"Bond, James Bond"`. We
handle the `Say` effect by printing to the terminal, and then resuming the
continuation.

In this case, the order of handlers does not matter, but in the general case the
order may matter. 

## Limitations

The technology preview has some limitations. We are working on lifting these.

### Polymorphic Effects

The Flix effect system does not yet support polymorphic effects. For example, if we declare:

```flix
eff Throw[a] {
    pub def throw(x: a): Void
}
```

the Flix compiler reports:

```
❌ -- Syntax Error --

>> Unexpected effect type parameters.

1 | eff Throw[a] {
              ^
              unexpected effect type parameters
```

We plan to support polymorphic effects. 

### Spawn

The Flix effect system does not yet enforce that all effects are handled in spawn.

For example, the program below will compile, but crash at runtime:

```flix
eff Ask {
    pub def ask(): String
}

def main(): Unit \ IO = 
    region rc {
        spawn do Ask.ask() @ rc
    }
```

> **Warning:** Do not use effects and handlers inside spawn expressions.

### New Object Expressions

The Flix effect system does not yet enforce that all effects are handled in new object expressions.

For example, the program below will compile, but crash at runtime:

```flix
import java.lang.Runnable

eff Ask {
    pub def ask(): String
}

def newRunnable(): Runnable \ IO = new Runnable {
    def run(_this: Runnable): Unit \ IO = 
        do Ask.ask(); ()
}

def main(): Unit \ IO = 
    import java.lang.Runnable.run(): Unit \ IO;
    let r = newRunnable();
    run(r)

```

> **Warning:** Do not use effects and handlers inside new object expressions.
