# Practical Use of Finite-State Machines

This is the first article in a series dedicated to FSM usage in distributed system architecture. We will talk about domains, transactions, and sagas. But let's start with the basics.

## Finite-State Machine

When we think about the finite-state machine, we probably imagine some computer science-related entities, math, and diagrams like that:

![Boring FSM](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8egg9mj6k8b7o7yu1bh6.png)

Besides scientific language, a finite-state machine is a final set of states and the transitions between them. When it comes to real engineering, states are a set of consistent states in which the model can be. Normally the set is not huge - having an FSM of a hundred states will cause really complex code. In my experience, it is something between three for the simplest models and 20-30 for most complex ones.

Each state represents a real-world state of the model in the current moment. Important note: the state has to describe the state completely - that means you have to rely on the state field only to identify the current model state. If you need to check some extra attributes to identify the state - your FSM isn't granular enough.

Transitions, in turn, are the logic of the model (or class, domain, etc.), which depends on the current state. So now everything your model is supposed to do depends on its current state - well like real-world things do.

## FSM in real life

You may wonder why we are talking about something that looks too scientific and forces your code in some boundaries? While you can still implement the same logic without any "weird-looking-diagrams"? That's a great question, and we will answer it.

Let's look at the example. Say, we have a hand-made furniture workshop and store. So the customer orders a tailor-made table for its kitchen. We have to accomplish the following steps:
- accept the order;
- charge customers money;
- request the workshop to produce the table;
- request workers to move the table to the warehouse once it is done;
- order transport company to deliver the table from the warehouse to customer.

It looks pretty straight-forward, but the devil is in details. You can't just do all these operations in the same API call, because of its duration. While the regular request processing time is under a second, fulfilling the order may take weeks. You can't even calculate the exact time because of the moving part amount involved.

So you have to save the milestones during the order fulfillment. And by "save" I mean "store in a database". Here we are talking about the Order model.

```go
type Order struct {
    ID       string
    ClientID string
    Detais   Details
    State    State
}
```

The simplest order has a unique ID, customer ID, order details (like delivery address, cost, positions, etc.). We add an attribute `State` and will discuss it in detail.

## A process described using FSM

The process consists of the following steps (more or less):

![Steps](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g1l2pw3imdlyox7zv6s3.png)

Now we can prepare a list of states for our order. They will represent each step via a related order state.

```go
type State string

const (
    OrderCreated         State = "order_created"
    PaymentPending       State = "payment_pending"
    ManufacturingPending State = "manufacturing_pending"
    StockMovePending     State = "stock_pending"
    DispatchPending      State = "dispatch_pending"
    DeliveryPending      State = "delivery_pending"
    ConfirmationPending  State = "confirmation_pending"
    OrderConfirmed       State = "order_confirmed"
)
```

Note that we don't call states like `"order_accepted"` or `"order_delivered"`, but `"payment_pending"` and `"confirmation_pending"` instead. That's because `"order_accepted"` doesn't represent the state, it represents an event in the past. We can use events to trigger state transitions, but the state has to describe the real-world state.

After successful order placement, we have to charge money from the customer's bank account. And this is the current state of the order - awaiting payment, or `"payment_pending"`. When it's done, we sent the order to the workshop and wait until it is manufactured. In this case, the state `"order_paid"` will also be irrelevant - it is an event of successful payment, but the state is waiting till the table will be done by carpenters.

So now we can go a bit nerdy and draw a state diagram.

![State Diagram](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x41yy5w29pobado3wgof.png)

Thus, we got our process explicitly described as a finite-state machine. In fact, it is a pipeline pattern, defining which step will run in which order. But FSM allows us to clearly define validations at each step, and visibly decouple the logic of each step (or state), so you don't have to hold the whole sequence in mind while programming.

Let's now dive deeper into this example.

### More than just a pipeline

In real life, things can sometimes go wrong. When you are developing software, there is a myriad of places, where something may go wrong. In other words, you have to handle errors and failures.

It's simple to handle an error when you are processing the synchronous HTTP request. You probably can fit it into a single database transaction and just rollback any changes if an error occurs. You also can simply return an error to the HTTP client without any extra operations like cleanups, compensation transactions, etc.

But when we talk about long or complex operations or actions across several distributed services, a simple approach doesn't work anymore. We need to split into steps, and steps may also include some sub-steps and so on. But how to handle errors in the middle of the pipeline? You can't just return an error message to the user because you now have some mess to clean up. Let's take a look.

In a workshop, many things may go unexpected. Carpenter may take a sick leave or even resign, rare wood may be out of stock, material delivery may delay, and so on. Anyway, there are real cases, when the workshop can't fulfill the order when it is already paid.

So now we have to handle an error in the middle of the process. There may be different kinds of errors or the same error, but the handling will depend on the model state. 

If the error occurs when the state is `"manufacturing_pending"`, that means the workshop can't fulfill the order, and you have to refund the money.

There will be new states also.

```go
const (
    RefundPending  State = "refund_pending"
    OrderCancelled State = "order_cancelled"
)
```

The state diagram will now look like this.

![State Diagram with Errors](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bujoapyc5lw9disoic2f.png)

## Consistent error handling

The more errors that need to be processed, the more complex the state diagram becomes. But thanks to the FSM, the processing always corresponds to the current state, and the actions taken will always be appropriate.

If an error (or other events) occurs, actions depend primarily on the current state of the model. Next, the transition conditions are checked, and only then the transition logic is executed. Thus, the processing of any events occurring to the entity will always be consistent with its state. 

This is what the complete process might look like, with all sorts of errors taken into account.

![Full State Diagram](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ihrh578na2h3z85z9xx5.png)

It may look similar to the saga pattern for those, who have dealt with distributed transactions, and it actually does. But I will tell you about more advanced FSM applications in the following posts.

## Practical Advice

Use the FSM to describe states and transitions between them. This will help to clarify and separate the logic of the different states into distinct modules. Try to describe the states in a granular way. Don't be afraid of a large number of states if the data model can actually be in each of them. Have fun drawing diagrams.
