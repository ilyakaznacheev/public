# Clean way to pass configs in a Go application  

There are many concurrent approaches to organize application configuration nowadays.  

Classic `.ini` files, `.json`, `.toml`, `.yaml`, configs, modern `.env` and, of course, container environment. And don’t forget about CLI arguments! Am I missing something? 

Let me be honest, I really dislike any implicitness in interfaces. Same for the CLI of course. Any of your interfaces, whether public or internal, API or object interface, a class method or module facade – they have to cooperate fair. The contract between you and the other side should be explicit, rightful and without any notes in small text at the bottom of the page.

That means, that they have to take exactly what they are asking you for and give you exactly what they promise. They shouldn’t take anything from your pocket when you turn away. They shouldn’t send your stuff after a week or two if you agreed on an immediate transaction. They can hold the inner state. Not all of them should be immutable. But you always should be able to interact clearly and fair, that means all interactions have to be visible and legitimate, i.e. you will be able to pass the data from hand to hand. Otherwise, you will lose the contract details and sooner or later you will be fooled.

Any interface also has to have a single entry point, that means you pass the whole amount of data that the interface requires at once, by passing it explicitly. It can be any type of data – values, structured or serialized data, path to file to parse or a socket to connect – the main idea is to put the data at the interface call.

## A way to clean configuration

So the modern trend to move any meaningful settings into environment arguments according to [twelve-factor app](https://12factor.net/) looks ugly to me. No, the ideas of the twelve-factor app are sweet. But the fact that I’m compelled to pass my configuration implicitly with environment variable setup (i.e. with global variables) makes me sad.

So sad, that at the beginning I’ve put all my configuration at the single config file and made a separate config file for each environment (e.g. local, stage, test, prod, etc.). Normally config files are well structured, so I can group parameters depending on their use. But it was not so useful, because I needed to store several configs in the repo for each environment and update each one after configuration structure change.

Then I’ve moved to combine config with CLI arguments, because I have to pass some data directly to the app, e.g. secrets, local paths, and other variable data and the data that I can’t keep in the config file. It was better because now I can pass some values that vary in different environments as a parameter and use the same configuration file for each system. And it worked pretty well, until...

Containers come. And the common way to setup a container environment is to use environment variables. It’s possible, in theory, to store config files in secrets, but it isn’t useful at all.

The main disadvantage of environment variables is their implicit nature. You can’t just look at the application `help` output, or check a configuration file structure to see a variable set and structure. When your application config is based on environment variables, you should have documentation for them, and you’re getting the burden of its update and synchronization. Otherwise, the user will not be allowed to get config information, and his experience with the app will be really terrible.  

It’s also hard to debug such apps. As globals, environment variables act implicitly and can affect the program behavior at any moment. It’s not a big deal when your code is small enough to fit into your memory, but when it grows, you will struggle to try to remember where you get one or another config value.

But after all I have found approach, that looks workable to me and satisfies my requirements. It combines several techniques, so let me make a quick overview.

## Configuration file

First, we still use a configuration file. I use it for several purposes:

- declare configuration structure;
- keep defaults for each variable;
- document variables or sections;
- provide a config example for the app users;

I prefer to store my configs in YAML files. I don’t want to start a holy war, but just will declare why is it the best for me: 

- hierarchical structure - I can be as flexible in a variable organization as I want;
- clean markup - I don’t need any brackets or a conglomeration of symbols to speak with the parser;
- comments - I can provide a detailed documentation, options, limitations, examples, best practices, etc.;  
- rich semantic - really advanced techniques like anchors, aliases, extensions, embedding, and others. I don’t use them really often, but sometimes they are extremely helpful;

Let’s say we have a simple config file like this:

```yaml
# Server configurations
server:
  host: "localhost"
  port: 8000

# Database credentials
database:
  user: "admin"
  pass: "super-pedro-1980"
```

To use data from a `.yml` file in Go you need to unmarshal it into the structure like you do for JSON. 

The mapping looks similar to JSON:

```go
type Config struct {
    Server struct {
        Port string `yaml:"port"`
        Host string `yaml:"host"`
    } `yaml:"server"`
    Database struct {
        Username string `yaml:"user"`
        Password string `yaml:"pass"`
    } `yaml:"database"`
}
```

I use ` gopkg.in/yaml.v2` library by Cannonical to parse YAML files.

You can use `yml. Unmarshal` to parse a byte slice, but In most cases, you will work with some data provided as `io.Reader` implementation, so I/m using decoder that reads byte stream instead of full data stored in the memory:

```go
f, err := os.Open("config.yml")
if err != nil {
    processError(err)
}

var cfg Config
decoder := yaml.NewDecoder(f)
err = decoder.Decode(&cfg)
if err != nil {
    processError(err)
}
```

And that’s it. Just like any JSON file. Now you can go ahead and write your own well-documented and well-structured config file, that will also serve as an example in your repository. 

## Environment variables

Now let’s talk about the environment variables. Usually, they scattered through the app. But there is another approach. In Go, you can assign environment variables to structure fields as well you do that for JSON, YAML, and others. It will look like this:

```go
type Config struct {
    Server struct {
        Port string `envconfig:"SERVER_PORT"`
        Host string `envconfig:"SERVER_HOST"`
    }
    Database struct {
        Username string `envconfig:"DB_USERNAME"`
        Password string `envconfig:"DB_PASSWORD"`
    }
}
```

The magic is in the `github.com/kelseyhightower/envconfig` library.

In a couple of lines, it retrieves environment variables and assigns them to structure fields you have defined:

```go
var cfg Config
err := envconfig.Process("", &cfg)
if err != nil {
    processError(err)
}
``` 

And that’s it. Now you have all your environment variables in the same place. No need to browse through all the repo to find where you use this variable, you can simply track down the config structure, that has a single entry-point.  

Also, now you have all your environment variables declared in the same place. You can just open the config structure and see a full list of envs that the app needs. The lib provides a set of `Usage` functions with wide output possibilities, that will allow you to add the list of env. variables to help output or wherever you want. Your application’s user will appreciate that.  

Now you can use your favorite way to provide the environment variables - .env file, container settings, makefile or just a shell script. It’s up to you.

## All together

So now let's mix them! 

The same structure will look like this:

```go
type Config struct {
    Server struct {
        Port string `yaml:"port", envconfig:"SERVER_PORT"`
        Host string `yaml:"host", envconfig:"SERVER_HOST"`
    } `yaml:"server"`
    Database struct {
        Username string `yaml:"user", envconfig:"DB_USERNAME"`
        Password string `yaml:"pass", envconfig:"DB_PASSWORD"`
    } `yaml:"database"`
} 
```

So first I load the data from the YAML file. It also serves me as a set of default values.

Then I load envs and overwrite filled fields. That is, you don’t need to take care of missing values, they will be filled from the config file.

```go
func main() {
    var cfg Config
    readFile(&cfg)
    readEnv(&cfg)
    fmt.Printf("%+v", cfg)
}

func processError(err error) {
    fmt.Println(err)
    os.Exit(2)
}

func readFile(cfg *Config) {
    f, err := os.Open("config.yml")
    if err != nil {
        processError(err)
    }

    decoder := yaml.NewDecoder(f)
    err = decoder.Decode(cfg)
    if err != nil {
        processError(err)
    }
} 

func readEnv(cfg *Config) { 
    err := envconfig.Process("", cfg) 
    if err != nil { 
        processError(err)
    }
}
``` 

So, there are many advantages to this approach:  

- single entry point - easy to find where each config variable came from;  
- simple declaration via tags;  
- structured configuration - you can group config sections depending on their usage;  
- declare defaults in a config file, overwrite what you need in each environment;  
- explicit list of environment variables used in the app;  

I’m not sure, that this approach covers any possible usage scenario, but it’s pretty useful and most important explicit.