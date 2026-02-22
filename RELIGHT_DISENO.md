# RELIGHT (Reiluminación IA) – Documento de diseño

## 1) Objetivo

Construir una herramienta de **reiluminación** que, a partir de **una imagen de referencia**, genere una versión donde **solo cambie la iluminación**:

- Dirección
- Color
- Intensidad
- Sombras asociadas

Opcionalmente, se podrá aplicar un **contexto/grade** mediante presets.

> **Regla de oro:** mantener extremadamente consistente todo lo demás: identidad, geometría, composición, pose, encuadre, fondo, objetos, texturas y nitidez.

---

## 2) Componentes de interfaz (UI)

### 2.1 Esfera de luces (control principal)

- Una **esfera/globo** con rejilla (estética galaxia) representa el espacio alrededor del sujeto.
- En el centro hay un **cubo** y la **imagen de referencia** se renderiza en su cara frontal.
- Las luces se representan como **naves espaciales** (íconos) ubicadas sobre/alrededor de la esfera.
- Cada nave tiene una **animación suave de flotación** decorativa que **no altera** la posición real seleccionada.

### 2.2 Selector de contexto (presets)

Botón cuadrado blanco que abre un selector de presets.

Cada preset contiene:

- `nombre` (ej. “Day”, “Night Neon”, “Golden Hour”)
- `prompt` guardado

Acciones:

- Seleccionar preset
- Crear/editar/eliminar presets (opcional)

### 2.3 Gestión de luces (1–3)

- Mínimo **1** luz
- Máximo **3** luces

Cada luz (nave) tiene:

- Toggle **ON/OFF**
- **Color picker**
- **Slider de intensidad** (0–100)
- **Posición** (drag en la esfera)
- Botón **“Fijar luz”** (inyecta/congela parámetros al prompt)

### 2.4 Panel de configuración por luz (hover + tuerca)

- Al pasar el mouse sobre una nave:
  - Si está **ON**, aparece botón de **tuerca (⚙︎)**
  - Al hacer click, despliega configuración:
    - ON/OFF
    - Color
    - Intensidad
    - “Fijar luz”

Debe existir un botón global **“Minimizar configuración”** para colapsar paneles.

### 2.5 Botón de imagen de referencia

Permite:

- Elegir desde historial de generaciones
- O **subir** imagen

Al seleccionar, la imagen se muestra en el cubo central.

### 2.6 Botón “Generar”

Debe estar **desactivado** si:

- No hay imagen de referencia
- Hay 0 luces activas
- Existe alguna luz en **ON** que **no está fijada**
- Alguna luz quedó en estado **dirty** tras haberse fijado

---

## 3) Interacciones clave

### 3.1 Drag de posición en esfera

- Usuario hace click sostenido sobre la nave y la mueve.
- Puede ubicarse en cualquier punto de la esfera (incluido “detrás”, si se habilita).
- La coordenada se mapea a dirección 3D `(x, y, z)` para el descriptor del prompt.

### 3.2 “Fijar luz” (inyección controlada)

Al fijar, se toma un snapshot de:

- Posición (yaw/pitch + descriptores)
- Color (`hex`)
- Intensidad (`0..1`)
- Rol (`key/fill/rim`)

Ese snapshot se añade al prompt final.

Si luego el usuario modifica posición, color, intensidad o toggle:

- La luz pasa a **dirty**
- Requiere volver a fijarse

### 3.3 Reglas para roles con múltiples luces

- 1 luz ON → **Key**
- 2 luces ON → mayor intensidad = **Key**, otra = **Fill**
- 3 luces ON → mayor = **Key**, segunda = **Fill**, tercera = **Rim/Back**
  - Si tercera está detrás (`z < 0`), reforzar “back/rim” en prompt

---

## 4) Modelo de datos (state)

### 4.1 Presets

```json
Preset {
  id: string,
  name: string,
  prompt: string,
  createdAt: number,
  updatedAt: number
}
```

### 4.2 Luz

```json
Light {
  id: string,
  enabled: boolean,
  colorHex: string,
  intensity: number,
  position: { x: number, y: number, z: number },

  fixed: boolean,
  fixedSnapshot?: {
    role: "key"|"fill"|"rim",
    colorHex: string,
    intensity01: number,
    yawDeg: number,
    pitchDeg: number,
    descriptors: {
      side: "left"|"right"|"center",
      height: "above"|"level"|"below",
      depth: "front"|"back"|"side",
      naturalText: string
    }
  },

  dirty: boolean
}
```

### 4.3 Estado global

```json
RelightState {
  referenceImage: {
    id?: string,
    url?: string,
    source: "upload"|"history"
  } | null,

  presetId: string | null,
  basePromptVersion: string,

  lights: Light[],
  ui: {
    activeLightId?: string,
    anyConfigOpen: boolean,
    minimized: boolean
  }
}
```

---

## 5) Mapeo de posición (esfera → lenguaje)

### 5.1 Sistema de coordenadas

- Centro del cubo = `(0,0,0)`
- Cámara mirando hacia el cubo por eje `+Z`
- Frente del cubo: cara visible con normal a `+Z`
- Luz: vector dirección normalizado `v=(x,y,z)` desde cubo a nave

### 5.2 Conversión a ángulos

- `yaw = atan2(x, z)`
- `pitch = asin(y)`
- Convertir a grados

### 5.3 Descriptores para prompt

- **Side**
  - `yaw > +20°` → “from camera-right”
  - `yaw < -20°` → “from camera-left”
  - else → “near frontal”
- **Height**
  - `pitch > +15°` → “from above”
  - `pitch < -15°` → “from below”
  - else → “at eye level”
- **Depth**
  - `z > +0.2` → “front/near-camera side”
  - `z < -0.2` → “from behind/backlight”
  - else → “side wrap”

### 5.4 Texto natural sugerido

> “Key light coming from above camera-right (~35°) slightly in front, creating directional shadows to camera-left.”

---

## 6) Intensidad y color (UI → prompt)

### 6.1 Intensidad

Transformación:

- `intensity01 = clamp(intensity/100, 0, 1)`

Etiquetas:

- 0–25: subtle
- 26–55: moderate
- 56–80: strong
- 81–100: very strong (evitar clipping)

Inyectar en prompt:

- Descriptor textual (subtle/moderate/strong)
- Valor numérico (`intensity: 0.62/1.0`)

### 6.2 Color

Fuente de verdad: `#RRGGBB`.

Incluir en prompt:

- “light color: #RRGGBB”
- Descriptor opcional (ej. warm amber, cool blue gel)

---

## 7) Prompting: arquitectura de prompts

> Recomendación: prompts internos en inglés para consistencia entre modelos; UI en español.

### 7.1 Prompt base (siempre presente)

```txt
You are a professional photo lighting retoucher.
Task: relight the provided reference image while keeping everything else identical.
Do NOT change: identity, face, body shape, pose, expression, clothing, hair, background objects, camera angle, framing, perspective, lens look, texture fidelity.
Do NOT add/remove objects, text, logos, props, or effects.
Only modify: lighting direction, light color, intensity, shadows, highlights, and subtle global color grading consistent with the selected preset.
Keep realism and physically plausible shadows. Preserve fine details and sharpness.
```

**Negative prompt (si aplica):**

```txt
no geometry changes, no style transfer, no painterly look, no extra objects,
no face change, no smoothing, no artifacts.
```

### 7.2 Prompt de contexto (preset)

Se concatena el bloque guardado del preset.

Ejemplos:

- **Day:**
  `Bright natural daylight, neutral white balance, soft sky fill, realistic outdoor-like contrast, clean highlights, gentle shadow softness.`
- **Night:**
  `Low-key nighttime lighting, deeper shadows, cooler ambient, controlled highlights, cinematic contrast, minimal noise.`
- **Neon Magenta/Blue:**
  `Cyberpunk neon ambience, magenta and blue split, glossy speculars, strong color separation, still photoreal.`

### 7.3 Bloque por luz (solo FIXED + ON)

```txt
Light {n} ({role}):
direction {naturalText} (yaw {yawDeg}°, pitch {pitchDeg}°, vector {x,y,z}).
Color {colorHex}.
Intensity {label} (0–1: {intensity01}).
Ensure shadows and highlights match this direction; keep exposure natural and avoid clipping.
```

### 7.4 Prompt final (builder)

Orden de concatenación:

1. BASE_PROMPT
2. PRESET_PROMPT
3. LIGHT_BLOCK (1..3)
4. Cierre: “Return a single relit image. Preserve all details.”

---

## 8) Reglas de habilitación del botón “Generar”

```ts
GenerateEnabled =
  referenceImage != null &&
  lights.filter(l => l.enabled).length >= 1 &&
  lights.filter(l => l.enabled).every(l => l.fixed && !l.dirty)
```

**Dirty rules**

- Si se cambia posición/color/intensidad tras fijar:
  - `dirty = true`
  - invalidar snapshot (`fixed = false` o snapshot invalidado)
  - deshabilitar “Generar” hasta refijar
- Si una luz está OFF, no requiere fijación

---

## 9) UX sugerido (flujo)

1. Usuario sube/selecciona imagen → aparece en el cubo.
2. Existe 1 luz por defecto (ON, intensidad media, color neutro).
3. Usuario arrastra nave en esfera.
4. Ajusta color/intensidad en panel ⚙︎.
5. Click “Fijar luz”.
6. (Opcional) agrega luz 2 y/o 3 y repite.
7. Selecciona preset.
8. Si todas las luces ON están fijadas y limpias → “Generar” habilitado.
9. Generación, visualización y guardado en historial.

---

## 10) Detalles visuales (tema galaxia)

- Fondo con gradientes oscuros y estrellas sutiles.
- Esfera con líneas blancas finas.
- Naves con glow del color de su luz.
- Flotación decorativa:
  - desplazamiento leve (±2–4px)
  - rotación leve (±1–2°)
  - sin alterar coordenadas reales

---

## 11) Pseudocódigo del Prompt Builder

```ts
function canGenerate(state: RelightState) {
  if (!state.referenceImage) return false;
  const onLights = state.lights.filter(l => l.enabled);
  if (onLights.length < 1) return false;
  return onLights.every(l => l.fixed && !l.dirty && !!l.fixedSnapshot);
}

function buildPrompt(state: RelightState) {
  const preset = getPreset(state.presetId);
  const onFixed = state.lights
    .filter(l => l.enabled && l.fixed && !l.dirty && l.fixedSnapshot)
    .slice(0, 3);

  const base = BASE_PROMPT;
  const presetBlock = preset ? `\n\n[CONTEXT PRESET]\n${preset.prompt}` : "";

  const lightsBlock = onFixed.map((l, idx) => {
    const s = l.fixedSnapshot!;
    return `\n\n[LIGHT ${idx + 1}]\n` +
      `Role: ${s.role}\n` +
      `Direction: ${s.descriptors.naturalText} (yaw ${s.yawDeg.toFixed(1)}°, pitch ${s.pitchDeg.toFixed(1)}°)\n` +
      `Color: ${s.colorHex}\n` +
      `Intensity: ${s.intensity01.toFixed(2)} / 1.0\n` +
      `Keep everything else identical; only lighting/shadows/highlights and subtle grading.`;
  }).join("");

  return `${base}${presetBlock}${lightsBlock}`.trim();
}

function fixLight(light: Light, state: RelightState) {
  const { yawDeg, pitchDeg, descriptors } = describePosition(light.position);
  const intensity01 = clamp(light.intensity / 100, 0, 1);

  const role = computeRole(state.lights);

  light.fixed = true;
  light.dirty = false;
  light.fixedSnapshot = {
    role,
    colorHex: light.colorHex,
    intensity01,
    yawDeg,
    pitchDeg,
    descriptors
  };
}
```

---

## 12) Checklist de implementación

- [ ] Vista esfera + cubo central + carga de imagen
- [ ] 1 luz por defecto (ON)
- [ ] Añadir/quitar luces hasta 3
- [ ] Drag en esfera → cálculo `(x,y,z)`
- [ ] Hover nave → mostrar ⚙︎
- [ ] Panel por luz: ON/OFF, color, intensidad, “Fijar luz”
- [ ] Estado dirty/fixed
- [ ] Selector de presets (CRUD opcional)
- [ ] Builder de prompt + validación
- [ ] Botón Generar con reglas de habilitación
- [ ] Estética galaxia + flotación decorativa

---

## 13) Textos UI sugeridos

- Preset: nombre del preset (ej. “Day”)
- Subida: “Seleccionar imagen”
- Añadir luz: “+ Añadir luz”
- Fijar: “Fijar luz”
- Estado fijo: “Fijada ✓”
- Estado dirty: “Cambios sin fijar”
- Global: “Minimizar configuración”
- Acción final: “Generar”
