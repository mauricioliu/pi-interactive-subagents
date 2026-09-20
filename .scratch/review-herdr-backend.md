REQUEST_CHANGES

## Major

- **M1 — `pi-extension/subagents/cmux.ts:783–789`: procesos Herdr sin deadline.**
  Ambos wrappers pasan únicamente `{ encoding: "utf8" }`: ningún `timeout`, ninguna cancelación.
  Si CLI queda esperando, operaciones síncronas bloquean pi; lectura asíncrona deja `pollForExit`
  esperando sin volver a comprobar abort, sidecar ni `onTick`.
  Repro aislado, CLI mockeado sin resolver: abort a 10 ms; polling seguía pendiente a 100 ms.
  Esto prueba falta de protección del adaptador, no un cuelgue observado de Herdr 0.8.2.
  **Arreglo:** deadline finito en ambos wrappers; preservar `pollForExit` y backends existentes.

## Minor

- **m1 — `pi-extension/subagents/cmux.ts:80–87`: detección comprueba otro ejecutable.**
  Ejecución respeta `PI_HERDR_BIN` o `/usr/bin/herdr`; disponibilidad exige `herdr` en PATH.
  Repro sin invocar CLI: `HERDR_ENV=1`, `PI_SUBAGENT_MUX=herdr`,
  `PI_HERDR_BIN=/usr/bin/herdr`, `PATH=/nonexistent` → disponibilidad `false`, backend `null`,
  aunque ejecutable seleccionado existe. Override no permite instalación fuera de PATH.
  **Arreglo:** comprobar ejecutabilidad del binario resuelto, sin interpolar ruta en shell.

- **m2 — `pi-extension/subagents/cmux.ts:80–83`: prioridad `/usr/bin` asume versión del servidor.**
  Existencia del archivo no implica compatibilidad con sesión activa. Con servidor actualizado
  desde PATH y paquete de sistema antiguo, se elige cliente antiguo pese a existir cliente compatible.
  Regla soluciona esta máquina, pero introduce fallo inverso en otras instalaciones; override tampoco
  aparece en README.
  **Arreglo:** preferir PATH por defecto y documentar `PI_HERDR_BIN=/usr/bin/herdr` para esta flota,
  o resolver explícitamente cliente compatible con sesión.

- **m3 — `test/test.ts:2383–2487`: falta cobertura del límite con procesos.**
  `isHerdrAvailable` solo exige boolean: implementación constante `false` pasa.
  Tests de argv/parser sí fijan contratos útiles, pero ninguno llama dispatch Herdr, lee output realista
  ni comprueba comportamiento ante CLI bloqueado. No detectan M1/m1.
  **Arreglo:** matriz env/binario con caché aislada y prueba de wrappers con child-process mock;
  incluir lectura de centinela y fallo/timeout. Caso pendiente más sensible: centinela en pane estrecho,
  envuelto físicamente y recuperado mediante `recent-unwrapped --lines 5`.

## Verificaciones / límites

- `/usr/bin/herdr --version`: **0.8.2**. Consultados `pane split/run/read/close/rename/send-keys --help`.
  Todos los flags usados existen: `--pane`, `--current`, `--direction right|down`, `--no-focus`,
  `--cwd`, `--source recent-unwrapped`, `--lines`; `esc` es nombre canónico documentado.
- IDs, nombres, cwd y comandos pasan como elementos argv con `execFile*`; sin shell intermedio.
  `sendLongCommand` conserva quoting del script. Sin inyección shell encontrada en adiciones.
- Parser extrae `result.pane.pane_id`; JSON inválido/vacío y estructura incompleta lanzan error.
  Errores de procesos se propagan salvo rename cosmético. Pane muerto sin sidecar puede dejar polling
  reintentando: comportamiento compartido preexistente, no regresión atribuida a este diff.
- `recent-unwrapped` documenta unión de soft wraps en documentación embebida del binario.
  Flag correcto; no verificado extremo a extremo en pane estrecho: no operé ni leí otros panes.
  No garantiza recuperar saltos duros o contenido ya fuera del tail.
- Dispatch previo conserva orden/lógica; no cambios de firmas existentes. `pollForExit` byte-idéntico.
  Detección, override `herdr`, split sin foco, `pane run`, close y renames no-op presentes; README actualizado.
  No existe código even-layout en esta revisión base ni en working tree: nada que saltar.
- No repetí suite 142/142. Pruebas adicionales aisladas, sin modificar archivos del diff.
