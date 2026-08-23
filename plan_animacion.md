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
| 5 | Fase 4 — Lenis (condicionado) | Medio (solo desktop) | Medio | Fase 5a | ✅ Hecho |
| 6 | Fase 3 — Migrar el reveal | Bajo | **Alto** | Fases 1 y 4 | ✅ Hecho |
| 7 | Fase 5b — Split de texto | Medio | Medio | Fase 1 | ✅ Hecho |
| 8 | Fase 6 — Rendimiento y pruebas | — | — | todas | ⏳ Siguiente |

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

## Fase 4 — Lenis, sin tocar touch — ✅ Completada

Implementada en `d7c08e9`. Decisión final, tras discutirlo: en vez de condicionar toda
la librería a `pointer: fine`, se instancia Lenis en cualquier dispositivo con
**`syncTouch: false` explícito** — así el modo continuo (rueda/trackpad) solo actúa en
desktop porque ahí es donde Lenis realmente engancha el scroll, mientras que
`lenis.scrollTo()` (destino puntual, no gesto continuo) queda disponible y funciona
bien en todos los dispositivos por igual.

- CDN de Lenis **1.1.18** (versión fijada, vía jsDelivr), cargada bloqueante junto a
  GSAP. Guard `typeof window.Lenis !== 'undefined'` igual que con GSAP.
- `lenis.raf` enganchado a `gsap.ticker.add(...)` en vez de un `requestAnimationFrame`
  propio — resuelve de raíz la preocupación original de "dos bucles rAF compitiendo"
  al no crear ninguno nuevo.
- `lenis.on('scroll', ScrollTrigger.update)` para mantener los triggers sincronizados.
- **CSS resuelto de forma distinta a lo previsto:** en vez de eliminar
  `html { scroll-behavior: smooth }`, se añadió `html.lenis-smooth { scroll-behavior:
  auto !important }` — la propia librería agrega esa clase a `<html>` solo mientras
  anima, así que el `scroll-behavior` original queda intacto el resto del tiempo (p.
  ej. si Lenis nunca llega a cargar).
- **`href="#padres"` migrado**, no solo "revisado": se intercepta el click y se llama
  `lenis.scrollTo('#padres')`, porque con la clase `lenis-smooth` activa durante el
  scroll el salto nativo se sentiría instantáneo.
- **`abrirPadrinos()`/`cerrarPadrinos()` migrados** a `lenis.scrollTo()`, con fallback
  a `scrollIntoView` si `lenis` es `null` (CDN caído o `reduced-motion`). El centrado
  de `cerrarPadrinos()` reproduce a mano la matemática de `block:'center'`
  (`offset = altura/2 - alturaViewport/2`), verificada leyendo el código fuente de
  Lenis: `offset` se suma directamente al `top` documental del elemento destino.
- `#enredadera` no se tocó: como `syncTouch` es `false`, en touch Lenis no intercepta
  el scroll y los eventos `scroll` nativos siguen disparándose igual que siempre; en
  desktop Lenis sigue escribiendo el `scrollTop` real (no un transform virtual), así
  que el listener existente sigue recibiendo eventos válidos sin cambios.
- Guardado detrás de `animGsap.gsapListo && !animGsap.reducida`, igual criterio que el
  resto de la infraestructura GSAP.
- **Pendiente de Fase 6:** verificar en dispositivo real que el flujo de
  RSVP/WhatsApp no se vio afectado (no debería, son solo `<a>` sin lógica de scroll).

## Fase 3 — Migrar el sistema de "reveal" a GSAP ScrollTrigger — ✅ Completada

Implementada en `02b81ba`. Se decidió (a propósito, preguntando antes de proceder)
hacer la **migración completa** de las 10 variantes en vez de acotarla a 2-3
secciones, aprovechando que toda la infraestructura de guard/fallback ya estaba
construida en fases anteriores. Arquitectura: un solo `if (animGsap.gsapListo &&
!animGsap.reducida) {...} else {...}` que decide en tiempo de ejecución entre el
sistema nuevo (GSAP) y el original (`IntersectionObserver` + CSS), este último
**sin ningún cambio** respecto a como estaba.

- Cada `data-efecto` mapeado a un objeto `EFECTOS[tipo]` con los mismos valores que
  ya tenía en CSS (`y`, `x`, `scale`, `rotation`, `filter`, y para `desplegar`/
  `cortina` sus casos especiales de `rotationX`/`clipPath`), para que el resultado
  visual sea equivalente al de antes.
- **Bug evitado antes de escribir código:** `gsap.from()` infiere el destino leyendo
  el estilo computado actual del elemento; como el CSS ya deja los `.reveal` ocultos
  de entrada (anti-FOUC), origen y destino habrían coincidido y no se habría visto
  ningún movimiento. Se usó `gsap.fromTo()` con ambos extremos explícitos.
- **`html.gsap-reveal .reveal { transition: none }`** añadido al CSS existente (no se
  tocó la regla `transition` original, solo se neutraliza cuando GSAP está activo) —
  resuelve exactamente el riesgo de jank que este documento marcaba.
- `data-stagger` migrado a `gsap.fromTo(hijos, ..., { stagger: .13 })`; ya no se
  inyecta `transitionDelay` inline en la rama GSAP (la rama de fallback sí lo sigue
  haciendo, sin cambios).
- **`.orla-caja` preservado tal cual pedía el plan:** no anima transform/opacity con
  GSAP (el CSS ya la deja siempre visible); solo se dispara el trazo vía
  `ScrollTrigger.create({..., onEnter: () => el.classList.add('visible')})`,
  reutilizando la transición CSS existente en `.tz > *`.
- **Collapse de padrinos resuelto con el enfoque correcto, más allá de lo previsto:**
  en vez de solo añadir `ScrollTrigger.refresh()`, sus `.reveal` (la intro + los
  hijos de `#padrinos`) se **excluyen por completo** del barrido automático — al
  estar en `grid-template-rows: 0fr` cuando está cerrado, un `ScrollTrigger` normal
  mediría su posición de forma incorrecta, no solo "tarde". Se animan a mano, una
  sola vez, en el click que abre el panel. Y sí se añadió
  `ScrollTrigger.refresh()` en el `transitionend` de `grid-template-rows`, porque
  abrir/cerrar cambia el alto real de todo lo que viene después en el documento.
- `#enredadera` no se tocó (no forma parte del sistema `.reveal`).
- Fallback de `prefers-reduced-motion` y guard de `gsapListo` heredados de la
  infraestructura de la Fase 1, sin duplicar lógica.
- **Verificación:** sintaxis validada con `node --check`. Pendiente de Fase 6:
  confirmar visualmente en navegador que las 10 variantes se ven equivalentes a
  como se veían con CSS, y que el collapse de padrinos anima correctamente en la
  primera apertura y no vuelve a animar en aperturas posteriores.

## Fase 5b — Detalles de elegancia adicionales — ✅ Completada (nombre de portada)

Implementada en `605cc9e`. Se hizo solo el candidato de mayor valor (el nombre de
portada); frases/otros textos quedan como candidatos futuros si se quiere repetir el
patrón.

- **Bump de versión necesario, no anticipado en el plan:** GSAP/ScrollTrigger subidos
  de `3.12.5` a `3.13.0` — se verificó con `curl` que SplitText devuelve 404 en
  `3.12.5` (era plugin de pago) y 200 en `3.13.0` (gratuito desde ahí). Sin este bump,
  la afirmación original de este documento ("SplitText es gratis desde 3.13") habría
  sido cierta pero inútil, porque seguíamos pinneados en una versión anterior.
- `animGsap.splitListo` como guard independiente de `gsapListo`, con su propio
  `gsap.registerPlugin(SplitText)` condicional — si ese único script falla, el resto
  de la infraestructura GSAP no se ve afectada.
- **Decisión de diseño no prevista: dividir por palabras, no por letra.** El nombre usa
  `font-script` (Great Vibes, cursiva conectada); por carácter, SplitText mete cada
  letra en su propia caja `inline-block` y rompe la conexión visual entre letras. Por
  palabra ("Valeria", "&", "David") da una entrada igual de elegante sin ese problema.
- **Dos bugs evitados antes de escribir el código final:**
  1. El bloque debía colocarse *después* del `$$('[data-bind]').forEach(...)` que
     rellena el `<h1>` — puesto antes, se habría dividido un elemento vacío.
  2. Aquí sí corresponde `gsap.from()` (a diferencia de la migración de la Fase 3):
     el `<h1>` no tiene ningún estado oculto por CSS de antemano, así que GSAP infiere
     correctamente el destino leyendo su estilo actual.
- `aria-label` con el texto completo puesto **antes** de dividir, para que el lector de
  pantalla anuncie el nombre completo y no palabra por palabra.
- Guardado detrás de `animGsap.gsapListo && animGsap.splitListo && !animGsap.reducida`.
- **Pendiente / fuera de esta pasada:** reveal de otras frases (frase de portada,
  itinerario), y micro-parallax adicional más allá del de portada (ya cubierto en
  Fase 5a). El resto de animaciones existentes (palomas, cuenta regresiva, enredadera)
  no se tocaron — no lo necesitaban tras la Fase 3/4.

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
