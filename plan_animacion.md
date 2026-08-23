# Plan de animación — invitación de boda

> Revisado y corregido contra `index.html` (1678 líneas). Todas las referencias de
> línea de este documento están verificadas sobre el archivo actual.

## Objetivo

Reemplazar las animaciones genéricas de scroll (`fadeIn`/`bounce` vía CSS +
`IntersectionObserver`) por una experiencia de scroll coreografiada y elegante con
GSAP + ScrollTrigger, y añadir una capa ambiental de pétalos de rosa cayendo y
meciéndose que funcione también en mobile (a diferencia de la capa actual, que solo
sigue al cursor y se desactiva en touch).

Todo se implementa dentro de `dist/index.html` (sin build step: Tailwind ya se carga
por CDN, se añade GSAP igual). Prioridad: que se vea bien y rinda bien en mobile, que
es donde la mayoría verá el sitio.

## Restricciones a respetar

- Sin bundler/npm: todo vía `<script src>` de CDN, igual que Tailwind hoy.
- Respetar `prefers-reduced-motion` en todas las animaciones nuevas (el archivo ya lo
  hace en 6 puntos).
- No degradar el rendimiento en gama media/baja: limitar partículas, evitar
  post-processing pesado, usar `will-change` con moderación.
- Mantener el contenido dirigido por `CONFIG` y los `data-bind`/`data-stagger`
  existentes — no reescribir el marcado de las secciones.
- **El sitio nunca puede quedar en blanco.** Si un CDN falla, el contenido debe verse.
  Es una invitación que 200 personas abrirán desde datos móviles.

## Orden de ejecución

El plan original iba 1→6, pero ese orden no coincide con el de riesgo/valor. Orden real:

| Orden | Fase | Valor | Riesgo | Depende de | Estado |
|---|---|---|---|---|---|
| 1 | Fase 0 — Salvaguardas | — | — | — | ✅ Hecho |
| 2 | Fase 2 — Pétalos ambientales | Alto | Bajo | nada (ni GSAP) | ✅ Hecho |
| 3 | Fase 1 — GSAP en el terreno | — | Bajo | Fase 0 | ✅ Hecho |
| 4 | Fase 5a — Parallax de portada | Alto | Bajo | Fase 1 | ✅ Hecho |
| 5 | Fase 4 — Lenis (condicionado) | Medio (solo desktop) | Medio | Fase 5a | ⏳ Siguiente (decisión pendiente) |
| 6 | Fase 3 — Migrar el reveal | Bajo | **Alto** | Fases 1 y 4 | Pendiente |
| 7 | Fase 5b — Split de texto | Medio | Medio | Fase 1 | Pendiente |
| 8 | Fase 6 — Rendimiento y pruebas | — | — | todas | Pendiente |

Cada fase termina en un commit propio. `index.html` es un único archivo de 91KB; sin
puntos de retorno, un error sale caro.

---

## Fase 0 — Salvaguardas (antes de tocar nada) — ✅ Completada

- Commit del estado actual / rama de trabajo. (`1d92137` — plan añadido al repo.)
- Criterio de aceptación transversal definido: **con JS de terceros bloqueado, la
  invitación se sigue leyendo entera.** Pendiente de ejecutar esta prueba en la Fase 6,
  una vez haya CDN de terceros que puedan fallar (GSAP).

## Fase 2 — Capa de pétalos ambientales (mobile-first) — ✅ Completada

Implementada en `410d428`, `ef23201` y `8f6737b`. Sin GSAP ni Lenis, como estaba
previsto.

- **`FORMAS`** extraída del IIFE `petalos()` a scope de módulo compartido, tal como
  exigía el riesgo original (antes nunca existía en touch por el `return` temprano).
- **`#petalosAmbiente`**: capa nueva, `position: fixed`, `z-index: 35` propio (no
  hereda el `40` de `#petalos`/`#novios`).
- Pétalos generados por tiempo (`setTimeout` recursivo), no por el cursor →
  funcionan en touch. Movimiento vía `el.animate()` (Web Animations API): caída +
  vaivén lateral en 5 keyframes + rotación, tal como pedía el plan.
- **Color resuelto:** blanco con trazo tinta muy sutil (`stroke: rgb(var(--c-ink)/.22)`)
  + `drop-shadow`, para que se lea tanto sobre fondos claros como oscuros sin
  coordinar con la sección visible — la opción (b) de las dos que planteaba este
  documento.
- **Saturación con la capa del cursor:** no se redujo `MAX` del rastro del cursor;
  se dejó así tras validación visual con el usuario. Revisar si en la Fase 6 se ve
  saturado en desktop con puntero fino.
- Cantidad por viewport: 12 en mobile (<640px), 24 en desktop — dentro del rango que
  proponía el plan.
- **Tres planos de profundidad** (mejora no prevista en el plan original, pedida tras
  ver el resultado): cerca (~50-70px, opaco, cae rápido), media (~34-46px) y lejos
  (~18-26px, tenue, cae lento), con más peso en los planos lejanos para simular
  profundidad de campo.
- Pausa completa en `visibilitychange` (con fix de un bug propio: el `setTimeout`
  no se reanudaba al volver a la pestaña por un ID de timeout caducado que
  evaluaba `truthy`).
- Desactivada bajo `prefers-reduced-motion` (CSS `display:none` + guard que ni
  arranca el IIFE).
- **Verificación:** sintaxis validada con `node --check` sobre el script extraído.
  Visualmente confirmada por el usuario en el navegador — pétalos y profundidad
  se ven bien. No se probó aún en dispositivo real de gama media (queda para
  Fase 6).

## Fase 1 — Preparar el terreno (GSAP) — ✅ Completada

Implementada en `51d53fb`.

- CDN de GSAP **3.12.5** (versión fijada, no `latest`) + ScrollTrigger, vía cdnjs,
  añadidos justo después del `<script>` de Tailwind en el `<head>`. Ambas URLs
  verificadas con `curl` (200 OK) antes de commitear.
- **Sin `defer`**, tal como exigía el riesgo original: el script principal sigue
  siendo inline al final del `<body>`, así que los scripts de GSAP se cargan de
  forma bloqueante para que `gsap`/`ScrollTrigger` existan cuando el inline corra.
- **Guard de fallback implementado:** `animGsap.gsapListo` se calcula comprobando
  `typeof window.gsap`/`window.ScrollTrigger`; si el CDN falla, se registra un
  `console.warn` y nada más — el `IntersectionObserver` de `.reveal` (que esta fase
  no toca) sigue siendo el único que corre. Cumple el criterio de "nunca en blanco".
- Bloque nuevo (`animGsap`), separado del IIFE de pétalos-cursor y de cualquier otra
  lógica existente, colocado junto a las utilidades (`$`, `$$`, `get`) cerca del
  principio del script principal.
- `gsap.registerPlugin(ScrollTrigger)` se ejecuta solo si ambas librerías cargaron.
- **Verificación:** sintaxis validada con `node --check`; el sistema de reveal
  actual no se modificó, así que el sitio se comporta exactamente igual que antes
  de esta fase — pendiente solo la prueba real de "bloquear el CDN en DevTools" que
  corresponde formalmente a la Fase 6.

## Fase 5a — Parallax de portada — ✅ Completada

Implementada en `09df0e2`.

- `<section>` de portada e `<img>` con `id="portada"` / `id="imgPortada"` para poder
  apuntarles desde GSAP sin tocar el resto del marcado.
- `gsap.set(imgPortada, { scale: 1.15 })` como base, para que GSAP registre la
  escala en su propio transform antes de animar `yPercent: 12` con `scrub: true`
  ligado a `ScrollTrigger` (`start: 'top top'`, `end: 'bottom top'`). El
  `overflow-hidden` que ya tenía la sección recorta el sobrante de la imagen
  escalada, así el desplazamiento nunca revela un borde vacío.
- Guardado detrás de `animGsap.gsapListo && !animGsap.reducida`: si el CDN de
  GSAP falla o hay `prefers-reduced-motion`, la portada queda estática, igual que
  antes de esta fase.
- **Ahora sí se decide sobre Lenis** (ver Fase 4): con este parallax en producción,
  hay algo real que Lenis puede suavizar en desktop.

**Ajustes posteriores, fuera del alcance original de esta fase:**
- `object-position` de `#imgPortada` fijado en `32% center` (commit `b9aada7`): la
  foto es panorámica y la pareja está a ~32% del ancho, no en el centro geométrico;
  `object-cover` por defecto recortaba usando el centro de la imagen y los dejaba
  desplazados a la izquierda en pantallas más angostas que la foto.
- Zoom por pellizco bloqueado en todo el sitio (commit `add6b00`): se añadió
  `maximum-scale=1.0, user-scalable=no` al `<meta viewport>`. No es parte del plan
  de animación, pero se hizo en la misma sesión — **nota de accesibilidad:** esto le
  quita a alguien con baja visión la opción de hacer zoom (WCAG 1.4.4); pedido
  explícito del usuario, queda documentado por si se quiere revisar más adelante.

## Fase 4 — Lenis, condicionado a desktop

**Qué aporta realmente:** suaviza la rueda del ratón (que sin él avanza a saltos
discretos de ~100px) y, sobre todo, hace que el `scrub`/parallax se vea continuo en vez
de a escalones. También normaliza la diferencia entre rueda de Windows y trackpad de
Mac. Nada de eso aplica a táctil: ahí el scroll nativo ya tiene inercia del sistema y
corre en el compositor, y por eso el default de Lenis (`syncTouch: false`) literalmente
no hace nada en móvil.

**Decisión:** integrarlo **solo donde aporta**, no globalmente:

```js
const usarLenis = matchMedia('(pointer: fine)').matches &&
                  !matchMedia('(prefers-reduced-motion: reduce)').matches;
```

Así el caso mayoritario (táctil) mantiene el scroll nativo intacto y el riesgo queda
acotado a desktop. Si al final se descarta el parallax de la Fase 5a, esta fase sobra.

Correcciones obligatorias si se integra:

- **Eliminar `html { scroll-behavior: smooth }` (línea 83)** o Lenis y el navegador
  pelean por el mismo scroll. (Ojo: la línea 145 lo pone en `auto` bajo
  `reduced-motion`; esa parte se mantiene.)
- **Migrar los `scrollIntoView({behavior:'smooth'})` a `lenis.scrollTo()`:**
  `abrirPadrinos()` (línea 1201) y `cerrarPadrinos()` (línea 1209).
- Revisar el ancla `href="#padres"` (línea 363) y el deep-link `#padrinos`.
- Conectar a ScrollTrigger: `lenis.on('scroll', ScrollTrigger.update)`.
- **`#enredadera` no se rompe con Lenis** (usa `window.addEventListener('scroll')` +
  rAF en la línea 1345, y Lenis mueve el scroll real, no un transform), pero quedarían
  dos bucles rAF compitiendo. Engancharlo a `lenis.on('scroll')` o al ticker de GSAP.
- Verificar que el flujo de RSVP/WhatsApp (`#rsvp`, líneas 628 y 747) sigue bien.

## Fase 3 — Migrar el sistema de "reveal" a GSAP ScrollTrigger

**Advertencia de valor:** esta fase reescribe 10 variantes CSS que *ya funcionan*
(líneas 112-133) para obtener lo mismo pero en JS. Justificarla solo si se quiere
coreografía real —timelines encadenadas entre bloques—, no como fin en sí. Alternativa
razonable: aplicarla solo a 2-3 secciones donde aporte y dejar el resto con el
`IntersectionObserver` actual.

- Reutilizar los atributos `data-efecto` de cada `<section>` como mapeo a un tipo de
  animación GSAP en vez de a una variante CSS.
- **Neutralizar la `transition` de `.reveal` (líneas 100-108) antes de animar con
  GSAP.** Si no, cada frame de GSAP dispara además una transición CSS sobre las mismas
  propiedades → jank garantizado. El CSS debe quedar solo como estado inicial anti-FOUC.
- **Usar `once: true` / `toggleActions`, no `scrub`, para las entradas.** Con scrub el
  texto *retrocede* al hacer scroll hacia arriba; la semántica actual del código es
  "una vez y ya" (`obs.unobserve`, línea 1662). El scrub es para parallax (Fase 5a), no
  para entradas.
- Migrar `data-stagger` a `gsap.timeline` con `stagger`, y **quitar la inyección de
  `transitionDelay` inline en los hijos (línea 1658)** o los delays CSS quedan sumándose
  por debajo.
- Casos especiales a preservar: `.orla-caja` (trazo que se dibuja, `stroke-dasharray`,
  líneas 185-189), enredadera del itinerario (`#enredadera`), collapse de padrinos.
- **Collapse de padrinos: necesita `ScrollTrigger.refresh()`, no el parche actual.** El
  código de hoy (líneas 1668-1676) marca `.visible` a mano porque los `.reveal` dentro
  de `#padrinosPanel` miden 0px cerrados y nunca cruzan el umbral del observer. Con
  ScrollTrigger el equivalente es refrescar **al terminar la transición de altura del
  panel**; si no, las posiciones de trigger quedan calculadas sobre un panel colapsado.
- Mantener fallback de `prefers-reduced-motion` y el guard de `!window.gsap`.

## Fase 5b — Detalles de elegancia adicionales

- Reveal de texto letra por letra/línea por línea (portada, nombres, frases).
  **Usar SplitText, no SplitType:** SplitText es plugin gratuito de GSAP desde 3.13, así
  que no añade otro CDN.
- **Accesibilidad:** al partir el `<h1>` de los nombres hay que poner `aria-label` en el
  contenedor, o el lector de pantalla lo lee letra por letra.
- Mantener el resto de animaciones que ya funcionan bien (palomas por sección
  `data-pose`, cuenta regresiva, enredadera) — solo ajustar donde haga falta para que
  convivan con ScrollTrigger.

## Fase 6 — Rendimiento y pruebas en mobile

- Probar en un dispositivo real de gama media (no solo DevTools) con throttling de CPU.
- **Prueba de fallo de red:** bloquear los CDN en DevTools y confirmar que la invitación
  se lee entera.
- **Coste de carga:** la página ya carga Tailwind por CDN, que es un compilador JIT en
  runtime y bloquea el render. Sumarle GSAP + ScrollTrigger + Lenis es el riesgo real de
  esta fase, más que la cantidad de partículas. Son ejes distintos —Tailwind afecta al
  tiempo hasta el primer render, los pétalos al FPS sostenido— y ninguno cancela al
  otro. Si el peso preocupa de verdad, la respuesta es compilar Tailwind, no diferir
  scripts.
- **Mejora de perf independiente y barata:** `actualizarEnredadera()` (línea 1327) hace
  `getBoundingClientRect()` por cada ítem del itinerario **en cada frame de scroll**.
  Cachear los offsets en `construirEnredadera()`/resize. No depende de ninguna otra fase.
- Verificar que `prefers-reduced-motion` desactiva correctamente: pétalos ambientales,
  parallax, Lenis y las entradas.
- Verificar que no se rompe el layout (scroll horizontal, overlaps) en anchos de mobile
  (360-430px).
- Revisar consumo de batería/CPU en sesión larga (el countdown tiene un `setInterval` de
  1s corriendo siempre, línea 1092 — confirmar que sumado a lo nuevo no genera jank).

## Entregable final

`dist/index.html` actualizado con:
- Capa de pétalos ambientales funcionando en mobile.
- Parallax de portada, y scroll suave en desktop si la Fase 4 se confirma.
- Coreografía de scroll donde aporte, con fallback intacto si GSAP no carga.
- Mismo contenido y estructura de secciones, sin build step adicional.
