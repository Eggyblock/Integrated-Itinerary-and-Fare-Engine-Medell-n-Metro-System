# Integrated Itinerary and Fare Engine — Medellín Metro System

Segunda Práctica (20%) · Lenguajes y Paradigmas de Computación (SI1002-7) · Universidad EAFIT · 2026-2

## Integrantes

1. Miguel Ángel Palacios Parra
2. Joel Atlai Romero

## Descripción

Motor de tarifas para el sistema integrado de transporte del Valle de Aburrá. Modela los modos de transporte (metro, tranvía, cable, cable turístico y ruta alimentadora) y los cinco perfiles de pasajero de la tarjeta Cívica, y cobra un itinerario completo aplicando la regla de integración de 90 minutos con hasta dos transbordos gratis.

El programa es un único archivo, `main.cpp`, organizado en cuatro partes:

| Parte | Contenido |
|-------|-----------|
| 1 | Jerarquía abstracta `TransportMode` con funciones virtuales puras y hooks de política |
| 2 | Jerarquía `Passenger` con el idioma Non-Virtual Interface (NVI) |
| 3 | Clase `Network` (dueña de los modos mediante `std::unique_ptr`) y la función `chargeItinerary` |
| 4 | Reportes construidos con algoritmos de la STL, lambdas y una `std::function` inyectada |

## Cómo compilar y ejecutar

Con g++ (C++17):

```bash
g++ -std=c++17 -Wall -Wextra -o practice main.cpp
./practice
```

El programa compila con cero errores y cero warnings con estas banderas.

En OnlineGDB: seleccionar el lenguaje **C++17**, pegar todo el contenido de `main.cpp` y presionar **Run**. El programa no lee nada de la entrada estándar.

## Principales decisiones de diseño

- **Funciones virtuales puras en lugar de preguntar por el tipo.** `baseFare()` y `serviceType()` son virtuales puras, así que `TransportMode` es abstracta y el compilador obliga a cada modo concreto a definir ambos valores. Las diferencias de comportamiento (tarifa plana de estudiante, integración, apertura de ventana) se expresan con tres hooks virtuales que por defecto devuelven `true` y solo se sobrescriben donde el comportamiento cambia de verdad. En ningún lugar se usa `dynamic_cast`, `typeid` ni comparación de los strings de `serviceType()`.
- **Método plantilla en `describe()`.** No es virtual: el formato se escribe una sola vez en la clase base y llama a los miembros virtuales que aporta cada subclase.
- **Reutilización con llamadas calificadas.** `TouristCable` obtiene su tarifa con `CableLine::baseFare() + surcharge_`, y cada perfil de pasajero obtiene la tarifa completa con `Passenger::computeFare(mode)`, que es el equivalente en C++ de `super`. El literal `3820` aparece solo en `MetroLine`, `TramLine` y `CableLine`.
- **Non-Virtual Interface.** `Passenger::fareOn()` es pública y no virtual: llama a la función protegida y virtual `computeFare()` y acota el resultado con `std::max(0LL, ...)`. Como el recorte está en un solo lugar, ningún perfil puede producir una tarifa negativa (por ejemplo, un adulto mayor en la ruta alimentadora gratuita) y ninguna subclase puede saltárselo. `FrequentPassenger` no sobrescribe nada, porque el comportamiento por defecto ya es su tarifa.
- **Propiedad de los objetos.** `Network` es dueña de los modos mediante `std::vector<std::unique_ptr<TransportMode>>`, creados con `std::make_unique`; `main()` y `Leg` solo los observan mediante referencias y punteros que no son dueños. No hay ningún `new` ni `delete` en el archivo.
- **Regla de integración.** Solo los modos para los que `allowsIntegration()` y `opensIntegrationWindow()` responden `true` pueden abrir, reiniciar o usar la ventana. El cable turístico y la ruta alimentadora pagan su propia tarifa y dejan la ventana intacta; por eso un viaje que empieza en una ruta alimentadora no queda gratis por completo.

## Respuestas a las preguntas de la defensa

### 1. ¿Qué pasa si el destructor de `TransportMode` no se declara virtual?

La red guarda cada modo como un `std::unique_ptr<TransportMode>`, aunque los objetos reales sean `MetroLine`, `TouristCable`, etc. Cuando el vector se destruye, cada `unique_ptr` ejecuta `delete` sobre un puntero de tipo `TransportMode*`.

Si el destructor no es virtual, eliminar un objeto derivado a través de un puntero a la base es **comportamiento indefinido** según el estándar de C++. En la práctica, la llamada se resuelve de forma estática: solo se ejecuta `~TransportMode()` y el destructor de la clase derivada nunca corre. Cualquier recurso que tenga la parte derivada (un string, un contenedor, un archivo, memoria) nunca se libera, y el programa incluso puede corromper la memoria.

Declarar `virtual ~TransportMode() = default;` hace que la llamada al destructor pase por despacho dinámico como cualquier otra función virtual: primero se ejecuta el destructor del tipo real y luego el de la base. Por eso es un requisito de corrección y no una formalidad: toda clase pensada para eliminarse de forma polimórfica debe tener destructor virtual.

### 2. ¿Por qué una clase abstracta con funciones virtuales puras es preferible a una enumeración con un switch?

Con una enumeración, cada decisión que depende del modo (tarifa, nombre del servicio, tarifa plana de estudiante, integración, ventana) se convierte en un `switch` repetido en varias funciones. Agregar un modo nuevo obliga a abrir y modificar todos esos lugares, y olvidar uno es un error silencioso que muchas veces solo aparece en tiempo de ejecución.

Con la clase abstracta, todo lo relacionado con un modo vive en su propia clase. Agregar un modo nuevo significa escribir una sola clase nueva: `chargeItinerary`, los perfiles de pasajero y los reportes no cambian. Este es el principio abierto/cerrado: el diseño está abierto a la extensión y cerrado a la modificación. En este proyecto, `TouristCable` cambia la tarifa de estudiante, la integración y la ventana, y nada de eso obligó a tocar `StudentPassenger` ni `chargeItinerary`, porque estos le preguntan al objeto por su política en vez de preguntarle qué es.

Además, el compilador protege el diseño: una clase que no implemente `baseFare()` o `serviceType()` no se puede instanciar, y nunca se puede crear por error un `TransportMode` abstracto.

### 3. ¿Qué nos aporta la STL al reemplazar un ciclo escrito a mano por un algoritmo con una lambda?

- **El código expresa su intención.** `accumulate` dice "sumar", `count_if` dice "contar los que cumplen esto", `find_if` dice "el primero que cumple esto". Un ciclo `for` hay que leerlo completo para saber qué hace.
- **Menos errores.** No hay índices, límites ni variables temporales que manejar mal, así que desaparecen los típicos errores de "uno de más o uno de menos".
- **Separación de responsabilidades.** El algoritmo se encarga del recorrido y la lambda solo expresa el criterio. En `namesMatching`, el criterio llega como una `std::function` inyectada, de modo que `Network` no conoce ni necesita conocer qué filtro aplica quien la llama; un criterio nuevo no requiere ningún cambio en la red.

El único ciclo explícito del código es el de `chargeItinerary`, porque ese ciclo es el propio algoritmo de integración, con un estado (ventana abierta, minuto de inicio, transbordos gratis usados) que se arrastra de una pierna del viaje a la siguiente.
