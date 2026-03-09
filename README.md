# Software Architecture Guide

A comprehensive guide to software architectural patterns, principles, and best practices for building scalable, maintainable applications.

---

<table>
  <tr>
    <td width="30%" align="center">
      <a href="https://dykyi-roman.github.io/projects/architecture-visualizer">
        <img src="https://dykyi-roman.github.io/projects/architecture-visualizer/img.svg" alt="Visualizer Preview" width="200"/>
      </a>
    </td>
    <td width="70%">
      <h3>Interactive Architecture Visualizer</h3>
      <p>Explore all architectural patterns and their relationships through an interactive visual map.</p>
      <a href="https://dykyi-roman.github.io/projects/architecture-visualizer">
        <img src="https://img.shields.io/badge/Open-Architecture%20Visualizer-blue?style=for-the-badge&logo=github" alt="Architecture Visualizer"/>
      </a>
    </td>
  </tr>
</table>

---

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

## Tools

### Claude Code Extension for Architecture

If you use [Claude Code](https://claude.ai/claude-code), consider installing the **[Awesome Claude Code](https://github.com/dykyi-roman/awesome-claude-code)** extension:

```bash
composer require dykyi-roman/awesome-claude-code
```

**Why use it with this guide?**

This extension transforms theoretical knowledge from this repository into practical automation:

| Feature | Benefit |
|---------|---------|
| **Architecture Audit** | Automatically verify your code against DDD, Clean Architecture, Hexagonal, CQRS patterns |
| **PSR Compliance** | Check adherence to PSR-1, PSR-4, PSR-12 standards |
| **Code Generation** | Generate DDD components (Entity, ValueObject, Aggregate, Repository) from templates |
| **CQRS Scaffolding** | Create Commands, Queries, UseCases, ReadModels with proper structure |
| **Stability Patterns** | Generate Circuit Breaker, Retry, Rate Limiter implementations |
| **Documentation Audit** | Ensure code documentation meets quality standards |

**What's included:**
- 🎯 **8 commands** — slash commands for audits and generation
- 🤖 **11 agents** — specialized sub-agents for different architectural tasks
- 📚 **73 skills** — knowledge bases, generators, and templates

The extension helps enforce the architectural principles described in this guide directly in your development workflow with Claude Code.

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