# Automatización de inventario con n8n

Automatiza la detección de productos con stock bajo en un almacén de tecnología y crea automáticamente una tarjeta de reposición en Trello por cada alerta.

**Curso:** Automatización inteligente de procesos con n8n — EAM 2026-II

## Integrantes

- William Rodriguez — [COMPLETAR rol]
- Kevin Rengifo — [COMPLETAR rol]
- Bryan Benavides — [COMPLETAR rol]
- Michael Menotti — [COMPLETAR rol]

## Qué problema resuelve

El inventario se lleva en una hoja de cálculo que alguien debe revisar a mano, comparando producto por producto la cantidad disponible contra el stock mínimo. Cuando un producto llega al mínimo, la alerta depende de que una persona se acuerde de revisar y escriba el aviso. El resultado es que las reposiciones llegan tarde, se saltan productos y no queda registro de qué ya se reportó.

La solución revisa el inventario completo, clasifica cada alerta según la categoría del producto y deja una tarjeta en el tablero del equipo, sin intervención manual.

---

## Alcance actual

**Funciona y está comprobado**

- Disparo por webhook autenticado, por programación diaria a las 8:00 AM y manual para pruebas
- Validación del evento de entrada con rechazo controlado antes de escribir en cualquier sistema
- Lectura del inventario desde Google Sheets
- Filtrado por stock mínimo y clasificación en tres rutas según categoría
- Creación de una tarjeta en Trello por cada alerta
- Registro de fallos en una hoja de Google Sheets

**Limitaciones principales**

- No hay control de duplicados: cada ejecución vuelve a crear tarjetas de los productos que siguen bajo mínimo
- El webhook solo responde en `localhost`; no se montó túnel para recibir eventos desde Internet
- La comparación de categorías distingue mayúsculas y tildes; un valor mal escrito cae en la ruta por defecto
- La resolución DNS del equipo local es intermitente; conviene ejecutar `ipconfig /flushdns` antes de correr

---

## Arquitectura

![Proceso automatizado](documentacion/diagramas/Proceso_Automatizado.png)

| Workflow | Responsabilidad |
|---|---|
| `workflows/Automatizacion-Inventario.json` | **Principal.** Recibe el evento, lo valida, lee el inventario, decide qué productos necesitan reposición y coordina el aviso |
| `workflows/Alerta-Inventario-Trello.json` | **Subworkflow.** Recibe una alerta ya armada y crea la tarjeta en Trello |
| `workflows/Manejador_Errores.json` | **Manejador de errores.** Se dispara cuando cualquiera de los dos falla y registra el fallo en Google Sheets |

**Términos usados aquí**

- **Webhook:** la dirección a la que otro sistema envía una petición para iniciar el flujo. En este proyecto es la entrada por la que se pide una revisión de inventario.
- **Subworkflow:** un flujo aparte que el principal llama como si fuera una función. Recibe datos, hace su trabajo y devuelve un resultado.
- **API:** la interfaz por la que n8n le pide a Trello que cree una tarjeta.

---

## Requisitos

- **n8n 2.36.9** o superior, autohospedado
- Cuenta de **Google** con acceso a Google Sheets
- Proyecto propio en **Google Cloud** con las APIs de Google Sheets y Google Drive habilitadas
- Cuenta de **Trello** con un tablero y una lista para las alertas
- **Node.js** instalado, si se ejecuta n8n por npm

---

## Configuración

### 1. Google Cloud

1. Crear un proyecto nuevo en [console.cloud.google.com](https://console.cloud.google.com)
2. En **APIs y servicios → Biblioteca**, habilitar **Google Sheets API** y **Google Drive API**
3. Configurar la pantalla de consentimiento de OAuth
4. En **Credenciales**, crear un **ID de cliente de OAuth** de tipo *Aplicación web* con esta URI de redirección:
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
5. Guardar el Client ID y el Client Secret

### 2. Google Sheets

Crear un documento con dos hojas:

**`Hoja 1`** — el inventario:

| id | producto | categoria | cantidad | stock_minimo |
|---|---|---|---|---|
| 1 | Teclado mecanico | Perifericos | 4 | 5 |
| 2 | Mouse inalambrico | Perifericos | 3 | 5 |

**`Errores`** — el registro de fallos:

| fecha_hora | workflow | nodo | mensaje | ejecucion |
|---|---|---|---|---|

### 3. Trello

1. Crear un tablero con una lista para las alertas de reposición
2. Obtener la **API Key** y el **Token** en [trello.com/power-ups/admin](https://trello.com/power-ups/admin)
3. Copiar el **ID de la lista** donde se crearán las tarjetas

### 4. Credenciales en n8n

Crear estas tres. **Ninguna viene en los archivos de este repositorio.**

| Nombre | Tipo | Contenido |
|---|---|---|
| `Google Sheets account` | Google Sheets OAuth2 API | Client ID y Client Secret del paso 1 |
| `Header Auth Inventario` | Header Auth | Un nombre de encabezado y un valor definidos por ti |
| `Trello Header` | Header Auth | Name: `Authorization` · Value: `OAuth oauth_consumer_key="TU_KEY", oauth_token="TU_TOKEN"` |

---

## Importación y conexiones

Importar en **este orden**, porque el principal depende de los otros dos:

1. `Manejador_Errores.json`
2. `Alerta-Inventario-Trello.json`
3. `Automatizacion-Inventario.json`

**Después de importar hay que reconectar a mano:**

| Dónde | Qué reconectar |
|---|---|
| `Get row(s) in sheet` | Credencial, documento y hoja |
| `Webhook` | Credencial de Header Auth |
| `Crear tarjeta Trello` (subworkflow) | Credencial de Header Auth y el `idList` de tu lista |
| `Append row in sheet` (manejador) | Credencial, documento y hoja `Errores` |
| `Call 'Alerta-Inventario-Trello'` | **Volver a seleccionar el subworkflow del desplegable.** El JSON trae el identificador de la instalación original y no coincide con el de la tuya |
| Settings de los dos workflows | **Error Workflow** → `Manejador_Errores` |

Publicar siempre **primero el subworkflow y después el principal**. Si se edita el subworkflow, vuelve a borrador y hay que publicarlo de nuevo.

---

## Ejecución

### Uso habitual

Publicar `Automatizacion-Inventario`. El Schedule Trigger revisa el inventario todos los días a las 8:00 AM y crea las tarjetas que correspondan.

**Para detener la ejecución programada:** despublicar el workflow desde el menú junto a su nombre.

### Prueba manual

Usar la URL de prueba, que exige pulsar **Listen for test event** en el nodo Webhook antes de cada envío:

```
http://localhost:5678/webhook-test/revisar-inventario
```

La URL de producción no necesita escucha manual, pero requiere el workflow publicado:

```
http://localhost:5678/webhook/revisar-inventario
```

### Dónde ver el resultado

- **Tarjetas creadas:** en la lista del tablero de Trello
- **Fallos:** en la hoja `Errores` del documento de Google Sheets
- **Detalle de cada corrida:** en la pestaña **Executions** de n8n

---

## Prueba de ejemplo

Con los datos de ejemplo de la hoja, seis productos quedan por debajo de su stock mínimo.

**Entrada**

```powershell
Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/revisar-inventario" `
  -Method Post `
  -Headers @{"X-API-KEY"="<el valor de tu credencial>"} `
  -ContentType "application/json" `
  -Body '{"accion":"revisar_stock","solicitud_id":"PRUEBA-001"}'
```

**Salida esperada**

```json
{
  "ok": true,
  "mensaje": "Revisión de inventario completada"
}
```

Y seis tarjetas nuevas en el tablero de Trello, una por producto.

**Si el cuerpo viene incompleto**, por ejemplo `{"accion":"otra_cosa"}`, la respuesta es HTTP 400 y no se escribe nada en ningún sistema:

```json
{
  "ok": false,
  "errores": [
    "El campo accion debe ser revisar_stock",
    "Falta el campo obligatorio solicitud_id"
  ]
}
```

---

## Documentación y evidencias

- [Bitácora unificada del proyecto](documentacion/Bitacora_Unificada_Proyecto.md)
- [Presentación en PDF](documentacion/Presentacion_Corte_2.pdf)
- [Diagrama del proceso manual](documentacion/diagramas/Proceso_Manual.png)
- [Diagrama del proceso automatizado](documentacion/diagramas/Proceso_Automatizado.png)
- [Evidencias de las pruebas](documentacion/evidencias/)

---

## Seguridad

Este repositorio no contiene tokens, claves ni credenciales. Cada persona debe crear las suyas siguiendo la sección de configuración. Los datos de inventario son ficticios.
