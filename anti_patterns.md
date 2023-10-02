# [Architecture](README.md)

## Anti-patterns

* Copy and Paste
* MVC (Model-View-Controller)
* Controller
* Spaghetti code
* Hard code
* Over configuration
* Reinventing the wheel
* Dead code
* God Object
* Dependency hell
* Race condition
* Null hell
* Trait
* Singleton
* Anemic model
* ActiveRecord

### Copy and Paste

The task is completed faster, but bring language support for this code in the future. Should be detected and refactoring on the review step or creating a tech dept task for fixing the feature.

* The developer, instead of refactoring, adds one more similar code (condition as an example). 
* The developer copy same code from different context. Each context has own rules and behaviours, copying someone else's code we not understand logic of work. Better to use DI or extending if we want to reuse some please of the code instead of copying. 
* Copy please of code from third party resources (Github, StackOverflow). By copying code, we copy solutions to someone else's problem. and in 90% of cases it will not suit in your code.

### MVC (Model-View-Controller)

I advise you to avoid this pattern. Since it is definitely not suitable for writing long-term enterprise application.
Coming from the description, we can conclude that we have no place for organization our business logic.

Most of Web Application Developers use MVC architecture pattern to create applications. But when we dive into history, we discover that MVC was proposed way before Web Application were even taken into consideration.
It was proposed in 1979 by **[Trygve Reenskaug](https://en.wikipedia.org/wiki/Trygve_Reenskaug)**. It was designed to handle small parts of desktop application button, textbox, etc. Not a whole application or a whole page. Time passed, this pattern evolved

* Model - must be rich, characterized by combining data and behavior and only keeping a thin service above
* Controller - business logic, located in methods called **Actions** and usually containing an algorithm mainly in a procedural style
* View - accepts variables and displays views

### Controller

There are many projects that use modern architectural patterns, but still use a Controller for collect actions.
A better approach would be to use Action from **[ADR](https://github.com/pmjones/adr)** approach. Instead, to put all your actions in one class separate each Action in own class. 

Problems:
* Broken **SRP** principe. The module is available to all actors
* Many not using dependency in the class
* The code is not readable. `UserController` do not say anything for us, prefer use name what does it actually do your code. Example: `ChangeUserEmailAction`, `CreateUserAction`, `SubscribeUserAction`
* The code is too big, in some project it could be **1000** and more line of code, it depend on or your business. Action should be thin

Benefits:
* We have context agnostic and well segregated code
* Code is cleaner and easier to change and reuse

### Spaghetti code

Basically, such code appears due to a lack of experience in development and a lack of understanding of programming principles. Example: "it works - don't touch it".

* Hard to read
* Hard to do code review
* Contains another anti-patterns inside
* Not reusable
* Hard to write a tests

### Hard code

Applying a quick solution without further correction. Should be prevented on the code review. Fixed by creating tech tasks for refactoring.

* Rush in development
* Lack of code development practice in the team
* No Code Review

### Over configuration

Absolutely everything that is possible is configured and put into the config.

* This makes the project configuration incredibly difficult to understand
* DevOps does not respect you. Deploying such a system also entails additional costs
* Most of the settings are stay with default value and not used

### Reinventing the wheel

The point of this anti-pattern is that the developer implements his own solution for the problem from scratch. 

### Dead code

Existing dead code in the codebase

* Complexity of the project is growing
* Development speed slows down
* Inability to refactor or develop new functionality

Solution: Refactoring, Code review.

### God Object

In general, it happens when we forgot about SRP.

* The object takes on too many responsibilities
* Code non-portability 
* Difficult to maintain code
* Code complexity

Solution: Follow: DDD, TDD, DRY, KISS, SOLID, YAGNI, GRASP

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
* Memory allocated to an Singleton can't be freed
* Global unique instance

Solution: Use a single instance and propagate it to places that use the object as a parameter to make the dependency explicit

### Anemic model

The domain objects contain little or no business logic like validations, calculations, rules, and so forth.

Problems:
* Data is exposed through getters and setters. No encapsulation. 
* High coupling. Low Cohesion.
* Data set by setter instead of using constructor arguments.

Solution: Use Rich model

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