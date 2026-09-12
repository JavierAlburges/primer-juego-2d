# Guía para Agentes de IA (AGENTS.md)

Este documento proporciona el contexto técnico, convenciones de diseño y reglas de desarrollo para agentes autónomos y asistentes de IA que colaboren en el mantenimiento y evolución de este repositorio.

---

## 1. Visión General del Proyecto

- **Nombre del Proyecto**: PrimerJuego2D (Dodge the Creeps)
- **Motor / Versión**: Godot Engine 4.x
- **Lenguaje**: GDScript (Godot 4 syntax)
- **Tipo de Juego**: 2D Arcade / Supervivencia (Evadir enemigos generados aleatoriamente)
- **Resolución Base**: `480x720` (Modo stretch: `canvas_items`)
- **Renderizador**: `gl_compatibility`

---

## 2. Arquitectura de Escenas y Nodos

El proyecto sigue una arquitectura jerárquica basada en componentes y escenas modulares:

### `Main` (`main.tscn`, `main.gd`)
- **Rol**: Orquestador principal del ciclo de vida del juego.
- **Responsabilidades**:
  - Iniciar y reiniciar partidas (`new_game()`).
  - Manejar el estado de fin de juego (`game_over()`).
  - Spawneo dinámico de enemigos mediante `Path2D` (`MobPath`) y `PathFollow2D` (`MobSpawnLocation`).
  - Control de temporizadores (`MobTimer`, `ScoreTimer`, `StartTimer`).
  - Coordinar la reproducción de música y efectos sonoros (`Music`, `DeathSound`).
  - Comunicación con la interfaz a través de `HUD`.

### `Player` (`player.tscn`, `player.gd`)
- **Tipo de Nodo**: `Area2D`
- **Señales**:
  - `hit`: Emitida cuando un cuerpo entra en contacto con el jugador (`_on_body_entered`).
- **Mecánicas**:
  - Movimiento multidireccional continuo normalizado (velocidad: `400 px/s`).
  - Restricción de posición dentro de los límites del viewport (`clamp`).
  - Selección dinámica de animación en `AnimatedSprite2D` (`walk`, `up`) e inversión de sprites (`flip_h`, `flip_v`).
  - Desactivación diferida de la colisión (`set_deferred("disabled", true)`) al recibir impacto para evitar colisiones múltiples en el mismo frame de física.

### `Mob` (`mob.tscn`, `mob.gd`)
- **Tipo de Nodo**: `RigidBody2D` (con `gravity_scale = 0` para simular movimiento arcade en el plano 2D)
- **Comportamiento**:
  - Al instanciarse, selecciona aleatoriamente una animación de entre sus frames disponibles (ej. `fly`, `swim`, `walk`).
  - Movimiento rectilíneo impulsado mediante `linear_velocity`.
  - Destrucción automática al salir de pantalla usando `VisibleOnScreenNotifier2D` y la señal `screen_exited` (`queue_free()`).
  - Pertenecen al grupo de nodos `"mobs"` para su eliminación en lote al reiniciar la partida (`get_tree().call_group("mobs", "queue_free")`).

### `HUD` (`hud.tscn`, `hud.gd`)
- **Tipo de Nodo**: `CanvasLayer`
- **Señales**:
  - `start_game`: Emitida cuando el usuario pulsa el botón de inicio.
- **Responsabilidades**:
  - Visualización y actualización del puntaje.
  - Gestión de mensajes transitorios y de game over con temporizadores asíncronos (`await $MessageTimer.timeout`).

---

## 3. Convenciones de Código y GDScript (Godot 4)

Los agentes deben acatar las siguientes pautas al generar o modificar código:

1. **Sintaxis de Godot 4**:
   - Usar anotaciones `@export`, `@onready`, etc. No usar la sintaxis obsoleta de Godot 3 (`export var`).
   - Conexión de señales y llamadas usando nombres de métodos tipados o lambdas cuando sea apropiado.
   - Usar `hit.emit()` en lugar del viejo `emit_signal("hit")`.
2. **Tipado Estático y Seguridad**:
   - Preferir tipado en firmas de funciones cuando aporte claridad (ej. `func _process(delta: float) -> void:`).
   - Siempre que se modifiquen propiedades físicas dentro de callbacks de colisión, utilizar `set_deferred()` para evitar errores del motor.
3. **Manejo de Memoria**:
   - Asegurar que cualquier entidad instanciada dinámicamente (`mob.instantiate()`) se elimine limpiamente mediante `queue_free()` para prevenir fugas de memoria.
4. **Respeto a los Nombres de Acciones de Entrada**:
   - Las acciones configuradas en `project.godot` son:
     - `move_right`
     - `move_left`
     - `move_up`
     - `move_down`
     - `start_game`
   - Si se agregan nuevas mecánicas (ej. turbo, dash, disparo), deben documentarse y registrarse tanto en `project.godot` como en el `README.md`.

---

## 4. Instrucciones para Tareas y Extensiones Comunes

- **Agregar Nuevos Tipos de Enemigos**:
  - Extender o agregar animaciones en `mob.tscn` -> `AnimatedSprite2D`, o derivar una nueva escena que herede de `RigidBody2D` y mantenga la señal de salida de pantalla y pertenencia al grupo `"mobs"`.
- **Power-ups / Items**:
  - Crear escenas basadas en `Area2D`, agregarlas al grupo correspondiente y conectar señales de área con el `Player` o gestionarlas en `Main`.
- **Modificación de HUD**:
  - Asegurar que cualquier nuevo elemento mantenga el orden de renderizado adecuado dentro del `CanvasLayer`.
- **Exportación y Generación de Ejecutables (.exe)**:
  - El proyecto define presets en `export_presets.cfg`. El preset activo de Windows es `"Equiva"` (`platform="Windows Desktop"`, ruta de salida `./builds/Windows/PrimerJuego2D.exe`).
  - Para generar el ejecutable mediante CLI o integración continua (CI/CD):
    ```bash
    godot --headless --export-release "Equiva" builds/Windows/PrimerJuego2D.exe
    ```
  - Se generan los archivos vinculados: binario ejecutable (`.exe`) y paquete de recursos (`.pck`). Asegurarse de mantener ambos juntos al distribuir.
- **Exportación a Web (HTML5 / WebAssembly)**:
  - Preset activo: `"Web"` (`platform="Web"`, ruta de salida `./builds/Web/index.html`).
  - Comando CLI para exportación:
    ```bash
    godot --headless --export-release "Web" builds/Web/index.html
    ```
  - Produce `index.html`, `index.js`, `index.wasm` e `index.pck`. Requiere ser servido por HTTP/HTTPS debido a políticas de seguridad del navegador para WebAssembly.


