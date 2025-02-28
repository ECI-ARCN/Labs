# Laboratorio de BDD con Selenium, ChromeDriver y Java en GitHub Codespaces

## 1. Crear un nuevo repositorio en GitHub
- Ve a GitHub y crea un nuevo repositorio.
- Dale un nombre y una descripción a tu repositorio.

## 2. Configurar GitHub Codespaces
- Crear un nuevo archivo
- En nombre de archivo ingresar ".devcontainer/devcontainer.json"
- Pegar la siguiente configuración

```json
{
  "name": "BDD Lab with Selenium and Java",
  "image": "mcr.microsoft.com/devcontainers/java:17",
  "settings": {
    "terminal.integrated.shell.linux": "/bin/bash"
  },
  "features": {
    "ghcr.io/devcontainers/features/java:1": {
      "version": "17"
    },
    "ghcr.io/devcontainers-extra/features/maven-sdkman:2": {},
    "ghcr.io/devcontainers/features/node:1": {
      "version": "lts"
    },
    "ghcr.io/kreemer/features/chrometesting:1": {}
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "vscjava.vscode-java-pack",
        "vscjava.vscode-maven",
        "dbaeumer.vscode-eslint",
        "ms-python.python",
        "ms-python.vscode-pylance"
      ]
    }
  },
  "forwardPorts": [8080],
  "portsAttributes": {
    "8080": {
      "label": "Application",
      "onAutoForward": "notify"
    }
  },
  "postCreateCommand": "sudo apt update",
  "remoteUser": "vscode"
}
```
- Confirmar los cambios "**Commit changes...**"
- Agrega un comentario y "**Commit changes**"

## 3. Crear Codespace
- Volver a la carpeta base de tu repositorio haciendo click en "**<> Code**" ubicado en la parte superior izquierda.
- Hacer click en el botón **<> code**, pestaña Codespaces y click en **Create codespace on main**.


## 4. Creación del Proyecto Maven

Para desarrollar el laboratorio, primero debemos crear un proyecto Maven. Ejecuta el siguiente comando, reemplazando las variables -DgroupId y -DartifactId:

```sh
mvn archetype:generate -DgroupId={com.eci.myproject} -DartifactId={bdd-java} -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

Este comando generará la estructura básica del proyecto con las carpetas `src/main/java` y `src/test/java`.

## 4. Agregar dependencias de Selenium y Cucumber
- Crea un archivo `pom.xml` en la raíz de tu proyecto y agrega las dependencias necesarias para Selenium, ChromeDriver y Cucumber:

```xml
  <dependencies>
    <dependency>
        <groupId>org.seleniumhq.selenium</groupId>
        <artifactId>selenium-java</artifactId>
        <version>4.0.0</version>
    </dependency>
    <!-- Cucumber -->
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-java</artifactId>
        <version>7.0.0</version>
    </dependency>
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-junit</artifactId>
        <version>7.0.0</version>
        <scope>test</scope>
    </dependency>
  </dependencies>
```

## 5. Crear la estructura BDD
- Dentro de la carpeta src/test/java, generar las siguientes carpetas:
    - features
    - steps
    - runners

## 6. Crear escenario BDD
- Dentro de la carpeta **features**, crear un archivo con el nombre "**google_search.feature**"
- Agregar el siguiente texto

```gherkin
Feature: Google Search

  Scenario: Search for a term
    Given I am on the Google search page
    When I search for "GitHub"
    Then I should see "GitHub" in the results
```
- Dentro de la carpeta **steps**, crear el archivo **SearchSteps.java**
- Pegar el siguiente código

```java
import io.cucumber.java.After;
import io.cucumber.java.Before;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.concurrent.TimeUnit;

public class SearchSteps {
    private WebDriver driver;

    @Before
    public void setUp() {
        try {
            System.setProperty("webdriver.chrome.driver", "/usr/local/bin/chromedriver");
            ChromeOptions options = new ChromeOptions();
            options.addArguments("--headless");
            options.addArguments("--disable-gpu");
            options.addArguments("--no-sandbox");
            options.addArguments("--disable-dev-shm-usage");
            options.addArguments("--remote-allow-origins=*");
            driver = new ChromeDriver(options);
            driver.manage().timeouts().implicitlyWait(10, TimeUnit.SECONDS);
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("Failed to initialize ChromeDriver", e);
        }
    }

    @Given("I am on the Google search page")
    public void i_am_on_the_google_search_page() {
        driver.get("https://www.google.com");
    }

    @When("I search for {string}")
    public void i_search_for(String term) {
        WebElement searchBox = driver.findElement(By.name("q"));
        searchBox.sendKeys(term);
        searchBox.submit();
    }

    @Then("I should see {string} in the results")
    public void i_should_see_in_the_results(String term) {
        assert driver.getPageSource().contains(term);
    }

    @After
    public void tearDown() {
        driver.quit();
    }
}
```

- Dentro de la carpeta **runners**, crear el archivo **TestRunner.java**
- Pegar el siguiente código

```java
import org.junit.runner.RunWith;
import io.cucumber.junit.Cucumber;
import io.cucumber.junit.CucumberOptions;


@RunWith(Cucumber.class)
@CucumberOptions(features = "src/test/java/features", glue="steps",
monochrome = true,
publish = true,
plugin = {"pretty", "junit:target/JUnitReports/report.xml",
		"json:target/JSonReports/report.json",
		"html:target/HtmlReports/report.html"
		}
)
public class TestRunner {
}
```
- ejecutar los escenarios con 

```bash
mvn test
```
- validar el reporte html, descargando el archivo "target/HtmlReports/report.html"