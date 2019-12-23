% Problemas al tratar con números enteros grandes % Albert % 2019
\newpage
\setcounter{tocdepth}{4} \tableofcontents
\newpage

## Introducción al problema
Actualmente estamos construyendo un puente entre Binance Chain y NEO, uno de los componentes de dicho puente es un programa alojado en NEO que verifica los bloques de Binance Chain, y parte de la verificación de estos bloques incluye la verificación de varias [firmas de Schnorrf](https://en.wikipedia.org/wiki/Schnorr_signature) que fueron creadas utilizando el [Ed25519](https://en.wikipedia.org/wiki/EdDSA#Ed25519). Así que resumiendo, necesitamos implementar un programa que verifique las firmas del Ed25519 dentro de NeoVM, que es la máquina virtual usada en NEO.

## Entorno
NeoVM, el entorno en el que se ejecutará el algoritmo es especial debido a las restricciones asociadas a él:
- Opera con números de 256 bits firmados, lo que significa que puede operar directamente sobre números que pertenecen^[_De hecho creo que esta suposición no ha sido probada_] puede operar sobre números que son marginalmente más grandes, debido a que puede almacenar los números en _two's complement_ pero eso no debería cambiar nada, ya que sólo permite que un número más sea representado, de €(2^255, -2^255)€.
- La multiplicación, la adición, la suma, el módulo, la división de números enteros y todas las operaciones de números estándar  son soportadas y tienen todas el mismo coste computacional.
- La VM (máquina virtual) es [turing completa](https://en.wikipedia.org/wiki/Turing_completeness), por lo que es posible utilizar bucles y condicionales.
- Si alguna operación matemática resulta en un _underflow_ o _overflow_ la ejecución completa se detiene y la VM falla, por lo tanto a través de toda la ejecución del algoritmo debemos asegurarnos de que nunca suceda esto. Por ejemplo, si se intenta calcular `(a*b)%p` con números grandes fallará porque el resultado intermedio `a*b` se desbordará, ocurrira un _overflow_. También cabe destacar, que es imposible comprobar si hay _overflows/underflows_ después de que hayan ocurrido, ya que para entonces la VM ya habrá fallado y no soporta el manejo de _excepciones_.
-Las operaciones tienen los siguientes costos, todos los precios están en GAS, que es aproximadamente equivalente a USD. (Mirar tabla).

| Operación | Coste |
|-----------|------|
| Almacenar 1KB en memoria permanente | 1 |
| Leer dato en memoria permanente | 0.1 |
| Todas las otras operaciones | 0.001 |
 
Esta diferencia de costes significa que si tomamos un conjunto de opcodes (_códigos de operación_) que pueden ser ejecutados en 1 segundo en un procesador Intel Core i7 6950X de 3GHz (lanzado en 2016) y en su lugar los ejecutamos dentro del NeoVM, el coste de esa ejecución^[IPS/ciclo de reloj tomado los datos de https://en.wikipedia.org/wiki/Instructions\_per\_second#Timeline\_of_instructions\_per_second] será de 3*10^9*106*0.001=318.000.000 de GAS y alrededor de la misma cantidad en USD. Por lo tanto, está claro que el coste de ejecutar código dentro de NeoVM es masivamente caro y requeriremos una amplia optimización. Encontrar alternativas a este problemas será la principal motivación del presente texto.


