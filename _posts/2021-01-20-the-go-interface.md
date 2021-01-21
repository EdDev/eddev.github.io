---
title: "The Go Interface"
date: 2020-10-19
categories:
  - blog
tags:
  - golang
  - cleancode
---

![Michelangelo](/assets/images/creation_of_adam.jpg)

The word "interface" originates from the merge of two:
*inter* (between) and *face* (shape, figure).

Here is a short definition: *An interface is where two entities meet*.

In the software & programming context, an interface defines the connection
between two software parts.

Multiple programming languages introduce interfaces as part of their syntax,
while others (e.g. C) are leaving them as concepts for the developer to
implement.

As a concept, interfaces serve a common purpose. Each language having special
syntax and different ways to use them.

## What is an Interface?

Software is a collection of components that implement logic through code.
Interfaces connect components in an abstract, behavior driven approach.

An interface expresses a required behavior and through it defines a contract
between software components. In this relation, a user uses the interface
to request an operation from the interface implementer (a concrete entity).
All that without any of them knowing about (depending on) each other.

It is worth noting that an interface does not need to define the full
behavior of the concrete implementation. That is, it can describe only
a small subset of the concrete behavior.

A concrete object may implement a large number of functionalities. Different
concrete objects may have little in common, but it is enough for a single
behavior to be in common to place such two objects under the same interface.

> **_EXAMPLE:_** A human and a dog have the **runner** behavior in common,
so we can ask them both to *run()* without knowing their concrete identity.

### Breaking Dependency

Interfaces are really good at breaking long dependency flows. They do so by
inverting the dependency while keeping the call flow as is.

If we have this chain of dependencies:
> [main]-->[restaurant]-->[menu]-->[food]

Then changes in food would create a chain of reaction that recompiles and
changes the whole chain, up to "main".
Such a dependency chain is coupling entities, rejects re-use and does not
support change without heavy implications and risks.

By placing interfaces between each component, a protocol between each entity
is created which in turn contributes to decoupling and a modular software
structure.

> [main]-->i<--[restaurant]-->i<--[menu]-->i<--[food]

Such a structure now allows each component to be interchangeable as long as
it confirms with the interface. This also implies that testing each component
is now much easier, as a simple replacement can be placed by the test to
mimic the interface expected behavior.

## Interfaces in Golang

In Go, interfaces are types that are defined by a set of method signatures.

Unlike other languages, an interface is implicitly bind to an object
that implements the methods.
A type error will occur if the concrete type does not implement the methods
of the interface and an assignment occurs.

Here is a simple interface:
```golang
type Runner interface {
    Run(distance int) error
}
```

Any concrete implementation that implements the `Run` function signature
can be placed "under" the `Runner` interface.

The concrete will look like this:
```golang
type Dog struct {}

func (d Dog) Run(distance int) error {
    return nil
}
```

And the usage will look like this:
```golang
func main() {
    var runners []Runner

    for i := 0; i < 10; i++ {
        runners = append(runners, Dog{})
    }
    startRace(runners, 42195)
}

func startRace(runners []Runner, distance int) error {
    for _, r := range runners {
        if err := r.Run(distance); err != nil {
            return err
        }
    }
    return nil
}
```

## Usage
To start our journey, lets invent a story to implement.

We have been asked to save application data to storage.
There are several storage types (remote, disk, volatile-memory) which should
be supported.

### The first attempt

```golang
package app

import (
    "fmt"

    diskstore   "storage/disk"
    memstore    "storage/memory"
    remotestore "storage/remote"
)

func Save(data string, storeType string) error {
   encodedData := encodeData(data)
   switch storeType {
   case diskstore.TypeName: return diskstore.Save(encodedData)
   case remotestore.TypeName: return remotestore.Store(encodedData)
   case memstore.TypeName: return memstore.Write(encodedData)
   }

   return fmt.Errorf("unsupported store type: %s", storeType)
}
```

This seems simple, right?

Simple, yes. But it has several problems:
- The app, which contains the business logic depends on storage implementation
  details.
  We prefer internal changes in those "drivers" to not require a rebuild of
  the app. We prefer a more independent app.
- Unit testing the new `Save` function in the app package is impossible
  without knowing what to mock in each storage type.
  The code logic that we look to check are in the app package, we do not need
  (or interested) to check its "drivers".
- This solution makes it harder to add a new storage type. The app needs to
  change (import new type, add it to the switch) and obviously the tests
  need to be adjusted to cover the new addition.

### Interfaces to the rescue

Isolating and decoupling the app and the storages can be accomplished by
breaking the dependency flow.
A well known mechanism to do so is *dependency injection*.

And here the interfaces shine.

A well defined interface can create a contract between the app and the storage,
such that it will fit all types. On one hand the app depends on the common
interface and on the other, the storage types implement them.
No party knows about the other, they just know about the interface.

First, lets define an interface that all storage types can implement.

We need to identify the behavior needed here:
- open:   To prepare the storage resources (e.g. open files, allocate memory).
- writer: To write the data.

We intentionally do not define any other behavior the storage types support.
The other behaviors are irrelevant to the operation of storing the data and
as a by-product also declaratively limits what the app controls.

```golang
type OpenWriter interface {
    Open() (closer, error)
    Write(p []byte) (n int, err error)
}
```

Each storage type needs to implement the methods that our interface defined.
That implies that unlike the previous implementation, each storage should
have a struct with methods bind to it.

On the app side, we just need to replace the storeType type from a string to
our newly interface and then just call the methods to prepare the storage and
write on it.

```golang
func Save(data string, store OpenWriter) error {
   close, err := store.Open()
   if err != nil {return err}
   defer close()

   if _, err := store.Write(encodeData(data)); err != nil {
       return err
   }
}
```

### What about tests?

Testable code *requires* decoupling and isolation. The test is a user of the
tested code and needs to control its dependent data.

When the tested code depends on interfaces, the test can substitute its own
concrete objects and control the flow.

Our current `app.Save` function is already testable.
Lets examine the flows we need to check:
- Open() fails.
- Write() fails.
- Data saved successfully.

To do so, we just need to build a concrete that implements the interface
and at the same time allows the test to control the responses.

Lets implement our test stub:

```golang
type testStore struct {
    openFail  bool
    writeFail bool
}

func (s testStore) Open() (closer, error) {
    var close closer = func(){}
    if s.openFail {
        return close, fmt.Errorf("failed")
    }
    return close, nil
}

func (s testStore) Write(data []byte) (int, error) {
    if s.writeFail {
        return 0, fmt.Errorf("failed")
    }
    return nil
}
```

And here are our tests (using ginkgo/gomega):

```golang
var _ = Describe("app Save", func() {
    It("fail to open store", func() {
        store := testStore{openFail: true}
        Expect(app.Save([]byte{}, store)).NotTo(Succeed())
    })

    It("fail write to store", func() {
        store := testStore{writeFail: true}
        Expect(app.Save([]byte{}, store)).NotTo(Succeed())
    })

    It("succeed write to store", func() {
        store := testStore{}
        Expect(app.Save([]byte{}, store)).To(Succeed())
    })
})
```

## Please avoid these

### Return an interface
Please return a concrete type from a factory or a constructor, not an interface.

Returning an interface hints that the concrete type exposes only a specific
behavior which is used by its consumers.
But this limits the concrete type to expand and support other behaviors (it is
limited by the interface).

It may also be a premature abstraction.

Given:
```golang
type Animal interface {
    Walk(distance int) error
}

type Cat struct {
    name string
}
func (c Cat) Walk(distance int) error {
    return nil
}
```

Prefer:
```golang
func NewCat(name string) Cat {
    return Cat{name: name}
}
```

And avoid:
```golang
func NewCat(name string) Animal {
    return Cat{name: name}
}
```

But there are always exceptions. A factory that may return different concrete
types must have an interface as the return value.

This is fine:
```golang
func NewAnimal(name, species string) Animal {
    switch species {
    case "Dog": return Dog{name: name}
    case "Cat": return Cat{name: name}
    }
    return nil
}
```

### Define interfaces side by side with concrete objects
Placing the interface definition in the same package as the concrete type
hints that it is "owned" by it. But as elaborated earlier, an interface
is looking to break the coupling.

Interfaces fit either at the location in which they are used or in a separate
package from which each user can fetch it without depending on other users or
the concrete implementations of it.

A good example is the [io](https://golang.org/pkg/io) package. It provides
basic interfaces to I/O concrete implementations like
[os](https://golang.org/pkg/os)

### Define interfaces to mock a complete type

Mocking in general is a costly effort.
It essentially checks too much inner details. Whenever possible, prefer
stubs or simple placeholders.

What is surely a problem is the mocking of an entire type.
It requires a definition of an interface that tracks all methods of the type
and which is going to change with every added method.

If mocking is indeed a must, define independent, small and focused interfaces
and mock those.

## Some extras

### Equality
An interface is identified by two values, the underlying type and its value.
Both need to be equal for an equality check between two different variables
of different interface types to exist.

Here is an example of the same value (`nil`) but a different underlying type:
```golang
type runner interface {
    run()
}

type dog struct{}
func (d dog) run() {
}

var r runner
var d dog
fmt.Println(r == d) //false
```

### The underlying type
When a variable type is an interface, only the interfaces methods are
accessible. The concrete type other methods or its data members are not
accessible.
In some cases however, it may be useful to access those.

Fortunately, it is possible to retrieve the underlying type if you know it:
```golang
dog := i.(Dog)
```

### Embedding other interfaces
Similar to structs, interfaces may be embedded into others.

Here is a simple example:
```golang
type ReaderWriter interface {
    io.Reader
    io.Writer
}
```

Which is equivalent to:
```golang
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}
```

### interface{}

All types implement the empty interface.

It allows a level of generics, but has a high penalty with the need to
always retrieve the underlying type.

