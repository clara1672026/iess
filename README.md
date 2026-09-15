# Banco de Casos Radiológicos — IESS San Francisco

Aplicación de una sola página (`index.html`) que ahora usa **Cloud Firestore** como base
de datos principal y compartida de verdad entre todos los equipos que abran la web.

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
