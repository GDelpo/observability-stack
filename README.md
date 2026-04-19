# Observability Stack

Stack de observabilidad para los microservicios de tu organización: **Loki** (almacenamiento de logs) + **Promtail** (agente de recolección) + **Grafana** (dashboards y alertas).

## Arquitectura

```
Docker containers
      │
   Promtail          ← lee logs de /var/lib/docker/containers
      │
    Loki             ← almacena y indexa los logs
      │
   Grafana           ← dashboards, alertas por email (SendGrid)
      │
   Traefik           → /grafana/
```

---

## Configuración

```bash
cp .env.example .env
# Completar: GRAFANA_ADMIN_PASSWORD, GRAFANA_SECRET_KEY, SENDGRID_API_KEY
```

| Variable | Descripción |
|----------|-------------|
| `SERVER_IP` | IP del servidor (`your-server.example.com`) |
| `GRAFANA_ADMIN_USER` | Usuario admin de Grafana (default: `admin`) |
| `GRAFANA_ADMIN_PASSWORD` | Contraseña admin. Generar: `openssl rand -hex 16` |
| `GRAFANA_SECRET_KEY` | Clave secreta Grafana. Generar: `openssl rand -hex 32` |
| `SENDGRID_API_KEY` | API key para alertas por email. Copiar de `mailsender/.env` |

---

## Deploy

```bash
cd /opt/services/observability

# Crear directorios de datos (una sola vez)
mkdir -p /opt/services/data/observability/{loki,promtail,grafana}

cp .env.example .env
# Completar .env

docker compose --env-file .env -f docker/docker-compose.prod.traefik.yml up -d
```

Acceso: `http://your-server.example.com/grafana/`

> Traefik debe estar corriendo antes de levantar este stack.

---

## Estructura

```
observability/
├── docker/
│   └── docker-compose.prod.traefik.yml   # Loki + Promtail + Grafana
├── config/
│   ├── loki.yml                          # Config de Loki
│   ├── promtail.yml                      # Config de Promtail (fuentes de logs)
│   └── grafana/
│       └── provisioning/
│           ├── datasources/loki.yml      # Datasource Loki en Grafana
│           ├── dashboards/               # Dashboards JSON provisioned
│           └── alerting/                 # Reglas y canales de alerta
└── .env.example
```

---

## Volúmenes de datos (servidor)

| Path en host | Descripción |
|---|---|
| `/opt/services/data/observability/loki` | Chunks y índices de Loki |
| `/opt/services/data/observability/promtail` | Estado de Promtail |
| `/opt/services/data/observability/grafana` | Dashboards, usuarios, alertas |

---

## Agregar un servicio nuevo

Promtail descubre containers automáticamente vía Docker socket, pero solo procesa los que están en la lista de `keep` en `config/promtail.yml`. Para sumar un servicio nuevo:

### 1. Agregar el container a la lista de scraping

En `config/promtail.yml`, dentro de `relabel_configs`, agregar el nombre del container a la regex:

```yaml
- source_labels: [service]
  regex: 'identity_api|mailsender_api|...|mi_nuevo_servicio'
  action: keep
```

El nombre debe coincidir exactamente con el `container_name` del compose del servicio.

### 2. Configurar el parsing de logs (si aplica)

**Servicio FastAPI** (emite JSON con campos `level` y `message`): agregar el nombre a ambas reglas `match` de `pipeline_stages`:

```yaml
# Bloque JSON parsing
- match:
    selector: '{service=~"...|mi_nuevo_servicio"}'
    stages:
      - json:
          expressions:
            level: level
            message: message
      - labels:
          level:

# Bloque Gunicorn fallback
- match:
    selector: '{service=~"...|mi_nuevo_servicio", level=""}'
    stages:
      - regex:
          expression: '\[(?P<level>CRITICAL|ERROR|WARNING|INFO|DEBUG)\]'
      - labels:
          level:
```

**Otro tipo de servicio** (texto plano, Django, etc.): solo agregarlo a la lista de `keep`. Los logs llegan a Loki sin label `level` — igualmente son consultables en Grafana.

### 3. Reiniciar Promtail

```bash
cd /opt/services/observability
docker compose --env-file .env -f docker/docker-compose.prod.traefik.yml restart promtail
```

### 4. Verificar en Grafana

Explore → Loki → `{service="mi_nuevo_servicio"}` — los logs deben aparecer en segundos.

---

## Troubleshooting

### Grafana no carga en `/grafana/`

Verificar que `GF_SERVER_ROOT_URL` y `GF_SERVER_SERVE_FROM_SUB_PATH=true` están seteados en el compose.

### Loki no recibe logs

```bash
docker logs promtail | grep -i error
# Verificar que /var/lib/docker/containers está montado y accesible
docker exec -it promtail ls /var/lib/docker/containers | head -5
```

### Alertas por email no llegan

Verificar `SENDGRID_API_KEY` en `.env` y que `GF_SMTP_ENABLED=true` está activo en el compose.
