---
title: "GenServer, Registry, DynamicSupervisor. Combined."
date: 2019-10-05T00:34:00+07:00
tags: ["tech", "programming", "elixir", "concurrency"]
language: "english"
# Alias is used to redirect from old urls used by the previous template
aliases: ["/posts/2019-10-05-genserver-registry-dynamic-supervisor-combined"]
---

_Originally published at [dev.to](https://dev.to/unnawut/genserver-registry-dynamicsupervisor-combined-4i9p)_.

In [omisego/ewallet](https://github.com/omisego/ewallet), we're building/refactoring a `TransactionTracker` that listens to a huge number of transactions (think money transactions) that also happen to be residing in an external source that is slow to update (Hello, Ethereum).

So we need a way for us to:

1. Run the trackers concurrently, so to enable massive amount of transaction tracking.
2. Look up a running tracker, so we can reuse it for different purposes.
3. Automatically restart a specific tracker that goes wonky, because with external sources, anything can go wrong.

With Elixir and OTP/BEAM behind the scene, we are able to solve the problem by utilizing 3 Elixir core features:

1. [`GenServer`](https://hexdocs.pm/elixir/GenServer.html) for building long-running, concurrent tasks.
2. [`Registry`](https://hexdocs.pm/elixir/Registry.html) for looking up those running `GenServer`'s.
3. [`DynamicSupervisor`](https://hexdocs.pm/elixir/DynamicSupervisor.html) for monitoring those arbitrary number of `GenServer`'s, and automatically restart one when it goes wonky.

## The problem

If we were to implement our own custom wiring, we would have to do the following:

1. Start a new tracker process (a `GenServer`) attached to a `DynamicSupervisor`.
2. Register the tracker process with the registry.
3. To invoke the process, lookup the registry for the process ID.
4. Make sure the registry handles the process's crash, and remove the process from the registry.
5. Make sure the registry knows when the process is restarted and registers the new process ID back.
6. Deregister the process from the registry when it shuts down.

That's a lot of code for the registry, and a lot of code to wire up the `GenServer`, `DynamicSupervisor` and `Registry` together. Since there's a lot of moving parts, our implementation and wiring could be very prone to errors. All of this represents very little business value.

## The solution

Because Elixir designed the `GenServer`, `Registry` and `DynamicSupervisor` to work together seamlessly, we are surprised by how few lines of code needed to wire these up together.

```elixir
defmodule TransactionTracker do
  use GenServer

  @registry TransactionTracker.Registry
  @supervisor TransactionTracker.TrackerSupervisor

  def start(transaction_id) do
    opts = [
      transaction_id: transaction_id,
      name: {:via, Registry, {@registry, transaction_id}}
    ]

    DynamicSupervisor.start_child(@supervisor, {__MODULE__, opts})
  end

  def lookup(transaction_id) do
    case Registry.lookup(@registry, transaction_id) do
      [{pid, _}] -> {:ok, pid}
      [] -> {:error, :not_found}
    end
  end

  def start_link(opts) do
    {name, opts} = Keyword.pop(opts, :name)
    GenServer.start_link(__MODULE__, opts, name: name)
  end

  def init(opts) do
    state = %{
      transaction_id: Keyword.fetch!(opts, :transaction_id),
    }

    {:ok, state}
  end
  #...
end

defmodule TransactionTracker.Application do
  use Application

  def start(_type, _args) do
    children = [
      {Registry, keys: :unique, name: TransactionTracker.Registry},
      {DynamicSupervisor, name: TransactionTracker.TrackerSupervisor, strategy: :one_for_one}
    ]

    Supervisor.start_link(children, name: TransactionTracker.Supervisor, strategy: :one_for_one)
  end
end
```

With above, we can now do a simple one-liner to start the tracker:

```elixir
iex> TransactionTracker.start("txn_01dp371w0fnjhf9z2tjebx4vr4")
{:ok, #PID<0.104.0>}
```

This one-line call would automatically:

1. Start a new `TransactionTracker` GenServer for the given transaction ID.
2. Register the tracker with `TransactionTracker.Registry`.
3. Register the tracker with `TransactionTracker.TrackerSupervisor`.
4. Restart the tracker when it shuts down abnormally.
5. Return the correct process ID lookup even after a tracker restart.
6. Deregister the tracker on expected shutdown.
7. Allow interactions with the process via the returned `pid` or by looking up: `TransactionTracker.lookup("txn_01dp371w0fnjhf9z2tjebx4vr4")`

And this is what happens with an abnormal exit.

```elixir
iex> TransactionTracker.start("txn_01dp371w0fnjhf9z2tjebx4vr4")
{:ok, #PID<0.157.0>}

iex> {:ok, pid} = TransactionTracker.lookup("txn_01dp371w0fnjhf9z2tjebx4vr4")
{:ok, #PID<0.157.0>}

iex> :ok = GenServer.stop(pid, :its_a_crash)
17:25:49.286 [error] GenServer {TransactionTracker.Registry, "abcd"} terminating
** (stop) :its_a_crash
Last message: []
State: %{transaction_id: "txn_01dp371w0fnjhf9z2tjebx4vr4"}

iex> {:ok, restarted_pid} = TransactionTracker.lookup("txn_01dp371w0fnjhf9z2tjebx4vr4")
{:ok, #PID<0.162.0>}

iex> :sys.get_state(restarted_pid)
%{transaction_id: "txn_01dp371w0fnjhf9z2tjebx4vr4"}
```

You would see that the lookup returns the new process automatically, and the process holds the same state as the previous one. Just start the process with your ideal identifier and you'll be able to access the process from anywhere, with guarantee that it'll point you to the correct process even if the process got restarted and the process ID changed.

Isn't it pretty?

## Conclusion

By using Elixir's [`GenServer`](https://hexdocs.pm/elixir/GenServer.html), [`Registry`](https://hexdocs.pm/elixir/Registry.html) and [`DynamicSupervisor`](https://hexdocs.pm/elixir/DynamicSupervisor.html), we're able to reap the following benefits when running long-running tasks.

1. A one-liner way to start a long-running process, encapsulating away the registry and supervisor.
2. The process, when it goes woowoo, gets restarted automatically by the supervisor.
3. The registry handles a process's shutdown automatically, so no need to worry about deregistering dead processes.
4. The process can be looked up via the registry with ease, using our own arbitrary identifier, and works across process crashes.

What do you think? Do you have better ways to manage long-running processes? Let me know!
