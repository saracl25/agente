# agente
# 🤖 Price Monitor Agent

> Agente de software autónomo para monitoreo de precios en tiempo real con alertas automáticas por email.

[![CI/CD](https://img.shields.io/github/actions/workflow/status/your-org/price-monitor-agent/ci-cd.yml?label=CI%2FCD&logo=github)](https://github.com/your-org/price-monitor-agent/actions)
[![Coverage](https://img.shields.io/codecov/c/github/your-org/price-monitor-agent?logo=codecov)](https://codecov.io/gh/your-org/price-monitor-agent)
[![Snyk](https://img.shields.io/snyk/vulnerabilities/github/your-org/price-monitor-agent?logo=snyk)](https://snyk.io)
[![Python](https://img.shields.io/badge/python-3.11-blue?logo=python)](https://python.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 📋 Tabla de Contenidos

- [Descripción](#descripción)
- [Arquitectura](#arquitectura)
- [Requisitos de Calidad](#requisitos-de-calidad)
- [Estructura del Proyecto](#estructura-del-proyecto)
- [Instalación](#instalación)
- [Configuración](#configuración)
- [Uso](#uso)
- [Tests](#tests)
- [CI/CD Pipeline](#cicd-pipeline)
- [Despliegue en Staging](#despliegue-en-staging)
- [Seguridad](#seguridad)

---

## Descripción

El **Price Monitor Agent** es un agente autónomo que:

| Entrada | Lógica | Salida |
|---------|--------|--------|
| Lista de `product_id` + precio objetivo | Consulta API externa → sanitiza datos → compara con umbrales | `PriceRecord[]` + email de alerta si precio ≤ objetivo |

**API externa integrada:** `GET /v1/prices/{product_id}` → `{ price, currency, name }`

---

## Arquitectura

### C4 — Contexto

```
[Operador/Scheduler]
       │ cron / trigger
       ▼
[Price Monitor Agent] ──HTTP──▶ [External Price API]
       │
       ▼ email
[Alert Recipients]
```

### C4 — Contenedores

```
┌─────────────────────────────────────────────┐
│              Price Monitor Agent            │
│                                             │
│  ┌────────────┐   ┌──────────────────────┐  │
│  │ PriceMonitor│   │   PriceAPIClient     │  │
│  │   Agent    │──▶│  (HTTP + Retry 3x)   │──┼──▶ External API
│  └─────┬──────┘   └──────────────────────┘  │
│        │                                    │
│  ┌─────▼──────┐   ┌──────────────────────┐  │
│  │  Sanitizer │   │     Notifier          │  │
│  │  (input    │   │  (SMTP / email)       │──┼──▶ Recipients
│  │  validation│   └──────────────────────┘  │
│  └────────────┘                             │
└─────────────────────────────────────────────┘
```

---

## Requisitos de Calidad

### 1. 🚀 Rendimiento
- Tiempo de respuesta p95 ≤ 500ms por producto monitoreado.
- Reintentos automáticos con backoff exponencial (3 intentos, factores 0.5s).
- Timeout configurable vía `PRICE_API_TIMEOUT_SEC` (default: 10s).

### 2. 🔒 Seguridad
- **Zero secrets en código**: todas las credenciales por variables de entorno / GitHub Secrets.
- **Input sanitization**: IDs de producto validados contra regex estricta (`[a-zA-Z0-9_\-]{1,64}`).
- **Non-root container**: imagen Docker ejecuta bajo usuario UID 1000.
- **SAST + SCA**: Snyk integrado en pipeline (falla en vulnerabilidades HIGH).
- **TLS forzado**: SMTP usa `starttls()`, API usa `https://`.

### 3. 🔧 Mantenibilidad
- Cobertura de tests ≥ 80% (enforced en CI).
- Arquitectura modular: `agent/`, `api/`, `utils/` desacoplados.
- Tipado estático con `mypy` y linting con `ruff`.
- Logging estructurado con niveles configurables.

---

## Estructura del Proyecto

```
price-monitor-agent/
├── src/
│   ├── agent/
│   │   └── price_monitor.py     # Core: PriceMonitorAgent
│   ├── api/
│   │   └── price_api.py         # HTTP client con retry
│   ├── utils/
│   │   ├── sanitizer.py         # Validación de entradas
│   │   └── notifier.py          # Alertas por email
│   └── main.py                  # Entry point
├── tests/
│   ├── test_sanitizer.py        # 14 casos de prueba
│   └── test_price_monitor.py    # 12 casos de prueba
├── docs/
│   └── architecture.md
├── configs/
│   └── ci-cd.yml                # GitHub Actions pipeline
├── Dockerfile                   # Multi-stage, non-root
├── .env.example                 # Template de variables
├── requirements.txt
├── pyproject.toml               # pytest + ruff + black config
└── README.md
```

---

## Instalación

```bash
# 1. Clonar repositorio
git clone https://github.com/your-org/price-monitor-agent.git
cd price-monitor-agent

# 2. Crear entorno virtual
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# 3. Instalar dependencias
pip install -r requirements.txt

# 4. Configurar variables de entorno
cp .env.example .env
# Editar .env con los valores reales
```

---

## Configuración

| Variable | Descripción | Requerida |
|----------|-------------|-----------|
| `PRICE_API_KEY` | API Key del servicio de precios | ✅ |
| `PRICE_API_BASE_URL` | URL base de la API | ✅ |
| `PRICE_API_TIMEOUT_SEC` | Timeout HTTP en segundos | ❌ (default: 10) |
| `SMTP_HOST` | Servidor SMTP para alertas | ✅ |
| `SMTP_PORT` | Puerto SMTP | ❌ (default: 587) |
| `SMTP_USER` | Usuario SMTP | ✅ |
| `SMTP_PASS` | Contraseña SMTP | ✅ |
| `ALERT_FROM_EMAIL` | Email remitente | ❌ |
| `WATCH_PRODUCTS` | IDs separados por coma | ❌ |
| `ALERT_EMAIL` | Email destinatario de alertas | ❌ |

---

## Uso

```python
from src.agent.price_monitor import PriceMonitorAgent, AlertConfig
from src.api.price_api import PriceAPIClient

agent = PriceMonitorAgent(api_client=PriceAPIClient())

# Registrar alerta: notificar si widget-001 baja de $45
agent.register_alert(AlertConfig(
    product_id="widget-001",
    target_price=45.0,
    alert_email="buyer@example.com",
))

# Ejecutar ciclo de monitoreo
records = agent.run(["widget-001", "gadget-42"])
```

---

## Tests

```bash
# Ejecutar todos los tests
pytest tests/ -v

# Con reporte de cobertura
pytest tests/ --cov=src --cov-report=term-missing

# Test específico
pytest tests/test_sanitizer.py -v
```

**Cobertura mínima requerida:** 80%

---

## CI/CD Pipeline

El archivo `configs/ci-cd.yml` define 5 jobs secuenciales:

```
Push/PR
  │
  ├─▶ [1] lint      → Ruff + Black + Mypy
  │         │
  ├─────────▶ [2] tests     → Pytest + Coverage ≥80%
  │         │
  └─────────▶ [3] security  → Snyk SCA + SAST
                    │
              (solo develop)
                    │
                    ▶ [4] build      → Docker build + push
                          │
                          ▶ [5] deploy   → SSH deploy to staging
```

**Secrets requeridos en GitHub:**
- `PRICE_API_KEY`, `PRICE_API_KEY_TEST`
- `SNYK_TOKEN`
- `DOCKER_USERNAME`, `DOCKER_TOKEN`
- `STAGING_HOST`, `STAGING_USER`, `STAGING_SSH_KEY`
- `SMTP_HOST`, `SMTP_USER`, `SMTP_PASS`

---

## Despliegue en Staging

### Opción A — Docker Compose (recomendado)

```bash
# En el servidor de staging
docker pull your-org/price-monitor-agent:staging

docker run -d \
  --name price-monitor-agent \
  --restart unless-stopped \
  -e PRICE_API_KEY="<key>" \
  -e SMTP_HOST="smtp.mailgun.org" \
  -e SMTP_USER="<user>" \
  -e SMTP_PASS="<pass>" \
  your-org/price-monitor-agent:staging
```

### Opción B — Build local

```bash
docker build -t price-monitor-agent:local .
docker run --env-file .env price-monitor-agent:local
```

### Health Check

```bash
docker inspect --format='{{.State.Health.Status}}' price-monitor-agent
# Expected: healthy
```

---

## Seguridad

- **Nunca** commitear `.env` al repositorio (está en `.gitignore`).
- Rotar `PRICE_API_KEY` cada 90 días.
- Revisar resultados de Snyk en cada PR antes de mergear.
- El contenedor corre como usuario no-root (UID 1000).

---

## Licencia

MIT © 2024 Your Organization
