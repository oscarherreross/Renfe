# 🚆 Renfe – Proximidad entre trenes en tiempo real

Sistema que descarga cada ~15 segundos la posición de los trenes de larga distancia de Renfe, la guarda en PostgreSQL, **detecta pares de trenes que circulan a menos de una distancia umbral** y lo muestra todo en un mapa web con una API REST.

> ⚠️ **Aviso:** son datos operativos orientativos, con posible retraso o imprecisión de GPS. Este proyecto **no es un sistema de seguridad ferroviaria**.

## Arquitectura

```
 Feed JSON de Renfe ──► collector ──► PostgreSQL ◄── backend (FastAPI) ◄── frontend (nginx + Leaflet)
   (cada ~15 s)        (Python)      (histórico)        :8000                    :8080
```

| Servicio | Carpeta | Descripción |
|---|---|---|
| `db` | — | PostgreSQL 16. Crea las tablas automáticamente desde `schema.sql`. |
| `collector` | raíz (`renfe_collector.py`) | Descarga el feed, valida los registros, calcula las alertas y guarda todo en la base de datos. |
| `backend` | `backend/` | API REST con FastAPI (documentación Swagger incluida). |
| `frontend` | `frontend/` | Mapa Leaflet con los trenes y las alertas, servido por nginx (que hace de proxy de `/api/` hacia el backend). |

## Cómo funciona la detección

1. El colector descarga el JSON de flota de Renfe y descarta registros sin coordenadas válidas.
2. Dos trenes se consideran **comparables** si tienen el mismo origen y destino, o la misma estación siguiente.
3. Para cada par comparable se calcula la distancia con la fórmula de **Haversine**.
4. Si la distancia es **estrictamente menor** que el umbral (por defecto 1000 m), se registra una alerta en `proximity_alerts`.

Las posiciones se deduplican por `(vehicle_id, position_timestamp)`: si la fuente repite una lectura, se ignora.

## Requisitos

- [Docker](https://docs.docker.com/get-docker/) y Docker Compose

## Puesta en marcha

1. Clona el repositorio:

   ```bash
   git clone https://github.com/oscarherreross/Renfe.git
   cd Renfe
   ```

2. Crea un archivo `.env` en la raíz (la contraseña es obligatoria):

   ```env
   DB_PASSWORD=pon_aqui_una_contraseña
   ```

3. Arranca todo:

   ```bash
   docker compose up --build
   ```

4. Abre:

   | Qué | URL |
   |---|---|
   | Mapa web | http://localhost:8080 |
   | API + Swagger | http://localhost:8000/docs |
   | PostgreSQL (desde tu equipo) | `localhost:5435` |

> Los primeros datos aparecen tras el primer ciclo del colector (unos segundos). Los datos se conservan en el volumen `pgdata`.

## Configuración

Todas las variables son opcionales salvo `DB_PASSWORD`.

| Variable | Por defecto | Descripción |
|---|---|---|
| `DB_PASSWORD` | — (obligatoria) | Contraseña de PostgreSQL. |
| `DB_NAME` | `renfe` | Nombre de la base de datos. |
| `DB_USER` | `renfe` | Usuario de la base de datos. |
| `DB_PORT` | `5435` | Puerto expuesto en tu equipo para PostgreSQL. |
| `ALERT_DISTANCE_METERS` | `1000` | Umbral de proximidad, en metros. |
| `POLL_SECONDS` | `15` | Intervalo de descarga (mínimo 15 s, que es lo que tarda en actualizarse la fuente). |
| `RECENT_SECONDS` | `300` | Antigüedad máxima para considerar "reciente" una posición en la API. |
| `RENFE_URL` | feed de flota de larga distancia de Renfe | URL de la fuente de datos. |

## API

| Método | Endpoint | Descripción |
|---|---|---|
| GET | `/getTrenes` | Posiciones recientes de todos los trenes. |
| GET | `/getTren/{id}` | Última posición de un tren (por matrícula). |
| GET | `/getAlertas?limit=50` | Alertas de proximidad, de más recientes a más antiguas (`limit` de 1 a 1000). |
| GET | `/getDistancia/{trenId1}/{trenId2}` | Distancia actual en metros entre dos trenes. |

Ejemplo:

```bash
curl http://localhost:8000/getAlertas?limit=5
```

## Modelo de datos

Definido en [`schema.sql`](schema.sql):

- **`train_positions`**: histórico de posiciones (`vehicle_id`, `trip_id`, `route_id`, coordenadas, marca de tiempo de la fuente, momento de recepción y el JSON original).
- **`proximity_alerts`**: pares de trenes detectados por debajo del umbral, con sus posiciones, la distancia y el umbral usado.

## Estructura del proyecto

```
.
├── renfe_collector.py   # Colector: descarga, detección de proximidad y persistencia
├── Dockerfile           # Imagen del colector
├── requirements.txt     # Dependencias del colector
├── schema.sql           # Esquema de la base de datos
├── docker-compose.yml   # Orquestación de los 4 servicios
├── backend/             # API FastAPI
└── frontend/            # Mapa web (Leaflet + nginx)
```

## Tecnologías

Python 3.12 · FastAPI · psycopg 3 · PostgreSQL 16 · Leaflet · nginx · Docker Compose

## Créditos y datos

Las posiciones proceden del visor público en tiempo real de Renfe y el mapa base es de © [OpenStreetMap](https://www.openstreetmap.org/copyright). Este proyecto no está afiliado a Renfe.
