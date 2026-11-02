# CLAUDE.md — Módulo didáctico: banda clasificadora (Tesis UTLA)

Este repo contiene una página web de estudio (un solo archivo, `index.html`) que explica a tres estudiantes de Ingeniería Eléctrica cómo funciona el módulo de su tesis. Ellos tienen poco dominio técnico y la usan para prepararse antes de reunirse con su asesor y para la defensa. La página se publica con GitHub Pages.

Quien pide los cambios es David, un ingeniero mecatrónico que les ayuda. El público final son los estudiantes, no ingenieros.

---

## 1. La tesis

- **Título:** "Estudio y construcción de módulo para la simulación de entornos industriales automáticos utilizando el software Factory I/O"
- **Universidad:** Universidad Técnica Latinoamericana (UTLA), Santa Tecla, El Salvador. Escuela de Ingeniería Eléctrica.
- **Autores:** Agustín José Durán López, Irsia Ismary Aquino Mendoza, Idalia Madelyn Osegueda González.
- **Asesor:** Ing. Julio Adalberto Martínez López.
- **Materia de las guías:** Control Automático de Motores.

### Objetivos específicos (Cap. I, 1.2.2)
1. Demostrar por Ethernet y TCP/IP la visualización y control de un sistema automatizado con Factory I/O.
2. Implementar elementos de control y mando para adquirir señales digitales y poner en marcha el módulo.
3. Estructurar **cuatro guías prácticas de laboratorio** con el PLC S7-1200 y TIA Portal.

### Estructura obligatoria del Capítulo IV (viene del anteproyecto, no se cambia)
4.1 Módulo de pruebas de automatización y comunicaciones · 4.2 Elementos del equipo · 4.3 Características · 4.4 Característica módulo · 4.5 Salidas analógicas · 4.6 Salida analógica en el módulo · 4.7 Entradas y salidas digitales · 4.8 Configuración para establecer las variables.

Las fuentes que cita el cuestionario (campo `s`) usan esta numeración.

---

## 2. Qué hace el módulo

Es una banda transportadora que **separa piezas metálicas de piezas no metálicas**. El orden es lo que hace que funcione:

1. La pieza pasa primero por el **sensor inductivo**. Si es metal, el **cilindro 1 (Y1)** la saca.
2. Lo que sigue en la banda llega al **sensor fotoeléctrico**, que solo detecta presencia. Como el metal ya salió, lo que llega ahí es no metálico, y el **cilindro 2 (Y2)** lo saca.
3. Los **sensores Reed** S3 y S4 confirman que cada cilindro llegó al final de su carrera. Con esa señal, y cuando el sensor ya no ve la pieza, el cilindro regresa.

Hay dos modos, que se eligen con la maneta de 3 posiciones (MANUAL / 0 / AUTO):
- **Automático:** se sostiene START 3 s y enciende la luz verde. 3 s después arranca el variador y la banda se mueve. Los sensores deciden. STOP lo detiene todo.
- **Manual:** cada toque de START avanza un paso con contadores CTU: 1 luz, 2 banda, 3 cilindro 1, 4 cilindro 2, 5 y 6 (ver pendientes), 7 reinicio de todos los contadores.

---

## 3. Datos técnicos verificados

Esta es la fuente de verdad. Si el documento de la tesis o las guías dicen otra cosa, vale lo de aquí.

### PLC
- Siemens S7-1200, **CPU 1214C AC/DC/RLY**, ref. 6ES7214-1BG40-0XB0.
- 14 entradas digitales a 24 V DC, 10 salidas de relé (2 A), 2 entradas analógicas de 0 a 10 V, **sin salidas analógicas**.
- Se alimenta con **120 V AC** (rango 85–264 V AC), protegido por Q4. Los bornes L+ y M de este modelo son una salida de 24 V DC para sensores.
- Programa en **TIA Portal V17**, proyecto "PRUEBA TESIS", lenguaje Ladder, bloque **Main [OB1]**, **11 networks**.
- IP del PLC **192.168.0.5**, máscara 255.255.255.0. IP de la laptop **192.168.0.2** (adaptador Ethernet; la 192.168.3.x es el Wi-Fi).
- Protección: acceso "Full access (no protection)" y **PUT/GET habilitado** en Protection & Security.

### Entradas (tabla "ETIQUETAS DE PROGRAMA")
| Dirección | Tag | Notas |
|---|---|---|
| %I0.0 | PULADOR START | Así está escrito en TIA Portal (falta la "S"), pulsador NO |
| %I0.1 | PULSADOR STOP | NC |
| %I0.2 | AUTO | |
| %I0.3 | MANUAL | |
| %I0.4 | INDUCTIVO | PNP, **NC**: en reposo vale 1, al detectar cae a 0 |
| %I0.5 | FOTOELECTRICO | PNP, **NC** |
| %I0.6 | SEN REED S3 | cilindro 1 |
| %I0.7 | SEN REED S4 | cilindro 2 |

### Salidas
| Dirección | Tag | Relé | Acción |
|---|---|---|---|
| %Q0.0 | LUZ START | K1 | luz verde |
| %Q0.1 | LUZ STOP | K2 | luz roja (contacto NC de LUZ START) |
| %Q0.2 | ACCIONAMIENTO VFD | K3 | habilita el variador |
| %Q0.3 | Y1 | K4 | electroválvula del cilindro 1 (metal) |
| %Q0.4 | Y2 | K5 | electroválvula del cilindro 2 (no metal) |

### Marcas internas ("Default tag table")
%M2.0–M2.3 Tag_1 a Tag_4 (biestables SR) · %M3.0 Tag_7 · %M3.5 MARCA MANUAL · %M3.6 MARCA MANUAL LUZ · %M3.7 MARCA MANUAL VFD · %M4.0 Tag_8 · %M4.1 MARCA QUE CONTRAE Y2 · %M4.3 MARCA MANUAL QUE REINICIA · %M4.4 MARCA MANUAL Y1 · %M4.5 MARCA MANUAL Y2 · %M4.6 rESETEA MANUAL QUE REINICIA.

### Networks
- **N1:** AUTO + START → TON 3 s (DB1) → SR Tag_1 → LUZ START. LUZ START + NO MANUAL → TON 3 s (DB2) → SR Tag_2 → VFD. STOP (NC) resetea. M3.6 y M3.7 encienden luz y VFD desde manual.
- **N2:** NO MANUAL + LUZ START + VFD + INDUCTIVO (NC) → SR Tag_3 → Y1. Reset: Reed S3 + INDUCTIVO. M4.4 en paralelo.
- **N3:** igual que N2 con FOTOELECTRICO, Reed S4 → Y2. M4.5 en paralelo.
- **N4:** MANUAL → SR Tag_7 → M3.5. Reset con STOP, NO MANUAL o AUTO.
- **N5–N11:** CTU con PV 1 a 7 (DB4, DB5, DB6, … DB10). N11 (PV=7) activa M4.3, que resetea todos; un TON de 1 s (DB3) genera M4.6 para soltar M4.3.

### Factory I/O
- Versión 2.5.10 Siemens Edition. Driver **Siemens S7-1200/1500**, Host 192.168.0.5.
- **Entradas Bool con offset 10** (%I10.0 en adelante) y **salidas Bool con offset 0** (%Q0.0 en adelante).
- Consecuencia: Factory I/O **solo visualiza**. Sus botones escriben en %I10.x y el programa lee %I0.x, así que no arrancan la banda real. Es una decisión de diseño: el mando es físico, por seguridad.
- Mapeo del driver: BOTON INICIO %I10.0, BOTON PARO %I10.1, SENSOR INDUCTIVO %I10.4, SENSOR FOTO ELECTRICO %I10.5, REED INDUCTIVO %I10.6, REED FOTOELECTRICO %I10.7; LUZ START %Q0.0, LUZ STOP %Q0.1, MOTOR BANDA %Q0.2, Y1 %Q0.3, Y2 %Q0.4. La escena tiene un Emitter que genera piezas.

### Comunicación
Ethernet con el **protocolo S7 de Siemens** (S7comm sobre ISO-on-TCP, RFC 1006, **puerto 102**). **No es Modbus.** Se prueba con `ping 192.168.0.5`.

### Hardware del gabinete
- Q1 TENGEN 3P 32 A (principal) · KM1 Telemecanique CA2 DN31, 10 A, bobina 220 V AC (corte por emergencia) · Q2/Q4 Siemens C16 · F2/Q3 CHINT NXB-63 C25 · Q5 guardamotor Siemens SIRIUS 3RV2011 (10–16 A).
- Fuente MEAN WELL RS-150-24 (24 V DC, 6.5 A). Fuente aparte de 12 V y 30 A para el compresor.
- Variador VEVOR 3 kW, entrada monofásica 220 V, salida trifásica 220 V. Solo recibe encendido y apagado desde el PLC; la velocidad se fija en su potenciómetro.
- Motorreductor **trifásico** SEW-EURODRIVE ¼ HP (0.25 kW, 4 polos).
- 5 relés OMRON MY2-GS-R DC24 (K1–K5).
- 2 electroválvulas SmartSolu 4V110-06, 5/2 monoestables, 24 V DC.
- 2 cilindros de doble efecto Ø12 mm, carrera 100 mm.
- Sensor inductivo SICK PNP 1.4 mm · fotoeléctrico difuso Autonics PNP 10 cm · 2 Reed MK26 (Standex).
- Compresor Goodyear 12 V DC, depósito de 450 psi, presostato Genebre 20–60 psi, unidad de mantenimiento de 135 psi.
- Panel: START verde, STOP rojo, luces verde y roja, maneta de 3 posiciones, hongo de emergencia (se libera girándolo).

### Operación
- Encendido: **Q1 → F2/Q3 → Q2/Q4**. Apagado en orden inverso. El hongo no se usa para apagar.
- Fallas reales de las pruebas: nada enciende → emergencia activada o breakers abajo · variador con `0E` → STOP/RESET · Factory I/O con error `#510` → PLC en STOP (Online → Start CPU) · ping sin respuesta → se escribió 162 en vez de 192.

---

## 4. Correcciones que ya se decidieron

El documento original y las guías tienen errores. La página **siempre usa la versión corregida**:

| Dice el documento | Lo correcto |
|---|---|
| Clasifica ferroso / no ferroso | Clasifica **metálico / no metálico** |
| Protocolo Modbus TCP/IP, codificación ASCII | **Protocolo S7** sobre TCP/IP, puerto 102 |
| Motor monofásico 120 V / 1 kW | Motorreductor **trifásico** SEW ¼ HP |
| Variador monofásico 1.1 kW | Variador 3 kW, 1F → 3F |
| Contactor 24 V DC 25 A | Telemecanique CA2 DN31, 10 A, bobina 220 V AC |
| PLC alimentado en L+ y M a 24 V DC (Guía 1) | CPU alimentado con 120 V AC |
| El último número de la IP de la PC debe ser menor que el del PLC (Guía 1) | Solo debe ser distinto y estar en la misma red |
| Factory I/O es un SCADA que controla | Simulador 3D usado como **visualización**; no controla |

---

## 5. Las 4 guías

Del documento de guías solo están redactadas la **Práctica 1** (configurar el PLC, IP en PC y PLC con TIA Portal: Online access → Update accessible devices → Online & diagnostics → Assign IP address) y la **Práctica 2** (crear la escena en Factory I/O). Las 3 y 4 solo se mencionan en los objetivos. La página las presenta así:

1. **Guía 1. Conectar:** PLC, red e IP.
2. **Guía 2. Dibujar:** escena 3D en Factory I/O; sensores y actuadores.
3. **Guía 3. Programar:** TIA Portal, tags, TON, SR, CTU, descarga al PLC.
4. **Guía 4. Integrar:** driver de Factory I/O, conexión y operación en automático y manual.

Figuras de la Guía 1 que se usan en preguntas: diagrama del CPU (alimentación por L1/N a 120–240 VAC, DI a a 24 VDC, RELAY OUTPUTS), pasos de Windows hasta TCP/IPv4 (adaptador Realtek PCIe GbE), firmware V4.2.2 en Diagnostics → General, y Functions → Assign IP address.

---

## 6. Cómo está hecho `index.html`

Es un solo archivo sin dependencias ni build. Lo único externo son las Google Fonts (Bricolage Grotesque para títulos, Nunito para texto y JetBrains Mono para direcciones). Todo el JS va en un IIFE al final.

Funciona como una app de cinco pestañas. En teléfono la barra de pestañas (`.tabs`) va fija abajo; desde 900 px pasa a la barra superior. Cada pestaña es una `<section class="view">` y solo se muestra una a la vez según el hash de la URL (función `route()`):
- `#inicio` (`#v-inicio`): qué es el módulo, "Cómo usar esta página" con el avance de cada parte y un botón de siguiente paso recomendado (`#nextStep`).
- `#aprender-N` (`#v-aprender`): 6 lecciones, cada una es un `<article class="lesson" data-n data-title data-short>`: 1 cuerpo, cerebro y ventana · 2 los elementos (explorador de componentes, "Sigue la señal" y "¿Quién soy?") · 3 recorrido de una pieza · 4 las 4 guías · 5 cinco detalles que confunden (acordeones) · 6 encendido y fallas. Arriba va el indicador de lecciones y abajo los botones Anterior/Siguiente (`#pager`). Una lección cuenta como vista al pulsar Siguiente o al acertar todas sus preguntas de repaso.
- `#simulador` (`#v-simulador`): simulador, retos y el panel de entradas y salidas. En teléfono la banda (`.sim-top`) se queda fija arriba mientras se usa el gabinete. `cue()` marca con borde amarillo lo que conviene tocar después, y al cumplir un reto sale un aviso (`toast()`).
- `#programar-N`, `#programa-modulo`, `#taller-N` y `#libre` (`#v-programar`): programación en Ladder, en cuatro partes:
  - **Lecciones** (`PL`, 11). Las 1 a 3 son "desde cero" (`intro:true`), para quien no sabe nada: qué es programar un PLC (con el ciclo de scan animado), entradas, salidas y bits (con un lector de direcciones) y de los relés a Ladder (con un recorrido guiado de 6 pasos por un escalón). Las 4 a 11 son los símbolos: primer escalón, NO y NC, serie y paralelo, enclavamiento, Set/Reset, TON, CTU y los NC del módulo. Cada lección tiene `s` (nombre corto), `t`, `life` (ejemplo de la vida diaria), `body`, `mod` (relación con el módulo) y, según el caso, `tryit`, `tip` (error común), `chk` (id de una pregunta de `Q`), `pre`/`html` (bloques interactivos propios), `D` (etiquetas, entradas y salidas del diagrama) y `R` (escalones). Debajo de las lecciones hay un glosario de palabras clave. Explicar con paciencia y desde cero: cada palabra técnica se define la primera vez que aparece.
  - **Programa del módulo** (`MOD_D`, `MOD_R`): los Networks 1 a 4 de la sección 3 dibujados y funcionando en vivo. Cada caja SR se dibuja como una bobina S y una R. Los Networks 5 a 11 solo se explican con una tabla, sin simularlos.
  - **Taller** (`EX`, 6 ejercicios): editor de escalones. Cada ejercicio tiene `goal`, `hint`, `tags` permitidas, `outs` (tipos de salida permitidos), `inputs`, `outputs`, `start` (escalones iniciales) y `tests`. Cada test es `{set:{bit:valor}, w:ms, ex:{bit:valor}, m:'mensaje si falla'}`; `runTests()` simula el tiempo de 20 en 20 ms. Al agregar un ejercicio, probar que una solución correcta aprueba y una incorrecta típica falla.
  - **Zona libre** (`#libre`): hoja en blanco para armar cualquier programa. La paleta (`SYMS`) tiene contacto NO y NC, bobina, Set, Reset, TON, CTU y reset de contador. Los símbolos se arrastran con eventos pointer (sirven con dedo y mouse) a los espacios con `data-drop` (`ser`, `par`, `out` o `new`), o se tocan y luego se toca el espacio. `placeSym()` coloca el símbolo y abre el panel para elegir el bit. Bits disponibles en `SB_TAGS`; entradas y salidas iguales a las del programa del módulo. Se guarda en `lad-libre`. El taller y la zona libre comparten el editor: `ED` es el editor activo (`EDT` para el taller, `EDS` para la zona libre).
- `#practicar` y `#tarjetas` (`#v-practicar`): cuestionario y tarjetas del asesor, una tarjeta a la vez.

### Banco de preguntas
- Está en el array `Q`. Cada pregunta tiene este formato:
  ```js
  {id:'p4', c:'prog', q:'Pregunta', o:['correcta','distractor','distractor','distractor'], a:0, e:'Explicación', s:'Fuente'}
  ```
- `c` es la categoría y tiene que ser una clave de `CATS`: `red` (Guía 1), `fio` (Guía 2), `prog` (Guía 3), `int` (Guía 4), `gen`, `comp`, `sen`, `seg`.
- `a` es el índice de la respuesta correcta en `o`. Por convención la correcta va primero (`a:0`); las opciones se revuelven al mostrarse.
- `e` explica **por qué** es la correcta, en lenguaje sencillo. `s` dice de dónde sale (por ejemplo `'Cap. IV, tabla 4.5'`, `'Guía 1'`, `'Pruebas del módulo'`).
- Los `id` son únicos. Las preguntas de repaso de cada sección los usan en `<div class="check" data-q="ID">`.
- Hoy hay 143 preguntas: red 35, fio 18, prog 29, int 18, gen 8, comp 14, sen 14, seg 7.
- Ids por categoría: `r` Guía 1, `f` Guía 2, `p` Guía 3, `i` o `f` Guía 4, `g` general, `c` componentes, `s` sensores, `e` seguridad.

### Otras piezas
- `COMP`: los 30 componentes del explorador (lección 2). Cada uno es `{id, g, n, ref, w, what, role, sig, fail}`: grupo (clave de `GROUPS`: `energia`, `control`, `mando`, `motor`, `aire`, `sens`, `red`), nombre, modelo, dónde está, qué es, qué hace, con qué se conecta y qué revisar si falla. El juego "¿Quién soy?" usa `what` y la primera frase de `role`, así que `what` no debe incluir el nombre del componente.
- En teléfono (menos de 820 px) la ficha del componente (`#compDetail`) sale como panel desde abajo con `openDetail()`/`closeDetail()`; en pantallas anchas queda fija a un lado de la cuadrícula.
- `CHAINS`: recorridos de "Sigue la señal". Cada paso es el `id` de un componente o `['t','texto']` para un paso que no es componente. En teléfono los pasos se ven en vertical.
- `FLASH`: tarjetas del asesor, cada una es `[pregunta, respuesta]`.
- Motor de Ladder: un escalón es `{g:[[contacto,...],...], o:salida}`. Cada grupo de `g` va en serie y los contactos dentro de un grupo van en paralelo. Contacto: `{t:'NO'|'NC', k}`. Salida: `{t:'coil'|'S'|'R'|'TON'|'CTU'|'RC', k, pt (ms), pv}`. `scan()` revisa los escalones de arriba hacia abajo como el PLC. Las entradas se definen por lo que pasa físicamente: `push` (NO, se mantiene), `ncpush` (STOP: sin presionar vale 1), `sensor` (NC: sin pieza vale 1), `toggle` y `reed` (se activa solo cuando el cilindro de `follow` lleva 0.4 s afuera). `MT` tiene las direcciones y nombres de las etiquetas del módulo.
- `RETOS`: cada reto es `[clave, texto, explicación]`. Se marca llamando a `reto('clave')` desde la lógica del simulador.
- Simulador: el objeto `S` guarda el estado. Las posiciones están en `IND, FOT, C1, C2` (en % del ancho). `paint()` dibuja y `loop()` corre en `requestAnimationFrame`.
- El avance se guarda en `localStorage` con el prefijo `modbanda-`, envuelto en try/catch. La página tiene que funcionar aunque falle. El objeto `progress` tiene `retos`, `checks`, `best`, `flash`, `lessons` y `ladder` (`l`: lecciones de Ladder vistas, `t`: ejercicios resueltos); también se guardan `lesson` (última lección abierta), `theme`, `howtoClosed`, `plesson` y `exercise` (última lección de Ladder y último ejercicio), `lad-<id>` (programa de cada ejercicio del taller), `lad-libre` (programa de la zona libre) `ladderV` (igual que `lessonsV`, para las lecciones de Ladder: al agregar las 3 lecciones desde cero se recorrieron tres números) y `lessonsV` (versión de la numeración de lecciones: al agregar la lección 2, el avance guardado se recorrió un número; si se vuelve a insertar una lección, hay que subir esta versión y migrar igual).
- Hay botón de tema claro/oscuro en la barra superior: pone `data-theme` en `<html>`.
- Colores: todos son tokens CSS en `:root`, con versión oscura en `@media (prefers-color-scheme: dark)` y en `[data-theme="dark"]`. No se usan colores sueltos.
- Estilo propio, que no debe verse genérico: fondo de papel con cuadrícula de puntos como un protoboard (`--dot`), tarjetas y botones tipo calcomanía con contorno de tinta (`--edge`) y sombra sólida (`--shadow`, `--shadow-sm`), y un color plano y vivo por sección: `--c-home` (amarillo), `--c-learn` (azul), `--c-sim` (coral), `--c-prog` (menta) y `--c-prac` (lavanda), cada uno con `-text` (para texto sobre fondo claro) y `-soft` (fondo suave). `route()` pone `data-view` en `<body>` y eso cambia `--petrol` (relleno de lo seleccionado, siempre con texto `--on-fill` encima), `--acc-text` y `--hl-bg`. Nunca usar `--petrol` como color de texto: no se lee sobre fondo claro. Lo que es parte del módulo (banda, cilindros, sensores) usa `--slate`. Verde, rojo y amarillo (`--go`, `--stop`, `--amber`) se reservan para marcha, paro y sensor activo. Evitar degradados genéricos.
- Tiene que verse bien a 400 px de ancho sin scroll horizontal.

---

## 7. Reglas de contenido y estilo

- **Todo en español** de El Salvador, claro y directo, tuteando en plural ("pongan", "revisen").
- Escribir para estudiantes que se sienten perdidos: frases cortas, sin jerga innecesaria, y cuando se use un término técnico, explicarlo.
- **Sin emojis.**
- No inventar datos técnicos. Si algo no está en la sección 3, preguntar antes de escribirlo.
- Mantener las correcciones de la sección 4 en todo el contenido nuevo.
- Las preguntas nuevas tienen que salir de las guías, del documento de la tesis o de las pruebas reales del módulo, y llevar `e` y `s`.
- Distractores plausibles, sin "todas las anteriores" ni "ninguna".
- Hacer cambios puntuales: no reescribir secciones enteras si no se pidió. Explicar el enfoque antes de cambios grandes.

---

## 8. Pendientes conocidos

- **Toques 5 y 6 del modo manual (Networks 9 y 10):** sin confirmar. La página asume "regresa cilindro 1" y "regresa cilindro 2" por la marca M4.1 "MARCA QUE CONTRAE Y2". Hay que verificarlo en TIA Portal.
- **Guías 3 y 4:** no están redactadas; su descripción se infirió de los objetivos.
- Las guías y el documento todavía tienen los errores de la sección 4; la página ya usa la versión correcta.
