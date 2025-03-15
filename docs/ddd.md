
# **Taller de Event Storming: Ejemplo - Tienda en Línea**

## **1. Introducción**
Event Storming es una técnica de modelado colaborativo para entender y diseñar sistemas a partir de eventos del dominio. En este ejemplo, modelaremos una **tienda en línea** desde la selección de productos hasta la entrega del pedido.

---

## **2. Eventos del Dominio**
Los eventos representan cosas que suceden en el sistema y se escriben en **pasado**.

<style>
    .postit {
        display: inline-block;
        padding: 10px;
        margin: 5px;
        border-radius: 5px;
        box-shadow: 2px 2px 5px gray;
    }
    .evento { background-color: #ffa500; color: black; }
    .agregado { background-color: #fdfd96; color: black; }
    .comando { background-color: #87ceeb; color: black; }
    .actor { background-color: #ffeb3b; color: black; }
    .politica { background-color: #9b59b6; color: white; }
    .problema { background-color: #ffcccb; color: black; }
    .bounded-context { background-color:rgb(8, 214, 245); color: black; }
    table { border-collapse: collapse; }
    td { padding: 10px; border: none; }
</style>

<div class="postit evento">Producto Agregado al Carrito</div>
<div class="postit evento">Pedido Creado</div>
<div class="postit evento">Pago Procesado</div>
<div class="postit evento">Pedido Enviado</div>
<div class="postit evento">Pedido Entregado</div>

---

## **3. Actores y Comandos**
Los actores son quienes interactúan con el sistema y los comandos representan acciones que desencadenan eventos.

<table>
    <tr>
        <td class="postit actor">Cliente</td>
        <td class="postit comando">Agregar Producto al Carrito</td>
    </tr>
    <tr>
        <td class="postit actor">Cliente</td>
        <td class="postit comando">Crear Pedido</td>
    </tr>
    <tr>
        <td class="postit actor">Sistema de Pagos</td>
        <td class="postit comando">Procesar Pago</td>
    </tr>
    <tr>
        <td class="postit actor">Sistema de Envíos</td>
        <td class="postit comando">Enviar Pedido</td>
    </tr>
    <tr>
        <td class="postit actor">Cliente</td>
        <td class="postit comando">Confirmar Entrega</td>
    </tr>
</table>

---

## **4. Agregados y Políticas**
Los agregados representan unidades lógicas del sistema, mientras que las políticas definen reglas de negocio.

<table>
    <tr>
        <td class="postit agregado">Carrito de Compras</td>
        <td class="postit politica">Verificar Stock Antes de Confirmar Pedido</td>
    </tr>
    <tr>
        <td class="postit agregado">Pedido</td>
        <td class="postit politica">Validar Método de Pago Antes de Procesarlo</td>
    </tr>
    <tr>
        <td class="postit agregado">Pedido</td>
        <td class="postit politica">Notificar al Cliente sobre el Estado del Pedido</td>
    </tr>
</table>

---

## **5. Problemas Identificados**
Durante el taller, pueden surgir dudas o problemas que necesitan ser resueltos.

<div class="postit problema">¿Qué sucede si el pago es rechazado?</div>
<div class="postit problema">¿Cómo manejamos productos sin stock?</div>
<div class="postit problema">¿Cómo gestionamos devoluciones?</div>

---

## **6. Refinamiento y Modelado del Dominio**
Basándonos en los eventos, podemos identificar **bounded contexts** dentro de la arquitectura del sistema.

<div class="postit bounded-context">Carrito de Compras</div>
<div class="postit bounded-context">Procesamiento de Pedidos</div>
<div class="postit bounded-context">Envíos</div>

---

## **7. Conclusión**
Este Event Storming nos permite visualizar el flujo de eventos y mejorar la colaboración entre equipos técnicos y de negocio, asegurando una mejor comprensión del dominio.


🚀 **¡Ahora estás listo para aplicar Event Storming en tus proyectos!**


## **Taller caso de negocio**
Reunete con tu grupo y plantea el event storming en el tablero de la clase [ARCN - 2025-1](https://miro.com/welcomeonboard/NkhoVStiYUxGeEwxcU5Qd09uRnBjVEF1SlJ1MnU3VnYraVVOanRLSXVzQnFoYjNyQnF2SFFZK1hYT0pJeXdjZXpJSGMvcjJEVG92dEpFNGF5Wko5SWgyTWliTGVRUmtURmh4ekc1R0RJNjhab0p0cktEQ2kwTHlhYzJFeUN2OGV0R2lncW1vRmFBVnlLcVJzTmdFdlNRPT0hdjE=?share_link_id=967622799927), con el caso de negocio (Dominio) seleccionado y realiza las siguientes actividades:

1. Ejecuta el ciclo propuesto en [ddd-practitioners](https://ddd-practitioners.com/2023/03/20/remote-eventstorming-workshop/)
2. Importante manejar colores en lo posible estándares del event storming.
3. Agregar lenguaje ubicuo y modelo de datos.
4. Mostrar cómo se hace el ejercicio fase a fase y al final entregar un blue print del modelo de dominio completo.
5. En qué fase del proyecto se debería ejecutar este workshop y quienes consideran que deberían participar?
6. Proponer como realizarían este evento si su rol es el de coach facilitador.