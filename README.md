# Spring Boot Developer Skill

A comprehensive GitHub Copilot skill for building production-grade Spring Boot 3.x applications. This skill provides expert guidance on everything from project scaffolding through testing, security, observability, and deployment readiness.

## 📋 Overview

This skill enhances GitHub Copilot with deep Spring Boot knowledge, enabling it to help you:

- Scaffold new Spring Boot projects with best-practice structure
- Build REST APIs with proper layering, validation, and error handling
- Design data layers with JPA, repositories, transactions, and migrations
- Implement JWT authentication and Spring Security configurations
- Integrate with external services using WebClient, RestClient, and Kafka
- Add resilience patterns (circuit breakers, retries, rate limiting)
- Write comprehensive tests (unit, integration, with Testcontainers)
- Configure observability, metrics, and structured logging
- Troubleshoot common Spring Boot errors and anti-patterns

## 🚀 Installation

### Option 1: Clone This Repository

```bash
# Clone to your VS Code skills directory
git clone https://github.com/chuls50/springboot-developer.git ~/.vscode/skills/springboot-developer
```

### Option 2: Download and Install Manually

1. Download this repository as a ZIP
2. Extract to `~/.vscode/skills/springboot-developer` (or your configured skills directory)
3. Restart VS Code or reload the window

### Option 3: Use as a Custom Skill

Copy the `.claude/skills/springboot-developer` folder to your project's `.claude/skills/` directory for project-specific customization.

## 📚 What's Included

### Main Skill Guide ([SKILL.md](.claude/skills/springboot-developer/SKILL.md))

Comprehensive guide covering:

- **Scaffolding** - Maven/Gradle setup, project structure, dependencies
- **REST Layer** - Controllers, DTOs, validation, error handling
- **Data Layer** - JPA entities, repositories, service layer, transactions
- **Database Migrations** - Flyway patterns and best practices
- **Configuration** - YAML profiles, typed configuration, externalized secrets
- **Cross-cutting Concerns** - Caching, async, scheduling
- **Security** - JWT, Spring Security, CORS, method-level security
- **Testing** - Unit tests, integration tests, Testcontainers
- **Production Readiness** - Actuator, observability, structured logging
- **Modern Java** - Virtual threads, records, HTTP interfaces
- **Troubleshooting** - Common errors and fixes

### Reference Guides

#### [Architecture Reference](references/architecture.md)

- Layered architecture patterns
- Hexagonal / Ports & Adapters
- Multi-module Maven projects
- Docker Compose for local development
- Event-driven patterns (in-process)
- Database migration strategies

#### [Security Reference](references/security.md)

- Complete JWT setup (jjwt 0.12.x)
- SecurityFilterChain configuration
- UserDetailsService implementation
- OAuth2 resource server patterns
- Method-level security (@PreAuthorize, @PostAuthorize)
- CORS configuration

#### [Integration Patterns](references/integrations.md)

- **Resilience4j** - Circuit breakers, retries, rate limiters, bulkheads
- **WebClient** - Reactive HTTP client with timeouts and error handling
- **RestClient** - Synchronous HTTP client (Spring Boot 3.2+)
- **Spring Kafka** - Producer/consumer patterns, error handling, DLT
- **API Integration Best Practices** - Timeouts, correlation IDs, health checks

## 🎯 How to Use

Once installed, GitHub Copilot will automatically use this skill when you:

- Ask about Spring Boot, Spring Data JPA, Spring Security
- Request help with REST APIs, microservices in Java
- Mention Maven, Gradle, Flyway, Liquibase, Testcontainers
- Need assistance with authentication, JWT, OAuth2
- Ask about caching, scheduling, async processing
- Request help debugging Spring errors like `NoSuchBeanDefinitionException`
- Work with Kafka, WebClient, Resilience4j

### Example Prompts

```
"Create a Spring Boot REST controller for products with pagination"

"Add JWT authentication to my Spring Boot app"

"Set up Flyway migrations for my existing database"

"Write an integration test using Testcontainers"

"Add circuit breaker to my external API calls"

"Configure Spring Security with JWT tokens"

"Set up Kafka consumer with error handling"

"How do I fix LazyInitializationException?"
```

## 🛠️ Technology Coverage

### Core Spring Boot

- **Version**: Spring Boot 3.3+ / Java 21 (Java 17 minimum)
- **Web**: Spring MVC, REST, validation
- **Data**: Spring Data JPA, Hibernate, pagination
- **Security**: Spring Security 6.x, JWT, OAuth2
- **Testing**: JUnit 5, Mockito, Testcontainers

### Integrations

- **HTTP Clients**: WebClient, RestClient, HTTP Interfaces
- **Messaging**: Spring Kafka, event-driven patterns
- **Resilience**: Resilience4j (circuit breaker, retry, rate limiter, bulkhead)
- **Observability**: Actuator, Micrometer, structured logging
- **Migrations**: Flyway, Liquibase
- **Caching**: Spring Cache, Redis

### Modern Java Features

- Virtual threads (Project Loom)
- Records for DTOs
- HTTP Interfaces for declarative clients
- Pattern matching and sealed types
- Text blocks and smart string formatting

## 📖 Quick Reference Tables

All guides include quick reference tables for fast lookup:

| Topic              | What You'll Find                                          |
| ------------------ | --------------------------------------------------------- |
| New project setup  | Package structure, Maven POM, entry point                 |
| REST endpoints     | Controller patterns, DTOs, validation, error handling     |
| Database access    | JPA entities, repositories, query methods, N+1 prevention |
| Testing strategies | Unit tests, slice tests, integration tests with real DB   |
| Security setup     | JWT filter chain, UserDetailsService, CORS config         |
| External APIs      | Circuit breakers, timeouts, retry logic, correlation IDs  |
| Common errors      | Troubleshooting table with causes and fixes               |

## 🎓 Best Practices Emphasized

This skill teaches and enforces:

✅ **DTOs, not entities, cross layer boundaries** - Decouple API from persistence  
✅ **Transactions in service layer** - Proper transaction management  
✅ **Validate at the boundary** - `@Valid` on controller inputs  
✅ **Constructor injection** - Testable, explicit dependencies  
✅ **Never hardcode secrets** - Environment variables and vaults  
✅ **Paginate everything unbounded** - `Page<T>` not `List<T>`  
✅ **Migrations, not ddl-auto** - Flyway for schema management  
✅ **Lazy associations by default** - Prevent N+1 queries  
✅ **Circuit breakers for external calls** - Prevent cascading failures  
✅ **Structured logging** - Proper observability

## 🔧 Customization

To customize this skill for your team:

1. Fork this repository
2. Edit `SKILL.md` and reference files with your:
   - Company-specific naming conventions
   - Custom base packages and structures
   - Internal libraries and frameworks
   - Team coding standards
3. Update the `description` field in the YAML frontmatter to adjust trigger patterns
4. Share with your team via your fork

## 📝 Version Notes

**Current Version**: 1.0.0  
**Spring Boot Baseline**: 3.3+  
**Java Requirement**: Java 21 recommended, Java 17 minimum

> **Note**: Spring Boot 4.0 was released in November 2025. The core patterns in this skill remain applicable. The major breaking change in Boot 3.0 was the `javax.*` → `jakarta.*` migration, which is already covered.

## 🤝 Contributing

Found an error or want to add coverage for a new Spring feature?

1. Fork the repository
2. Create a feature branch
3. Make your changes (update SKILL.md or reference files)
4. Submit a pull request with a clear description

## 📄 License

MIT License - feel free to use, modify, and share.

## 🙏 Acknowledgments

This skill synthesizes patterns from:

- Official Spring documentation and guides
- Spring Boot best practices from production systems
- Common patterns from enterprise Spring development
- Community knowledge from Stack Overflow and GitHub discussions

## 📧 Support

For questions or issues:

- Open an issue on GitHub
- Check existing issues for common problems
- Review the troubleshooting section in SKILL.md

---

**Happy Spring Boot Development! 🍃**
