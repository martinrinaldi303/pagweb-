# Retro Green Salón · Documentación técnica

Para Martín. Estado al 24/09/2026.
Repositorio: <https://github.com/senxrgreep-hub/pagweb-> · Sitio: <https://peluqueriagreen.com>

---

## 1. Qué es esto y cómo está armado

Un sitio de una sola página para Retro Green Salón (Av. Emilio Caraffa 2086, Córdoba), publicado
gratis con GitHub Pages, más un panel en `/admin` para que el dueño cambie textos, fotos, horarios y
secciones sin tocar código.

| Archivo / carpeta | Para qué sirve |
|---|---|
| `index.html` | **Toda la página.** HTML, estilos y scripts en un solo archivo. Sin compilar, sin framework. |
| `datos/*.json` | Lo que el dueño edita: textos, fotos, galería, horarios, secciones y contacto. |
| `img/` | Imágenes. Lo que sube el dueño va a `img/subidas/`. |
| `admin/index.html` | El arranque del panel: pantalla de carga, la vista previa y la carga del CMS. |
| `admin/sveltia-cms.js` | **El programa del panel** (Sveltia CMS 0.220.0), guardado acá a propósito. Ver §4. |
| `admin/config.yml` | Qué campos ve el dueño. Es la fuente de verdad de todo el panel. |
| `admin/vista.css` · `admin/vista.js` | La vista previa en vivo. Ver §5. |
| `admin/mapa.js` | Qué texto de la página cambia cada campo del panel. Ver §5.3. |
| `CNAME` · `.nojekyll` | Dominio para Pages y "publicá los archivos tal cual". No borrarlos. |

### Cómo fluyen los datos

```
El dueño edita en /admin
        ↓  (Sveltia CMS hace commit vía la API de GitHub)
datos/*.json en la rama main
        ↓  (GitHub Pages publica, 1–2 min)
index.html los lee con fetch() al cargar
        ↓
La página se pinta con esos valores
```

**Regla de oro del diseño: nada de lo que se guarde desde el panel puede romper la página.**
Cada texto, foto y lista tiene su valor escrito en el HTML como respaldo. Si `datos/` no se puede
leer, o si un campo queda vacío, se usa lo que está en el HTML. Campo por campo, no todo o nada.

---

## 2. Infraestructura

| Qué | Dónde | Notas |
|---|---|---|
| Hosting | GitHub Pages, cuenta `senxrgreep-hub`, repo `pagweb-`, rama `main`, carpeta raíz | Gratis. El repo tiene que ser público. |
| Dominio | `peluqueriagreen.com`, registrado en DonWeb | Vence el 18/09/2027. DNS apunta a las 4 IP de GitHub Pages. |
| HTTPS | Certificado de GitHub, gratis | No hace falta el que ofrece DonWeb. |
| Login del panel | Worker de Cloudflare `sveltia-cms-auth.senxrgreep.workers.dev` | GitHub Pages no puede guardar secretos: el OAuth pasa por ahí. |
| Permisos | Solo cuentas con permiso de escritura en el repo pueden guardar | Cualquier otro no pasa del login. |

---

## 3. Lo que se hizo en esta sesión

Tres commits propios, todos en `main`:

| Commit | Qué |
|---|---|
| `62811b6` | El panel se puede usar desde el celular, con la página al lado |
| `3a311e6` | Que el panel abra en el celular y nunca quede una pantalla negra |
| `1035bf2` | Las secciones nuevas aceptan varias fotos, no una sola |
| `68bdb1c` | Texto de políticas al pie, arriba del copyright |

(Entre medio hay commits `Update Contenido del sitio "…"`: esos los genera el panel cuando el dueño
guarda. Son datos, no código.)

---

## 4. La pantalla negra: el problema más importante que se arregló

**Síntoma:** desde el celular, `/admin/` se veía completamente en negro. Siempre, sin ningún mensaje.

**Causa:** `admin/index.html` bajaba el programa del panel desde `unpkg.com` con un `<script>` en el
`<head>`. Un script ahí es *bloqueante*: el navegador para de leer el HTML hasta que lo descarga. Si
esa dirección no responde —filtro del operador, DNS, región—, el navegador queda trabado y **la
página ni siquiera llega a tener `<body>`**. Sin `<body>` no hay nada que dibujar; con el modo oscuro
del teléfono, eso se ve negro.

Lo comprobé simulando que el CDN cuelga: a los 4 segundos `document.body` seguía siendo `null`.

**Arreglo, en dos partes:**

1. **El programa vive en el repo** (`admin/sveltia-cms.js`, 2,1 MB). Ya no depende de que el celular
   alcance un servidor ajeno. Como respaldo quedan dos copias en internet, **con la versión fijada**:
   antes se bajaba "la última", así que una actualización del autor podía romper el panel sin aviso.
2. **Pantalla de arranque** con su propio CSS embebido en `admin/index.html`, a propósito: si no se
   pudiera descargar ningún archivo, ese cartel igual se ve. Si falla, muestra el motivo y un botón
   de reintentar en vez de dejar la pantalla vacía.

### Cómo actualizar Sveltia más adelante

No hace falta seguido. Cuando quieras:

```bash
npm pack @sveltia/cms
tar xzf sveltia-cms-*.tgz
cp package/dist/sveltia-cms.js admin/sveltia-cms.js
```

Cambiá también el número de versión de las dos copias de respaldo en `admin/index.html` (están en el
array `COPIAS`) y **abrí el panel una vez** para confirmar que entra antes de dar por buena la
actualización.

---

## 5. La vista previa (`admin/vista.css` + `admin/vista.js`)

Muestra la web de verdad mientras se edita. En la compu, una columna a la derecha; en el celular,
una ventanita abajo. Vive **por fuera** de Sveltia: no toca su código, solo le acomoda el lugar.

### 5.1 La idea central

La vista previa **nunca tapa los campos**. El panel se achica exactamente lo que ella mide. En el
celular se descuenta además el alto del teclado, medido con `visualViewport`.

### 5.2 Decisiones que parecen raras y tienen motivo

Si tocás este archivo, leé esto antes:

- **El alto del panel se mide, no se calcula.** `acomodar()` lee `VISTA.getBoundingClientRect().top`
  en vez de hacer `calc(100% - alto)`. Es a prueba de las dos formas distintas en que los navegadores
  del celular cuentan el alto de la pantalla.
- **Se fuerza `box-sizing: border-box`.** Sin eso, un panel con márgenes internos queda más alto de
  lo pedido y se mete abajo de la ventanita. Me costó 24 píxeles y un rato entenderlo.
- **Los estilos se ponen como `!important`, y al soltarlos se devuelve el valor anterior.** Borrar a
  lo bruto le sacaba al panel el `right: 0` que venía de su propio `inset: 0`.
- **Se fuerza `overflow-y: auto` en lo que se achica.** La raíz de Sveltia es `overflow: hidden` y no
  se desplaza: al achicarla quedaban botones abajo, inalcanzables.
- **`enEdicion()` esconde la vista si el hash no es `#/collections/`.** En la pantalla de inicio de
  sesión no hay nada que previsualizar y los botones de entrar merecen la pantalla entera.

### 5.3 `mapa.js`: de un campo del panel a un texto de la página

Cada texto editable de `index.html` está marcado con `data-texto="seccion.clave"`. `mapa.js` dice qué
clave le corresponde a cada campo del panel, buscándolo por el nombre que Sveltia le pone al recuadro
(`aria-label`, con la forma `"⁨Frase principal⁩" Field`).

**Si agregás o renombrás un campo en `config.yml`, hay que actualizar `mapa.js`**, o ese campo deja de
verse en vivo (todo lo demás sigue funcionando).

Estado verificado: **151 textos en la página ↔ 151 destinos en el mapa**, y los **147 nombres de campo
del mapa existen exactamente igual en el panel real**.

---

## 6. Cómo probarlo

No hace falta entrar con GitHub. Sveltia soporta un modo de prueba con datos en memoria.

```bash
# 1. Servir el sitio
python -m http.server 8000

# 2. Panel normal (pide login con GitHub):
#    http://localhost:8000/admin/
#    → "Trabajar con un repositorio local" guarda directo en datos/ e img/subidas/

# 3. Panel de prueba, sin cuenta: copiar admin/ y en el config.yml de la copia
#    reemplazar todo el bloque backend por:
#        backend:
#          name: test-repo
#    → botón "Work with Test Repository", entra directo.
```

Con Playwright (Chromium) se automatiza todo: entrar, recorrer las 6 pantallas, desplegar los grupos,
escribir en los campos y comprobar que la página de la ventanita cambia. **Ojo:** el panel dibuja los
campos de a poco a medida que te acercás, así que una prueba que mire demasiado rápido ve menos
campos de los que hay. Hay que desplazarse e ir juntando.

### Lo que quedó verificado

- 23 comprobaciones de pantalla: iPhone, Android chico, tablet, celular acostado y escritorio.
  En todas, el borde de abajo del panel coincide exacto con el borde de arriba de la vista previa.
- Las 6 pantallas del panel abren y dibujan sus campos (1, 4, 5, 1, 1 y 13 campos declarados).
- Los 147 nombres de campo coinciden con el panel real.
- Escribir se refleja en la página al instante; guardar responde sin error.
- Agregar filas en Horarios, fotos en Galería y secciones nuevas: funciona.
- Cero errores de JavaScript en todo el recorrido.

### Lo que NO está verificado

1. **Entrar con la cuenta de GitHub y guardar de verdad.** Eso pasa por el Worker de Cloudflare y la
   cuenta del dueño. (Dato: el dueño ya guardó varias veces desde que se publicó, así que en la
   práctica funciona.)
2. **Subir una foto desde el teléfono.** Los controles están y responden, pero elegir el archivo abre
   el selector del sistema, que no se puede automatizar.

---

## 7. Pendientes, ordenados por lo que más mueven la aguja

### 1. Vista previa al compartir el link — *15 minutos*

Faltan `og:image` y `og:title`. El salón se difunde por WhatsApp e Instagram, o sea que **su canal
principal muestra un link pelado** en vez de la foto del local con el nombre. Es el arreglo de mayor
retorno de todos.

### 2. Peso de las fotos — *1 hora*

6,5 MB en total, y la de portada sola pesa **720 KB**. Es lo primero que carga, arriba de todo, muchas
veces con datos móviles. El panel ya las pasa a WebP, pero a 1920px y calidad 82; bajando a 1600/75
no se nota y pesa casi la mitad. Se cambia en `admin/config.yml`, en `media_libraries`.

### 3. El CDN de Tailwind — *1 a 2 horas*

`<script src="https://cdn.tailwindcss.com">` es la versión **para desarrollo**: genera todos los
estilos en el navegador de cada visitante y bloquea el dibujado. Es la misma clase de problema que
causó la pantalla negra del panel, y la última dependencia externa crítica que queda.
Compilar el CSS al repo y sacar el script.

### 4. Datos estructurados y favicon — *30 minutos*

Faltan `application/ld+json` (negocio local: dirección, horarios, teléfono), `rel="icon"` y
`canonical`. Para "peluquería Córdoba" pesa más el perfil de Google Business, pero ayuda.

### 5. Analítica — *30 minutos*

Hoy nadie sabe si la página recibe 3 visitas o 300. Sin eso, cualquier decisión sobre la web es a
ciegas.

### 6. Los 6 registros DNS que faltan — *5 minutos, pero hay que entrar a DonWeb*

GitHub Pages publica desde cuatro servidores y la zona tiene cargado **uno solo** (`185.199.108.153`
y `2606:50c0:8000::153`). Si ese se cae, se cae la web, aunque los otros tres estén sanos. Faltan
`185.199.109/110/111.153` y `2606:50c0:8001/8002/8003::153`.

### 7. Las fotos de respaldo apuntan a direcciones que caducan — *20 minutos*

Dentro de `index.html` quedaron **7 direcciones `lh3.googleusercontent.com`**: son las imágenes
generadas del principio, que funcionan como respaldo si `datos/` no se puede leer. Hoy no se ven
nunca, porque el dueño ya cargó las reales. Pero esas direcciones son temporales: el día que caduquen,
el respaldo no ataja nada. Conviene reemplazarlas por fotos del propio repositorio.

### Lo que NO haría

- **Rediseñarla.** El diseño está bien y es distintivo. Tocarlo es gastar en lo que ya funciona.
- **Un sistema de turnos online.** Para un salón de este tamaño gana WhatsApp: el dueño ya vive ahí,
  no hay curva de aprendizaje y no hay nada nuevo que mantener.

### Detalle menor

En la portada dice "una experiencia **unica**", sin tilde. Se arregla desde el panel en diez segundos
(*Textos de la página → Portada → "Frase debajo de la principal"*).

---

## 8. Políticas del pie

Bloque breve arriba del copyright, pensado para un sitio informativo de un comercio en Argentina:
quiénes somos, datos personales, la leyenda de la Ley 25.326 con la Agencia de Acceso a la Información
Pública, los terceros que carga la página y una nota sobre fotos ilustrativas, precios sujetos a
cambio y Defensa del Consumidor (Ley 24.240).

Antes de redactarlo verifiqué en el código que lo que afirma sea cierto: **0 formularios, 0 cookies,
0 almacenamiento propio**.

La dirección y la ciudad salen de "Teléfono y ubicación" (`data-texto`), así que se actualizan solas.
El resto es texto legal y queda fuera del panel a propósito.

**Tres advertencias:**

1. **No es un dictamen legal.** Es una base razonable, no el trabajo de un abogado. Si querés quedarte
   tranquilo, que lo lea alguien del rubro: son 20 minutos para un profesional.
2. **Falta CUIT y razón social.** La Resolución 104/2005 pide identificar al proveedor con esos datos.
   No los tenía y preferí no inventarlos.
3. **Falta a propósito** el "botón de arrepentimiento" y el enlace a la Ventanilla Única de Defensa
   del Consumidor: son obligatorios para sitios que **venden online**, y este no vende, deriva a
   WhatsApp. Ponerlos daría a entender una operación de comercio electrónico que no existe.

---

## 9. Cosas que conviene no olvidar

- **El dominio vence el 18/09/2027.** Ponete un recordatorio: si se cae, se cae todo.
- **El repo tiene que seguir siendo público** para que Pages sea gratis. El panel igual está protegido.
- **No borrar `.nojekyll` ni `CNAME`.**
- **El enlace del crédito del pie** (tu WhatsApp) no se toca desde el panel, a propósito.
- **Martín ya no es colaborador del repositorio** (lo sacó por privacidad, 24/09). Para proponer
  cambios hay que ir por *Fork + Pull Request*, o que el dueño le dé permiso de nuevo.
- **El dominio, el Worker de Cloudflare y la OAuth App siguen en cuentas de Martín.** Si alguna de
  esas cuentas desaparece, el dueño se queda sin panel o sin web. Conviene pasárselas.
- **Cada cambio del dueño queda en el historial de Git.** Cualquier error se deshace con un revert.
- **El dueño ya está usando el panel solo.** Guardó textos, galería, horarios, secciones y subió
  fotos sin ayuda. Ese era el objetivo.
