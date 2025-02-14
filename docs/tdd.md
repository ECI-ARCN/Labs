# Laboratorio: TDD con FizzBuzz en GitHub Codespaces

## Objetivo
Aplicar la metodología de Desarrollo Guiado por Pruebas (TDD) mediante la implementación del ejercicio clásico FizzBuzz en Java dentro de GitHub Codespaces.

## Requisitos Previos
- Cuenta en GitHub.
- Repositorio habilitado con GitHub Codespaces.
- Conocimientos básicos de Java.
- Familiaridad con TDD y JUnit.

## Configuración del Entorno
1. **Clonar el repositorio en GitHub Codespaces:**
   - Abre tu repositorio en GitHub.
   - Haz clic en el botón `Code` y selecciona `Codespaces`.
   - Inicia un nuevo Codespace o selecciona uno existente.

2. **Configurar el entorno de desarrollo:**
   - Abre la terminal en Codespaces y ejecuta:
     ```bash
     mkdir -p src/main/java src/test/java
     touch src/main/java/FizzBuzz.java src/test/java/FizzBuzzTest.java
     ```
   - Agrega un archivo `pom.xml` para configurar Maven y JUnit.

## Paso 1: Escribir la primera prueba
Crea el archivo `src/test/java/FizzBuzzTest.java` y agrega la siguiente prueba inicial:

```java
import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;

class FizzBuzzTest {
    @Test
    void testFizzBuzzReturnsNumber() {
        assertEquals("1", FizzBuzz.fizzbuzz(1));
    }
}
```

## Paso 2: Implementar la función mínima
Crea el archivo `src/main/java/FizzBuzz.java` y define la función:

```java
public class FizzBuzz {
    public static String fizzbuzz(int n) {
        return String.valueOf(n);
    }
}
```

Ejecuta las pruebas con:
```bash
mvn test
```

## Paso 3: Agregar más pruebas y lógica
Añadir pruebas para los casos de múltiplos de 3 y 5 en `FizzBuzzTest.java`:

```java
@Test
void testFizzBuzzMultiplo3() {
    assertEquals("Fizz", FizzBuzz.fizzbuzz(3));
}

@Test
void testFizzBuzzMultiplo5() {
    assertEquals("Buzz", FizzBuzz.fizzbuzz(5));
}

@Test
void testFizzBuzzMultiplo3y5() {
    assertEquals("FizzBuzz", FizzBuzz.fizzbuzz(15));
}
```

Modifica `FizzBuzz.java` para pasar estas pruebas:

## Paso 4: Ejecutar y validar las pruebas
Corre nuevamente `mvn test` y verifica que todas las pruebas pasen.

## Conclusión
Has aplicado TDD para desarrollar la función FizzBuzz. Sigue esta metodología para otros problemas y mejorarás la calidad de tu código.

## Siguientes Pasos
- Implementar pruebas adicionales.
- Refactorizar el código para mejorar su claridad.
- Integrar este flujo en un pipeline CI/CD en GitHub Actions.
