# CUBI Bridge — Cómo cortar un release

Desde la v1.3.0 el Bridge se distribuye como **instalador `.exe` NSIS** publicado en **GitHub Releases**. Auto-actualización integrada via `electron-updater`.

---

## Cortar un release nuevo (3 comandos)

```bash
# 1. Subir versión en cubi-bridge/package.json (semver)
#    "version": "1.3.1"
# 2. Commit + tag + push
git add cubi-bridge/package.json
git commit -m "Bridge v1.3.1 — <qué cambia>"
git tag bridge-v1.3.1
git push origin main --tags
```

Eso dispara `.github/workflows/release-bridge.yml` → corre en `windows-latest` → construye el `.exe` con `electron-builder` → lo sube a `https://github.com/cubi-bridge/cubi-bridge/releases/latest`.

**Demora:** ~5 minutos desde el push hasta que el .exe está disponible.

---

## Qué hace cada release automáticamente

1. **Bridges activos del Pastor (y futuros usuarios)** detectan la versión nueva en su próximo arranque (o dentro de 6h max).
2. Descargan el .exe en background, sin interrumpir.
3. Muestran notificación nativa "v1.3.1 lista — click para reiniciar".
4. Al click → el Bridge se reinicia con la versión nueva. Pairing y configuración se preservan (`%APPDATA%\cubi-bridge-config\`).

---

## Setup inicial (solo se hace una vez)

Para que el workflow funcione, este repo de Replit tiene que estar conectado a un repo en GitHub público bajo `cubi-bridge/cubi-bridge` (o donde se quiera — cambiar `package.json` → `build.publish` + `.github/workflows/release-bridge.yml` si se elige otro nombre).

1. Crear repo público en GitHub: `https://github.com/new` → owner `cubi-bridge`, name `cubi-bridge`, **Public**.
2. En Replit → Tools → Git/GitHub → conectar a ese repo (Replit empuja todo el código + workflows).
3. Primer release: bumpea version a `1.3.0`, commit, tag `bridge-v1.3.0`, push. El primer build puede tardar ~7 min porque cachea node_modules. Releases siguientes: ~5 min.

**Sin token, sin secrets adicionales** — el workflow usa el `GITHUB_TOKEN` automático de Actions.

---

## Endpoints del servidor que dependen del release

| URL | Comportamiento |
|---|---|
| `/downloads/cubi-bridge/installer` | 302 → último `.exe` en GitHub Releases |
| `/downloads/cubi-bridge/info` | JSON con `downloadUrl` + `version` esperada |
| `/downloads/cubi-bridge/` | 302 → página de Releases en GitHub |
| `/api/bridge/installer-info` | JSON consumido por el botón "Descargar" en `/lab` |

Si querés usar **otra org/repo de GitHub**, seteá los env vars `BRIDGE_GITHUB_OWNER` y `BRIDGE_GITHUB_REPO` en Replit Secrets y reiniciá el server.

---

## Firma de código (opcional, futuro)

Hoy el .exe NO está firmado → Windows SmartScreen muestra "Editor desconocido" la 1ª vez. El usuario hace click en "Más información → Ejecutar de todos modos". Después no aparece nunca más.

Cuando se quiera eliminar ese warning desde el día 1, comprar cert OV/EV (Sectigo, SSL.com, ~$200/año), añadir `CSC_LINK` + `CSC_KEY_PASSWORD` a los secrets del repo GitHub, y `electron-builder` firma automáticamente.

---

## Rollback de un release malo

```bash
# Borrar release + tag en GitHub
# Settings → Releases → Delete → "delete tag" también
# El Bridge se queda con la versión que tenía instalada hasta el próximo release.
```

Si querés forzar a los Bridges actuales a un downgrade, hay que cortar un release nuevo con versión MAYOR pero contenido del rollback (semver no permite ir hacia atrás). En la práctica: cortar v1.3.2 con el código de la v1.3.0 buena.
