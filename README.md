# Software Architecture Guide

A comprehensive guide to software architectural patterns, principles, and best practices for building scalable, maintainable applications.

## Table of Contents

- [Principles](#principles)
- [Architectural Patterns](#architectural-patterns)
- [Data & Integration Patterns](#data--integration-patterns)
- [Quality & Operations](#quality--operations)
- [Development Practices](#development-practices)
- [Resources](#resources)

---

## Principles

Foundational principles that guide software design decisions.

| Principle | Description |
|-----------|-------------|
| [SOLID](solid.md) | Five principles for object-oriented design |
| [DIP](dip.md) | Dependency Inversion Principle in depth |
| [GRASP](grasp.md) | General Responsibility Assignment patterns |

**Core Values:**
- Stability, Performance, Durability
- Scalability, Flexibility, Encapsulation
- Testability, Separation of Concerns
- Independence from framework, DB, UI, external services

---

## Architectural Patterns

Patterns for structuring applications at different scales.

### Application Architecture

| Pattern | When to Use |
|---------|-------------|
| [Layered Architecture](layered_architecture.md) | Simple apps, clear separation of concerns |
| [Clean Architecture](clean_architecture.md) | Complex business logic, long-term projects |
| [Hexagonal Architecture](hexagonal_architecture.md) | Multiple integrations, testability focus |
| [Modular Monolithic](modular_monolithic_architecture.md) | Growing apps, pre-microservices stage |

### Distributed Systems

| Pattern | When to Use |
|---------|-------------|
| [Microservice Architecture](microservice_architecture.md) | Large teams, independent deployments |
| [Event-driven Architecture](event_driven_architecture.md) | Async processing, loose coupling |
| [Event Sourcing](event_sourcing.md) | Audit trails, temporal queries |

### Domain Modeling

| Pattern | When to Use |
|---------|-------------|
| [DDD](ddd.md) | Complex domains, collaborative design |
| [Design Patterns](https://github.com/kamranahmedse/design-patterns-for-humans) | Common OOP solutions |
| [Anti-patterns](anti_patterns.md) | What to avoid |

---

## Data & Integration Patterns

Patterns for handling data flow and system integration.

### Command/Query Patterns

| Pattern | Description |
|---------|-------------|
| [CQS](cqs.md) | Method-level command/query separation |
| [CQRS](cqrs.md) | Architectural-level read/write separation |

### API & Communication

| Topic | Description |
|-------|-------------|
| [API Design](api_design.md) | REST, GraphQL, gRPC best practices |
| [Proxy Server](proxy_server.md) | Forward/Reverse proxy, Load balancing |
| [Concurrency Patterns](concurrency_patterns.md) | Locking, distributed coordination |

---

## Quality & Operations

Ensuring reliability, security, and maintainability.

| Topic | Description |
|-------|-------------|
| [Testing Strategy](testing_strategy.md) | Test types, coverage, best practices |
| [Observability](observability.md) | Logging, Metrics, Tracing |
| [Security Architecture](security.md) | Authentication, Authorization, OWASP |

---

## Development Practices

Standards and processes for development teams.

| Topic | Description |
|-------|-------------|
| [Code Standards](code_standards.md) | PSR-12, naming, PHPDoc |
| [Code Review Guide](code_review.md) | Review checklist, best practices |

---

## Resources

### Courses
- [Udi Dahan - Advanced Distributed Systems Design](https://coursehunter.net/course/prodvinutaya-arhitektura-raspredelennyh-sistem?lesson=1)
  - [Notes 1](https://www.michalbialecki.com/2020/06/23/what-i-learned-from-2500-udi-dahan-course/) | [Notes 2](https://hackmd.io/@pierodibello/Advanced-Distributed-System-Design) | [Notes 3](https://gist.github.com/craigtp/05a82b51557adc278acd71b5a2b88905)

### Curated Lists
- [awesome-scalability](https://github.com/binhnguyennus/awesome-scalability)
- [awesome-design-patterns](https://github.com/DovAmir/awesome-design-patterns)
- [professional-programming](https://github.com/charlax/professional-programming)
- [system-design](https://github.com/karanpratapsingh/system-design)

### Reference Architectures
- [Clean Architecture (.NET)](https://github.com/ardalis/CleanArchitecture)
- [.NET Microservices Architecture](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/)
- [Azure Architecture Guide](https://learn.microsoft.com/en-us/azure/architecture/guide/)

### Leadership
- [Awesome CTO](https://github.com/kuchin/awesome-cto)

---

## License

This repository is for educational purposes.