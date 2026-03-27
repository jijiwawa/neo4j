# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

This is a multi-module Maven project requiring **Java 17** and **Maven 3.8.2+**.

```bash
# Full build with tests
mvn clean install -T1C

# Build without tests (faster)
mvn clean install -DskipTests -T1C

# Build a single module
mvn clean install -pl community/kernel -am -T1C
```

Increase Maven memory if builds fail: `export MAVEN_OPTS="-Xmx2048m"`

## Running Tests

```bash
# Run all unit tests in a module
mvn test -pl community/kernel

# Run a specific test class
mvn test -pl community/kernel -Dtest=ClassName

# Run a specific test method
mvn test -pl community/kernel -Dtest=ClassName#methodName

# Run integration tests (matches *IT.java, *IntegrationTest.java)
mvn verify -pl community/community-it/cypher-it

# Skip Cypher module (useful for faster builds)
mvn clean install -DskipCypher -T1C
```

## Code Style

- **Java**: Palantir Java Format (enforced via Spotless)
- **Scala**: scalafmt with `.scalafmt.conf` (max column 100)

Run formatter before committing:
```bash
mvn spotless:apply
```

## Architecture

### Top-level modules
- `annotations/` - Annotation processors and public API markers
- `community/` - All main modules (kernel, cypher, bolt, server, etc.)
- `packaging/` - Standalone distribution assembler

### Key community modules
| Module | Description |
|--------|-------------|
| `graphdb-api` | Public graph database API (`org.neo4j.graphdb`) |
| `kernel` | Core storage engine and transaction processing |
| `kernel-api` | Internal kernel APIs |
| `cypher/` | Cypher query language (Scala + Java) |
| `bolt` | Bolt protocol server for client connections |
| `server` | HTTP REST API server |
| `dbms` | Database management service |
| `neo4j-harness` | Test harness for embedded testing |
| `community-it/` | Integration tests by component |

### Test utilities
- `kernel-test-utils/` - Test helpers for kernel testing
- `testing/` - General test utilities
- `server-test-utils/` - Server test helpers

## License Headers

All source files require GPL-3.0 license headers. The build enforces this via license-maven-plugin.

## Cypher Module

The Cypher query engine is a mix of Scala (parser, planner) and Java. It can be excluded with `-DskipCypher` for faster builds when not modifying Cypher code.
