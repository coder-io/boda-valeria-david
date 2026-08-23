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

| Orden | Fase | Valor | Riesgo | Depende de |
|---|---|---|---|---|
| 1 | Fase 0 — Salvaguardas | — | — | — |
| 2 | Fase 2 — Pétalos ambientales | Alto | Bajo | nada (ni GSAP) |
| 3 | Fase 1 — GSAP en el terreno | — | Bajo | Fase 0 |
| 4 | Fase 5a — Parallax de portada | Alto | Bajo | Fase 1 |
| 5 | Fase 4 — Lenis (condicionado) | Medio (solo desktop) | Medio | Fase 5a |
| 6 | Fase 3 — Migrar el reveal | Bajo | **Alto** | Fases 1 y 4 |
| 7 | Fase 5b — Split de texto | Medio | Medio | Fase 1 |
| 8 | Fase 6 — Rendimiento y pruebas | — | — | todas |

Cada fase termina en un commit propio. `index.html` es un único archivo de 91KB; sin
puntos de retorno, un error sale caro.

---

## Fase 0 — Salvaguardas (antes de tocar nada)

- Commit del estado actual / rama de trabajo.
- Definir el criterio de aceptación transversal: **con JS de terceros bloqueado, la
  invitación se sigue leyendo entera.** Se prueba bloqueando los CDN en DevTools.

## Fase 2 — Capa de pétalos ambientales (mobile-first) — *empezar por aquí*

Independiente de todo lo demás: no toca `scroll-behavior`, no necesita GSAP, no depende
de Lenis. Es el mayor impacto visual con el menor riesgo.

- **Extraer `FORMAS` (línea 1585) a un scope compartido antes de nada.** Hoy está
  declarada *dentro* del IIFE `petalos()`, que hace `return` temprano en la línea 1578
  si el puntero no es fino — o sea que en mobile, justo donde la capa nueva debe
  funcionar, esa constante nunca llega a existir.
- Nueva capa independiente de la actual (`#petalos` sigue siendo el rastro del cursor;
  esta es una capa aparte, ej. `#petalosAmbiente`, `position: fixed`,
  `pointer-events: none`).
- **z-index propio y decidido.** `#petalos` (línea 324) y `#novios` (línea 278)
  comparten `z-40`; la capa nueva no puede heredar ese valor por accidente.
- Pétalos generados por tiempo, no por movimiento de mouse — debe funcionar en touch.
- **Usar Web Animations API (`el.animate()`), no un bucle `requestAnimationFrame`
  manual.** Es lo que ya hace la capa de cursor (línea 1621) y se compone fuera del
  hilo principal. El vaivén tipo seno se consigue con ~6 keyframes de `transform`.
- Movimiento: caída vertical + oscilación lateral (mecerse), rotación propia, variación
  de velocidad/tamaño/opacidad por pétalo.
- Cantidad por viewport: ~10-14 vivos en mobile, ~24 en desktop.
- **Decisión de color pendiente:** las secciones alternan claras (`bg-fondo`,
  `bg-fondo2`) y oscura (`bg-hondo`), y la capa es `fixed`, así que cruza ambas. Un
  pétalo blanco es invisible sobre marfil. Opciones: (a) tomar color de las vars del
  tema como hace la capa de cursor (línea 1594), (b) silueta con trazo/sombra suave que
  funcione sobre ambos fondos. Decidir antes de implementar.
- **Dos capas de pétalos en desktop:** el rastro del cursor mantiene hasta 34 pétalos
  vivos (`MAX`, línea 1581). Sumado al ambiente puede saturar. Decidir: reducir el
  ambiente cuando `pointer: fine`, o retirar el rastro.
- Pausar la capa en `visibilitychange` para no gastar batería en segundo plano.
- Respetar `prefers-reduced-motion`: capa desactivada.

## Fase 1 — Preparar el terreno (GSAP)

- Añadir CDN de GSAP core + ScrollTrigger junto al `<script>` de Tailwind, **con
  versión fijada** (no `latest`).
- **No usar `defer` en esos `<script>`.** El script principal del sitio es *inline* al
  final del `<body>` (termina en la 1676); un script diferido se ejecuta *después* del
  parseo, o sea después del inline → `gsap is not defined`. Deben cargarse de forma
  bloqueante en el `<head>`, o convertir todo el inline en externo diferido, que
  contradice el "sin build step".
- **Guard de fallback obligatorio:** `if (!window.gsap) { /* ruta IntersectionObserver
  actual */ }`. Sin esto, la Fase 3 deja `.reveal { opacity: 0 }` sin nadie que lo
  revierta y la invitación se ve en blanco.
- Crear un bloque JS nuevo, separado del IIFE de pétalos-cursor, para no mezclar
  responsabilidades.
- Verificar que el sitio sigue funcionando igual antes de tocar animaciones.

## Fase 5a — Parallax de portada

- Micro-parallax de la imagen de fondo de la portada ligado al scroll. Valor alto,
  riesgo bajo, no toca el sistema de reveals.
- Es la fase que justifica o descarta Lenis (ver Fase 4).

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
