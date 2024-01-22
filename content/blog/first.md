+++
title = "The Mechanics of mutable references and dereference in Rust"
date = 2024-01-22
+++

I was working with the sqlx library trying to create a database transaction that I could use to update the database and also insert into the database.

I created the tx like so:

```rust
	 	//create a database transaction in order to be able
    //to insert and update the db atomically
    let mut tx = ctx.db.begin().await.map_err(| err | {
        error!("error starting database transaction: {err}");
        ApiError::InternalServerError
    })?;
```

I then intended to use the transaction as an executor to query the db for user insertion:

```rust
let new_user = sqlx::query_as!(
        User,
        r#"
            insert into users (username, referral_code, referred_by)
            values ($1, $2, $3)
            returning *
        "#,
        new_username,
        new_referral_code,
        referrer.referral_code
    )
    .fetch_one(&mut tx)
    .await
    .map_err(| err | {
        error!("error creating new user from username: {err}");
        ApiError::InternalServerError
    })?;
```

However, the code threw an error that goes like this:

```bash
the trait bound `&mut Transaction<'_, Postgres>: sqlx::Executor<'_>` is not satisfied
the following other types implement trait `sqlx::Executor<'c>`:
  <&'c mut AnyConnection as sqlx::Executor<'c>>
  <&'c mut PgConnection as sqlx::Executor<'c>>
  <&'c mut PgListener as sqlx::Executor<'c>>
  <&Pool<DB> as sqlx::Executor<'p>>rustcClick for full compiler diagnostic
signup.rs(102, 6): required by a bound introduced by this call
```

This error message is pointing out a trait-bound issue. It's saying that the trait **`sqlx::Executor`** is not satisfied for the type **`&mut Transaction<'_, Postgres>`**. However, it is satisfied for a few other types like **`&mut AnyConnection`**, **`&mut PgConnection`**, **`&mut PgListener`**, and **`&Pool<DB>`**. Thank you Rustc, very helpful! How do we solve it? The first logical thing I thought to do is look up sqlx transactions to see how they’re usually composed. 

I came across [this](https://github.com/launchbadge/sqlx/issues/1560) GitHub issue that describes a problem similar to the one I was facing.
