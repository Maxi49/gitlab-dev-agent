 # SnapShop vs Freight-Saver — Visión Conceptual
> Google Cloud Rapid Agent Hackathon · Comparativa de ideas

---

# IDEA 1 — SNAPSHOP

## ¿Qué es?

SnapShop es una plataforma de dos caras que conecta comercios chicos con sus clientes locales usando inteligencia artificial. Es el PedidosYa de los almacenes de barrio — pero donde PedidosYa tardó años y millones en construir su red de comercios, SnapShop lo hace en una foto y una conversación.

Es dos apps en una: una para el vendedor que quiere existir online, y una para el cliente que quiere encontrar lo que necesita cerca suyo.

---

## ¿Qué problema resuelve?

**Del lado del vendedor:**
El 90% de los comercios chicos del mundo — almacenes, kioscos, ferreterías, verdulerías — no tienen presencia online. No es porque no quieran. Es porque crear una tienda en Tiendanube o Shopify requiere fotografiar cada producto individualmente, escribir nombres y descripciones uno por uno, configurar categorías, métodos de pago y dominio, y dedicar días o semanas de trabajo. Para un almacenero que trabaja solo 12 horas por día, eso es imposible.

**Del lado del cliente:**
El cliente que necesita sal a las 11pm, o que busca un repuesto específico, no tiene forma de saber qué comercio cercano lo tiene sin salir a caminar de puerta en puerta. Los buscadores generales como Google Maps no tienen información de stock en tiempo real de comercios chicos.

**El resultado:** millones de comercios invisibles para sus propios clientes a tres cuadras de distancia. Dinero perdido de ambos lados.

---

## ¿Qué hace?

### Para el vendedor:
El vendedor abre SnapShop, tiene una conversación corta con un agente de IA (nombre del negocio, dirección, tipo de productos), saca **una sola foto** a su estantería, y en menos de 5 minutos tiene una tienda online publicada y funcional. No necesita saber nada de tecnología. No necesita pagar hosting. No necesita configurar nada. La tienda vive en la infraestructura de SnapShop para siempre.

Después puede ajustar precios, agregar información, cambiar colores — todo conversando con el agente en lenguaje natural, como si le mandara un mensaje de WhatsApp.

### Para el cliente:
El cliente abre SnapShop y busca lo que necesita de dos formas:

- **Por texto:** escribe "azúcar y fideos" y el sistema le muestra qué almacenes cercanos los tienen, con precio y distancia.
- **Por foto:** saca una foto del producto que necesita (aunque no sepa el nombre exacto, aunque la foto sea borrosa) y el sistema encuentra tiendas cercanas que tienen ese producto por similitud visual.

En ambos casos recibe la lista de tiendas ordenadas por cercanía, con precios, y un botón para ir directo al mapa.

---

## ¿Cómo lo hace? (sin tecnicismos)

El agente de IA analiza la foto de la estantería y "ve" los productos — entiende qué es cada cosa, en qué categoría entra, y si puede leer el precio. Con esa información construye automáticamente el catálogo de la tienda.

Lo realmente interesante es lo que pasa con las búsquedas del cliente. Cuando el vendedor carga sus productos, el sistema no solo guarda el nombre — guarda una "huella digital" de cada producto (llamada embedding) que captura su significado visual y semántico. Cuando un cliente busca "sal" o saca una foto de un paquete de sal, el sistema compara esa huella con todas las huellas guardadas y encuentra los productos más similares en tiendas cercanas. Es lo mismo que hace Google Photos cuando te agrupa fotos por persona — pero aplicado a productos en almacenes de barrio.

---

## ¿Por qué es diferente a todo lo que existe?

| | Tiendanube / Shopify | Google Maps | SnapShop |
|---|---|---|---|
| Tiempo para tener tienda | 1-2 semanas | No aplica | ~5 minutos |
| Carga de productos | Manual, uno por uno | No aplica | Una foto |
| Buscar por foto | No | No | Sí |
| Buscar en múltiples tiendas | No | Limitado | Sí |
| Información de stock real | No | No | Sí |
| Perfil del vendedor requerido | Conocimiento técnico | — | Cualquier persona |

---

## ¿Para quién es?

**Vendedores:** cualquier persona que tenga un comercio físico y quiera que sus clientes lo encuentren online. El almacenero de 60 años. La kiosquera que no sabe qué es Shopify. El ferretero que no tiene tiempo para cargar 500 productos.

**Clientes:** cualquier persona que quiera saber dónde encontrar algo cerca suyo sin perder tiempo. Especialmente útil en situaciones de urgencia (necesito algo ahora) o cuando no se sabe el nombre exacto de lo que se busca.

---

## ¿Por qué importa?

El comercio de proximidad es la columna vertebral económica de miles de ciudades en Latinoamérica y el mundo. Cada vez que un vecino no encuentra lo que busca en su barrio y lo compra en un supermercado de cadena o en e-commerce, un comercio chico pierde. SnapShop no compite con el e-commerce — conecta a la gente con lo que ya existe a 200 metros de su casa.

Es tecnología de punta al servicio de la economía más simple y humana que existe: el almacén de la esquina.

---

## El pitch en una oración

> *"SnapShop le da al almacenero de la esquina lo que Shopify solo le da a quien tiene tiempo, dinero y conocimiento técnico — en una foto y cinco minutos."*

---
---

# IDEA 2 — FREIGHT-SAVER

## ¿Qué es?

Freight-Saver es un agente autónomo de rescate logístico. Cuando un camión de carga crítica sufre una falla mecánica o de refrigeración en ruta, el agente detecta la crisis, evalúa todas las opciones disponibles en la flota, toma la decisión óptima de rescate, y ejecuta las acciones necesarias — en menos de 60 segundos, sin intervención humana inicial.

No es un sistema de alertas. No es un dashboard. Es un agente que actúa.

---

## ¿Qué problema resuelve?

Cada día, camiones que transportan carga crítica — medicamentos, vacunas, insulina, alimentos perecederos — sufren fallas en ruta. Cuando eso pasa, el proceso de rescate hoy es completamente manual:

1. El conductor llama a la central por teléfono
2. El despachador busca frenéticamente en planillas qué camión está cerca
3. Llama por radio a ese camión, negocia el desvío
4. Avisa manualmente al cliente final
5. Actualiza el sistema uno por uno

**Todo esto toma entre 2 y 4 horas.**

En ese tiempo, una partida de insulina se echa a perder. Una carga de vacunas para una comunidad rural se pierde. Un hospital no recibe lo que necesitaba.

El problema no es la falla del camión. El problema es que la respuesta humana es demasiado lenta para la urgencia de la situación.

---

## ¿Qué hace?

Freight-Saver monitorea la flota en tiempo real. Cuando detecta una crisis (falla de motor, temperatura fuera de rango, camión detenido), actúa en cuatro pasos simultáneos:

**1. Entiende qué hay en juego:** consulta la base de datos operativa para saber exactamente qué lleva el camión averiado, qué temperatura requiere la carga, cuánto tiempo queda antes de que se eche a perder, y qué penalidad contractual hay por no entregar a tiempo.

**2. Busca el rescate óptimo:** analiza toda la flota activa en tiempo real — posición GPS, espacio disponible, capacidad de refrigeración, distancia al incidente — y encuentra los candidatos que pueden llegar a tiempo.

**3. Decide:** evalúa los candidatos contra las restricciones del problema (temperatura, volumen, tiempo crítico) y elige el rescate óptimo. Si hay empate, reserva un backup automáticamente.

**4. Ejecuta en paralelo:** reasigna la carga en el sistema, actualiza la ruta del camión de rescate, notifica al cliente afectado con un mensaje claro ("su entrega se retrasará 35 minutos, la cadena de frío está garantizada"), y le avisa al despachador humano que todo está resuelto — por si quiere revisar o revertir.

El despachador humano no desaparece. Recibe un resumen completo de lo que pasó y por qué el agente tomó cada decisión. Siempre puede intervenir. Pero en el 90% de los casos, no necesita hacer nada.

---

## ¿Cómo lo hace? (sin tecnicismos)

El agente tiene acceso a dos fuentes de información complementarias:

**La base de datos operativa (MongoDB):** es la fuente de verdad. Sabe qué lleva cada camión, quién es el cliente, cuáles son los SLAs y penalidades, y cuál es el historial de cada envío. Cuando el agente necesita entender qué hay en juego, va acá.

**El motor de búsqueda en tiempo real (Elastic):** es el cerebro de la decisión. Tiene indexados todos los camiones de la flota con su posición GPS actualizada cada 30 segundos, su temperatura actual, su espacio disponible y sus capacidades. Cuando el agente necesita encontrar el rescate óptimo, hace una búsqueda geoespacial multi-criterio acá — combina distancia, temperatura y capacidad en un solo score de relevancia, como Google combina decenas de factores para rankear resultados.

Los dos son necesarios. Sin MongoDB, el agente no sabe qué hay en juego ni puede actualizar el sistema. Sin Elastic, no puede encontrar el rescate óptimo en tiempo real. Ninguno reemplaza al otro.

---

## ¿Por qué es diferente a todo lo que existe?

| | GPS / Waze | Software logístico tradicional | Freight-Saver |
|---|---|---|---|
| Detecta fallas | ❌ | ⚠️ Alerta, no actúa | ✅ Detecta y actúa |
| Busca camión de rescate | ❌ | Manual | ✅ Automático |
| Considera refrigeración + capacidad + distancia | ❌ | Manual | ✅ En una query |
| Reasigna carga en el sistema | ❌ | Manual | ✅ Automático |
| Notifica al cliente | ❌ | Manual | ✅ Automático |
| Tiempo de respuesta | — | 2-4 horas | < 60 segundos |

El software logístico tradicional avisa que algo salió mal. Freight-Saver resuelve el problema.

---

## ¿Para quién es?

**Empresas de transporte de carga crítica:** especialmente las que manejan cadena de frío — farmacéuticas, distribuidoras de alimentos perecederos, operadores logísticos que trabajan con hospitales y clínicas.

**Despachadores:** que hoy pasan su día apagando incendios manualmente. Freight-Saver les devuelve tiempo y les elimina la ansiedad de una crisis nocturna.

**Clientes finales** (hospitales, supermercados, distribuidoras): que necesitan certeza de entrega más que excusas.

---

## ¿Por qué importa?

Las fallas logísticas en cadena de frío tienen consecuencias reales: medicamentos que no llegan a pacientes que los necesitan, alimentos que se pierden cuando hay escasez, penalidades contractuales millonarias. El costo promedio de un incidente de cadena de frío no resuelto a tiempo supera los $50.000 USD entre pérdida de mercadería, penalidades y daño a la relación con el cliente.

Freight-Saver no elimina las fallas. Elimina el tiempo de respuesta humana que convierte una falla manejable en una catástrofe.

---

## El pitch en una oración

> *"450 unidades de insulina. Un hospital que las esperaba. Un camión varado en la Ruta 9. Nuestro agente resolvió el rescate completo en 52 segundos, sin que ningún humano tuviera que hacer nada. Eso es lo que significa que la IA tome acción."*
