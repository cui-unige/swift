---
layout: post
title:  A Case Against Class Inheritance
description: Please, stop using classes everywhere.
categories: object-oriented programming, protocol-oriented programming
---

In the first chapter of my [Swift tutorial]({{ site.baseurl }}tutorial/chapter-1),
I write quickly mention some common counterintuitive problems one faces with reference types (a.k.a. classes).
Swift is **not** an object-oriented programming language.
It *can* use classes (and sometimes they do have their places),
and supports them quite well actually.
But really, Swift is protocol-oriented.
Just look at its standard library if you're not convinced:
`Array`, isn't a class, it's a good old struct that implements clever protocols.
Yet, I see classes are everywhere!
Almost all examples of the Apple's language guide are given with classes,
and I can count with my fingers how many times I've seen a `struct` in a blog post.

The reason is probably that object-oriented programming has been hammered in the skull
of every student in computer science for years (and will continue to for years to come).
And at first, everything looks so perfect.
Classes represent the perfect level abstraction to handle complexity,
and subclassing is the best way to factor common behaviors, right?
But as soon as we start scratching the surface,
we realize that like the cake, object-oriented was a lie!

Others have talked about it much more eloquently that I could ever do,
but still I wanted to weigh on this as well.
In [Chapter 6]({{ site.baseurl }}tutorial/chapter-6) of my tutorial,
I talk about class inheritance, which I illustrate with a small class hierarchy.
I make a tour of almost all features of class inheritance,
overriding methods and properties, inheriting convenience initializers, etc.
In this post, I'll show how protocols and extensions can do the exact same thing,
and yet not suffer from the design flaws an object-oriented approach is most likely to introduce.

## The Object-Oriented Version

Here is a skeleton of the the final class inheritance I defined in [Chapter 6]({{ site.baseurl }}tutorial/chapter-6):

```swift
struct Pokemon { /* ... */ }

class PokemonLover {
  var name: String
  var friends: [PokemonLover] = []

  init(name: String) { /* ... */ }
  convenience init(name: String, bestFriend: PokemonLover) { /* ... */ }

  func makeFriends(with another: PokemonLover) { /* ... */ }
}

class Trainer: PokemonLover {
  var pokemons: [Pokemon]

  init(name: String, pokemons: [Pokemon]) { /* ... */ }
}

class GymLeader: Trainer {
  let location: String

  init(name: String, location: String, pokemons: [Pokemon]) { /* ... */ }
}

class EliteFourMember: Trainer {
  let specialty: SpeciesType

  init(name: String, specialty: SpeciesType, pokemons: [Pokemon]) { /* ... */ }
}

```

In addition to this stub,
it should be noted that I overrode the `init(name:)` designated initializer of `PokemonLover` in `Trainer`.
I also overrode the `makeFriends` method, and `name` property of `PokemonLover` in `EliteFourMember`.
Finally, I created one instance of `PokemonLover` (Professor Oak),
one instance of `Trainer` (Ash),
one instance of `GymLeader` (Brock) and
one instance of `EliteFourMember` (Malva).
The complete code of this object-oriented version is available [here]({{ site.baseurl }}annexes/2017-02-17/object_oriented.swift).

## The Protocol-Oriented Version

The first step is to define a protocol for everything these classes have in common.
Notice that we omit the initializers on purpose,
as most of them were there just because the class had one.
A conforming type will be free to initialize its properties the way it wants.
Only the convenience initializer of `PokemonLover` deserves our attention,
but we'll see how to handle it later.

```swift
protocol Person {
  var name: String { get set }
  var friends: [Person] { get set }

  mutating func makeFriends(with another: inout Person)
}

protocol PokemonHolder {
  var pokemons: [Pokemon]
}

protocol Champion {
  var location { get }
}

protocol Specialist {
  var specialty: SpeciesType { get }
}
```

> The function `makeFriends(with:)` has to mark its parameter inout,
> because we want to be able to modify the other person as well.
> This wasn't required in the object-oriented version, as the `PokemonLover` class was a reference,
> so its properties were mutable anyway.

There's no reason to make this protocols conform to each other,
and each notion can remain completely orthogonal to the others.
In the greater scheme, nothing *requires* a champion to also be a Person.
Maybe one day Pokemons will be allowed to be gym leaders as well,
and they'll greatly appreciate our application handles this case in a swift (pun intended).

Now we can define our types, value type!

```swift
struct PokemonLover: Person {
  var name: String
  var friends: [PokemonLover]

  mutating func makeFriends(with another: PokemonLover) {
    self.friends.append(another)
    another.friends.append(self)
  }

}

struct Trainer: Person, PokemonHolder {
  var name: String
  var friends: [Person]
  var pokemons: [Pokemon]

  mutating func makeFriends(with another: inout Person) {
    self.friends.append(another)
    another.friends.append(self)
  }
}
```

Hmm, let's stop for a second.
It starts to look like we're gonna have a lot of pasting with this `makeFriend(with:)` method.
Let's put it in an extension so we can factor it out of our two first structures:

```swift
extension Person {
    mutating func makeFriends(with another: inout Person) {
        self.friends.append(another)
        another.friends.append(self)
    }
}
```

Notice also that (at least for now) we don't need these boring designated initializers that were simply forwarding
their arguments to our class properties.
Instead, we can take advantage of the memberwise initializers Swift kindly gives us.

Okay, let's continue:

```swift
struct GymLeader: Person, PokemonHolder, Champion {
    var name: String
    var friends: [Person]
    var pokemons: [Pokemon]
    let location: String
}

struct EliteFourMember: Person, PokemonHolder, Specialist {
    var name: String
    var friends: [Person]
    var pokemons: [Pokemon]
    let specialty: SpeciesType
}
```

And that's it for the structures.
Now let's reimplement the initializer of `Trainer` so it matches that of our class version:

```swift
struct Trainer: Person, PokemonHolder {
  /* ... */

  init(name: String, friends: [Person] = [], pokemons: [Pokemon] = []) {
    print("new challenger approaching")
    self.name = name
    self.friends = friends
    self.pokemons = pokemons
  }
}
```

Let's also reimplement `EliteFourMember`'s properties so that they match that of our class version:

```swift
struct EliteFourMember: Person, PokemonHolder, Specialist {
  /* ...*/

  var _name: String
  var name: String {
    get {
      return "Elite Four \(self._name)"
    }

    set {
      if newValue.hasPrefix("Elite Four ") {
        self._name = String(newValue.characters.dropFirst(11))
      } else {
        self._name = newValue
      }
    }
  }

  mutating func makeFriends(with another: inout Person) {
    guard another is EliteFourMember else {
      print("Elite Four members make friends with other Elite Four members only")
      return
    }

    self.friends.append(another)
    another.friends.append(self)
  }
}
```

The last thing we didn't translated is the convenience initializer of `PokemonLover`.
It had been made available in the `Trainer` class,
because it implemented all designated initializer of its base classe (i.e. `PokemonLover`).
The problem is now that `init(name:bestFriend:)` won't initialize the `pokemons` property of the `Trainer` structure,
we we can't factorize it in an extension to the `Person` protocol.
But we can define another protocol for just that initializer:

```swift
protocol InitializableWithBestFriend {
  init(name: String, bestFriend: Person)
}

extension PokemonLover: InitializableWithBestFriend {
  init(name: String, bestFriend: Person) {
    self.init(name: name, friends: [bestFriend])
  }
}

extension Trainer: InitializableWithBestFriend {
  init(name: String, bestFriend: Person) {
    self.init(name: name, friends: [bestFriend], pokemons: [])
  }
}
```

And we're finished.
The complete code of this protocol-oriented version is available [here]({{ site.baseurl }}annexes/2017-02-17/protocol_oriented.swift).

## Comparing the Two Approaches

If we compare the file sizes, the protocol-oriented version is 118 lines,
while the object-oriented version is 90.
The difference isn't so big but still favors the object-oriented version.
So why is this post is called "A Case Against Class Inheritance"?

First of all, now that we only have value types,
we can safely forget about object copies or unintended captured references.
If we try anything funny,
Swift will scold us about some misuse of immutable `self`.
But mostly, we've gained flexibility.
It doesn't look like so, but our class hierarchy is actually quite rigid:

* There's no way for an Elite Four member to also be a Gym Trainer.
* There's no way to define another kind of Elite Four member without a specialty.
* There's np way to define another kind of Pokemon trainer without friends
  (seriously, who's friend with that obnoxious bug catcher who always want to fight with its
  team of useless team of Weedle).

Besides, making changes in that hierarchy is very committing.
Let's say we want the `EliteFourMember` class to be associated with a location as well.
Either we'll have to redefine a `location` property,
and potentially all the method helpers that we'd have defined in the `GymLeader` class
(which means code duplication).
Or we'll have to push everything in the `Trainer` class,
even if it doesn't make any sense for a trainer to be associated with a location.
As time goes, we'll realize that we should in fact push this even deeper in the `PokemonLover` class,
because some other other subclass might need the `location` property.
And finally, we'll end up with a 112 lines `PokemonLover` class that does everything, for everybody.
This problem is well known, and the motivation for at least a thousands of blog posts about composition over inheritance
(or put it more bluntly, why Java is a disaster).

Protocols are an interesting alternative to that issue.
By defining very orthogonal notions,
it's easy to describe what our types should be able to do, and not more.
If we want to associate a location to `EliteFourMember` in our protocol-oriented version,
we can simply make it conform to the `Champion` protocol.
In parallel, extensions allows us to eliminate (or to the very least drastically reduce) code duplication.

If you're not convinced yet, there's a great WWDC 2015 talk by Dave Abrahams on the subject:
[Protocol-Oriented Programming in Swift](https://www.youtube.com/watch?v=g2LwFZatfTI).
Actually, even if you've been convinced, take a look at the talk, it's really great!
