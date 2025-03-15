
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
    .actor { background-color: #fdfd96; color: black; }
    .comando { background-color: #87ceeb; color: black; }
    .agregado { background-color: #ffeb3b; color: black; }
    .politica { background-color: #9b59b6; color: black; }
    .problema { background-color: #ffcccb; color: black; }
    .bounded-context { background-color: #8e44ad; color: white; }
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
Este Event Storming nos permite visualizar el flujo de eventos y mejorar la colaboración entre equipos técnicos y de negocio, asegurando una mejor comprensión del dominio de la tienda en línea.

---

## **8. Blueprint General del Ejercicio Completo**

### **Flujo de Eventos y Componentes**

1. **Cliente**: Agrega productos al carrito, crea el pedido, y confirma la entrega.
2. **Sistema de Pagos**: Procesa el pago.
3. **Sistema de Envíos**: Envía el pedido.

### **Eventos**
- Producto Agregado al Carrito
- Pedido Creado
- Pago Procesado
- Pedido Enviado
- Pedido Entregado

### **Actores y Comandos**
- **Cliente**: Agregar Producto al Carrito, Crear Pedido, Confirmar Entrega
- **Sistema de Pagos**: Procesar Pago
- **Sistema de Envíos**: Enviar Pedido

### **Agregados y Políticas**
- **Carrito de Compras**: Verificar Stock Antes de Confirmar Pedido
- **Pedido**: Validar Método de Pago Antes de Procesarlo, Notificar al Cliente sobre el Estado del Pedido

### **Problemas Identificados**
- ¿Qué sucede si el pago es rechazado?
- ¿Cómo manejamos productos sin stock?
- ¿Cómo gestionamos devoluciones?

### **Bounded Contexts**
- Carrito de Compras
- Procesamiento de Pedidos
- Envíos

---

🚀 **¡Ahora estás listo para aplicar Event Storming en tus proyectos!**
