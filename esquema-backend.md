# Necesito — Mapeo de datos del frontend a esquema de backend

Este documento traduce las estructuras de JavaScript que hoy viven en memoria
(`needs`, `PEOPLE`, `DMS`, `notifs`, `saved`) a un esquema de base de datos
relacional (PostgreSQL). El objetivo es que sea el punto de partida cuando
abras el proyecto en Claude Code: cada tabla indica de qué variable del
frontend sale, y qué endpoints necesita.

> Convención: todos los IDs son `uuid` salvo que se indique lo contrario.
> Los campos marcados **(nuevo)** no existen hoy en el frontend — hay que
> agregarlos porque el prototipo asumía un solo usuario real y datos
> ficticios.

---

## 1. `users` — reemplaza a `PEOPLE{}`

Hoy `PEOPLE` es un objeto fijo en el código con ~17 personas inventadas
(`yo`, `yo_neg`, `torres`, `motor`, `vecina`, etc.). Cada una de esas debe
convertirse en un registro real, y `yo`/`yo_neg` deja de ser un caso
especial: pasan a ser dos **cuentas** del mismo usuario dueño de WISA.

| Campo actual (`PEOPLE[k]`) | Columna sugerida         | Tipo           | Notas |
|---|---|---|---|
| `n` (nombre)                | `nombre`                 | `text`         | |
| `e` (emoji avatar)          | `avatar_url`             | `text`         | El emoji era un placeholder; pasa a URL de imagen real |
| `ac` ("persona"/"negocio")  | `tipo_cuenta`            | `enum`         | `persona`, `negocio` |
| `st` (rating 1-5)           | *(calculado)*            | —              | No se guarda: se calcula con `AVG` sobre `reviews` |
| `di` (distrito)             | `distrito_id`            | `fk → distritos`| |
| `v` (verificado bool)       | `verificado`             | `boolean`      | |
| `neu` (cuenta nueva)        | *(calculado)*            | —              | `created_at` reciente reemplaza esta bandera |
| `rol` (texto libre)         | `rol_publico`            | `text`         | |
| `ops` (operaciones)         | *(calculado)*            | —              | `COUNT` sobre `needs` cerradas donde participó |
| `rev` (nº reseñas)          | *(calculado)*            | —              | `COUNT` sobre `reviews` |
| `id` (ID ficticio, ej. "784512") | `codigo_publico`    | `text` único   | Vitrina pública, no el PK real |
| `desc`                      | `descripcion`            | `text`         | |
| `services[]`                | tabla aparte: `servicios_ofrecidos` | — | `negocio_id, etiqueta` |
| `resp` (texto tiempo resp.) | *(calculado)*            | —              | Derivar de `tiempo_respuesta_prom_seg` |
| `respRate`                  | *(calculado)*            | —              | `% mensajes respondidos / recibidos` |
| `since`                     | `created_at`             | `timestamptz`  | |
| `locFull`                   | *(join)*                 | —              | Se arma con `distrito + provincia + region` |
| **(nuevo)**                 | `dni_o_ruc`              | `text`         | Para verificación real de identidad |
| **(nuevo)**                 | `email`, `password_hash`, `telefono` | — | Autenticación (ver sección 6) |

**Endpoints:** `GET /users/:id`, `PATCH /users/:id`, `GET /users?q=&tipo=&distrito=` (para Explorar).

---

## 2. `needs` — la tabla central

Sale directo de `needs[id] = {...}` en `addNeed()`.

| Campo actual        | Columna            | Tipo               | Notas |
|---|---|---|---|
| `id`                 | `id`                | `uuid` (PK)        | Hoy es un contador `++nid` en memoria |
| `t` (texto)          | `titulo`            | `text`             | |
| `tipo`               | `tipo`              | `enum`             | `Producto`, `Servicio` |
| `by`                 | `creador_id`        | `fk → users`       | |
| `di`                 | `distrito_id`       | `fk → distritos`   | |
| `at`                 | `created_at`        | `timestamptz`      | |
| `closed`             | `estado`            | `enum`             | Mejor que boolean: `abierta`, `cerrada` |
| `revealed`           | *(eliminar)*        | —                  | Era para simular el retraso de ofertas; el backend no lo necesita |
| `picked`             | `oferta_ganadora_id`| `fk → offers`, nullable | |
| `verif`              | `estado_verificacion` | `enum`           | `pendiente`, `ok`, `con_observaciones` |
| `delivered`          | `entregado`         | `boolean`          | |
| `deliveredPhoto`     | `entrega_foto_url`  | `text`, nullable   | **(nuevo)** debe ser una URL real, no un nombre inventado |
| `dispute`            | `en_disputa`         | `boolean`          | |
| `rating`             | *(mover)*           | —                  | Pasa a su propia tabla `reviews` |
| `ev.buyer[]` / `ev.seller[]` | tabla aparte: `evidencias` | — | `need_id, lado, archivo_url, subido_por` |

**Endpoints:** `POST /needs`, `GET /needs?distrito=&tipo=&estado=`, `PATCH /needs/:id/cerrar`, `PATCH /needs/:id/entregar`, `PATCH /needs/:id/confirmar`.

---

## 3. `thread_entries` — el hilo unificado de mensajes y ofertas

Esta es la parte más importante de portar bien, porque en el frontend
`n.thread[]` mezcla dos tipos de objetos distintos en un mismo arreglo.
Conviene modelarlo como **una tabla con un discriminador**, no dos tablas
separadas, para no perder el orden cronológico real que ya tienes resuelto
en la UI.

| Campo actual (ambos tipos) | Columna | Tipo | Notas |
|---|---|---|---|
| `id`                 | `id`            | `uuid` (PK)     | |
| `type` ("msg"/"offer")| `tipo_entrada`  | `enum`          | Discriminador |
| — (implícito: `n`)   | `need_id`       | `fk → needs`    | |
| `at`                 | `created_at`    | `timestamptz`   | |
| `w` (mensaje) / `k` (oferta) | `autor_id` | `fk → users`  | Unificar: ambos apuntan a quién escribió |
| `x` (solo msg)       | `texto`         | `text`, nullable| |
| `ph` (solo msg)      | `es_foto`       | `boolean`       | |
| `re` (responde a)    | `responde_a_id` | `fk → thread_entries`, nullable | |
| `pr` (solo oferta)   | `precio`        | `numeric(10,2)`, nullable | |
| `c[]` (condiciones)  | `condiciones`   | `jsonb` o tabla `offer_conditions` | 3 etiquetas fijas hoy; si van a ser libres, mejor tabla aparte |
| `msg` (nota opcional de oferta) | `nota`  | `text`, nullable | |
| `mine`               | *(eliminar)*    | —               | Era relativo al usuario viendo la app; se resuelve comparando `autor_id` |
| `edited`             | `editado`       | `boolean`       | |
| `seen`               | *(eliminar)*    | —               | Ya no se usa (quitamos el badge Nueva/Leída) |

**Endpoints:** `GET /needs/:id/thread`, `POST /needs/:id/thread` (mensaje u oferta según `tipo_entrada`), `PATCH /thread_entries/:id` (solo ofertas propias, para editar), `DELETE /thread_entries/:id` (retirar oferta).

---

## 4. `dm_conversations` + `dm_messages` — reemplaza a `DMS{}`

Hoy `DMS[key] = {msgs: [{mine, x, at}]}` está indexado por la otra persona,
asumiendo que solo existe "yo" como usuario real. Con usuarios reales
necesitas identificar la conversación por **par de personas**, no por una
sola clave.

| Campo actual (`DMS[key].msgs[i]`) | Columna | Tipo | Notas |
|---|---|---|---|
| — (clave del objeto) | `dm_conversations.usuario_a_id`, `usuario_b_id` | `fk → users` | Un registro de conversación por par |
| `mine`               | `remitente_id`  | `fk → users`    | Reemplaza el booleano relativo |
| `x`                  | `texto`         | `text`          | |
| `at`                 | `created_at`    | `timestamptz`   | |
| **(nuevo)**          | `leido_en`      | `timestamptz`, nullable | Para saber si ya se vio (hoy no existe) |

**Endpoints:** `GET /dm/conversaciones`, `GET /dm/:conversacion_id/mensajes`, `POST /dm/:conversacion_id/mensajes`.

---

## 5. `notifications` — reemplaza a `notifs[]` y `pushNotif()`

Hoy las notificaciones las genera el propio navegador (`pushNotif` se llama
desde el cliente cada vez que pasa algo). En producción **el servidor** debe
ser quien las crea, para que lleguen igual si el usuario entra desde otro
dispositivo.

| Campo actual   | Columna       | Tipo    | Notas |
|---|---|---|---|
| `t` (tipo)      | `tipo`        | `enum`  | `respuesta`, `oferta`, `ganada`, `pago`, `resena`, `expiro`, `verif`, `cerca` |
| `g` (grupo "Hoy"/"Ayer") | *(calculado)* | — | Se deriva de `created_at` en el frontend, no se guarda |
| `h` (hora)      | `created_at`  | `timestamptz` | |
| `x` (texto HTML)| `texto`       | `text`  | Mejor guardar datos crudos (`need_id`, nombres) y armar el texto en el cliente, no HTML ya armado |
| `u` (no leída)  | `leida`       | `boolean` | |
| —               | `usuario_id`  | `fk → users` | Falta hoy: son globales para "yo" |

**Endpoints:** `GET /notificaciones`, `PATCH /notificaciones/marcar-leidas`. Idealmente con WebSocket o push para que lleguen en vivo, no solo al refrescar.

---

## 6. Lo que hoy no existe y hay que construir desde cero

Esto es tan importante como el mapeo de arriba — ninguna de estas piezas
tiene ni un borrador en el frontend actual:

- **Autenticación real.** El toggle Comprador/Vendedor solo cambia una
  variable (`acctSel`) en el navegador; no hay login. Necesitas `auth`
  (email/password o Google/DNI) y sesiones (JWT o cookies).
- **Verificación de identidad.** Los badges "✅ DNI" / "🏪 RUC" son solo
  texto decorativo hoy. Si de verdad quieres verificar RUC/DNI en Perú,
  eso es integración con un proveedor externo (RENIEC/SUNAT o un servicio
  intermedio), no una tabla más.
- **Almacenamiento de archivos.** Cada "foto" (evidencia, avatar, adjunto
  de chat) es hoy un nombre de archivo inventado como texto
  (`"evidencia_1.jpg"`). Necesitas subida real a algo como Supabase Storage
  o S3, y guardar la URL resultante.
- **Pagos.** No hay ni un solo campo de pago real; `paid:false` existe en
  el modelo pero nunca se usa. Si vas a cobrar contra-entrega nada más,
  quizás no necesitas pasarela de pago todavía — pero si planeas cobrar en
  la app, es un proveedor aparte (Culqi, Niubiz, etc.), no una tabla.
- **Tiempo real.** Los mensajes y ofertas simuladas hoy "llegan" con
  `setTimeout`. Para que dos usuarios reales se vean los mensajes sin
  refrescar, necesitas WebSockets (o Supabase Realtime / Pusher) sobre
  `thread_entries` y `dm_messages`.

---

## 7. Resumen de tablas

```
users              — perfiles reales (reemplaza PEOPLE)
distritos          — catálogo de distritos (Chilca, El Tambo, Huancayo...)
servicios_ofrecidos— servicios de un negocio (antes PEOPLE[k].services[])
needs              — necesidades publicadas
thread_entries     — mensajes + ofertas de una necesidad, en un solo hilo
offer_conditions   — condiciones de cada oferta (si se vuelven libres)
evidencias         — fotos de evidencia en una disputa
reviews            — calificaciones (antes need.rating)
dm_conversations   — una fila por par de usuarios que se escriben
dm_messages        — mensajes dentro de una conversación privada
notifications      — notificaciones por usuario
saved_needs        — reemplaza saved{} (guardados del usuario)
```

`saved{}` en el frontend es un objeto `{[need_id]: true}` — se traduce
directo a una tabla puente `saved_needs(usuario_id, need_id)`.
