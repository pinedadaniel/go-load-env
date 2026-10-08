# go-load-env

[![Go Reference](https://pkg.go.dev/badge/github.com/pinedadaniel/go-load-env/pkg/env.svg)](https://pkg.go.dev/github.com/pinedadaniel/go-load-env/pkg/env)

`go-load-env` is a Go library for loading environment-variable values into structs. It supports defaults, validation tags, nested configuration, slices and maps, custom parsers, and parsing from an explicit environment map.

## Requirements

- Go 1.27.1 or later

## Installation

```bash
go get github.com/pinedadaniel/go-load-env/pkg/env
```

Import the package in your application:

```go
import "github.com/pinedadaniel/go-load-env/pkg/env"
```

## Quick start

Declare a configuration struct with `env` tags, then call `env.Parse`:

```go
package main

import (
	"fmt"
	"log"

	"github.com/pinedadaniel/go-load-env/pkg/env"
)

type Config struct {
	AppName string `env:"APP_NAME" envDefault:"my-service"`
	Port    int    `env:"PORT" envDefault:"8080"`
	Debug   bool   `env:"DEBUG" envDefault:"false"`
}

func main() {
	var cfg Config
	if err := env.Parse(&cfg); err != nil {
		log.Fatal(err)
	}

	fmt.Printf("%s listening on port %d (debug=%t)\n",
		cfg.AppName, cfg.Port, cfg.Debug)
}
```

Set the variables before running the application:

```bash
APP_NAME=payments PORT=9000 DEBUG=true go run .
```

On PowerShell:

```powershell
$env:APP_NAME = "payments"
$env:PORT = "9000"
$env:DEBUG = "true"
go run .
```

`Parse` reads the process environment. It does **not** load a `.env` file automatically; load such a file separately if your application needs that behavior.

## Parsing APIs

Use the pointer-based API when you already have a config value:

```go
var cfg Config
if err := env.Parse(&cfg); err != nil {
	return err
}
```

Or create and parse a value in one call with Go generics:

```go
cfg, err := env.ParseAs[Config]()
if err != nil {
	return err
}
```

Both APIs have `WithOptions` variants for custom environment maps and parser behavior:

```go
opts := env.Options{
	Environment: map[string]string{
		"APP_NAME": "payments",
		"PORT":     "9000",
		"DEBUG":    "true",
	},
}

cfg, err := env.ParseAsWithOptions[Config](opts)
if err != nil {
	return err
}
```

`ParseWithOptions` and `ParseAsWithOptions` use the provided `Environment` map in place of the process environment. To build a map from the format returned by `os.Environ()`, use `env.ToMap`:

```go
import "os"

opts := env.Options{Environment: env.ToMap(os.Environ())}
```

## Struct tags

The default tag name is `env`. A field's tag specifies its environment-variable name and optional comma-separated options:

```go
type Config struct {
	APIKey string `env:"API_KEY,required"`
	Region string `env:"REGION" envDefault:"us-east-1"`
	Secret string `env:"SECRET,file"`
}
```

| Tag or option | Description |
| --- | --- |
| `env:"NAME"` | Reads the value of `NAME`. |
| `env:"-"` or `env:"NAME,-"` | Skips the field. |
| `envDefault:"value"` | Uses the given value when the variable is unset or empty. |
| `required` | Returns an error if the named variable is not set. |
| `notEmpty` | Returns an error if the resolved value is empty. |
| `file` | Treats the value as a file path and loads the file contents. |
| `expand` | Expands `$VAR` and `${VAR}` references in the value using the configured environment. |
| `unset` | Unsets the named variable from the process environment after reading it. |
| `init` | Initializes a nil pointer to a nested struct before parsing its fields. |

For example, `env:"API_KEY,required"` makes `API_KEY` mandatory. You can also make fields required by default with `Options.RequiredIfNoDef`; a field's `envDefault` value satisfies the missing-variable check when the variable is unset.

## Nested configuration and prefixes

Use `envPrefix` on a nested struct to group its variables:

```go
type Config struct {
	Database struct {
		Host string `env:"HOST" envDefault:"localhost"`
		Port int    `env:"PORT" envDefault:"5432"`
	} `envPrefix:"DB_"`
}
```

This struct reads `DB_HOST` and `DB_PORT`. To add a prefix to every key, set `Options.Prefix`. If you prefer to derive names from field names when an `env` tag is omitted, set `Options.UseFieldNameByDefault` (for example, `ServerPort` becomes `SERVER_PORT`).

## Supported values

Built-in parsers handle strings, booleans, integer types, and floating-point types. The package also supports `time.Duration`, `time.Location`, and `net/url.URL`, types implementing `encoding.TextUnmarshaler`, slices, and maps.

By default, slice values are comma-separated. Use `envSeparator` to choose another separator:

```go
type Config struct {
	AllowedOrigins []string `env:"ALLOWED_ORIGINS" envSeparator:";"`
}
```

Map values use comma-separated `key:value` pairs by default. `envSeparator` changes the pair separator, and `envKeyValSeparator` changes the separator between each key and value:

```go
type Config struct {
	Labels map[string]string `env:"LABELS" envSeparator:";" envKeyValSeparator:"="`
}
```

For `LABELS=team=platform;region=west`, `Labels` contains `team: platform` and `region: west`.

## Custom parsers and options

`Options.FuncMap` lets you register parsers by Go type. Custom parsers are merged with the built-in parsers:

```go
type LogLevel string

opts := env.Options{
	FuncMap: map[reflect.Type]env.ParserFunc{
		reflect.TypeOf(LogLevel("")): func(value string) (interface{}, error) {
			switch value {
			case "debug", "info", "warn", "error":
				return LogLevel(value), nil
			default:
				return nil, fmt.Errorf("unsupported log level %q", value)
			}
		},
	},
}
```

Other `Options` fields let you customize the environment map, tag names, global prefix, default handling, and an `OnSet` callback. `ParseWithOptions` and `ParseAsWithOptions` apply these settings.

## Inspecting fields

Use `GetFieldParams` or `GetFieldParamsWithOptions` to inspect the environment keys and tag settings associated with a struct:

```go
params, err := env.GetFieldParams(&Config{})
if err != nil {
	return err
}
for _, param := range params {
	fmt.Println(param.Key, param.Required, param.DefaultValue)
}
```

## Errors

Parsing returns an error when the input is not a pointer to a struct, a required variable is missing, a value cannot be parsed, or a field has no supported parser. Parse errors may be aggregated when multiple fields fail. Use `errors.Is` and `errors.As` to inspect the exported error types when needed.

## License

This project is distributed under the [MIT License](LICENSE).
