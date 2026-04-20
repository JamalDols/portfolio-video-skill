# Portfolio Video Generator

Genera videos de scroll para portfolio web usando Puppeteer (grabación) + Remotion (composición).
Cada video muestra una página web scrolleando dentro de un panel con fondo oscuro.

---

## Paso 1 — Recoger inputs del usuario

Usa `AskUserQuestion` con estas preguntas (todas en un solo bloque, máximo 4):

**Pregunta 1 — Páginas a grabar** (texto libre)
Pide al usuario que liste las páginas en formato:
```
slug → URL completa
```
Ejemplo:
```
home → https://miempresa.com/
work → https://miempresa.com/work/
about → https://miempresa.com/about/
```

**Pregunta 2 — Color de fondo** (opciones + texto libre)
- `#111111` — Negro carbón (Recommended)
- `#0a0a1a` — Azul noche
- `#1a1a1a` — Gris oscuro
- `#ffffff` — Blanco

**Pregunta 3 — Velocidad de scroll** (opciones)
- 14 segundos — Estándar
- 20 segundos — Lento (Recommended para páginas largas)
- 28 segundos — Muy lento
- Personalizado (texto libre)

**Pregunta 4 — ¿Alguna página es móvil/responsive?** (multiselect de los slugs introducidos)
Las páginas marcadas se grabarán a 390px de ancho con marco de móvil en lugar del panel estándar.

---

## Paso 2 — Determinar ubicación del proyecto

El proyecto de videos se crea en un directorio hermano al directorio de trabajo actual.
Nombre por defecto: `{nombre-proyecto}-videos`

Ejemplo: si el usuario trabaja en `/Sites/miempresa`, crear en `/Sites/miempresa-videos`.

Pregunta al usuario si quiere otra ubicación antes de continuar.

---

## Paso 3 — Crear la estructura del proyecto

Crea este árbol de directorios y archivos:

```
{proyecto}-videos/
├── package.json
├── tsconfig.json
├── remotion.config.ts
├── src/
│   ├── index.ts
│   ├── Root.tsx
│   ├── components/
│   │   ├── PageVideo.tsx      ← panel estándar con <Video>
│   │   └── PhoneFrame.tsx     ← marco de móvil (solo si hay páginas responsive)
│   └── compositions/
│       └── {Slug}.tsx         ← una por página
├── scripts/
│   └── record.ts
├── public/                    ← aquí irán los .mp4 grabados
├── recordings/                ← frames temporales (se auto-limpia)
└── out/                       ← videos finales renderizados
```

### package.json

```json
{
  "name": "{proyecto}-videos",
  "version": "1.0.0",
  "scripts": {
    "dev": "remotion studio",
    "record": "tsx scripts/record.ts",
    "render:all": "tsx scripts/render-all.ts"
  },
  "dependencies": {
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "remotion": "^4.0.0"
  },
  "devDependencies": {
    "@remotion/cli": "^4.0.0",
    "@types/node": "^20.0.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "puppeteer": "^22.0.0",
    "tsx": "^4.21.0",
    "typescript": "^5.4.0"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src", "scripts"]
}
```

### remotion.config.ts

```ts
import { Config } from "@remotion/cli/config";
Config.setVideoImageFormat("jpeg");
Config.setJpegQuality(95);
Config.setOverwriteOutput(true);
Config.setConcurrency(4);
```

### src/index.ts

```ts
import { registerRoot } from "remotion";
import { RemotionRoot } from "./Root";
registerRoot(RemotionRoot);
```

---

## Paso 4 — Generar scripts/record.ts

Adapta este template con los slugs, URLs y viewports del usuario.
Las páginas marcadas como responsive usan `viewport: { width: 356, height: 796 }` y `deviceScaleFactor: 2`.
Las estándar usan `viewport: { width: 1000, height: 1040 }` y `deviceScaleFactor: 1`.
El `scrollDuration` usa el valor elegido por el usuario (puede ser distinto por página si el usuario lo pide).

```ts
/**
 * Graba las páginas scrolleando → public/{slug}.mp4
 *
 * Uso:
 *   npm run record              → todas las páginas
 *   npm run record -- {slug}   → una sola página
 */
const FPS = 60;
const EASE_POWER = 2.5;
const PAUSE_AT_TOP_MS = 600;

const PAGES = {
  // GENERADO: un entry por página
  "{slug}": {
    url: "{url}",
    viewport: { width: 1000, height: 1040 },  // o 356x796 para responsive
    scrollDuration: {N},
    deviceScaleFactor: 1,                      // o 2 para responsive
  },
} as const;

type PageSlug = keyof typeof PAGES;

import { execSync } from "child_process";
import fs from "fs";
import path from "path";
import puppeteer from "puppeteer";

const FRAMES_DIR = path.resolve(__dirname, "../recordings/frames");
const PUBLIC_DIR = path.resolve(__dirname, "../public");

function easeInOut(t: number, power: number): number {
  if (t < 0.5) return 0.5 * Math.pow(2 * t, power);
  return 1 - 0.5 * Math.pow(2 * (1 - t), power);
}

async function recordPage(slug: PageSlug) {
  const cfg = PAGES[slug];
  const totalFrames = Math.round(cfg.scrollDuration * FPS);
  const outputPath = path.join(PUBLIC_DIR, `${slug}.mp4`);

  console.log(`\n${"═".repeat(60)}`);
  console.log(`📹 ${slug} (${cfg.viewport.width}×${cfg.viewport.height} @${cfg.deviceScaleFactor}× · ${FPS}fps · ${cfg.scrollDuration}s)`);

  if (fs.existsSync(FRAMES_DIR)) fs.rmSync(FRAMES_DIR, { recursive: true });
  fs.mkdirSync(FRAMES_DIR, { recursive: true });

  const browser = await puppeteer.launch({
    headless: true,
    args: ["--no-sandbox", "--disable-setuid-sandbox", "--ignore-certificate-errors"],
  });

  const page = await browser.newPage();
  await page.setViewport({ ...cfg.viewport, deviceScaleFactor: cfg.deviceScaleFactor });

  await page.goto(cfg.url, { waitUntil: "networkidle0", timeout: 45000 });
  await new Promise((r) => setTimeout(r, 3000));

  await page.evaluate(() => {
    const w = window as Window & { lenis?: { destroy(): void } };
    if (w.lenis) { try { w.lenis.destroy(); } catch (_) {} }
    document.documentElement.style.scrollBehavior = "auto";
    document.body.style.scrollBehavior = "auto";
  });

  const totalHeight = await page.evaluate(() => {
    window.scrollTo(0, document.body.scrollHeight);
    return document.body.scrollHeight;
  });
  await new Promise((r) => setTimeout(r, 1500));
  await page.evaluate(() => window.scrollTo(0, 0));
  await new Promise((r) => setTimeout(r, 800));

  const maxScroll = totalHeight - cfg.viewport.height;
  const pauseFrames = Math.round((PAUSE_AT_TOP_MS / 1000) * FPS);
  const scrollFrames = totalFrames - pauseFrames;

  for (let i = 0; i < totalFrames; i++) {
    let scrollY = 0;
    if (i >= pauseFrames) {
      const progress = (i - pauseFrames) / Math.max(scrollFrames - 1, 1);
      scrollY = Math.round(easeInOut(Math.min(progress, 1), EASE_POWER) * maxScroll);
    }
    await page.evaluate((y) => {
      window.scrollTo(0, y);
      const st = (window as Window & { ScrollTrigger?: { update(): void } }).ScrollTrigger;
      if (st) st.update();
    }, scrollY);
    await new Promise((r) => setTimeout(r, 6));
    const framePath = path.join(FRAMES_DIR, `frame-${String(i).padStart(6, "0")}.jpg`);
    await page.screenshot({ path: framePath as `${string}.jpg`, type: "jpeg", quality: 90 });

    if (i % 60 === 0 || i === totalFrames - 1) {
      process.stdout.write(`\r   ${Math.round((i / (totalFrames - 1)) * 100)}% (${i + 1}/${totalFrames})`);
    }
  }

  await browser.close();

  execSync(
    `ffmpeg -y -r ${FPS} -i "${FRAMES_DIR}/frame-%06d.jpg" ` +
    `-c:v libx264 -preset slow -crf 16 -pix_fmt yuv420p ` +
    `-vf "scale=${cfg.viewport.width}:${cfg.viewport.height}" ` +
    `"${outputPath}"`,
    { stdio: "inherit" }
  );

  fs.rmSync(FRAMES_DIR, { recursive: true });
  console.log(`\n   ✅ ${outputPath}`);
}

async function main() {
  const arg = process.argv[2] as PageSlug | undefined;
  const slugs = arg ? [arg] : (Object.keys(PAGES) as PageSlug[]);
  for (const slug of slugs) await recordPage(slug);
}

main().catch((err) => { console.error(err); process.exit(1); });
```

---

## Paso 5 — Generar src/components/PageVideo.tsx

Sustituye `{BG_COLOR}` con el color elegido por el usuario.

```tsx
import React from "react";
import { interpolate, staticFile, useCurrentFrame, useVideoConfig, Video } from "remotion";

const CANVAS_W = 1080;
const CANVAS_H = 1080;
const PAD = 40;
const PANEL_W = CANVAS_W - PAD * 2; // 1000
const PANEL_H = CANVAS_H - PAD;     // 1040
const RADIUS = 14;
const BG = "{BG_COLOR}";

interface Props { videoFile: string }

export const PageVideo: React.FC<Props> = ({ videoFile }) => {
  const frame = useCurrentFrame();
  const { durationInFrames, fps } = useVideoConfig();
  const fadeDuration = Math.round(fps * 0.7);

  const opacity = interpolate(
    frame,
    [0, fadeDuration, durationInFrames - fadeDuration, durationInFrames],
    [0, 1, 1, 0],
    { extrapolateLeft: "clamp", extrapolateRight: "clamp" }
  );

  return (
    <div style={{ width: CANVAS_W, height: CANVAS_H, background: BG, opacity, position: "relative" }}>
      <div style={{
        position: "absolute", top: PAD, left: PAD, width: PANEL_W, height: PANEL_H,
        borderTopLeftRadius: RADIUS, borderTopRightRadius: RADIUS,
        borderBottomLeftRadius: 0, borderBottomRightRadius: 0,
        overflow: "hidden",
        boxShadow: "0 -8px 40px rgba(0,0,0,0.5), 0 30px 80px rgba(0,0,0,0.7)",
      }}>
        <Video
          src={staticFile(videoFile)}
          style={{ width: PANEL_W, height: PANEL_H, objectFit: "cover", objectPosition: "top left", display: "block" }}
        />
      </div>
    </div>
  );
};
```

---

## Paso 6 — Generar src/components/PhoneFrame.tsx (solo si hay páginas responsive)

El área de pantalla (SCREEN_W × SCREEN_H) debe coincidir con el viewport definido en record.ts para esa página (356×796).
Sustituye `{BG_COLOR}` con el color elegido.

```tsx
import React from "react";
import { interpolate, staticFile, useCurrentFrame, useVideoConfig, Video } from "remotion";

const CANVAS_W = 1080;
const CANVAS_H = 1080;
const PAD = 40;
const PANEL_W = CANVAS_W - PAD * 2;
const PANEL_H = CANVAS_H - PAD;
const BG = "{BG_COLOR}";

// Área de pantalla — coincide con viewport en record.ts
const SCREEN_W = 356;
const SCREEN_H = 796;
const BORDER = 14;
const PHONE_W = SCREEN_W + BORDER * 2;        // 384
const PHONE_H = SCREEN_H + BORDER * 2 + 20;  // 844
const RADIUS = 56;
const INDICATOR_W = 120;
const INDICATOR_H = 5;

interface Props { videoFile: string }

export const PhoneFrame: React.FC<Props> = ({ videoFile }) => {
  const frame = useCurrentFrame();
  const { durationInFrames, fps } = useVideoConfig();
  const fadeDuration = Math.round(fps * 0.7);

  const opacity = interpolate(
    frame,
    [0, fadeDuration, durationInFrames - fadeDuration, durationInFrames],
    [0, 1, 1, 0],
    { extrapolateLeft: "clamp", extrapolateRight: "clamp" }
  );

  const left = (PANEL_W - PHONE_W) / 2;
  const top = (PANEL_H - PHONE_H) / 2;

  return (
    <div style={{ width: CANVAS_W, height: CANVAS_H, background: BG, opacity, position: "relative" }}>
      <div style={{ position: "absolute", top: PAD, left: PAD, width: PANEL_W, height: PANEL_H }}>
        <div style={{ position: "absolute", left, top, width: PHONE_W, height: PHONE_H }}>
          {/* Cuerpo */}
          <div style={{
            position: "absolute", inset: 0, borderRadius: RADIUS, background: "#0a0a0a",
            boxShadow: "0 0 0 1px #2a2a2a, 0 40px 120px rgba(0,0,0,0.9), 0 10px 40px rgba(0,0,0,0.7)",
          }} />
          {/* Botones */}
          <div style={{ position: "absolute", left: -4, top: 140, width: 4, height: 36, background: "#222", borderRadius: "2px 0 0 2px" }} />
          <div style={{ position: "absolute", left: -4, top: 190, width: 4, height: 36, background: "#222", borderRadius: "2px 0 0 2px" }} />
          <div style={{ position: "absolute", right: -4, top: 160, width: 4, height: 56, background: "#222", borderRadius: "0 2px 2px 0" }} />
          {/* Pantalla */}
          <div style={{
            position: "absolute", top: BORDER + 10, left: BORDER,
            width: SCREEN_W, height: SCREEN_H,
            borderRadius: RADIUS - BORDER - 2, overflow: "hidden", background: "#000",
          }}>
            <Video
              src={staticFile(videoFile)}
              style={{ width: SCREEN_W, height: SCREEN_H, display: "block", objectFit: "cover", objectPosition: "top left" }}
            />
            <div style={{
              position: "absolute", top: 0, left: 0, right: 0, height: 52,
              background: "linear-gradient(to bottom, rgba(0,0,0,0.4) 0%, transparent 100%)",
              pointerEvents: "none",
            }} />
          </div>
          {/* Home indicator */}
          <div style={{
            position: "absolute", bottom: 10, left: "50%", transform: "translateX(-50%)",
            width: INDICATOR_W, height: INDICATOR_H,
            background: "rgba(255,255,255,0.35)", borderRadius: INDICATOR_H / 2,
          }} />
        </div>
      </div>
    </div>
  );
};
```

---

## Paso 7 — Generar compositions y Root.tsx

### Composition por página (src/compositions/{Slug}.tsx)

Para páginas estándar:
```tsx
import React from "react";
import { PageVideo } from "../components/PageVideo";
export const {Slug}: React.FC = () => <PageVideo videoFile="{slug}.mp4" />;
```

Para páginas responsive:
```tsx
import React from "react";
import { PhoneFrame } from "../components/PhoneFrame";
export const {Slug}: React.FC = () => <PhoneFrame videoFile="{slug}.mp4" />;
```

### Root.tsx

```tsx
import React from "react";
import { Composition } from "remotion";
// GENERADO: imports de cada composition

const FPS = 60;
const f = (s: number) => s * FPS;
const base = { width: 1080, height: 1080, fps: FPS };

export const RemotionRoot: React.FC = () => (
  <>
    {/* GENERADO: una Composition por página */}
    <Composition id="{slug}" component={{Slug}} {...base} durationInFrames={f({scrollDuration})} />
  </>
);
```

---

## Paso 8 — Instalar dependencias y verificar que Remotion compila

```bash
cd "{proyecto}-videos"
npm install
# Verificar que compila (timeout 12s):
timeout 12 npx remotion studio src/index.ts || true
# Debe mostrar "Built in Xms" sin errores
```

---

## Paso 9 — Grabar los videos

```bash
npm run record
```

El script graba todas las páginas secuencialmente.
Requiere que la web esté accesible (servidor local o URL pública).
Requiere `ffmpeg` instalado (`brew install ffmpeg` en Mac).

Si la URL usa HTTPS local con certificado autofirmado, Puppeteer lo maneja con `--ignore-certificate-errors`.

---

## Paso 10 — Confirmar y guiar al usuario

Una vez grabados todos los videos, indica al usuario:

1. **Preview**: `npm run dev` → abre http://localhost:3000
2. **Ajustar velocidad**: edita `scrollDuration` en `scripts/record.ts` y vuelve a grabar solo esa página: `npm run record -- {slug}`
3. **Ajustar fondo/estilo**: edita `BG` en `PageVideo.tsx` o `PhoneFrame.tsx`
4. **Renderizar**: `npm run render:all` (o añade el script manualmente si no lo generaste)

---

## Notas técnicas importantes

- **Lenis / smooth scroll**: el script lo desactiva automáticamente para controlar el scroll frame a frame
- **GSAP ScrollTrigger**: se llama `ScrollTrigger.update()` en cada frame para que las animaciones respondan correctamente al scroll position
- **Lazy images**: el script hace scroll hasta el fondo antes de grabar para precargar todo
- **FPS**: siempre 60fps. No bajar de 30fps
- **CRF 16**: calidad alta. Subir a 20-22 si los archivos son demasiado grandes
- **Dimensiones**: siempre pares (divisibles por 2) para compatibilidad con h264/yuv420p
- **`durationInFrames` en Root.tsx** debe coincidir con `scrollDuration × FPS` de record.ts — si no coinciden el video se corta o queda en negro
