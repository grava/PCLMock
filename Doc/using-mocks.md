# Using Mocks

Once a mock has been [defined](defining-mocks.md), you can of course use it in your unit tests for anything that depends on the mocked interface (or class). For example:

```C#
var mock = new AuthenticationServiceMock();
var sut = new MyViewModel(mock);
```

Here we have a view model that requires an instance of `IAuthenticationService`.

## Returning Specific Values

To specify that an invocation returns a certain value:

```C#
// a string property
mock.When(x => x.SomeProperty)
    .Return("foo");

// a parameterless string method
mock.When(x => x.SomeMethod())
    .Return("foo");

// a string method that takes two parameters
mock.When(x => x.SomeMethod(It.IsAny<int>(), It.IsAny<string>())
    .Return("foo");
```

A callback can also be provided so that the return value can be determined dynamically:

```C#
var valueToReturn = "foo";
mock.When(x => x.SomeMethod())
    .Return(() => valueToReturn + "bar");
```

If the specification is against a method that takes parameters, the callback can accept those parameters:

```C#
mock.When(x => x.SomeMethod(It.IsAny<int>(), It.IsAny<string>())
    .Return((int i, string s) => i == 0 ? "zero" : "not zero");
```

## Throwing Exceptions

To specify that an invocation throws an exception:

```C#
mock.When(x => x.SomeProperty)
    .Throw(new InvalidOperationException("some message"));
```

If you just want to throw an exception and don't particularly care about the exception type: 

```C#
mock.When(x => x.SomeProperty)
    .Throw();
```

## Invoking Callbacks

To specify that an invocation results in a callback being invoked:

```C#
var called = false;
mock.When(x => x.SomeProperty)
    .Do(() => called = true);
```

If the specification is against a method that takes parameters, the callback can accept those parameters:

```C#
var calledWithZero = false;
mock.When(x => x.SomeMethod(It.IsAny<int>(), It.IsAny<string>())
    .Do((int i, string s) => calledWithZero = i == 0);
```

After your `Do` specification, you can continue with a `Return` or `Throw` if relevant:

```C#
var calledWithZero = false;
mock.When(x => x.SomeMethod(It.IsAny<int>(), It.IsAny<string>())
    .Do((int i, string s) => calledWithZero = i == 0)
    .Return("foo");
```

## Matching Arguments

When writing specifications, you can use the `It` class to define how arguments in your specification are to be filtered. Here are some examples:

```C#
// accept any string at all
mock.When(x => x.SomeMethod(It.IsAny<string>()))
    .Return(1);

// accept only a specific string
mock.When(x => x.SomeMethod(It.Is("foo")))
    .Return(1);

// same as above, only easier to read and write
mock.When(x => x.SomeMethod("foo"))
    .Return(1);

// accept any of a set of values
mock.When(x => x.SomeMethod(It.IsIn("foo", "bar"))
    .Return(1);

// accept values matching a regular expression
mock.When(x => x.SomeMethod(It.IsLike("[Hh]ello"))
    .Return(1);
```

Property setters are specified slightly differently:

```C#
// accept any string at all
mock.WhenPropertySet(x => x.SomeProperty)
    .Do(...);

// accept only the specified string
mock.WhenPropertySet(x => x.SomeProperty, It.Is("foo"))
    .Do(...);

// same as above, only easier to read and write
mock.WhenPropertySet(x => x.SomeProperty, "foo")
    .Do(...);

// accept any of a set of values
mock.WhenPropertySet(x => x.SomeProperty, It.IsIn("foo", "bar")
    .Do(...);

// accept values matching a regular expression
mock.WhenPropertySet(x => x.SomeProperty, It.IsLike("[Hh]ello")
    .Do(...);
```

There are a whole range of methods on the `It` class to help you specify constraints against arguments:

* `IsAny<T>`
* `Is<T>`
* `IsNot<T>`
* `IsNull<T>`
* `IsNotNull<T>`
* `IsOfType<T>`
* `IsNotOfType<T>`
* `IsLessThan<T>`
* `IsLessThanOrEqualTo<T>`
* `IsGreaterThan<T>`
* `IsGreaterThanOrEqualTo<T>`
* `IsBetween<T>`
* `IsNotBetween<T>`
* `IsLike`
* `IsNotLike`
* `Matches`

Multiple specifications can be configured, each one for different arguments:

```C#
mock.When(x => x.SomeMethod("foo"))
    .Return(1);
    
mock.When(x => x.SomeMethod("bar"))
    .Return(2);

Assert.Equals(1, mock.SomeMethod("foo"));
Assert.Equals(2, mock.SomeMethod("bar"));
```

Note that order is important. Later specifications take precedence over earlier ones:

```C#
// these are the wrong way around to illustrate
mock.When(x => x.SomeMethod("foo"))
    .Return(1);
    
mock.When(x => x.SomeMethod(It.IsAny<string>()))
    .Return(2);

// this will fail because 2 will be returned!
Assert.Equals(1, mock.SomeMethod("foo"));
```

### `ref` and `out` Parameters

If a mocked method takes a `ref` or `out` parameter, you can provide a value via the `AssignOutOrRefParameter` method. Consider the following example:

```C#
var mock = ...;
string s;
mock.When(x => x.Method(out s))
    .AssignOutOrRefParameter(0, "out value")
    .Return("something");
```

Here, `Method` takes a single parameter which is marked as `out`. As such, in order to call `Method` we need a `string` variable. We declare this string as `s` and use it in our `When` specification. We could have called it anything, of course.

The `AssignOutOrRefParameter` is the important part. It allows us to dictate what the consumer of the mock will receive for the `out` parameter. In this case, we're saying that the parameter at index zero (i.e. the first parameter) will be assigned the value `"out value"`. This would work equally well if the parameter was marked as `ref` rather than `out`.

The need for a parameter index is unfortunate, but cannot be avoided.

## Verification

Calls against mocks can be verified via the `Verify` and `VerifyPropertySet` methods:

```C#
mock.Verify(x => x.SomeMethod("foo"))
    .WasCalledExactlyOnce();
    
mock.Verify(x => x.SomeOtherMethod(It.IsIn("foo", "bar"))
    .WasCalledAtLeast(times: 3);
    
mock.VerifyPropertySet(x => x.SomeProperty, "foo")
    .WasCalledExactly(times: 2);
    
mock.VerifyPropertySet(x => x.SomeProperty, "bar")
    .WasNotCalled();
```

If verification fails, a `VerificationException` is thrown with a helpful message.