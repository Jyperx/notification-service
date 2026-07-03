# Punto — Notification Service

Servicio independiente que **escucha Firestore** y envía notificaciones
(push vía Expo + notificación in-app en `users/{uid}/notifications`).

Antes vivía dentro de `search-backend`; se extrajo a su propio repo porque no
depende del backend de búsqueda (init propio de Firebase, no usa el índice SQLite).

## Qué hace

Escucha 3 colecciones de Firestore (`on_snapshot`) y reacciona:

- **`orders`** → notifica cambios de estado del pedido (recibido, preparando,
  listo, en camino, entregado, cancelado…) al cliente, comercio o repartidores.
- **`users`** → cuando el admin aprueba/rechaza, avisa al **comercio**
  (`commerceStatus`) o al **repartidor** (`driverStatus`).
- **`marketing_campaigns`** → modo de **anuncios masivos**: al pasar a `sending`,
  manda un push a todos los usuarios y marca la campaña como `sent`.

## Cómo se conecta

Firestore actúa como bus de eventos: la app / admin / backend escriben, y este
servicio reacciona. No hay que llamarlo directo.

```
App / Admin / Backend ──▶ Firestore (orders, users, marketing_campaigns)
                              │ (on_snapshot)
                              ▼
                       notification-service ──▶ Expo Push
                                            └──▶ Firestore users/{uid}/notifications
```

## De-duplicación (Redis o memoria)

Recuerda el último estado visto por documento para no re-notificar. Usa **Redis**
si `REDIS_URL` está definido (persiste entre reinicios y expira con TTL, así no
crece sin límite); si no, cae a **memoria local** sin romper nada.

Beneficio del modo Redis: si el servicio se cae y un comercio/repartidor es
aprobado en ese lapso, al reiniciar **detecta la transición y sí notifica**.

## Variables de entorno

| Variable | Requerida | Descripción |
|---|---|---|
| `FIREBASE_SERVICE_ACCOUNT` | Sí (prod) | JSON del service account en una línea. En local puede usarse `serviceAccountKey.json`. |
| `REDIS_URL` | No | Redis para de-dup persistente. Sin él, usa memoria. |

## Correr en local

```bash
pip install -r requirements.txt
# coloca serviceAccountKey.json  o  exporta FIREBASE_SERVICE_ACCOUNT
python notifications_server.py
```

## Deploy (Railway)

Es un **worker** (no expone HTTP). El start command está en `nixpacks.toml`
(`python notifications_server.py`) y también en el `Procfile`. Configura las
variables de entorno del cuadro anterior.
