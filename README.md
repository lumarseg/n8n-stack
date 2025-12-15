# n8n Stack

Despliegue de n8n con Task Runners en modo externo usando Docker Compose.

## Componentes

- **n8n Main**: Interfaz web, webhooks y broker de tareas
- **Task Runners especializados**:
  - 2 runners de JavaScript
  - 2 runners de Python
- **PostgreSQL**: Base de datos (docker-compose separado)
- **Traefik**: Proxy reverso con SSL/TLS

## Inicio Rápido

### 1. Crear redes (solo la primera vez)
```bash
docker network create backend_net
docker network create proxy_net
```

### 2. Iniciar servicios
```bash
# Iniciar PostgreSQL primero
docker compose -f postgres.yaml up -d

# Iniciar stack n8n
docker compose up -d
```

### 3. Verificar estado
```bash
docker compose ps
docker compose logs -f
```

## Comandos Útiles

```bash
# Ver logs
docker compose logs -f main
docker compose logs -f runner-js-1
docker compose logs -f runner-py-1

# Detener servicios
docker compose down

# Reiniciar servicio específico
docker compose restart main
docker compose restart runner-js-1
```

## Configuración

Edita el archivo `.env` o las variables de entorno en `docker-compose.yml`:

- `N8N_RUNNERS_AUTH_TOKEN`: Token de autenticación (generar con `openssl rand -base64 32`)
- Dominio: Configurado en labels de Traefik
- Versiones: n8n y runners deben coincidir (actualmente 2.0.1)

## Escalado

Para añadir más task runners, edita `docker-compose.yml` y duplica la configuración del runner correspondiente:

```bash
# Añadir un tercer runner de JavaScript
docker compose up -d runner-js-3

# Añadir un tercer runner de Python
docker compose up -d runner-py-3
```

## Notas

- **Runners especializados**: Los runners de JavaScript y Python están separados para mejor rendimiento
- Los task runners se conectan al broker en el puerto 5679
- Cada runner puede ejecutar hasta 5 tareas concurrentes
- Auto-shutdown después de 15 segundos de inactividad
- El volumen `n8n_storage` es compartido entre Main y todos los runners
- Healthcheck disponible en `/healthz`
