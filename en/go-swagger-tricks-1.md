# Go-Swagger Tricks. Standard HTTP handler.

Hey, reader! Maybe you had tried `go-swagger` library so far. If yes, you may notice that sometimes it's not so easy to use. And it may look a bit complicated to start using it.

In this number of small articles, I will share my experience on how to make go-swagger more friendly. Let's start.

## Handling requests in go-swagger

By default, go-swagger generates a specific handler type for each endpoint in your Swagger scheme - that is, you will get a code-generated structure with all import parameters kindly parsed for you and a number of responders for each response type (with pre-generated structures as well).

So usually it's ok, because you don't have to parse JSON requests by yourself and can never mind the response data marshaling - the go-swagger framework will handle all that stuff for you. But sometimes you just want to use a standard `net/http` compatible handler with some endpoint - maybe some library provides that for you (like Prometheus metrics endpoint or Kubernetes probes). Or maybe you have some HTTP libraries written in your organization - e.g. for custom authentication and so on. 

So you can't do that in vanilla `go-swagger` - each generated handler function requires an exact handler to be assigned. Let's say we want to expose a Prometheus metrics endpoint, so for the spec like this

```yaml
paths:
  /metrics:
    get:
      tags:
        - instruments
      summary: "Prometheus metrics"
      produces:
      - "application/json"
      responses:
        200:
          description: ok
          schema:
            $ref: "#/definitions/Any"
```

the generated handler function will look like that:

```go
func MetricsHandler(p instruments.GetMetricsParams) middleware.Responder {
    // some logic here
    return instruments.NewGetMetricsOK().
        WithPayload(models.Any(
        // response data here
    ))
}
```

So is it even possible to use standard handlers? Or should I rewrite all of them in go-swagger-way?

**Don't worry, there is no need to do extra work.**

## Adding wrapper

There is a way to do that by adding a tiny wrapper. We just need to find a reader and writer and put them to our `http.HandlerFunc` function. The reader is always in the generated request structure, but the reader is provided to the generated responder object. So we have to implement a `middleware.Responder` interface to get it. That may look like this:

```go
type CustomResponder func(http.ResponseWriter, runtime.Producer)

func (c CustomResponder) WriteResponse(w http.ResponseWriter, p runtime.Producer) {
	c(w, p)
}
```

And now we just need to wrap our standard handler into this wrapper and provide reader and writer as usual!

```go
func MetricsHandler(p instruments.GetMetricsParams) middleware.Responder {
	return CustomResponder(func(w http.ResponseWriter, _ runtime.Producer) {
		promhttp.Handler().ServeHTTP(w, p.HTTPRequest)
	})
}
```

## Improvements

We can go deeper and add constructor function, which will save us a couple of lines in future:

```go
func NewCustomResponder(r *http.Request, h http.Handler) middleware.Responder {
	return CustomResponder(func(w http.ResponseWriter, _ runtime.Producer) {
		h.ServeHTTP(w, r)
	})
}
```

and simplify our handler a bit:

```go
func MetricsHandler(p instruments.GetMetricsParams) middleware.Responder {
	return NewCustomResponder(p.HTTPRequest, promhttp.Handler())
}
```

Sure it may look not so impressive as a distributed database internals, but it will save you some time and help to integrate a `go-swagger` with ease.

See you in the next "Go-Swagger Tricks" episode!