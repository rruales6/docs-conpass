# conpass — Directrices visuales de la landing
**Tipografía, color, estilos y estructura visual (v1.1)**
Estas directrices describen el sistema visual del prototipo, independiente de su implementación. Sirven para reconstruir el sitio en cualquier tecnología.

---

## 1. Principio rector

**Duotono tinta + verde sobre papel.** La página se siente impresa, sobria y editorial — no "tech". El verde se reserva para lo que importa: acciones, la notificación, la tarjeta. Todo lo demás vive en tintas y papel. Menos es más: nada de degradados decorativos, iconografía innecesaria ni cifras de relleno.

**Terminología fija:** el producto es la **tarjeta de lealtad** (EN: *loyalty card*). Nunca se le llama "pase" ni "pass" — en ningún texto, rótulo ni etiqueta.

---

## 2. Paleta de color

### Tintas (neutros)
| Rol | Valor | Uso |
|---|---|---|
| Tinta | `#1B231E` | Titulares, texto principal, botones secundarios, secciones oscuras |
| Tinta profunda | `#141B16` / `#10160F` | Fondos de énfasis (demo intermedia, footer), pantalla del teléfono |
| Gris texto | `#4A544D` | Párrafos y texto de apoyo |
| Gris suave | `#6A736C` / `#7A837C` | Subtítulos, notas, metadatos |
| Gris placeholder | `#9AA39B` | Contenido pendiente (logos, citas, QR) |

### Papel (fondos claros)
| Rol | Valor | Uso |
|---|---|---|
| Papel | `#FAFAF7` | Fondo base de la página |
| Papel tostado | `#F3F2EC` | Secciones alternas (video, prueba social, enterprise) |
| Blanco | `#FFFFFF` | Tarjetas sobre papel |

### Verde (acento — usar con moderación)
| Rol | Valor | Uso |
|---|---|---|
| Verde conpass | `#1E7A50` | CTA primario, tarjeta de lealtad, enlaces, precios destacados, el punto del logo |
| Verde oscuro | `#145C3B` | Hover de enlaces |
| Verde claro | `#8FD1AE` / `#8FB8A2` | Énfasis de texto sobre fondos oscuros |
| Tinte verde | `rgba(30,122,80,.055–.1)` | Fondo de la columna conpass en el comparador, CTA terciarios |

**Regla:** máximo 2 fondos por página (papel y papel tostado) más las secciones de tinta. El verde nunca se usa como fondo de sección — solo en componentes (botones, tarjeta, notificación).

---

## 3. Tipografía

### Familias
- **Newsreader (serif)** — titulares y momentos editoriales. Peso 500, itálica para citas y remates ("El cartón no es gratis…"). Interletrado ligeramente negativo (−1 a −1.5 %).
- **Instrument Sans** — UI: botones, navegación, etiquetas, datos, cifras. Pesos 400–650.
- **Work Sans** — texto de párrafo (cuerpo). Peso 400, línea 1.5–1.6.
- **Monospace del sistema** — solo para marcar placeholders (etiquetas de QR, logos, guion del video). Señala "esto no es contenido final".

### Escala (móvil → desktop)
Los titulares escalan de forma fluida con el ancho de pantalla (`clamp()` o equivalente): el valor móvil es el mínimo y crece hasta el máximo en desktop.
| Nivel | Fuente | Tamaño / línea | Uso |
|---|---|---|---|
| H1 | Newsreader 500 | 37 → 52px / 1.12 | Titular del hero |
| H2 | Newsreader 500 | 27 → 34px / 1.2 | Títulos de sección |
| H2 menor | Newsreader 500 | 24 → 29px / 1.25 | Secciones secundarias |
| Cita / remate | Newsreader itálica | 15–17px | Frases editoriales |
| Cuerpo | Work Sans 400 | 14.5–16.5px / 1.55 | Párrafos |
| UI | Instrument Sans 600 | 13–16px | Botones, navegación |
| Etiqueta | Instrument Sans 600–650 | 10–12px, mayúsculas, tracking +7–9 % | Rótulos de columna, badges |
| Nota | Work Sans 400 | 11.5–12.5px | Legales, aclaraciones |

**Reglas:**
- El contraste serif/sans es la voz de la marca: Newsreader habla, Instrument Sans opera.
- Énfasis dentro de párrafos: **negrita en tinta** sobre claro, **verde claro** sobre oscuro. Nunca subrayado ni cursiva para énfasis funcional.
- `text-wrap: pretty` (o equivalente) en titulares y párrafos.

---

## 4. Logotipo

Palabra "conpass" en minúsculas, Instrument Sans 650, interletrado −2 %, seguida de un **punto verde** (`#1E7A50`, circular, ~30 % del cuerpo de la letra) alineado a la línea base. Nunca en mayúsculas; el punto nunca cambia de color.

---

## 5. Estructura y ritmo de página

**Diseño fluido, mobile-first.** En móvil todo es columna única; en tablet y desktop cada sección se centra con su propio ancho máximo — no hay un contenedor global estrecho. El contenido siempre respira a **22px** de los bordes laterales.

| Sección | Ancho máx. | Comportamiento en tablet/desktop |
|---|---|---|
| Nav | 1140px | Misma barra; el menú sigue siendo desplegable |
| Hero | 1060px | Dos columnas: texto (máx. 560px) + teléfono, centradas verticalmente |
| Video, Enterprise | 760px | Columna centrada |
| Comparador | 1104px | Rejilla de tarjetas (2–3 columnas, mín. 300px por tarjeta) |
| Cómo funciona | 1060px | Los 3 pasos en fila (mín. 280px por paso) |
| Demo intermedia, ROI | 560–604px | Columna estrecha centrada — momento íntimo |
| Prueba social | 920px | Citas lado a lado |
| Precios | 1144px | 4 tarjetas en fila (mín. 250px), botones alineados abajo |
| FAQ | 804px | Columna centrada |
| CTA final | 640px | Columna centrada |
| Footer | 1096px | Igual, alineado al ancho de nav |

Las rejillas colapsan solas a columna única en móvil (auto-fit con mínimos). Orden narrativo fijo:

1. **Nav** (sticky, papel translúcido con blur, borde inferior fino)
2. **Hero** (papel) — titular, subtítulo, CTA doble, teléfono bloqueado
3. **Video** (papel tostado)
4. **Comparador** (papel)
5. **Cómo funciona** (tinta — primera ruptura oscura)
6. **Demo intermedia** (tinta profunda — continúa la oscura)
7. **ROI** (papel)
8. **Prueba social** (papel tostado)
9. **Precios** (papel)
10. **Enterprise** (papel tostado)
11. **FAQ** (papel)
12. **CTA final** (tinta)
13. **Footer** (tinta profunda)

**Ritmo:** claro → claro alterno → oscuro, en bloques. Las secciones oscuras marcan los dos momentos de mayor convicción (cómo funciona + demo; cierre). Separadores: bordes de 1px en `rgba(tinta, .08–.1)`, nunca sombras entre secciones.

**Espaciado vertical:** secciones respiran 40–52px arriba y abajo; 44–48px es el estándar. Título de sección a contenido: 18–28px.

---

## 6. Componentes

### Botones
- **Primario:** relleno verde, texto blanco, radio 12px, padding vertical 14–16px. En el hero lleva un pulso suave de sombra verde (una sola vez por página).
- **Secundario:** relleno tinta, texto papel, radio 12px o píldora (99px) en la nav.
- **Terciario:** solo texto verde, o contorno verde de 1.5px sin relleno.
- Área táctil mínima: **44px**. Texto de botón: Instrument Sans 600.

### Tarjetas
Fondo blanco, borde 1px `rgba(tinta,.1)`, radio 14–16px, padding 18–20px. **Sin sombras** (la elevación se reserva para la notificación y el teléfono). La tarjeta destacada (plan Growth) invierte: fondo tinta, texto papel, badge verde en píldora.

### El teléfono (motivo central)
Bisel oscuro con degradado sutil verde-tinta, radio exterior ~38px, pantalla con radio ~28px. Reloj grande en peso ligero (300). La **notificación** es el único elemento con sombra dramática (`0 8px 24px rgba(0,0,0,.35)`) y entra con animación de rebote suave desde arriba. La **tarjeta de lealtad** es la única superficie 100 % verde: sellos como círculos con contorno blanco que se rellenan.

### Placeholders de contenido pendiente
Trama diagonal de rayas suaves (tonos papel), borde discontinuo, etiqueta en monospace gris. Se usa igual para: póster del video, QR, logos de comercios, avatares y citas. Nunca inventar contenido real.

### Comparador
Tarjetas por criterio, dos columnas internas: la del cartón en gris neutro, la de conpass con tinte verde de fondo. El criterio va centrado en mayúsculas; los rótulos de columna en 12px — el de conpass en verde y mayúsculas ("CONPASS"). Solo la fila clave ("cuando el premio está cerca") lleva el texto en negrita. En pantallas anchas, las tarjetas de criterio se acomodan en rejilla de 2–3 columnas.

### FAQ
Acordeón con bordes inferiores de 1px; indicador +/− en verde; sin cajas ni fondos.

---

## 7. Movimiento

Discreto y con propósito — el movimiento cuenta la historia del producto (la notificación que llega sola):

| Animación | Uso | Carácter |
|---|---|---|
| Entrada de notificación | Lock screen (hero y demo) | Caída con rebote suave, ~0.7s, retardo ~1s |
| Pulso | Solo el CTA del hero | Sombra verde que respira, ciclo 3.2s |
| Fade-up | Acordeones, modal, contenido revelado | 0.2–0.35s, desplazamiento 10px |

Nada más se mueve. Sin parallax, sin scroll-triggers, sin carruseles.

---

## 8. Bilingüe (ES/EC + EN)

- Conmutador ES/EN en píldora, siempre visible en la nav.
- **No es traducción espejo:** el copy ES usa "usted", cartón sellado, WhatsApp y Google Wallet primero; el EN prioriza Apple Wallet y stamp card. El orden de los botones de wallet se invierte según el idioma.
- Los precios se muestran en USD en ambos idiomas.

---

## 9. Lo que NO se hace

- Degradados de color como decoración (solo el bisel del teléfono).
- Emoji e iconografía genérica; los iconos se limitan a marcas de wallet y señales funcionales (+/−, ▶, ✕).
- Sombras en tarjetas o secciones.
- Más de un elemento pulsando por pantalla.
- Verde como fondo de sección o de página.
- Contenido inventado donde va material real: siempre placeholder tramado.
