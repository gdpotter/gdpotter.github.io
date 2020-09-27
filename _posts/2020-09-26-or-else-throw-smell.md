---
layout: post
title: Java's Optional.orElseThrow() is a Code Smell
comments: true
crosspost_to_medium: false
---
Anytime a developer uses `.orElseThrow()` they are trying to express one of two things:
1. "If the value is missing, that would be unexpected (but possible) and we want to throw an exception"
2. "At this point, there should **definitely** be a value here"

That first case is part of what `Optional` was designed to do. For example, you might have some code that looks like the following:
```java
public void deactivateUser(int userId) {
    User user = userRepository.findById(userId)
        .orElseThrow(() -> new NotFoundException(...));

    user.deactivate();
}
```

This is a totally acceptable usage of Optional. An invalid id would be _exceptional_ and there's no way to catch that at compile time. The non-optional version of this code is as follows:
```java
public void deactivateUser(int userId) {
    User user = userRepository.findById(userId)

    if (user == null) {
        throw new NotFoundException(...);
    }

    user.deactivate();
}
```

Although not as modern, the logic intuitively makes sense and expresses the desired behavior.

## So what's the problem?

Let's consider the following code. What is the author trying to do?
```java
// BEFORE
public class Person {
  ...

  public Optional<String> getFirstName() {
    ...
  }

  public Optional<String> getLastName() {
    ...
  }
}

public void printFullName(Person person) {
    person.getFirstName().ifPresent(firstName -> {
        String lastName = person.getLastName().orElseThrow(); // Last name is definitely there if they have a first name
        System.out.println(firstName + " " + lastName);
    });
}
```

I see this type of code often, although it may take different forms. Sometimes they pass an exception in (i.e. `() -> new RuntimeException("This should never happen!")`) and sometimes they use a naked `.get()`. But they all mean the same thing: **I know something contextual that the compiler doesn't know.** But why? Being a strongly-typed language, Java's compiler genuinely wants to help you. But the author of this code isn't letting it help. When you write something like this, you should instead be asking yourself, "How can I help the compiler help me? How can I clue Java into this contextual information?"

So let's look back at that example and think about what we are actually trying to say. First and last name are either both present or both absent. Since they are logically coupled (hence why the author was comfortable just calling `.orElseThrow()`) let's group them together in the code as well. Instead of having two optional fields, let's create a class with two required fields and then _that_ class can be optional.
```java
// AFTER
public class Name {
    private final String firstName;
    private final String lastName;

    // Constructor, getters, etc omitted for brevity
}

public class Person {
    ...
    public Optional<Name> getName() {
       ...
    }
    ...
}

public void printFullName(Person person) {
    person.getName().ifPresent(name -> {
        String firstName = name.getFirstName();
        String lastName = name.getLastName();
        System.out.println(firstName + " " + lastName);
    });
}
```

Now the compiler makes it impossible for someone to change something and accidentally end up in a situation where the last name is missing. And if you're using JPA (like Hibernate) this can be a great use of [`@Embeddable` and `@Embedded`](https://www.baeldung.com/jpa-embedded-embeddable).

In addition to hiding business-context, another place I commonly see this anti-pattern is hiding simple code flow context. What do you think of the following:

```java
// BEFORE
public class Person {
  ...
  public void randomizeFavorites() {
    this.favoriteColor = Math.random() > 0.5 ? "Blue" : "Green";
    this.favoriteNumber = Math.floor(Math.random() * 100);
  }
}

public void spinWheel(Person person) {
  person.randomizeFavorites();

  // favoriteColor and favoriteNumber were just set above, so orElseThrow is safe
  String favoriteColor = person.getFavoriteColor().orElseThrow();
  String favoriteNumber = person.getFavoriteNumber().orElseThrow();

  System.out.println("New favorite color/number: " + favoriteColor + ", " + favoriteNumber);
}
```
Here, the author feels okay calling orElseThrow() since they _know_ that the fields would have been set as part of `randomizeFavorites()`. But what if someone were to change that implementation such that it no longer set the favoriteNumber? Well, exceptions would start being thrown and the compiler couldn't have saved you.

This one is a bit trickier, but I would probably improve this code by having the `randomizeFavorites()` method _return_ what it did:
```java
// AFTER
public class Person {
  ...
  public RandomFavoriteResult randomizeFavorites() {
    this.favoriteColor = Math.random() > 0.5 ? "Blue" : "Green";
    this.favoriteNumber = Math.floor(Math.random() * 100);
    return new RandomFavoriteResult(favoriteColor, favoriteNumber);
  }
}

public void spinWheel(Person person) {
  RandomFavoriteResult favorites = person.randomizeFavorites();

  String favoriteColor = favorites.getFavoriteColor();
  String favoriteNumber = favorites.getFavoriteNumber();

  System.out.println("New favorite color/number: " + favoriteColor + ", " + favoriteNumber);
}
```

Like the first example, here we've been able to introduce a new class to encapsulate the previously-hidden context. Now, if someone were to change that method to no longer set the favoriteNumber, something would have to change with that return type (and thus the usages of `RandomFavoriteResult.getFavoriteNumber()`). This dependency that `spinWheel` has on the implementation of `randomizeFavorites` is now discoverable and documented thanks to Java's type-safety.

## Conclusion

You've seen that using `.orElseThrow()` can often be a signal that you are hiding some context from the compiler. I challenge you to follow these smells and try to find ways to reorganize. Feel free to reach out in the comments below if this was helpful or if you're stuck trying to refactor!
