# Clean Configuration Management in Golang

There are many good approaches to handle configuration in a modern application. Now we use such things as configuration files, environment variables, command-line parameters, as well as CI configuration patterns and on-fly config file builds, remote config servers, specific mapping, and binding services and even more complex things.

But the target is the same - provide the app with a configuration, which is fast to get and easy to use. But how to do that?

## How our applications consume configurations

As a nice developer, I always fight with implicitly. I don’t use global variables, I structure my code the readable way and may my code flow certain and clear. There is something I really don’t like - environment variables because they are the same bad as global variables. But you can handle them the right way too if you will read them once at the beginning.

So the main idea is to get the application settings from where they are (file, env, remote server, etc.) and distribute them through the app, to make them available wherever they are required. Simple task, huh? But there are still so many different ways to do so. 

## Many approaches, even more tools

So it’s pretty clear how to get configs. Each approach has it’s own best practices. But what to do next? There is no common answer.

Some libraries (like [viper](https://github.com/spf13/viper) store configs as a key-value set in a global area. Pretty useful, but very implicit. Within big projects sometimes you can just lose the way how one or another variable came to this state.

Other libraries put everything in one bucket - environment, files, command-line options. Even there is a nice-looking structure with all config values, that still looks unclear and useless to me. Normally, the app uses either a command-line approach (CLI tools) or config file + environment approach (web services, containerized apps, etc.).

There are a lot of really good libraries for command-line argument handling like [go-flags](https://github.com/jessevdk/go-flags), and I don’t think that’s there is a reason to put everything in one single tool.

But both approaches have the same issue - a lot of magic inside.

## Magic

Go isn’t a language for everyone. It has a lot of boilerplate, the type system isn’t so much forgivable, and OOP possibilities are very limited. But there is a nice thing that worth it - it is instantly readable, it has no magic inside. And I like that.

But work with configs is often tricky, in turn. Even if you define a structure instead of using an unsafe map, you still have to declare which environment variables do you use. There is no worst thing as globally used environment variables. But even if the structure contains the explicit mapping of each environment variable, it is still hard to set up the environment outside the app. Someone (it may be you, another developer, DevOps, or just someone forced to do that) needs to find a list of environment variables used, and then use them properly.

Okay, they can be listed in the documentation or in the source code. But both are bad. Documentation can be outdated, and source code needs time and skill to read.

In my view, the only proper way to do that - add a full list of environment variables into a help output, including descriptions, default values and other meaningful information.

I haven’t found anything not overcomplicated but still useful, with explicit configuration setup and clean and informative help output, so I’ve built it on my own.

## One way to rule them all the clean way

So, I’ve analyzed many popular configuration management libraries and decided to build my own. Why? There a couple of reasons:

handle config files and environment variables, not command-line parameters;
no magic, explicit way to read and use configuration;
no implicit names, all mapping should be done using tags;
the nice and informative help output;
easy integration with other libraries.

And here we are: [cleanenv](https://github.com/ilyakaznacheev/cleanenv) library is now released and proven in production.

Let’s talk about its major features.

### Explicit configuration structure

The main idea behind the cleanenv library is to make everything explicit and clean. No global state, no encapsulated maps, no magical background updates nether any other hidden work.

Everything goes clean and visible. And that’s why a structured format was chosen as a base for configuration. The structure (as deep and complex as you want) being used to parse configuration files and environment variables. It’s also a good place to write config documentation.

### Explicit naming and options

The next “no-magic” step is the explicitness of variable names. There are no such things like complex environment name generators based on nested structure names - you have to set names as-is, so you can easily find them using a search.

For file parsing, we are using the same approach as corresponding libraries - JSON, YAML, TOML, etc.

Here is an example of a simple server configuration structure:

```go
type ConfigDatabase struct {
    Port     string `yml:"port" env:"PORT" env-default:"5432"`
    Host     string `yml:"host" env:"HOST" env-default:"localhost"`
    Name     string `yml:"name" env:"NAME" env-default:"postgres"`
    User     string `yml:"user" env:"USER" env-default:"user"`
    Password string `yml:"password" env:"PASSWORD"`
}
```

This structure can be used to parse YAML configuration file, and then read some data from the environment. First, it will read the file, and then try to find environment variables using names in `env` tags. If there were no data found in file neither in the environment, the constant from the `env-default` will be used instead.

Supported data types are:

- integers;
- floating-point numbers;
- strings;
- booleans;
- arrays (with customizable separator);
- maps (with customizable separator);

### Readable help output

Modern containerizable applications use environment variables as the main configuration. So, the app can have up to hundreds of variables it depends on. The common problem is that the exact list of variables is often uncertain or outdated (or even worth, they are distributed through the app and being read in some unexpectable places).

To fix that the cleanenv library contains the possibility to add a well-structured list of environment variables with descriptions into help output:

```go
import github.com/ilyakaznacheev/cleanenv

type ConfigServer struct {
    Port     string `env:"PORT" env-description:"server port"`
    Host     string `env:"HOST" env-description:"server host"`
}

var cfg ConfigRemote

help, err := cleanenv.GetDescription(&cfg, nil)
if err != nil {
    ...
}
```

You will get the following:

```
Environment variables:
  PORT  server port
  HOST  server host
```

It will help you to get the documentation synchronized with your app with no need to have extra files.

### Integration

Simplicity is everything. But I want not only to give a simple tool but also to make it easy to use with other codes.

Now you can easily combine it with `flag` library help function:

```go
type config struct {
	Port     string `env:"PORT" env-description:"server port" env-default:"5432"`
	Host     string `env:"HOST" env-description:"server host" env-default:"localhost"`
	Name     string `env:"NAME" env-description:"server name" env-default:"postgres"`
	User     string `env:"USER" env-description:"server username" env-default:"user"`
	Password string `env:"PASSWORD" env-description:"server password"`
}

var (
	cfg     config
	cfgPath string
)

fset := flag.NewFlagSet("My app", flag.ContinueOnError)
fset.StringVar(&cfgPath, "cfg", "", "path to config file")

fset.Usage = cleanenv.FUsage(fset.Output(), &cfg, nil, fset.Usage)

fset.Parse(os.Args[1:])
```

If you will run `go run your_app.go -h`, the output will be:

```
& go run your_app.go -h
Usage of My app:
  -cfg string
    	path to config file

Environment variables:
  PORT string
    	server port (default "5432")
  HOST string
    	server host (default "localhost")
  NAME string
    	server name (default "postgres")
  USER string
    	server username (default "user")
  PASSWORD string
    	server password
```


### Possibility to do more

That’s nice, but you may need more. For example, you may want to get configuration from a remote server, or some other tool, or update them. To do so, you can use the enhancement possibilities of the library. There are some ways to write your own logic to read the data from your own source. Read more in [documentation](https://github.com/ilyakaznacheev/cleanenv#custom-functions).

## Conclusion

So the [cleanenv](https://github.com/ilyakaznacheev/cleanenv) library is not the key for every lock. It is definitely not a tool-for-everything. But it is designed to do a simple, clean and readable, but flexible enough if you need so.

So, I’m happy if it will help you. I’m actively using it in my projects, so it is productive-proved.

Also, feel free to request any features you think may be helpful.

And stay clean!