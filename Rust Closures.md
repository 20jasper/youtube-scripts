---
markdown: 
enableLinks: true
highlightTheme: css/vs2015.css
---

# Functional Language Features—Closures


---
## Closures

Closures are functions that capture from the lexical scope

```rust
fn pad_input(input: Option<&str>, padding: &str) -> String {
    input.map_or_else(
        || padding.to_string(),
        |input| format!("{padding} {input} {padding}"),
    )
}

pad_input(None, "🦀🦀"); // "🦀🦀"
pad_input(Some("crab time"), "🦀"); // "🦀 crab time 🦀"
``` 

^padInput

notes:
add crabs stringifies the padding if none, and pads the input with padding if some

---
## How are closures different than functions?

notes:

closures are meant to be short and specific to their contexts. They are typically used inline like in the example or stored in a variable

---
### Inferred Type

Closures arguments and returns can optionally be explicitly typed

```rust
fn pad_input(input: Option<&str>, padding: &str) -> String {
    input.map_or_else(
        || -> String { padding.to_string() },
        |input: &str| -> String {
            format!("{padding} {input} {padding}")
        },
    )
}
```

notes:
Closures aren't exported like functions, so an explicit type signature is unnecessary

---
### Capture values from environment

```rust
fn pad_input(input: Option<&str>, padding: &str) -> String {
    input.map_or_else(
        || padding.to_string(),
        |input| format!("{padding} {input} {padding}"),
    )
}
```

notes:
padding is available to us, and it wouldn't be available

---

### Capturing vs arguments

Capturing values is similar to passing arguments to a function

Values can be

- borrowed immutably
- borrowed mutably
- moved

---
#### Immutable borrow

```rust
let val = "hi".to_string();
let print_val = || println!("{val}");

print_val(); // "hi"
print_val(); // "hi"
```

---
#### Mutable borrow

```rust
let mut val = "hi".to_string();
let mut push_then_print = |x| {
		val.push(x);
		println!("{val}");
};

push_then_print('i'); // "hii"
push_then_print('i'); // "hiii"
```

---
#### Move

```rust
let val = "hi".to_string();
let push_then_print = move |x| {
		let val2 = val;
		println!("{val2}");
};

push_then_print('i');
push_then_print('i'); 
// error[E0382]: use of moved value: `push_then_print`
``` 

---

##### Move use case[^1]

```rust
let list = vec![1, 2, 3];
println!("Before defining closure: {:?}", list);

thread::spawn(move || println!("From thread: {:?}", list))
		.join()
		.unwrap();
```

<%? footnotes %>

[^1]: definitely not stolen from the book

notes:
We need to move here since the new thread could outlive the current function,
which means list would dropped and invalidated during execution

Compiler complains since the closure needs a lifetime of 'static since new thread could outlive the program

---

### `Fn` Traits

There are three `Fn` traits, which are automatically implemented depending on what happens to the values they capture

---

#### `FnOnce`

Can only be called once. All closures implement this trait

```rust
let v = vec![1, 2, 3];
let f = || v;
```

notes:

This is helpful when we only need to call a closure once like in `unwrap_or_else` from before.

V is moved into the closure, so it cannot be used again, and the closure cannot transfer ownership of v again

---

#### `FnMut`

Can be called many times, but potentially mutate values

```rust
let mut v = vec![1, 2, 3];
let f = || v.push(5);

mutation(f);

``` 

```rust
fn mutation<F>(mut f: F)
where
    F: FnMut(),
{
    f();
    f();
    f();
}
```

notes:
FnMut closures are also FnOnce closures, and can be used in their place

---

#### `Fn`[^1]

Can be called many times and don't mutate values captured

```rust
let f: impl Fn(i32) -> i32 = |x| 10 + x;

let val = f(10);
```

<%? footnotes %>
2. This is not a valid way to type a closure—it's just to show its inferred type

[^1]: Not to be confused with `fn`, which is a function pointer

notes: good for concurrency. Closures could be called many times in parallel

---

### Practice!

What kind of `Fn` trait is used for each of these standard library functions

---
#### `Option::map`

```rust
fn pad_input(
    input: Option<&str>,
    padding: &str,
) -> Option<String> {
    input
        .map(|input| format!("{padding} {input} {padding}"))
}
```

notes:
takes in a closure and returns a new option

---

`FnOnce`

```rust
    pub fn map<U, F>(self, f: F) -> Option<U>
    where
        F: FnOnce(T) -> U,
    {
        match self {
            Some(x) => Some(f(x)),
            None => None,
        }
    }
```

notes:
FnOnce since map takes ownership of `self` and doesn't return the option, so it can't be called again

---

#### Iterator::map

Called once for every element in `a`

```rust
a.iter().map(|x| 2 * x);
```

---

`FnMut`

```rust
    fn map<B, F>(self, f: F) -> Map<Self, F>
    where
        Self: Sized,
        F: FnMut(Self::Item) -> B,
    {
        Map::new(self, f)
    }
```

notes:
Needs called multiple times, and there is no need to restrict it to capture immutably

---

### Captured lifetimes

`make_padder` captures `x` from the environment

```rust
// error[E0700]: hidden type for `impl Fn(usize) -> String` 
// captures lifetime that does not appear in bounds
fn make_padder(x: &str) -> impl Fn(usize) -> String {
    move |repeat| {
        let padding = "🦀".repeat(repeat);
        format!("{padding}{x}{padding}")
    }
}
```

note:
make_padder pads a string with a variable amount of crab padding
needs x to be valid since it uses it to create padding every time

---

The closure can't live longer than `x`

```rust
fn make_padder<'a>(
    x: &'a str,
) -> impl Fn(usize) -> String + 'a {
    move |repeat| {
        let padding = "🦀".repeat(repeat);
        format!("{padding}{x}{padding}")
    }
}
```

---

The lifetime can be elided with `'_`

```rust
fn make_padder(x: &str) -> impl Fn(usize) -> String + '_ {
    move |repeat| {
        let padding = "🦀".repeat(repeat);
        format!("{padding}{x}{padding}")
    }
}
```

---

### Coding practice!

Let's make `Option::map`!
