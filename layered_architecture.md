# [Architecture](README.md)

## Layered Architecture
The goal of traditional Layered Architectures is to segregate an application into different tiers, where each tier contains modules and classes that have shared or similar responsibilities, and work together to perform specific tasks.
As a result, our application will be divided into small, decoupled, components. Each of these components belongs to one of the three main layers:
* UI (Framework layer) - holds framework-specific configuration and allows leverages with the outside world
* Application layer - acts as a mediator for the Domain Layer: validating and mapping the data coming in
* Domain layer - is responsible for all the business logic
* Infrastructure layer - the majority of your application's external dependencies
![8.png](docs/8.png)

### When to use

### Rules
* The application is built around an independent object model
* Inner layers define interfaces. Outer layers implement interfaces
* Direction of coupling is toward the center
* All application core code can be compiled and run separate from infrastructure

### How to implement
* Divide applications into layers
* Follow dependency rules
* Follow dependency inversion principle [DIP](dip.md)

### Summary

### Tools
Keep your architecture clean help [Deptrac](https://qossmic.github.io/deptrac/) With it you can freely define your architectural layers over classes and which rules should apply to them.

### Read
* [The Clean Code Blog - The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
* [Layered Architecture](https://herbertograca.com/2017/08/03/layered-architecture/)
* [The Onion Architecture (Part 1...4)](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/)
* [4+2 Layered Architecture](https://medium.com/@nogueira.cc/4-2-layered-architecture-313329082989)
* [Book - Patterns of Enterprise Application Architecture ](https://www.amazon.com/-/en/dp/0321127420)