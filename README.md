# selenium-cucumber-framework

A compact reference implementation of a Selenium + Cucumber test automation
framework, written against [saucedemo.com](https://www.saucedemo.com/).

The architecture reflects patterns from production work on a large-scale BDD
suite (130+ page objects, 6,800+ Cucumber scenarios), where the design
decisions below were made under real constraints rather than in principle.

## Tech stack

- Java 17
- Maven
- Selenium WebDriver 4.27
- Cucumber 7 (`cucumber-java`, `cucumber-testng`)
- TestNG 7 (test runner, parallel scenario execution)
- WebDriverManager (automatic driver binaries)
- Allure 2 (test reporting)

## Project structure

```
src/main/java/com/dejan/automation
├── config/            # ConfigReader – reads config.properties, overridable via -D system properties
├── core/              # DriverFactory / DriverManager – WebDriver lifecycle
└── pages/             # Page Objects (LoginPage, ProductsPage, CartPage, CheckoutStep*, ...)
    └── components/    # Shared UI components (Header, SideMenu, ErrorBanner)

src/test/java/com/dejan/automation
├── runners/           # TestNGRunner – Cucumber/TestNG entry point
├── stepdefinitions/   # Step definitions per feature area
└── hooks/             # Cucumber hooks (setup/teardown, driver lifecycle)

src/test/resources
├── features/          # Gherkin .feature files (login, product catalog, cart, checkout, navigation)
├── config.properties  # Environment configuration (base URL, browser, timeouts)
└── allure.properties
```

## Configuration

Default settings live in [`src/test/resources/config.properties`](src/test/resources/config.properties):

| Key                  | Default                      | Description                    |
|----------------------|------------------------------|--------------------------------|
| `baseUrl`            | `https://www.saucedemo.com/` | Application under test         |
| `browser`            | `chrome`                     | Browser to run tests in        |
| `headless`           | `false`                      | Run browser in headless mode   |
| `explicitWaitSeconds`| `10`                         | Explicit wait timeout          |

Any value can be overridden at runtime with a `-D` system property, e.g. `-Dbrowser=firefox -Dheadless=true`.

## Running the tests

Run the full suite (driven by `testng.xml`):

```bash
mvn test
```

Run headless in a specific browser:

```bash
mvn test -Dbrowser=chrome -Dheadless=true
```

Run a subset of scenarios by Cucumber tag (e.g. only `@smoke`):

```bash
mvn test -Dcucumber.filter.tags="@smoke"
```

Available tags include `@smoke` and `@regression` — check the `.feature` files under [`src/test/resources/features`](src/test/resources/features) for the full list.

## CI pipeline (Jenkins + Docker)

The [`Jenkinsfile`](Jenkinsfile) defines a three-stage pipeline:

1. **Build** — `mvn -B -DskipTests clean compile`
2. **Test – Chrome** — full suite headless, JUnit results + Allure report published
3. **Test – Firefox** — same suite on Firefox, separate Allure report

Key pipeline details:
- Polls GitHub every 5 minutes for new commits (`pollSCM`)
- Keeps the last 20 builds (`buildDiscarder`)
- Hard timeout of 30 minutes per run
- Each stage runs in an isolated Docker container built from [`Dockerfile.jenkins-agent`](Dockerfile.jenkins-agent)
- `--shm-size=2g` on the container prevents Chrome from crashing on memory-intensive pages
- Chrome and Firefox run sequentially (not parallel) to stay within a single-agent resource budget; `data-provider-thread-count="2"` in `testng.xml` still parallelises data-driven scenarios within each browser run
- Allure results are archived and published as HTML — survives workspace cleanup between builds

The Docker image bakes in Chrome stable, Firefox ESR and the Allure CLI, so the Jenkins agent needs no pre-installed browsers or test tooling.

## Design decisions

**`ThreadLocal<WebDriver>` with `remove()` in a `finally` block**

`DriverManager` stores the driver in a `ThreadLocal` and calls `DRIVER.remove()` inside `quitDriver()`, which is invoked from the `@After` hook's `finally`-equivalent path. Without the `remove()`, a thread that finishes one scenario and is reused for the next one would pick up the stale, already-quit driver on its next `getDriver()` call. On a suite running scenarios in parallel this produces intermittent `SessionNotCreatedException` failures that are difficult to reproduce and impossible to trace to a root cause without knowing to look here.

**No `Thread.sleep` anywhere in the framework**

`BasePage` exposes `waitForVisible`, `waitForClickable` and `isVisible` — all backed by `WebDriverWait` with an `ExpectedCondition`. Pages and step definitions call these methods rather than sleeping. A fixed sleep of 2 seconds is either too short (flaky on slow CI) or too long (slows every run). An explicit wait returns the moment the condition is met, which means it is both more reliable and faster in practice.

**Screenshot only on a failed scenario**

The `@After` hook checks `scenario.isFailed()` before taking a screenshot. Attaching a screenshot to every scenario quadruples report size and makes Allure slow to render on large suites. A screenshot attached only to failed scenarios is always the one that matters and is immediately visible in the report without scrolling.

**`components/` package separate from page objects**

`Header` and `SideMenu` are used across every page in the application. Duplicating their locators inside each page object means updating the same XPath in a dozen files when the layout changes. Extracting them into a `components/` package makes each a single source of truth. On a production suite with 130+ page objects, this distinction becomes significant — a one-line locator change in a component propagates everywhere.

**`ConfigReader` with `-D` override**

Every configurable value — URL, browser, timeouts — is read through `ConfigReader`, which checks for a system property before falling back to `config.properties`. This means CI can switch browsers or environments with a single flag (`-Dbrowser=firefox`) without touching committed configuration, and local development uses sensible defaults without any setup.

## Reports

- **Cucumber HTML/JSON**: generated at `target/cucumber-report/`
- **Allure**: results are written to `target/allure-results/`. Generate and open the report with:

```bash
allure serve target/allure-results
```

(Requires the [Allure commandline](https://allurereport.org/docs/install/) to be installed and on your `PATH`.)
