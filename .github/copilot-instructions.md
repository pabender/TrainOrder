# Copilot Coding Agent Instructions

Trust these instructions. Only search the codebase if the information here appears incomplete or incorrect.

## What This Repository Does

**TrainOrder** is a single-module Spring Boot web application that manages railroad **Form 19 Train Orders** for Free-moN Operations Railway. It lets a dispatcher create, view, print, and delete Form 19 orders. Once saved, orders are immutable (no edit). The print view is a pixel-faithful Form 19 paper replica.

## Stack & Versions

| Tool | Version |
|---|---|
| Java | OpenJDK 17 (Ubuntu) |
| Spring Boot | 4.0.5 |
| Maven Wrapper | 3.9.14 (`.mvn/wrapper/maven-wrapper.properties`) |
| Database | H2 in-memory (`trainorderdb`) — **data lost on restart** |
| Templates | Thymeleaf |
| CSS/JS | Plain CSS + vanilla JS (no build tool, no bundler) |
| Lombok | annotation processor only — excluded from final JAR |

## Build, Test, and Run Commands

Always run commands from the repository root via the Maven wrapper. Never use a system `mvn`.

```bash
# Run all tests (13 tests, ~15 s) — must be green before any PR
./mvnw -q test

# Package (produces target/TrainOrder-0.0.1-SNAPSHOT.jar, ~27 MB)
./mvnw -q package -DskipTests

# Run the application (port 8080)
./mvnw spring-boot:run

# Confirm the app is healthy after startup
curl -s http://localhost:8080/actuator/health   # → {"status":"UP"}
```

`./mvnw -q test` is the **CI gate**. A GitHub Actions workflow (`.github/workflows/ci.yml`) runs `./mvnw --batch-mode test` on every push to `main` and on every pull request. Always run `./mvnw -q test` locally after any non-trivial change and fix all failures before finishing.

**Known H2 SQL pitfall:** H2 does not support `DATE(column)`. Use `CAST(column AS DATE)` instead. Example: `WHERE CAST(created_at AS DATE) = CURRENT_DATE`.

## Architecture

```
Browser → TrainOrderController (/orders)
              ↓
          TrainOrderService
              ↓
          TrainOrderRepository → H2 via NamedParameterJdbcTemplate
```

All new classes must be under `org.trainbeans.trainorder.<layer>`.

## Key File Locations

| Purpose | Path |
|---|---|
| Domain model | `src/main/java/org/trainbeans/trainorder/model/TrainOrder.java` |
| Repository (JDBC) | `src/main/java/org/trainbeans/trainorder/data/TrainOrderRepository.java` |
| Service | `src/main/java/org/trainbeans/trainorder/service/TrainOrderService.java` |
| Web controller | `src/main/java/org/trainbeans/trainorder/web/TrainOrderController.java` |
| Home redirect | `src/main/java/org/trainbeans/trainorder/web/HomeController.java` |
| App entry point | `src/main/java/org/trainbeans/trainorder/TrainOrderApplication.java` |
| DB schema | `src/main/resources/schema.sql` (run on every startup via `spring.sql.init.mode=always`) |
| Runtime config | `src/main/resources/application.properties` |
| CSS | `src/main/resources/static/css/style.css` |
| Templates | `src/main/resources/templates/orders/{list,form,print}.html` |
| Build file | `pom.xml` |

## Controller Routes

| Method | Path | Action |
|---|---|---|
| GET | `/` | Redirect → `/orders` |
| GET | `/orders` | List all orders |
| GET | `/orders/new` | New order form (order number pre-filled) |
| POST | `/orders` | Save new order |
| GET | `/orders/{id}/print` | Print / view order |
| POST | `/orders/{id}/delete` | Delete order |

Orders **cannot be edited** after creation. There are no edit or complete endpoints.

## Model & Schema Conventions

- `TrainOrder` uses Lombok `@Data @Builder @NoArgsConstructor @AllArgsConstructor`.
- Every Form 19 blank maps 1-to-1 to a column in `train_orders`. Field names use camelCase; column names use snake_case.
- `instructions` and multi-line fields are `CLOB`/`VARCHAR(2000)` stored as `String`.
- `toLine` stores multiple addressees joined by `\n`; the controller splits it for the print view.
- `completed BOOLEAN DEFAULT FALSE` and `form_type VARCHAR(20)` columns exist but completing is not exposed in the UI.
- When adding a new field: update `schema.sql`, `TrainOrder.java`, `TrainOrderRepository` (MAPPER + INSERT + UPDATE + `params()`), both templates, and the entry form.

## Repository Patterns

- Always pass the key-column array to `jdbc.update(sql, params, keys, new String[]{"id"})` — H2 returns all generated columns otherwise and `getKey()` throws.
- The `update()` SQL has `AND completed = FALSE` as a safety guard.
- `nextOrderNumberForToday()` uses `CAST(created_at AS DATE) = CURRENT_DATE` (not `DATE()`).

## Test Annotation Packages (Spring Boot 4 — differ from SB3)

```java
@JdbcTest   → org.springframework.boot.jdbc.test.autoconfigure.JdbcTest
@WebMvcTest → org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest
@MockitoBean → org.springframework.test.context.bean.override.mockito.MockitoBean
```

Test slice starters in `pom.xml`: `spring-boot-starter-jdbc-test`, `spring-boot-starter-webmvc-test`.

## Test Suite (13 tests total)

| Class | Tests | Scope |
|---|---|---|
| `TrainOrderApplicationTests` | 1 | Full context load |
| `TrainOrderRepositoryTest` | 5 | `@JdbcTest` slice + `@Import(TrainOrderRepository.class)` |
| `HomeControllerTest` | 2 | `@WebMvcTest({HomeController.class, TrainOrderController.class})` |
| `TrainOrderControllerTest` | 5 | `@WebMvcTest(TrainOrderController.class)` |

`HomeControllerTest` includes `TrainOrderController` in its slice so it can test `GET /orders`. It requires `@MockitoBean TrainOrderService service`.

`print()` adds `toLinesList` (a `List<String>`) to the model in addition to `order`; tests for that endpoint must assert `model().attributeExists("order", "toLinesList")`.

## Form 19 Paper Layout (print.html & form.html)

Both templates share the identical `f19-paper` CSS structure. In `form.html`, blank `<span>` elements are replaced by `<input type="text" th:field="*{…}">` with the same `f19-blank` CSS classes. The "To" section supports dynamic multi-row entry via vanilla JS (`addToLine` / `removeToLine`). The `no-print` CSS class hides elements from `@media print`.

