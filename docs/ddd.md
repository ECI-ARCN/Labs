# **Taller de Event Storming: Ejemplo - Tienda en Línea**

## **1. Introducción**
Event Storming es una técnica de modelado colaborativo para entender y diseñar sistemas a partir de eventos del dominio. En este ejemplo, modelaremos una **tienda en línea** desde la selección de productos hasta la entrega del pedido.

---
## **2. Eventos del Dominio**
Los eventos representan cosas que suceden en el sistema y se escriben en **pasado**.


<style>
    .postit {
        display: inline-block;
        background-color: #fdfd96;
        padding: 10px;
        margin: 5px;
        border-radius: 5px;
        box-shadow: 2px 2px 5px gray;
    }
</style>

<div class="postit">Producto Agregado al Carrito</div>
<div class="postit">Pedido Creado</div>
<div class="postit">Pago Procesado</div>
<div class="postit">Pedido Enviado</div>
<div class="postit">Pedido Entregado</div>

---
## **3. Actores y Comandos**
Los actores son quienes interactúan con el sistema y los comandos representan acciones que desencadenan eventos.

```html
<div class="postit">Cliente</div>
<div class="postit">Sistema de Pagos</div>
<div class="postit">Sistema de Envíos</div>
<br>
<div class="postit">Agregar Producto al Carrito</div>
<div class="postit">Crear Pedido</div>
<div class="postit">Procesar Pago</div>
<div class="postit">Enviar Pedido</div>
<div class="postit">Confirmar Entrega</div>
```

---
## **4. Agregados y Políticas**
Los agregados representan unidades lógicas del sistema y las políticas definen reglas de negocio.

```html
<div class="postit">Carrito de Compras</div>
<div class="postit">Pedido</div>
<br>
<div class="postit">Verificar Stock Antes de Confirmar Pedido</div>
<div class="postit">Validar Método de Pago Antes de Procesarlo</div>
<div class="postit">Notificar al Cliente sobre el Estado del Pedido</div>
```

---
## **5. Problemas Identificados**
Durante el taller, pueden surgir dudas o problemas que necesitan ser resueltos.

```html
<div class="postit">¿Qué sucede si el pago es rechazado?</div>
<div class="postit">¿Cómo manejamos productos sin stock?</div>
<div class="postit">¿Cómo gestionamos devoluciones?</div>
```

---
## **6. Refinamiento y Modelado del Dominio**
Basándonos en los eventos, podemos identificar **bounded contexts** dentro de la arquitectura del sistema.

```html
<div class="postit">Carrito de Compras</div>
<div class="postit">Procesamiento de Pedidos</div>
<div class="postit">Envíos</div>
```

---
## **7. Conclusión**
Este Event Storming nos permite visualizar el flujo de eventos y mejorar la colaboración entre equipos técnicos y de negocio, asegurando una mejor comprensión del dominio de la tienda en línea.

🚀 **¡Ahora estás listo para aplicar Event Storming en tus proyectos!**
