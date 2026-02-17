Tipo, formato y peso:

Video 720p (HD) 450px (o el ancho necesario)

1280x720 (720p) (para BG)

24fps / 30fps

.mp4 H.264 y de ser posible WebM

H.264 (estándar) o H.265 (HEVC). El H.265 es el "Wow" de la codificación actual: mantiene la misma calidad que el H.264 pero con casi la mitad del peso. Ideal para que la web de Kirei vuele.

En Handbrake:

Andá a la pestaña Video.

En Video Encoder, elegí H.265 (x265).

En Constant Quality, deslizá el marcador entre 22 y 24 RF (es el punto dulce donde el ojo humano no nota pérdida de calidad pero el archivo pesa muy poco).

Tildá Web Optimized.

Dimensiones: Si el video es para el Hero de fondo, con 1280px de ancho suele sobrar y ganás muchísima velocidad de carga.

usar esta estructura:

```
<video loop muted playsinline>

    <source src="video.webm" type="video/webm">

    <source src="video.mp4" type="video/mp4">

</video>

```

IMPORTANTE:

1. Bitrate: Al exportar que el video no supere los 2 Mbps.

2. Peso: Un video de fondo no debería pesar más de 2MB - 5MB. Si pesa más, afecta SEO.

3. Muted: El atributo muted es obligatorio para que el navegador permita el autoplay.

4. Lazy Load en Videos

preload="none": No descarga nada hasta que el usuario interactúa.

preload="metadata": Solo descarga el tamaño y duración (muy recomendado para no pesar al inicio / LPC).

---

Me parece una decisión excelente. Integrarlo en `global-utils.js` es la forma más limpia de escalar tu sitio; simplemente añades un video con un atributo específico y el sistema se encarga de la magia sin que tengas que escribir JS nuevo cada vez.

Para que esto no afecte el **LCP (Largest Contentful Paint)**, usaremos la técnica de `data-src`. El navegador ignorará el video al cargar la página y solo empezará la descarga cuando el usuario haga scroll hacia él.

### 1. Estructura HTML recomendada

En lugar de `src`, usaremos `data-src`. Así el navegador no "ve" el archivo pesado al inicio.

```html
<video 
    data-src="assets/videos/ritual-kirei.mp4" 
    muted 
    loop 
    playsinline 
    preload="none" 
    class="lazy-video">
</video>

```

### 2. El JS para `global-utils.js`

Este script es "elegante" porque utiliza el **Intersection Observer**, que es nativo y consume poquísimos recursos del procesador.

```javascript
/**
 * Global Video Lazy Loader & Auto-Play on Viewport
 * Detecta videos con la clase .lazy-video y los carga/reproduce solo al ser vistos.
 */
const initLazyVideos = () => {
    const lazyVideos = document.querySelectorAll('video.lazy-video');

    if ('IntersectionObserver' in window) {
        const videoObserver = new IntersectionObserver((entries, observer) => {
            entries.forEach(videoEntry => {
                if (videoEntry.isIntersecting) {
                    const video = videoEntry.target;
                    
                    // 1. Pasamos el data-src al src real para iniciar descarga
                    if (video.dataset.src) {
                        video.src = video.dataset.src;
                        video.load(); // Forzamos la carga inicial
                        delete video.dataset.src; // Limpiamos para no repetir
                    }

                    // 2. Intentamos reproducir
                    const playPromise = video.play();
                    
                    if (playPromise !== undefined) {
                        playPromise.catch(error => {
                            console.log("Autoplay preventivo: El video espera interacción o scroll.");
                        });
                    }

                    // Opcional: Si solo quieres que cargue una vez y siga su curso, 
                    // dejas de observar. Si quieres que se PAUSE al salir de vista, 
                    // quita el unobserve y añade un 'else'.
                    // observer.unobserve(video); 
                } else {
                    // OPCIONAL: Pausar si sale del viewport para ahorrar CPU
                    videoEntry.target.pause();
                }
            });
        }, {
            // Margen de 100px para que empiece a cargar un poquito antes de entrar
            rootMargin: '0px 0px 100px 0px',
            threshold: 0.1 // Se activa cuando el 10% del video es visible
        });

        lazyVideos.forEach(video => videoObserver.observe(video));
    }
};

// Ejecutar al cargar el DOM
document.addEventListener('DOMContentLoaded', initLazyVideos);

```

### 3. Beneficios técnicos para Kirei:

* **Impacto en LCP:** Al no tener `src` inicial, el navegador prioriza la descarga del Logo, el H1 y la primera imagen (tu LCP real). El video ni siquiera figura en la cola de descarga inicial.
* **Ahorro de Datos:** Si un usuario entra a la Home pero no hace scroll hasta el final donde está el video del ritual, **nunca descarga el video**. Esto es vital para usuarios con datos móviles.
* **Gestión de CPU:** Al incluir la lógica del `else { video.pause() }`, evitas que el navegador gaste recursos procesando un video que el usuario ya pasó de largo.

### Un pequeño truco de CSS

Para evitar saltos visuales (CLS) mientras el video carga, dale siempre un aspecto de proporción o un color de fondo:

```css
video.lazy-video {
    background: var(--color-kirei-mist); /* Color neutro de tu marca */
    min-height: 200px; /* Evita que el contenedor colapse antes de cargar */
    width: 100%;
    transition: opacity 0.5s ease;
}

video.lazy-video[src] {
    opacity: 1; /* Podrías poner opacidad 0 inicial para un fade-in */
}

```
La respuesta corta es: **el atributo `poster` nativo no es responsivo**. Si pones una imagen en `poster="imagen.jpg"`, el navegador cargará esa misma imagen tanto en un monitor 4K como en un celular pequeño.

Sin embargo, podemos aplicar una técnica "estilo Kirei" para que sea inteligente, aprovechando que ya estamos usando un **Intersection Observer**.

Aquí tienes las tres formas de manejar esto, de la más simple a la más profesional:

### 1. La técnica del "Object-Fit" (Sencilla)

Si usas una sola imagen de poster (por ejemplo, cuadrada 1:1), puedes hacer que se adapte al contenedor del video usando CSS. No cambia la resolución del archivo, pero sí cómo se ve.

```css
video.lazy-video {
    width: 100%;
    aspect-ratio: 16 / 9; /* O la que necesites */
    object-fit: cover;    /* La imagen del poster se estirará para cubrir el área */
    background-color: #f4f4f4;
}

/* Para mobile podrías cambiar el aspect-ratio */
@media (max-width: 600px) {
    video.lazy-video {
        aspect-ratio: 9 / 16; 
    }
}

```

---

### 2. El "Poster Responsivo" (Vía JS)

Como ya tenemos un script en `global-utils.js`, podemos decirle que detecte el ancho de pantalla y asigne un poster diferente antes de que el video cargue.

**HTML:**

```html
<video 
    class="lazy-video"
    data-src="video-desktop.mp4"
    data-poster-desktop="poster-16-9.jpg"
    data-poster-mobile="poster-9-16.jpg"
    muted loop playsinline preload="none">
</video>

```

**JS (Integrado en tu función `initLazyVideos`):**

```javascript
const video = videoEntry.target;
const isMobile = window.innerWidth < 600;

// Asignar poster según resolución ANTES de cargar el video
if (!video.poster) {
    video.poster = isMobile ? video.dataset.posterMobile : video.dataset.posterDesktop;
}

```

---

### 3. La técnica del "Skeleton" con CSS (La más optimizada)

En lugar de cargar una imagen de poster (que es otra petición al servidor), puedes usar un **graduado o un color sólido** que coincida con el primer frame del video. Esto es lo que hace YouTube o Netflix mientras cargan.

```css
/* Definimos las proporciones por clase */
.v-16-9 { aspect-ratio: 16 / 9; }
.v-9-16 { aspect-ratio: 9 / 16; }
.v-1-1  { aspect-ratio: 1 / 1; }

.lazy-video {
    display: block;
    width: 100%;
    height: auto;
    /* Efecto de carga suave */
    background: linear-gradient(110deg, #ececec 8%, #f5f5f5 18%, #ececec 33%);
    background-size: 200% 100%;
    animation: shine 1.5s linear infinite;
}

@keyframes shine {
    to { background-position-x: -200%; }
}

```

### Mi recomendación para tu caso:

Si buscas que el sitio vuele (mejor LCP):

1. **No uses el atributo `poster**` con imágenes pesadas.
2. Usa las **clases de proporción CSS** (`v-16-9`, etc.) para que el espacio ya esté reservado y no haya saltos de contenido (CLS).
3. Usa un **color de fondo sólido** que sea similar al color predominante de tu video.

**Un detalle importante:** Si el video es vertical (9:16) pero el usuario lo ve en Desktop, el video mostrará barras negras a los lados a menos que uses `object-fit: cover`.


---

| Uso | Resolución Recomendada | Bitrate |
| --- | --- | --- |
| **Hero Principal (Full Screen)** | 1920 x 1080 | 2500 - 3000 kbps |
| **Secciones Internas / Cards** | 1280 x 720 | 1500 - 2000 kbps |
| **Mobile (Vertical 9:16)** | 720 x 1280 | 1000 - 1500 kbps |