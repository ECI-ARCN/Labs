# Análisis Estático de Código en Java con SonarQube Cloud

**Duración estimada:** 2 horas.

**Prerrequisitos:**

- Cuenta de GitHub con acceso a Codespaces.
- Manejo básico de Git: ramas, commits, push y pull requests.
- Nociones de Java y Maven (no se requiere experiencia con Sonar ni con GitHub Actions).
- Navegador con sesión iniciada en GitHub.

**Qué vas a construir:** un pipeline que compila [Spring PetClinic](https://github.com/spring-projects/spring-petclinic), ejecuta sus pruebas y envía el análisis a SonarQube Cloud (antes SonarCloud) en cada push y pull request hacia `main`.

> **Versiones de Java.** PetClinic exige Java 17 o superior (`<java.version>17</java.version>` en su `pom.xml`). El escáner de SonarQube Cloud necesita Java 21 o superior. Por eso todo el laboratorio usa JDK 21: compila el proyecto y ejecuta el análisis.

---

## **1. Preparar el Repositorio**

### Hacer el Fork
1. Abre <https://github.com/spring-projects/spring-petclinic> y haz clic en **Fork**.
2. Deja marcada la opción **Copy the `main` branch only** y crea el fork en tu cuenta.
3. En tu fork, ve a la pestaña **Actions** y haz clic en **I understand my workflows, go ahead and enable them**. GitHub desactiva los workflows en los forks hasta que lo autorizas.

### Configurar Codespaces
El repositorio ya trae un `.devcontainer/devcontainer.json`. Reemplaza su contenido desde el editor web de GitHub por el siguiente y haz commit en `main`:

```json
{
    "name": "Java with SonarQube Cloud",
    "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
    "features": {
        "ghcr.io/devcontainers/features/java:1": {
            "version": "21"
        },
        "ghcr.io/devcontainers/features/docker-in-docker:2": {},
        "ghcr.io/devcontainers/features/github-cli:1": {}
    },
    "customizations": {
        "vscode": {
            "extensions": [
                "vscjava.vscode-java-pack",
                "redhat.vscode-xml",
                "SonarSource.sonarlint-vscode"
            ]
        }
    },
    "remoteUser": "vscode"
}
```

- `SonarSource.sonarlint-vscode` es la extensión **SonarQube for IDE** (antes SonarLint); conserva el identificador antiguo.
- `docker-in-docker` es necesario porque algunas pruebas de PetClinic levantan bases de datos en contenedores.

### Abrir el Codespace
1. En tu fork: **Code > Codespaces > Create codespace on main**.
2. Cuando termine de cargar, verifica el entorno en la terminal:

```bash
java -version
./mvnw -B verify
```

La primera compilación tarda varios minutos porque descarga las dependencias. Debe terminar en `BUILD SUCCESS`.

### Revisar `.gitignore`
El repositorio ya ignora `target/` y los archivos de los IDE. No necesitas modificarlo.

---

## **2. Configurar SonarQube Cloud**

### Crear la Organización y el Proyecto
1. Entra a <https://sonarcloud.io> e inicia sesión con **GitHub**.
2. Haz clic en **+ > Analyze new project** e importa tu cuenta de GitHub como organización. Autoriza la aplicación de SonarQube Cloud sobre tu fork.
3. Elige el plan **Free** (suficiente para repositorios públicos).
4. Selecciona el repositorio `spring-petclinic` y haz clic en **Set Up**.
5. Cuando pregunte por la definición de código nuevo (*New Code*), elige **Previous version**.
6. Anota la **organization key** y la **project key**: aparecen en **Information**, en el menú lateral del proyecto.

### Desactivar el Automatic Analysis
1. En el proyecto, ve a **Administration > Analysis Method**.
2. Apaga el interruptor **Automatic Analysis**.

> **¿Por qué?** SonarQube Cloud puede analizar el repositorio por su cuenta, sin compilar. Aquí el análisis lo lanza el pipeline, que sí compila y aporta cobertura. Los dos métodos no pueden convivir: si dejas el automático encendido, el escáner falla en el pipeline.

### Generar el Token
1. Ve a **My Account > Security** (<https://sonarcloud.io/account/security>).
2. Escribe un nombre, por ejemplo `petclinic-github-actions`, y haz clic en **Generate Token**.
3. Copia el valor: no se vuelve a mostrar.

### Guardar el Token como Secreto en GitHub
1. En tu fork: **Settings > Secrets and variables > Actions > New repository secret**.
2. Nombre: `SONAR_TOKEN`. Valor: el token copiado.

> **El token nunca va en un archivo del repositorio.** Ni en `pom.xml`, ni en `sonar-project.properties`, ni en el workflow. Un token en el historial de Git queda expuesto para siempre, aunque borres el archivo después, y en un repositorio público cualquiera puede usarlo para publicar análisis en tu nombre. Como secreto, GitHub lo cifra y lo oculta en los logs.

### Agregar la Configuración al `pom.xml`
Dentro del bloque `<properties>` del `pom.xml`, agrega estas tres líneas:

```xml
<properties>
    <!-- ... propiedades existentes ... -->

    <!-- SonarQube Cloud -->
    <sonar.organization>TU_ORGANIZACION_SONAR</sonar.organization>
    <sonar.projectKey>TU_PROJECT_KEY</sonar.projectKey>
    <sonar.host.url>https://sonarcloud.io</sonar.host.url>
</properties>
```

Estos valores son identificadores públicos, no credenciales: pueden vivir en el repositorio.

**Valores a reemplazar en esta sección:**

- `TU_ORGANIZACION_SONAR`: la organization key (normalmente, tu usuario de GitHub en minúsculas).
- `TU_PROJECT_KEY`: la project key (normalmente, `usuario_spring-petclinic`).

---

## **3. Configurar el Pipeline de GitHub Actions**

### Crear el Archivo de Workflow
Guarda el siguiente contenido en `.github/workflows/sonarqube-analysis.yml`:

```yaml
name: SonarQube Cloud Analysis

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  sonarqube:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v7
        with:
          fetch-depth: 0

      - name: Set up JDK 21
        uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Cache SonarQube packages
        uses: actions/cache@v6
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: ${{ runner.os }}-sonar

      - name: Build, test and analyze
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: ./mvnw -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar
```

Qué hace cada parte:

- **`fetch-depth: 0`** descarga todo el historial. Sonar lo necesita para saber qué líneas son nuevas y quién las escribió.
- **`cache: maven`** guarda `~/.m2` entre ejecuciones para no descargar las dependencias cada vez.
- **`verify`** compila, ejecuta las pruebas y genera el reporte de cobertura de JaCoCo, que PetClinic ya trae configurado.
- **`sonar-maven-plugin:sonar`** es el SonarScanner for Maven. Lee la configuración del `pom.xml` y el token de la variable `SONAR_TOKEN`. No hay que descargar ningún escáner a mano.

### Publicar y Verificar
```bash
git add pom.xml .github/workflows/sonarqube-analysis.yml
git commit -m "ci: add SonarQube Cloud analysis"
git push origin main
```

1. Abre la pestaña **Actions** de tu fork y espera a que **SonarQube Cloud Analysis** termine en verde.
2. Abre el proyecto en SonarQube Cloud: debe aparecer el primer análisis de `main`.

> El fork trae otros workflows (`maven-build.yml`, `gradle-build.yml`, `deploy-and-test-cluster.yml`). Puedes ignorarlos o desactivarlos en **Actions > (workflow) > ⋯ > Disable workflow**.

---

## **4. Ejercicio Práctico: Introducir y Corregir Defectos**

PetClinic es código limpio, así que vas a ensuciarlo a propósito.

### Crear la Rama
```bash
git checkout -b lab/defectos
mkdir -p src/main/java/org/springframework/samples/petclinic/lab
```

### Agregar la Clase con Defectos
Crea `src/main/java/org/springframework/samples/petclinic/lab/OwnerReport.java`:

```java
package org.springframework.samples.petclinic.lab;

import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

public class OwnerReport {

	// Defecto 1: credencial en el código
	private static final String PASSWORD = "Petclinic2024!";

	private final Map<String, String> phones = new HashMap<>();

	public void addPhone(String owner, String phone) {
		phones.put(owner, phone);
	}

	public boolean isAdmin(String user, String secret) {
		return "admin".equals(user) && PASSWORD.equals(secret);
	}

	// Defecto 2: recurso sin cerrar
	public String readFirstLine(String path) throws IOException {
		BufferedReader reader = new BufferedReader(new FileReader(path));
		return reader.readLine();
	}

	// Defecto 3: posible NullPointerException
	public int phoneLength(String owner) {
		String phone = null;
		if (phones.containsKey(owner)) {
			phone = phones.get(owner);
		}
		return phone.length();
	}

	// Defecto 4: excepción silenciada
	public int parseAge(String value) {
		int age = 0;
		try {
			age = Integer.parseInt(value);
		}
		catch (NumberFormatException ex) {
		}
		return age;
	}

	// Defecto 5: código duplicado (formatOwner y formatVet son idénticos)
	public String formatOwner(String firstName, String lastName, String city) {
		StringBuilder line = new StringBuilder();
		line.append(lastName.toUpperCase());
		line.append(", ");
		line.append(firstName);
		line.append(" (");
		line.append(city);
		line.append(")");
		return line.toString();
	}

	public String formatVet(String firstName, String lastName, String city) {
		StringBuilder line = new StringBuilder();
		line.append(lastName.toUpperCase());
		line.append(", ");
		line.append(firstName);
		line.append(" (");
		line.append(city);
		line.append(")");
		return line.toString();
	}

}
```

Regla que debería activar cada defecto:

| # | Defecto | Regla esperada | Tipo |
|---|---------|----------------|------|
| 1 | Credencial en el código | `java:S2068` — las contraseñas escritas en el código son sensibles | Security Hotspot |
| 2 | Recurso sin cerrar | `java:S2095` — los recursos deben cerrarse | Bug |
| 3 | Posible NullPointerException | `java:S2259` — no se debe desreferenciar un valor nulo | Bug |
| 4 | Excepción silenciada | `java:S108` — los bloques de código no deben quedar vacíos | Code Smell |
| 5 | Código duplicado | `java:S4144` — los métodos no deben tener implementaciones idénticas | Code Smell |

> **Verificar:** las reglas dependen del perfil de calidad activo (*Sonar way*) y pueden cambiar entre versiones. Compara esta tabla con lo que reporte tu análisis; es normal que aparezcan hallazgos adicionales.

### Dar Formato y Publicar
PetClinic valida el formato del código en cada compilación. Aplícalo antes del commit:

```bash
./mvnw spring-javaformat:apply
git add src/main/java/org/springframework/samples/petclinic/lab
git commit -m "feat: add owner report"
git push origin lab/defectos
```

### Abrir el Pull Request
1. En tu fork, haz clic en **Compare & pull request**.
2. **Cambia el repositorio base.** GitHub propone `spring-projects/spring-petclinic`; selecciona `TU_USUARIO_GITHUB/spring-petclinic` con base `main`.
3. Crea el pull request.

> Un PR hacia el repositorio original no tiene acceso a tu secreto `SONAR_TOKEN` y molestaría a los mantenedores de PetClinic.

### Observar los Hallazgos
1. Espera a que el workflow termine. El paso de Maven pasa en verde: el código compila y las pruebas existentes pasan.
2. Revisa el comentario que SonarQube Cloud deja en el pull request y el check **SonarCloud Code Analysis**: el Quality Gate debe aparecer como **Failed**.
3. Abre el enlace al análisis y responde:
    - ¿Qué defectos encontró y cuáles no?
    - ¿Qué condiciones del Quality Gate fallaron?
    - ¿Qué cobertura tiene el código nuevo?

### Corregir los Defectos
Reemplaza el contenido de `OwnerReport.java`:

```java
package org.springframework.samples.petclinic.lab;

import java.io.BufferedReader;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.HashMap;
import java.util.Map;

public class OwnerReport {

	private final Map<String, String> phones = new HashMap<>();

	private final String adminSecret;

	// Corrección 1: el secreto llega desde fuera (variable de entorno, vault...)
	public OwnerReport(String adminSecret) {
		this.adminSecret = adminSecret;
	}

	public void addPhone(String owner, String phone) {
		phones.put(owner, phone);
	}

	public boolean isAdmin(String user, String secret) {
		return "admin".equals(user) && adminSecret != null && adminSecret.equals(secret);
	}

	// Corrección 2: try-with-resources cierra el lector
	public String readFirstLine(String path) throws IOException {
		try (BufferedReader reader = Files.newBufferedReader(Path.of(path))) {
			return reader.readLine();
		}
	}

	// Corrección 3: se controla el caso sin teléfono
	public int phoneLength(String owner) {
		String phone = phones.get(owner);
		return (phone != null) ? phone.length() : 0;
	}

	// Corrección 4: la excepción se propaga con contexto
	public int parseAge(String value) {
		try {
			return Integer.parseInt(value);
		}
		catch (NumberFormatException ex) {
			throw new IllegalArgumentException("Edad no válida: " + value, ex);
		}
	}

	// Corrección 5: una sola implementación reutilizada
	public String formatOwner(String firstName, String lastName, String city) {
		return formatPerson(firstName, lastName, city);
	}

	public String formatVet(String firstName, String lastName, String city) {
		return formatPerson(firstName, lastName, city);
	}

	private String formatPerson(String firstName, String lastName, String city) {
		return lastName.toUpperCase() + ", " + firstName + " (" + city + ")";
	}

}
```

### Agregar Pruebas
El Quality Gate también exige cobertura sobre el código nuevo, así que corregir los defectos no basta. Crea `src/test/java/org/springframework/samples/petclinic/lab/OwnerReportTests.java`:

```java
package org.springframework.samples.petclinic.lab;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

class OwnerReportTests {

	private final OwnerReport report = new OwnerReport("s3cret");

	@Test
	void adminRequiresMatchingSecret() {
		assertThat(report.isAdmin("admin", "s3cret")).isTrue();
		assertThat(report.isAdmin("admin", "other")).isFalse();
		assertThat(report.isAdmin("guest", "s3cret")).isFalse();
	}

	@Test
	void readsFirstLine(@TempDir Path dir) throws IOException {
		Path file = Files.writeString(dir.resolve("owners.txt"), "first\nsecond\n");
		assertThat(report.readFirstLine(file.toString())).isEqualTo("first");
	}

	@Test
	void phoneLengthIsZeroWhenOwnerHasNoPhone() {
		report.addPhone("george", "6085551023");
		assertThat(report.phoneLength("george")).isEqualTo(10);
		assertThat(report.phoneLength("nobody")).isZero();
	}

	@Test
	void parseAgeRejectsInvalidValues() {
		assertThat(report.parseAge("42")).isEqualTo(42);
		assertThatThrownBy(() -> report.parseAge("abc")).isInstanceOf(IllegalArgumentException.class);
	}

	@Test
	void formatsOwnersAndVets() {
		assertThat(report.formatOwner("George", "Franklin", "Madison")).isEqualTo("FRANKLIN, George (Madison)");
		assertThat(report.formatVet("James", "Carter", "Sun Prairie")).isEqualTo("CARTER, James (Sun Prairie)");
	}

}
```

### Verificar que el Quality Gate Pase
```bash
./mvnw spring-javaformat:apply
./mvnw -B verify
git add src
git commit -m "fix: resolve SonarQube findings in owner report"
git push origin lab/defectos
```

1. El push actualiza el pull request y dispara el workflow de nuevo.
2. Confirma que el Quality Gate cambia a **Passed** y que los hallazgos anteriores desaparecen.
3. Haz merge del pull request y revisa el análisis de `main`.

**Valores a reemplazar en esta sección:**

- `TU_USUARIO_GITHUB`: tu usuario de GitHub, dueño del fork.

---

## **5. Interpretar los Resultados**

### Qué Mirar en el Tablero

| Métrica | Qué significa | Pregunta que responde |
|---------|---------------|------------------------|
| **Bugs** | Código que está mal o fallará en ejecución | ¿Esto se va a romper? |
| **Vulnerabilities** | Código que un atacante puede explotar | ¿Esto es inseguro con certeza? |
| **Security Hotspots** | Código sensible que una persona debe revisar | ¿Esto podría ser inseguro? |
| **Code Smells** | Código que funciona pero es difícil de mantener | ¿Cuánto costará cambiar esto? |
| **Duplications** | Porcentaje de líneas repetidas | ¿Cuánto código sobra? |
| **Coverage** | Porcentaje de líneas ejecutadas por las pruebas | ¿Qué tan protegido está el código? |

- Las versiones recientes del tablero agrupan los hallazgos por atributo: **Reliability** (bugs), **Security** (vulnerabilidades) y **Maintainability** (code smells). Son las mismas categorías con otro nombre.
- Cada categoría recibe una calificación de **A** a **E**.
- Un Security Hotspot no es un defecto confirmado: se cierra corrigiendo el código o marcándolo como revisado (*Safe*).

### Código Nuevo y Código Total
El tablero separa **New Code** de **Overall Code**. En un pull request, el código nuevo son las líneas que el PR cambia. La idea es no empeorar: se exige calidad a lo que entra, sin tener que arreglar primero todo lo heredado.

### Qué es el Quality Gate
Es un conjunto de condiciones que el código debe cumplir para considerarse apto. Responde con **Passed** o **Failed** a una sola pregunta: ¿se puede integrar este cambio?

El Quality Gate por defecto, *Sonar way*, evalúa el código nuevo: sin problemas nuevos de fiabilidad, seguridad o mantenibilidad, todos los hotspots revisados, cobertura mínima del 80 % y duplicación máxima del 3 %.

> **Verificar:** consulta las condiciones exactas en **Quality Gates**, en el menú de tu organización; pueden variar con el tiempo.

---

## **6. Cierre**

### Preguntas de Reflexión
1. El workflow terminó en verde aunque el Quality Gate falló. ¿Qué habría que cambiar para que un Quality Gate fallido impida el merge? ¿Conviene hacerlo siempre?
2. ¿Por qué la credencial apareció como Security Hotspot y no como Vulnerability? ¿Qué implica esa diferencia para el equipo?
3. ¿Qué defectos reales no puede encontrar el análisis estático, por bien configurado que esté?
4. ¿Qué ventaja tiene evaluar solo el código nuevo frente a exigir que todo el proyecto cumpla el Quality Gate?

### Problemas Frecuentes

| Síntoma | Causa y solución |
|---------|------------------|
| El workflow no se ejecuta | Los workflows del fork están desactivados. Actívalos en la pestaña **Actions**. |
| `You are running CI analysis while Automatic Analysis is enabled` | Apaga **Automatic Analysis** en **Administration > Analysis Method**. |
| `Not authorized` o `Project not found` | El secreto `SONAR_TOKEN` no existe o está mal copiado, o `sonar.organization` / `sonar.projectKey` no coinciden con los de SonarQube Cloud. |
| La compilación falla en `spring-javaformat:validate` | El código no cumple el formato del proyecto. Ejecuta `./mvnw spring-javaformat:apply` y vuelve a hacer commit. |
| La compilación falla en `checkstyle` con `NoHttp` | Algún archivo contiene una URL con `http://`. Usa `https://`. |
| El escáner se queja de la versión de Java | El workflow no usa JDK 21. Revisa el paso **Set up JDK 21**. |
| El análisis del PR falla por falta de token | El PR apunta a `spring-projects/spring-petclinic`. Ciérralo y ábrelo contra tu fork. |
| El Quality Gate sigue en **Failed** tras corregir | Revisa la cobertura del código nuevo y los hotspots sin revisar. |
| El Quality Gate de `main` aparece como **Not computed** | Falta definir el código nuevo: **Administration > New Code**. Se calcula en el siguiente análisis. |
| Fallan pruebas en el Codespace por Docker | Reconstruye el contenedor (**Codespaces: Rebuild Container**) para que cargue `docker-in-docker`. |

### Reto Opcional
- Configura **SonarQube for IDE** en *Connected Mode* con tu proyecto de SonarQube Cloud y comprueba que los defectos de la sección 4 se marcan en el editor antes del commit.
- Protege la rama `main` para exigir el check de SonarQube Cloud antes del merge.
