# Trend 200 · Panel 80/20 — notas técnicas

Panel móvil (PWA estática) para operar la estrategia **SATOSHI Trend 200 v2** de BTC con reparto 80/20.
Muestra si el indicador está DENTRO o FUERA, qué hacer en cada señal, el tamaño por volatilidad, el reparto
80/20 y la comparación con haber comprado BTC el primer día. Última actualización: 2026-11-14.

## 1. Arquitectura
- `index.html` único y autocontenido: HTML + CSS + JS vanilla. Sin build, sin dependencias, sin CDN ni fuentes externas.
- `manifest.json`, `icon-180/192/512.png`: instalable en la pantalla de inicio (iOS: Compartir → Añadir a pantalla de inicio).
  Icono: ₿ blanca en pixel art (cuadrícula de 32×32, ampliada sin suavizar) sobre fondo naranja `#F2AC3E` con sombra larga diagonal `#EE8833`, estilo retro; PNG a sangre (iOS aplica la máscara).
  Se generó pintando cada píxel de la cuadrícula en un canvas: la ₿ son rectángulos por filas y la sombra extruye cada píxel en diagonal abajo-izquierda.
  Si se cambia, subir `V` en `sw.js` y, en iOS, borrar y volver a añadir la app (iOS guarda el icono al instalarla).
- Los iconos se enlazan con `?v=N` (en `index.html` y `manifest.json`): iOS y Android cachean el icono de la pantalla de inicio y, sin versión, siguen enseñando el antiguo.
  `background_color` del manifiesto = naranja del icono (pantalla de arranque). Al cambiar un icono: subir `?v=` y `V` de `sw.js`.
- `sw.js`: service worker, la app abre sin conexión. Estrategia *stale-while-revalidate* solo para el propio sitio;
  las llamadas a Binance no se cachean. **Al cambiar `index.html`, subir `V` en `sw.js`.**
- Estilo copiado del sistema de SOLO GASTOS (tokens, tarjetas de cristal, dock con burbuja, hojas, animación de confirmación),
  con el naranja de BTC (`#F7931A`) como acento en lugar del verde.
- **Un solo repo público: aquí no hay datos de usuario.** Todo el estado del usuario vive en `localStorage`
  (clave `trend200.v2`) y nunca viaja a ningún servidor ni se sube al repo.

## 2. Lógica de la estrategia (réplica del Pine)
Fuente: `SATOSHI_Trend200_v2_Ciclo_Fib_rev.pine`. El motor (`t200core`, entre `/*T200_START*/` y `/*T200_END*/`) usa solo cierres diarios:
- Entrada: cierre > SMA50 **y** > SMA200 (SMA simples). Salida: cierre < SMA50 **o** < SMA200.
- Señal al cierre de la vela, ejecución en la apertura siguiente.
- **Enfriamiento:** la vela de señal de compra debe abrir ≥ 7 días después de la *ejecución* de la venta (`time - lastV2Exit >= 7 días`,
  con `lastV2Exit = time_close` de la vela de señal de venta). Como la compra se ejecuta al día siguiente, entre la ejecución de la venta y la de la compra
  pasan como mínimo **8 días** (`COOL = 8`). Pasada la espera se compra si la señal sigue activa (no hace falta cruce nuevo).
- **Tamaño por volatilidad:** `min(1, 0,40 / vol)`, `vol` = desviación típica *poblacional* de los 30 últimos rendimientos logarítmicos × √365.
  1x = todo, 0,5x = la mitad. Se fija al entrar (no se reajusta dentro). Tope 1x por defecto (`levCap` del Pine).
- Nivel de la próxima señal: `max(SMA49(últimos 49 cierres), SMA199)` aplicado al cierre de la vela siguiente
  (despeja `c > (S_{n-1}+c)/n`).
- La parte del Pine de ciclo (medidor techo/suelo) y Fibonacci **no** se replica: no afecta a señales ni órdenes.

## 3. Fuentes de datos (en orden)
1. **Binance** (`api.binance.com`, y `data-api.binance.vision` de respaldo): histórico diario BTCUSDT desde 2017-08 (paginado de 1000 en 1000).
   Funciona en GitHub Pages; recalcula el estado con el histórico completo.
2. **SIGNUM** (`INDEX:BTC/USD`, 1D) vía el conector MCP, solo cuando la página corre como artefacto de claude.ai. Pide cierre, SMA50 y SMA200
   de las últimas 100 velas (el indicador `sma` admite tres longitudes: 1, 50, 200) y, aparte, SMA49 y SMA199 para el nivel de señal.
   El estado de partida de la ventana se infiere (`warm`).
3. **Instantánea** embebida (`SNAP`): cierre, SMA50 y SMA200 de 100 velas hasta 2026-11-13. Se usa si no hay red; avisa si tiene ≥ 2 días.
   Para actualizarla, volver a pedir esas 100 velas a SIGNUM y sustituir `SNAP`.
Diferencia a tener en cuenta: SIGNUM usa el precio compuesto `INDEX:BTC/USD`; el Pine usa el gráfico que tenga abierto el usuario.

## 4. Decisiones no obvias
- **Posición registrada ≠ estado del indicador.** `S.pos` es lo que el usuario tiene; el indicador es lo que debería tener. El panel compara ambos
  (`calc()`): «Toca comprar/vender» solo aparece si no coinciden.
- **«BTC por tu cuenta» (`S.manual`):** el usuario mantiene BTC sin señal de compra. No se le avisa de vender; al saltar la compra se le pide
  registrar «Ya compré» sin comprar nada (alinea el panel). Cualquier compra/venta registrada pone `manual=false`.
- **Historial con deshacer:** cada acción guarda el estado previo (`pos, posDate, lastSell, manual`); «Deshacer» lo restaura.
- **Mi cartera:** BTC × precio + USDC + Quantfury = valor actual (`portfolio()`); sobrescribe `curVal` si hay datos.
- **Comparación contra HODL:** `HODL = unidades_BTC × precio_actual`, con `unidades = capital_inicial / precio_inicial + Σ aportaciones`. Las «fotos» (snaps) dibujan las curvas y la caída máxima.
- **Aportaciones de BTC (`S.contribs`):** «He sumado (o restado) BTC» guarda `{fecha, btc, precio}`. Suman a las unidades de HODL y al capital invertido, y las curvas
  y la caída máxima se normalizan por el capital invertido en cada fecha, de modo que meter o sacar BTC no cuenta como ganancia o pérdida de la estrategia.
  Las aportaciones anteriores a la fecha de inicio se ignoran; «Empezar la comparación hoy» las borra.
- **Exportar/importar:** JSON `{app:'trend200', v:2, data:S}`. Exportar muestra el texto (copiar) y ofrece descarga; importar pide confirmación
  y sustituye los datos. Los datos viven solo en el navegador: si se borra, se pierden salvo copia.

## 5. Despliegue (GitHub Pages)
Repo público, rama `main`, Settings → Pages → *Deploy from a branch* → `main` / `/ (root)`. URL: `https://USUARIO.github.io/trend200/`.
Tras cada cambio de `index.html`: subir `V` en `sw.js`; en el móvil, cerrar y reabrir la app (el service worker actualiza en segundo plano).
Una copia sin service worker se publica también como artefacto de claude.ai (allí no hay `fetch` a Binance: se usa SIGNUM o la instantánea).

## 6. Bugs y lecciones
- `const top` chocaba con `window.top` → `Identifier 'top' has already been declared` y el panel no arrancaba. Renombrado `topbar`.
- La plantilla del estado «espera» evaluaba `addDays('')` aunque no hubiera venta → `Invalid time value`. `addDays`/`fdate` ahora toleran vacío.
- El panel dejaba comprar un día antes que el Pine (contaba 7 días desde la ejecución de la venta hasta la *ejecución* de la compra, y son hasta la *señal*).
- Los artefactos de claude.ai bloquean `fetch` a otros hosts: por eso existen las vías SIGNUM e instantánea.
- Formato `es-ES` de `toLocaleString` no agrupa miles en números de 4 cifras (5200): se usa `nf()` propio.
- Un `localStorage` vacío o bloqueado no debe romper la app: `store` con respaldo en memoria.
- Tras cambiar el icono, el móvil seguía mostrando el anterior (fondo oscuro): el icono de la pantalla de inicio se guarda al instalar y el navegador cachea el PNG. Solución: versionar la URL (`?v=N`), reinstalar la app y comprobar el PNG abriendo `/icon-180.png` en el navegador.
- `gh repo create` por API fue denegado en la sesión de desarrollo: el repo se crea a mano en GitHub y luego se empuja.
