+++
title = "The Mechanics of Ownership, References, and Dereferences in Rust"
date = 2024-02-07
+++
The design choices of a programming language has direct implications on how we write code. One such design choice is the concept of ownership and borrowing in the Rust programming language. 

## Ownership, References and Immutability

Stated simply, the concept of ownership deems that we can only have one owner of a value at a time, this value can be passed around severally for read-only operations by immutably borrowing or referencing, but we can only have one mutable borrow at a time. 

```rust
fn main() {
    let mut input = String::new();
    input.push_str("Hello World");
    println!("input value is: {:?}", input);

    let take_ownership = input;
    println!("Taken ownership of value {:?}", take_ownership);

    let _illegal = input; // throws an error -> use of moved value input
}
```

The error message tells us we are trying to use a value which has already been moved to another memory location. When we create the variable `take_ownership`, we move the value in the `input`'s memory location to `take_ownership`, thus trying to allocate `input`'s value to another variable fails, because the value is no longer there. This is all by design, imagine if we were successful in having two variables own the value at the same time, and then have both variables mutate the value, we got ourselves a race condition, not good. Innit?

There's a work-around to this that makes the error go away quickly; Instead of taking ownership, we just borrow the value
```rust
let borrow_value = &input;
println!(...);

let another_borrow = &input
println!(...);
```
The ampersand sign `&` denotes that we are taking a reference to the memory address the value is stored, but we're not taking ownership of the value itself. Now we can have as many borrows as we want, but there's a catch: We can only have one mutable reference at a time.
```rust
let borrow_value = &mut input; 
let another_borrow = &mut input;
```

If you tried writing code like the one above, your Rust [LSP](https://microsoft.github.io/language-server-protocol/) should already be telling you that what you're doing is unacceptable:

```rust
error[E0499]: cannot borrow input as mutable more than once
```
But it's possible to have another mutable borrow when the first borrow goes out of scope. The code below runs successfully because we create the variable `another_borrow` after `borrow_value` has already been used and it's scope ended. 

```rust
let borrow_value = &mut input;
borrow_value.push_str(" Changers");

let another_borrow = &mut input;
println!("another_borrow: {:?}", another_borrow);
```

The reference's scope start from where it is introduced and continues through the last time that reference is used; when the reference goes out of scope, it's borrow ends, and the value becomes available for borrowing again. When all references to the value are gone, Rust frees the memory where the value is located.  These rules of ownership, borrowing and scope are what enables Rust to be able to allocate and free memory safely without a garbage collector, how practical!
## Dereferencing

We've established that references do not hold any values by themselves, instead they point to the location in memory where the actual value is stored. Now, what if we want to modify the actual value the reference is pointing at? Take this simple example:

```rust
let mut value = 20;
let ref_val = &mut value;
```
Imagine that we want to change the `ref_value` to 30; to a new Rustacean, the first thought would be to do something like this:

```rust
fn main() {
   let mut value = 20;
   let ref_val = &mut value;
   ref_val = 30;
}
```

Trying to write the above code will have the compiler sympathizing with us:

```bash
error[E0308]: mismatched types
  --> src/main.rs:20:17
   |
|     let ref_value = &mut value;
   |                     ---------- expected due to this value
|     ref_value = 30;
   |                 ^^ expected &mut {integer}, found integer
   |
help: consider dereferencing here to assign to the mutably borrowed value
   |
 |     *ref_value = 30;
   |     +
```
I encourage you to take a minute and read the error message keenly. We get the error because `ref_value` holds the reference to the value, and not the value itself; in other terms, we are trying to assign to the reference, and not the value it points to, and Rust does not allow us to assign directly to the reference because this would mutate the actual value. So, what do we do? The compiler has already helped us with that. To assign the integer to the actual value, we need to *dereference*` ref_value` and access the actual value behind it; we do this using the unary operator` *`. 

   ```rust
let mut value = 20;
let ref_val = &mut value;
*ref_val = 30;
```

One question you may already have is why we didn't have to do that with the `push_str` operations earlier:

```rust
let borrow_value = &mut input;
borrow_value.push_str(" Changers");
```
That's because when we use the dot operator, the expression on the left-hand side of the dot is auto-referenced/auto-derefenced automatically. You can read more about the dot-operator [here](https://doc.rust-lang.org/nomicon/dot-operator.html).

## A more Interesting Problem

This article was inspired by a problem I encountered while working with the [sqlx](https://docs.rs/sqlx/latest/sqlx/index.html) library; I wanted to have a database connection that I can reuse across multiple queries. Sqlx has a `begin` method that creates a database connection and immediately begins a new transaction, so what I need to do is figure out how to compose multi-statement transactions with it. My first attempt looks like so:

```rust
async fn demo_txn(db: PgPool) -> Result<()> {
	let tx = db.begin().await.map_errr(|err| {
        error!("error starting database transaction: {err}");`
    })?;
    
	let row = sqlx::query("...").fetch_one(&mut tx).await?;
	
	sqlx::query("...").execute(&mut tx).await?;
}
```

Above, I create a transaction, and then try to consume a mutable reference to the transaction in my queries; however, the code throws this error:

```bash
error[E0277]: the trait bound &mut Transaction<'_, Postgres>: sqlx::Executor
<'_> is not satisfied
   --> src/routes/signup.rs:117:14
    |
 |     .execute(&mut tx)
    |      ------- ^^^^^^^ the trait sqlx::Executor<'_> is not implemented 
for &mut Transaction<'_, Postgres>
    |      |
    |      required by a bound introduced by this call
    |
    = help: the following other types implement trait sqlx::Executor<'c>:
              <&'c mut PgConnection as sqlx::Executor<'c>>
              <&'c mut PgListener as sqlx::Executor<'c>>
              <&'c mut AnyConnection as sqlx::Executor<'c>>
              <&Pool<DB> as sqlx::Executor<'p>>
```

The error tells us that a trait we need is not implemented for our mutable reference to the `Transaction` type. These are one of those errors that have you scratching your head because the compiler has no more help to offer you, we are basically on our own now. The logical thing to do here is to dig into other people's code and see if I can find out how others construct multi-statement transactions in sqlx. I came across a similar problem on Github, and the solution offered is to dereference the transaction:

```rust
async fn demo_txn(db: PgPool) -> Result<()> {
	let tx = db.begin().await.map_errr(|err| {
        error!("error starting database transaction: {err}");
    })?;
    
	let row = sqlx::query("...").fetch_one(&mut *tx).await?; // notice the *
	
	sqlx::query("...").execute(&mut *tx).await?; //notice *
}
```

And true enough, the errors go away; interesting. What happens here is that when we *dereference* the type `&mut Transaction`, we are left with the referent type `Transaction` which implements the missing traits, resolving the compilation error. This example goes to show how some of the errors you may account even with trait bound issues are directly tied to the concepts of ownership and references.

## Finishing Up

Thanks for reading! I hope this article has helped shine some light on arguably some of the most important Rust concepts. 
