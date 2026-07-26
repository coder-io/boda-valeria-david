# Invitación de boda · Valeria & David

Sitio estático de una sola página. 24 de abril de 2027, Huasca de Ocampo, Hidalgo.

## Publicar en GitHub Pages

1. Sube **todo el contenido de esta carpeta** a la raíz del repositorio.
2. En el repo: **Settings → Pages → Source: Deploy from a branch → `main` → `/ (root)`**.
3. El repositorio debe ser **público** (o tener GitHub Pro).

En un par de minutos queda en `https://TU-USUARIO.github.io/TU-REPO/`.

## Contenido

```
index.html      la invitación completa (estilos, animaciones y lógica dentro)
favicon.svg     icono de la pestaña
img/            la única foto que usa el sitio
.nojekyll       evita que GitHub procese los archivos con Jekyll
```

## Antes de mandar el enlace

- [ ] **Datos reales** — todo lo editable está en el objeto `CONFIG`, dentro del
      `<script>` al final de `index.html`: nombres, fecha, lugares, padres,
      padrinos, itinerario, hoteles y datos bancarios.
- [ ] **WhatsApp** — `CONFIG.rsvp.whatsapp`, en formato internacional sin `+`
      ni espacios (ej. `5215512345678`).
- [ ] **Lista de invitados** — `CONFIG.rsvp.invitados`, con el código y los
      lugares de cada uno.
- [ ] **Vista previa del enlace** — en las etiquetas `og:` del `<head>`,
      sustituye `TU-USUARIO` y `TU-REPO`. Sin esto, al compartir por WhatsApp
      no aparece la foto.
- [ ] **Mapa** — `CONFIG.mapaEmbed`, con el "Insertar un mapa" de Google Maps
      de la hacienda real.

## Confirmaciones de asistencia

Tal como está, las confirmaciones **se guardan solo en el navegador del
invitado**: tú no las recibes. Para que te lleguen, pon una URL en
`CONFIG.rsvp.endpoint` y el formulario hará un `POST` con los datos en JSON.

La opción más simple y gratuita es un Google Apps Script que escriba en una
hoja de cálculo. Mientras tanto, el botón de respaldo por WhatsApp sí funciona:
arma un mensaje con todos los datos ya rellenados.

## Enlaces personalizados

Cada invitado puede recibir su propio enlace con el código ya puesto:

```
https://TU-USUARIO.github.io/TU-REPO/?codigo=VYD-100
```

Al abrirlo ve su nombre y sus lugares reservados, sin teclear nada.

## Nota sobre privacidad

El código fuente de una página pública es visible para cualquiera. Los códigos
de invitado no son un problema, pero **la CLABE y los datos bancarios quedan a
la vista de quien tenga el enlace**. Si prefieres evitarlo, quítalos de
`CONFIG.regalos` y compártelos por privado a quien los pida.

## Nota técnica

Los estilos se generan con Tailwind desde CDN, que compila en el navegador de
cada visitante: verás un aviso en consola y la página parpadea un instante al
cargar. Funciona bien, pero para la versión definitiva conviene compilar el CSS
a un archivo estático.

Las tipografías vienen de Google Fonts y el mapa de Google Maps, así que el
sitio necesita conexión a internet para verse completo.
