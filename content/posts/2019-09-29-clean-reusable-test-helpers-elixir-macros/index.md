---
title: "Clean and reusable test helpers with Elixir macros"
date: 2019-09-29T02:09:00+07:00
tags: ["tech", "programming", "elixir", "testing"]
language: "english"
---

_Originally published at [dev.to](https://dev.to/unnawut/clean-and-reusable-test-helpers-with-elixir-macros-26e5)_.

Sometimes, seemingly redundant tests could serve as an assurance that our code works. While it's possible to abstract away the tests to a higher level, it may mean sacrificing readability.

In this post, I'd like to suggest a way to create test helpers that can be reused easily and works seamlessly with ExUnit. While we generally avoid creating macros as the [official guide](https://elixir-lang.org/getting-started/meta/macros.html) says:

> Macros should only be used as a last resort. Remember that explicit is better than implicit. Clear code is better than concise code.

I believe this post presents an exact use case where macros allow us to have explicit, concise and clean tests at the same time.

## The problem

To begin, let's say we have two schemas called `User` and `Account`. Each of them contain a `name` field that should not be blank. We could add tests like this:

```elixir
defmodule UserTest do
  use ExUnit.Case

  describe "insert/1" do
    #...
    test "fails when given a blank name" do
      {:error, changeset} = User.insert(%{name: nil})
      assert changeset.errors == [{:name, {"can't be blank", [validation: :required]}}]
    end
  end
end

defmodule AccountTest do
  use ExUnit.Case

  describe "insert/1" do
    #...
    test "fails when given a blank name" do
      {:error, changeset} = Account.insert(%{name: nil})
      assert changeset.errors == [{:name, {"can't be blank", [validation: :required]}}]
    end
  end
end
```

Imagine if you have a dozen of schemas, most of which will require a check for blank fields. How much of your test code will be redundant? And how incomprehensible it would be, as the test code gets larger and larger?

## Interim solution: Helper functions

The interim solution we tried was by abstracting away the assertion into a helper function, like below:

```elixir
defmodule TestHelper do
  def not_blank(schema, field) do
    {result, changeset} = schema.insert(%{field => nil})

    assert result == :error
    assert changeset.errors == [{field, {"can't be blank", [validation: :required]}}]
  end
end

defmodule UserTest do
  use ExUnit.Case
  import TestHelper

  describe "insert/1" do
    #...
    test "fails when given a blank name" do
      assert not_blank(User, :name)
    end
  end
end

defmodule AccountTest do
  use ExUnit.Case
  import TestHelper

  describe "insert/1" do
    #...
    test "fails when given a blank name" do
      assert not_blank(Account, :name)
    end
  end
end
```

While above works great, the problem is that it is still cluttered when you want many assertions in a single test case, or you rather prefer lean test cases by testing one thing at a time.

## The real deal: Macros as test helpers

Here's how we use macros to generate clean test cases:

```elixir
defmodule TestHelper do
  defmacro test_insert_prevent_blank(schema, field) do
    quote do
      test "fails when given a blank :#{unquote(field)}" do
        schema = unquote(schema)
        field = unquote(field)

        {result, changeset} =
          schema
          |> get_factory
          |> params_for(%{field => ""})
          |> schema.insert

        assert result == :error
        assert changeset.errors == [{field, {"can't be blank", [validation: :required]}}]
      end
    end
  end
end
```

With the macro above, we can now do one-liners like these:

```elixir
defmodule UserTest do
  use ExUnit.Case
  import TestHelper

  describe "insert/1" do
    #...
    test_insert_prevent_blank(User, :name)
  end
end

defmodule AccountTest do
  use ExUnit.Case
  import TestHelper

  describe "insert/1" do
    #...
    test_insert_prevent_blank(Account, :name)
  end
end
```

And it is very readable when combined with other similar test helpers:

```elixir
defmodule UserTest do
  use ExUnit.Case
  import TestHelper

  describe "insert/1" do
    test_insert_generate_uuid(User, :uuid)

    test_insert_prevent_blank(User, :name)
    test_insert_prevent_blank(User, :email)
    test_insert_prevent_duplicate(User, :email)

    test_insert_generate_timestamps(User)

    # Other schema-specific tests...
  end
end
```

## Conclusion

We have been using this test-helper-as-a-macro approach in our project at [`omisego/ewallet`](https://github.com/omisego/ewallet) with satisfaction. It has worked well so far with the following benefits:

1. One-liner tests. Helps optimize valuable screen estate for browsing through the tests. Meanwhile the tests still have their explicity, not being hidden away behind some higher-level abstraction.
2. Not having to worry about human-error applying a change to the test behavior across the codebase. Even the test names are reflected by the macro.
3. When a test fails, its error pinpoints to the exact assertion. The error messages are very clear, and the test name represents the assertion exactly.
4. We're not hacking how ExUnit works, relying purely on ExUnit's public API.

If you find this approach interesting, you can find real-world examples of the helper macros at [`EWalletDB.SchemaCase`](https://github.com/omisego/ewallet/blob/master/apps/ewallet_db/test/support/schema_case.ex), and usage at [`EWalletDB.RoleTest`](https://github.com/omisego/ewallet/blob/master/apps/ewallet_db/test/ewallet_db/role_test.exs).

What do you think? Do you find any drawback or a better solution? Let me know!
