# What's wrong with godoc? 

Like many modern languages, Golang has built-in inline documentation support tool called `godoc`. 

To be honest, it’s awesome. It is a really great tool, that has a real impact on the everyday coding process. At least if you use plugins with function call tips as I do. 

But there is the big problem of the majority Go-projects that is tightly related to `godoc` but lays outside of it. 

The `godoc` is really cool. You can document your methods and packages, paste a code snippet, that will have highlighting on the web page. Some IDE plugins will give you signature tips based on function documentation, e.g. description of parameters. 

Also, some code analyzers like `go vet` will warn you if you miss a package header or global type, function or variable description comment. It disciplines you and makes sources more readable. 

Only pros, huh? But there is a big con, that isn’t a fault of `godoc` itself, but many developers expect it to solve this problem. 

## The main idea escapes the attention 

The major disadvantage of `godoc` is its declarative nature. You can write excellent documentation with code snippets and parameter description. You can write simple use-cases for your methods, you can give a rich description of each global constant and variable. You can measure your documentation coverage and strive to beat 100%. Finally, you can write a big article in the package header as many open-source projects do, and you will have great documentation. 

But it will be a book. An encyclopedia. _Not the manual. Not the how-to guide._ 

There are many examples of encyclopedia-like documentation, even at big and popular projects – take a look at Docker or Kubernetes, for example. They both have really well-documented features. Each aspect has its own page on the documentation portal. They even have some examples with code snippets. But if you really want to use them, if you will try to understand some corner-cases or cross-module interaction, you will rather go to SO, then search through the docks. And that’s why. 

## You don’t need a book. You need a guide 

Last two decades the way of programming has changed radically. Instead of “hackermen” in sweaters, we have now stylish smoothie-boys, which look better than their CEOs on the investment pitch. Instead of memory leakage and Y2K problems, we argue which front-end framework is better, and performance is now a taboo subject (try to ask about performance at some front-end meetup). Modern tools and languages do the majority of the low-level stuff by their own and we can focus on more abstract tasks without the need to understand how each bit flows through the system. 

Sure, it’s better to know the low-level stuff. It’s much better to understand, how the compiler processes your code and when it can optimize the runtime. Nice to know the technical limitations of the tool you use, to understand what to expect. 

But after all, you need to invest an enormous amount of time to learn all that stuff. So much, that it probably will become obsolete before you will become a master in it. And that even can’t give you a guarantee, that a deep understanding of the tool basics will reward you as much, as much time you have spent to learn it. 

As an example, I use Docker in my applications a lot. I understand the basics of containerization and virtualization, but to be honest, I don’t know what _exactly_ goes on in the black box. Will I make better containers if I will meticulously read all the docks and learn each technical aspect, lying under the containerization principle? Sure. Will I make better containers if I will investigate the sources and try to understand or even debug each method. Hmm... Not sure. Probably I will become an expert in the Docker architecture and some interesting solutions, but that will not make you 10x more powerful in the container configuration. You will spend so much time to get almost nothing related to your actual tasks. 

That leads us to the following situation: you don’t need to know how the tool functions, but you need to know how to use it in your daily work. That means, that people will be interesting rather in real use-cases and specific problem solutions than a method documentation and option description. For many tools, Stack Overflow is much useful than the official documentation. 

In fact, you don’t need book-like documentation. It’s nice to have when you want to check some parameters or unclear behavior. But it shouldn’t be on the first place. What you really need is use-cases. The real examples of the app or tool usage from the first steps. 

I’ve seen many great tools, which are really difficult to use because you simply don’t understand where to start. They have a perfect description of each function or class, but you just missing the main idea. It’s like to talk with a person with schizophasia – you understand the words but can’t get the idea of the speech. 

The problem of `godoc` is that there is no place for such use-case-oriented documentation. You can’t just put a ton of examples into a package comment. At least, you can’t structure them. You need something that will give you a documentation hierarchy different from your package content tree. But the solution exists. 

## How to make your docs useful 

When a child learns to read, he doesn’t start with a class book. He starts with fairy tales. He reads funny stories about brave characters, but not definitions of the words. That’s why children can understand us even if they don’t know all the words we use. 

To let your users start with ease, don't give them an encyclopedia. Give them a story. Guide them from the beginning to the end. Or at least tell them where to start. 

Unfortunately, it isn’t so much possible with the `godoc`. If your project is small enough, you can do that in your `README` or in the package descriptions, but usually, it is not the case. 

So, what you can do? 

### Explicit entry point 

Often when you find a new package, you don’t want to do research. You ask yourself: “ok, does it fit my requirements, or I just need to search further?”. Don’t force the user to read all the docs. Help him to start using your tool right now. 

Can you run the simplest option with only two lines of code? Great, let’s start with that and then add new functions one-by-one. 

Can you run a bare module with different parameters? No more words! Let’s do that and then give the user more and more parameters with a description of how they can help to fulfill his needs. 

This approach is very useful. In one hand, you will help newcomers to begin using your product with ease. In the other hand, you will gather a bigger and more involved community. 

### Real use-cases and tips 

As I mentioned above, in many cases you don’t need a detailed description of the tool to solve your problem. You need just a solution to your cases. Sometimes you even not interested in the tool or library description at all, you just want it to work. 

And that’s what people will be looking for in your documentation. They don’t need a detailed description of every function and every parameter, they need a way to bring it to life from scratch.

So just give them what they need. Describe the main idea of your package and the use-cases. Write from the point of view of usage, not functionality. Let the user read it as a story, chapter by chapter, but avoid formal encyclopedia structure. 

To do all of this you need something more flexible, then a `README` file. 

### Hierarchical documentation 

To navigate resulting “story of tool X usage” with ease you need a flexible structure capable to serve both as a table of contents and as a roadmap. Here you need something like hierarchical navigation/content tree. Maybe there are other good solutions, but I find a hierarchical view most suitable for this task. There are several ways to do that without extra work: 

- Built-in Wiki on GitHub, Bitbucket or same hosting – they are easy to use, support markdown and requires nothing but a repository that you already have; 
- [Read the Docs](https://readthedocs.org/) - this service is a de-facto standard for documentation. It supports versioning, so you can have documentation for each version in parallel, that can be very helpful if you want to support previous versions; 
- Directory with documents - you can simply create a directory with a set of markdown documents and add a table of contents in you `README.md` or same document in the root directory. The same can be done in a single document using anchors – choice of style is up to you. Check this as a [nice example](https://github.com/SAP/styleguides/blob/master/clean-abap/CleanABAP.md). 

There are many other options, you can even use your `godoc` to organize something like this, but I think, better to use each tool for its main purpose. 

## So, what about godoc? 

After all, `godoc` is still a very cool tool to write functional documentation. I really respect people who write rich and clear code documentation using built-in tools like this. 

Don’t forget to use it. Start your documentation with the `godoc` and then write more complex and more usage-oriented user manuals. As a really nice example, I can give a [go-pg](https://github.com/go-pg/pg) library, that has both good inline documentation and also a Wiki with examples and use-cases. 

It may be difficult sometimes to write a good manual for a package, so start with simple Q&A page, and fill it from SO and other recourses. There you will see, what causes difficulties for users and what could be misunderstood.  Do this work iteratively and listen to your community, and you will be good.