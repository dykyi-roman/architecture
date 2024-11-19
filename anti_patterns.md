# [Architecture](README.md)

## Anti-patterns

* Shared Kernel (Code)
* Copy and Paste
* MVC (Model-View-Controller)
* Controller
* Monolithic
* Spaghetti code
* Big Ball of Mud
* Hard code
* Overengineering
* Reinventing the wheel
* Dead code
* God Object
* Golden Hammer
* Boat Anchor
* Service Avalanche
* Dependency hell
* Race condition
* Null hell
* Trait
* Singleton
* Anemic model
* ActiveRecord

### Shared Kernel (Code)
The architectural anti-pattern occurs when multiple independent teams or services within a microservices architecture (or other distributed systems) use shared code or a common part of the core. While the idea may seem appealing for improving consistency and code reuse, in practice, it leads to a number of problems.

Why does this happen?
- Shared code or data is created to speed up development.
- Lack of clear design and context boundaries.
- The desire to avoid code duplication.
- Issues with data consistency.

Disadvantages:
- Difficulties with the independent development of modules.
- Challenges in managing versioning.
- Lack of flexibility when making changes due to side effects in other services.
- Increased complexity in testing and deployment.
- Leads to the accumulation of technical debt and scaling challenges.

### Copy and Paste

The task is completed faster, but bring language support for this code in the future. Should be detected and refactoring on the review step or creating a tech dept task for fixing the feature.

* The developer, instead of refactoring, adds one more similar code (condition as an example). 
* The developer copies the same code from different contexts. Each context has its own rules and behaviors, copying someone else's code we do not understand the logic of work. Better to use DI or extend if we want to reuse some please of the code instead of copying. 
* Copy please of code from third-party resources (Github, StackOverflow). By copying code, we copy solutions to someone else's problem. and in 90% of cases, it will not suit your code.

### MVC (Model-View-Controller)

I advise you to avoid this pattern. Since it is definitely not suitable for writing long-term enterprise applications.
Coming from the description, we can conclude that we have no place for organization our business logic.

Most of Web Application Developers use MVC architecture patterns to create applications. But when we dive into history, we discover that MVC was proposed way before Web Application were even taken into consideration.
It was proposed in 1979 by **[Trygve Reenskaug](https://en.wikipedia.org/wiki/Trygve_Reenskaug)**. It was designed to handle small parts of desktop application buttons, text boxes, etc. Not a whole application or a whole page. As time passed, this pattern evolved

* Model - must be rich, characterized by combining data and behavior and only keeping a thin service above
* Controller - business logic, located in methods called **Actions** and usually containing an algorithm mainly in a procedural style
* View - accepts variables and displays views

### Controller

There are many projects that use modern architectural patterns, but still use a Controller for collect actions.
A better approach would be to use the Action from **[ADR](https://github.com/pmjones/adr)** approach. Instead, put all your actions in one class and separate each Action in your own class. 

Problems:
* Broken **SRP** principe. The module is available to all actors
* Many do not use dependency in the class
* The code is not readable. `UserController` does not say anything for us, prefer to use the name what does it actually do in your code. Example: `ChangeUserEmailAction`, `CreateUserAction`, `SubscribeUserAction`
* The code is too big, in some projects it could be **1000** and more lines of code, depending on your business. Action should be thin

Benefits:
* We have context-agnostic and well-segregated code
* Code is cleaner and easier to change and reuse

### Monolithic
All functions and components of the system are implemented in a single large codebase, making deployment, updating, and scaling difficult.

Why does this happen?
- Historically developed architecture in older systems.
- Fear of change and moving to a more modern architecture.
- Lack of time and resources for refactoring.

Disadvantages:
- Any change can affect the entire system.
- Difficulties in scaling individual components.
- Slow deployment and updates.

### Spaghetti code

Basically, such code appears due to a lack of experience in development and a lack of understanding of programming principles. Example: "it works - don't touch it".

* Hard to read
* Hard to do code review
* Contains another anti-pattern inside
* Not reusable
* Hard to write tests

### Big Ball of Mud
A system built using this anti-pattern is a tangled mess of code with no clear structure or modularity. In such a project, it's difficult to identify layers, components, or understand the logic.

Why does this happen?
- Lack of planning and design at early stages.
- Constant changes in requirements without refactoring.
- Rushing development to meet deadlines.

Disadvantages:
- Very hard to maintain and develop: any new feature requires significant effort.
- High risk of errors when making changes.
- Complicates testing and bug-finding.

### Hard code

Applying a quick solution without further correction. This should be prevented during the code review. Fixed by creating tech tasks for refactoring.

* Rush in development
* Lack of code development practice in the team
* No Code Review

### Overengineering

Absolutely everything that is possible is configured and put into the config.

* This makes the project configuration incredibly difficult to understand
* DevOps does not respect you. Deploying such a system also entails additional costs
* Most of the settings are stay with default value and not used

### Reinventing the wheel

The point of this anti-pattern is that the developer implements his own solution for the problem from scratch. 

### Dead code

Existing dead code in the codebase

* The complexity of the project is growing
* Development speed slows down
* Inability to refactor or develop new functionality

Solution: Refactoring, Code review.

### God Object

In general, it happens when we forget about SRP.

* The object takes on too many responsibilities
* Code non-portability 
* Difficult to maintain code
* Code complexity

Solution: Follow: DDD, TDD, DRY, KISS, SOLID, YAGNI, GRASP

### Golden Hammer
Using the same tool or technology to solve all problems, even when it's not suitable for the specific case. Example: trying to use the same database or library in all projects, regardless of their characteristics.

Why does this happen?
- Familiarity with a certain technology and reluctance to learn new tools.
- Business pressure to reduce costs on training or adopting new technologies.

Disadvantages:
- Reduced solution effectiveness: an unsuitable tool can slow down development and degrade performance.
- Poor fit for long-term projects where requirements may change.
- Can lead to technological limitations and scalability issues.

### Boat Anchor
This is a situation where unused or unnecessary modules, classes, or libraries are left in the code, which were initially planned to be used but never became part of the system.

Why does this happen?
- The desire to keep code written for future use, even if it's not being used.
- A mistaken belief that the code might be useful later.

Disadvantages:
- Increased complexity of the code, making it harder to understand and maintain.
- Codebase pollution, increasing the risk of errors.
- Difficulties in maintaining and updating outdated code.

### Service Avalanche
This occurs in distributed systems when a failure in one service triggers cascading failures in other services, leading to complete system unavailability.

Why does this happen?
- Poor error handling and failure architecture.
- Lack of load-limiting mechanisms (circuit breakers) and timeouts.
- Services have strong dependencies on each other.

Disadvantages:
- Reduced system reliability.
- Difficulties in recovery after failure.
- Requires more time and resources for debugging and preventing such issues in the future.

### Dependency hell

* A lot of dependencies (class)
* A long dependency chain (modules)
* Conflicting dependencies (package management tool)
* Circular dependencies (class, modules)

### Race condition

Events occurring in an unexpected order.

Problems:
* Loss or corruption of data
* Mutual locks
* Vulnerabilities
* Floating bugs

Solution: Write an idempotent code

### Null hell

Avoid using **NULL** in the function argument and function return value.

Problems:
* Additional check for **NULL**

Solution: Use [Fail fast](https://www.yegor256.com/2015/08/25/fail-fast.html) approach or Value Objects

### Trait

Problems:
* Tight coupling
* Broken OCP
* Cannot be overwritten through inheritance
* Not testable in isolation
* Harder code understanding

Solution: Use inheritance instead of traits!

### Singleton

Problems:
* Difficulty testing
* Memory allocated to a Singleton can't be freed
* Global unique instance

Solution: Use a single instance and propagate it to places that use the object as a parameter to make the dependency explicit

### Anemic model

The domain objects contain little or no business logic like validations, calculations, rules, and so forth.

Problems:
* Data is exposed through getters and setters. No encapsulation. 
* High coupling. Low Cohesion.
* Data set by setter instead of using constructor arguments.

Solution: Use a Rich model

### ActiveRecord

Active Record is one of the most controversial architectural patterns with many supporters and opponents.

Problems:
* Broken OOP and SRP
* Mixing Infrastructure and Domain layers
* High coupling makes writing unit tests almost impossible

Solution: Use Datta Mapper Pattern

### Read
* [Антипаттерны в программировании и проектировании архитектуры](https://bool.dev/blog/detail/antipatterny-v-programmirovanii-i-proektirovanii-arkhitektury)
* [Why NULL is Bad](https://www.yegor256.com/2014/05/13/why-null-is-bad.html)
* [Anemic vs. Rich Domain Objects](https://www.baeldung.com/java-anemic-vs-rich-domain-objects#low-coupling)
* [Anemic Domain Model vs. Rich Domain Model](https://medium.com/@inzuael/anemic-domain-model-vs-rich-domain-model-78752b46098f)
