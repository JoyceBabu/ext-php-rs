# Classes

Structs can be exported to PHP as classes with the `#[php_class]` attribute
macro. This attribute derives the `RegisteredClass` trait on your struct, as
well as registering the class to be registered with the `#[php_module]` macro.

## Options

The attribute takes some options to modify the output of the class:

- `name` - Changes the name of the class when exported to PHP. The Rust struct
  name is kept the same. If no name is given, the name of the struct is used.
  Useful for namespacing classes.

There are also additional macros that modify the class. These macros **must** be
placed underneath the `#[php_class]` attribute.

- `#[extends(ce)]` - Sets the parent class of the class. Can only be used once.
  `ce` must be a valid Rust expression when it is called inside the
  `#[php_module]` function.
- `#[implements(ce)]` - Implements the given interface on the class. Can be used
  multiple times. `ce` must be a valid Rust expression when it is called inside
  the `#[php_module]` function.

You may also use the `#[prop]` attribute on a struct field to use the field as a
PHP property. By default, the field will be accessible from PHP publically with
the same name as the field. Property types must implement `IntoZval` and
`FromZval`.

You can rename the property with options:

- `rename` - Allows you to rename the property, e.g.
  `#[prop(rename = "new_name")]`

## Restrictions

### No lifetime parameters

Rust lifetimes are used by the Rust compiler to reason about a program's memory safety.
They are a compile-time only concept;
there is no way to access Rust lifetimes at runtime from a dynamic language like PHP.

As soon as Rust data is exposed to PHP,
there is no guarantee which the Rust compiler can make on how long the data will live.
PHP is a reference-counted language and those references can be held
for an arbitrarily long time, which is untraceable by the Rust compiler.
The only possible way to express this correctly is to require that any `#[php_class]`
does not borrow data for any lifetime shorter than the `'static` lifetime,
i.e. the `#[php_class]` cannot have any lifetime parameters.

When you need to share ownership of data between PHP and Rust,
instead of using borrowed references with lifetimes, consider using
reference-counted smart pointers such as [Arc](https://doc.rust-lang.org/std/sync/struct.Arc.html).

### No generic parameters

A Rust struct `Foo<T>` with a generic parameter `T` generates new compiled implementations
each time it is used with a different concrete type for `T`.
These new implementations are generated by the compiler at each usage site.
This is incompatible with wrapping `Foo` in PHP,
where there needs to be a single compiled implementation of `Foo` which is integrated with the PHP interpreter.

## Example

This example creates a PHP class `Human`, adding a PHP property `address`.

```rust,no_run
# #![cfg_attr(windows, feature(abi_vectorcall))]
# extern crate ext_php_rs;
# use ext_php_rs::prelude::*;
#[php_class]
pub struct Human {
    name: String,
    age: i32,
    #[prop]
    address: String,
}
# #[php_module]
# pub fn get_module(module: ModuleBuilder) -> ModuleBuilder {
#     module
# }
# fn main() {}
```

Create a custom exception `RedisException`, which extends `Exception`, and put
it in the `Redis\Exception` namespace:

```rust,no_run
# #![cfg_attr(windows, feature(abi_vectorcall))]
# extern crate ext_php_rs;
use ext_php_rs::prelude::*;
use ext_php_rs::{exception::PhpException, zend::ce};

#[php_class(name = "Redis\\Exception\\RedisException")]
#[extends(ce::exception())]
#[derive(Default)]
pub struct RedisException;

// Throw our newly created exception
#[php_function]
pub fn throw_exception() -> PhpResult<i32> {
    Err(PhpException::from_class::<RedisException>("Not good!".into()))
}
# #[php_module]
# pub fn get_module(module: ModuleBuilder) -> ModuleBuilder {
#     module
# }
# fn main() {}
```

## Implementing an Interface

To implement an interface, use `#[implements(ce)]` where `ce` is an expression returning a `ClassEntry`.
The following example implements [`ArrayAccess`](https://www.php.net/manual/en/class.arrayaccess.php):
```rust,no_run
# #![cfg_attr(windows, feature(abi_vectorcall))]
# extern crate ext_php_rs;
use ext_php_rs::prelude::*;
use ext_php_rs::{exception::PhpResult, types::Zval, zend::ce};

#[php_class]
#[implements(ce::arrayaccess())]
#[derive(Default)]
pub struct EvenNumbersArray;

/// Returns `true` if the array offset is an even number.
/// Usage:
/// ```php
/// $arr = new EvenNumbersArray();
/// var_dump($arr[0]); // true
/// var_dump($arr[1]); // false
/// var_dump($arr[2]); // true
/// var_dump($arr[3]); // false
/// var_dump($arr[4]); // true
/// var_dump($arr[5] = true); // Fatal error:  Uncaught Exception: Setting values is not supported
/// ```
#[php_impl]
impl EvenNumbersArray {
    pub fn __construct() -> EvenNumbersArray {
        EvenNumbersArray {}
    }
    // We need to use `Zval` because ArrayAccess needs $offset to be a `mixed`
    pub fn offset_exists(&self, offset: &'_ Zval) -> bool {
        offset.is_long()
    }
    pub fn offset_get(&self, offset: &'_ Zval) -> PhpResult<bool> {
        let integer_offset = offset.long().ok_or("Expected integer offset")?;
        Ok(integer_offset % 2 == 0)
    }
    pub fn offset_set(&mut self, _offset: &'_ Zval, _value: &'_ Zval) -> PhpResult {
        Err("Setting values is not supported".into())
    }
    pub fn offset_unset(&mut self, _offset: &'_ Zval) -> PhpResult {
        Err("Setting values is not supported".into())
    }
}
# #[php_module]
# pub fn get_module(module: ModuleBuilder) -> ModuleBuilder {
#     module
# }
# fn main() {}
```