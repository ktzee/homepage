---
title: "Polyglot"
date: 2020-04-25T18:37:29+02:00
draft: false
---

Yatta allows interoperability with other languages on GraalVM through the [Polyglot APIs](https://www.graalvm.org/docs/reference-manual/polyglot/).

## Built-in function `eval`
First form of interoperability with other scripting languages on GraalVM is supported by function `eval`.
This function takes two arguments, first is the symbol of a language, such as `:yatta`, `:js` or `:python` and the second argument is a string of the expression to be evaluated in this language.

Example:

    eval :yatta "raise :test \"test msg\""

## Java interop
Interoperability with Java is an important feature in Yatta and allows calling Java code directly from Yatta and vice-versa.

In order to call Yatta code from Java, one must build a GraalVM Polyglot context and then evaluate a Yatta source using this context:

    import org.graalvm.polyglot.Context;
    import org.graalvm.polyglot.Value;

    Context context = Context.newBuilder().build();
    Value returnValue = context.eval("yatta", "5 + 3");

- currying

### Type-conversions, constructing objects and calling methods
Yatta API for calling code in Java consists of two modules:

* `Java` - for instantiating new objects, checking instance type of an object, raising Java exceptions and casting Java objects
* `java\Types` - for converting Yatta types to Java types (for example Yatta contains only 64bit integers, so they need to be converted to Java integers when calling a Java method that expects an integer. Same for double vs float.

Simple example to create a BigInteger in Yatta and then check that it is actually of `BigInteger` type:

    let
        type = Java::type "java.math.BigInteger"
        instance = Java::new type ["50"]
    in Java::instanceof instance type

### Calling methods on an object
Yatta allows using module call operator `::` to be used to call object methods as well. This is shown in the following example for multiplying two `BigInteger`s:

    let
        type = Java::type "java.math.BigInteger"
        big_two = Java::new type ["2"]
        big_three = Java::new type ["3"]
        result = big_two::multiply big_three  # calling a method `multiply` on object of type BigInteger.
    in result::longValue

### Calling static methods in Java
Static methods may be called just as easily from Yatta:

    java\util\Collections::singletonList 5

Java static methods can be easily called from Yatta, as if it were a Yatta module/function.

### Handling Java exceptions
Raising a Java exception from Yatta is possible thanks to function `Java::throw`:

    try
        let
            type = Java::type "java.lang.ArithmeticException"
            error = Java::new type ["testing error"]
        in Java::throw error
    catch
        (:java, error_msg, _) -> error_msg  # "java.lang.ArithmeticException: testing error" will be the result
    end

The example above shows how to throw and catch Java exceptions in Yatta code.

## Limitations of Polyglot APIs and other notes
* Java interop is implemented via Java reflection. This may be an issue when using native image built version of Yatta interpreter.
* Another consequence of Java interop being implemented via reflection is that when an instance is created using a static method, such as `java.util.Collections.singletonList`, the actual underlying class would be `java.util.Collections$SingletonList` and not a `java.util.List`. This results in a `java.lang.IllegalAccessException` exception if trying to invoke a method of `java.util.List` on such instance. Therefore, one must be careful to understand the limitations of Java reflection APIs.
* Currying on Java static function/method calls works with the same semantics as for Yatta functions. Therefore, one may create a curried Java method simply by not passing all the necessary arguments to method calls.
* Java methods are resolved by names only. Since Yatta is a dynamically typed language, Yatta simply doesn't have the knowledge to determine the correct method otherwise. It cannot even utilize the number of arguments, because the method call may be curried. Therefore it picks the first method of the given name and tries to use that.
* Exceptions related to Polyglot APIs in Yatta have a symbol of `:polyglot`.