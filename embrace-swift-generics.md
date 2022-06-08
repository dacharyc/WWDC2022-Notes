# Embrace Swift generics

Presenters:
- Holly Borla, Swift Compiler Team

Abstractions separate ideas from specific details. For example, when you factor code out into a function or variable for reuse.

Swift lets you abstract away concrete types, when the set of types are all the same idea with different details.

- Model code with concrete types
- Identify common capabilities
- Build an interface
- Write generic code

## Model code with concrete types

```swift
struct Cow: Animal {
  func eat(_ food: Hay) {}
}

struct Hay: AnimalFeed {
  static func grow() -> Alfalfa {
    Alfalfa()
  }
}

struct Alfalfa: Crop {
  func harvest() -> Hay {
    Hay()
  }
}

struct Farm {
  func feed(_ animal: Cow) {
    let alfalfa = Hay.grow()
    let hay = alfalfa.harvest()
    animal.eat(hay)
  }
}
```

Can add more structs to represent other animals, like Cow, Horse, and Chicken. Want to be able to feed cows, horses, and chickens on the farm

```swift
farm.feed(Cow())
farm.feed(Horse())
farm.feed(Chicken())
```

Could overload the feed method:

```swift
struct Farm {
    func feed(_ animal: Cow) { ... }

    func feed(_ animal: Horse) { ... }

    func feed(_ animal: Chicken) { ... }
}
```

But each overload would have a similar implementation. This would add extra boilerplate and be largely repeated code:

```swift
struct Farm {
  func feed(_ animal: Cow) {
    let alfalfa = Hay.grow()
    let hay = alfalfa.harvest()
    animal.eat(hay)
  }

  func feed(_ animal: Horse) {
    let root = Carrot.grow()
    let carrot = root.harvest()
    animal.eat(carrot)
  }

  func feed(_ animal: Chicken) {
    let wheat = Grain.grow()
    let grain = wheat.harvest()
    animal.eat(grain)
  }
}
```

## Identify the common capabilities

We've got a collection of animal types that all have the ability to eat some type of food. Each implementation of the eat method will have some variability in behavior.

```swift
struct Cow {
  func eat(_ food: Hay) {
  // Eat the hay
  }
}

struct Horse {
  func eat(_ food: Carrot) {
  // Munch on the carrot
  }
}

struct Chicken {
  func eat(_ food: Grain) {
  // Peck at the grain
  }
}
```

Polymorphism allows one piece of code to have many behaviors.

- Function overloading: where the same function call can mean different things depending on the argument type. Overloading is called `ad-hoc polymorphism`.
- Code operating on a supertype can have different behavior based on the specific subtype the code is using at runtime. Subtypes achieve `subtype polymorphism`.
- Generic code uses type parameters to allow writing one piece of code that works with different types. Concrete types themselves are used as arguments. Generics achieve `parametric polymorphism`.

### Subtype Polymorphism

We could introduce an `Animal` class. Change animal structs to subclasses. Each specific animal inherits from the Animal superclass, and overrides the eat method.

```swift
class Animal {
    func eat(_ food: ???) { fatalError("Subclass must implement `eat`") }
}

class Cow: Animal {
  override func eat(_ food: Hay) {
  // Eat the hay
  }
}

class Horse: Animal {
  override func eat(_ food: Carrot) {
  // Munch on the carrot
  }
}

class Chicken: Animal {
  override func eat(_ food: Grain) {
  // Peck at the grain
  }
}
```

We haven't filled in a parameter type for eat method. And using classes forced us into reference semantics. Requires subclasses to override methods in the base class, but forgetting to do this wouldn't be caught until runtime. Each animal subtype eats a different type of food, and this is difficult to use with a class hierarchy.

Represent animal feed in a type-safe way by introducing a type parameter on the Animal superclass:

```swift
class Animal<Food> {
    func eat(_ food: ???) { fatalError("Subclass must implement `eat`") }
}

class Cow: Animal<Hay> {
  override func eat(_ food: Hay) {
  // Eat the hay
  }
}

class Horse: Animal<Carrot> {
  override func eat(_ food: Carrot) {
  // Munch on the carrot
  }
}

class Chicken: Animal<Grain> {
  override func eat(_ food: Grain) {
  // Peck at the grain
  }
}
```

This type parameter serves as a placeholder for the specific feed type for each subclass. Unfortunately, a lot of code that works with animals probably won't care about Food at all, and Food isn't the core purpose of the animal. Introduces boilerplace. Not good ergonomics or the right semantics.

Need a different language feature.

## Build an Interface (Protocol)

A protocol separates ideas about what code does from its implementation details.

Each animal has two common capabilities:
- A specific type of food
- An operation for consuming some of its food

```swift
protocol Animal {
    // Like a type parameter, an associated type serves as a placeholder for a concrete type.
    associatedType Feed: AnimalFeed
    func eat(_ food: Feed)
}
```

Now we can conform our animals to this new protocol:

```swift
struct Cow: Animal {
  func eat(_ food: Hay) {
  // Eat the hay
  }
}

struct Horse: Animal {
  func eat(_ food: Carrot) {
  // Munch on the carrot
  }
}

struct Chicken: Animal {
  func eat(_ food: Grain) {
  // Peck at the grain
  }
}
```

Compiler checks for conformance. We could specify the feed type explicitly using a typeAlias, but don't need to because it's a parameter for `eat`.

## Write generic code

```swift
protocol Animal {
    // Like a type parameter, an associated type serves as a placeholder for a concrete type.
    associatedType Feed: AnimalFeed
    func eat(_ food: Feed)
}

struct Farm {
    // Type parameter annotated with protocol conformance
    func feed<A: Animal>(_ animal: A) { ... }

    // Or we could use a where clause
    // This is a complex way of representing this
    func feed<A>(_ animal: A) where A: Animal
    // But we can also just do this identical declaration:
    func feed(_ animal: some Animal)
}
```

Some indicates there is a specific type you're working with. The "some" keyword is always followed by a conformance requirement - in this case, the Animal protocol.

An abstract type that represents a placeholder for a specific concrete type is called an opaque type. The specific concrete type that is substituted in is called the underlying type.

```swift
protocol Animal {
    // Like a type parameter, an associated type serves as a placeholder for a concrete type.
    associatedType Feed: AnimalFeed
    func eat(_ food: Feed)
}

struct Farm {
    func feed(_ animal: some Animal) {
        let crop = type(of: animal).feed.grow()
        let produce = crop.harvest()
        animal.eat(produce)
    }
}
```

What if we want to feed all the animals? We can't use `some` to create a type-erased array where the types might be mixed, because the underlying type might be cows _or_ chickens _or_ horses. Instead, we can use `any`:

```swift
struct Farm {
    func feed(_ animal: some Animal) {
        let crop = type(of: animal).feed.grow()
        let produce = crop.harvest()
        animal.eat(produce)
    }

    func feedAll(_ animals: [any Animal]) {
        for animal in animals {
            feed(animal)
        }
    }
}
```

Like `some`, the `any` keyword is always followed by a conformance requirement. 

Existential types, dynamic types, type-erasure - tyyyyyyyypes

Swift takes care of "unboxing" the value of an any type dynamically in Swift 5.7

With `some`, the underlying type is fixed:
- Holds a concrete type
- Guarantees type relationships

With `any`:
- Holds an arbitrary concrete type
- Erases type relationships

In general, write `some` by default, and only change it to `any` when you need type erasure. Similar to writing `let` constants by default until you know you need mutability.

## Related Sessions

- Design protocol interfaces in Swift
