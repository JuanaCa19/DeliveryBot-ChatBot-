# 🤖☕ DeliveryBot

**DeliveryBot** es un chatbot de pedidos para cafetería, construido sobre **n8n** e integrado con **Telegram** y **Google Sheets**. Permite a los clientes registrarse, consultar el menú, armar un carrito, confirmar pedidos y hacer seguimiento de su estado, mientras acumulan puntos de un programa de lealtad. También incluye herramientas para el personal de cocina (gestión de estados de pedido) y reportes automáticos de ventas.

No es una aplicación tradicional con backend propio: toda la lógica vive en **workflows de n8n** (archivos `.json` exportados), usando Google Sheets como base de datos y la API de Telegram como interfaz de usuario.

---

## ✨ Funcionalidades

- **Registro de usuarios** mediante un formulario web enlazado desde Telegram (nombre, dirección, puntos de lealtad iniciales).
- **Menú interactivo** por categorías (Bebidas, Comidas, Snacks) con botones inline de Telegram.
- **Carrito de compras**: agregar productos, ajustar cantidades (+/-), ver resumen, **eliminar el carrito completo** o confirmar el pedido.
- **Validación de stock** antes de confirmar un pedido, con aviso al usuario si algo no está disponible.
- **Generación de pedidos** con id único, cálculo de total y descuento automático de stock.
- **Programa de puntos de lealtad**: por cada $10.000 gastados en un pedido confirmado, el usuario suma 1 punto, con notificación automática del saldo acumulado.
- **Seguimiento de pedidos**: consulta del pedido activo y de historial de pedidos anteriores.
- **Panel para cocina/administración**: cambio de estado del pedido (Preparación → En camino → Entregado) mediante botones, restringido solo a personal autorizado (por `chat_id`).
- **Reporte diario automático** (20:00): total vendido, cantidad de pedidos, producto estrella y hora pico, enviado al chat de cocina.
- **Detección de intención con IA**: un agente conversacional (vía Ollama) distingue entre saludo/inicio, consulta general y una acción que debe delegarse a otro flujo (silencio).

---

## 🏗️ Arquitectura

El proyecto está compuesto por tres workflows de n8n:

| Workflow | Disparador | Propósito |
|---|---|---|
| `FlujoPrincipal.json` | Telegram Trigger (mensajes y callback_query) | Flujo principal del bot: menú, carrito, confirmación de pedido, puntos, estado de pedidos, historial y panel de cocina. |
| `RegistrarUsuario.json` | Form Trigger | Formulario de registro de nuevos usuarios, que guarda sus datos en Google Sheets y redirige de vuelta al bot de Telegram. |
| `ReporteDiario.json` | Schedule Trigger (cron `0 20 * * *`) | Genera y envía el reporte de ventas del día al chat de cocina. |

### Flujo general de `FlujoPrincipal`

```
Telegram Trigger
 ├─ ¿Es callback_query? ─ No ─▶ Verifica registro ─▶ Agente IA (INICIO / CONSULTA / SILENCIO)
 │                                                     ├─ INICIO ─▶ Menú principal (Menu / Pedidos / Historial / Confirmar)
 │                                                     └─ SILENCIO/CONSULTA ─▶ (delegado a otro flujo / mensaje genérico)
 └─ Sí ─▶ Switch (mostrarMenu / mostrarPedido / mostrarHistorial / confirmarPedido / categorías)
           └─ Router de acciones (prod: / qty: / cart: / estado:)
              ├─ prod:   ─▶ Mostrar producto y selector de cantidad
              ├─ qty:    ─▶ Ajustar cantidad ─▶ Agregar al carrito
              ├─ cart:   ─▶ ¿confirm o delete?
              │             ├─ cart:confirm ─▶ Validar stock ─▶ Generar pedido ─▶ Descontar stock
              │             │                    ├─▶ Notificar cocina y usuario
              │             │                    └─▶ Sumar puntos de lealtad ─▶ Notificar puntos
              │             └─ cart:delete  ─▶ Vaciar carrito ─▶ Confirmar al usuario
              └─ estado: ─▶ (solo admin/cocina) Actualizar estado del pedido y notificar al cliente
```

---

## 🗄️ Estructura de datos (Google Sheets)

Todo el estado del bot vive en una sola hoja de cálculo (`DeliveryBot_DB`) con estas pestañas:

- **Usuarios**: `telegram_id`, `nombre_completo`, `departamento`, `puntos_lealtad`
- **Menu**: `id_producto`, `nombre`, `categoria`, `precio`, `stock`
- **Sesiones**: `telegram_id`, `pantalla_actual`, `carrito_temporal` (carrito en curso, en JSON), `ultimo_cambio`
- **Pedidos**: `id_pedido`, `id_usuario`, `detalles_pedido`, `total_pago`, `estado`, `fecha`, `hora`

> ⚠️ **Nota:** la columna de puntos en la hoja `Usuarios` se llama `puntos_lealtad`. Si vas a activar el flujo de puntos, asegúrate de que los nodos de Google Sheets lean y escriban ese nombre de columna exacto (y no `puntos`).

---

## ⚙️ Requisitos

- Una instancia de **n8n** (self-hosted o cloud) con los siguientes nodos community/base: `telegramTrigger`, `telegram`, `googleSheets`, `httpRequest`, `code`, `if`, `switch`, `formTrigger`, `scheduleTrigger`, `splitOut`, y el paquete `@n8n/n8n-nodes-langchain` (agente + modelo de chat Ollama).
- Un **bot de Telegram** creado con [@BotFather](https://t.me/BotFather).
- Una hoja de **Google Sheets** con la estructura descrita arriba.
- Un servidor **Ollama** accesible (o reemplazar el nodo de modelo por el proveedor de IA que prefieras).
- Un dominio/túnel (ej. ngrok) para exponer el formulario de registro.

### Variables de entorno

| Variable | Uso |
|---|---|
| `TELEGRAM_BOT_TOKEN` | Token del bot, usado en las llamadas HTTP directas a la API de Telegram. |
| `KITCHEN_CHAT_ID` | Chat ID del personal de cocina/administración: recibe notificaciones de nuevos pedidos, reportes diarios y valida permisos para cambiar el estado de un pedido. |

### Credenciales en n8n

- **Telegram API** (bot `DeliveryBot`), usada en el Trigger y en los nodos `telegram`.
- **Google Sheets OAuth2**, usada en todos los nodos de lectura/escritura sobre `DeliveryBot_DB`.
- **Ollama API**, usada por el modelo de lenguaje del agente de intención.

---

## 🚀 Instalación

1. Clona este repositorio.
2. En n8n, importa los tres archivos JSON (`FlujoPrincipal.json`, `RegistrarUsuario.json`, `ReporteDiario.json`) como workflows independientes.
3. Crea/duplica la hoja de Google Sheets con las pestañas y columnas indicadas en [Estructura de datos](#️-estructura-de-datos-google-sheets).
4. Configura las credenciales de Telegram, Google Sheets y Ollama en cada nodo que las requiera.
5. Define las variables de entorno `TELEGRAM_BOT_TOKEN` y `KITCHEN_CHAT_ID` en tu instancia de n8n.
6. Configura el webhook del formulario de registro (`RegistrarUsuario`) y actualiza la URL en el botón "Registrarme" del flujo principal si cambia.
7. Activa los tres workflows.

---

## 🗺️ Ideas para seguir mejorando

- Canje de puntos de lealtad por productos o descuentos.
- Panel de administración fuera de Telegram para gestionar el menú y el stock.
- Soporte multi-sucursal.
- Migrar el estado de sesión/carrito a una base de datos relacional para mayor escalabilidad.

---

## 📄 Licencia

Sin licencia definida todavía. Agrega un archivo `LICENSE` si vas a publicar el proyecto como open source.
