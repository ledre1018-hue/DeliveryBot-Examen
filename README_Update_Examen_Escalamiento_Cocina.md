# DeliveryBot — Update: Examen — Escalamiento por Retraso en Cocina

## 1. Objetivo de la actualización

Se agregó un flujo independiente para detectar pedidos que permanecen demasiado tiempo en estado **En preparación** y generar una alerta automática en Telegram.

La finalidad es evitar que un pedido quede olvidado en cocina porque el personal no cambió su estado a **En camino**.

### Regla principal

Un pedido se considera atrasado cuando:

- Su estado es `En preparación`.
- Han transcurrido **más de 45 minutos** desde la combinación de sus campos `fecha` y `hora`.

## 2. Flujo implementado

La nueva rama del workflow queda organizada así:

`Schedule Trigger → Leer Pedidos Preparacion → Filtrar Pedidos Atascados → Hay Pedidos Atascados → Armar Mensaje Alerta → Notificar Supervisor → Registrar En Logs`

Esta rama funciona de manera independiente del `Telegram Trigger` que atiende las interacciones normales del bot.

## 3. Schedule Trigger

El nodo **Schedule Trigger** inicia la revisión automáticamente cada **30 minutos**.

Configuración utilizada:

- Tipo: Schedule Trigger
- Intervalo: Minutes
- Cada: 30 minutos

Esto permite revisar periódicamente la hoja `PEDIDOS` sin depender de que un usuario escriba al bot.

## 4. Lectura de Google Sheets

El nodo **Leer Pedidos Preparacion** consulta la pestaña `PEDIDOS` del mismo documento de Google Sheets utilizado por DeliveryBot.

El documento configurado corresponde al proyecto DeliveryBot y contiene las hojas que utiliza el sistema, entre ellas `SESSIONS`, `MENU`, `PEDIDOS` y `LOGS`.

## 5. Lógica de antigüedad

El nodo **Filtrar Pedidos Atascados** utiliza JavaScript para:

1. Obtener la fecha y hora actual.
2. Recorrer los pedidos recibidos desde Google Sheets.
3. Conservar únicamente los pedidos cuyo estado sea `En preparación`.
4. Construir una marca de tiempo usando `fecha` + `hora`.
5. Calcular los minutos transcurridos.
6. Seleccionar los pedidos cuyo tiempo sea superior a 45 minutos.
7. Crear la lista de IDs atrasados.

La idea central de la lógica es:

```text
minutos transcurridos = hora actual - hora de creación
```

y se valida:

```text
minutos transcurridos > 45
```

## 6. Validación de pedidos encontrados

El nodo **Hay Pedidos Atascados** comprueba si la cantidad de pedidos detectados es mayor que cero.

- Si `cantidad > 0`, continúa hacia la alerta.
- Si no hay pedidos atrasados, no se envía una alerta al supervisor.

Esto evita enviar mensajes innecesarios cuando todo está funcionando normalmente.

## 7. Construcción de la alerta

El nodo **Armar Mensaje Alerta** toma los IDs detectados y genera un mensaje de Telegram con el siguiente formato:

> 🚨 ATENCIÓN: Los siguientes pedidos llevan más de 45 min en preparación: [ID1, ID2...]. Favor verificar en cocina.

De esta manera, el supervisor recibe directamente la información necesaria para localizar los pedidos problemáticos.

## 8. Notificación por Telegram

El nodo **Notificar Supervisor** utiliza la credencial de Telegram del proyecto y envía el mensaje al chat configurado para la operación.

Esto conecta la detección automática con el canal operativo de DeliveryBot, permitiendo que el supervisor actúe sin tener que revisar manualmente la hoja.

## 9. Registro en LOGS

Después de la notificación se ejecuta **Registrar En Logs**, que agrega información de la ejecución en la pestaña `LOGS`.

Los campos registrados son:

- `fecha_ejecucion`: momento en que se realizó la revisión.
- `cantidad`: número de pedidos atrasados detectados.
- `pedidos_detectados`: IDs encontrados.
- `mensaje_enviado`: texto enviado al supervisor.

Este registro sirve como evidencia de ejecución y permite consultar posteriormente qué alertas fueron generadas.

## 10. Relación con el workflow original

La actualización no reemplaza el flujo principal de DeliveryBot.

El workflow original continúa utilizando:

- **Telegram** como interfaz de interacción con clientes y cocina.
- **Google Sheets** como almacenamiento de sesiones, menú, pedidos y registros.
- **Code / JavaScript** para transformación de datos y reglas de negocio.
- **Switch / IF** para enrutar y tomar decisiones.

La nueva rama añade una automatización temporal para controlar pedidos olvidados.

## 11. Diferencia entre Telegram Trigger y Schedule Trigger

### Telegram Trigger

Se activa cuando llega una interacción desde Telegram, por ejemplo un mensaje o un botón.

### Schedule Trigger

Se activa por tiempo, independientemente de que alguien escriba al bot.

Por eso, para probar esta actualización desde n8n se debe ejecutar la rama desde **Schedule Trigger**.

## 12. ¿Es necesario cambiar el botón naranja de ejecución?

**No para el funcionamiento automático del workflow.**

Si el workflow está publicado/activo, el `Telegram Trigger` permanece escuchando eventos reales de Telegram. Por eso n8n puede impedir ejecutar manualmente desde ese trigger mientras está ocupado.

Para probar la nueva lógica manualmente:

1. Abrir el menú de ejecución del workflow.
2. Seleccionar **Schedule Trigger**.
3. Ejecutar esa rama.

También se puede ejecutar directamente desde el nodo `Schedule Trigger` si n8n muestra el botón de reproducción sobre el nodo.

Esto **no significa que haya que despublicar el bot**. De hecho, despublicarlo apagaría la recepción normal de Telegram.

## 13. Prueba esperada

Para comprobar la actualización se puede colocar o conservar en `PEDIDOS` un pedido con:

- Estado: `En preparación`
- Fecha/hora suficientemente antigua para superar 45 minutos.

Al ejecutar el `Schedule Trigger`, el flujo debe:

1. Leer `PEDIDOS`.
2. Detectar el pedido atrasado.
3. Generar el ID en `idsAtascados`.
4. Confirmar que `cantidad > 0`.
5. Crear el mensaje.
6. Enviarlo por Telegram.
7. Registrar la ejecución en `LOGS`.

## 14. Evidencias para la evaluación

Se recomienda presentar:

1. Captura del canvas mostrando los nuevos nodos.
2. Captura de la configuración del `Schedule Trigger` en 30 minutos.
3. Captura del resultado de ejecución sin errores.
4. Captura del mensaje recibido en Telegram.
5. Captura de la fila creada en `LOGS`.

## 15. Relación con la rúbrica

### Configuración del Cron — 5 puntos
Se implementó `Schedule Trigger` con ejecución cada 30 minutos.

### Lógica de Antigüedad — 9 puntos
Se filtra por estado `En preparación` y se calcula el tiempo transcurrido para detectar pedidos con más de 45 minutos.

### Calidad de la Alerta — 7 puntos
La alerta identifica los IDs de los pedidos retrasados y solicita verificar la situación en cocina.

### Registro en LOGS — 3 puntos
La ejecución se registra en la hoja `LOGS` con fecha, cantidad, pedidos detectados y mensaje.

### Dominio y conocimiento del código — 16 puntos
La solución combina automatización temporal, lectura de Google Sheets, JavaScript, decisiones condicionales, Telegram y registro de auditoría.

## 16. Resumen para sustentación

> “Agregué una rama independiente al DeliveryBot para controlar pedidos que pueden quedar olvidados en cocina. El Schedule Trigger se ejecuta cada 30 minutos, consulta la hoja PEDIDOS y el código filtra únicamente los pedidos que siguen En preparación y llevan más de 45 minutos desde su creación. Si encuentra alguno, se construye una alerta con sus IDs y se envía por Telegram al supervisor. Finalmente, la ejecución queda registrada en LOGS. Esta rama funciona de manera independiente del Telegram Trigger, por eso para probarla manualmente selecciono Schedule Trigger y no necesito despublicar el bot.”
