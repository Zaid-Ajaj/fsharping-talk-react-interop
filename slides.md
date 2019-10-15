### Bindings For React Components in Fable

---

### Table of Contents

* About Me
* About Fable
* React in Fable
* Interop in Fable
* Interop with React components
* Debugging Interop Code
* Available Implementations
* React DSL vs. Feliz
* Proper Packaging

---

### About Me

- Github  @Zaid-Ajaj
- Twitter @Zaid-Ajaj

---

### About Fable

- F#-to-Javascript compiler
- Highly interoperable
- Plays well with Javascript and Friends

---

### React in Fable

- Used in Elmish
- Main rendering engine
- Standalone Usage
- Proper React

---

### Interop in Fable

- Basics with `Emit`: values and macros
- `import`, `importDefault`
- Object literals
- using `createObj [ ]`
- using `keyValueList` with discriminated unions
- Anonymous Records
- Creating object literals by hand

----

### Basics with `[<Emit>]`: Values

```fsharp
open Fable.Core
open Fable.Core.JsInterop

[<Emit("1")>]
let one : int = jsNative
```

----

### `[<Emit>]`: Macros

```fsharp
open Fable.Core
open Fable.Core.JsInterop

[<Emit("1")>]
let one : int = jsNative

[<Emit("console.log($0)")>]
let log(x: 'a) : unit = jsNative

log (one + one)
```
Compiles to
```js
console.log(1 + 1)
```

----

### Creating object literals: `createObj`

```fsharp
[<Emit("1")>]
let one : int = jsNative

[<Emit("console.log($0)")>]
let log(x: 'a) : unit = jsNative

let options = createObj [
    "one" ==> one
    "two" ==> "whatever"
    "three" ==> fun (x: int) -> x + 1
]

log options
```

----

Compiles to
```js
export const options = {
  one: 1,
  two: "whatever",
  three(x) {
    return x + 1;
  }
};

console.log(options);
```

----

### Using `keyValueList`

```fsharp
open Fable.Core
open Fable.Core.JsInterop

type Person =
    | Name of string
    | Age of int

let objectLiteral (xs: Person list) =
    keyValueList CaseRules.None xs

let person = objectLiteral [ Name "John"; Age 20 ]
log person /// { Name: "John", Age: 20 }
```