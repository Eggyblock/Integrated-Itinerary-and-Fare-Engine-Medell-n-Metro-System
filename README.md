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
- **Reutilización con llamadas calificadas.** `TouristCable` obtiene su tarifa con `CableLine::baseFare() + surcharge_`, y cada perfil de pasajero obtiene la tarifa completa con `Passenger::computeFare(mode)`, que es el equivalente en C++ de `super`. El literal
