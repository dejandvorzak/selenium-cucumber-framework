# selenium-cucumber-framework

Test automation framework for [saucedemo.com](https://www.saucedemo.com/), built with Selenium, Cucumber (BDD) and TestNG, using the Page Object Model. Test results are reported with Allure.

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
├── core/               # DriverFactory / DriverManager – WebDriver lifecycle
└── pages/              # Page Objects (LoginPage, ProductsPage, CartPage, CheckoutStep*, ...)
    └── components/     # Shared UI components (Header, SideMenu, ErrorBanner)

src/test/java/com/dejan/automation
├── runners/            # TestNGRunner – Cucumber/TestNG entry point
├── stepdefinitions/    # Step definitions per feature area
└── hooks/              # Cucumber hooks (setup/teardown, driver lifecycle)

src/test/resources
├── features/           # Gherkin .feature files (login, product catalog, cart, checkout, navigation)
├── config.properties   # Environment configuration (base URL, browser, timeouts)
└── allure.properties
```

## Configuration

Default settings live in [`src/test/resources/config.properties`](src/test/resources/config.properties):

| Key                    | Default                     | Description                     |
|-------------------------|------------------------------|----------------------------------|
| `baseUrl`               | `https://www.saucedemo.com/` | Application under test          |
| `browser`                | `chrome`                     | Browser to run tests in          |
| `headless`                | `false`                      | Run browser in headless mode     |
| `explicitWaitSeconds`   | `10`                          | Explicit wait                    |

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

## Reports

- **Cucumber HTML/JSON**: generated at `target/cucumber-report/`
- **Allure**: results are written to `target/allure-results/`. Generate and open the report with:

```bash
allure serve target/allure-results
```

(Requires the [Allure commandline](https://allurereport.org/docs/install/) to be installed and on your `PATH`.)
