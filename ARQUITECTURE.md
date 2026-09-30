# Arquitectura: lectura y escritura en el filesystem

Documento de análisis centrado en cómo Waliki almacena, lee y renderiza el contenido de páginas como archivos en disco. El resto del sistema (ACL, admin, plugins) solo se menciona cuando toca ese flujo.

## Resumen

Waliki usa un **almacenamiento dual**:

| Capa | Qué guarda | Qué no guarda |
|------|------------|---------------|
| Base de datos (`Page`) | `title`, `slug`, `path`, `markup` | el cuerpo de la página |
| Filesystem (`WALIKI_DATA_DIR`) | texto plano UTF-8 (`.rst`, `.md`, …) | HTML renderizado |

El HTML se genera **en lectura** (con caché opcional). No hay HTML persistido en disco ni en la DB.

```text
Request ──► View ──► Page (DB) ──► page.body
                                      │
                         cache miss? ─┤
                                      ▼
                              page.raw (FS)
                                      ▼
                         markup.get_document_body
                                      ▼
                              sanitize(<script>)
                                      ▼
                           cache + template |safe
```

---

## Configuración relevante

Definida en [`waliki/settings.py`](waliki/settings.py):

| Setting | Rol |
|---------|-----|
| `WALIKI_DATA_DIR` | Raíz del contenido. Por defecto `<project_root>/waliki_data`. |
| `WALIKI_DEFAULT_MARKUP` | Markup por defecto (`reStructuredText` o el primero de `WALIKI_AVAILABLE_MARKUPS`). |
| `WALIKI_MARKUPS_SETTINGS` | Extensiones / overrides por markup (Markdown wikilinks+toc, RST writer HTML5, etc.). |
| `WALIKI_CACHE_TIMEOUT` | TTL del HTML cacheado (default 24h; en el demo `waliki_project` suele ser `0`). |
| `WALIKI_ATTACHMENTS_DIR` | Adjuntos (otro árbol; no es el cuerpo de la página). |
| `WALIKI_SANITIZE_FUNCTION` | Post-proceso del HTML (default: quitar `<script>…</script>`). |

Ruta absoluta de un archivo:

```text
abspath = abspath(join(WALIKI_DATA_DIR, page.path))
```

Ejemplo: slug `guia/instalacion`, markup Markdown → `path = guia/instalacion.md` → archivo en `waliki_data/guia/instalacion.md`.

---

## Modelo `Page` y el filesystem

Código: [`waliki/models.py`](waliki/models.py).

### Metadatos vs contenido

```python
class Page(models.Model):
    title = ...
    slug = ...          # URL / identidad lógica
    path = ...          # ruta relativa bajo WALIKI_DATA_DIR
    markup = ...        # "reStructuredText" | "Markdown" | ...
```

Al `save()`, si `path` está vacío:

```text
path = slug + markup_.file_extensions[0]
# p.ej. "home" + ".rst" → "home.rst"
```

### Lectura: propiedad `raw`

```python
@property
def raw(self):
    filename = self.abspath
    if not os.path.exists(filename) or os.path.isdir(filename):
        return ""
    return codecs.open(filename, "r", encoding="utf-8").read()
```

- Siempre UTF-8.
- Archivo ausente o directorio → string vacío (no excepción).
- Usado por el editor (`PageForm` pone `raw` como `initial`), la vista “Raw”, el render y el PDF.

### Escritura: setter `raw`

```python
@raw.setter
def raw(self, value):
    filename = join(WALIKI_DATA_DIR, self.path)
    makedirs(dirname(filename))   # ignora si ya existe
    codecs.open(..., "w", encoding="utf-8").write(value)
```

Crea directorios intermedios (namespaces tipo `foo/bar.md`).

### Movimiento y borrado

| Operación | Filesystem | DB |
|-----------|------------|-----|
| `page.move(new_path)` | `shutil.move` + actualiza `page.path` | hay que `save()` aparte |
| `update_extension()` | renombra extensión según markup | vía `move` |
| `pre_delete` (signal) | `os.remove(abspath)` si existe | fila eliminada |
| Redirects (`Redirect`) | no tocan FS | solo slugs HTTP |

---

## Pipeline de renderizado (lectura → HTML)

### Entrada HTTP

Vista [`detail`](waliki/views.py):

1. Normaliza `slug` (`strip('/')`).
2. Si hay `Redirect`, responde 301/302.
3. `Page.objects.get(slug=…)`. Si no existe → plantilla “página no existe / crear”.
4. Modo raw (`waliki_detail_raw`): `HttpResponse(page.raw, text/plain)` — **sin** markup.
5. Caso normal: `render(..., 'waliki/detail.html', {'page': page})`.

Plantilla [`waliki/detail.html`](waliki/templates/waliki/detail.html):

```django
{{ page.body|safe }}
```

### De `body` al HTML

```text
page.body
  → get_cached_content()
      → cache.get("waliki:content:<slug>")
      → si miss:
            _get_part("get_document_body")
              → markup_ = get_markup_instance(page.markup)
              → markup_.get_document_body(page.raw)   # LEE EL FS aquí
            sanitize(html)
            cache.set(..., WALIKI_CACHE_TIMEOUT)
```

Detalles:

- **`markup_`**: instancia lazy de Markups (subclases en [`waliki/_markups.py`](waliki/_markups.py)), cacheada en el objeto Python (`__markup_instance`), no en Django cache.
- **`get_document_body`**: API de la librería [Markups](https://github.com/retext-project/pymarkups) (docutils / Markdown / Textile).
- **Errores RST** (`docutils.utils.SystemMessage`): `_get_part` los captura y devuelve `''`.
- **`sanitize`**: por defecto solo elimina tags `<script>`; no es un sanitizer HTML completo. Por eso la plantilla usa `|safe`.

Otras “partes” del markup (mismo patrón `_get_part` + `raw`):

- `get_document_title` — usado al importar con `from_path`
- `get_stylesheet` / `get_javascript` — assets opcionales del markup

### Preview (sin página / sin cache de slug)

`Page.preview(markup, text)` convierte texto en memoria (AJAX del editor), aplica `sanitize` y no escribe al FS.

### Invalidación de caché

Signal `post_save` en `Page` → `cache.delete(page.get_cache_key())`.

Cualquier `save()` de metadatos o tras editar el raw limpia el HTML cacheado de ese slug.

---

## Pipeline de escritura (HTTP → FS → señales)

### Crear página

1. `new` / primer `POST` en `edit` → `Page.objects.create(slug=…)` (path se deriva en `save`).
2. `page.raw = ""` → crea archivo vacío en disco.
3. Señal `page_saved` (git puede hacer commit inicial).

### Editar

Vista [`edit`](waliki/views.py):

1. `page_preedit` — el plugin git inyecta el commit padre para merge.
2. `PageForm`: campo `raw` inicializado desde `instance.raw` (lectura FS).
3. Si cambia el markup → `update_extension()` (renomea archivo en disco).
4. `page.raw = form.cleaned_data['raw']` → **escritura FS**.
5. `page.save()` → DB + limpia cache HTML.
6. `page_saved` → git commit / auto-merge; puede lanzar `Page.EditionConflict`.

El formulario no persiste `raw` en la DB: el campo es del form, y el modelo lo escribe solo vía el setter.

### Mover

- Actualiza redirecciones en DB.
- Si no es “solo redirect”: `page.move(new_slug + extensión)` + `slug` nuevo + `save`.
- Señal `page_moved` → git `mv`.

---

## Markups y el filesystem

[`waliki/_markups.py`](waliki/_markups.py) adapta Markups:

| Clase | Extensiones típicas | Notas de render |
|-------|---------------------|-----------------|
| `ReStructuredTextMarkup` | `.rst` | Writer HTML5 propio, reader/transforms Waliki, autolinks RST |
| `MarkdownMarkup` | `.md` | Extensiones `wikilinks`, `toc`; output html5 |
| `TextileMarkup` | `.textile` | Opcional |

La extensión del archivo en disco **debe** coincidir con el markup almacenado en DB. Cambiar markup sin `update_extension` dejaría `path` desalineado.

Autolinks (no leen FS en sí, pero generan URLs hacia otras páginas):

- RST: targets indefinidos (`somewhere_`) → links internos.
- Markdown: `[[Somewhere]]` vía extensión WikiLinks + `get_url` → `reverse('waliki_detail', slug)`.

---

## Sincronización FS ↔ DB

Comando [`sync_waliki`](waliki/management/commands/sync_waliki.py):

```text
os.walk(WALIKI_DATA_DIR)
  → archivos con extensión en {'.rst', '.md'} (configurable)
  → si no hay Page con ese path: Page.from_path(path)
       - infiere markup por extensión
       - slug = get_slug(filename)
       - title = get_document_title(raw)   # lee y parsea el archivo
  → Pages cuyo abspath no existe: page.delete()  # también borra el registro;
                                                # el archivo ya faltaba
```

Útil para contenido sembrado a mano o clonado por git sin pasar por la UI.

---

## Rol de Git (colateral al FS)

El plugin `waliki.git` **no** sustituye la lectura en request. Opera sobre el mismo árbol:

- Tras `page_saved`: commit del archivo ya escrito.
- Tras `page_moved`: `git mv`.
- En `page_preedit`: guarda versión padre; al guardar, merge si hubo edición concurrente.

La fuente de verdad del **contenido servido** sigue siendo el working tree bajo `WALIKI_DATA_DIR`.

---

## Diagrama de secuencia: GET de una página

```text
Cliente          detail()           Page (DB)         FS              Markups / cache
   |                |                   |              |                     |
   |-- GET /slug -->|                   |              |                     |
   |                |-- get(slug) ----->|              |                     |
   |                |<-- page ----------|              |                     |
   |                |-- render detail.html             |                     |
   |                |     page.body ------------------>|                     |
   |                |                   |              |-- cache.get? ------>|
   |                |                   |              |   miss              |
   |                |                   |<-- raw open -|                     |
   |                |                   |-- get_document_body(raw) --------->|
   |                |                   |              |<-- HTML ------------|
   |                |                   |              |-- sanitize          |
   |                |                   |              |-- cache.set         |
   |<-- HTML -------|                   |              |                     |
```

## Diagrama de secuencia: POST de edición

```text
Cliente          edit()            PageForm          Page / FS         Signals (git)
   |               |                   |                 |                  |
   |-- POST ------>|                   |                 |                  |
   |               |-- page_preedit ---|-----------------|----------------->|
   |               |-- validate ------>|                 |                  |
   |               |                   |-- raw = text -->| write file       |
   |               |                   |-- save() ------>| DB + clear cache |
   |               |-- page_saved -----|-----------------|----------------->|
   |               |                   |                 |              commit/merge
   |<-- redirect --|                   |                 |                  |
```

---

## Invariantes y límites

1. **Contenido canónico = archivo en `WALIKI_DATA_DIR`**. La DB es índice.
2. **Render on read**: no hay artefacto HTML en disco.
3. **Slug y path pueden divergir** tras moves mal hechos o sync parcial; `sync_waliki` alinea por `path`/existencia de archivo.
4. **Sanitización mínima**: XSS vía markup malicioso no está cubierto de forma exhaustiva.
5. **Caché**: con `WALIKI_CACHE_TIMEOUT = 0` (demo) el HTML no se reutiliza entre requests de forma útil.
6. **Encoding**: solo UTF-8 en lectura/escritura de `raw`.
7. **Namespaces**: subdirectorios bajo `WALIKI_DATA_DIR` mapean a slugs con `/` (p.ej. `docs/faq.rst` ↔ slug `docs/faq`).

---

## Mapa de archivos del flujo

| Archivo | Responsabilidad |
|---------|-----------------|
| [`waliki/models.py`](waliki/models.py) | `raw` R/W, `body`, cache, `from_path`, move/delete FS |
| [`waliki/_markups.py`](waliki/_markups.py) | Adaptadores Markups → HTML |
| [`waliki/settings.py`](waliki/settings.py) | `WALIKI_DATA_DIR`, markups, cache, sanitize |
| [`waliki/views.py`](waliki/views.py) | detail / edit / new / preview / move / delete |
| [`waliki/forms.py`](waliki/forms.py) | `PageForm.raw` ↔ filesystem vía modelo |
| [`waliki/templates/waliki/detail.html`](waliki/templates/waliki/detail.html) | `{{ page.body\|safe }}` |
| [`waliki/utils.py`](waliki/utils.py) | `get_slug`, `sanitize`, `get_url` |
| [`waliki/management/commands/sync_waliki.py`](waliki/management/commands/sync_waliki.py) | Import / limpieza FS ↔ DB |
| [`waliki/git/models.py`](waliki/git/models.py) | Commits sobre el mismo árbol tras señales |
| [`waliki/pdf/views.py`](waliki/pdf/views.py) | Consume `page.raw` para PDF (rst2pdf) |

---

## Conclusión

Waliki trata el wiki como **archivos versionables (git) indexados por Django**. Cada request de lectura resuelve slug → fila `Page` → archivo → conversión Markups → HTML (cacheable). Cada escritura válida actualiza primero el archivo y luego metadatos/señales. Entender `page.raw` y `page.body` es suficiente para localizar casi todo el comportamiento filesystem.
