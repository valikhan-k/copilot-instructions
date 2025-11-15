# Project Development Guidelines

This project follows specific standards and patterns for .NET development. Copilot should assist developers while adhering to these guidelines.

## Quick Reference

- **Platform**: .NET 8/9+ with C# 12+
- **API Style**: Minimal APIs (not Controllers)
- **Architecture**: Hexagonal Architecture with pragmatic evolution
- **Organization**: Feature-oriented folders, one endpoint per file

## Detailed Guidelines

For comprehensive guidance on specific topics, refer to:

- [Architecture & Design](./instructions/architecture.instructions.md)
- [Security Standards](./instructions/security.instructions.md)
- [Code Quality](./instructions/code-quality.instructions.md)
- [Code Review Approach](./instructions/code-review.instructions.md)
- [Testing Standards](./instructions/testing.instructions.md)

## Core Principles

**Start Simple, Evolve Deliberately**
- Begin with single project and folder boundaries
- Extract to separate projects only when complexity demands it
- Let actual usage patterns dictate structure

**Pragmatism Over Dogma**
- Empirical cleanliness over theoretical purity
- Duplication is better than wrong abstraction
- Question every abstraction - does it earn its place?

**Code for Readability**
- Code is read 10x more than written
- Document "why", not "what"
- Use descriptive names and clear intent

## Technology Stack

### Core
- .NET 8/9+ with latest stable SDK
- C# 12+ with nullable reference types enabled
- Minimal APIs for HTTP interfaces

### Preferred Libraries
- **ORM**: Entity Framework Core or Dapper
- **Validation**: FluentValidation
- **Logging**: Serilog with structured logging
- **Testing**: xUnit, NSubstitute, FluentAssertions, Testcontainers
- **Mapping**: Mapster or manual mapping

## When Generating Code

1. Match existing patterns in the codebase
2. Follow established folder structure and naming
3. Use latest C# features (records, pattern matching, collection expressions)
4. Keep code simple and focused
5. Include appropriate error handling and logging

## Questions or Clarifications

When uncertain, favor:
- The simpler solution
- Explicit over clever
- Readable over concise
- Working over theoretically perfect