# [Architecture](README.md)

## Event-driven architecture

### Event Sourcing
Each service publishes an event whenever it updates its data. Other services subscribe to events.
When an event is received, a service do it something.

Event Sourcing idea is simple: our domain is producing events that represent every change made in system. 
If we take every event from the beginning of the system and replay them in the initial state, we will get to the current state of the system. 
It works similarly to transactions on our bank accounts; we can start with an empty account, replay every single transaction and (hopefully) get the current balance.

### Summary
* All events is immutable (+/-)
* Easy make a rollback
* Hard to support

### Read
