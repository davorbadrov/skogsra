# Skogsrå

[![Build Status](https://travis-ci.org/gmtprime/skogsra.svg?branch=master)](https://travis-ci.org/gmtprime/skogsra) [![Hex pm](http://img.shields.io/hexpm/v/skogsra.svg?style=flat)](https://hex.pm/packages/skogsra) [![hex.pm downloads](https://img.shields.io/hexpm/dt/skogsra.svg?style=flat)](https://hex.pm/packages/skogsra)

> The _Skogsrå_ was a mythical creature of the forest that appears in the form
> of a small, beautiful woman with a seemingly friendly temperament. However,
> those who are enticed into following her into the forest are never seen
> again.

This library attempts to improve the use of OS environment variables for
application configuration:

* Automatic type casting of values.
* Variable defaults.
* Automatic documentation generation for variables.
* Runtime reloading.
* Setting variable's values at runtime.
* Fast cached values access by using `:persistent_term` as temporal storage.
* YAML configuration provider for Elixir releases.

## Small Example

You would create a settings module e.g:

```elixir
defmodule MyApp.Settings do
  use Skogsra

  @envdoc "My hostname"
  app_env :my_hostname, :myapp, :hostname,
    default: "localhost"
end
```

Calling `MyApp.Settings.my_hostname()` will retrieve the value for the
hostname in the following order:

1. From the OS environment variable `$MYAPP_HOSTNAME`.
2. From the configuration file e.g:
   ```
   config :myapp,
     hostname: "my.custom.host"
   ```
3. From the default value if it exists (In this case, it would return
`"localhost"`).

## Handling different environments

If it's necessary to keep several environments, it's possible to use a
`namespace` e.g:

Calling `MyApp.Settings.my_hostname(Test)` will retrieve the value for the
hostname in the following order:

1. From the OS environment variable `$TEST_MYAPP_HOSTNAME`.
2. From the configuration file e.g:
   ```
   config :myapp, Test,
     hostname: "my.custom.test.host"
   ```
3. From the OS environment variable `$MYAPP_HOSTNAME`.
4. From the configuraton file e.g:
   ```
   config :myapp,
     hostname: "my.custom.host"
   ```
5. From the default value if it exists.

## Required variables

It is possible to set a environment variable as required with the `required`
option e.g:

```elixir
defmodule MyApp.Settings do
  use Skogsra

  @envdoc "My port"
  app_env :my_port, :myapp, :port,
    required: true
end
```

If the variable `$MYAPP_PORT` is undefined and the configuration is missing,
calling to `MyApp.Settings.my_port()` will return an error tuple. Calling
`$MyApp.Settings.my_port!()` (with the bang) will raise a runtime
exception.

## Automatic casting

If the default value is set, the variable value will be casted as the same type
of the default value. Otherwise, it is possible to set the type for the
variable with the option `type`. The available types are:
 - `:binary` (default).
 - `:integer`.
 - `:float`.
 - `:boolean`.
 - `:atom`.
 - `:module` (modules loaded in the system).
 - `:unsafe_module` (modules that might or might not be loaded in the system)

> **Note:** `:module` and `:unsafe_module` types cannot be derived from the
> default value type, so they need to be explicitly specified.

Additionally, you can create a `Skogsra.Type` e.g. a naive implementation for
casting `"1, 2, 3, 4"` to `[1, 2, 3, 4]` would be:

```elixir
defmodule MyList do
  use Skogsra.Type

  def cast(value) when is_binary(value) do
    list =
      value
      |> String.split(~r/,/)
      |> Stream.map(&String.trim/1)
      |> Enum.map(String.to_integer/1)
    {:ok, list}
  end

  def cast(_) do
    :error
  end
end
```

If then we define the following enviroment variable with `Skogsra`:

```elixir
defmodule Settings do
  use Skogsra

  app_env :my_integers, :myapp, :my_integers, type: MyList
end
```

We can then set `$MYAPP_INTEGERS`'s value as `"1, 2, 3"` and it'll be casted
casted to `[1, 2, 3]` when we call `Settings.my_integers/1` function e.g:

```elixir
iex> Settings.my_integers()
{:ok, [1, 2, 3]}
```

## Setting and reloading variables

It's possible to set a value for the variable at runtime with e.g.
`MyApp.Settings.put_my_hostname("my.other.hostname")`.

Also, for debugging purposes is possible to reload variables at runtime with
e.g. `MyApp.Settings.reload_my_hostname()`.

## Using with `Hab`

[_Hab_](https://github.com/alexdesousa/hab) is an
[Oh My ZSH](https://github.com/robbyrussell/oh-my-zsh) plugin for loading OS
environment variables automatically.

By default, `Hab` will try to load `.envrc` file, but it's possible to have
several of those files for different purposes e.g:

- `.envrc.prod` for production OS variables.
- `.envrc.test` for testing OS variables.
- `.envrc` for development variables.

`Hab` will load the development variables by default, but it can load the
other files using the command `hab_load <extension>` e.g. loading
`.envrc.prod` would be as follows:

```bash
~/my_project $ hab_load prod
[SUCCESS]  Loaded hab [/home/user/my_project/.envrc.prod]
```

## YAML Config Provider

`Skogsra` includes a simple YAML configuration provider compatible with
`mix release` for Elixir ≥ 1.9.

The following is the supported configuration format:

```yaml
# file: /etc/my_app/config.yml
- app: "my_app"              # Name of the application.
  module: "MyApp.Repo"       # Optional module/namespace.
  config:                    # Actual configuration.
    - database: "my_app_db"
      username: "postgres"
      password: "postgres"
      hostname: "localhost"
      port: 5432
```

The previous configuration file would translate to:

```elixir
config :my_app, MyApp.Repo,
  database: "my_App_db",
  username: "postgres"
  password: "postgres"
  hostname: "localhost"
  port: 5432
```

For using this config provider, just add the following to your release
configuration:

```elixir
config_providers: [{Skogsra.Provider.Yaml, ["/path/to/config/file.yml"]}]
```

> **Note**: If the `module` you're using in you're config does not exist, then
> change it to `namespace` e.g: `namespace: "MyApp.Repo"`. Otherwise, it will
> fail loading it.

## Installation

The package can be installed by adding `skogsra` to your list of dependencies
in `mix.exs`.

- For Elixir ≥ 1.8 and Erlang ≥ 22

  ```elixir
  def deps do
    [{:skogsra, "~> 2.0"}]
  end
  ```

- For Elixir ≥ 1.9, Erlang ≥ 22 and YAML config provider support:

  ```elixir
  def deps do
    [
      {:skogsra, "~> 2.0"},
      {:yamerl, "~> 0.7"}
    ]
  end
  ```

## Author

Alexander de Sousa.

## License

_Skogsrå_ is released under the MIT License. See the LICENSE file for further
details.
