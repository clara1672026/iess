# Banco de Casos Radiológicos — IESS San Francisco

Aplicación de una sola página (`index.html`) que ahora usa **Cloud Firestore** como base
de datos principal y compartida de verdad entre todos los equipos que abran la web.

## 🔧 Corrección (ronda actual): hora de ingreso automática

Se agregó un nuevo dato al caso, **`horaIngreso`**, con la misma filosofía que la fecha
automática (ver ronda anterior, justo abajo): sin selector, sin campo editable, sin que el
usuario escriba nada.

- Al guardar un caso **nuevo** (Administrador, Posgradista o Tratante, sin diferencia entre
  perfiles), la aplicación toma automáticamente la hora local exacta del dispositivo en el
  instante de pulsar "Guardar caso" (`nowHM()`, con `getHours()`/`getMinutes()` — hora LOCAL,
  nunca UTC) y la guarda en formato 24 horas `HH:mm` (ej. `20:47`).
- El formulario **no muestra ningún selector de hora ni campo editable**: solo, junto a la
  fecha, un texto de solo lectura con la hora actual (referencial, mientras el formulario está
  abierto); la hora que realmente queda guardada es la que exista en el instante exacto de
  guardar, no la que se veía al abrir el formulario.
- Una vez creado el caso, la hora de ingreso **nunca vuelve a recalcularse ni a sobrescribirse**:
  al editar un caso existente (cambiar HCL, estudio, diagnóstico, fecha histórica, etc.) el
  campo `horaIngreso` se conserva tal cual quedó al crearlo — no hay ningún punto del código que
  lo reescriba fuera de la creación inicial. Sigue igual después de recargar la página, cerrar
  sesión, volver a iniciar sesión o releer el caso desde Firestore, porque se guarda como un
  campo más del documento del caso (`set()` completo, sin recalcular nada al leer).
- En la Biblioteca de casos, la columna "Fecha de ingreso" ahora muestra fecha y hora juntas de
  forma compacta: `16/09/2026 · 20:47`. Un caso antiguo que no tenga hora guardada (de antes de
  este cambio) simplemente muestra solo la fecha, sin romper el formato.

## 🔧 Corrección (ronda anterior): fecha automática al agregar un caso y titileo real de "NUEVO"

Esta ronda corrige únicamente dos puntos, en los tres perfiles (Administrador, Posgradista y
Médico Tratante); no se tocó ninguna otra función de la aplicación.

**1) Fecha automática al agregar un caso — sin calendario ni selector manual.**
- El formulario **"Agregar caso"** ya NO muestra ningún calendario, selector de fecha ni campo
  editable de fecha de ingreso. En su lugar se ve, como texto de solo lectura, la fecha local de
  hoy (ej. `16/09/2026`), calculada automáticamente en el momento de abrir el formulario.
- Al guardar, la fecha de ingreso del caso nuevo se toma **siempre** de la fecha local del
  dispositivo en ese instante (`todayISO()`), sin depender de ningún valor de formulario — el
  usuario no elige ni puede elegir una fecha distinta.
- Esto aplica exactamente igual para Administrador, Posgradista y Médico Tratante: los tres usan
  el mismo formulario y la misma lógica de guardado.
- El formulario de **editar un caso ya existente** sigue teniendo su propio campo de fecha
  editable (sin cambios), porque ahí sí puede ser necesario corregir manualmente un dato
  histórico — esa función no forma parte de esta corrección y se dejó intacta.
- Se revisó todo el ciclo de vida de la fecha (guardar → recargar la página → cerrar sesión →
  volver a iniciar sesión → leer de nuevo desde Firestore) y no hay ningún punto que recalcule o
  reinterprete la fecha histórica ya guardada de un caso: `parseISO()` interpreta el texto
  `"YYYY-MM-DD"` siempre como fecha LOCAL (nunca pasa por UTC) y `fmtDate()` la muestra
  formateando directamente ese mismo texto, sin crear ningún objeto `Date` intermedio — así se
  elimina definitivamente el error de "un día menos" en cualquier paso (guardar, recargar,
  reabrir sesión o recuperar desde Firestore).

**2) Rótulo "NUEVO" con titileo real.**
- Un caso muestra el rótulo "NUEVO" el día de su publicación y el día siguiente (2 días en
  total); desde el tercer día desaparece automáticamente. Esta regla ya funcionaba correctamente
  y se mantiene sin cambios (`esCasoReciente()`).
- Lo que sí se corrigió es que el rótulo ahora **titila de verdad**, de forma visible, suave y
  continua (ciclo de 1 segundo, dentro del rango pedido de 0.8–1.2s), alternando opacidad
  (1 → 0.35 → 1) y una leve escala, vía `@keyframes parpadeoNuevo`.
- Como respaldo — para que el titileo esté garantizado y no dependa únicamente de que el CSS se
  aplique tal cual en cada navegador — se agregó además un pequeño bucle en JavaScript
  (`requestAnimationFrame`) que aplica exactamente la misma oscilación de opacidad/escala en
  cada fotograma sobre cualquier rótulo "NUEVO" presente en la página en ese momento. Así, el
  titileo sigue funcionando aunque la tabla se vuelva a dibujar (por ejemplo al guardar un caso o
  al llegar una actualización en tiempo real de otro equipo), y sigue funcionando igual después
  de recargar la página. Se probó en Chromium (motor de Chrome y de Edge) verificando que la
  opacidad calculada del rótulo cambia de forma continua en el tiempo. Ambos mecanismos respetan
  `prefers-reduced-motion` (si el sistema operativo pide menos movimiento, el rótulo no titila).

## 🔧 Corrección (ronda anterior): permiso real de Tratante y desfase de fecha

Esta ronda corrige dos fallas reales que las rondas anteriores no habían resuelto del todo
(la interfaz ya era visualmente idéntica, pero faltaba el permiso; y la fecha se calculaba
bien pero se **mostraba** mal):

- **Tratante ahora puede gestionar el catálogo de verdad.** La función `puedeGestionarCatalogo()`
  solo autorizaba a `ADMINISTRADOR` y `POSGRADISTA`; el rol `MÉDICO TRATANTE` quedaba fuera,
  así que aunque la pantalla era la misma, Tratante seguía viendo la versión de solo lectura
  (sin "+ Agregar estudio", sin Editar, sin Eliminar). Se agregó `MÉDICO TRATANTE` a esa misma
  función — sigue existiendo un único componente de catálogo para los tres roles.
- **Tratante ya puede crear estudios nuevos**, con la misma normalización que el resto:
  MAYÚSCULAS, sin espacios dobles ni al inicio/final, y verificación de duplicados antes de
  guardar (mensaje exacto: *"Este tipo de estudio ya existe en el catálogo."*, ahora mostrado
  también al usar el campo "+ Agregar otro estudio" del asistente).
- **Eliminación de estudios usados sigue bloqueada** para todos los roles autorizados por
  igual (ya funcionaba; se confirmó que el permiso ampliado a Tratante no debilita esta regla).
- **Corregido el desfase de un día en las fechas.** La causa real: `fmtDate()` volvía a crear
  un `Date` a partir del texto `"YYYY-MM-DD"` guardado, y `new Date("YYYY-MM-DD")` lo interpreta
  como medianoche **UTC**; al leerlo de nuevo con los métodos de hora local (`getDate()`, etc.),
  el día retrocedía uno en zonas con offset negativo como Ecuador (UTC-5). La fecha que se
  guardaba en Firestore siempre fue la correcta — el error estaba solo en cómo se volvía a leer
  para mostrarla. Se añadió `parseISO()`, que interpreta `"YYYY-MM-DD"` como fecha local sin
  pasar nunca por UTC, y `fmtDate()` ahora formatea directamente el texto sin crear ningún
  objeto `Date`. Se revisó todo el ciclo (automática al abrir el formulario → guardar →
  Firestore → recuperar → mostrar en la lista) y usa la misma lógica en todos los puntos.
- **(Superado por la ronda actual, arriba)** En esta ronda anterior la fecha de ingreso se había
  hecho editable también al *agregar* un caso nuevo. La ronda actual (ver arriba) volvió a quitar
  ese selector manual solo para "Agregar caso": un caso nuevo siempre usa la fecha local de hoy en
  automático. El campo de fecha editable se conservó únicamente en "Editar caso", para corregir
  datos históricos.
- **Casos recientes resaltados.** Un caso con fecha de ingreso de hoy o de ayer recibe un fondo
  sutil y una etiqueta discreta "NUEVO" en la Biblioteca de casos. El orden por defecto (más
  reciente primero) ya existía y se mantiene, ahora usando la misma fecha local corregida.
- **Nuevo filtro por rango de fechas** ("Desde" / "Hasta") en la Biblioteca de casos, combinable
  con el buscador de texto y los demás filtros existentes (médico, estado, área), con un botón
  para limpiarlo. No se quitó el buscador existente.

## 🔧 Corrección: protección de estudios en uso, usuarios eliminados y pie de página

**Interfaz de Tratante confirmada de nuevo:** se volvió a verificar (comparando el HTML real
generado, carácter por carácter) que Tratante usa exactamente la misma pantalla de catálogo
que Posgradista y Administrador. La única diferencia sigue siendo la presencia de los botones
de gestión, nunca el diseño.

**Un tipo de estudio ya utilizado no puede eliminarse.** Antes de borrar un estudio del
catálogo, la app comprueba si existe algún caso guardado con ese estudio. Si lo hay, el botón
Eliminar ni siquiera aparece junto a ese estudio (se ve la etiqueta "En uso" en su lugar), y si
de todos modos se intentara forzar la acción, la función de eliminación la rechaza y muestra:
*"Este tipo de estudio no puede eliminarse porque ya está siendo utilizado en casos
registrados."* — el estudio y los casos que lo usan quedan intactos. Solo puede eliminarse un
estudio que nunca se haya usado en ningún caso.

**Eliminar un usuario nunca elimina sus casos.** La eliminación de una cuenta solo borra el
documento de esa cuenta en la colección `usuarios` — nunca toca la colección `casos`. Cada
caso guarda el nombre de quien lo aportó (`aportadoPor`) como un dato propio del caso, no como
una referencia en vivo a la cuenta del usuario; por eso, aunque se elimine la cuenta, el
nombre histórico del propietario sigue mostrándose con total normalidad. Si la cuenta ya no
existe, la app agrega junto al nombre la anotación **"(USUARIO ELIMINADO)"**, sin borrar ni
alterar el nombre original. El Administrador conserva la posibilidad de reasignar
posteriormente el propietario de un caso a otra cuenta activa, desde el formulario de edición
del caso.

**Corrección de duplicados existentes sin perder casos.** El proceso que limpia estudios
duplicados (ver sección de abajo) primero revisa qué casos usan cada nombre duplicado y los
reasigna al registro principal normalizado — solo después de eso elimina los documentos
sobrantes del catálogo. Ningún caso puede quedar sin tipo de estudio ni con una referencia
rota como resultado de esta limpieza.

**Pie de página actualizado.** El texto ahora es exactamente:

> © 2026 Pg Imagen Lara PUCE. Todos los derechos reservados. Uso exclusivamente académico
> para Team Imagen IESS San Francisco.

Visible de forma consistente tanto en la pantalla de ingreso como en la aplicación principal.

## 🔧 Corrección: usuarios que desaparecían

**Causa real:** la carga inicial de datos (usuarios, catálogo, categorías, áreas) se resolvía
con el primer aviso de `onSnapshot()`. Ese primer aviso muchas veces llega de la **caché local**
del navegador antes de que el servidor responda. Si la caché todavía no tenía sincronizado el
documento de usuarios (otro dispositivo, caché recién borrada, reconexión de red), la app creía
que "no había usuarios todavía" y volvía a crear solo el administrador — **borrando a todos los
demás**, y esa sobrescritura se sincronizaba al instante a todo el mundo conectado.

**Corrección:** la carga inicial ahora usa `get({source:"server"})`, una lectura explícita que
espera la respuesta real del servidor (con `cache` solo como respaldo si no hay red en absoluto).
`onSnapshot()` se sigue usando, pero únicamente para recibir cambios *después* de esa primera
lectura confiable — nunca para decidir si hay que sembrar datos. Además, `usuarios` pasó a ser
una colección con un documento por persona (igual que `casos` y `catalogo_estudios`), en vez de
un único documento con la lista completa, para que crear o editar a alguien nunca implique
reescribir a todos los demás.

## 🔧 Corrección: Tratante tenía una interfaz distinta a Posgradista en el catálogo

**Causa real:** aunque el catálogo de estudios ya usaba una sola función de pantalla y una sola
colección de Firestore para todos los roles, esa función dibujaba internamente **dos diseños
distintos** según el permiso: quien podía gestionar el catálogo (Administrador y Posgradista)
veía una lista en filas con botones de Editar/Eliminar; quien no podía gestionarlo (Médico
Tratante) veía en cambio una nube de "chips" redondeados de solo lectura — un diseño
completamente diferente, no solo sin botones.

**Corrección aplicada:**
- Se eliminó por completo el diseño alternativo de "chips". Ahora existe un único diseño en
  filas (el que ya usaba Posgradista) y **todos los roles lo usan exactamente igual**.
- **Tratante reutiliza exactamente la misma interfaz, estructura HTML y estilos que
  Posgradista.** La única diferencia que queda entre ambos es la presencia de los botones
  "+ Agregar estudio", "Editar" y "Eliminar" — que dependen únicamente del permiso
  (`puedeGestionarCatalogo()`), nunca del diseño.
- Verifiqué esto generando el HTML real de la pantalla para Posgradista y para Tratante y
  comparándolo carácter por carácter (quitando solo los controles de gestión): son idénticos.
- Ambos perfiles siguen leyendo de la misma y única colección `catalogo_estudios` en
  Firestore — no existen catálogos ni consultas separadas por rol.
- Se reconfirmó la prevención de duplicados y la normalización a MAYÚSCULAS (ver sección
  siguiente) sobre este mismo catálogo unificado.

## 🔧 Corrección: tipos de estudio duplicados (ej. "UROTAC SIMPLE" repetido)

**Causa real:** existían dos huecos. (1) Antes de la corrección anterior, la carga inicial del
catálogo podía interpretar una caché local desactualizada como "catálogo vacío" y volver a
sembrarlo — cada siembra usaba ids aleatorios, así que sembrar dos veces creaba copias
duplicadas del mismo estudio con distinto id. (2) Dos botones de "crear estudio" (el del
formulario de casos y el asistente de agregar estudios) decidían si un estudio ya existía
únicamente al **dibujar** la lista, sin volver a comprobarlo en el momento exacto de guardar —
si alguien más creaba ese mismo estudio mientras la lista seguía abierta, se podía crear un
duplicado.

**Corrección aplicada:**
- **Limpieza automática** (`limpiarDuplicadosCatalogo()`, se ejecuta en cada carga de la app):
  agrupa los estudios por nombre normalizado, conserva un solo registro por estudio real (el
  de más uso, para no perder historial), suma el uso de todos los duplicados en ese registro,
  **re-vincula automáticamente los casos** que usaban cualquiera de las variantes duplicadas
  hacia el nombre ya normalizado, y solo entonces elimina los documentos sobrantes de
  Firestore. Es segura de ejecutar siempre: si el catálogo ya está limpio, no escribe nada.
- **Normalización estricta al guardar**: todo nombre de estudio se convierte a MAYÚSCULAS,
  sin espacios dobles ni espacios al inicio/final, antes de compararlo o guardarlo — así
  `"Urotac simple"`, `"UROTAC SIMPLE"`, `" UROTAC SIMPLE "` y `"UROTAC  SIMPLE"` se reconocen
  siempre como el mismo estudio único: `UROTAC SIMPLE`.
- **Revalidación en el momento exacto de guardar** (no solo al mostrar la lista) en los dos
  puntos donde se crean estudios nuevos, cerrando la ventana de tiempo en la que dos personas
  podían crear el mismo estudio a la vez.
- **Siembra inicial con ids deterministas**: si el catálogo está realmente vacío y dos equipos
  abren la app al mismo tiempo, ambos escriben en los mismos documentos (en vez de ids al azar
  que crearían duplicados).
- **Una sola colección, una sola interfaz**: Administrador, Posgradista y Tratante siempre
  leyeron y siguen leyendo del mismo `catalogo_estudios` en Firestore, a través del mismo
  componente de pantalla — la interfaz de referencia es la del Posgradista; lo único que
  cambia según el rol son los botones de crear/editar/eliminar que se muestran, nunca la
  lista en sí ni la fuente de datos.

## ✅ Qué cambió con esta integración

- **Antes**: los datos se guardaban en `localStorage` del navegador — cada persona veía
  solo lo suyo.
- **Ahora**: todo (casos, catálogo de estudios, categorías, áreas, usuarios) se guarda en
  **Cloud Firestore**. Si el Dr. Carlos Lara registra un caso, la Dra. María Gómez lo ve
  aparecer en su pantalla **sin recargar la página**, y viceversa.
- `localStorage` solo se sigue usando para UNA cosa puramente local: recordar qué usuario
  quedó seleccionado en ESE navegador específico (para no pedir el usuario cada vez que
  se abre la app en el mismo equipo). Ningún dato de casos, catálogo ni usuarios pasa por
  `localStorage`.
- Cada caso y cada estudio del catálogo es su **propio documento** en Firestore (no un solo
  archivo gigante con todo adentro). Así, si dos personas guardan cosas distintas al mismo
  tiempo, no se pisan entre sí.
- El número de caso (`#000123`) se genera con una **transacción atómica** de Firestore, para
  que dos equipos guardando un caso nuevo en el mismo segundo nunca reciban el mismo número.
- Firestore mantiene su propia caché local (IndexedDB) automáticamente — sirve como respaldo
  temporal si hay un corte de red momentáneo, pero la nube sigue siendo la única fuente real.

## ⚠️ PASO OBLIGATORIO ANTES DE USARLA: configurar Firestore

Tu proyecto de Firebase fue creado en **modo producción**, lo que significa que **por
defecto bloquea toda lectura y escritura**. Si no haces esto, la app cargará pero no podrá
guardar ni leer nada, y verás errores de "permission-denied" en la consola del navegador.

### 1. Activar Autenticación Anónima

La app usa autenticación anónima de Firebase (no es el mismo selector de usuario de la
app — eso sigue igual; esto solo identifica al *navegador* ante Firebase para que las
reglas de seguridad puedan exigir "debe estar autenticado").

1. Ve a [console.firebase.google.com](https://console.firebase.google.com) y abre tu
   proyecto **banco-de-casos**.
2. En el menú lateral, ve a **Authentication** (Compilación → Authentication).
3. Haz clic en **"Comenzar"** si es la primera vez.
4. En la pestaña **"Sign-in method"**, busca **"Anónimo"** y actívalo (toggle a "Habilitado").
5. Guarda.

### 2. Configurar las reglas de seguridad de Firestore

1. En el menú lateral, ve a **Firestore Database**.
2. Ve a la pestaña **"Reglas"**.
3. Reemplaza todo el contenido por lo que está en el archivo `firestore.rules` (incluido
   en este proyecto), que es exactamente esto:

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /{document=**} {
         allow read, write: if request.auth != null;
       }
     }
   }
   ```

4. Haz clic en **"Publicar"**.

**¿Por qué estas reglas y no reglas totalmente abiertas?** Con `if request.auth != null`,
solo quien pase por la autenticación anónima (que la propia app hace automáticamente al
cargar) puede leer o escribir. No es una seguridad perfecta — cualquiera que copie la
configuración pública de tu app técnicamente podría autenticarse también de forma anónima
— pero evita que bots o rastreadores automáticos que recorren Firestore al azar (buscando
bases de datos abiertas con `allow read, write: if true`) puedan tocar tus datos sin
esfuerzo. Para este uso académico interno es un balance razonable entre seguridad y
simplicidad (no se pidió un sistema de login con contraseñas).

Si en algún momento quieres reforzarlo más (por ejemplo, restringir por dominio o agregar
usuarios reales con contraseña en Firebase Auth en vez de autenticación anónima), dímelo y
ajustamos las reglas.

### 3. Verificar las colecciones (se crean solas)

No necesitas crear nada a mano: la primera vez que la app cargue, ella misma creará estas
colecciones en Firestore al guardar información por primera vez:

| Colección            | Qué contiene                                                      |
|-----------------------|--------------------------------------------------------------------|
| `casos`               | Un documento por cada caso registrado                             |
| `catalogo_estudios`   | Un documento por cada tipo de estudio del catálogo                |
| `usuarios`            | Un documento por cada persona registrada (posgradistas, tratantes, administrador) |
| `configuracion`       | Documentos: `categorias`, `areas`, `contador_casos`               |

## Pasos para publicar en GitHub Pages

1. Ve a [github.com](https://github.com) e inicia sesión (o crea una cuenta gratis).
2. Haz clic en **"New repository"**. Ponle un nombre (ej. `banco-de-casos`), déjalo en
   **Public**, y haz clic en **"Create repository"**.
3. Haz clic en **"uploading an existing file"** (o "Add file → Upload files").
4. Arrastra **todos** los archivos de este proyecto (`index.html` y este `README.md`;
   `firestore.rules` no hace falta subirlo a GitHub, solo lo usas para copiar/pegar en la
   consola de Firebase).
5. Haz clic en **"Commit changes"**.
6. Ve a **Settings → Pages**. En "Branch" elige `main` y `/ (root)`. Guarda.
7. Espera uno o dos minutos y abre el enlace que te da GitHub
   (`https://tu-usuario.github.io/banco-de-casos/`).

## Cuota gratuita de Firestore

El plan gratuito ("Spark") de Firebase incluye, por día: 50,000 lecturas, 20,000 escrituras
y 1 GB de almacenamiento. Para un banco de casos de un solo servicio hospitalario esto es
más que suficiente incluso con varios equipos usándolo activamente. Si algún día lo supera,
Firebase simplemente te lo notifica — no cobra automáticamente a menos que actives el plan
"Blaze" (pago por uso) tú mismo.

## Primer uso

Al abrir la app por primera vez, ella misma crea en Firestore un usuario **ADMINISTRADOR
DEL SISTEMA**. Entra con ese usuario y ve a la pestaña **"Usuarios"** para registrar a los
posgradistas y médicos tratantes reales del equipo.
