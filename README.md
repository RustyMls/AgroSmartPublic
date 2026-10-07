# AgroSmart

Sistema para invernadero de tomate y pimiento (Cotacachi, 1,500 m²): automatización IoT + ERP simplificado, con frontend compartido pero backends y bases de datos independientes.

## Estructura

- `backend-erp/` — Inventario, gastos y ventas (Node/Express + PostgreSQL vía Prisma)
- `backend-automation/` — Telemetría MQTT, riego, alertas (Node/Express + PostgreSQL vía Prisma + Mosquitto)
- `firmware-esp32/` — Firmware del ESP32 central (PlatformIO/Arduino): sensores, control de bomba, seguridad edge
- `frontend/` — Panel único React (Vite + TS): dashboard, riego, inventario, gastos, ventas, usuarios

## Backend ERP — desarrollo local

```bash
docker compose up -d postgres-erp   # levanta PostgreSQL en localhost:5434

cd backend-erp
npm install
cp .env.example .env   # apunta al Postgres del docker-compose
npx prisma migrate dev
npm run dev
```

## Backend Automation — desarrollo local

```bash
docker compose up -d postgres-automation mosquitto   # Postgres en localhost:5435, MQTT en localhost:1883

cd backend-automation
npm install
cp .env.example .env
npx prisma migrate dev
npm run dev
```

Endpoints principales: `GET /api/telemetry/latest`, `GET /api/alerts`, `POST /api/pump/authorize`, `GET/POST/PATCH/DELETE /api/schedules`.

Para simular al ESP32 mientras no exista el firmware:

```bash
docker exec agrosmart-mosquitto mosquitto_pub -h localhost -t invernadero/sensor/temperatura -m "24.8"
```

## Firmware ESP32

Ver [firmware-esp32/README.md](firmware-esp32/README.md) para instrucciones de configuración, calibración y flasheo.

## Autenticación compartida

Los usuarios y el login viven únicamente en `backend-erp` (`POST /api/auth/login`). `backend-automation` no tiene tabla de usuarios: solo verifica el JWT con el mismo `JWT_SECRET` (debe ser idéntico en ambos `.env`). Un solo login sirve para los dos backends.

Roles: `ADMIN` (todo), `ADMINISTRATIVO` (inventario/gastos/ventas), `OPERADOR` (autorizar riego, horarios). Las lecturas (`GET`) solo requieren estar autenticado; las escrituras requieren el rol correspondiente.

```bash
cd backend-erp
npm run seed   # crea el admin inicial desde SEED_ADMIN_EMAIL/SEED_ADMIN_PASSWORD del .env
```

Gestión de usuarios (solo ADMIN): `GET/POST/PATCH/DELETE /api/users` en `backend-erp`.

## Frontend — desarrollo local

Requiere que `backend-erp` y `backend-automation` estén corriendo (ver arriba).

```bash
cd frontend
npm install
cp .env.example .env   # URLs de ambas APIs
npm run dev            # http://localhost:5173
```

Login único contra `backend-erp`; el mismo token se usa para llamar a `backend-automation`. La navegación se adapta al rol: `ADMIN` ve todo, `OPERADOR` solo Dashboard/Riego, `ADMINISTRATIVO` ve Dashboard/Inventario/Gastos/Ventas (no Riego ni Usuarios).
