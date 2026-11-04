# Proyecto 3 – Pochoco SoC: de RTL a GDSII (LibreLane + GF180MCU)

**Integrantes:** Constanza Bustamante, Elena Durán y Mario Messen.

**Punto de partida:** el repositorio del Proyecto 2 del grupo (RTL modificado, `game.s` y assembler propio). El juego de reflejos sobre Espino (RV32E) se adapta desde la FPGA iCE40 (Go Board) a un **macro digital de celdas estándar** en GF180MCU. Los puertos de `asic_top` son terminales lógicos del macro, no pads de I/O.

## Versiones del entorno

| **Elemento** | **Versión / revisión** |
|---|---|
| LibreLane | **3.0.17**, flake `github:librelane/librelane/64ef80802d3d189b476636d2a6b17888c03304ee` |
| PDK | **gf180mcuD**, revisión de `open_pdks` `54435919abffb937387ec956209f9cf5fd2dfbee`, gestionado por `ciel` en `~/.ciel` |
| Biblioteca estándar | **gf180mcu_fd_sc_mcu7t5v0** (7 tracks, 5 V) |
| Herramientas del flujo | Yosys, OpenROAD, KLayout, Magic, Netgen, Verilator e Icarus Verilog, disponibles en el entorno de LibreLane/Nix |
| Python | Python 3 para el assembler, los scripts y las pruebas unitarias |

La corrida física ROM de referencia fue ejecutada con `CLOCK_PERIOD = 100 ns` y `FP_CORE_UTIL = 40 %`. La revisión del PDK y la versión de LibreLane indicadas arriba corresponden al entorno registrado para esa ejecución.

## Estructura del repositorio

| **Ruta** | **Contenido** |
|---|---|
| `rtl/` | RTL para ASIC: `asic_top.v`, `pochoco_soc.v`, núcleo Espino, periféricos y las dos estrategias de memoria |
| `rtl/pochoco_rom.v`, `rtl/pochoco_dram.v` | Estrategia 1: ROM generada desde `game.s` más RAM de datos reducida (`MEM_ROM`) |
| `rtl/pochoco_ram_ff.v` | Estrategia 2: memoria unificada implementada con flip-flops (`MEM_UNIFIED_FF`) |
| `rtl/pochoco_cycle_counter.v` | Contador de ciclos del Proyecto 2 separado en un módulo para medir su contribución al área |
| `sw/game.s` | Programa del juego con constantes temporales derivadas de `CLK_HZ` |
| `sw/game_25mhz_p2.s` | Versión del programa del Proyecto 2, a 25 MHz, conservada como referencia |
| `sw/build/` | Archivos generados para la implementación normal: `game.hex`, `game.symbols` y `game_program.vh` |
| `sw/build_sim/` | Archivos generados para simulación con `CLK_HZ=2000` (0,1 s = 200 ciclos; 3 s = 6000 ciclos) |
| `tools/assembler.py` | Assembler extendido con `.equ`, `li`, `-D` y `--verilog` |
| `tests/` | Pruebas unitarias del assembler |
| `tb/tb_game.v` | Testbench común para simulación RTL y post-síntesis |
| `librelane/{rom,rom_opt,ff}/config.json` | Configuraciones de las arquitecturas. `rom_opt` incluye la optimización `OPT_PERIPH_READ` |
| `librelane/pin_order.cfg` | Distribución de terminales del macro |
| `librelane/p2_*`, `p2_reference/` | Configuraciones y evidencia de las etapas de adaptación del RTL del Proyecto 2 (sección 3.1) |
| `scripts/` | Scripts para ejecutar LibreLane, hacer barridos, simular, recopilar métricas y analizar síntesis |
| `results/` | CSV/Markdown de métricas y gráficas generadas a partir de las corridas |
| `artifacts/final_rom/` | GDS, netlist, métricas, log y reportes conservados de la corrida física ROM `T100_U40` |
| `artifacts/comparison/` | Resumen de métricas, gráficas y logs de simulaciones RTL y post-síntesis ROM/FF |
| `evidence/` | Logs e informes de evidencia de etapas específicas del proyecto |
| `docs/` | Informe del proyecto |

## 1. Instalar LibreLane (Linux o WSL2)

LibreLane se ejecuta mediante Nix. En Ubuntu o Ubuntu en WSL2 se requiere que systemd esté habilitado y que el servicio de Nix pueda ejecutarse.

Instalación de Nix (solo si todavía no está instalado):

```bash
sh <(curl -L https://nixos.org/nix/install) --daemon --yes
```

Después de instalarlo, abrir una nueva terminal. Configurar las flakes y la caché de binarios de LibreLane en `/etc/nix/nix.conf` (agregar estas líneas una sola vez):

```text
extra-experimental-features = nix-command flakes
extra-substituters = https://nix-cache.fossi-foundation.org
extra-trusted-public-keys = nix-cache.fossi-foundation.org:3+K59iFwXqKsL7BNu6Guy0v+uTlwsxYQxjspXzqLYQs=
```

Reiniciar el servicio:

```bash
sudo systemctl restart nix-daemon
```

Comprobar la versión fijada de LibreLane:

```bash
nix shell github:librelane/librelane/64ef80802d3d189b476636d2a6b17888c03304ee --command librelane --version
```

La salida esperada identifica LibreLane **3.0.17**. Los scripts del repositorio (`scripts/env.sh`) utilizan `librelane` si está disponible en `PATH` y, en caso contrario, recurren a Nix usando la revisión fijada. El PDK se descarga y administra mediante `ciel` en `~/.ciel` durante la primera ejecución.

Si Nix informa problemas de permisos del daemon en WSL, comprobar que el usuario pertenezca al grupo `nix-users` y volver a iniciar la sesión de WSL. En WSL2 también se debe tener systemd habilitado.

> En WSL conviene mantener el repositorio en el sistema de archivos de Linux (`~/...`) y no en `/mnt/c`, porque LibreLane genera muchos archivos pequeños y el flujo es considerablemente más lento sobre el montaje de Windows.

## 2. Regenerar el programa desde `game.s`

Desde la raíz del repositorio:

```bash
cd ~/pochoco_asic_project_trabajo
mkdir -p sw/build
python3 tools/assembler.py sw/game.s -o sw/build/game.hex \
    --symbols sw/build/game.symbols \
    --verilog sw/build/game_program.vh
```

La salida esperada informa que `sw/game.s` genera **136 palabras**. Ejecutar las pruebas automatizadas:

```bash
python3 -m unittest tests.test_assembler -v
```

En la ejecución realizada, las **21 pruebas terminaron correctamente** (`OK`).

`sw/build/game_program.vh` define `GAME_WORDS` y la función `game_word(idx)`. La herramienta de síntesis utiliza ese contenido para implementar la ROM de la estrategia `MEM_ROM` o para establecer la imagen inicial de la memoria unificada de la estrategia `MEM_UNIFIED_FF`. Cada modificación de `game.s` requiere regenerar los archivos antes de volver a sintetizar o simular.

### Extensiones del assembler

- `.equ NOMBRE, expresión`: define constantes con nombre y permite expresiones aritméticas enteras (`+`, `-`, `*`, `/`, `%` y paréntesis). La división es entera.
- `li rd, expresión`: pseudo-instrucción que se expande a `lui` y `addi` (dos palabras).
- `-D NOMBRE=valor`: permite sobrescribir desde la línea de comandos una constante definida mediante `.equ`.
- `--verilog archivo.vh`: genera el contenido Verilog de la ROM, incluyendo `GAME_WORDS` y `game_word(idx)`.

La compatibilidad con el programa del Proyecto 2 se comprobó ensamblando con `CLK_HZ=25000000` y comparando el resultado con el `game.hex` original; la prueba automatizada correspondiente verifica la igualdad bit a bit.

## 3. Ejecutar las corridas de LibreLane

Antes de ejecutar, situarse en la raíz del repositorio. Los comandos usan `bash` explícitamente para no depender del permiso de ejecución de los scripts.

### Corrida física ROM de referencia

```bash
cd ~/pochoco_asic_project_trabajo
bash scripts/run.sh rom T100_U40
```

Esta es la corrida física de referencia conservada como entrega final: `CLOCK_PERIOD = 100 ns` y `FP_CORE_UTIL = 40 %`. Sus archivos principales se encuentran en:

```text
librelane/rom/runs/T100_U40/final/gds/asic_top.gds
librelane/rom/runs/T100_U40/final/nl/asic_top.nl.v
librelane/rom/runs/T100_U40/final/metrics.json
```

Se conservaron copias junto con los reportes y el log de consola en `artifacts/final_rom/`.

### Otras arquitecturas y variantes

Para volver a ejecutar la variante optimizada y la arquitectura de memoria unificada FF con la misma configuración objetivo:

```bash
bash scripts/run.sh rom_opt T100_U40
bash scripts/run.sh ff T100_U40
```

`rom_opt` incorpora el cambio RTL `OPT_PERIPH_READ`. En los experimentos registrados, su menor período **probado** que cerró timing fue 58 ns; no se probó 57 ns, por lo que no se afirma que 58 ns sea el límite absoluto.

La comparación FF se conserva en `results/runs.csv` y `artifacts/comparison/`. La fila histórica `ff,T100_U40` registra que el flujo terminó, pero `closes=no` debido a dos redes con violaciones de antena. En la copia de trabajo actual no se conservan el GDS ni todos los informes físicos originales de esa corrida FF; para regenerarlos es necesario volver a ejecutar el comando FF anterior.

### Recopilar métricas y reporte de síntesis

```bash
python3 scripts/collect.py
python3 scripts/synth_report.py rom T100_U40
```

`collect.py` recorre las corridas disponibles en `librelane/{rom,rom_opt,ff}/runs/` y actualiza `results/runs.csv` y `results/runs.md`. El campo `closes` aplica los criterios definidos por el propio script: flujo completo, DRC Magic, LVS, antena y DRC de ruteo en cero, WNS de setup/hold no negativo y TNS de setup/hold igual a cero en los corners TT y SS.

`synth_report.py` genera el resumen de las celdas más utilizadas y el desglose jerárquico de área de la corrida indicada.

### Barridos de período y utilización

Los barridos registrados para la arquitectura ROM original se pueden reproducir con:

```bash
bash scripts/sweep.sh period rom 100 80 70 65 63 62 60 55
bash scripts/sweep.sh util rom 40 50 60 70 80
```

Para la variante optimizada se registraron corridas de período a 100, 62, 60 y 58 ns:

```bash
bash scripts/sweep.sh period rom_opt 100 62 60 58
```

El caso `FP_CORE_UTIL=80 %` no completó el placement detallado; su área preliminar no se considera un layout final válido. El 70 % fue el área mínima entre las configuraciones de utilización probadas que sí completaron el flujo. La corrida de referencia y signoff, sin embargo, es `rom/T100_U40`.

Los resultados de las corridas se guardan en `librelane/<diseño>/runs/<tag>/`. Cada ejecución genera su propio directorio con configuración resuelta, logs, métricas, reportes y, si completa el flujo físico, los archivos finales.

## 4. Simulaciones

### Simulación RTL

La simulación funcional utiliza una versión temporalmente reducida del programa para evitar esperar los tres segundos reales. Se ensambla el mismo `game.s` con `CLK_HZ=2000`: una décima corresponde a 200 ciclos y la espera de preparación de tres segundos corresponde a 6000 ciclos.

Generar el programa de simulación:

```bash
mkdir -p sw/build_sim tb/out
python3 tools/assembler.py sw/game.s -o sw/build_sim/game.hex \
    -D CLK_HZ=2000 --verilog sw/build_sim/game_program.vh
```

Ejemplo de compilación y ejecución RTL para ROM:

```bash
source scripts/env.sh
IVERILOG=$(find_tool iverilog iverilog)
VVP=$(find_tool vvp iverilog)
"$IVERILOG" -g2005 -DMEM_ROM -I sw/build_sim \
    -o tb/out/rtl_rom.vvp rtl/*.v tb/tb_game.v
"$VVP" -n tb/out/rtl_rom.vvp +LOG=tb/out/rtl_rom.log
```

Para simular la memoria unificada FF, cambiar `-DMEM_ROM` por `-DMEM_UNIFIED_FF` y el nombre del archivo de salida.

En las pruebas realizadas, ambas arquitecturas completaron diez respuestas correctas, detectaron una respuesta incorrecta y reiniciaron la ronda. Las mediciones fueron 269, 353, 482, 196, 625, 311, 793, 426, 168 y 538 ciclos; el testbench registró cero accesos a la RAM de datos durante esa partida.

### Simulación RTL versus netlist post-síntesis

Para ejecutar la comparación de ROM:

```bash
bash scripts/gate_sim.sh rom
```

Para FF:

```bash
bash scripts/gate_sim.sh ff
```

El script prepara la síntesis de `asic_top`, ejecuta la simulación RTL y la simulación del netlist post-síntesis con modelos funcionales de las celdas GF180, sin retardos SDF anotados, y compara los logs de eventos y ciclos medidos. En las ejecuciones realizadas, ROM y FF obtuvieron `RESULT: PASS (errors=0)`; en cada arquitectura, los logs RTL y netlist fueron idénticos en sus 47 eventos registrados.

La comparación es RTL versus netlist **dentro de cada arquitectura**. No debe interpretarse como una comparación física entre ROM y FF.

## 5. Resultados y limitaciones que deben tenerse presentes

- La corrida física ROM de referencia `rom/T100_U40` completó el flujo. Magic DRC reportó cero errores, Netgen informó que los circuitos coincidían de forma única y el reporte de antena posterior al ruteo detallado no registró redes ni pines con violaciones.
- La etapa KLayout DRC fue omitida porque `KLAYOUT_DRC_RUNSET` no estaba configurado. El resultado XOR entre los GDS de Magic y KLayout fue de cero diferencias, pero este XOR no equivale a ejecutar KLayout DRC.
- El checker de Power Grid informó resultado despejado y la generación PDN indicó que las formas de VDD y VSS estaban conectadas. En una etapa intermedia posterior, `RepairDesignPostGPL` emitió una advertencia de dos redes flotantes (VDD y VSS); se conserva como advertencia intermedia del flujo.
- El análisis IR drop reportó una caída porcentual aproximada de 0,01 %, pero LibreLane advirtió que `VSRC_LOC_FILES` no estaba configurado. Por ello, el resultado es estimativo y no constituye una validación completa de alimentación para un chip destinado a fabricación.
- En el corner lento se registraron advertencias eléctricas de max slew y max cap. La métrica `max_slew_viol_ss` de la corrida ROM de referencia registró 2532 violaciones. Estas advertencias no forman parte del criterio personalizado `closes` de `scripts/collect.py`, pero deben considerarse una limitación eléctrica documentada.
- La corrida física FF registrada completó el flujo, pero `closes=no` porque quedaron dos redes con violaciones de antena. Sus métricas se conservan en `results/runs.csv`; en la copia de trabajo actual no se mantienen todos los artefactos físicos originales de esa ejecución.

## 6. Terminales del macro

| **Borde** | **Terminales** | **Motivo** |
|---|---|---|
| Oeste | `clk`, `rst_n` | Reloj y reset entran juntos por un lado |
| Sur | `btn[3:0]` | Entradas del usuario, agrupadas |
| Norte | `led[3:0]`, `seg1[6:0]`, `seg2[6:0]` | Salidas visibles agrupadas |
| Este | `spi_sclk`, `spi_mosi`, `spi_cs_n`, `spi_miso` | El bus SPI se mantiene junto |

Son terminales lógicos en metal: no incluyen protección ESD, adaptación de nivel ni capacidad de corriente para manejar directamente los dispositivos externos. En un chip completo se necesitarían celdas pad de I/O, protección ESD y drivers adecuados. El diseño de este proyecto corresponde al macro digital, no al chip completo encapsulado.

## 7. Mapa de memoria y periféricos

| **Dirección** | **Uso** |
|---|---|
| `0x00000000`–`0x0000021F` | Programa: 136 palabras de 32 bits (ROM o memoria unificada) |
| RAM de datos | `MEM_ROM`: 4 palabras, conectadas al puerto de datos. `MEM_UNIFIED_FF`: palabras 136–139 de la memoria unificada |
| `0x80000000` | Display (2 dígitos BCD) |
| `0x80000004` | LEDs (escribir un valor one-hot captura la marca de tiempo) |
| `0x80000008` | Botones |
| `0x8000000C` | Contador de ciclos |
| `0x80000010` | Marca de tiempo del objetivo |

## 8. Artefactos conservados

La corrida física ROM de referencia y sus informes se guardan en `artifacts/final_rom/`:

- `asic_top.gds`: GDS final.
- `asic_top.nl.v`: netlist final.
- `metrics.json`: métricas de la corrida.
- `run_rom_T100_U40_console.log`: log de consola completo.
- `reports/drc.magic.rpt`, `reports/lvs.netgen.rpt`: informes DRC Magic y LVS.
- `reports/antenna.rpt`, `reports/antenna_summary.rpt`: informes de antena posteriores al ruteo detallado.
- `reports/VDD-grid-errors.rpt`, `reports/VSS-grid-errors.rpt`: archivos de errores de la generación PDN.
- `reports/openroad-repairdesignpostgpl.log`: registro que conserva la advertencia intermedia de redes flotantes.

La carpeta `artifacts/comparison/` conserva `runs.csv`, `runs.md`, el reporte de síntesis ROM, las gráficas de período y utilización y los logs de las simulaciones RTL y post-síntesis de ambas arquitecturas.
