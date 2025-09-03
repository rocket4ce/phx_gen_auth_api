This is a web application written using the Phoenix web framework.

## Elixir guidelines

- Elixir lists **do not support index based access via the access syntax**

  **Never do this (invalid)**:

      i = 0
      mylist = ["blue", "green"]
      mylist[i]

  Instead, **always** use `Enum.at`, pattern matching, or `List` for index based list access, ie:

      i = 0
      mylist = ["blue", "green"]
      Enum.at(mylist, i)

- Elixir supports `if/else` but **does NOT support `if/else if` or `if/elsif`. **Never use `else if` or `elseif` in Elixir**, **always** use `cond` or `case` for multiple conditionals.

  **Never do this (invalid)**:

      <%= if condition do %>
        ...
      <% else if other_condition %>
        ...
      <% end %>

  Instead **always** do this:

      <%= cond do %>
        <% condition -> %>
          ...
        <% condition2 -> %>
          ...
        <% true -> %>
          ...
      <% end %>

- Elixir variables are immutable, but can be rebound, so for block expressions like `if`, `case`, `cond`, etc
  you *must* bind the result of the expression to a variable if you want to use it and you CANNOT rebind the result inside the expression, ie:

      # INVALID: we are rebinding inside the `if` and the result never gets assigned
      if connected?(socket) do
        socket = assign(socket, :val, val)
      end

      # VALID: we rebind the result of the `if` to a new variable
      socket =
        if connected?(socket) do
          assign(socket, :val, val)
        end

- Use `with` for chaining operations that return `{:ok, _}` or `{:error, _}`
- **Never** nest multiple modules in the same file as it can cause cyclic dependencies and compilation errors
- **Never** use map access syntax (`changeset[:field]`) on structs as they do not implement the Access behaviour by default. For regular structs, you **must** access the fields directly, such as `my_struct.field` or use higher level APIs that are available on the struct if they exist, `Ecto.Changeset.get_field/2` for changesets
- Elixir's standard library has everything necessary for date and time manipulation. Familiarize yourself with the common `Time`, `Date`, `DateTime`, and `Calendar` interfaces by accessing their documentation as necessary. **Never** install additional dependencies unless asked or for date/time parsing (which you can use the `date_time_parser` package)
- Don't use `String.to_atom/1` on user input (memory leak risk)
- Predicate function names should not start with `is_` and should end in a question mark. Names like `is_thing` should be reserved for guards
- Elixir's builtin OTP primitives like `DynamicSupervisor` and `Registry`, require names in the child spec, such as `{DynamicSupervisor, name: MyApp.MyDynamicSup}`, then you can use `DynamicSupervisor.start_child(MyApp.MyDynamicSup, child_spec)`
- Use `Task.async_stream(collection, callback, options)` for concurrent enumeration with back-pressure. The majority of times you will want to pass `timeout: :infinity` as option

## Mix guidelines

- Read the docs and options before using tasks (by using `mix help task_name`)
- To debug test failures, run tests in a specific file with `mix test test/my_test.exs` or run all previously failed tests with `mix test --failed`
- `mix deps.clean --all` is **almost never needed**. **Avoid** using it unless you have good reason

# Writing Generators

In `Igniter`, generators are done as a wrapper around `Mix.Task`, allowing them to be called individually or composed as part of a task.

Since an example is worth a thousand words, lets take a look at an example that generates a file and ensures a configuration is set in the user's `config.exs`.

> ### An igniter for igniters?! {: .info}
>
> Run `mix igniter.gen.task your_app.task.name` to generate a new, fully configured igniter task!

```elixir
# lib/mix/tasks/your_lib.gen.your_thing.ex
defmodule Mix.Tasks.YourLib.Gen.YourThing do
  use Igniter.Mix.Task

  @impl Igniter.Mix.Task
  def igniter(igniter) do
    [module_name | _] = igniter.args.argv

    module_name = Igniter.Code.Module.parse(module_name)
    path = Igniter.Code.Module.proper_location(module_name)
    app_name = Igniter.Project.Application.app_name(igniter)

    igniter
    |> Igniter.create_new_elixir_file(path, """
    defmodule #{inspect(module_name)} do
      use YourLib.Thing

      ...some_code
    end
    """)
    |> Igniter.Project.Config.configure(
      "config.exs",
      app_name,
      [:list_of_things],
      [module_name],
      updater: &Igniter.Code.List.prepend_new_to_list(&1, module_name)
    )
  end
end
```

Now, your users can run

`mix your_lib.gen.your_thing MyApp.MyModuleName`

and it will present them with a diff, creating a new file and updating their `config.exs`.

Additionally, other generators can "compose" this generator using `Igniter.compose_task/3`

```elixir
igniter
|> Igniter.compose_task(Mix.Tasks.YourLib.Gen.YourThing, ["MyApp.MyModuleName"])
|> Igniter.compose_task(Mix.Tasks.YourLib.Gen.YourThing, ["MyApp.SomeOtherName"])
```

## Writing a library installer

Igniter will look for a mix task called `your_library.install` when a user runs `mix igniter.install your_library`. As long as it has the correct name, it will be run automatically as part of installation!

## Task Groups

Igniter allows for _composing_ tasks, which means that many igniter tasks can be run in tandem. This happens automatically when using `mix igniter.install`, for example:
`mix igniter.install package1 package2`. You can also do this manually by using `Igniter.compose_task/3`. See the example above.

However, composing tasks means that sometimes a flag from one task may conflict with a flag from another task. Igniter will alert users when this happens, and ask them to
prefix the option with your task name. For example, the user may see an error like this:

```sh
Ambiguous flag provided `--option`.

The task or task groups `package1, package2` all define the flag `--option`.

To disambiguate, provide the arg as `--<prefix>.option`,
where `<prefix>` is the task or task group name.

For example:

`--package1.option`
```

It is not possible to prevent this from happening for all combinations of invocations of your task, but you can help by using a `group`.

```elixir
%Igniter.Mix.Task.Info{
  group: :your_package,
  ...
}
```

Setting this group performs two functions:

1. any tasks that share a group with each other will be assumed that the same flag has the same meaning. That way,
   users don't have to disambiguate when calling `mix igniter.install yourthing1 yourthing2 --option`, because it is assumed
   to have the same meaning.
2. it can provide a shorter/semantic name to type, i.e instead of `--ash-authentication-phoenix.install.domain` it could be just `--ash.domain`.

By default the group name is the _full task name_. We suggest setting a group for all of your tasks.
You should _not_ use a group name that is used by someone else, just like you should not use a module prefix used by someone else in general.

## Navigating the Igniter Codebase

A large part of writing generators with igniter is leveraging our built-in suite of tools for working with zippers and AST, as well as our off-the-shelf patchers for making project modifications. The codebase is split up into four primary divisions:

- `Igniter.Project.*` - project-level, off-the-shelf patchers
- `Igniter.Code.*` - working with zippers and manipulating source code
- `Igniter.Mix.*` - mix tasks, tools for writing igniter mix tasks
- `Igniter.Util.*` - various utilities and helpers