# Presentación EduTrack — DSY1106

Dos entregables con el **mismo contenido** (la web es la principal; el `.pptx` es solo por
exigencia institucional, su fidelidad visual es menor):

| Archivo | Qué es | Cuándo usarlo |
|---|---|---|
| `edutrack-slides.html` | Deck **reveal.js** autocontenido (se proyecta en el navegador) | presentación en vivo |
| `edutrack.pptx` | Versión PowerPoint/Google Slides (12 slides) | entrega institucional / editar |

## Abrir la presentación web

> reveal.js se carga desde CDN (jsDelivr), así que necesitas **conexión a internet** la
> primera vez. El navegador moderno bloquea `file://` para algunos recursos, por eso lo más
> fiable es servirla por HTTP:

```bash
cd presentacion
python3 -m http.server 8000
# abrir http://localhost:8000/edutrack-slides.html
```

(Abrir el `.html` con doble clic también suele funcionar; si algún diagrama no carga, usa el
servidor local de arriba.)

### Controles
- `→ / ←` o `Espacio`: avanzar/retroceder. Algunas slides tienen **sub-slides verticales**
  (flecha `↓`): diagrama de arquitectura, secuencia, código y resultados de tests.
- `Esc`: vista general · `S`: notas del orador · `F`: pantalla completa.

### Exportar la web a PDF
1. Abrir `http://localhost:8000/edutrack-slides.html?print-pdf`
2. Imprimir → **Guardar como PDF** (márgenes: ninguno; gráficos de fondo: activado).

## Estructura

```
presentacion/
├── edutrack-slides.html         ← deck principal (reveal.js)
├── edutrack.pptx                ← versión institucional
└── assets/
    ├── diagramas/
    │   ├── arquitectura.svg / .png        ← diagrama de arquitectura (editable: SVG)
    │   ├── toma-asistencia.svg / .png     ← caso destacado: Open/Closed + Adapter
    │   ├── resultados-tests.html          ← gráfico de tests (HTML autocontenido)
    │   └── respaldo-permisos/             ← diagrama de permisos (respaldo, no se presenta)
    └── evidencia/
        ├── RESUMEN-tests.txt              ← conteo agregado real
        └── *.txt                          ← reportes surefire reales (muestra)
```

Los diagramas son **SVG/HTML autocontenidos** (sin PNG rasterizado como fuente). Los `.svg`
son editables; los `.png` existen solo porque PowerPoint no embebe SVG.

> El diagrama de la verificación de permisos (`respaldo-permisos/caso-destacado.svg`) quedó
> respaldado: era correcto pero se cambió el caso destacado por la **toma de asistencia**,
> que ilustra un patrón conocido (principio Abierto/Cerrado + Adapter).

## Contenido (8 temas → 14 vistas en la web)

1. Portada · 2. Contexto y objetivos · 3. Plan vs. realidad (+ estado 6/9 MS) ·
4. Stack + diagrama de arquitectura · 5. Caso destacado: **toma de asistencia**
(principio Abierto/Cerrado + Adapter — diagrama + snippets + patrones) ·
6. Testing backend/frontend con resultados reales · 7. Guion de la demo · 8. Cierre.

## Datos reales (verificados desde el repo)

- **Backend:** 206 tests, 0 fallos (auth 62, student 40, attendance 38, assessment 33,
  course 17, annotation 16). Fuente: `*/target/surefire-reports/*.txt` (`./mvnw test`).
- **Frontend:** 26 archivos de test; cobertura ~96% statements / 88.8% branches /
  97.4% functions / 97.6% lines. Fuente: `pnpm test:coverage` → `front/coverage/index.html`.

Para refrescar cifras antes de presentar, vuelve a correr los tests y actualiza los números
en `edutrack-slides.html` (busca "206", la tabla de barras) y regenera el `.pptx`.

## ⚠️ Pendiente de completar
- Portada: **nombres de integrantes** y **fecha** — marcados como `{{ COMPLETAR NOMBRES }}`
  y `{{ FECHA }}` en `edutrack-slides.html` (slide 1) y en `edutrack.pptx` (slide 1).

## Regenerar el `.pptx`
Requiere `python-pptx` y `rsvg-convert` (para SVG→PNG):
```bash
rsvg-convert -w 1600 assets/diagramas/arquitectura.svg    -o assets/diagramas/arquitectura.png
rsvg-convert -w 1600 assets/diagramas/toma-asistencia.svg -o assets/diagramas/toma-asistencia.png
python build_pptx.py   # incluido en esta carpeta
```

> Nota: `INSTRUCCIONES_SLIDES.md` (raíz del repo) es el brief narrativo **original** y sirve
> como guía de apoyo. Ojo: describe el caso destacado anterior (verificación de permisos);
> el deck ya usa la **toma de asistencia**.
