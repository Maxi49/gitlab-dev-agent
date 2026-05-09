# SnapShop — Architecture & Technical Specification
> Google Cloud Rapid Agent Hackathon · MongoDB Track (+ Arize secondary)

---

## 1. VISION

**SnapShop** es un agente de IA que permite a cualquier comercio chico del mundo tener una tienda online en minutos, sin conocimiento técnico, sin cargar productos uno por uno, y sin fricción.

El flujo completo es una sola acción: **sacar una foto a la estantería**.

El agente hace el resto — detecta los productos, construye la tienda, y la publica. El vendedor solo customiza conversacionalmente lo que quiera cambiar. El cliente busca lo que necesita, encuentra el comercio más cercano que lo tiene, y recibe la ruta para llegar.

**El problema real:**
El 90% de los comercios chicos del mundo no tienen presencia online. No es porque no quieran — es porque crear una tienda en Tiendanube, Shopify, o cualquier plataforma requiere cargar productos uno por uno, escribir descripciones, subir fotos, configurar pagos, y dedicar días de trabajo. Para un almacenero, kiosquero, o ferretero, eso es imposible.

**El diferencial:**
Ninguna plataforma existente — ni Tiendanube, ni Shopify, ni MercadoShops — hace lo que SnapShop hace: foto → tienda lista en minutos, sin fricción, impulsada por un agente que razona y actúa.

---

## 2. OVERVIEW DE ALTO NIVEL

```
┌─────────────────────────────────────────────────────────────┐
│                    VENDEDOR                                 │
│  Abre SnapShop, hace onboarding conversacional con          │
│  el agente, saca foto a su estantería                       │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│              SNAPSHOP AGENT (Python ADK)                    │
│                                                             │
│  Gemini 2.0 Flash (texto + visión)                         │
│                                                             │
│  Tools via MongoDB MCP:                                     │
│    - create_store()          - update_store_config()        │
│    - upsert_product()        - search_products()            │
│    - get_store()             - vector_search()              │
│                                                             │
│  Tools via Arize MCP:                                       │
│    - log_detection_trace()   - get_accuracy_metrics()       │
│    - flag_low_confidence()                                  │
└─────────┬───────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────┐       ┌──────────────────────────────┐
│   MONGODB ATLAS     │       │   ARIZE PHOENIX              │
│                     │       │                              │
│   stores/           │       │   Trazas de detección        │
│   products/         │       │   Métricas de accuracy       │
│   embeddings        │       │   Alertas de baja confianza  │
│   (Vector Search)   │       │                              │
└─────────────────────┘       └──────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│              SVELTEKIT FRONTEND (Vercel)                    │
│                                                             │
│  Vista Vendedor:                                            │
│    - Onboarding + chat con el agente                        │
│    - Preview de la tienda generada                          │
│    - Customización conversacional                           │
│                                                             │
│  Vista Cliente:                                             │
│    - Búsqueda de productos en lenguaje natural              │
│    - Lista de tiendas cercanas con el producto              │
│    - Ruta para llegar (Google Maps embed)                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. FLUJOS PRINCIPALES

### 3.1 Flujo del Vendedor — Crear tienda

```
PASO 1 — ONBOARDING CONVERSACIONAL
  Agente: "Hola! Vamos a crear tu tienda. ¿Cómo se llama tu negocio?"
  Vendedor: "Almacén Don Pepe"
  Agente: "¿En qué dirección estás?"
  Vendedor: "Rivadavia 450, Río Cuarto"
  Agente: "¿Qué tipo de productos vendés principalmente?"
  Vendedor: "Almacén, fiambrería"
  → Agente crea el documento base de la tienda en MongoDB

PASO 2 — FOTO A LA ESTANTERÍA
  Vendedor sube foto(s) de su estantería
  → Gemini Vision analiza la imagen
  → Detecta productos: nombre, precio (si está visible), categoría
  → Genera embeddings para vector search
  → Upserta cada producto en MongoDB
  → Arize loguea la traza de detección con confidence scores

PASO 3 — PREVIEW Y CONFIRMACIÓN
  Agente muestra la tienda generada:
  "Detecté 23 productos. Tu tienda está lista.
   Aquí está el preview: [link]
   ¿Querés cambiar algo?"

PASO 4 — CUSTOMIZACIÓN CONVERSACIONAL
  Vendedor: "Cambiá el color a verde"
  → Agente actualiza store_config en MongoDB

  Vendedor: "El aceite Cocinero no cuesta $1500, cuesta $1800"
  → Agente actualiza el producto específico

  Vendedor: "Agregá que hacemos envíos los martes y jueves"
  → Agente actualiza la descripción de la tienda

PASO 5 — PUBLICACIÓN
  Agente: "Tu tienda está publicada en snapshop.app/don-pepe
           Compartila con tus clientes!"
```

### 3.2 Flujo del Cliente — Encontrar producto

```
PASO 1 — BÚSQUEDA EN LENGUAJE NATURAL
  Cliente escribe: "necesito azúcar y fideos, estoy en el centro"

PASO 2 — RAZONAMIENTO DEL AGENTE
  Agente procesa la consulta:
  - Identifica productos buscados: ["azúcar", "fideos"]
  - Identifica ubicación: "centro de Río Cuarto"
  - Ejecuta vector search en MongoDB por cada producto
  - Filtra tiendas por proximidad geográfica
  - Rankea por: distancia + cantidad de productos disponibles

PASO 3 — RESULTADO
  Agente devuelve:
  "Encontré 3 tiendas cerca tuyo que tienen azúcar y fideos:
   
   1. Almacén Don Pepe — 200m
      ✅ Azúcar Ledesma 1kg — $850
      ✅ Fideos Matarazzo — $620
      
   2. Kiosco El Rápido — 350m
      ✅ Azúcar — $900
      ❌ Fideos (no disponible)
   
   [Ver ruta a Don Pepe →]"
```

---

## 4. COMPORTAMIENTO DEL AGENTE EN DETALLE

### 4.1 Detección de productos (Gemini Vision)

El agente usa Gemini 2.0 Flash con capacidades de visión para analizar cada foto. El prompt de detección está diseñado para extraer:

```json
{
  "products": [
    {
      "name": "Aceite Cocinero 900ml",
      "category": "aceites",
      "price": 1500,
      "price_visible": true,
      "confidence": 0.94,
      "quantity_estimate": "varios"
    },
    {
      "name": "Yerba Playadito 500g",
      "category": "yerba/mate",
      "price": null,
      "price_visible": false,
      "confidence": 0.88,
      "quantity_estimate": "varios"
    }
  ],
  "image_quality": "good",
  "products_detected": 23,
  "low_confidence_count": 2
}
```

**Manejo de casos borde:**
- Producto con baja confianza (<0.7): se agrega con flag `needs_review`, el agente lo menciona al vendedor
- Precio no visible: producto se crea sin precio, agente pregunta al vendedor
- Foto de mala calidad: agente pide otra foto antes de continuar
- Producto no identificable: se ignora, se loguea en Arize como `undetected`

### 4.2 Customización conversacional

El agente interpreta comandos en lenguaje natural y los traduce a operaciones sobre MongoDB:

| Comando del vendedor | Acción del agente |
|---------------------|-------------------|
| "El aceite cuesta $1800" | `update_product(name="aceite", price=1800)` |
| "Cambiá el color a verde" | `update_store_config(primary_color="#00C851")` |
| "Agregá que abrimos de 8 a 20" | `update_store(hours="8:00-20:00")` |
| "Ocultá los productos sin precio" | `filter_products(price_visible=true)` |
| "Agregá una foto del local" | `update_store(banner_image=<upload>)` |
| "Sacamos empanadas los viernes" | `add_product(name="Empanadas", category="comidas", days="viernes")` |

### 4.3 Rol de Arize Phoenix

Arize monitorea la calidad del agente de visión en producción:

- **Por cada detección:** loguea un span con `confidence_score`, `product_name`, `image_quality`
- **Alertas automáticas:** si confidence promedio baja de 0.75 en una sesión, flagea para revisión
- **Dashboard de accuracy:** cuántos productos detectados vs confirmados por el vendedor
- **Dataset de evaluación:** acumula pares imagen→detección para mejorar el prompt

Esto le da al proyecto una capa de **self-improvement** — el agente puede consultar sus propias métricas via MCP de Arize y ajustar su comportamiento.

---

## 5. ESTRUCTURA DE DATOS EN MONGODB

### Colección: `stores`
```json
{
  "_id": "store_uuid",
  "slug": "don-pepe",
  "name": "Almacén Don Pepe",
  "owner_id": "user_uuid",
  "address": {
    "street": "Rivadavia 450",
    "city": "Río Cuarto",
    "province": "Córdoba",
    "country": "Argentina",
    "coordinates": [-33.1232, -64.3493]
  },
  "category": "almacén",
  "config": {
    "primary_color": "#2E7D32",
    "logo_url": null,
    "banner_url": null,
    "hours": "8:00-20:00",
    "delivery": false,
    "whatsapp": "+5493584123456"
  },
  "description": "Almacén de barrio con fiambrería",
  "created_at": "2026-05-09T18:00:00Z",
  "published": true
}
```

### Colección: `products`
```json
{
  "_id": "product_uuid",
  "store_id": "store_uuid",
  "name": "Aceite Cocinero 900ml",
  "category": "aceites",
  "price": 1800,
  "currency": "ARS",
  "available": true,
  "needs_review": false,
  "detection_confidence": 0.94,
  "image_url": null,
  "embedding": [0.023, -0.145, ...],  // vector para búsqueda semántica
  "created_at": "2026-05-09T18:02:00Z",
  "updated_at": "2026-05-09T18:10:00Z"
}
```

### Vector Search Index
```json
{
  "mappings": {
    "fields": {
      "embedding": {
        "type": "knnVector",
        "dimensions": 1536,
        "similarity": "cosine"
      }
    }
  }
}
```

---

## 6. STACK TÉCNICO

| Capa | Tecnología | Por qué |
|------|-----------|---------|
| Agente | Python ADK | Requerimiento hackathon, nativo en Google Cloud |
| Visión + Texto | Gemini 2.0 Flash | Multimodal, rápido, barato |
| DB Principal | MongoDB Atlas | Track principal, vector search nativo |
| Observabilidad | Arize Phoenix | Track secundario, monitorea calidad de visión |
| Runtime | Agent Engine | Managed, stateful |
| Backend | FastAPI + Cloud Run | Python, serverless |
| Frontend | SvelteKit | Mínimo overhead, TypeScript |
| Hosting FE | Vercel | Un comando para deployar |
| Hosting BE | Cloud Run | Escala a cero, usa créditos de Google |

---

## 7. FRONTEND — TRES VISTAS

### Vista 1 — Landing / Onboarding del Vendedor
```
┌─────────────────────────────────────────────────┐
│  📸 SnapShop                                    │
│  Tu tienda online en una foto                   │
│                                                 │
│  [Crear mi tienda gratis →]                     │
└─────────────────────────────────────────────────┘
```

### Vista 2 — Chat con el Agente (Vendedor)
```
┌─────────────────────────────────────────────────┐
│  SnapShop Agent                                 │
├─────────────────────────────────────────────────┤
│  🤖 ¡Hola! Vamos a crear tu tienda.             │
│     ¿Cómo se llama tu negocio?                  │
│                                                 │
│  👤 Almacén Don Pepe                            │
│                                                 │
│  🤖 Perfecto. ¿En qué dirección estás?          │
│                                                 │
│  👤 Rivadavia 450, Río Cuarto                   │
│                                                 │
│  🤖 Genial. Ahora sacá una foto a tu            │
│     estantería y subila acá 👇                  │
│                                                 │
│  [📷 Subir foto]                               │
│                                                 │
│  ── Analizando imagen... ──                     │
│                                                 │
│  🤖 Detecté 23 productos ✅                     │
│     Tu tienda está lista. ¿Querés ver           │
│     el preview o cambiar algo?                  │
│                                                 │
│  [Ver mi tienda →]                             │
├─────────────────────────────────────────────────┤
│  Escribí algo...                    [Enviar]   │
└─────────────────────────────────────────────────┘
```

### Vista 3 — Tienda pública (Cliente)
```
┌─────────────────────────────────────────────────┐
│  Almacén Don Pepe                 📍 200m       │
│  Rivadavia 450 · Abre 8:00-20:00               │
├─────────────────────────────────────────────────┤
│  🔍 ¿Qué estás buscando?                       │
│  [azúcar, fideos, leche...           ] [Buscar] │
├─────────────────────────────────────────────────┤
│  ACEITES              YERBA/MATE    LÁCTEOS     │
│  ┌──────────┐         ┌──────────┐             │
│  │ Cocinero │         │Playadito │             │
│  │ 900ml    │         │ 500g     │             │
│  │  $1.800  │         │  $2.100  │             │
│  └──────────┘         └──────────┘             │
├─────────────────────────────────────────────────┤
│  [📍 Cómo llegar]  [💬 WhatsApp]               │
└─────────────────────────────────────────────────┘
```

### Vista 4 — Búsqueda global (Cliente sin tienda específica)
```
┌─────────────────────────────────────────────────┐
│  📸 SnapShop                                    │
│  Encontrá lo que necesitás cerca tuyo           │
├─────────────────────────────────────────────────┤
│  [¿Qué necesitás?                    ] [Buscar] │
├─────────────────────────────────────────────────┤
│  Resultados para "azúcar y fideos":             │
│                                                 │
│  📍 Almacén Don Pepe — 200m                    │
│     ✅ Azúcar Ledesma 1kg — $850               │
│     ✅ Fideos Matarazzo — $620                  │
│     [Ver tienda] [Cómo llegar]                 │
│                                                 │
│  📍 Kiosco El Rápido — 350m                    │
│     ✅ Azúcar — $900                           │
│     ❌ Fideos no disponible                     │
│     [Ver tienda] [Cómo llegar]                 │
└─────────────────────────────────────────────────┘
```

---

## 8. CASOS BORDE DEL AGENTE

| Situación | Comportamiento |
|-----------|---------------|
| Foto muy oscura o borrosa | Agente pide otra foto antes de procesar |
| Producto con confianza < 0.7 | Se agrega con flag, agente avisa al vendedor |
| Precio no visible en la imagen | Producto creado sin precio, agente pregunta |
| Texto en otro idioma | Gemini detecta y traduce automáticamente |
| Estantería con >100 productos | Procesa en batches, muestra progreso |
| Cliente busca algo que no existe | Agente sugiere productos similares disponibles |
| Vendedor pide algo imposible | Agente explica limitaciones con alternativas |

---

## 9. PITCH PARA LOS JUECES

> *"El 90% de los comercios chicos del mundo no tienen presencia online. No es porque no quieran — es porque el proceso es demasiado complejo. Tiendanube tarda 1-2 semanas. Shopify requiere conocimiento técnico. Nuestro agente lo hace en una foto y una conversación."*

**Por qué gana en el track de MongoDB:**
- MongoDB Atlas es la base de todo el catálogo — no es un extra, es el core
- Vector Search es lo que hace posible que un cliente busque "sal" y encuentre "sal fina", "sal gruesa", "sal entrefina" en distintas tiendas
- La integración es meaningful, no decorativa

**Por qué Arize agrega valor real:**
- No es un extra para sumar puntos — monitorea activamente la calidad de las detecciones
- El agente puede consultar sus propias métricas y mejorar su comportamiento
- Es el loop de self-improvement que los jueces de Arize valoran

---

## 10. DEFINICIÓN DE DONE — HACKATHON SCOPE

Para el 11 de junio, el proyecto está completo cuando:

- [ ] Agente completa el onboarding conversacional
- [ ] Gemini Vision detecta productos de una foto real
- [ ] Productos guardados correctamente en MongoDB con embeddings
- [ ] Tienda pública generada y accesible por URL
- [ ] Customización conversacional funciona (al menos 3 comandos)
- [ ] Vector search retorna tiendas con el producto buscado
- [ ] Arize loguea trazas de detección con confidence scores
- [ ] Frontend vendedor (chat + preview) deployado en Vercel
- [ ] Frontend cliente (búsqueda + resultado) deployado en Vercel
- [ ] Backend deployado en Cloud Run
- [ ] Repo público con LICENSE
- [ ] Video demo de 3 minutos

**Fuera de scope:**
- Pagos y checkout
- Autenticación robusta (para demo alcanza con user_id simple)
- Actualización de stock en tiempo real
- Notificaciones push
- App nativa (iOS/Android) — web mobile es suficiente

---

## 11. PLAN DE TRABAJO (33 días · tiempo parcial)

### Semana 1 (9-16 mayo) — Fundaciones
- Setup Google Cloud + MongoDB Atlas + Arize Phoenix
- Agente básico en ADK con MongoDB MCP conectado
- Primer test de Gemini Vision detectando productos de una foto

### Semana 2 (16-23 mayo) — Core del agente
- Flujo completo de onboarding → foto → detección → MongoDB
- Vector search funcionando para búsqueda de productos
- Arize logueando trazas

### Semana 3 (23-30 mayo) — Frontend
- SvelteKit: vista del chat con el agente
- Vista de la tienda pública generada
- Vista de búsqueda del cliente

### Semana 4 (30 mayo-6 junio) — Polish + Deploy
- Customización conversacional
- Deploy completo (Cloud Run + Vercel + Agent Engine)
- Testing con casos reales (fotos de estanterías reales)

### Días finales (6-11 junio) — Submission
- Grabación del video demo
- README completo
- Submission en Devpost
