# DeliveryBot 🤖☕

Bot de Telegram para gestión de pedidos de una cafetería institucional, construido con **n8n**, un **AI Agent (Google Gemini)** y **Google Sheets** como base de datos.

**Autor:** Raymond Zabala
**Curso:** M4
**Fecha:** 01-09-2026

---

## 📋 Descripción general

DeliveryBot permite a los usuarios de una cafetería institucional pedir productos, consultar el menú, revisar y modificar su carrito, consultar y cancelar sus pedidos, todo mediante conversación natural y botones interactivos en Telegram. Un AI Agent interpreta la intención del usuario y decide qué herramienta (sub-workflow) ejecutar en cada paso, mientras Google Sheets funciona como base de datos persistente.

## 🏗️ Arquitectura

El sistema está compuesto por **5 workflows de n8n** independientes:

| Workflow | Trigger | Función |
|---|---|---|
| **Flujo principal (bot)** | Telegram Trigger | Conversación con el usuario: menú, carrito, pedidos, confirmación de compra |
| **Gestor de Estados** | Telegram Trigger (comandos del staff) | Actualiza el estado de un pedido y notifica al cliente |
| **Reportes Diarios** | Schedule Trigger | Genera y envía un resumen de ventas del día |
| **Limpiar Sesiones** | Schedule Trigger | Elimina carritos temporales abandonados/vencidos por inactividad |
| *(Puntos de lealtad)* | — | Integrado dentro del sub-workflow `agregar_pedido`, no es un workflow aparte |

### Flujo principal

```
Telegram Trigger → If (mensaje vs callback) → Answer Query a callback → AI Agent → Send a rich message
```

El **AI Agent** (modelo Google Gemini) tiene acceso a las siguientes herramientas, cada una implementada como un sub-workflow n8n:

| Tool | Sub-workflow | Qué hace |
|---|---|---|
| `verificar_o_crear_usuario` | Busca en `USUARIOS` por `telegram_id`; si no existe, lo crea | Se ejecuta siempre al inicio de la conversación |
| `ConsultarMenu` | Lee la hoja `MENU` | Devuelve productos, precios y stock actualizado |
| `agregar_pedido` | Valida el total, guarda el pedido en `PEDIDOS`, suma puntos de lealtad, notifica a cocina | Se ejecuta tras la confirmación explícita del usuario |
| `actualizar_stock` | Descuenta unidades vendidas en `MENU` | Se ejecuta inmediatamente después de `agregar_pedido` |
| `consultar_carrito` | Lee `SESSIONS` filtrando por `telegram_id` | Muestra el carrito temporal actual |
| `consultar_pedidos` | Lee `PEDIDOS`, ordena por fecha, toma los últimos 5 | Muestra historial de pedidos con estado |
| `vaciar_carrito` | Actualiza `SESSIONS`, limpiando `carrito_temporal` | Vacía el carrito a pedido explícito del usuario |
| `cancelar_pedido` | Verifica y actualiza el estado en `PEDIDOS` a "Cancelado" | Cancela un pedido, solo si aún no pasó a "Preparación" |

El bot usa **Simple Memory** (n8n) para mantener contexto de la conversación, y **Google Gemini** como modelo de lenguaje.

### Sub-workflow: `agregar_pedido` (incluye validación y puntos de lealtad)

```
When Executed by Another Workflow
  → Validar Total (Code: rechaza total <= 0)
    → ¿Total Válido? (IF)
        ├─ true (inválido)  → Responder Error (mensaje amigable, corta el flujo aquí)
        └─ false (válido)   → Google Sheets - Agregar Pedido (append a PEDIDOS)
                                → Buscar Usuario Puntos (Get Row en USUARIOS)
                                → Calcular Puntos (Code: 10 pts por cada $5 gastados)
                                → Update row in sheet (actualiza puntos_lealtad)
                                → Send a text message (notificación a grupo de cocina)
                                → Get row(s) in sheet (respuesta final para el AI Agent)
```

### Sub-workflow: `verificar_o_crear_usuario`

```
Execute Workflow Trigger → Buscar Usuario (Get Row en USUARIOS) → Existe Usuario (IF)
  ├─ true (no existe, 0 filas)  → Crear Usuario (append) ─┐
  └─ false (ya existe)          ───────────────────────────┴→ Merge → Formatear Respuesta
```

### Sub-workflow: `consultar_carrito`

```
Execute Workflow Trigger → Get Sesion (Get Row en SESSIONS, Always Output Data) → Formatear Respuesta
```
Devuelve `"carrito vacío"` si el usuario no tiene sesión activa.

### Sub-workflow: `vaciar_carrito`

```
When Executed by Another Workflow → Vaciar Carrito Manual (Update Row en SESSIONS) → Formatear Respuesta
```
Limpia `carrito_temporal` y actualiza `ultimo_cambio`. El AI Agent lo ofrece conversacionalmente después de mostrar el carrito ("¿deseas vaciarlo?"), sin necesidad de un botón dedicado.

### Sub-workflow: `consultar_pedidos`

```
Execute Workflow Trigger → Get Pedidos (read sheet, Always Output Data) → Ordenar por Fecha → Últimos 5 → Formatear Respuesta
```

### Sub-workflow: `cancelar_pedido`

```
When Executed by Another Workflow → Buscar Pedido (Get Row en PEDIDOS por id_pedido)
  → ¿Se Puede Cancelar? (IF: estado == "Recibido")
      ├─ true  → Actualizar Estado ("Cancelado") → Notificar Cocina → Formatear Respuesta (éxito)
      └─ false → Formatear Respuesta (ya no se puede cancelar)
```
Solo permite cancelar pedidos que aún no entraron en preparación; el AI Agent pide confirmación explícita antes de ejecutar la cancelación.

### Workflow: Gestor de Estados

```
Telegram Trigger (comando del staff) → Parsear Comando → Es Válido (IF)
  ├─ true  → Buscar Pedido → Actualizar Estado → Notificar Cliente → Confirmar Admin
  └─ false → Responder Error
```
Permite al staff de cocina actualizar el estado de un pedido (Recibido → Preparación → Entregado) desde un grupo de Telegram, notificando automáticamente al cliente.

### Workflow: Reportes Diarios

```
Schedule Trigger → Traer Pedidos → Calcular Metricas → Guardar Reporte → Enviar Reporte
```
Corre diariamente, calcula métricas de ventas del día y envía un resumen por Telegram.

### Workflow: Limpiar Sesiones

```
Schedule Trigger → Traer Sesiones → Filtrar Sesiones Viejas → Limpiar Carrito
```
Elimina periódicamente los carritos temporales que quedaron abandonados por inactividad (basado en `ultimo_cambio`) — mecanismo de mantenimiento automático, independiente de `vaciar_carrito` (que es a pedido explícito del usuario).

## 🗂️ Base de datos (Google Sheets)

| Hoja | Columnas principales | Uso |
|---|---|---|
| `MENU` | producto, precio, stock, categoría | Catálogo de productos |
| `USUARIOS` | telegram_id, nombre_completo, departamento, puntos_lealtad | Registro de usuarios y puntos acumulados |
| `PEDIDOS` | id_pedido, telegram_id, detalles_pedido, total_pago, estado, fecha | Historial de pedidos |
| `SESSIONS` | telegram_id, carrito_temporal, ultimo_cambio, pantalla_actual | Carrito de compras en curso |

## ⚙️ Configuración / despliegue

1. Importar los archivos `.json` de la carpeta [`/workflows`](./workflows) en tu instancia de n8n (uno por cada workflow listado arriba).
2. Crear las credenciales necesarias en n8n:
   - **Telegram account** (bot token)
   - **Google Sheets account** (OAuth o Service Account)
   - **Google Gemini (PaLM) API account**
3. Duplicar la plantilla de Google Sheets (`[link a la plantilla]`) y actualizar el `Document ID` en cada nodo de Google Sheets.
4. Configurar el Telegram Trigger del flujo principal con `Trigger On: Message, Callback Query`.
5. Activar (`Published`) cada uno de los 4 workflows.
6. Probar enviando `/start` (o cualquier mensaje) al bot.

## 🐛 Problemas conocidos resueltos durante el desarrollo

- Expresiones tipo `$('Telegram Trigger').item` fallaban dentro de las tool calls del AI Agent por pérdida del *pairing* de items; se resolvió usando `$('Telegram Trigger').first()`.
- Nodos Google Sheets con 0 filas detenían el sub-workflow por defecto; se activó **Always Output Data** en los nodos de lectura que pueden legítimamente no encontrar resultados (carrito vacío, sin pedidos).
- El AI Agent generaba las respuestas en Markdown (`**negrita**`) pero el nodo `Send a rich message` está configurado en formato `HTML`; se ajustó el System Prompt del agente para que use etiquetas HTML (`<b>`, `<i>`, `<code>`).
- `agregar_pedido` no validaba montos: se podían registrar pedidos con total 0 o negativo. Se agregó un nodo `Validar Total` con una rama de error dedicada, en vez de solo confiar en una instrucción del System Prompt.

## 📌 Pendientes / mejoras futuras

- Enriquecer la Notificación de Cocina con botones inline y un `id_pedido` más legible/reutilizable.
- Activar `On Error → Continue` en el nodo "Answer Query a callback" y `Enable Fallback Model` en el AI Agent, como manejo de errores adicional.
- Extender `cancelar_pedido` para permitir cancelación en estado "Preparación" con confirmación adicional del staff, si el negocio lo requiere.

## 📁 Estructura del repositorio

```
Proyecto_DeliveryBot_[ApellidoNombre]/
├── README.md
├── workflows/
│   ├── flujo_principal.json
│   ├── verificar_o_crear_usuario.json
│   ├── ConsultarMenu.json
│   ├── agregar_pedido.json
│   ├── actualizar_stock.json
│   ├── consultar_carrito.json
│   ├── consultar_pedidos.json
│   ├── vaciar_carrito.json
│   ├── cancelar_pedido.json
│   ├── gestor_de_estados.json
│   ├── reportes_diarios.json
│   └── limpiar_sesiones.json
└── docs/
    └── capturas/
```


Link Google Sheets: https://docs.google.com/spreadsheets/d/1H2vYttefS7U8Am4vfw8c792pfKAUT_6BACjc6ryBZ78/edit?usp=sharing
