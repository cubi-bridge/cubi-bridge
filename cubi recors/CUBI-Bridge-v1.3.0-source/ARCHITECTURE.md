# 🏛 CUBI Bridge — Arquitectura y Roadmap

> Documento vivo. Cada fase del bridge respeta los mismos seams para que
> Bridge 2 y Bridge 3 no requieran refactors invasivos.

## Principios inviolables

1. **READ-ONLY**. Nunca escribir al DAW, nunca mover un parámetro de Cubase.
2. **Audio nunca sale del PC**. Solo metadata derivada.
3. **El bridge es opcional**. Si no está, el LAB sigue funcionando con análisis de archivos.
4. **ADN CUBI**: cero wizards, cero modales, cero unidades técnicas en UI emocional.

## Capas del sistema

```
┌─────────────────────────────────────────────────────────────┐
│                    Cubase + plugins                          │
│                          ▼ (observación pasiva)              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ CUBI Bridge (Electron tray app)                        │  │
│  │  ┌─────────────────┐  ┌─────────────────┐              │  │
│  │  │ Capture Engine  │  │ Future: Plugin  │              │  │
│  │  │ (WASAPI loop)   │  │ Companion VST3  │  ...         │  │
│  │  └────────┬────────┘  └────────┬────────┘              │  │
│  │           │ metrics            │ inserts                │  │
│  │           ▼                    ▼                        │  │
│  │  ┌────────────────────────────────────┐                │  │
│  │  │ WS Gateway (token, ping, reconnect)│                │  │
│  │  └────────────────┬───────────────────┘                │  │
│  └───────────────────┼─────────────────────────────────────┘  │
│                      │ wss://                                  │
│  ┌───────────────────▼─────────────────────────────────────┐  │
│  │ Server (apocalipsisconcafe.com)                         │  │
│  │  ┌─────────────────┐  ┌────────────────────┐           │  │
│  │  │ Bridge Gateway  │  │ Observation Engine │           │  │
│  │  └────────┬────────┘  └─────────┬──────────┘           │  │
│  │           │ socket.io rooms     │                       │  │
│  │  ┌────────▼─────────────────────▼────────┐             │  │
│  │  │ Coproductor IA (contexto + chat)      │             │  │
│  │  └───────────────────┬───────────────────┘             │  │
│  └──────────────────────┼─────────────────────────────────┘  │
│                         │                                     │
│  ┌──────────────────────▼─────────────────────────────────┐  │
│  │ Web /lab (BridgeConnect + BridgeLiveHUD + chat)        │  │
│  └────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Seams ya preparados

### 1. Capabilities en `hello`

El bridge declara qué puede observar:

```js
ws.send({
  type: "hello",
  capabilities: ["capture-master"],  // Bridge 1
  // Bridge 2 agregará "plugin-chain"
  // Bridge 3 agregará "project-parse"
})
```

El servidor guarda capabilities en `ActiveBridge.capabilities`. La UI puede consultar
qué tiene disponible el estudio del usuario y mostrar funcionalidad acorde.

### 2. Tipo `MetricSample` extensible

El Observation Engine consume samples con shape conocida pero ampliable:

```ts
interface MetricSample {
  ts, lufs, truePeak, rms, crest, bands?,
  // Bridge 2 agregará:  plugins?: PluginChain
  // Bridge 3 agregará:  project?: ProjectMeta
}
```

Nuevas reglas se agregan a `evaluateRules()` sin tocar el resto.

### 3. Rooms socket.io privados

Toda telemetría va a `user:${userId}` — nunca se filtra entre usuarios. Cualquier
nuevo evento (`bridge:plugin-loaded`, `bridge:project-info`) debe usar la misma sala.

### 4. Feature flags por capability

```ts
if (active.capabilities?.includes("plugin-chain")) {
  // mostrar UI de inserts en el HUD
}
```

## Roadmap de fases

### Bridge 0 — Foundations ✅
Pairing + WSS gateway + tray app + token persistente + rooms privados + rate limit.

### Bridge 1 — Audio del master (en curso) ✅
Capture v2 (WASAPI loopback + 8 bandas + crest + TP4x) + Observation Engine (7 reglas
contextuales) + HUD live (sparklines + bandas + feed de observaciones) + auto-restart
de captura + telemetría CPU/RAM.

### Bridge 2 — Companion VST3 ✅ ARQUITECTURA LISTA · BINARIO PENDIENTE
**Problema**: el master sólo nos da la SUMA. No vemos qué plugins están cargados,
qué presets, en qué orden. El Coproductor no puede decir "el Pro-L2 está
martillando 4 dB, por eso el crest cayó".

**Solución implementada (lado web/desktop)**:

Toda la infraestructura está activa. Falta sólo el binario VST3 (C++) que el
Pastor compilará externamente con Steinberg VST3 SDK.

```
Cubase Master Bus
    │ inserta una vez
    ▼
┌──────────────────────────┐
│ CUBI Companion VST3      │  (C++, externo a este repo)
│ - lee insert chain        │
│ - GR, threshold, bypass   │
└──────────┬───────────────┘
           │ TCP 127.0.0.1:49162 (JSONL, READ-ONLY)
           ▼
┌──────────────────────────┐
│ CUBI Bridge (Electron)    │  cubi-bridge/main.js
│ - net.createServer        │  ➜ startLocalIpcServer()
│ - currentCapabilities()   │  ➜ ["capture-master","plugin-chain"]
└──────────┬───────────────┘
           │ WS msg {type:"plugins"|"capabilities"}
           ▼
┌──────────────────────────┐
│ Server (bridge.ts)        │
│ - pluginChains Map        │  TTL 5min
│ - getBridgeLiveContext()  │  expone pluginChain
│ - emite bridge:plugins    │  socket.io sala privada
└──────────┬───────────────┘
           │
   ┌───────┴────────┐
   ▼                ▼
HUD (oro)      Coproductor IA
PluginChain    bloque "CADENA DEL MASTER"
Strip           cruzado con métricas live
```

#### Protocolo IPC local (bridge desktop ⇄ VST3)

- **Transporte**: TCP loopback `127.0.0.1:49162`, sólo loopback (rechazado fuera).
- **Encoding**: JSON-line (un JSON por línea, terminada en `\n`, ≤64 KB).
- **Dirección**: SIEMPRE VST3 → bridge. El bridge NUNCA envía comandos al VST3.
  Esta unidireccionalidad es la garantía READ-ONLY a nivel de transporte.

**Mensaje único soportado** (`type: "plugin-chain"`):
```json
{
  "type": "plugin-chain",
  "ts": 1737654000000,
  "bus": "master",
  "plugins": [
    {
      "slot": 1,
      "name": "FabFilter Pro-Q 3",
      "vendor": "FabFilter",
      "category": "eq",
      "bypass": false,
      "preset": "Master HPF 30Hz"
    },
    {
      "slot": 5,
      "name": "FabFilter Pro-L 2",
      "vendor": "FabFilter",
      "category": "limiter",
      "bypass": false,
      "thresholdDb": -3,
      "ceilingDb": -1.0,
      "gainReductionDb": 4.2,
      "oversampling": 4
    }
  ]
}
```

**Categorías**: `eq | comp | limiter | reverb | delay | saturation | analyzer | gate | imager | other`

**Cadencia recomendada**: 1 snapshot cada 1–3 s. No más rápido (no aporta) ni más
lento (el HUD parece colgado). El bridge no exige frecuencia fija.

#### Protocolo WS (bridge desktop ⇄ server)

Dos nuevos tipos de mensaje:

- `{type:"plugins", ts, bus, plugins[]}` — reenvío del snapshot del VST3.
- `{type:"capabilities", capabilities:["capture-master","plugin-chain"]}` — el
  desktop avisa al server cuando el VST3 se conecta/desconecta del puerto IPC
  local, sin esperar a un nuevo HELLO.

El server emite a la sala `user:${userId}`:

- `bridge:plugins` (objeto `PluginChainSnapshot` o `null` al desconectar)
- `bridge:capabilities` (`{capabilities:string[]}`)

#### Simulador (sin VST3 real)

Para probar todo el flujo de punta a punta antes de tener el binario:

```bash
# 1. Arranca el bridge desktop normalmente
# 2. En otra terminal:
node cubi-bridge/vst3-simulator.js
```

El simulador emite cada 2 s un snapshot realista (Pro-Q 3 → smart:EQ 3 →
Pro-C 2 → Ozone Imager → Pro-L 2 → Youlean) con GR variando entre 2–6 dB.
Sirve para validar el HUD (`PluginChainStrip`), el bloque "CADENA DEL MASTER"
del Coproductor, y el cruce métricas + plugins en las respuestas.

#### Implementación del VST3 Companion (trabajo C++ externo)

El dev que compile el binario implementa:

1. **VST3 plugin "fx" no-procesador** — pasa audio sin tocar (`processReplacing`
   sólo copia in→out). El plugin existe sólo para tener un slot insertado.
2. **`IComponentHandler` callbacks** — Cubase entrega la lista de plugins del
   bus padre a través de las APIs de Steinberg. Documentación: VST3 SDK,
   módulo `pluginterfaces/vst/ivstcomponent.h`.
3. **TCP client** — `boost::asio` o `std::net` (C++20). Conecta a
   `127.0.0.1:49162`, reintenta con backoff. Si el bridge no está, el VST3
   sigue siendo transparente (no rompe la mezcla).
4. **Throttle** — emite snapshot al cambiar la cadena O cada 2 s, lo que ocurra
   primero. Diff opcional para enviar sólo deltas (no hace falta para MVP).
5. **GR lectura** — para Pro-L2/Pro-C2/Ozone donde el SDK lo permite, leer el
   parámetro de gain reduction. Para los que no, omitir el campo.
6. **NUNCA**: escribir parámetros, automatizar, recibir comandos. La conexión
   TCP es write-only desde el VST3.

#### Reglas futuras del Observation Engine que destrabará Bridge 2

Cuando el binario esté listo, el server puede agregar (en `evaluateRules()`):

- `LIMITER_HAMMER_CONFIRMED` — TP_UNSAFE + crest cayendo + limiter GR > 4 dB
- `BYPASSED_EQ_VS_SUB_BUILDUP` — SUB_BUILDUP activo + EQ del master en bypass
- `OZONE_MAX_BITING` — Ozone Maximizer presente + crest < 6 dB
- `STACKED_LIMITERS` — más de un plugin categoría `limiter` en el master

### Bridge 3 — Parser .cpr parcial (futuro lejano)
**Problema**: queremos contexto del proyecto sin necesidad de que el Pastor reproduzca.

**Solución**: parser parcial del `.cpr` de Cubase. **NO** intentar leerlo completo
(formato propietario, cambia entre versiones, alto riesgo de ruptura). Solo extraer:
- Track count + nombres
- Marcadores + estructura
- Tempo map + signatura
- Buses principales

El bridge expone `capability: "project-parse"`. El Coproductor puede preguntar
"¿cuántas pistas tienes?" o "¿en qué compás vas?" sin que el Pastor reproduzca.

**Riesgo alto**: cada update mayor de Cubase puede romper el parser. Decisión:
**diferir hasta validar Bridge 2 con uso real**.

## Cómo agregar una nueva capability

1. **Desktop**: agregar a `BRIDGE_CAPABILITIES` en `main.js`.
2. **Desktop**: implementar el productor (nueva ventana oculta o módulo nativo).
3. **Desktop**: enviar mensajes WS con `type: "<nuevo-tipo>"`.
4. **Server**: agregar handler en `setupBridgeWebSocket` que valide y reenvíe a `io.to('user:'+userId)`.
5. **Server (opcional)**: extender `MetricSample` y agregar reglas al Observation Engine.
6. **Web**: agregar listener en `BridgeLiveHUD` o componente nuevo según el feature.

## Cómo agregar una nueva regla de observación

1. Editar `evaluateRules()` en `server/bridge.ts`.
2. Usar el helper `emit(code, severity, text)` — el cooldown 30s y la sala privada se manejan solos.
3. El frontend la renderiza automáticamente en el feed del HUD (sin código nuevo).

## Cuándo NO usar el bridge

El bridge es para **observación continua mientras se mezcla**. Para análisis ad-hoc de
archivos exportados (mezcla terminada, comparar versiones, recibir stems de terceros),
seguir usando el LAB con upload directo — es 100% local en el navegador y no requiere
nada instalado.
