# Java Tooling And Modern Java

## Baseline

- Target a current **LTS** JDK (set it with a toolchain, not whatever is on the developer's PATH) and use the build wrapper (`./mvnw` or `./gradlew`) so everyone builds the same way.
- Prefer **records**, **sealed interfaces**, **pattern matching** (`instanceof`, `switch`), **text blocks** and `var` for locals where the type is obvious.
- Use **virtual threads** for blocking, request-per-task workloads (`Executors.newVirtualThreadPerTaskExecutor()`); keep CPU-bound work on a bounded pool.

```java
import java.util.List;

public class Demo {
    sealed interface Shape permits Circle, Rect {}
    record Circle(double radius) implements Shape {}
    record Rect(double w, double h) implements Shape {}

    // Exhaustive switch over a sealed hierarchy: adding a permitted type breaks compilation here
    static double area(Shape shape) {
        return switch (shape) {
            case Circle c -> Math.PI * c.radius() * c.radius();
            case Rect r -> r.w() * r.h();
        };
    }

    // Records validate in a compact constructor and are immutable
    record Email(String value) {
        Email {
            if (value == null || !value.contains("@")) {
                throw new IllegalArgumentException("invalid email");
            }
        }
    }

    public static void main(String[] args) {
        List<Shape> shapes = List.of(new Circle(1), new Rect(2, 3));
        System.out.println(shapes.stream().mapToDouble(Demo::area).sum());
        System.out.println(new Email("ada@example.com"));
    }
}
```

## Build And Quality Gate

```bash
./mvnw -B verify          # or ./gradlew check
```

| Concern | Tool |
|---------|------|
| Formatting | Spotless (google-java-format) or Checkstyle, enforced in the build |
| Bugs | Error Prone and/or SpotBugs |
| Nullness | JSpecify annotations with NullAway, or Optional at API boundaries |
| Dependencies | `dependency:analyze` / Gradle dependency locking; OWASP Dependency-Check or `osv-scanner` |
| Tests | JUnit 5, AssertJ, Mockito (sparingly), Testcontainers for real databases and brokers |
| Coverage | JaCoCo (report, with a sensible floor rather than 100%) |

- Fail the build on warnings for new code (`-Xlint:all -Werror` or Error Prone severity).
- Commit lockfiles or a BOM; do not use version ranges for dependencies.

## Coding Practices

- Prefer immutability: `final` fields, records, `List.of`, `Map.copyOf`. Copy mutable inputs defensively.
- Use `Optional` for return values only, never for fields or parameters. Return empty collections, not `null`.
- Use try-with-resources for every `AutoCloseable`; never swallow exceptions; wrap with context (`new IllegalStateException("load user " + id, e)`).
- Use `java.time` (never `Date`/`Calendar`), `BigDecimal` for money, and `SecureRandom` for tokens.
- Use parameterised queries (`PreparedStatement`, JPA named parameters); never concatenate SQL. Disable XML external entities when parsing XML. Avoid Java serialization of untrusted data.
- Log with SLF4J placeholders (`log.info("created {}", id)`), never log secrets.

## Spring Boot (When You Use It)

- Constructor injection only (single constructor, no `@Autowired` on fields); keep controllers thin and services transactional at the service boundary.
- Bind configuration with `@ConfigurationProperties` records and validate them at startup.
- Return DTOs, not entities; validate input with Bean Validation (`@Valid`); centralise errors in `@RestControllerAdvice` producing RFC 9457 problem details.

## Quick Checklist

- [ ] JDK toolchain and wrapper pinned; builds are reproducible
- [ ] Format, static analysis, tests and dependency scan run in CI
- [ ] Immutable data (records), exhaustive `switch` over sealed types
- [ ] Resources closed with try-with-resources; no `null` collections
