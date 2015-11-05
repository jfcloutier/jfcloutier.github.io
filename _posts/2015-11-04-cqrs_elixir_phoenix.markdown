---
layout: post
title:  "CQRS with Elixir and Phoenix"
date:   2015-11-04 13:37:24 -0500
categories: jekyll update
---

Wherein I describe how easy it was to implement the CQRS pattern in Elixir.

I am currently implementing a REST API for a California startup (it's all very hush hush). Since I could pick my own tools, I gleefully chose [Elixir](http://elixir-lang.org) and the [Phoenix Framework](http://http://www.phoenixframework.org/).

_Note: Elixir can be thought of as a Ruby-ish dialect of [Erlang](http://www.erlang.org/), plus really great meta-programming capabilities_

The REST API I am responsible for is quite rich and bound to grow quickly over time. The API is a mix of commands (calls that alter the database) and queries (calls that fetch data from the database).

To handle the expected complexity, I went for an architecture that cleanly separates concerns. Among other things, I tried to clearly distinguish commands from queries, following the [CQRS - Command Query Responsibility Separation pattern](http://http://cqrs.nu/Faq/command-query-responsibility-segregation)

>[CQRS] means that a method is either a command performing an action, or a query that returns data, but not both.
>Being purely action-performing methods, commands always have a void return type.
>Queries, on the other hand, should be idempotent, that is, they don't have any visible side effects on the system.

I found that the mix of Phoenix controllers, [OTP](http://learnyousomeerlang.com/what-is-otp) and [Elixir macros] (http://elixir-lang.org/getting-started/meta/macros.html) made it very easy for me to implement my own simplified version of CQRS.

First, a bird's eye view of the architecture as it pertains to CQRS:

![architecture](/assets/cqrs_elixir.png)

In a nutshell:

* The Phoenix Endpoint receives REST calls and routes them to controller functions
* Each controller function converts the JSON parameters into arguments for calls to application service functions (each application service is a supervised [GenServer](http://20bits.com/article/erlang-a-generic-server-tutorial))
* An application service function invoked by a Phoenix controller is either a command or a query
* Commands and queries hit the same database (CQRS would rather have each target a different)
* Execution of a command raises a "command event" sent to an [Event Manager](http://www.tattdcodemonkey.com/blog/2015/4/24/event-handling-in-elixir) that dispatches it to a command event handler
* The command event handler journals the event on a remote object store and, periodically, saves a snapshot of the entire database to that same store
* Queries don't raise events but they do go through a caching server (queries are idempotent, so unless a command has been executed between two repeated queries, the results wil be the same and can be cached)

It was remarkably easy to implement the CQRS pattern for two reasons:

* All API calls are funneled through GenServers (my application services)
* A GenServer is only one thin layer short of explicit commands vs queries

First, here's how a GenServer API function looks like that a Phoenix controller would call:

{% highlight elixir %}
defmodule AnApplicationService do
  use GenServer
  #...
  
  def update_something(param1, param2) do
    GenServer.call(@name, {:do_something, param1, param2})
  end
  
  #...
end
{% endhighlight %}

The GenServer API function invokes a named GenServer process via a callback that looks likes this:

{% highlight elixir %}
defmodule SomeApplicationService do
  use GenServer
  #...
  
  def handle_call({:do_something, param1, param2}, caller, state) do
    # ...
    result = SomeDataService.write_to_db(...)
    {:reply, {:ok, result}, state}
  end
  
  #...
end
{% endhighlight %}

The important point is that the Genserver process is called with an *immutable* data structure ({:do_something, param1, param2} which is a tuple). The tuple is the command.

Queries handled by a GenServer/application service look the same. The difference, which is so far only implied in the code, is that a command will alter the database whereas a query will not.

All I need to do is make that difference explicit and "wrap" some behavior around all commands (so they get journaled) and around all queries (so they are cached).

As it turns out, Elixir makes it ridiculously easy, thanks to macros.

So I defined these two macros:

{% highlight elixir %}
defmodule CqrsMacros do
  #...
  
  @doc "Execute a command for a named server"
  defmacro command(name, command) do
    quote bind_quoted: [name: name, command: command], unquote: true do
      usecs = Timex.Time.now() |> Timex.Time.to_usecs()
      result = GenServer.call(name, command)
      EventManager.notify_command(usecs, name, command)
      result
     end
  end

  @doc "Execute a query for a named server"
  defmacro query(name, query, caching \\ true) do
    quote bind_quoted: [name: name, query: query], unquote: true do
      if unquote(caching) do
        cached(name, query) do
          GenServer.call(name, query)
	end
      else
        GenServer.call(name, query)
      end
    end
  end
  #...
  
end
{% endhighlight %}

The command macro wraps the GenServer call with a notification sent to the Event Manager of the command-as-event. The query macro wraps the GenServer call in another macro ('cached(name, query) do...end') that looks in the cache for the query result and, if not there, caches what the GenServer.call(...) returns.

I then modified my application service functions to explicitly issue commands or queries like this:

{% highlight elixir %}
defmodule SomeApplicationService do
  use GenServer
  use CqrsMacros
  #...
  
  def update_something(param1, param2) do
    command(@name, {:do_something, param1, param2})
  end

  def get_something(param1, param2) do
    query(@name, {:get_something, param1, param2})
  end

end
{% endhighlight %}

I then implemented the Event Manager function that receives the event command and dispatches to event handlers (I have only one event handler that does something with it; other event handlers ignore it)

The Event Manager processes command events as follows:

{% highlight elixir %}
defmodule EventManager do
  #...
  
  def notify_command(usecs, server_name, command) do
    GenEvent.notify(@name, {:command, usecs, server_name, command})
  end

  #...
end
{% endhighlight %}

The command event handler, registered with the EventManager, catches the command event:

{% highlight elixir %}
defmodule CommandEventHandler do
  #...
  def handle_event({:command, usecs, server_name, command}, {:changes_count, count}) do
    if count >=  DB.changes_to_backup() do
      BackupServer.dump()
      {:ok, {:changes_count, 0}}
    else
      BackupServer.journal(usecs, server_name, command)
      {:ok, {:changes_count, count + 1}}
    end
  end
  #...
end
{% endhighlight %}

I wrote a BackupServer (a GenServer) that synchronizes taking a database snapshot or journaling a command (only one at a time and in the order requested).

Now back to queries and how they are cached. I wrote a macro to wrap GenServer calls that I want cached (caching yielded sub-micro second REST API response times!)

{% highlight elixir %}
defmodule CqrsMacros do
  #...
  
  defmacro cached(cacheName, key, do: block) do
    quote bind_quoted: [cacheName: cacheName, key: key], unquote: true do
      cached_value = CacheServer.get_cached(cacheName, key)
      if cached_value != nil do
  	  cached_value
      else
	  value = unquote(block)
	  case value do
            nil -> value
            {:error, _} ->
	      value 
            _ ->	
              CacheServer.cache(cacheName, key, value)
              value
          end
      end
   end

   #...
 end
{% endhighlight %}

I implemented a simple CacheServer (another GenServer) that manages all cache accesses.

And that's it! It took me about a day's work. I must say that I am more impressed that ever by the power and elegance of Elixir and OTP.