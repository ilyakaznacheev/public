# Setup, build, automate

## deploy a dockerized app to Heroku fast! 

Heroku is a beautiful service. Although it doesn’t support Kubernetes or give a so bright range of cloud infrastructure options, it still does a great job by hosting small applications (even for free). 

But the remote cloud configuration may be painful. It’s easy and nice until something goes wrong. It’s ok to dive into manuals and documentation, spend hours to understand how things work and so on. You are a developer after all.  

But sometimes you don’t want to know that. You just want to deploy you fizz-buzz blockchain service and bring it to live. And you may not want to spend time on configurations and troubleshooting, don’t want to set up a ton of build dependencies fitting to a cloud environment. You just want to let some magic happen and get a `200` status code in your service response. So, let me tell about how to do that fast. 

## Application 

It’s not about the application you want to deploy, but for example, let’s say it will be a simple greetings server written in Go (`server.go`). 
 

```go

package main

import (
	"fmt"
	"net/http"
	"os"
)

func main() {
	// setup port
	port := os.Getenv("PORT")
	if port == "" {
		port = "8000"
	}
	// handle home page requests
	http.HandleFunc("/", func(w http.ResponseWriter, _ *http.Request) {
		fmt.Fprint(w, "Hi internet rando!")
	})
	// serve
	err := http.ListenAndServe(":"+port, nil)
	if err != nil {
		fmt.Println(err)
		os.Exit(2)
	}
}
```

It just starts to serve simple greeting on the port specified in the `$PORT` environment variable or 8000 by default. Note that Heroku [will provide you](https://devcenter.heroku.com/articles/runtime-principles#web-servers) a service port to bind to the `$PORT` environment variable. 

> **Note:** if you will choose Go, don't forget to initiate a go module with `go mod init <module_name>`, otherwise you will have to set up a build environment for `GOPATH` in Docker.

That is, let’s build it. 

## Container 

Containerization is a really nice thing in a nowadays programmer’s world. You don’t want to rely on any infrastructure environment anymore, don’t have to feel pain while trying to run your script on the foreign land. You can set up everything once and deploy anywhere. 

Don’t have a proper version of your language on the hosting platform – not a big deal, just take a proper Docker image. 

Fear to run deployment in an unknown environment – fear not, just configure everything you need in your container. 

Even if you don’t want to have applications in a huge OS wrapper – you can use a multistage build, i.e. build in Debian and run in Alpine, etc. 

So, let’s configure the container (`Dockerfile`): 


```yml
FROM golang:1.12-alpine

WORKDIR /opt/code/

ADD ./ /opt/code/

RUN go build -o greetings server.go

ENTRYPOINT ["./greetings"]
```

It will build the app and run it when you will start the container. 

## Deployment 

To deploy to Heroku you need to configure the deployment (`heroku.yml`): 


```yml
build:
  docker:
    web: Dockerfile
```

Assuming you have created a Heroku app already, we will deploy it. The easiest way is to deploy directly using CLI. 

### Manual deployment 

First, install [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli#download-and-install) and login: 

```bash 
heroku login 
``` 

Then you need to add Heroku remote to your Git repo. If you don’t have a Git repo yet, create it: 

```bash 
git init 
``` 

And then set up Heroku: 

```bash 
# greetings-server-example is my app name. Please use yours 
heroku git:remote -a greetings-server-example 
``` 

Then commit changes and push them to the Heroku:

```bash 
git add --all 
git commit -m "initial commit"
git push heroku master 
``` 

Now you will see the deployment log. If it was successful, you can call the service with 

```bash 
heroku open 
``` 

That will just open your service in the browser. There you will see: 

> Hi internet rando! 

### Automated deployment with GitHub 

You can go further and automate this process. Let’s say, you’re hosting the code on a GitHub. So, you can connect the Heroku app to your GitHub repo and automatically deploy each time you push something to the master branch. 

Open Heroku Dashboard and go to "Deploy". There click to the GitHub under "Deployment method". Go to App connected to GitHub and choose your repo and press "Connect". Then go to "Automatic deploys" and press "Enable Automatic Deploys". By default, you will connect to a master branch, but you can change it.  

![](https://thepracticaldev.s3.amazonaws.com/i/klil3ku1e2hcgl766qqr.png)

Now you are done. As a bonus, you can set a checkbox “Wait for CI to pass before deploy” if you’re using some CI on GitHub.  

After a push into GitHub master you will see a new deploy on the “environment” tab: 

![](https://thepracticaldev.s3.amazonaws.com/i/ftezsfunkjfe6gh6kdb3.png)

And you can check the deployment log in the “Activity” tab of Heroku dashboard. 

![](https://thepracticaldev.s3.amazonaws.com/i/b3399etjdwmjwiwdavm6.png)

Have fun with your fast deploy!