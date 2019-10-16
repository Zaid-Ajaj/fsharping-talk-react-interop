### Bindings For React Components in Fable

---

### Table of Contents

* About Me
* About Fable
* React in Fable
* Interop in Fable
* Interop with React components
* Debugging Interop Code
* Available Implementations: React DSL vs. Feliz
* Proper Packaging: Femto

---

### About Me

- Github  @Zaid-Ajaj
- Twitter @Zaid-Ajaj

---

### About Fable

- F#-to-Javascript compiler
- Highly interoperable
- Preserves semantics of F#

---

### React in Fable

- Used in Elmish
- Main rendering engine
- Can be used standalone

---

### React DSL in Fable

```fsharp
div [ Id "main" ] [
    h1 [ ClassName "shiny" ] [
        span [ ] [ str "Hello Fable" ]
    ]
]
```

---

### Interop in Fable

- Basics with `Emit`: values and macros
- Imports with `import` and `importDefault`
- Object literals
- using `createObj [ ]`
- using `keyValueList` with discriminated unions

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

### Interop with Fable: importing code

Given the following project structure
```
 src
  |
  | -- Code.js
  | -- App.fs
  | -- App.fsproj
```
Where `Code.js` has the following exports:
```js
module.exports = {
    value: 20
}
```

----

### Import

You can import a specific value from an "exported" javascript module
```fsharp
// App.fs
open Fable.Core
open Fable.Core.JsInterop

let value : int = import "value" "./Code.js"

printfn "%d" value // 20
```

----

### Import the entire module

```fsharp
// App.fs
open Fable.Core
open Fable.Core.JsInterop

interface Code =
    abstract value : int

let code = importDefault "./Code.js"

printfn "%d" code.value // 20
```

----

### Import the entire module untyped

```fsharp
// App.fs
open Fable.Core
open Fable.Core.JsInterop

let code : obj = importDefault "./Code.js"

printfn "%d" (!!code?value) // 20
```

----

### Importing a npm package

```fsharp
// App.fs
open Fable.Core
open Fable.Core.JsInterop

let package : obj = importDefault "package-name"

let packageValue : string = import "value" "package-name"

printfn "%s" packageValue
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

---

### Interop with React components

So you can import stuff, but what the hell is this?

```jsx
<LineChart width={500} height={300} data={data}>
  <XAxis dataKey="name"/>
  <YAxis/>
  <CartesianGrid stroke="#eee" strokeDasharray="5 5"/>
  <Line type="monotone" dataKey="uv" stroke="#8884d8" />
  <Line type="monotone" dataKey="pv" stroke="#82ca9d" />
</LineChart>
```

----

### Desugaring JSX Syntax

The following JSX snippet
```jsx
const App = () => {
    return (
        <div className="shiny" style={{ width: "200px" }}>
            Hello Fable
        </div>
    );
}

ReactDOM.render(<App />, document.getElementById("root"))
```

----

is actually
```js
import { createElement } from 'react'
import { render } from 'react-dom'

const App = () => {
    const props = {
        className: "shiny",
        style: {  width: "200px" }
    }
    return createElement('div', props, "Hello Fable");
}

render(createElement(App, {}, null),
    document.getElementById("root"))
```

----

### Nested elements

```js
<h1>
    <span id="title">Title</span>
</h1>
```
compiles to
```js
createElement('h1', { },
    createElement('span', { id: "title" }, "title"))
```

----

### Imported JSX

```js
import { createElement } from 'react'
import { LineChart, XAxis, Line } from 'recharts'

<LineChart data={data}>
  <XAxis dataKey="x" />
  <Line dataKey="y" />
</LineChart>
```
compiles to
```js
import { createElement } from 'react'
import { LineChart, XAxis, Line } from 'recharts'

createElement(LineChart, { data: data },
    createElement(XAxis, { dataKey: "x" }, null),
    createElement(Line,  { dataKey: "y" }, null))
```

----

### Putting it together: `ofImport`

`ofImport` calls `createElement` internally

```fsharp
let inline lineChart
    (props: IProp list)
    (children: React.ReactElement list): React.ReactElement =
    ofImport
      "LineChart"
      "recharts"
      (keyValueList CaseRules.LowerFirst props)
      children
```

---

### Debugging Generated Code

- Use fable-splitter
- `fable-splitter demo -o ./dist`
- The `--commonjs` flag


---

### A BETTER APPROACH

A look into the [fable-recharts](https://github.com/fable-compiler/fable-recharts) implementation

Discriminated unions for properties won't cut it

----

### Problems with `keyValueList`

```fsharp
type Style =
    | Width of int
    | Height of int

let style = objectLiteral [ Width 200 ]

log style // { Width: 200 }

// How to get { Width: "200px" } instead?
```

----

### "Hand-written" with static overloads

```fsharp
type IStyleAttribute = interface end

module internal Interop =
    let styleAttribute (key: string) (value: obj) =
        unbox<IStyleAttribute> (key, value)

type Style =
    static member inline width(value: int) =
        Interop.styleAttribute "width" (unbox<string> value + "px")
    static member inline width(value: float) =
        Interop.styleAttribute "width" (unbox<string> value + "px")
    static member inline width(value: string) =
        Interop.styleAttribute "width" value
    // etc.
```

----

### Consuming API

```fsharp
let inline createStyle (attributes: IStyleAttribute list)  =
    createObj (unbox attributes)

[
    Style.width 150
    Style.height "200px"
]
|> createStyle
|> log /// { width: "150px", height: "200px" }
```

----

### Enums made simple

```fsharp
type Style =
    static member inline blablahblah () = ...

module Style =
    type textAlign =
        static member inline center = Interop.mkStyle "textAlign" "center"
        // etc.
```

----

### Consuming API

```fsharp
[
    Style.width 200
    Style.height "150px"
    Style.textAlign.center
]
```

---

### Available Implementations

- Default React DSL - `Fable.React`
- [Feliz](https://zaid-ajaj.github.io/Feliz/#/)

---

### Proper packaging

Introducing `Femto`

- Manages npm dependencies of Fable packages
- Optional package manager
- Abstracts away nuget/paket/npm/yarn

----

### Npm dependency metadata

Feliz

```xml
<NpmDependencies>
    <NpmPackage Name="react" Version="&gt;= 16.8.0" />
    <NpmPackage Name="react-dom" Version="&gt;= 16.8.0" />
</NpmDependencies>
```

Feliz.Recharts

```xml
<NpmDependencies>
    <NpmPackage Name="recharts" Version="&gt;= 1.7.1 &lt; 2.0.0" />
</NpmDependencies>
```

----

### Femto In Action

starting from almost nothing: [fable-getting-started](https://github.com/Zaid-Ajaj/fable-getting-started)