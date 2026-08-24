# Pruebas | Llamadas, caso de uso por caso de uso

>
> Fecha de redacción: **2026-08-24**.
>

---

## 0. Antes de empezar

### 0.1 La puesta en estado no se reinventa


| Regla | Por qué |
|---|---|
| El trato se crea **por API con el token de Alana** | `branch_True_ALANA` filtra por `creator_user_id`. Un trato creado a mano **no entra al bot** |
| **Teléfono en la mano al crear el trato** | El inicializador crea y **llama en la misma corrida**: no hay estado intermedio que mirar |
| **Un trato por prueba**, o teléfono distinto | Todas terminan en estado terminal y ninguna es reversible |
| **Contesta dentro de los 10 min** | Pasado ese umbral el barrido recoge el trato, le gasta un intento y la prueba arranca sucia |
| La hora que ves **no es la que se guardó** | Pipedrive renderiza en tu huso; Guatemala es **UTC−6 fijo**, sin horario de verano. Resta antes de reportar |
| **El panel de un nodo muestra su última ejecución** | Para saber qué corrió en *esta* llamada, la traza |
| **Cualquier mensaje del `ERROR_HANDLER` en Chat es un fallo** | Salvo donde esta hoja diga lo contrario (L5) |

**Sembrar estado sí está permitido, pero solo estos campos y solo donde la prueba lo pida:**
`contact_round`, `round_attempts`, `call_counter`, `call_schedule`. Es lo mismo que hizo la
ronda M. **Anota siempre los valores de partida**: sin ellos la prueba no se puede releer.
Todo lo demás —`call_activity_id`, etapa, etiquetas— lo tienen que escribir los flujos: si lo
escribes tú, la prueba verifica tu escritura y no la del bot.

### 0.2 Los campos que se miran en **todas** las pruebas

```
VAI-ERP (deal):  call_counter · contact_round · round_attempts · call_status
                 call_schedule · call_activity_id · last_error
Pipedrive:       actividad abierta (asunto + fecha) · actividad cerrada (cuál + notas)
                 etapa (498/499/500) · etiqueta (3360/3361/3362/3363) · status del trato
Chat:            ¿llegó aviso del ERROR_HANDLER?
```

**Vocabulario vigente de `call_status`** —el del canvas, no el del modelo objetivo—:
`programada · saliente · dispatched · fallida · completada · error`.
El modelo nuevo los renombra (`rellamada`, `perfilado`, `no_califica`…): es el pendiente
**V2**, sigue abierto. **No lo reportes como fallo**: si ves `completada` donde el modelo dice
`perfilado`, es esa divergencia.

### 0.3 Qué NO es un fallo — los gaps ya conocidos

Se listan aquí para que nadie los cuente dos veces. Si aparecen, se anotan como *confirmado*,
no como defecto:

| Gap | id |
|---|---|
| Reintentos y tope: el cliente pidió **cada 1–2 min, máximo 3 llamadas**; el acta vigente son 5 contactos × 3 | *(pedido del cliente, sin decidir)* |
| Ventana **8:30**–19:00 pedida en la reunión; implementada **08:00**–19:00 | *(ídem)* |
| **No existe registro inmutable por llamada** (`call_id`, `attempt_in_round`, `result`): solo el acumulado del deal | **V5** |
| El **punto cero de `visit_date`** sigue siendo el placeholder `{{call_start_time}}` | **E10** |
| El lead perdido **no se borra** del CRM | *(decisión del cliente)* |
| El inicializador tiene **trigger manual**, no cron | **C9** *(a propósito, hasta producción)* |
| `escalate_Asesor` crea su actividad **en VAI-ERP**, no en Pipedrive: no es duplicado | *(verificado 2026-08-22)* |

### 0.4 Dónde está el riesgo de esta ronda

Cinco pruebas cargan casi todo. Si hay poco tiempo, se corren estas:

| Prueba | Qué mide de verdad |
|---|---|
| **L3** candado | El **doble despacho** es el defecto más caro del proyecto, y `update_Saliente` del cron **tiene el puerto de fallo suelto** (**C12 a**): si la marca no se graba, la llamada sale igual y nadie se entera |
| **L8** intervalo entre intentos | `update_Fallida` **no escribe `call_schedule`**: el intento siguiente depende de una fecha ya vencida. Nadie ha cronometrado nunca esa cadena |
| **L15** contestada sin datos | Es el único camino donde el contador de consumo se puede **contar dos veces** |
| **L18** deja de llamar | Un desenlace terminal que no apague el cron = llamadas al cliente después de cerrar el caso. Es lo que el cliente vio en el bot viejo |
| **L19** tope del filtro | `get_Leads` filtra `call_counter:lt:15`, que es el tope del **modelo viejo**. Un trato que llegue a 15 sin estar agotado **desaparece del cron en silencio** |

---

## A · Origen de la llamada

### L1 · La primera llamada — el inicializador

**Caso de uso:** entra un trato con la etiqueta 3359 y el bot llama por primera vez.

**Estado inicial:** trato nuevo por P1. Nada sembrado.
**Guion:** contesta y cuelga en cuanto Alana salude. El desenlace da igual aquí.

| Qué | Esperado |
|---|---|
| Lead en VAI-ERP | existe, con `call_counter = 1`, `contact_round = 1`, `round_attempts = 0` |
| `call_schedule` | lo escribió el inicializador al crear el lead (`dueUtc`), **no está vacío** |
| Actividades | **Correo** y **WhatsApp** creadas **y cerradas** en la misma corrida |
| Actividad | **Llamada (1er contacto)** abierta, y `call_activity_id` **apunta a ella** |
| Etiquetas | la **3359 ya no está**; la 3360 sí; las etiquetas del equipo de ventas **siguen** |
| Secuencia | `programada → saliente → dispatched`, y de ahí lo que decida la extracción |

**El punto que más calla al fallar** es `call_activity_id`: si quedó en `0` o apunta a otra
actividad, las viñetas se escriben en el sitio equivocado y el trato se ve "casi bien".

**`call_counter = 2` tras una sola llamada = doble despacho** → salta a L3. Pero **mira
`last_error` primero**: si lo escribió el barrido, la causa es L4, no el candado.

---

### L2 · La llamada que despacha el cron

**Caso de uso:** un trato con hora agendada llega a su hora y el cron lo marca.

**Estado inicial:** trato ya en el flujo, `call_status = programada`, `call_schedule`
**vencido** (siembra una hora en el pasado, dentro de la ventana operativa).
**Guion:** no hace falta contestar; lo que se mide es el despacho.

| Qué | Esperado |
|---|---|
| Tiempo hasta la llamada | **un tick del cron** (confirma el intervalo real en el trigger antes de cronometrar; en la v2 eran 2 min) |
| Antes de `Make Call` | el trato pasa por `saliente` **con `call_counter` ya incrementado** |
| Después | `dispatched`, **sin** segundo incremento |
| `branch_toca_ahora` | con `call_schedule` **futuro** el trato **no** sale — pruébalo también al revés, sembrando una hora dentro de 1 h |

**Si con `call_schedule` vacío el trato tampoco sale, anótalo:** es la pregunta abierta de L8.

---

### L3 · El candado — que no salgan dos llamadas 🔴

**Caso de uso:** dos flujos ven el mismo trato en el mismo minuto.

**Estado inicial:** trato en `programada` con `call_schedule` vencido.
**Guion:** deja correr **tres ticks del cron** sin contestar, y **cuelga sin contestar** si
suena.

| Qué | Esperado |
|---|---|
| Llamadas recibidas | **una sola por tick**, nunca dos simultáneas |
| `call_counter` | sube **exactamente 1 por llamada real recibida** |
| Un trato en `dispatched` reciente | **el cron lo salta** (`condition_2` → vuelve al loop), no lo vuelve a marcar |

**Si ves dos llamadas del mismo tick**, el sospechoso está identificado: `update_Saliente` del
cron tiene el **puerto de fallo suelto** (**C12 a**). Con ese puerto sin arista, si el PATCH
del candado falla el motor sigue por `success` hasta `Make Call` y **la llamada sale igual**.
Adjunta la traza del nodo: es la evidencia que hace falta para cerrar C12.

---

### L4 · El barrido de `saliente` — 10 minutos

**Caso de uso:** la ejecución muere entre el candado y el resultado. Nadie levanta el trato.

**Estado inicial:** trato al que **dejas sonar sin contestar**, o sembrado en
`call_status = saliente` con el `updated_at` ya viejo.
**Guion:** esperar **más de 10 minutos**.

| Qué | Esperado |
|---|---|
| `call_status` | pasa a **`fallida`** |
| `last_error` | escrito, y dice que lo movió el barrido |
| `call_counter` | **lo incrementa** — la llamada se despachó, consumió presupuesto |
| Chat | **sin aviso**: el barrido de `saliente` no notifica |

---

### L5 · El barrido de `dispatched` — 30 minutos

**Caso de uso:** la llamada salió pero la extracción nunca escribió el resultado.

**Estado inicial:** trato en `dispatched` con más de 30 min sin tocar.
**Guion:** esperar.

| Qué | Esperado |
|---|---|
| `call_status` | **`fallida`** + `last_error` |
| `call_counter` | **NO lo incrementa** — ya se contó al despachar |
| Chat | **sí llega aviso**, y **es correcto**: es la única alerta esperada de la ronda |

**Ojo con el umbral:** son 30 min a propósito (**B4**). Con un valor bajo el barrido marcaría
`fallida` llamadas **en curso**. Si ves un trato recogido antes de los 30 min, es un fallo.

---

### L6 · Dos tratos en la misma corrida

**Caso de uso:** el cron encuentra más de un trato pendiente en el mismo tick.

**Estado inicial:** **dos** tratos listos para llamar a la vez (`programada`, `call_schedule`
vencido), con teléfonos distintos.
**Guion:** no contestes ninguno.

| Qué | Esperado |
|---|---|
| Los dos | se procesan, **ninguno se queda sin turno** |
| Contadores | cada trato con **su propio** `call_counter`; nada se mezcla |
| Paginación | `get_Leads` trae **un lead por página** (`limit=1`) y avanza por cursor: con dos tratos tiene que haber **dos páginas** en la traza |

**Esto contesta la pregunta abierta de plataforma** de `otros_canvas_nodos.md` §3: si el
`loop_iterator` avanza solo o hay que devolverle la ejecución con una arista. Anota la
respuesta, se cita en varios sitios.

---

## B · Nadie contesta

### L7 · Primera fallida — los contadores

**Estado inicial:** trato nuevo (L1 sirve de puesta en estado si su desenlace fue fallido).
**Guion:** **no contestes**.

| Qué | Esperado |
|---|---|
| `call_counter` | **1** |
| `round_attempts` | **1** |
| `contact_round` | **1**, sin moverse |
| `call_status` | `fallida` |
| Actividad | **Llamada (1er contacto)** sigue **abierta**, con **1 viñeta** |
| La viñeta | *"Llamada fallida"* o *"Llamada sin conversación: …"*, **sin link de grabación** — nadie contestó, no hay grabación que enlazar. Un enlace muerto aquí es fallo |
| Etapa | **498 Contactados** (solo se mueve en la primera llamada) |

---

### L8 · El intervalo entre los intentos 1 → 2 → 3 🔴

**Caso de uso:** *"los 3 intentos de un contacto son consecutivos"* (acta). Nadie ha medido
nunca cuánto tarda de verdad el segundo intento.

**Estado inicial:** el trato de L7, en `fallida` con `round_attempts = 1`.
**Guion:** no contestes ninguna de las dos llamadas siguientes. **Cronometra.**

| Qué | Esperado |
|---|---|
| 1º → 2º intento | **el siguiente tick del cron**. `update_Fallida` **no reescribe `call_schedule`**, así que la fecha sigue vencida y el trato vuelve a estar listo de inmediato |
| 2º → 3º | igual |
| Entre medias | `round_attempts` = 1 → 2 → 3, `call_counter` = 1 → 2 → 3, `contact_round` quieto en 1 |
| Viñetas | **una por llamada, las tres en la misma actividad** |

**Anota el intervalo real en minutos.** Es el dato que el cliente pidió en la reunión
(*"cada 1–2 minutos"*, 00:00:59) y hoy nadie lo tiene medido.

**Y el caso que hay que descartar:** si el trato **nunca vuelve a sonar** después de la
primera fallida, el problema es `branch_toca_ahora` con un `call_schedule` vacío o
inservible — el trato queda vivo pero mudo, sin aviso en Chat. Es el fallo más silencioso de
esta rama; repórtalo con la traza del branch.

---

### L9 · La tercera fallida cierra el contacto

**Estado inicial:** el trato de L8 con `round_attempts = 2`.
**Guion:** no contestes la tercera.

| Qué | Esperado |
|---|---|
| `call_counter` · `round_attempts` · `contact_round` | **3 · 0 · 2** |
| Actividad vieja | **cerrada** (`done`), con sus **3 viñetas** |
| Actividad nueva | abierta, asunto **`Llamada (2do contacto).`** |
| `call_activity_id` | apunta a la **nueva** |
| WhatsApp | sale la plantilla **`contacto1_alana`**, y `reminder_counter = 1` |
| Actividad de WhatsApp | creada en Pipedrive |
| Nota de cierre | la línea del contacto 1, coherente con la fecha que se calculó |
| `call_schedule` | **+6 h** si la 3ª llamada fue **antes de las 14:00 GT**; si no, **mañana 09:00 GT** — en UTC, recortado a la ventana |
| `call_status` | `programada` |

Es la regresión de **M3** (verde el 2026-08-22). Si algo de esta tabla falla, se rompió después
de esa fecha.

---

### L10 · La escalera entre contactos — 24 h · 48 h · 7 días

**Caso de uso:** la espera hasta el contacto siguiente depende de **qué contacto se acaba de
cerrar**, no de cuántas llamadas van.

**No se puede esperar 7 días.** Se prueba **sembrando** `contact_round` y `round_attempts = 2`
y dejando fallar una sola llamada. Se mide el `call_schedule` resultante, no el reloj.

| Siembra `contact_round` | Cierra el contacto… | `call_schedule` esperado |
|---|---|---|
| **2** | 2 | **+24 h** desde la última llamada |
| **3** | 3 | **+48 h** |
| **4** | 4 | **+7 días** |
| **5** | 5 | **vacío** — no hay contacto siguiente: es L17 |

Y en las tres: `contact_round` sube 1, `round_attempts` vuelve a **0**, la actividad se cierra,
se abre la del contacto siguiente con el ordinal correcto (`3er`, `4to`, `5to`) y sale la
plantilla `contacto2_alana` / `contacto3_alana` / `contacto4_alana`.

**La escalera se elige por el contacto que se cerró, sin importar cómo se cerró.** Si el
contacto 1 se cerró por rellamada y el 2 se agota, la espera es **+24 h**, la del 2.

---

## C · Cuándo se puede llamar

### L11 · El recorte a la ventana operativa

**Caso de uso:** la cadencia cae fuera del horario en que se puede llamar.

**Se mide sobre `call_schedule`**, no esperando. Siembra `contact_round` y provoca el cierre
del contacto a horas elegidas:

| La última llamada cae… | `call_schedule` esperado (GT) |
|---|---|
| Viernes 16:00, contacto 2 (+24 h) | sábado, **08:00–12:00** si cabe; si el cálculo da sábado **después de las 12:00** → **lunes 08:00** |
| Sábado 11:00, contacto 2 (+24 h) | domingo está cerrado → **lunes 08:00** |
| Cualquier día, resultado **después de las 19:00** | **día siguiente 08:00** |
| Resultado **antes de las 08:00** | **ese mismo día 08:00** |

**El recorte es un bucle, no un salto único**: empujar un sábado por la tarde da domingo, y el
domingo hay que volver a empujarlo al lunes. El caso *sábado 13:00* es el que lo prueba.

**Y el texto tiene que ir con la fecha:** si el `+6 h` se salió de la ventana, la nota **no**
puede decir *"más tarde hoy"*. Se lee la viñeta, no solo el campo.

---

### L12 · Un trato que entra fuera de horario — *gap esperado*

**Caso de uso:** el equipo de ventas etiqueta un trato a las 21:00.

**Estado inicial:** trato nuevo creado **fuera de la ventana** (después de las 19:00 GT, o en
domingo).
**Guion:** disparar el inicializador.

| Qué | Qué hace hoy |
|---|---|
| La llamada | **sale igual**: el inicializador llama en la misma corrida y **no consulta la ventana**. El recorte solo lo aplica `code_Cadencia`, que corre al cerrar un contacto |

**Esto se anota, no se arregla en esta ronda.** Hoy el inicializador es **manual** (**C9**), así
que solo se dispara en horario laboral; el día que pase a cron, esta prueba se convierte en un
defecto real. Es justo el dato que hace falta para decidirlo.

---

## D · El cliente contesta

### L13 · Rellamada en la primera llamada — *regresión de M1*

**Estado inicial:** trato nuevo. `call_counter = 0`, `contact_round = 1`, `round_attempts = 0`.
**Guion:** contesta la primera llamada y di **la frase completa**:

> *"Hoy no puedo, llámame mañana a las diez de la mañana."*

| Qué | Esperado |
|---|---|
| `call_counter` · `contact_round` · `round_attempts` | **1 · 2 · 0** — el contacto se cerró con **una sola llamada** |
| `call_status` · `call_schedule` | `programada` · mañana **10:00 GT** en UTC, recortado |
| Actividad vieja | **cerrada**, con su nota |
| Actividad nueva | abierta, **`Llamada (2do contacto).`**, con la viñeta de la reprogramación |
| Etapa | **499 Interesados** |
| **Lo que NO debe pasar** | que salgan los intentos **2 y 3 del contacto 1**. Ahí están las 2 llamadas ahorradas. Si el trato vuelve a sonar antes de mañana, el modelo no cerró el contacto |

Verde el 2026-08-22 (M1). Se repite porque es **la prueba que justifica todo el modelo**.

---

### L14 · La rellamada agendada tampoco contesta — *regresión de M5*

**Estado inicial:** el trato de L13, a la hora agendada.
**Guion:** no contestes.

| Qué | Esperado |
|---|---|
| `call_counter` | 2 |
| `round_attempts` | **1** — es la **primera fallida del contacto nuevo**, no tiene tratamiento propio |
| `contact_round` | **2**, quieto |
| Actividad | la del contacto 2, **abierta**, con su viñeta |

Confirma **D4**, que sigue siendo una pregunta abierta del modelo: si el cliente decide otra
cosa, esta prueba cambia.

---

### L15 · Contestó, colgó, sin datos — el contador que se puede contar dos veces 🔴

**Caso de uso:** el cliente descuelga, dice dos palabras y cuelga. Hubo conexión pero no hay
perfilamiento.

**Estado inicial:** trato nuevo.
**Guion:** contesta, **di "¿bueno?" y cuelga de inmediato**, antes de que Alana termine el
saludo.

| Qué | Esperado |
|---|---|
| `call_counter` | **sube exactamente 1** — lo incrementó el flujo de llamadas al despachar, y la extracción **no** lo vuelve a tocar |
| `round_attempts` | sube 1 |
| `contact_round` | quieto |
| `call_status` | `fallida` — reintenta como cualquier otra |
| Actividad | **sigue abierta**, con la viñeta *"Llamada sin conversación: …"* |
| El motivo | el `end_call_motive` **aparece en la viñeta**: con el prefijo vacío la nota se quedaría igual y el intento no dejaría rastro |

**Un `call_counter` en 2 después de esta única llamada es el defecto que busca la prueba**:
doble conteo entre el flujo de llamadas y la extracción. Descarta antes el barrido mirando
`last_error`.

---

### L16 · Rellamada en el contacto 5 — el escalado al asesor · *regresión de M2*

**Estado inicial:** `contact_round = 5`, `round_attempts = 0`, `call_status = programada`,
`call_schedule` vencido.
**Guion:** contesta y agenda para pasado mañana.

| Qué | Esperado |
|---|---|
| `contact_round` | **5** — no 6. No existe el contacto 6 |
| `call_status` | `completada` |
| `call_schedule` | **vacío** ← sin esto el cron vuelve a llamar. **Es el campo que hay que mirar** |
| Dueño del trato | **23131149**, el asesor |
| Actividad | `Atender cliente (cita agendada, 5to contacto)`, del asesor, con la fecha que pidió el cliente |
| Llamadas posteriores | **ninguna**. Quédate mirando 10 min |

---

## E · Cuándo deja de llamar

### L17 · Agotar sin llegar a 15 — *regresión de M4*

**Caso de uso:** los 5 contactos se cierran, pero por rellamadas se gastaron menos de 15
llamadas.

**Estado inicial:** `contact_round = 5`, `round_attempts = 2`, **`call_counter = 8`** — a
propósito lejos de 15.
**Guion:** no contestes.

| Qué | Esperado |
|---|---|
| Agotado | **sí**, con `call_counter = 9` |
| `round_attempts` | **3** persistido — no se queda en 2: si se queda, el barrido da el trato por vivo |
| Etiqueta · trato | **3362 Intentos agotados** · `status: lost` |
| WhatsApp | **ninguno** — el contacto 5 no manda despedida |
| Actividad nueva | **ninguna** |
| `call_schedule` | vacío |
| El cron | **no lo vuelve a tomar** |

Con el modelo viejo esto era imposible: el agotamiento disparaba en `call_counter == 15` y este
trato habría llamado para siempre.

---

### L18 · Un desenlace terminal apaga las llamadas 🔴

**Caso de uso:** el caso se cerró. El teléfono **no puede volver a sonar**.

Se corre **una vez por desenlace**. El guion de cada uno está en
[`test_v1.md` §0.2](test_v1.md) y [`test_v2.md`](test_v2.md); aquí lo que se mide es **lo que
pasa después de colgar**.

| Desenlace | Guion, en una frase | Etiqueta esperada |
|---|---|---|
| `perfilado` | perfilamiento completo, sin pedir visita | 3361 |
| `no_califica` | *"no tengo nada ahorrado y gano Q3,000 al mes"* | 3363 + `lost` |
| `visita_agendada` | *"puedo ir el viernes a las 3 de la tarde"* | 3361 + etapa 500 |
| `rechaza_ia` | *"no quiero hablar con una máquina"* | 3361 + actividad para el asesor |

**Lo que mide L18, igual en los cuatro:**

| Qué | Esperado |
|---|---|
| `call_schedule` | **vacío** |
| `call_status` | `completada` (si queda en `error`, es fallo) |
| Llamadas posteriores | **ninguna en 15 minutos**, con el cron corriendo |
| `contact_round` | **no se incrementa**: el ciclo termina, no avanza |
| Actividad de la llamada | **cerrada** |

**Si suena otra vez, es el defecto más visible para el cliente** y hay que anotar de qué rama
salió: el sospechoso es un `call_schedule` que quedó escrito o un `call_status` que el filtro
de `get_Leads` sigue aceptando.

---

### L19 · El tope del filtro — `call_counter:lt:15` 🔴

**Caso de uso:** un trato acumula 15 llamadas **sin** estar agotado. En el modelo nuevo
`call_counter` **no tiene tope** y el agotamiento se mide contra contactos.

**Estado inicial:** siembra `call_counter = 15`, `contact_round = 3`, `round_attempts = 1`,
`call_status = programada`, `call_schedule` vencido.
**Guion:** ninguno; se mira el cron.

| Qué | Qué se espera ver hoy |
|---|---|
| El cron | **no lo toma**: `get_Leads` filtra `call_counter:lt:15` |
| Chat | **ningún aviso** |
| El trato | queda **vivo y mudo**: no llama, no cierra, no alerta |

**Esto no prueba que algo funcione: mide un riesgo.** Anota el resultado tal cual — es lo que
decide si el filtro tiene que pasar a mirar `contact_round`. En la práctica solo se llega a 15
por el peor caso o por incrementos del barrido, pero cuando pasa el trato desaparece sin ruido.

---

## F · Cuando algo falla

### L20 · La llamada que no se puede hacer

**Caso de uso:** el número no existe, está fuera de servicio o Synthflow rechaza el despacho.

**Estado inicial:** trato nuevo con un **teléfono inválido** (formato correcto, número
inexistente).
**Guion:** ninguno.

| Qué | Esperado |
|---|---|
| `call_counter` | **sube 1** — se intentó, consumió despacho |
| `call_status` | el trato **no se queda en `saliente`** |
| El puerto | `Make Call` sale por `failed`, y `update_Completada` lo trata **igual que `completed`**: escribe `dispatched`. **El label engaña**, no es un fallo |
| Después | el barrido de 30 min lo recoge → `fallida` + aviso a Chat |

**Lo que hay que anotar:** cuánto tarda el trato en volver a estar disponible. Si tarda los 30
minutos del barrido, un número malo cuesta media hora de reloj por intento.

---

### L21 · La cadena de desenlace se rompe

**Caso de uso:** la llamada fue bien pero un PATCH a Pipedrive falla (4xx).

**Cómo provocarlo:** es difícil de forzar limpiamente. **Si ocurre durante cualquier otra
prueba, se documenta aquí** en vez de descartar la corrida.

| Qué | Esperado |
|---|---|
| Chat | llega el aviso del `ERROR_HANDLER`, con el `api_response` del nodo que falló |
| El trato | **sale de `dispatched`**: `call_status = error` + `last_error` (lo escribe `update_ErrorD`) |
| Lo que **no** debe pasar | que el trato quede tomado para siempre, ni que la nota anterior se **borre** |

El aviso **ya no nombra el nodo** —25 puertos comparten el webhook—, así que adjunta la traza.

---

### L22 · El cron corriendo en vacío

**Caso de uso:** `get_Leads` devuelve un 4xx y el cron sigue como si no hubiera nada que hacer.

**Cómo se detecta, sin provocarlo:** al final de la jornada de pruebas, con **al menos un trato
pendiente vivo**, revisa las últimas ejecuciones del cron.

| Qué | Esperado |
|---|---|
| Cada tick | encuentra el trato pendiente y hace algo con él |
| Un tick que termina en 0 leads teniendo uno pendiente | **es el síntoma de C12 (a)**: `get_Leads` con el puerto de fallo suelto deja a `code_Cursor` sin datos y **el cron corre en vacío sin decir nada** |

---

## 12. Hoja de resultados

Rellenar **siempre** los valores de partida: sin ellos la prueba no se puede releer.

| # | Prueba | Estado inicial (`cc`/`round`/`intentos`) | id del trato | Fecha | Resultado observado | Veredicto |
|---|---|---|---|---|---|---|
| L1 | primera llamada (inicializador) | 0 / 1 / 0 | | | | |
| L2 | despacho por cron | | | | | |
| L3 | 🔴 candado / doble despacho | | | | | |
| L4 | barrido `saliente` (10 min) | | | | | |
| L5 | barrido `dispatched` (30 min) | | | | | |
| L6 | dos tratos en la misma corrida | | | | | |
| L7 | primera fallida | 0 / 1 / 0 | | | | |
| L8 | 🔴 intervalo entre intentos | | | | **minutos medidos:** | |
| L9 | 3ª fallida cierra contacto | | | | | |
| L10a | escalera contacto 2 → 3 (+24 h) | — / 2 / 2 | | | | |
| L10b | escalera contacto 3 → 4 (+48 h) | — / 3 / 2 | | | | |
| L10c | escalera contacto 4 → 5 (+7 d) | — / 4 / 2 | | | | |
| L11 | recorte a la ventana operativa | | | | | |
| L12 | trato que entra fuera de horario | | | | *(gap esperado)* | |
| L13 | rellamada en la 1ª llamada | 0 / 1 / 0 | | | | |
| L14 | la rellamada no contesta | tras L13 | | | | |
| L15 | 🔴 contestó y colgó sin datos | 0 / 1 / 0 | | | | |
| L16 | rellamada en el contacto 5 | — / 5 / 0 | | | | |
| L17 | agotar sin llegar a 15 | 8 / 5 / 2 | | | | |
| L18a | deja de llamar — `perfilado` | | | | | |
| L18b | deja de llamar — `no_califica` | | | | | |
| L18c | deja de llamar — `visita_agendada` | | | | | |
| L18d | deja de llamar — `rechaza_ia` | | | | | |
| L19 | 🔴 tope del filtro `lt:15` | 15 / 3 / 1 | | | | |
| L20 | teléfono inválido | | | | | |
| L21 | cadena de desenlace rota | | | | *(oportunista)* | |
| L22 | cron en vacío | | | | | |

---

## 13. Lo que dijo Alana, literal

Lo que el asesor lee es la nota, así que se transcribe **tal cual**, sin resumir ni corregir.
Una fila por llamada, no por prueba.

| Prueba | Llamada | Viñeta escrita en la actividad |
|---|---|---|
| L7 | 1ª | |
| L8 | 2ª | |
| L8 | 3ª | |
| L13 | 1ª | |
| L15 | 1ª | |
| L18a–d | | |

---

## 14. Hallazgos

### 14.1 Nuevos de esta ronda

*(Vacío hasta que corran las pruebas. Un hallazgo por fila: qué se esperaba, qué pasó, id del
trato y traza.)*

### 14.2 Ya conocidos, vueltos a ver

Con su id de `pendientes.md`, para no contarlos dos veces. Se espera volver a ver **C12** (los
puertos de fallo sueltos del cron), **E10** (el punto cero de `visit_date`), **E8** (el motivo
duplicado) y **X4** (el guion de Synthflow agenda sin terminar de perfilar).

### 14.3 Cierre de la ronda

Ninguna prueba se marca cerrada sin evidencia: id del trato, fecha y traza o export. Al cerrar
un punto de `pendientes.md` con una prueba de esta hoja, **anota aquí dónde quedó la
evidencia** — ya ocurrió que se escribieran cierres inventados en varios documentos a la vez.
