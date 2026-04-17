# Laboratorio: SDD & Multi-Agent Orchestration en Codespaces

## 1. Configuración del Entorno (.devcontainer)
Para que el laboratorio funcione, el repositorio de GitHub debe tener una carpeta .devcontainer/ con el archivo devcontainer.json. Esto instalará automáticamente las herramientas necesarias.

Configuración sugerida:

'''
{
  "name": "NET 10 SDD Lab",
  "image": "mcr.microsoft.com/devcontainers/dotnet:10.0",
  "customizations": {
    "vscode": {
      "extensions": [
        "saoudrizwan.claude-dev", // Extensión de Cline
        "ms-dotnettools.csdevkit",
        "Codeium.codeium"
      ]
    }
  }
}
'''

## 2. Configuración de Cline en el Codespace

Una vez abierto el Codespace con el `.devcontainer` configurado, la extensión **Cline** (`saoudrizwan.claude-dev`) estará instalada pero necesita ser configurada manualmente.

> ⚠️ **No uses "Login to Cline" en un Codespace**
>
> El botón **"Login to Cline"** usa autenticación OAuth con redirect a `vscode://`. El navegador del Codespace **no puede interceptar ese esquema** cuando VS Code corre de forma remota, por lo que la autenticación queda colgada indefinidamente aunque hayas autorizado en el portal. Usa siempre la opción **"Bring my own API key"**.

### 2.1 Abrir la extensión Cline

En la barra lateral de VS Code (dentro del Codespace), haz clic en el ícono de **Cline** (el logo de hormiga 🐜) o usa `Ctrl+Shift+P` → `Cline: Open in Editor`.

### 2.2 Seleccionar el modo de uso

Al abrirse verás la pantalla **"How will you use Cline?"** con tres opciones:

| Opción | Descripción |
|--------|-------------|
| Absolutely Free | Modelos gratuitos gestionados por Cline (requiere Login) |
| Frontier Model | Claude 4.5, GPT-5 Codex, etc. (requiere Login) |
| **Bring my own API key** ✅ | Usa tu propio proveedor y API Key |

1. Selecciona **"Bring my own API key"**.
2. Haz clic en **"Continue"**.

### 2.3 Configurar el proveedor (Configure your provider)

En la siguiente pantalla **"Configure your provider"**:

1. **API Provider** → selecciona `OpenRouter` (si no está ya seleccionado).
2. **OpenRouter API Key** → pega tu API Key de OpenRouter.
   - Si aún no tienes una: ve a [https://openrouter.ai](https://openrouter.ai) → **Keys** → genera una nueva clave gratuita.
3. **Model** → Los modelos gratuitos de OpenRouter tienen alta demanda y cuotas que varían. Usa la siguiente lista en orden de preferencia — si uno falla, prueba el siguiente:

   | Prioridad | Modelo | Notas |
   |-----------|--------|-------|
   | 1️⃣ | `z-ai/glm-4.5-air:free` | ✅ **Confirmado funcionando** — buena capacidad de código |
   | 2️⃣ | `google/gemini-2.0-flash-exp:free` | Cuota alta, rápido, excelente para código |
   | 3️⃣ | `meta-llama/llama-3.3-70b-instruct:free` | Bueno para código y razonamiento |
   | 4️⃣ | `deepseek/deepseek-r1:free` | Razonamiento profundo, ideal para arquitectura |
   | 5️⃣ | `mistralai/mistral-7b-instruct:free` | Más ligero, fallback de última opción |

   > ⚠️ **Errores comunes con modelos `:free`:**
   > - **Error 402** → Olvidaste el sufijo `:free`. El modelo sin ese sufijo es de pago.
   > - **Error 429** → Límite de velocidad superado. Espera 1 minuto y reintenta, o cambia a otro modelo de la lista.
   >
   > 💡 La disponibilidad de modelos gratuitos en OpenRouter cambia frecuentemente. Si ninguno de la lista funciona, busca en [https://openrouter.ai/models?q=free](https://openrouter.ai/models?q=free) cualquier modelo con el badge **Free** y pruébalo.

4. Haz clic en **"Continue"**.

> 🔒 **Seguridad:** La API Key se almacena localmente en la extensión y solo se usa para hacer peticiones a OpenRouter. No la incluyas en commits ni en archivos del repositorio. Usa los **Secrets de GitHub Codespaces** (`Settings → Codespaces → Secrets`) para persistirla entre sesiones de forma segura.

### 2.4 Verificar la configuración

Escribe un mensaje de prueba en el chat de Cline:

```
Hola, ¿estás listo para trabajar?
```

Si recibes una respuesta coherente, la configuración es correcta y puedes continuar con el laboratorio.

---



## 3. Estructura de Agentes (.cline/agents/)

Crear estos archivos en la raíz del proyecto. Estos definen el comportamiento de Cline en cada fase del AIDLC.

.cline/agents/architect.json: Enfocado en diseño de proyectos .csproj y reglas de dependencia.

.cline/agents/developer.json: Enfocado en C#, Minimal APIs y patrones de diseño.

.cline/agents/po.json: Enfocado en la validación semántica de las Historias de Usuario.


A. El Arquitecto (.clinerules/architect.json)

```markdown
# Role: Architect
Eres un Arquitecto de Software experto en sistemas distribuidos y mantenibilidad a largo plazo y experto en .NET.

## Principios Mandatorios:
1. **Separación de Preocupaciones:** Todo código debe seguir Arquitectura Hexagonal. 
   - Infraestructura (Adaptadores)
   - Aplicación (Casos de Uso y lógica de negocio)
   - Dominio (Entidades y Value Objects)
   - Presentación (API REST con Minimal APIs.)
2. **Domain-Driven Design (DDD):** Antes de codificar, define los Bounded Contexts y el Ubiquitous Language.
3. **Emergent Design:** No sobre-diseñes. Aplica patrones solo cuando la complejidad lo justifique, siguiendo principios de XP.
4. **Spec-Driven Development (SDD):** Las especificaciones mandan. Valida que cada componente cumpla con el contrato definido.

## Proceso de Trabajo:
- Paso 1: Analiza el requerimiento y genera un diagrama de componentes (Mermaid).
- Paso 2: Define los puertos (interfaces) antes que las implementaciones.
- Paso 3: Revisa que no haya fugas de lógica de negocio hacia la capa de infraestructura.
```

B. El Desarrollador (.cline/agents/developer.json)

```markdown
# Role: Developer
Eres un Desarrollador Senior de .NET experto en C#, Minimal APIs y Test-Driven Development.

## Principios Mandatorios:
1. **TDD Estricto (Red → Green → Refactor):** Nunca escribas código de producción sin tener primero una prueba fallida. Trabaja escenario a escenario.
2. **Mínimo código suficiente:** En la fase Green, escribe solo el código necesario para pasar el test en curso. No anticipes otros escenarios.
3. **Separación de capas:**
   - **Domain:** las entidades solo contienen validación de sus propios atributos (guardas en constructor/setters). Sin lógica de negocio.
   - **Application:** toda la lógica de negocio reside en servicios/casos de uso que dependen de interfaces de repositorio vía inyección de dependencias.
   - **Infrastructure:** repositorios implementados con colecciones en memoria (`List<T>` o `Dictionary<TKey, TValue>`). Sin ORM ni base de datos real.
   - **Presentation:** endpoints REST expuestos únicamente con Minimal APIs en `Program.cs`. Sin controladores MVC ni vistas.
4. **Convenciones .NET:** sigue las convenciones de nomenclatura estándar de .NET (PascalCase para clases/métodos, camelCase para parámetros).
5. **Pruebas Gherkin + AAA:** los tests deben nombrarse `Given_[contexto]_When_[acción]_Then_[resultado]` y estructurarse con Arrange / Act / Assert.

## Restricciones:
- No usar Entity Framework, Dapper ni ningún ORM o conector de base de datos.
- No generar frontend, Razor Pages, Blazor ni vistas MVC.
- No introducir dependencias no solicitadas.
- No modificar la arquitectura sin aprobación del Arquitecto.
```

C. El Product Owner (.cline/agents/po.json)

```markdown
# Role: Product Owner
Eres un Product Owner experto en metodologías ágiles y en transformar necesidades de negocio en especificaciones accionables.

## Principios Mandatorios:
1. **Dueño del QUÉ, no del CÓMO:** Defines qué debe hacer el sistema, no cómo implementarlo. Las decisiones técnicas son del Arquitecto y el Developer.
2. **Historias de Usuario estructuradas:** Cada HU debe seguir el formato:
   - **Título:** identificador + nombre corto (ej. `HU-1: Registrar libro`)
   - **Descripción:** `Como [rol] quiero [acción] para [beneficio]`
   - **Criterios de Aceptación (CA):** escritos en formato Gherkin (`Given / When / Then`), técnicos y verificables.
3. **Backlog priorizado:** Las HUs deben estar ordenadas por valor de negocio. Guarda el resultado en `backlog.md`.
4. **Validación continua:** Revisa que el código entregado cumple los criterios de aceptación definidos.

## Restricciones:
- No sugerir soluciones técnicas ni cambios de arquitectura.
- Enfocarse en el valor para el usuario final.
- Mantener la comunicación clara, concisa y libre de jerga técnica innecesaria.
```

## 4. Flujo de Trabajo (AIDLC)

Los estudiantes deben ejecutar los siguientes pasos para construir un **backend en .NET con persistencia en memoria**, sin frontend ni base de datos real.

> ### 🔀 Modos de Cline: PLAN vs ACT
>
> Cline opera en dos modos que se seleccionan en la barra inferior del chat:
>
> | Modo | Qué hace | Cuándo usarlo |
> |------|----------|---------------|
> | **PLAN** | Analiza y propone. **No escribe archivos ni ejecuta comandos.** | Para revisar y aprobar antes de que el agente actúe |
> | **ACT** | Ejecuta acciones: crea archivos, escribe código, corre comandos. | Cuando el plan ya fue revisado y aprobado |

---

### Paso 1: Definición del Producto — `@po` → `backlog.md`

> 🟢 **Modo: ACT** — El PO debe crear el archivo `backlog.md` en el repositorio.

```
/po, quiero crear un sistema de gestión de biblioteca. Solo necesito el backend: los bibliotecarios pueden registrar libros y los usuarios pueden prestarlos. Sin base de datos real, sin UI. Genera el archivo backlog.md con las Historias de Usuario y Criterios de Aceptación correspondientes.
```

---

### Paso 2: Refinamiento — `po` ajusta criterios

> 🟢 **Modo: ACT** — El PO modifica el `backlog.md` con los nuevos criterios.

El estudiante revisa el `backlog.md` generado. Si falta algo, interactúa:

```
/po, añade un Criterio de Aceptación a la US1 que obligue a validar que el formato del ISBN sea válido.
```

---

### Paso 3: Definición de la Arquitectura — `architect` → `architecture.md`

> 🟡 **Modo: PLAN primero** → revisa la propuesta → cambia a **ACT** para que escriba el archivo.
>
> Usar PLAN primero permite al estudiante validar la arquitectura propuesta antes de que el agente la materialice.

```
/architect, lee el backlog.md generado por el @po. Crea un architecture.md que defina exclusivamente la arquitectura del sistema. El documento debe incluir:
- Estilo arquitectónico: Arquitectura Hexagonal (Ports & Adapters) con .NET.
- Capas y proyectos: nombre de cada proyecto, tipo (classlib / webapi / xunit) y dependencias entre ellos.
- Convenciones de nomenclatura para namespaces, clases, interfaces y métodos.
- Patrones a aplicar: repositorio, inyección de dependencias, casos de uso como servicios.
- Restricciones técnicas: persistencia en memoria (List<T>), sin base de datos real, sin frontend, API REST con Minimal APIs, pruebas con xUnit, separar en distintos proyectos las pruebas unitarias y las pruebas de integración.
No incluyas tareas de implementación ni desglose por Historia de Usuario.
```

---

### Paso 4: Desglose de Tareas — `architect` → `tasks.md`

> 🟡 **Modo: PLAN primero** → revisa las tareas propuestas → cambia a **ACT** para que escriba el archivo.
>
> Antes de que el developer implemente, el arquitecto desglosa cada Historia de Usuario en tareas concretas de desarrollo. Esto da al developer instrucciones precisas por HU en lugar de leer todo el plan técnico de una vez.

```
/architect, lee el backlog.md y el architecture.md. Para cada Historia de Usuario del backlog genera una lista de tareas técnicas de desarrollo en un archivo tasks.md. Cada tarea debe indicar: la capa donde se implementa (Domain / Application / Infrastructure / Presentation), el artefacto a crear o modificar (clase, interfaz, endpoint) y los criterios de aceptación técnicos que debe cumplir. Usa el formato de lista de verificación Markdown (- [ ] tarea).
```

---

### Paso 5: Implementación — `developer` construye el código

> 🟢 **Modo: ACT** en todos los sub-pasos. Asegúrate de haber aprobado el `tasks.md` antes de comenzar.

#### 5.0 — Estructura de la solución

El primer prompt crea únicamente el esqueleto de proyectos, sin implementar ninguna clase todavía.

```
/developer, crea la estructura de la solución .NET de acuerdo al architecture.md. No escribas ninguna clase ni lógica todavía.
```

> ✅ Verifica que la solución compila correctamente (`dotnet build`) antes de continuar.

---

#### 5.N — Implementación TDD por Historia de Usuario

Por cada HU del `tasks.md` se sigue el ciclo **Red → Green → Refactor** escenario a escenario. Repite los siguientes **tres prompts** para cada escenario de cada HU.

> 🟥 **Ciclo TDD:**
> 1. **Red** — Escribe la prueba que falla (aún no existe implementación).
> 2. 🟢 **Green** — Escribe el mínimo código necesario para que la prueba pase.
> 3. 🔵 **Refactor** — Limpia el código sin romper las pruebas. Luego pasa al siguiente escenario.

---

**Prompt A — 🟥 Red: Generar el siguiente criterio de aceptación como prueba fallida**

```
/developer, analiza el tasks.md. Toma la próxima tarea pendiente (- [ ]) de la HU actual. Escribe únicamente la prueba unitaria en el proyecto de tests que representa ese criterio de aceptación. El nombre del test debe seguir el formato Gherkin: Given_[contexto]_When_[acción]_Then_[resultado]. Usa xUnit y patrón AAA. NO implementes ninguna clase de producción todavía. El test debe compilar pero fallar al ejecutarse.
```

> 🟥 Ejecuta `dotnet test` y confirma que el test **falla** antes de continuar.

---

**Prompt B — 🟢 Green: Implementar el mínimo código**

```
/developer, el test que acabas de escribir está fallando. Escribe el mínimo código de producción necesario para que únicamente ese test pase, respetando la capa que corresponde según el tasks.md. No agregues lógica extra ni anticipes otros criterios.
```

> 🟢 Ejecuta `dotnet test` y confirma que el test **pasa** antes de continuar.

---

**Prompt C — 🔵 Refactor: Limpiar sin romper**

```
/developer, revisa el código recién implementado. Refactoriza si hay duplicación, nombres poco claros o violaciones a SOLID. No agregues nueva funcionalidad. Marca la tarea como completada (`- [x]`) en el tasks.md.
```

> 🔵 Ejecuta `dotnet test` y confirma que todos los tests siguen **pasando**. Luego repite desde el Prompt A con el siguiente criterio de aceptación.

---

> ✅ Al completar todos los criterios de una HU, confirma que todas sus tareas están marcadas como `[x]` en el `tasks.md` y pasa a la siguiente HU.


---

### Paso 6: Validación — `/po` revisa el resultado

> 🔵 **Modo: PLAN** — El PO solo evalúa. No necesita escribir archivos, únicamente reportar.

```
/po, revisa el código generado por el /developer y valida que las Historias de Usuario del backlog.md estén implementadas correctamente. Indica cuáles criterios se cumplen y cuáles faltan.
```

