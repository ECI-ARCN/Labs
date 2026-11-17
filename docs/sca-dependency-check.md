# Análisis de Dependencias (SCA) con OWASP Dependency-Check

**Duración estimada:** 2 horas (la primera descarga de la base de datos de la NVD toma buena parte de la sesión).

**Prerrequisitos:**

- Haber completado el laboratorio [Análisis estático (Java)](static-code-java.md): fork público de `spring-petclinic`, `devcontainer.json` con Java 21 y el workflow `.github/workflows/sonarqube-analysis.yml` funcionando.
- Cuenta de GitHub con acceso a Codespaces.
- Una dirección de correo para recibir la API key de la NVD.

**Qué vas a construir:** un segundo job en el workflow existente que revisa las librerías de PetClinic contra la base de datos pública de vulnerabilidades, publica los hallazgos en la pestaña **Security** de GitHub y falla cuando encuentra una vulnerabilidad grave.

---

## **1. Conceptos Mínimos**

- **SCA (Software Composition Analysis):** analiza las librerías de terceros que usa tu proyecto, no tu código. Responde: ¿alguna de mis dependencias tiene vulnerabilidades conocidas?
- **CVE (Common Vulnerabilities and Exposures):** identificador único de una vulnerabilidad publicada, por ejemplo `CVE-2022-42889`.
- **CPE (Common Platform Enumeration):** nombre estandarizado de un producto y su versión, por ejemplo `cpe:2.3:a:apache:commons_text:1.9`. Las vulnerabilidades de la NVD se asocian a CPE, no a coordenadas Maven.
- **CVSS (Common Vulnerability Scoring System):** puntaje de gravedad de 0 a 10. De 7.0 a 8.9 es *High*; de 9.0 a 10, *Critical*.
- **NVD (National Vulnerability Database):** base de datos del NIST que publica los CVE con sus CPE y puntajes CVSS. Es la fuente principal de Dependency-Check.

**Dependencias transitivas.** PetClinic declara unas 20 dependencias en su `pom.xml`, pero el build descarga más de 100: las librerías de tus librerías. Una vulnerabilidad en cualquiera de ellas también es tuya, aunque nunca la hayas declarado. Para verlas todas:

```bash
./mvnw dependency:tree
```

**¿Cómo funciona Dependency-Check?** Por cada JAR reúne *evidencia* (nombre del archivo, manifiesto, `pom.properties`), deduce el CPE más probable y busca los CVE asociados. Como el CPE se deduce, puede equivocarse: de ahí los falsos positivos de la sección 6.

> **SAST frente a SCA.** El laboratorio anterior fue **SAST** (*Static Application Security Testing*): SonarQube Cloud leyó *tu* código fuente en busca de defectos. Este laboratorio es **SCA**: no lee tu código, compara las versiones de *tus dependencias* con una lista de vulnerabilidades publicadas. Son complementarios: un código impecable puede ser vulnerable por una librería, y una librería al día no corrige un error tuyo.

---

## **2. Preparación**

### Solicitar la API Key de la NVD
1. Abre <https://nvd.nist.gov/developers/request-an-api-key>.
2. Completa el formulario con tu organización, tu correo y el tipo de organización.
3. Abre el correo que recibes y activa la key con el enlace (caduca si no la activas a tiempo).
4. Copia la key: la necesitas en los dos pasos siguientes.

Sin key, la NVD limita tanto las peticiones que la descarga inicial puede tardar horas.

### Guardar la Key como Secreto de Actions
1. En tu fork: **Settings > Secrets and variables > Actions > New repository secret**.
2. Nombre: `NVD_API_KEY`. Valor: la key.

### Guardar la Key como Secreto de Codespaces
1. En GitHub, ve a tu perfil: **Settings > Codespaces > Codespaces secrets > New secret**.
2. Nombre: `NVD_API_KEY`. Valor: la key.
3. En **Repository access**, selecciona `TU_USUARIO_GITHUB/spring-petclinic`.
4. Si el Codespace ya estaba abierto, recárgalo: en la paleta de comandos, ejecuta **Codespaces: Rebuild Container**. Los secretos solo se inyectan al iniciar el contenedor.
5. Verifica en la terminal que la variable existe, sin imprimir su valor:

```bash
[ -n "$NVD_API_KEY" ] && echo "NVD_API_KEY disponible" || echo "NVD_API_KEY NO disponible"
```

> **La key nunca va en el `pom.xml` ni en ningún archivo versionado.** El plugin lee el *nombre* de la variable de entorno (`NVD_API_KEY`) desde el `pom.xml`; el *valor* llega desde el secreto. Así puedes publicar el repositorio sin exponer la key, como hiciste con `SONAR_TOKEN`.

### Iniciar la Descarga de la Base de Datos Ahora
La primera descarga de la NVD tarda varios minutos (a menudo 10 a 30), incluso con key. Lánzala ya, en una terminal aparte, y continúa leyendo mientras termina:

```bash
./mvnw -B org.owasp:dependency-check-maven:13.0.0:update-only \
  -DnvdApiKeyEnvironmentVariable=NVD_API_KEY \
  -DdataDirectory=$HOME/.dependency-check-data
```

La base queda en `~/.dependency-check-data` y las siguientes ejecuciones solo descargan los cambios.

**Valores a reemplazar en esta sección:**

- `TU_USUARIO_GITHUB`: tu usuario de GitHub, dueño del fork.

---

## **3. Ejecución en Codespaces**

### Agregar el Plugin al `pom.xml`
Dentro de `<build><plugins>` del `pom.xml` (no dentro de `<profiles>`), agrega:

```xml
<plugin>
  <groupId>org.owasp</groupId>
  <artifactId>dependency-check-maven</artifactId>
  <version>13.0.0</version>
  <configuration>
    <!-- Nombre de la variable de entorno con la key (no la key) -->
    <nvdApiKeyEnvironmentVariable>NVD_API_KEY</nvdApiKeyEnvironmentVariable>
    <!-- Base de datos fuera de ~/.m2 para cachearla por separado -->
    <dataDirectory>${user.home}/.dependency-check-data</dataDirectory>
    <formats>
      <format>HTML</format>
      <format>SARIF</format>
    </formats>
    <!-- Falla con cualquier vulnerabilidad High o Critical -->
    <failBuildOnCVSS>7</failBuildOnCVSS>
    <!-- OSS Index exige cuenta en Sonatype; se desactiva -->
    <ossIndexAnalyzerEnabled>false</ossIndexAnalyzerEnabled>
  </configuration>
</plugin>
```

- **Sin `<executions>`:** el plugin no se ata a ninguna fase del build. `./mvnw verify` (y el job de Sonar) no lo ejecuta; solo corre cuando lo invocas con `dependency-check:check`.
- **`dataDirectory`:** por defecto la base vive dentro de `~/.m2/repository`. Moverla evita que termine dentro de la caché de Maven del workflow, que comparte el job de Sonar.
- **`failBuildOnCVSS`:** su valor por defecto es 11, es decir, nunca falla. Con 7, cualquier CVE *High* o *Critical* rompe el build.

> **Sobre OSS Index.** Dependency-Check también consulta Sonatype OSS Index, que ahora exige usuario y token. Se desactiva para no depender de otra cuenta. El costo: se pierde una segunda fuente de vulnerabilidades, así que el análisis queda limitado a la NVD (más el catálogo de vulnerabilidades explotadas de CISA, que el plugin consulta por defecto).

### Ejecutar el Análisis
Cuando termine la descarga de la sección 2:

```bash
./mvnw -B dependency-check:check
```

El resultado queda en `target/dependency-check-report.html` y `target/dependency-check-report.sarif`.

Si el build termina en `BUILD FAILURE` por CVSS, PetClinic ya tiene alguna dependencia con hallazgos de 7 o más. Revisa el reporte antes de seguir: puede ser un falso positivo (sección 6).

### Abrir el Reporte HTML
1. Sirve la carpeta `target` con el servidor web que trae el JDK:

```bash
jwebserver -d "$PWD/target" -b 0.0.0.0 -p 8000
```

2. Codespaces muestra un aviso de puerto reenviado. Haz clic en **Open in Browser** (o ve a la pestaña **Ports** y abre el puerto 8000).
3. Abre `dependency-check-report.html`.
4. Detén el servidor con `Ctrl+C` cuando termines.

### Leer el Reporte
El reporte tiene un resumen y, debajo, el detalle de cada dependencia.

| Campo | Dónde | Qué significa |
|-------|-------|---------------|
| **Dependency** | Resumen | El archivo JAR analizado (incluye las transitivas). |
| **Vulnerability IDs** | Resumen y detalle | El CPE que Dependency-Check le asignó al JAR. Es una deducción. |
| **Package** | Resumen y detalle | El identificador `pkg:maven/grupo/artefacto@versión`. |
| **Highest Severity** | Resumen | La severidad más alta de sus CVE. |
| **CVE Count** | Resumen | Cuántos CVE se le asociaron. |
| **Confidence** | Resumen | Qué tan seguro está de que el CPE es correcto (*Low* a *Highest*). |
| **Evidence** | Detalle | De dónde sacó el vendor, el producto y la versión: manifiesto, pom, nombre del archivo. |
| **Published Vulnerabilities** | Detalle | Cada CVE con su CVSS, su descripción y enlaces a la NVD. |

Para decidir si un hallazgo es real, compara el CPE con la evidencia: si el CPE describe otro producto, o una versión distinta de la que dice el pom, sospecha un falso positivo.

---

## **4. Pipeline de GitHub Actions**

> **Por qué los resultados van a GitHub y no a SonarQube Cloud.** SonarQube Cloud no admite el plugin comunitario de Dependency-Check que existe para SonarQube Server, y su importación de reportes SARIF no maneja bien los hallazgos de dependencias. En su lugar:
>
> - el reporte **SARIF** se sube a **Security > Code scanning** del repositorio, donde cada CVE queda como alerta con su historial;
> - el reporte **HTML** se guarda como **artefacto** de la ejecución, para revisarlo completo.
>
> SonarQube Cloud sigue a cargo del análisis de tu código (SAST).

### Actualizar el Workflow
Reemplaza el contenido de `.github/workflows/sonarqube-analysis.yml`. El job `sonarqube` queda idéntico al del laboratorio anterior; solo se agrega el job `dependency-check`:

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

  dependency-check:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      - name: Checkout code
        uses: actions/checkout@v7

      - name: Set up JDK 21
        uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Compute cache date
        id: date
        run: echo "today=$(date -u +%Y-%m-%d)" >> "$GITHUB_OUTPUT"

      - name: Restore NVD database
        uses: actions/cache/restore@v6
        with:
          path: ~/.dependency-check-data
          key: ${{ runner.os }}-nvd-${{ steps.date.outputs.today }}
          restore-keys: ${{ runner.os }}-nvd-

      - name: Run OWASP Dependency-Check
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}
        run: ./mvnw -B dependency-check:check

      - name: Save NVD database
        if: always()
        uses: actions/cache/save@v6
        with:
          path: ~/.dependency-check-data
          key: ${{ runner.os }}-nvd-${{ steps.date.outputs.today }}

      - name: Upload SARIF to GitHub Security
        if: always() && hashFiles('target/dependency-check-report.sarif') != ''
        uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: target/dependency-check-report.sarif
          category: dependency-check

      - name: Upload HTML report
        if: always()
        uses: actions/upload-artifact@v7
        with:
          name: dependency-check-report
          path: target/dependency-check-report.html
```

Qué hace cada parte del nuevo job:

- **Job separado:** corre en paralelo con el de Sonar. Si Dependency-Check falla, el análisis de Sonar sigue publicándose.
- **`permissions`:** `security-events: write` permite subir el SARIF. Se declara solo en este job, así que el job de Sonar no cambia.
- **Caché de la NVD en dos pasos (`restore` y `save`):** la acción `actions/cache` normal solo guarda la caché si el job termina bien. Como este job falla a propósito cuando encuentra un CVSS alto, se separa la restauración del guardado, y el guardado lleva `if: always()`.
- **Clave por día:** `restore-keys` recupera la base más reciente y el guardado crea, como máximo, una entrada nueva por día. Si ya existe la del día, el paso avisa que la clave existe y no hace nada.
- **`if: always()` en las subidas:** el reporte se sube aunque el paso anterior haya fallado por el umbral, que es justo cuando más lo necesitas. La condición `hashFiles` evita un segundo error confuso si el análisis falló antes de generar el SARIF.

### Publicar y Verificar
```bash
git add pom.xml .github/workflows/sonarqube-analysis.yml
git commit -m "ci: add OWASP Dependency-Check job"
git push origin main
```

1. En **Actions**, abre la ejecución: deben aparecer los dos jobs, `sonarqube` y `dependency-check`.
2. La primera ejecución de `dependency-check` tarda porque descarga la NVD completa. Las siguientes restauran la caché y tardan pocos minutos.
3. Cuando termine, revisa:
    - **Security > Code scanning:** las alertas de la herramienta *dependency-check*, si las hay.
    - **Summary > Artifacts:** el archivo `dependency-check-report` con el HTML.

> **Verificar:** si `main` ya falla por CVSS, revisa los hallazgos antes del ejercicio. Si son falsos positivos, adelanta la sección 6; si son reales, actualiza la dependencia afectada.

---

## **5. Ejercicio Práctico: Introducir y Remediar una Dependencia Vulnerable**

### Crear la Rama
```bash
git checkout main
git pull
git checkout -b lab/dependencia-vulnerable
```

### Agregar la Dependencia Vulnerable
Dentro de `<dependencies>` del `pom.xml`, agrega:

```xml
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-text</artifactId>
  <version>1.9</version>
</dependency>
```

**CVE esperado:**

| CVE | Nombre | CVSS 3.1 (NVD) | Versiones afectadas |
|-----|--------|----------------|---------------------|
| `CVE-2022-42889` | *Text4Shell*: ejecución remota de código mediante interpolación de variables en `StringSubstitutor` | **9.8 Critical** | 1.5 a 1.9 (corregido en 1.10.0) |

Datos consultados en la NVD. La NVD puede reclasificar o agregar CVE con el tiempo; compáralos con tu reporte.

> **¿Por qué no log4j-core o jackson-databind?** Spring Boot controla las versiones de log4j y Jackson y los usa en su configuración de logging y JSON. Forzar una versión antigua puede romper el arranque o las pruebas de PetClinic y desviar el ejercicio. `commons-text` es independiente: el build sigue en verde y solo cambia el resultado del análisis.

### Probar en el Codespace
```bash
./mvnw -B dependency-check:check
```

Debe terminar en `BUILD FAILURE` con un mensaje que menciona el umbral de CVSS 7 y `CVE-2022-42889`. Abre el reporte HTML (sección 3) y localiza `commons-text-1.9.jar`: revisa su CPE, su evidencia y el CVE.

Fíjate también en `commons-lang3`, dependencia de `commons-text`: su versión la impone Spring Boot y no la que pide `commons-text` 1.9. Así se resuelven las transitivas en Maven.

### Abrir el Pull Request
```bash
git add pom.xml
git commit -m "feat: add commons-text"
git push origin lab/dependencia-vulnerable
```

1. En tu fork, haz clic en **Compare & pull request**.
2. Cambia el repositorio base a `TU_USUARIO_GITHUB/spring-petclinic`, rama `main`. Un PR hacia `spring-projects/spring-petclinic` no recibe tus secretos.
3. Crea el pull request.

### Observar los Resultados
1. En la pestaña **Checks** del PR, el job `dependency-check` aparece en rojo y el de Sonar en verde: el código no cambió, solo las dependencias.
2. Abre el log del paso **Run OWASP Dependency-Check** y localiza el mensaje del umbral.
3. Confirma que los pasos de subida se ejecutaron pese al fallo.
4. Revisa los hallazgos:
    - En el PR, el check **Code scanning results / dependency-check** muestra las alertas nuevas.
    - En **Security > Code scanning**, filtra por la rama del PR (`branch:lab/dependencia-vulnerable` o con el menú **Branch**).
    - En **Summary > Artifacts**, descarga el HTML y búscalo ahí.

> **Verificar:** la forma exacta en que GitHub muestra las alertas en el PR (check aparte, comentario o anotación en `pom.xml`) depende de cómo ubica cada resultado del SARIF. Si no las ves en el PR, búscalas en **Security > Code scanning** filtrando por la rama.

### Remediar
Cambia la versión a la vigente:

```xml
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-text</artifactId>
  <version>1.15.0</version>
</dependency>
```

```bash
./mvnw -B dependency-check:check
git add pom.xml
git commit -m "fix: upgrade commons-text to 1.15.0 (CVE-2022-42889)"
git push origin lab/dependencia-vulnerable
```

1. Confirma que el comando local termina en `BUILD SUCCESS`.
2. En el PR, el job `dependency-check` debe pasar a verde.
3. En **Security > Code scanning**, la alerta de `CVE-2022-42889` aparece como **Fixed** en la rama.
4. Cierra el PR sin hacer merge: PetClinic no usa `commons-text`, y una dependencia sin uso solo suma superficie de ataque.

**Valores a reemplazar en esta sección:**

- `TU_USUARIO_GITHUB`: tu usuario de GitHub, dueño del fork.

---

## **6. Falsos Positivos y Supresiones**

Un **falso positivo** ocurre cuando Dependency-Check asigna a un JAR un CPE equivocado, por ejemplo uno de otro producto con nombre parecido, y le atribuye CVE que no le corresponden. La herramienta deja *suprimirlos* con un archivo XML versionado junto al código.

### Crear el Archivo de Supresión
En la raíz del repositorio, crea `dependency-check-suppression.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.4.xsd">

  <!--
    Motivo: MOTIVO_DE_LA_SUPRESION
    Evidencia: EVIDENCIA_REVISADA
    Revisado por: TU_USUARIO_GITHUB
    Fecha de revisión: FECHA_REVISION
  -->
  <suppress until="FECHA_VENCIMIENTO">
    <notes><![CDATA[
      MOTIVO_DE_LA_SUPRESION
    ]]></notes>
    <packageUrl regex="true">^pkg:maven/GRUPO_ESCAPADO/ARTEFACTO@.*$</packageUrl>
    <vulnerabilityName>CVE_A_SUPRIMIR</vulnerabilityName>
  </suppress>

</suppressions>
```

- **`packageUrl`:** a qué dependencia aplica. Cópialo del campo **Package** del reporte HTML. Los puntos del grupo se escapan con `\.`.
- **`vulnerabilityName`:** suprime solo ese CVE. Para un CPE equivocado completo, usa `<cpe>` en lugar de `<vulnerabilityName>`.
- **`until`:** fecha en formato `AAAA-MM-DDZ` desde la cual la supresión deja de aplicar y el hallazgo vuelve a aparecer. Obliga a revisarla de nuevo.
- **El reporte HTML ayuda:** junto a cada CPE y CVE hay un botón **suppress** que genera el bloque XML para copiar.

### Activar el Archivo en el `pom.xml`
Agrega `suppressionFiles` dentro del `<configuration>` del plugin. El bloque completo queda así:

```xml
<plugin>
  <groupId>org.owasp</groupId>
  <artifactId>dependency-check-maven</artifactId>
  <version>13.0.0</version>
  <configuration>
    <nvdApiKeyEnvironmentVariable>NVD_API_KEY</nvdApiKeyEnvironmentVariable>
    <dataDirectory>${user.home}/.dependency-check-data</dataDirectory>
    <formats>
      <format>HTML</format>
      <format>SARIF</format>
    </formats>
    <failBuildOnCVSS>7</failBuildOnCVSS>
    <ossIndexAnalyzerEnabled>false</ossIndexAnalyzerEnabled>
    <suppressionFiles>
      <suppressionFile>${project.basedir}/dependency-check-suppression.xml</suppressionFile>
    </suppressionFiles>
  </configuration>
</plugin>
```

### Ejercicio: Suprimir con Criterio
1. Revisa el reporte de `main` y elige un hallazgo cuyo CPE no coincida con la evidencia del JAR (otro producto u otra versión).
2. Si el reporte de `main` no tiene ninguno, practica con `CVE-2022-42889` en una rama nueva con `commons-text` 1.9, sabiendo que **no es un falso positivo**. Escribe como motivo una aceptación temporal de riesgo, por ejemplo: *"El proyecto no invoca `StringSubstitutor` con entrada externa; aceptado hasta migrar"*.
3. Completa el archivo y ejecuta `./mvnw -B dependency-check:check`. El hallazgo debe aparecer en la sección **Suppressed Vulnerabilities** del HTML y el build debe pasar.
4. Si usaste `commons-text`, **no hagas merge**: cierra el PR y borra la rama.

Ejemplo completo para el caso de `commons-text`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.4.xsd">

  <!--
    Motivo: aceptación temporal de riesgo, no es falso positivo.
    Evidencia: el proyecto no invoca StringSubstitutor.createInterpolator()
    con entrada externa (búsqueda en el código: sin usos).
    Revisado por: TU_USUARIO_GITHUB
    Fecha de revisión: FECHA_REVISION
  -->
  <suppress until="FECHA_VENCIMIENTO">
    <notes><![CDATA[
      Text4Shell: aceptado temporalmente; el proyecto no usa interpolación
      con entrada externa. Remediar actualizando a commons-text >= 1.10.0.
    ]]></notes>
    <packageUrl regex="true">^pkg:maven/org\.apache\.commons/commons-text@.*$</packageUrl>
    <vulnerabilityName>CVE-2022-42889</vulnerabilityName>
  </suppress>

</suppressions>
```

### Riesgos de Suprimir sin Criterio
- **Una supresión oculta el hallazgo para siempre.** Si la regla es amplia (`regex` que cubre todas las versiones, o un `<cpe>` genérico), también ocultará los CVE futuros de esa librería.
- **Un "falso positivo" puede no serlo.** Que hoy no uses la función vulnerable no significa que mañana alguien no la use.
- **El build verde da falsa tranquilidad.** El pipeline pasa, nadie mira el reporte y la vulnerabilidad llega a producción.
- **Buenas prácticas:**
    - una supresión por CVE y dependencia;
    - motivo y evidencia en el comentario;
    - fecha de vencimiento (`until`);
    - revisión en el pull request como cualquier otro cambio de código.

**Valores a reemplazar en esta sección:**

- `MOTIVO_DE_LA_SUPRESION`: por qué el hallazgo no aplica o se acepta temporalmente.
- `EVIDENCIA_REVISADA`: qué revisaste (CPE frente a la evidencia del JAR, búsqueda de usos en el código).
- `TU_USUARIO_GITHUB`: tu usuario de GitHub.
- `FECHA_REVISION`: fecha de hoy, en formato `AAAA-MM-DD`.
- `FECHA_VENCIMIENTO`: fecha límite de la supresión, en formato `AAAA-MM-DDZ` (por ejemplo, en 3 meses).
- `GRUPO_ESCAPADO`: el groupId con los puntos escapados (`org\.example`).
- `ARTEFACTO`: el artifactId.
- `CVE_A_SUPRIMIR`: el identificador del CVE (`CVE-AAAA-NNNNN`).

---

## **7. Cierre**

### Preguntas de Reflexión
1. Este laboratorio usó `failBuildOnCVSS` en 7. ¿Qué umbral usarías en un equipo real? ¿Debería ser el mismo para una rama de desarrollo que para un release a producción?
2. Dependabot también avisa de dependencias vulnerables y abre PRs de actualización. ¿Qué aporta Dependency-Check que Dependabot no aporte, y viceversa? ¿Usarías los dos?
3. Un CVE con CVSS 9.8 en una librería que tu código nunca invoca, ¿es urgente? ¿Qué información adicional necesitas para decidirlo?
4. ¿Quién debería aprobar un cambio en `dependency-check-suppression.xml`? ¿Qué controles pondrías para que no se convierta en una forma de "apagar" el pipeline?

### Problemas Frecuentes

| Síntoma | Causa y solución |
|---------|------------------|
| `NVD Returned Status Code: 403` | La key no llegó o no está activada. Revisa el secreto `NVD_API_KEY` (en Actions y en Codespaces) y que hayas hecho clic en el enlace de activación. En Codespaces, reconstruye el contenedor después de crear el secreto. |
| La descarga de la NVD tarda mucho o falla con `429` / `503` | La NVD limita peticiones y a veces está saturada. Vuelve a ejecutar más tarde; la descarga continúa desde donde quedó. Si persiste, agrega `<nvdApiDelay>8000</nvdApiDelay>` en la configuración del plugin. |
| Cada ejecución del workflow descarga la NVD completa | La caché está vacía o no se guarda. Revisa en el log que **Save NVD database** se ejecute y que **Restore NVD database** muestre `Cache restored`. Las cachés de un PR no se comparten con otros PR: la de `main` sí está disponible para todos. |
| Errores raros de base de datos tras un fallo | La caché quedó con una base dañada. Bórrala en **Actions > Caches** (las que empiezan por `Linux-nvd-`) y vuelve a ejecutar. |
| El SARIF no aparece en **Security** | Falta `security-events: write` en el job, el paso de subida se saltó porque no se generó el archivo, o la ejecución vino de un PR hacia el repositorio original. Revisa el log del paso **Upload SARIF to GitHub Security**. |
| `Resource not accessible by integration` al subir el SARIF | El job no tiene el permiso `security-events: write`. |
| Advertencias o errores de **OSS Index** (`401`, credenciales) | El analizador sigue activo. Confirma `<ossIndexAnalyzerEnabled>false</ossIndexAnalyzerEnabled>` en el plugin. |
| `Unable to load suppression file` | La ruta en `suppressionFiles` no coincide con el archivo, o el XML está mal formado. Valida el XML y la ruta `${project.basedir}/dependency-check-suppression.xml`. |
| El job de Sonar dejó de funcionar | El cambio de workflow tocó ese job. Compáralo con el del laboratorio anterior: debe quedar idéntico. |
| `./mvnw verify` ejecuta Dependency-Check | Se agregó un bloque `<executions>` al plugin. Quítalo: el análisis se invoca solo con `dependency-check:check`. |

### Reto Opcional
- Activa **Dependabot alerts** y **Dependabot security updates** en **Settings > Advanced Security** y compara sus alertas con las de Dependency-Check sobre la misma rama vulnerable.
- Protege `main` para exigir el check `dependency-check` antes del merge.
