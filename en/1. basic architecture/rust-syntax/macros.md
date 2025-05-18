# Rust Crate Configuration: Allowed Lints and Strict Quality Standards
```rust
#![allow(
    clippy::op_ref,
    clippy::assign_op_pattern,
    clippy::too_many_arguments,
    clippy::suspicious_arithmetic_impl,
    clippy::many_single_char_names,
    clippy::same_item_push
)]
#![deny(intra_doc_link_resolution_failure)]
#![deny(missing_debug_implementations)]
#![deny(missing_docs)]
#![deny(unsafe_code)]
```
## #![allow(...)]
This attribute allows certain lints (warnings) that are normally enabled by the Rust compiler or the Clippy tool. The specific lints being allowed here are:<br>
* `clippy::op_ref`: Allows operations on references instead of values (e.g., &a + &b instead of a + b).
* `clippy::assign_op_pattern`: Suppresses warnings about using assignment operators in patterns.
* `clippy::too_many_arguments`: Disables warnings about functions with a large number of parameters.
* `clippy::suspicious_arithmetic_impl`: Ignores potentially problematic arithmetic implementations.
* `clippy::many_single_char_names`: Allows the use of single-character variable names.
* `clippy::same_item_push`: Suppresses warnings about pushing the same item into a collection multiple times.

## #![deny(...)]
This attribute enforces strict rules by turning certain lints into compile errors. The denied lints are:<br>
* `intra_doc_link_resolution_failure`: Ensures all intra-document links (e.g., [Foo] references) resolve correctly.
* `missing_debug_implementations`: Requires all public types to implement the Debug trait.
* `missing_docs`: Mandates that all public items (modules, functions, types, etc.) have documentation comments.
* `unsafe_code`: Prohibits the use of unsafe code in the crate, ensuring it is 100% safe Rust.

# Macros in Arithmetic Module
### **Macro Syntax Breakdown**

#### **1. Macro Definition**
```rust
macro_rules! impl_add_binop_specify_output {
    ($lhs:ident, $rhs:ident, $output:ident) => { ... };
}
```
- `macro_rules!`: Keyword to define a declarative macro.
- `impl_add_binop_specify_output`: Macro name (convention: `impl_<trait>_<operation>`).
- `($lhs:ident, $rhs:ident, $output:ident)`: Input parameters (3 identifiers).
  - `$lhs`: Left-hand side type (e.g., `MyType`).
  - `$rhs`: Right-hand side type (e.g., `MyType`).
  - `$output`: Result type (e.g., `MyType`).
  - In Rust macros, ident is a macro fragment specifier that matches an identifier, such as a type name, variable name, or trait name.Explanation:
    * ident stands for "identifier".
    * In the macro pattern, $lhs:ident means "match an identifier and bind it to the variable $lhs".
    * For example, if you invoke the macro as impl_add_binop_specify_output!(Foo, Bar, Baz);, then:
        * $lhs will be Foo
        * $rhs will be Bar
        * $output will be Baz
    * These identifiers can then be used inside the macro body to generate code.
- `=>`: Separates the pattern from the expansion.
    - The left side of => is the pattern that the macro matches when it is invoked.
    - The right side of => is the code that will be generated (expanded) when the macro is used.

#### **2. Trait Implementations**
```rust
impl<'b> Add<&'b $rhs> for $lhs { ... }
```
- `impl<'b>`: Generic lifetime parameter `'b` for the reference.
- `Add<&'b $rhs> for $lhs`: Implements `Add` trait for `$lhs + &$rhs`.
- `type Output = $output`: Specifies the result type.

#### **3. Method Implementations**
```rust
fn add(self, rhs: &'b $rhs) -> $output {
    &self + rhs
}
```
- `self`: Takes ownership of `$lhs` (value).
- `rhs: &'b $rhs`: Reference to `$rhs` with lifetime `'b`.
- `&self + rhs`: Delegate to reference-based operation (avoids copying).

### **Key Symbols and Their Meanings**

| Symbol       | Meaning                                                                 |
|--------------|-------------------------------------------------------------------------|
| `$lhs`, `$rhs`, `$output` | Placeholders for types (resolved when the macro is invoked). |
| `:ident`     | Matches an identifier (e.g., type names like `u32`, `MyType`).       |
| `impl`       | Keyword to implement a trait for a type.                               |
| `'a`, `'b`   | Lifetime parameters (ensure references outlive their borrows).        |
| `&`          | Reference type (e.g., `&T` is a reference to `T`).                    |
| `#[inline]`  | Hint to the compiler to inline the function for performance.          |
| `self`       | Represents the instance of the type (value or reference).             |


### **Usage Examples**

#### **1. Basic Addition Implementation**
```rust
// Implement Add for MyType + MyType -> MyType
impl_add_binop_specify_output!(MyType, MyType, MyType);

// Usage:
let a = MyType::new(5);
let b = MyType::new(3);
let c = a + b;     // Calls the generated Add implementation
```

#### **2. Mixed Types**
```rust
// Implement Add for MyType + u32 -> MyType
impl_add_binop_specify_output!(MyType, u32, MyType);

// Usage:
let result = my_type_instance + 42;
```

#### **3. Combined Macros**
```rust
// Implement both Add and Sub for MyType
impl_binops_additive!(MyType, MyType);

// Usage:
let a = MyType::new(5);
let b = MyType::new(3);
let sum = a + b;     // Add
let diff = a - b;    // Sub
a += b;              // AddAssign
a -= b;              // SubAssign
```


### **Generated Code Example**
For `impl_add_binop_specify_output!(MyType, MyType, MyType)`, the macro expands to:
```rust
impl<'b> Add<&'b MyType> for MyType {
    type Output = MyType;
    fn add(self, rhs: &'b MyType) -> MyType { &self + rhs }
}

impl<'a> Add<MyType> for &'a MyType {
    type Output = MyType;
    fn add(self, rhs: MyType) -> MyType { self + &rhs }
}

impl Add<MyType> for MyType {
    type Output = MyType;
    fn add(self, rhs: MyType) -> MyType { &self + &rhs }
}
```


### **Key Features**

1. **Reference Handling**:
   - Automatically handles all combinations of values and references (`T`, `&T`).
   - Delegates to reference-based operations to avoid unnecessary copies.

2. **Custom Output Type**:
   - `$output` allows specifying a different result type (e.g., `BigInt` for `u32 + u32`).

3. **Consistency**:
   - Ensures all trait implementations follow the same pattern.

4. **Performance**:
   - `#[inline]` optimizes method calls.


### **Conclusion**
These macros leverage Rust's macro system to automate boilerplate code for arithmetic operations, making it easier to implement traits for custom types while maintaining type safety and performance.