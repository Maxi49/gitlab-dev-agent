# FREIGHT-SAVER

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
