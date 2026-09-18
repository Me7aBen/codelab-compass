# Codelab Compass

Buscador conversacional de codelabs para organizadores de GDG. Respondes seis preguntas y sale una agenda de taller con horarios, codelabs oficiales de Google y exportación a calendario.

Ninguna sesión pasa de **2 horas**. Los bloques son de **45 minutos como máximo**.

- Un solo archivo, sin build, sin dependencias.
- Funciona **sin IA**: el planificador local arma la agenda con las reglas del catálogo.
- Con IA conectada (Gemini), además escribe el objetivo de cada sesión, el checkpoint, las notas para quien facilita y los riesgos de sala.

---

## Subirlo a GitHub Pages

```bash
git init
git add index.html README.md
git commit -m "Codelab Compass"
git branch -M main
git remote add origin https://github.com/TU-USUARIO/codelab-compass.git
git push -u origin main
```

Luego en GitHub: **Settings → Pages → Source: Deploy from a branch → Branch: `main` / `(root)` → Save**.

En un minuto queda en `https://TU-USUARIO.github.io/codelab-compass/`.

Para probarlo antes localmente:

```bash
python3 -m http.server 8000
# http://localhost:8000
```

---

## Conectar Gemini

Botón **⚙︎ IA** arriba a la derecha. Hay tres caminos.

### 1. Sin IA (por defecto)
No configuras nada. La agenda se arma con el planificador local. Sirve para demos sin internet.

### 2. API key directa — solo para demos

1. Saca una key en [aistudio.google.com/apikey](https://aistudio.google.com/apikey).
2. ⚙︎ IA → Proveedor **Gemini API** → pega la key → **Probar conexión** → **Guardar**.

La key se guarda en el `localStorage` de ese navegador. **En un sitio público cualquiera puede leerla**, así que esto es para tu laptop en el stand, no para el link que repartes.

### 3. Backend propio — para producción

Un proxy en Cloud Run guarda la key del lado del servidor. En ⚙︎ IA eliges **Mi backend** y pegas la URL.

El contrato es mínimo: recibe `POST {prompt, lang}` y devuelve `{text}`.

**`index.js`**

```js
import express from "express";
import cors from "cors";
import { GoogleGenAI } from "@google/genai";

const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });
const app = express();

app.use(express.json({ limit: "256kb" }));
app.use(cors({ origin: "https://TU-USUARIO.github.io" }));

app.post("/plan", async (req, res) => {
  const prompt = String(req.body?.prompt || "");
  if (prompt.length < 50 || prompt.length > 30000) {
    return res.status(400).json({ error: "prompt fuera de rango" });
  }
  try {
    const r = await ai.models.generateContent({
      model: process.env.MODEL || "gemini-3.6-flash",
      contents: prompt,
      config: { responseMimeType: "application/json", temperature: 0.75 }
    });
    res.json({ text: r.text });
  } catch (e) {
    res.status(502).json({ error: String(e.message || e) });
  }
});

app.listen(process.env.PORT || 8080);
```

**`package.json`**

```json
{
  "type": "module",
  "dependencies": {
    "@google/genai": "^1.0.0",
    "express": "^4.19.2",
    "cors": "^2.8.5"
  }
}
```

**Desplegar**

```bash
gcloud run deploy codelab-compass-ai \
  --source . \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars MODEL=gemini-3.6-flash \
  --set-secrets GEMINI_API_KEY=gemini-key:latest
```

La key va en Secret Manager, no en una variable de entorno plana:

```bash
echo -n "TU_API_KEY" | gcloud secrets create gemini-key --data-file=-
```

Ajusta el `origin` del CORS a tu dominio de Pages. Sin eso, cualquiera puede usar tu cuota.

### Alternativa: Vertex AI

Si ya estás en Google Cloud con facturación, cambia el constructor por
`new GoogleGenAI({ vertexai: true, project: "...", location: "us-central1" })`
y elimina la key: Cloud Run autentica con la cuenta de servicio.

---

## El catálogo

Está al inicio del `<script>`, en la constante `CATALOG`. 35 codelabs con URL verificada. Cada entrada:

```js
{
  id: "compose-basics",                  // único
  t: { es: "…", en: "…" },               // título
  u: "https://developer.android.com/…",  // URL del codelab
  k: "android",                          // track principal
  x: ["ai"],                             // tracks secundarios (opcional)
  lv: 1,                                 // 1 básico · 2 intermedio · 3 avanzado
  m: 60,                                 // duración real del codelab completo
  tags: ["compose", "kotlin"]            // alimentan el match por texto libre
}
```

Dos cosas honestas sobre el catálogo actual:

- **Está sesgado a Android.** 28 de 35 entradas. Es donde pude verificar más URLs. Web, Flutter y Firebase están flacos.
- **Es estático.** Para producción hay que ingerir el índice vivo de `codelabs.developers.google.com`. Ahí está la diferencia entre demo y producto, y es el mejor argumento del pitch.

Agregar un track nuevo: una entrada en `TRACKS` con su emoji, y una regla de color en el CSS (`.tile[data-k="…"]` y `.tag.…`).

## Duraciones

En `FORMATS`. `m` es el total de la sesión, `brk` la pausa, `gap` los días entre sesiones. `MAX_BLOCK` y `MIN_BLOCK` acotan cada bloque.

Si el codelab dura más que el bloque asignado, la agenda lo marca como **extracto** y la IA indica qué parte correr.

## Exportación

- **`.ics`** — un evento por sesión, con el desglose horario y las URLs en la descripción. Abre en Google Calendar, Outlook y Apple Calendar.
- **Google Calendar** — abre el formulario de evento prellenado, una pestaña por sesión.
- **Markdown** — tabla por sesión, se pega directo en Notion.

Las horas del `.ics` son flotantes (sin zona horaria): el evento cae a la hora local de quien lo importa. Es lo correcto para un taller presencial.

---

Proyecto de comunidad. Sin afiliación con Google. Los codelabs enlazados son propiedad de sus autores.
