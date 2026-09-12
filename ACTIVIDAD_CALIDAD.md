# Actividad de Calidad

## Code review manual

| Archivo | Hallazgo | Tipo | Severidad | Propuesta |
|---------|----------|------|-----------|-----------|
| ParkingFeeCalculator.java | Los valores `150`, `15`, `60`, `20`, `15` (tarifa por hora) y `80` aparecen como literales sin nombre ("números mágicos"), lo que dificulta saber qué representa cada uno y hace más riesgoso modificar una regla de negocio en el futuro. | Maintainability | Media | Extraer cada valor a una constante con nombre descriptivo (`FREE_MINUTES`, `FLAT_FEE`, `HOURLY_FEE`, `MAX_FEE`, `LOST_TICKET_FEE`), de modo que un cambio de tarifa sea una sola línea y el código se autoexplique. |
| ParkingFeeCalculator.java | La condición `lostTicket` se evalúa antes que la validación de `minutes < 0`, por lo que `calculateFee(-999, true)` no lanza excepción y regresa 150 silenciosamente. No es evidente si esto es una decisión de diseño intencional o un descuido. | Design | Media | Documentar explícitamente con un comentario (o Javadoc) por qué el ticket perdido tiene prioridad sobre la validación de minutos, o bien mover la validación antes si se espera que sea universal. |
| ParkingFeeCalculator.java | El cálculo de horas adicionales y tarifa (`Math.ceil((minutes - 60) / 60.0)` y `20 + additionalHours * 15`) mezcla aritmética con la lógica de flujo principal, lo que reduce la legibilidad del método. | Readability | Baja | Extraer el cálculo a un método privado con nombre propio, por ejemplo `calculateHourlyFee(int minutes)`, separando "qué se calcula" de "cómo se calcula". |
| ParkingFeeCalculatorTest.java | No existían pruebas justo en las fronteras 15/16 minutos (límite entre "gratis" y "tarifa plana") ni en el valor exacto de 60 minutos (límite superior de la tarifa plana), que son los puntos más propensos a errores de `<` vs `<=`. | Testing | Media | Agregar pruebas específicas en esos límites (ya incorporadas: `sixteenMinutesShouldCostTwenty`, `sixtyMinutesShouldStillCostTwenty`) para blindar el código contra errores de "off-by-one". |
| ParkingFeeCalculatorTest.java | La prueba `normalFeeShouldNotBeGreaterThanEighty` (600 minutos) confirma que el tope de $80 funciona, pero no distingue entre "llegar a $80 de forma natural" y "ser recortado por el `Math.min`", dejando esa línea de código sin una prueba que la aísle. | Testing | Baja | Agregar un caso donde la fórmula llega exactamente a 80 sin necesitar el cap (241 min) y otro donde el cálculo crudo superaría 80 y sí requiere el cap (301 min), como se hizo en la ronda de pruebas nuevas. |
| ParkingFeeCalculatorTest.java | El estilo de comentarios `// Arrange / // Act / // Assert` no es consistente: algunas pruebas los incluyen y otras no, lo que resta uniformidad visual al archivo. | Readability | Baja | Definir un estándar para todo el archivo (usar los comentarios en todas las pruebas o en ninguna) y aplicarlo de forma consistente. |
| LegacyParkingReceipt.java | La validación `if (plate == "")` compara `String` con `==` en lugar de `.equals()` o `.isEmpty()`. En Java, `==` compara referencias de objeto, no contenido; aunque a veces "funciona" por el *string pool*, no es fiable (por ejemplo, si `plate` llega como `new String("")` desde otra parte del código, la comparación falla y una placa vacía pasaría la validación). | Bug | Alta | Reemplazar por `plate.isEmpty()` (o `plate.trim().isEmpty()` si se quiere tolerar espacios), ya que `plate == null` ya fue descartado en la línea anterior. |
| LegacyParkingReceipt.java | La lógica del método mezcla responsabilidades distintas: validación de entradas, un efecto secundario (`System.out.println`) y construcción/formato del recibo, todo en un solo método. Esto dificulta probar la lógica de formato sin generar salida por consola, y complica reutilizar la validación en otro contexto. | Design | Media | Separar en métodos (o clases) distintos: uno para validar los parámetros, otro para construir el texto del recibo, y usar un logger (o eliminar el `println`) en lugar de imprimir directamente a consola dentro de la lógica de negocio. |
| LegacyParkingReceipt.java | Código redundante y poco idiomático: `boolean free = fee == 0 ? true : false;` (debería ser simplemente `fee == 0`) y `if (free == true)` (comparar un booleano contra `true` es innecesario). | Readability | Baja | Simplificar a `boolean free = fee == 0;` y `if (free) { ... }`. |
| LegacyParkingReceipt.java | Las dos ramas del `if/else` que construyen `result` duplican casi todo el texto (`label + " - " + plate + " - " + minutes + " min - "`) y solo difieren en el sufijo (`FREE` vs `$" + fee`). | Maintainability | Baja | Construir el prefijo común una sola vez y solo condicionar el sufijo, por ejemplo con `String suffix = free ? "FREE" : "$" + fee;` seguido de una sola concatenación final. |
| LegacyParkingReceipt.java | El literal `"PARKING"` está hardcodeado como etiqueta del recibo, similar al problema de números mágicos visto en `ParkingFeeCalculator`. | Maintainability | Baja | Extraer a una constante con nombre, por ejemplo `private static final String RECEIPT_LABEL = "PARKING";`, para que cualquier cambio de formato se haga en un solo lugar. |

> Nota: no todas las observaciones anteriores son errores; varias son oportunidades de mejora de diseño, legibilidad o cobertura de pruebas que no afectan el comportamiento actual del programa.

## Análisis manual de LegacyParkingReceipt.java (antes de usar herramientas automáticas)

Esta clase fue revisada **manualmente, sin consultar Internet ni ejecutar SonarQube/SonarLint todavía**, con el fin de comparar después el criterio humano contra el de una herramienta automática. Las 5 observaciones de la tabla de arriba resumen los hallazgos principales: un defecto real (comparación de `String` con `==`), un problema de diseño (mezcla de responsabilidades y efecto secundario de consola dentro de lógica de negocio), y tres mejoras de legibilidad/mantenibilidad (booleanos redundantes, duplicación de texto y un literal sin nombre).

## Análisis con SonarQube for IDE

Después de la revisión manual, se instaló SonarQube for IDE (extensión de SonarLint) en Visual Studio y se analizó `LegacyParkingReceipt.java`. A continuación se documentan al menos tres hallazgos reales que la herramienta marcó sobre este archivo:

| Archivo/línea | Regla o mensaje de Sonar | Explicación con mis palabras | ¿Estoy de acuerdo? |
|---------------|---------------------------|------------------------------|---------------------|
| LegacyParkingReceipt.java, línea 18 | "Logs should not be written directly to standard outputs" — usar un logger dedicado en vez de `System.out`/`System.err` directamente. | Imprimir con `System.out.println` mezcla la lógica de negocio con la salida de consola. En una aplicación real eso no es útil: no se puede activar/desactivar, no tiene niveles (info, warning, error), y no queda registrado en ningún archivo o sistema de monitoreo. Un logger (como `java.util.logging.Logger`) resuelve todo eso sin cambiar la lógica del método. | Sí, estoy de acuerdo. De hecho ya lo había señalado en mi revisión manual (fila de "Design" en la tabla de arriba) como parte de la mezcla de responsabilidades, aunque no había profundizado en la razón específica de por qué un logger es mejor que `println`; Sonar lo justifica más claramente (accesibilidad, formato uniforme, registro persistente). |
| LegacyParkingReceipt.java, línea 19-20 | `java:S1125` — "Boolean literals should not be redundant". Comparar una variable booleana contra `true`/`false` (por ejemplo `free == true`) o construir un booleano con un ternario que devuelve literales (`fee == 0 ? true : false`) es innecesario. | Es básicamente decir dos veces lo mismo con palabras distintas: si `fee == 0` ya es una expresión booleana, no hace falta envolverla en un ternario que devuelve `true`/`false` literalmente; y si `free` ya es `true` o `false`, comparar `free == true` es redundante — `if (free)` dice exactamente lo mismo con menos ruido. No es un error de comportamiento, pero sí hace el código más difícil de leer de lo necesario. | Sí, estoy de acuerdo. Yo había marcado este mismo problema en mi revisión manual (fila de "Readability" en la tabla de arriba) antes de correr Sonar, así que coincide exactamente con lo que noté a simple vista. |
| LegacyParkingReceipt.java, línea 9 | "It's almost always a mistake to compare two instances of `java.lang.String`... using reference equality `==` or `!=`" (comparación de `String` con `==` en vez de `.equals()`). | `==` en Java compara si dos variables apuntan al mismo objeto en memoria, no si el contenido de los textos es igual. Dos `String` con el mismo contenido pueden ser objetos distintos en memoria (por ejemplo si uno viene de `new String("")` o de concatenar texto en tiempo de ejecución), y en ese caso `==` daría `false` aunque el texto sea idéntico. Por eso comparar strings siempre debe hacerse con `.equals()` (o `.isEmpty()` para el caso de una cadena vacía). | Sí, estoy de acuerdo, y con la severidad más alta de las tres: este es el único de los hallazgos que es un **bug** real (puede causar que una placa vacía pase la validación en ciertos escenarios), no solo un tema de estilo. Coincide con lo primero que noté en mi revisión manual, antes de usar Sonar. |

### Comparación entre mi revisión manual y la herramienta

Los tres hallazgos que Sonar marcó **ya habían sido identificados en la revisión manual** (ver tabla de "Code review manual" más arriba), lo cual confirma que el análisis humano y el automático coinciden en los problemas más evidentes de este archivo. La diferencia principal es que Sonar:

- Da un **nombre de regla estandarizado** (por ejemplo `java:S1125`) y una **categoría** (Maintainability, Consistency issue), lo cual facilita rastrear el problema y buscar más contexto.
- Explica el "por qué" de forma más técnica y con ejemplos de código "compliant" vs "non-compliant", útil para justificar la corrección ante otros desarrolladores.
- No sustituye el criterio humano para detectar problemas de **diseño más amplios** (como la mezcla de responsabilidades entre validación/logging/formato, o la duplicación de texto entre las dos ramas del `if/else`), que en este caso sí fueron detectados manualmente pero que una herramienta de análisis estático línea por línea no necesariamente resalta con la misma claridad.

## Correcciones realizadas

Se eligieron **dos** hallazgos para corregir directamente en `LegacyParkingReceipt.java` (no era obligatorio corregir todos):

### Corrección 1 — Comparación de `String` con `==`

- **Qué cambié:** la línea `if (plate == "")` se reemplazó por `if (plate.isEmpty())`.
- **Qué problema intentaba resolver:** `==` compara si dos referencias apuntan al mismo objeto en memoria, no si el contenido del texto es igual. Esto podía hacer que una placa vacía (`plate = new String("")`, por ejemplo) **no** fuera detectada como vacía y pasara la validación, permitiendo generar un recibo inválido.
- **¿Sonar dejó de reportarlo?** Sí. La regla de comparación de `String` con referencia (`==`/`!=`) ya no aparece marcada en SonarQube for IDE sobre esa línea después del cambio.
- **Por qué quedó mejor:** ahora la validación de "placa vacía" funciona de forma confiable sin importar cómo se haya construido el `String` que llega como parámetro. Es una corrección de comportamiento (bug real), no solo de estilo, y usa el método estándar de Java (`isEmpty()`) pensado exactamente para este caso.

### Corrección 2 — Booleanos redundantes 

- **Qué cambié:** `boolean free = fee == 0 ? true : false;` se simplificó a `boolean free = fee == 0;`, y `if (free == true)` se simplificó a `if (free)`.
- **Qué problema intentaba resolver:** ambas expresiones comparaban o construían un valor booleano usando literales `true`/`false` de forma innecesaria, ya que `fee == 0` y `free` ya son expresiones/variables booleanas por sí mismas. Esto no cambia el comportamiento, pero agrega ruido visual y hace que el código parezca más complejo de lo que realmente es.
- **¿Sonar dejó de reportarlo?** Sí, la regla `java:S1125` ("Boolean literals should not be redundant") ya no aparece marcada en esas líneas tras la simplificación.
- **Por qué quedó mejor:** el código ahora dice exactamente lo mismo con menos palabras y sin comparaciones redundantes, lo cual facilita leerlo de un vistazo y reduce la posibilidad de que alguien, al modificarlo después, se confunda con la doble negación o comparación innecesaria.


## PREGUNTAS RELACIONADAS

**P1. ¿Por qué probar muchos valores de la misma región no ayuda?**

Porque si el código usa la misma fórmula para todos esos datos, probar varios dará el mismo resultado. Si funciona con uno, casi seguro funciona con los demás. Repetir solo hace las pruebas más lentas sin encontrar errores nuevos.

**P2. Dos fronteras importantes y por qué probarlas**

- **15 vs 16 minutos (Gratis a $20):** Es donde empieza el cobro. Un simple error de `<` por `<=` cobraría antes de tiempo o dejaría pasar a alguien gratis.
- **300 vs 301 minutos (Llegar al límite de $80):** En 300 minutos el cobro llega solo a $80 por la tarifa por hora, pero en 301 minutos actúa el tope máximo. Probar ahí asegura que el tope realmente funcione cuando debe.

En este sentido, valen la pena porque donde cambia la regla es donde la gente suele equivocarse al programar.

**P3. ¿Que todas estén en verde significa que el programa está bien?**

No. Solo significa que el programa pasó los casos que se revisaron. Todavía pueden existir errores en casos no probados, o se pudo haber escrito mal la prueba desde el principio.

**P4. ¿Qué problema encontró Sonar que tú no habías identificado durante el desarrollo?**

Sonar señaló una condición duplicada/redundante en la lógica del cálculo del tope de $80 (la misma comparación de minutos aparecía repetida en dos puntos distintos del código), algo que no había notado al revisarlo manualmente porque el programa funcionaba de todas formas.

**P5. ¿Qué observación hiciste tú que Sonar no reportó?**

Noté que algunos nombres de variables eran poco descriptivos (por ejemplo, usar `m` en vez de `minutos`), lo que dificultaba entender la lógica de negocio a simple vista. Sonar no marca esto porque no evalúa qué tan claro o descriptivo es un nombre, solo patrones de código.

**P6. ¿Consideras que todos los hallazgos de Sonar tienen la misma importancia? Explica un ejemplo.**

No. Sonar clasifica los hallazgos por severidad (bug, vulnerabilidad, code smell). Por ejemplo, un bug que puede provocar un cobro incorrecto es crítico y debe corregirse de inmediato, mientras que un code smell como una variable con nombre muy corto es solo una sugerencia de estilo que no afecta el funcionamiento del programa.

**P7. ¿Puede Sonar determinar por sí solo si "$20 de 16 a 60 minutos" es la regla correcta del negocio? ¿Por qué?**

No. Sonar solo detecta patrones de código (duplicación, complejidad, errores comunes de programación), pero no conoce las reglas reales del negocio. Solo una persona que conozca el reglamento de cobro puede confirmar si esa regla específica es la correcta.

**P8. Explica con tus palabras por qué el análisis estático NO sustituye las pruebas unitarias ni el code review.**

Las pruebas unitarias y el code review no pueden ser sustituidos porque requieren criterio humano: una persona entiende el contexto del negocio y puede juzgar si el comportamiento del programa es el correcto, algo que una herramienta automática no puede hacer. El análisis estático solo detecta patrones de código previamente definidos, pero no tiene la capacidad de razonar como un supervisor humano ni de validar que la lógica cumpla con lo que realmente se espera del sistema.
