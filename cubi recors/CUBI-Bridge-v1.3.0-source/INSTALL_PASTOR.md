# CUBI Bridge — Guía de instalación (Pastor)

**Tiempo total:** 2 minutos. Sin terminal. Sin Node. Sin .bat.

---

## Paso 1 — Descargar

Desde la web del estudio, abrí `/lab` → click en **"Descargar instalador (.exe)"**. Se baja un archivo único:

**`CUBI-Bridge-Setup.exe`** (~80 MB)

---

## Paso 2 — Instalar

Doble click en `CUBI-Bridge-Setup.exe`.

> **Windows SmartScreen** puede mostrar "Windows protegió tu PC".
> Hacé click en **"Más información"** → **"Ejecutar de todas formas"**.
> Esto pasa solo la primera vez. (El instalador no está firmado todavía — completamente seguro, es solo que el certificado de firma cuesta $200/año y por ahora no lo compramos).

Aparece un asistente en español:

1. Aceptá la licencia.
2. Dejá la carpeta de instalación por defecto (`C:\Users\<tu_usuario>\AppData\Local\Programs\cubi-bridge`).
3. Click **"Instalar"** → se instala en ~10 segundos.
4. Dejá marcado **"Ejecutar CUBI Bridge"** y click **"Finalizar"**.

Listo. Vas a ver un ícono nuevo en la **bandeja del sistema** (cerca del reloj, abajo a la derecha): un círculo con el logo de CUBI Records.

---

## Paso 3 — Emparejar con tu cuenta

1. En el navegador, abrí `/lab` → expandí la tarjeta **"CUBI Bridge"** → click **"Conectar mi Estudio"**.
2. Aparece un código de 6 dígitos (vigente por 5 minutos).
3. Click derecho en el ícono de la bandeja → **"Emparejar..."**.
4. Pegá el código → **"Conectar"**.
5. La tarjeta en la web se pone verde con un pulso 🟢 — **estás en línea**.

---

## Qué hace el Bridge a partir de ahora

- Arranca solo cada vez que prendés la PC (puede apagarse desde el menú del tray).
- Cuando abrís Cubase y empezás a sonar audio en el master, el Coproductor IA en `/coproductor-ia` ve tus métricas live (LUFS, picos, espectro) y puede responder en tiempo presente.
- **READ-ONLY total:** nunca toca tu proyecto, nunca cambia parámetros de Cubase. Solo observa.
- **El audio nunca sale de tu PC** — solo metadata numérica (~1 KB/seg).
- Se actualiza solo cuando hay versión nueva (te aparece una notificación al reiniciar).

---

## Menú del tray (click derecho en el ícono)

- 🎚️ Estado actual (En línea / Desconectado)
- 🔗 Emparejar... (si no está conectado)
- 🚪 Desemparejar
- 🎧 Mostrar/Ocultar HUD flotante (overlay que muestra métricas sin sacarte de Cubase)
- 🔄 Buscar actualizaciones
- 🚀 Iniciar al encender PC (checkbox)
- ❌ Salir

---

## Si algo no funciona

| Síntoma | Solución |
|---|---|
| No aparece el ícono en la bandeja | Buscá en el ícono ⌃ (mostrar íconos ocultos). Si no está, abrí Inicio → "CUBI Bridge" |
| La tarjeta en `/lab` sigue roja | Click derecho en tray → "Reconectar". O regenerá código nuevo en `/lab` |
| El Coproductor no ve métricas | Asegurate de que Cubase esté sonando audio (no en pausa) |
| Quiero desinstalar | Inicio → Configuración → Apps → buscar "CUBI Bridge" → Desinstalar |

Si nada funciona, mandale captura al equipo técnico desde `/lab` → "Reportar problema".

---

**Versión de esta guía:** alineada con CUBI Bridge v1.3.0+ (instalador .exe NSIS).
