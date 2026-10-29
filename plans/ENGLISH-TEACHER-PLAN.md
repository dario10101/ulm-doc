# ULM - Plan del modulo English Teacher

Ultima actualizacion: 2026-10-17 (v5: **fases 1-6 implementadas y commiteadas** en `ulm-core` y `ulm-web`, sin push; falta la verificacion manual en el navegador, ver seccion 15; trabajo futuro en la seccion 14)
Documento de planeacion del modulo de practica de ingles. Lo pendiente va primero; las decisiones se registran en la seccion 12.

---

## 1. Resumen

Practicas cortas, cada una girando en torno a **una palabra** (objeto, accion, adjetivo...), con 4 pasos fijos:

```
Vocabulario -> Listening -> Writing -> Speaking (lectura + libre) -> Cierre (check "la domino" + fin / nueva / feedback)
```

Principio rector (requisito no funcional): **el LLM solo hace lo que no se puede resolver con codigo**.

| Tarea | Quien la resuelve | Por que |
|---|---|---|
| Elegir la palabra | Codigo: palabras del nivel, sin las dominadas, mezclando nuevas y ya practicadas (seccion 6) | Determinista y gratis |
| Imagen | APIs de imagenes/pictogramas, cacheada en base y curada (seccion 5) | El LLM no es buena fuente de URLs (D2) |
| Validar la palabra escrita | Codigo en el front (comparacion normalizada + sinonimos) | Trivial |
| Audio del listening | TTS del navegador (`speechSynthesis`) | Gratis (D11) |
| Frases, consigna de writing, texto de lectura, pregunta | **LLM, una vez por palabra+nivel**, en un banco compartido | Generacion de lenguaje, reutilizada entre usuarios |
| Detectar el minimo de caracteres | Codigo | Trivial |
| Transcribir la voz | Servicio de transcripcion (seccion 2, C1) | No hay alternativa sin costo y de calidad en Cloud Run |
| Comparar lectura vs texto esperado | Codigo (diff por palabras) | Gratis y objetivo |
| Sugerir cambio de nivel | Codigo (reglas, seccion 4.1) | Ver C7: sin datos no hay modelo que entrenar |
| Feedback final | **LLM, solo si el usuario lo pide**, 1 llamada | Unico paso que necesita "criterio" |

---

## 2. Criticas a los requisitos y su estado

### C1. Transcripcion "tal cual suena" — limitacion aceptada; eleccion de herramienta

Recordatorio: ningun ASR (*automatic speech recognition*) transcribe fonemas; todos eligen la palabra **mas probable**. Errores de gramatica y vocabulario si se conservan; la **pronunciacion** solo se mide con *pronunciation assessment* (evaluacion fonetica contra un texto de referencia). Se mantienen las reglas: nada de `SpeechRecognition` del navegador, nunca pasar el texto esperado como prompt, grabar con `MediaRecorder` y transcribir en el backend detras de un Protocol `SpeechTranscriber`.

**Tres opciones:**

| | **A. OpenAI transcription** (`whisper-1`, `gpt-4o-mini-transcribe`, `gpt-4o-transcribe`) | **B. Azure AI Speech** (STT + Pronunciation Assessment) | **C. faster-whisper** (Whisper abierto, autoalojado) |
|---|---|---|---|
| Que da | Texto. `whisper-1` acepta prompt (muletillas) y timestamps por palabra | Texto **+ puntaje por palabra y fonema** contra el texto de referencia: detecta palabras omitidas, insertadas y **mal pronunciadas**. Tambien modo sin guion (fluidez, gramatica, vocabulario) | Texto + timestamps y **probabilidad por palabra** (una palabra con probabilidad baja = posible mala pronunciacion: señal debil pero gratis) |
| Costo | ~0.003-0.006 USD/min | Free tier mensual (~5 h de audio), despues pago por hora | 0 por uso; paga en CPU/RAM del servidor |
| Ventajas | Misma key y proveedor que ya usamos; integracion de una tarde; buena precision con acentos | **La unica que cumple de verdad "detectar falencias de pronunciacion"**; encaja exacto con la fase 1 (lectura de un texto conocido) | Privacidad total, sin proveedor, control fino de parametros; ideal para probar en local |
| Desventajas | No mide pronunciacion. Los modelos `gpt-4o-*` son mas "contextuales": hipotesis a validar, pueden normalizar mas los errores que `whisper-1` | Otro proveedor, cuenta de Azure (pide tarjeta), SDK mas pesado, region. Mas piezas que mantener | En Cloud Run sin GPU: imagen de cientos de MB, *cold start* cargando el modelo, transcripcion lenta con modelos medianos. Los modelos chicos (`base`/`small`) cometen mas errores propios, que se confundirian con los del usuario |

**Decision (D17): opcion A con `whisper-1`** para las dos fases, sin spike previo. Es la de menor friccion y ya tenemos el cliente.
- Riesgo aceptado: sin comparar, no sabemos cuanto "normaliza" `whisper-1` los errores del usuario. Mitigacion barata: el modelo de transcripcion queda en config, no en el codigo, para cambiarlo sin tocar logica si al probarlo no convence.
- Azure (B) para la lectura y faster-whisper (C) quedan en trabajo futuro (seccion 14).

### C2. Imagenes por LLM — descartado (D2)
Se usa la propuesta de la seccion 5.

### C3. Ambiguedad de verbos/adjetivos — mitigaciones aceptadas (D3)
Categoria gramatical visible, sinonimos aceptados, forma base para verbos, primera letra en A1/A2, pictogramas para acciones/adjetivos.

### C4. Memoria de palabras — resuelto con el check "la domino" (D4), con reparos

Requisito nuevo: al cerrar la practica aparece un check; marcado = la palabra no vuelve a salir para ese usuario. Por defecto **marcado si la acerto solo** en vocabulario, **desmarcado si la revelo**. Las practicadas y no marcadas vuelven a salir mezcladas con nuevas.

Es mas simple que la repeticion espaciada (SM-2/FSRS) y me parece aceptable para v1, pero con dos riesgos:

1. **Acertar una vez no es dominar.** Reconocer una imagen es el nivel mas bajo de conocimiento de una palabra (no garantiza usarla al escribir o hablar). Con el check marcado por defecto tras un solo acierto, la mayoria de las palabras se descartaran en la primera vista y **el banco se consume rapido**.
2. **Agotamiento del banco**: resuelto (D19): al no quedar palabras sin dominar, se repiten las dominadas y se recomienda subir de nivel.
3. **Sin deshacer** (D20): un "la domino" marcado por error no se puede revertir en v1. Mitigado en parte por D19 (las dominadas vuelven cuando se agota el resto). Queda en trabajo futuro.

### C5. Nivel manual + sugerencia — aceptado (D5)
Reglas en la seccion 4.1.

### C6. Presupuesto — se pospone (D6), pero se agrega un registro de consumo desde ya

El control de tokens vendra despues y sera transversal (vision en D7). Lo que **si conviene hacer ahora**, porque es barato y evita un retrabajo:

- **Tabla transversal `ai_usage_events`** (`user_id`, `feature` p.ej. `english.transcription`/`english.feedback`/`ai.ask`, `model`, `input_tokens`, `output_tokens`, `audio_seconds`, `estimated_cost_usd`, `created_at`), escrita **en el momento de cada llamada al proveedor**.
- Motivo concreto: como la practica solo se guarda al final (D8), si el usuario transcribe y recarga la pagina, la practica desaparece pero **el gasto ya ocurrio**. Si el consumo solo se registrara con la practica, ese gasto seria invisible para el futuro control de cupos.
- Hoy `POST /ai/ask` solo escribe los tokens en el log; se puede migrar al mismo registro (una linea).
- **Concepto: punto unico de paso (*chokepoint*)**: si todas las llamadas pagadas pasan por un mismo servicio que registra el consumo, el futuro "sin saldo -> 402/boton deshabilitado" se implementa en un solo lugar.

### C7. Modelos propios para tareas puntuales — de acuerdo con el enfoque, con una condicion: datos

Aclarado: no es reemplazar el LLM, sino resolver tareas sencillas con modelos propios, incluso clasicos (arboles de decision, regresion). De acuerdo en principio, pero:

- **Un modelo supervisado necesita datos etiquetados.** Con 1-10 usuarios no habra suficientes en meses. Antes de eso, **reglas escritas a mano** hacen lo mismo y se pueden explicar. Entrenar con 50 filas es *sobreajuste* (*overfitting*: el modelo memoriza los casos en vez de aprender el patron).
- Candidatas reales en este modulo, de mas a menos viable:

| Tarea | Hoy | Cuando haya datos |
|---|---|---|
| Sugerir cambio de nivel | Reglas (seccion 4.1) | Regresion logistica sobre el historial de practicas |
| Dificultad real de cada palabra (no solo su banda CEFR) | Banda CEFR de la lista | Regresion con la tasa de acierto/revelado de todos los usuarios |
| Pre-revision del writing (errores gramaticales comunes) | **LanguageTool** (corrector por reglas, open source, sin redes neuronales) | — |
| Elegir la siguiente palabra | Reglas (seccion 6) | SRS clasico (no necesita entrenamiento) |

- **La oportunidad mas grande no es entrenar, son las herramientas deterministas existentes**: LanguageTool podria dar un feedback basico de writing **gratis** y mandar al LLM solo lo que las reglas no cubren. Queda como trabajo futuro (D22).
- **Implicacion de diseño ya incorporada:** guardar los resultados de cada practica en columnas estructuradas (no un JSON opaco) para que, cuando haya volumen, sean un *dataset* utilizable.

### C8. Detalles menores — resueltos

- **Validacion del vocabulario**: al completar todas las letras, sin tiempo limite y **sin guardar intentos** (D9). (Aclaracion: el plan anterior no proponia tiempo limite; decia que *no* se usara una espera por tiempo para adivinar que termino de escribir.)
- **Estado de la practica**: nada se guarda hasta el final (D8). Recargar = empezar de cero. Excepcion necesaria: el registro de consumo de C6.
- **Microfono y HTTPS**: **si se puede probar en dev**. Los navegadores tratan `localhost` como contexto seguro, asi que `getUserMedia` funciona con `http://localhost`. Lo que exige HTTPS es cualquier otro origen: en produccion Cloudflare Pages ya sirve HTTPS, asi que tampoco es problema. El unico caso que falla: **probar desde el celular** con `http://192.168.x.x:5173`. Para eso, un tunel HTTPS (ej. `cloudflared tunnel`) o el certificado de desarrollo de Vite.
- **"Show text" del listening**: se guarda, junto con todo lo demas, al final de la practica.
- **Idioma**: `/admin/**` esta en ingles; este modulo es la excepcion (instrucciones en español en A1-A2).

---

## 3. Flujo detallado

Toda la practica vive en memoria del front (`usePractice`, maquina de estados) hasta el cierre.

### Paso 1 - Vocabulario
- Imagen + categoria gramatical (+ primera letra en A1/A2) + atribucion de la imagen. Nunca la palabra escrita.
- Input de N casillas (espacios/guiones fijos en compuestas, ej. *ice cream*).
- Al completar las N letras: validacion en el front contra la respuesta y los sinonimos (vienen con la practica, D10). Feedback inmediato.
- **"Show hint"** (D34): pista corta del banco, en español en A1-A2 y en ingles desde B1. Se registra si se uso; no cambia el check "la domino" (acerto solo).
- Mientras no este correcta: **"Show word"** (autocompleta, marca `revealed`, habilita continuar). Correcta: **"Continue"**.
- Si escribio un sinonimo aceptado: se da por correcta y se muestra la palabra objetivo.

### Paso 2 - Listening
- Frase con la palabra, reproducida con `speechSynthesis` (velocidad por nivel). Se puede repetir.
- Al terminar el audio (`onend`): **"Continue"** y **"Show text"**. Se registra si mostro el texto.
- Riesgo tecnico: algunas voces/navegadores no disparan `onend` de forma confiable (bug conocido en Chrome con textos largos). Mitigacion: habilitar los botones tambien por tiempo estimado si `onend` no llega.

### Paso 3 - Writing (dos fases, D33)
- **Fase 1, dictado:** escribir la frase que escucho (se puede repetir el audio). "Revisar" la compara con la frase del listening (`POST /english/dictation/check`: el mismo diff por palabras de la lectura, sin LLM ni costo) y muestra coincidencia %, palabras faltantes/cambiadas y la frase correcta. Se continua este bien o no. El backend recalcula la coincidencia al guardar la practica (`dictation_text`, `dictation_match_pct`); entra al feedback como campo `dictation`.
- **Fase 2, respuesta:** consigna especifica sobre la frase del listening, minimo de caracteres por nivel, contador visible. Al superar el minimo se habilita **"Continue"**. Sin correccion aqui (se corrige en el feedback, si se pide).

### Paso 4 - Speaking
- **Fase 1 (lectura)**: frase del listening + 2-3 frases nuevas + pregunta final (del banco). Microfono: grabar/detener -> `POST /english/speech` -> transcripcion + diff contra el texto esperado (coincidencia % y palabras omitidas/cambiadas).
- **Fase 2 (libre)**: continuar y responder la pregunta, con tope de duracion por nivel. Se transcribe; no se evalua en el momento.
- Permiso de microfono denegado: mensaje con instrucciones y opcion de **saltar el speaking** (si no, la practica queda bloqueada).

### Cierre
- Check **"I know this word"** (en español en A1-A2), marcado por defecto si no revelo la palabra.
- La practica **se guarda al llegar a esta pantalla** (D21); cambiar el check actualiza solo ese campo.
- Si la palabra es una dominada que se repite (D19), el check **no se muestra**: desmarcarlo seria una forma encubierta de "deshacer", que esta fuera de v1 (D20).
- **Finish for today**: sale de la pagina de estudio (D12).
- **New practice**: siguiente palabra.
- **Get feedback**: 1 llamada al LLM; se guarda y se puede volver a ver sin pagar de nuevo.

---

## 4. Adaptacion por nivel (CEFR)

Tabla de configuracion en codigo (`english_levels.py`). Valores iniciales, a ajustar con uso real:

| Nivel | Palabras | Pista 1a letra | Velocidad TTS | Frase listening | Writing min. | Texto lectura | Speaking libre | Instrucciones |
|---|---|---|---|---|---|---|---|---|
| A1 | banda A1, objetos concretos | si | 0.8 | 4-7 palabras | 30 car. | 3 frases cortas | 30 s | Español |
| A2 | banda A2 | si | 0.9 | 6-10 | 60 | 3-4 frases | 45 s | Español |
| B1 | banda B1, mas verbos/adjetivos | no | 1.0 | 8-14 | 120 | 4 frases | 60 s | Ingles |
| B2 | banda B2 | no | 1.0 | 10-18 | 200 | 4-5 frases | 90 s | Ingles |
| C1 | banda C1, abstractas | no | 1.1 | 12-22 | 300 | 5 frases | 120 s | Ingles |
| C2 | banda C2 | no | 1.1 | libre | 400 | 5-6 frases | 120 s | Ingles |

El contenido se genera **solo para el nivel del usuario** (D13). El banco de cada nivel lo llena el admin desde la configuracion del sistema (D18, seccion 5.1). Un nivel sin banco aparece deshabilitado en el selector de nivel del usuario ("not available yet").

### 4.1 Sugerencia de cambio de nivel (reglas, sin LLM)

Sobre las **ultimas 10 practicas completadas en el nivel actual**:

| Señal | Subir si | Bajar si |
|---|---|---|
| Palabras acertadas sin revelar | ≥ 80% | ≤ 30% |
| Listening sin "Show text" | ≥ 80% | ≤ 30% |
| Coincidencia promedio en la lectura | ≥ 90% | ≤ 60% |

- Sugerir **subir** si se cumplen las 3 condiciones; **bajar** si se cumplen 2 de 3.
- Se muestra como banner en la pagina de estudio ("Parece que B1 te queda facil. ¿Probar B2?") con botones *cambiar* / *no por ahora*.
- "No por ahora" guarda `level_suggestion_dismissed_at`; no se vuelve a sugerir hasta 10 practicas mas.
- Solo se sugiere un nivel **con banco disponible**.
- Disparador adicional (D19): si el usuario ya domino todas las palabras de su nivel, se sugiere subir aunque no se cumplan los umbrales.
- El cambio manual sigue en Settings (y el boton de la pagina de estudio lleva ahi).
- Los umbrales son una suposicion: ajustarlos con uso real. Son justo el tipo de regla que en el futuro podria reemplazar una regresion (C7).

---

## 5. Imagenes

URL **resuelta una vez por palabra y guardada en base**, con fuente y atribucion:

| Fuente | Para que | Costo / licencia | Notas |
|---|---|---|---|
| **ARASAAC** (pictogramas) | Acciones, adjetivos, conceptos | Gratis, API publica sin key; CC BY-NC-SA (no comercial, exige atribucion) | Diseñado para aprendizaje y comunicacion; cubre verbos y emociones mejor que la fotografia |
| **Pexels** / **Unsplash** API | Objetos concretos (foto) | Gratis con key; exigen atribucion (Unsplash ademas hotlink + evento de descarga) | Revisar que no tenga texto |
| **Openverse** (Wikimedia + CC) | Respaldo: **solo si Pexels/ARASAAC no devolvieron ninguna imagen** (2026-10-17) | Gratis, sin key: 20 pedidos/min y 200/dia | Guardar la licencia por imagen. Usarlo para completar candidatas agotaba el cupo en un llenado |
| Generar con modelo de imagen | Huecos sin imagen clara | ~0.01-0.04 USD/imagen + almacenamiento (R2) | Futuro |

**Curacion (D24):** la imagen se elige y aprueba automaticamente al llenar el banco (pictograma para verbos/adjetivos, foto para sustantivos), y la curacion es **posterior**: un listado por nivel en el admin con "cambiar imagen" (siguiente candidata) y "deshabilitar palabra". Asi el boton de llenado deja el banco usable de inmediato.

### 5.1 Llenado del banco desde el admin (D18)

En **Settings -> System -> English bank** (solo admin): una fila por nivel con su estado (`palabras con contenido / palabras del nivel en la lista`) y un boton **"Fill bank"**. Si el nivel ya esta lleno: dialogo "This bank is already full. Fill it again?".

Lo que hace el boton (pipeline por palabra):
1. Toma palabras del nivel desde la lista importada (D16) que aun no esten en el banco.
2. Busca y guarda la imagen candidata (ARASAAC/Pexels).
3. Genera el contenido del nivel con el LLM y lo valida en codigo (seccion 7.1).
4. La palabra queda `approved` si los 3 pasos salieron bien; si no, `pending_review` con el motivo.

Dos problemas que el diseño tiene que resolver:

- **"Volver a llenar" debe ser idempotente** (*idempotencia*: ejecutar la operacion dos veces deja el mismo resultado que una). Decidido (D25): rellenar solo lo que falta (palabras nuevas de la lista, imagenes o contenido faltantes o fallidos), **nunca** regenerar lo existente. Regenerar todo pagaria de nuevo el contenido, borraria la curacion manual y dejaria practicas guardadas apuntando a un contenido que ya no es el que vio el usuario.
- **Es una operacion larga**: 100 palabras x (API de imagen + LLM) son varios minutos. Un request HTTP de esa duracion es fragil, y en Cloud Run una tarea en segundo plano despues de responder puede quedarse sin CPU (por defecto la CPU solo se asigna mientras hay un request activo). Diseño: **procesamiento por lotes dirigido por el cliente**: el front llama `POST /system/english/levels/{level}/fill` repetidas veces, cada llamada procesa ~10 palabras y devuelve el progreso; la UI muestra una barra y se puede cortar y retomar (por la idempotencia). Sin colas ni jobs. Consecuencia: **no se usa la Batch API** en v1 (tarda horas; el ahorro son centavos).
- **Tamaño de cada llenado (D26)**: las listas tipo CEFR-J tienen del orden de cientos a miles de palabras por nivel. Cada "Fill bank" agrega hasta 100 palabras nuevas; el banco esta "lleno" cuando ya no quedan palabras del nivel en la lista.

⚠️ La licencia de ARASAAC es **CC BY-NC-SA, no comercial** (propiedad del Gobierno de Aragon). Exige mostrar el credito junto al pictograma: *"Pictograms author: Sergio Palao. Origin: ARASAAC (http://www.arasaac.org). License: CC (BY-NC-SA). Owner: Government of Aragon (Spain)"*. Hoy ULM es personal, pero si algun dia se cobra, hay que reemplazar esas imagenes. Las condiciones publicadas son las del sitio; no hay terminos separados para la API.

**Concepto: cache de resultados externos**: la API externa se consulta al poblar el banco, no en cada practica.

---

## 6. Modelo de datos

Prefijo `eng_`. **Banco compartido** (sin `user_id`; lo escribe el admin) y **datos del usuario** (con `user_id`, aislamiento + tests A/B) (D14).

**Banco compartido:**

| Tabla | Campos clave |
|---|---|
| `eng_words` | `id`, `lemma`, `pos`, `cefr_level`, `accepted_answers` (sinonimos, jsonb), `source_list`, `status` (`pending_review`/`approved`/`disabled`) |
| `eng_word_images` | `word_id`, `url`, `source`, `license`, `attribution`, `status` |
| `eng_word_contents` | `id`, `word_id`, `level`, `listening_sentence`, `writing_prompt`, `writing_prompt_es`, `reading_text`, `reading_question`, `model`, `prompt_version`, `status`. Unico por (`word_id`, `level`) |

**Por usuario:**

| Tabla | Campos clave |
|---|---|
| `eng_user_settings` | `user_id` (pk), `level`, `level_suggestion_dismissed_at` |
| `eng_practices` | `id`, `user_id`, `word_id`, `content_id`, `level` (snapshot), `vocab_revealed`, `vocab_used_synonym`, `listening_text_shown`, `writing_text`, `reading_transcript`, `reading_match_pct`, `free_transcript`, `speaking_skipped`, `audio_seconds`, `completed_at` |
| `eng_user_words` | `user_id`, `word_id` (pk compuesta), `mastered` (bool), `times_practiced`, `last_practiced_at`, `mastered_at` |
| `eng_practice_feedback` | `practice_id` (pk), `feedback` (jsonb estructurado), `model`, `input_tokens`, `output_tokens` |
| `eng_user_memory` | `user_id` (pk), `summary` (texto corto, ~500 car. max), `updated_at` |

**Transversal (fuera del prefijo, ver C6):** `ai_usage_events`.

**Seleccion de la siguiente palabra** (todo en SQL/codigo):
1. Candidatas = palabras `approved` del nivel del usuario, con contenido `approved` para ese nivel, **sin** las `mastered` del usuario.
2. Con ~30% de probabilidad, elegir una ya practicada y no dominada (la de `last_practiced_at` mas antigua primero); si no, una nueva al azar. Si un grupo esta vacio, usar el otro.
3. Nunca repetir la palabra de la practica anterior.
4. Sin candidatas (domino todo el nivel, D19): elegir entre las **dominadas**, la de `last_practiced_at` mas antigua primero, y mostrar la sugerencia de subir de nivel.

Notas:
- **No se guarda el audio**, solo la transcripcion (D15).
- `eng_user_memory` es la memoria sin contexto en el LLM: un resumen acotado que el propio feedback actualiza.
- `eng_user_words` se actualiza en la misma transaccion que se guarda la practica.

---

## 7. Uso del LLM y servicios pagados

Reutiliza `TextGenerator` (`app/integrations/openai_client.py`): stateless, `store=False`. Salida **estructurada** (JSON schema validado con Pydantic). Toda llamada registra su consumo en `ai_usage_events`.

### 7.1 Generacion del banco (admin, desde la UI)
- Parte del pipeline de 5.1 (`EnglishBankService`, invocado por el endpoint de llenado; un script de consola puede reutilizarlo). Por palabra pide `listening_sentence`, `writing_prompt` (+ `_es`), `reading_text` (empieza con la frase del listening, termina en pregunta) y `reading_question`.
- Validaciones en codigo: la frase contiene la palabra (o una forma flexionada), el texto empieza con la frase, termina en `?`, largos en rango. Si falla: un reintento y despues `pending_review`.
- Las ~10 palabras de cada lote van en **una sola llamada** al LLM para amortizar las instrucciones.
- Se registra en `ai_usage_events` con `feature=english.bank` a nombre del admin (no consume creditos de usuarios en el futuro control).

### 7.2 Feedback (on demand)
- Entrada: nivel, palabra, revelada o no, "show text", texto escrito, transcripcion de lectura + diff, transcripcion libre, `eng_user_memory.summary`.
- Salida: correcciones de writing (original -> corregido + por que), observaciones de lectura (del diff), correcciones del speaking libre, si respondio la pregunta, 1-3 consejos y **`memory_update`**. Una llamada = feedback + memoria.
- Idioma segun nivel (español en A1-A2). Modelo barato de `ai_models.py`, tope de `max_output_tokens`.

### 7.3 Transcripcion
- `POST /english/speech` (multipart: audio webm/opus, `content_id`, `phase=reading|free`). No necesita practica guardada.
- `whisper-1` (configurable, D17), `language="en"`, prompt solo con muletillas, nunca el texto esperado.
- En `phase=reading` el backend calcula el diff contra `reading_text` del `content_id`.
- Validar duracion y tamaño antes de llamar al proveedor.

---

## 8. API y frontend

**Permiso:** dominio `english`, sin `.ai` (D7). Router en `PRIVATE_ROUTERS` con `require("english")`; la administracion del banco con `ADMIN_ONLY`.

| Metodo | Ruta | Que hace |
|---|---|---|
| GET/PUT | `/english/settings` | Nivel; el GET incluye la sugerencia de nivel (si aplica) y los niveles con banco |
| POST | `/english/settings/level-suggestion/dismiss` | "No por ahora" |
| GET | `/english/practices/next` | Elige palabra y devuelve todo lo de la practica: imagen, pos, largo, pista, respuesta + sinonimos, contenido del nivel. **No guarda nada** |
| POST | `/english/speech` | Audio -> transcripcion (+ diff en lectura). Registra consumo |
| POST | `/english/practices` | Guarda la practica completa + actualiza `eng_user_words` (check por defecto) |
| PATCH | `/english/practices/{id}` | Cambia el check "la domino" |
| POST | `/english/practices/{id}/feedback` | Genera (o devuelve el ya guardado) |
| GET | `/system/english/levels` | Admin: estado del banco por nivel |
| POST | `/system/english/levels/{level}/fill` | Admin: procesa un lote del llenado y devuelve el progreso (5.1) |
| GET/PATCH | `/system/english/words` | Admin: listado por nivel, cambiar imagen, deshabilitar palabra (curacion posterior, D24) |

Decision consciente: como el front arma la practica y la envia al final, un usuario podria alterar lo que guarda (ej. una transcripcion falsa). En una app personal no afecta a nadie mas; no se agrega firma ni validacion cruzada.

**Frontend:** `/admin/english` (pagina de estudio: un componente por paso + `usePractice`), boton a `/admin/settings/english`. Instrucciones en un diccionario `en`/`es` segun el nivel. En el futuro, los botones que consumen tokens (grabar, feedback) llevaran la marca visual (D7): conviene que **ya** sean un componente comun (`AiActionButton`) aunque hoy no muestre nada, para no tocar cada pantalla despues.

**Concepto: maquina de estados**: los pasos y sus transiciones (`vocabulary -> listening -> writing -> speaking_reading -> speaking_free -> closing`) viven en `usePractice`; la UI solo pinta el estado actual, y testearlo con Vitest no requiere montar componentes.

---

## 9. Costos estimados

| Concepto | Frecuencia | Estimado |
|---|---|---|
| Banco: ~100-500 palabras x 1 nivel | Una vez por nivel | Centavos con modelo mini (+ Batch) |
| Imagenes / TTS | — | 0 |
| Transcripcion, ~1.5 min/practica | Cada practica | ~0.005-0.01 USD |
| Feedback, ~2k in / 600 out | Solo si se pide | < 0.005 USD |

Orden de magnitud: **~1 centavo por practica con feedback**.

---

## 10. Fases

| Fase | Alcance | Servicios pagados |
|---|---|---|
| **0. Preparacion** | ✅ Hecha: commit de lo pendiente, preguntas resueltas, licencias verificadas (D27) | No |
| **1. Banco** ✅ (`d11190d` core / `0099e75` web) | Tablas del banco, importar la lista CEFR-J + Octanove (con `cefr_level` y `pos` de la lista; `source_list` guarda el origen), clientes ARASAAC/Pexels, `EnglishBankService` (pipeline idempotente por lotes), panel admin "English bank" con progreso y listado de curacion. Meta: banco B1 con ~100 palabras | Generacion (admin) |
| **2. Backend de practica** ✅ (`a766fb1` core) | Permiso `english`, settings, `GET /practices/next` con la seleccion de la seccion 6, `POST/PATCH /practices`, `ai_usage_events`. Tests de aislamiento y permisos | No |
| **3. UI pasos 1-3 + cierre** ✅ (`4c30cd9` web) | Pagina de estudio, vocabulario, listening, writing, cierre con check, settings de nivel | No |
| **4. Speaking** ✅ (`8f4e7cb` core / `51ced3b` web) | `MediaRecorder`, `SpeechTranscriber` + `/english/speech`, diff, saltar si no hay microfono | Transcripcion |
| **5. Feedback + memoria** ✅ (`47613ef` core / `897c79c` web) | Endpoint de feedback, `eng_user_memory`, vista del feedback guardado | LLM on demand |
| **6. Sugerencia de nivel** ✅ (`99711af` core / `f12db2c` web) | Reglas de 4.1 + banner | No |
| **Futuro** | Ver seccion 14 | — |

---

## 11. Preguntas abiertas

Ninguna. N1-N10 resueltas (D18-D26); licencias verificadas (D27).

---

## 12. Decisiones

| # | Decision | Fecha |
|---|---|---|
| D1 | Practicas sin limite por ahora; pasos fijos vocabulario -> listening -> writing -> speaking -> cierre | 2026-10-17 |
| D2 | Imagenes **no** las busca el LLM: APIs de imagenes/pictogramas + cache + curacion (seccion 5) | 2026-10-17 |
| D3 | Mitigaciones de ambiguedad de C3 aceptadas | 2026-10-17 |
| D4 | Check "la domino" al cierre: por defecto si si acerto solo, no si revelo; dominadas no vuelven; practicadas no dominadas se mezclan con nuevas. Reemplaza a la repeticion espaciada en v1 | 2026-10-17 |
| D5 | Nivel manual + sugerencia por reglas (4.1) | 2026-10-17 |
| D6 | Sin topes de consumo en este modulo; el control sera transversal y posterior. Si se registra el consumo (`ai_usage_events`) desde ya | 2026-10-17 |
| D7 | Vision futura de tokens: saldo por usuario, botones que consumen marcados y deshabilitados sin saldo, en todo ULM. Por eso **no** hay permiso `english.ai`: solo `english` | 2026-10-17 |
| D8 | Nada de la practica se guarda hasta el cierre; recargar = empezar de cero | 2026-10-17 |
| D9 | Sin tiempo limite ni registro de intentos en vocabulario | 2026-10-17 |
| D10 | Validacion de la palabra solo en el front (respuesta y sinonimos viajan con la practica) | 2026-10-17 |
| D11 | TTS del navegador en v1 | 2026-10-17 |
| D12 | "Finish for today" solo sale; resumen e integraciones a futuro | 2026-10-17 |
| D13 | Contenido generado solo para el nivel del usuario, sin niveles vecinos | 2026-10-17 |
| D14 | Banco compartido; por usuario solo los datos de sus practicas | 2026-10-17 |
| D15 | No se guarda el audio | 2026-10-17 |
| D16 | Lista de palabras libre tipo CEFR-J (ver D27); el LLM solo rellena huecos | 2026-10-17 |
| D17 | Transcripcion v1: OpenAI `whisper-1` (modelo en config), sin spike previo | 2026-10-17 |
| D18 | El banco de cada nivel lo llena el admin desde Settings -> System con un boton; si ya esta lleno, dialogo de confirmacion y se ejecuta el mismo mecanismo. Niveles sin banco: deshabilitados para el usuario | 2026-10-17 |
| D19 | Al dominar todas las palabras del nivel, se repiten las dominadas y se recomienda subir de nivel | 2026-10-17 |
| D20 | Deshacer un "la domino": fuera de v1 | 2026-10-17 |
| D21 | La practica se guarda al llegar a la pantalla de cierre; el check se actualiza aparte | 2026-10-17 |
| D22 | LanguageTool: trabajo futuro | 2026-10-17 |
| D23 | El saldo futuro se expresa en **creditos** derivados del costo estimado, no en tokens | 2026-10-17 |
| D24 | Imagen aprobada automaticamente al llenar el banco; curacion posterior (cambiar imagen / deshabilitar palabra) | 2026-10-17 |
| D25 | "Volver a llenar" es idempotente: solo completa lo faltante, nunca regenera lo existente | 2026-10-17 |
| D26 | Cada "Fill bank" agrega hasta 100 palabras; "lleno" = no quedan palabras del nivel en la lista | 2026-10-17 |
| D27 | Lista de palabras: **CEFR-J Vocabulary Profile 1.5** (A1-B2; uso de investigacion y comercial sin costo, citando la fuente; copyright Tono Laboratory, TUFS) via el repo `openlanguageprofiles/olp-en-cefrj`. **C1/C2: Octanove C1/C2 list** (CC BY-SA 4.0, mismo repo). La cita de ambas va en la pantalla de creditos/acerca de. Antes de importar, confirmar contra la pagina oficial de CEFR-J (la verificacion se hizo con el README del repo, fuente secundaria) | 2026-10-17 |
| D28 | Solo se importan sustantivos, verbos y adjetivos (en minusculas, sin abreviaturas); las palabras sin imagen de ninguna fuente quedan en revision | 2026-10-17 |
| D29 | Una entrada por palabra: si aparece con varias categorias se elige sustantivo > verbo > adjetivo; el nivel es el menor en que aparece | 2026-10-17 |
| D30 | Los sinonimos aceptados los genera el LLM en la misma llamada del contenido (incluye variantes britanica/americana); solo se aceptan los del mismo largo y patron que la palabra (las casillas tienen largo fijo) | 2026-10-17 |
| D31 | Modelo del banco y del feedback: el mas barato del catalogo con salida estructurada (`ENGLISH_MODEL`, por defecto `gpt-6-luna`), con su propio tope de tokens | 2026-10-17 |
| D32 | Openverse solo como respaldo cuando ninguna fuente principal tiene imagen (su cupo sin key es 20/min y 200/dia); Pexels y ARASAAC siguen completando hasta 5 candidatas | 2026-10-17 |
| D33 | El writing tiene una fase 1 de dictado (escribir la frase escuchada) antes de la respuesta; comparacion por codigo, sin LLM | 2026-10-17 |
| D34 | Pista del vocabulario detras de un boton, generada con el contenido (PROMPT_VERSION 2). Se valida en codigo que no contenga la palabra, sus sinonimos ni su traduccion al español (el LLM devuelve la traduccion solo para eso, no se guarda). Usarla no cambia el check. Las 100 palabras A2 de dev se completaron con un script de una sola vez (97 por LLM, 3 a mano) | 2026-10-17 |

---

## 13. ¿Listos para empezar el desarrollo?

**Si. Se puede empezar la fase 1 (banco).**

- Fase 0 completa: repos commiteados (2026-10-17, "blog features"), preguntas N1-N10 resueltas (D18-D26), licencias verificadas (D27).
- Flujo, adaptacion por nivel, modelo de datos, API y fases definidos; requisitos no funcionales cubiertos.

Unico pendiente menor, dentro de la fase 1: confirmar la licencia de CEFR-J en la fuente oficial al descargar la lista (D27).

Riesgos aceptados:
- Sin spike, `whisper-1` podria normalizar mas errores de los esperados; se cambia por config (D17).
- Con el check "la domino" por defecto tras un acierto y sin deshacer, el usuario repetira dominadas pronto (D19) y dependera de que el admin llene mas banco o de subir de nivel.
- Con aprobacion automatica (D24), un usuario puede ver una imagen mala antes de que el admin la corrija.
- La calidad del TTS del navegador varia por dispositivo.
- ARASAAC y Octanove son *share-alike*/no comercial: si ULM se vuelve comercial, hay que revisar ambas (seccion 14).

---

## 14. Trabajo futuro

Todo lo que se dejo fuera de v1, para no perderlo. Origen = decision o critica donde se discutio.

| Tema | Que | Origen |
|---|---|---|
| **Control de creditos transversal** | Saldo por usuario en creditos (derivados del costo), botones que consumen marcados (`AiActionButton`) y deshabilitados sin saldo, en todo ULM. Base: `ai_usage_events` | D6, D7, D23 |
| Migrar `/ai/ask` | Registrar su consumo en `ai_usage_events` (hoy solo log) | C6 |
| Pronunciacion | Azure Pronunciation Assessment para la fase de lectura (puntaje por palabra/fonema) | C1 |
| Transcripcion propia | faster-whisper autoalojado (local o si el costo lo justifica); explorar la probabilidad por palabra como señal de pronunciacion | C1, C7 |
| Comparar modelos de transcripcion | El spike descartado: `whisper-1` vs `gpt-4o-mini-transcribe` con audios con errores a proposito | D17 |
| Writing | LanguageTool como pre-revision gratuita antes del LLM | C7, D22 |
| Modelos clasicos | Regresion logistica para sugerir nivel; regresion para la dificultad real de cada palabra. Requiere volumen de datos | C7 |
| Deshacer "la domino" | Lista de dominadas en Settings con opcion de reactivar | C4, D20 |
| Repeticion espaciada | SM-2/FSRS en lugar del check binario, si el check resulta insuficiente | C4 |
| Cierre | Resumen del dia en "Finish for today"; integracion con checklists/habitos (meta diaria) | D12 |
| TTS pagado | Si la voz del navegador no alcanza: TTS de OpenAI + cache del audio en R2 | D11 |
| Guardar audio | Para re-escucharse o evaluar pronunciacion despues (object storage + privacidad) | D15 |
| Imagenes generadas | Modelo de imagen para palabras sin imagen clara (estilo uniforme) | Seccion 5 |
| Licencia de imagenes | Reemplazar ARASAAC (no comercial) si ULM llega a cobrar | Seccion 5 |
| Regenerar banco | Accion explicita "regenerar contenido" (con versionado, sin romper practicas guardadas), distinta del llenado idempotente | D25 |
| Batch API | Para llenados grandes del banco (~50% de descuento), con jobs asincronos | 5.1 |
| Retomar practica | Guardar estado por paso para retomar tras recargar | D8 |
| Topes por usuario | Limites diarios de audio/feedback (quedaria cubierto por el control de creditos) | C6 |
| Probar desde el celular | Tunel HTTPS (`cloudflared`) para usar el microfono en dev desde la red local | C8 |

---

## 15. Estado de la implementacion (2026-10-17)

Las 6 fases estan hechas y commiteadas (sin push). Tests: `ulm-core` 551 pasan; `ulm-web` 328 pasan, typecheck, lint y build OK. Migraciones `d5fe720bae38`, `a61495ef24e9` y `a48ff2159e3e` aplicadas en la base de dev (cada una probada con upgrade/downgrade/upgrade y `alembic check` sin diferencias).

**Donde quedo cada cosa:**

| Pieza | Backend (`ulm-core`) | Frontend (`ulm-web`) |
|---|---|---|
| Banco | `english_wordlist.py`, `english_content.py`, `english_bank_service.py`, `integrations/image_search.py`, rutas `/system/english/*` | Settings -> System -> English bank (`EnglishBankPanel.vue`) |
| Practica | `english_practice_service.py`, rutas `/english/settings` y `/english/practices*` | `/admin/english` (`EnglishPage.vue`, `usePractice`, `components/english/*`), Settings -> English (`EnglishLevelPanel.vue`) |
| Speaking | `integrations/speech_client.py`, `english_reading_diff.py`, `english_speech_service.py`, `POST /english/speech` | `lib/recorder.ts`, `SpeakingStep.vue` |
| Feedback | `english_feedback.py`, `english_feedback_service.py`, `POST /english/practices/{id}/feedback` | `FeedbackPanel.vue` |
| Nivel | `english_level_suggestion.py`, `POST /english/settings/level-suggestion/dismiss` | banner en `EnglishPage.vue` |
| Consumo | `ai_usage_events` + `AiUsageRecorder` (bank, transcription, feedback) | — |

**Probado contra los servicios reales** (centavos): generacion del banco con `gpt-6-luna` (3 palabras A2, ~570 tokens, las 3 validas), ARASAAC (pictogramas correctos para apple/run/happy), `whisper-1` (llamada y duracion OK), feedback (JSON valido en modo estricto, ~1.000 tokens, en español para A2, sin opinar de pronunciacion).

**No verificado todavia:** el flujo completo en el navegador (login con Google, microfono real, voz del navegador). Es lo siguiente:
1. `alembic upgrade head` (ya hecho en dev) y levantar backend + front.
2. Settings -> System -> English bank -> "Fill bank" en el nivel propio (~100 palabras, unos minutos, centavos).
3. Revisar el listado de palabras: cambiar imagenes malas y deshabilitar abstractas (ver abajo).
4. Settings -> English -> elegir nivel; sidebar -> English -> hacer una practica completa con microfono y pedir feedback.
5. Asignar el permiso `english` en Settings -> Users a quien corresponda (el admin ya lo tiene).

**Hallazgos durante la implementacion:**
- La lista filtrada (D28) igual trae palabras dificiles de dibujar (*remember*, *pardon*, *let*). La curacion posterior (D24) es la que las saca; vale la pena revisarlas en el primer llenado.
- Sin `PEXELS_API_KEY` los sustantivos tambien salen de ARASAAC (pictogramas): funciona, pero las fotos serian mejores para objetos. Agregar la key es opcional.
- `whisper-1` transcribio un tono puro como "oh": confirma C1, siempre "adivina" palabras. El diff y el feedback lo asumen.
- La practica no guarda estado intermedio (D8): si una transcripcion se paga y el usuario recarga, el consumo queda en `ai_usage_events` aunque la practica no exista.
- El vitest de `ulm-web` reporta 3 errores no capturados en `tests/permissions.spec.ts` (mock de comidas): **existian antes de este modulo**, no los toque.
- Los `CLAUDE.md` de `ulm-core`/`ulm-web` no se actualizaron con el modulo (son resumenes concisos por diseño); si se quiere, agregar una linea por repo.

