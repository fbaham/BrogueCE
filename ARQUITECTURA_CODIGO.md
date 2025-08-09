# Documentación de Arquitectura - Brogue CE

## Descripción General

Brogue CE (Community Edition) es un roguelike desarrollado en C que implementa una arquitectura modular con separación clara entre la lógica del juego y las diferentes plataformas de renderizado. El juego utiliza un sistema basado en turnos con generación procedimental de mazmorras.

## Estructura del Proyecto

```
BrogueCE/
├── src/
│   ├── brogue/           # Motor principal del juego
│   ├── platform/         # Implementaciones específicas de plataforma
│   └── variants/         # Variantes del juego (Brogue, Rapid Brogue, Bullet Brogue)
├── bin/                  # Ejecutables compilados
├── assets/               # Recursos gráficos y de audio
├── make/                 # Scripts de compilación
└── test/                 # Suite de pruebas y regresión
```

## Arquitectura Principal

### 1. Sistema de Abstracción de Plataforma

El juego utiliza una arquitectura basada en consola virtual que permite ejecutarse en múltiples plataformas:

#### Estructura de Consola (`platform.h`)
```c
struct brogueConsole {
    void (*gameLoop)(void);
    boolean (*pauseForMilliseconds)(short milliseconds, PauseBehavior behavior);
    void (*nextKeyOrMouseEvent)(rogueEvent *returnEvent, boolean textInput, boolean colorsDance);
    void (*plotChar)(enum displayGlyph ch, short xLoc, short yLoc, ...);
    boolean (*modifier_held)(int modifier);
    enum graphicsModes (*setGraphicsMode)(enum graphicsModes mode);
};
```

#### Plataformas Soportadas
- **SDL2** (`sdl2-platform.c`): Implementación principal con gráficos
- **Curses** (`curses-platform.c`): Terminal/consola
- **Web** (`web-platform.c`): Navegador web
- **Null** (`null-platform.c`): Modo servidor/pruebas

### 2. Motor Principal del Juego (`src/brogue/`)

#### Archivo Principal: `RogueMain.c`
- **`mainBrogueJunction()`**: Punto de entrada principal que maneja menús y estados del juego
- **`initializeRogue()`**: Inicialización del estado global del juego
- **`startLevel()`**: Carga o genera un nuevo nivel
- **`gameOver()`**: Manejo del final del juego

#### Gestión de Estados Globales
```c
typedef struct playerCharacter {
    boolean wizard;
    short depthLevel, deepestLevel;
    boolean gameInProgress, gameHasEnded;
    boolean playbackMode;
    unsigned long playerTurnNumber;
    // ... más de 100 campos de estado
} playerCharacter;

extern playerCharacter rogue; // Estado global principal
```

### 3. Sistema de Generación de Niveles (`Architect.c`)

#### Algoritmo Principal: `digDungeon()`
1. **Limpieza inicial**: Rellena todo con granito
2. **Tallado de mazmorras**: `carveDungeon()` usando perfiles de mazmorra
3. **Adición de bucles**: `addLoops()` para conectividad extra
4. **Diseño de lagos**: `designLakes()` y `fillLakes()`
5. **Autogeneradores**: `runAutogenerators()` para características del terreno
6. **Máquinas**: `addMachines()` para salas especiales
7. **Puentes**: `buildABridge()` para cruzar líquidos
8. **Acabados**: `finishDoors()` para puertas secretas

#### Tipos de Salas
```c
enum roomTypes {
    0, // Cross room (habitación en cruz)
    1, // Small symmetrical cross
    2, // Small room (habitación pequeña)
    3, // Circular room (habitación circular)
    4, // Chunky room (habitación irregular)
    5, // Cave (cueva)
    6, // Cavern (caverna grande)
    7, // Entrance room (sala de entrada)
};
```

#### Sistema de Máquinas
Las máquinas son complejos de salas especiales con características únicas:
- **Máquinas de recompensa**: Salas de tesoro con desafíos
- **Máquinas de área**: Modifican grandes secciones del nivel
- **Máquinas de vestíbulo**: Salas especiales en puertas

### 4. Sistema de Turnos (`Time.c`)

#### Función Principal: `playerTurnEnded()`
1. **Manejo de caídas**: `monstersFall()`
2. **Bucle de tiempo**: Procesa ticks hasta que el jugador tenga turno
3. **Actualización de entorno**: `updateEnvironment()` cada 100 ticks
4. **Turnos de monstruos**: `monstersTurn()` para cada criatura
5. **Efectos graduales**: Aplicación de estados y efectos

#### Gestión de Ticks
```c
// Los personajes acumulan "ticks" basados en su velocidad
creature->ticksUntilTurn -= soonestTurn;
if (creature->ticksUntilTurn <= 0) {
    monstersTurn(creature);
}
```

### 5. Sistema de Entrada (`IO.c`)

#### Bucle Principal de Entrada: `mainInputLoop()`
```c
void mainInputLoop() {
    while (!rogue.gameHasEnded) {
        nextBrogueEvent(&theEvent, false, rogue.cautiousMode, false);
        executeEvent(&theEvent);
    }
}
```

#### Manejo de Eventos
- **Movimiento**: `playerMoves()` y `playerRuns()`
- **Acciones**: Inventario, equipar, usar objetos
- **Modo cursor**: Navegación e inspección del mapa
- **Comandos especiales**: Guardar, cargar, configuración

### 6. Sistema de IA de Monstruos (`Monsters.c`)

#### Estados de Monstruos
```c
enum monsterStates {
    MONSTER_SLEEPING,     // Durmiendo
    MONSTER_TRACKING_SCENT, // Siguiendo rastro
    MONSTER_WANDERING,    // Vagando
    MONSTER_FLEEING,      // Huyendo
    MONSTER_ALLY          // Aliado del jugador
};
```

#### Algoritmo de IA: `monstersTurn()`
1. **Aplicar efectos**: Estado, veneno, etc.
2. **Actualizar estado**: Dormido → Alerta → Persiguiendo
3. **Tomar decisión**: Movimiento, ataque, habilidades especiales
4. **Ejecutar acción**: Mover, atacar, o usar habilidad

### 7. Sistema de Grabación y Reproducción (`Recordings.c`)

#### Determinismo
- Todas las acciones se graban como eventos
- RNG (Random Number Generator) con semilla controlada
- Reproducción exacta de partidas anteriores

#### Tipos de Grabación
- **Partidas guardadas**: Estado completo del juego
- **Grabaciones**: Secuencia de eventos desde el inicio
- **Modo reproducción**: Visualización de partidas grabadas

### 8. Variantes del Juego (`src/variants/`)

#### Brogue Estándar (`GlobalsBrogue.c`)
- Configuración base del juego original
- Tablas de probabilidades estándar

#### Rapid Brogue (`GlobalsRapidBrogue.c`)
- Versión más rápida con menos niveles
- Progresión acelerada

#### Bullet Brogue (`GlobalsBulletBrogue.c`)
- Enfoque en combate y acción
- Mecánicas modificadas

## Sistemas Secundarios

### Sistema de Visión y FOV (Field of View)
- **`updateVision()`**: Calcula visibilidad usando algoritmo de raycasting
- **`getFOVMask()`**: Máscara de campo de visión
- **Stealth**: Sistema de sigilo con rangos dinámicos

### Sistema de Pathing (Búsqueda de Caminos)
- **Dijkstra**: Para navegación de IA y jugador
- **Mapas de costo**: Diferentes costos por terreno
- **Waypoints**: Puntos de navegación precalculados

### Sistema de Objetos (`Items.c`)
- **Generación procedimental**: Objetos con estadísticas aleatorias
- **Identificación gradual**: Los objetos se identifican con el uso
- **Maldiciones y encantamientos**: Efectos positivos y negativos

### Sistema de Combate (`Combat.c`)
- **Cálculo de daño**: Basado en estadísticas y aleatoriedades
- **Estados de combate**: Aturdimiento, veneno, fuego, etc.
- **Armadura y defensa**: Reducción de daño y esquive

## Flujo Principal del Programa

### 1. Inicialización
```
main() → currentConsole.gameLoop() → mainBrogueJunction()
```

### 2. Menú Principal
```
titleMenu() → processButtonInput() → selección de comando
```

### 3. Nueva Partida
```
initializeRogue() → startLevel(1) → mainInputLoop()
```

### 4. Bucle de Juego
```
nextBrogueEvent() → executeEvent() → playerTurnEnded() → repeat
```

### 5. Generación de Nivel
```
digDungeon() → populateMonsters() → populateItems() → setUpWaypoints()
```

## Gestión de Memoria

### Grids Dinámicas
```c
short **grid = allocGrid();  // Asigna DCOLS x DROWS
// ... uso del grid
freeGrid(grid);             // Libera memoria
```

### Listas de Criaturas
```c
creatureList *monsters;     // Lista enlazada de monstruos
creature *monst = generateMonster(type);
prependCreature(monsters, monst);
```

### Limpieza Global
```c
void freeEverything() {
    // Libera todos los recursos al final del juego
    // Grids, criaturas, objetos, niveles
}
```

## Características Notables

### 1. Arquitectura Modular
- Separación clara entre motor y presentación
- Fácil portabilidad a nuevas plataformas
- Sistema de plugins para variantes

### 2. Determinismo Completo
- Reproducibilidad exacta de partidas
- RNG controlado por semillas
- Sistema de grabación robusto

### 3. Generación Procedimental Avanzada
- Algoritmos sofisticados de generación de niveles
- Sistema de máquinas para contenido especial
- Equilibrio dinámico basado en profundidad

### 4. Escalabilidad
- Soporte para múltiples variantes del juego
- Sistema de configuración flexible
- Código modular y extensible

## Herramientas de Desarrollo

### Depuración
- Modo wizard con comandos especiales
- Visualización de mapas internos (FOV, safety, etc.)
- Sistema de asserts para verificación

### Testing
- Suite de pruebas de regresión
- Comparación de catálogos de semillas
- Verificación automática de determinismo

### Compilación
- Makefiles multiplataforma
- Configuración mediante `config.mk`
- Soporte para diferentes backends gráficos

Este documento proporciona una visión general de la arquitectura de Brogue CE. El código es notable por su claridad, modularidad y el sofisticado sistema de generación procedimental que mantiene un equilibrio entre aleatoriedad y jugabilidad equilibrada.
