# Reverse Proxy - Runpath

Reverse proxy implementation using Nginx for the Runpath software architecture
system.

## 📋 Descripción

El reverse proxy actúa como punto de entrada único para todas las peticiones del
sistema Runpath.Enruta el tráfico hacia el api gateway y proporciona una capa
adicional de seguridad.

### Funcionalidades principales:

- **Routing centralizado**: Enruta peticiones hacia el API Gateway
- **Health checks**: Endpoint de salud para monitoreo
- **Seguridad**: Protege servicios internos al no exponerlos directamente

## 🛠️ Tecnología

- **Nginx**: Servidor web y reverse proxy de alto rendimiento
- **OpenSSL**: Generación de certificados SSL para desarrollo
- **Docker & Docker Compose**: Contenedorización y orquestación

## 📁 Estructura

```
reverse-proxy/
├── docker-compose.yml          # Configuración del contenedor
├── nginx.conf                  # Configuración de Nginx
├── certs/                      # Certificados SSL
│   ├── nginx-selfsigned.crt    # Certificado público
│   └── nginx-selfsigned.key    # Clave privada
└── README.md
```

## 🚀 Cómo ejecutar

### Prerrequisitos

- Docker y Docker Compose instalados
- Redes Docker creadas: `frontend_net` y `orchestration_net`
- OpenSSL (para generar certificados)

### 1. Generar certificados SSL (primera vez)

**En PowerShell:**

```powershell
# Crear carpeta para certificados
New-Item -ItemType Directory -Force -Path certs

# Generar certificados auto-firmados
openssl req -x509 -nodes -days 365 -newkey rsa:2048 `
  -keyout certs/nginx-selfsigned.key `
  -out certs/nginx-selfsigned.crt `
  -subj "/CN=localhost"
```

**En Git Bash/Linux:**

```bash
# Crear carpeta para certificados
mkdir -p certs

# Generar certificados auto-firmados
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/nginx-selfsigned.key \
  -out certs/nginx-selfsigned.crt \
  -subj "//CN=localhost"
```

### 2. Crear redes Docker (si no existen)

```bash
docker network create frontend_net
docker network create orchestration_net
```

### 3. Levantar el servicio

```bash
docker-compose up --build -d
```

### 4. Verificar funcionamiento

**HTTP (puerto 80):**

```bash
curl http://localhost/health
```

**HTTPS (puerto 443):**

```bash
curl -k https://localhost/health
```

Respuesta esperada: `Reverse Proxy is up and running`

### Comandos útiles

```bash
# Ver logs
docker-compose logs -f

# Detener el servicio
docker-compose down

# Reiniciar después de cambios en nginx.conf
docker-compose restart
```

## 🔒 Configuración SSL

### Desarrollo (certificados auto-firmados)

Los certificados generados con OpenSSL son válidos solo para desarrollo local.
El navegador mostrará una advertencia de seguridad que puedes aceptar
manualmente.

### Producción (Let's Encrypt)

Para producción, reemplaza los certificados auto-firmados con certificados
válidos de Let's Encrypt usando Certbot:

```bash
# Ejemplo con Certbot
certbot certonly --standalone -d tudominio.com
```

Actualiza las rutas en `nginx.conf`:

```nginx
ssl_certificate /etc/letsencrypt/live/tudominio.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/tudominio.com/privkey.pem;
```

## 🌐 Arquitectura

```
Cliente (Navegador)
    ↓ HTTPS (puerto 443)
Reverse Proxy (Nginx)
    ↓ HTTP (red interna Docker)
API Gateway (puerto 8888)
    ↓
Microservicios backend
```

El reverse proxy realiza **SSL Termination**: descifra el tráfico HTTPS del
cliente y lo reenvía como HTTP a los servicios internos en la red Docker
privada.

## 📝 Notas

- Los puertos 80 y 443 son los estándar para HTTP/HTTPS
- Puedes usar otros puertos modificando `docker-compose.yml`: `"8080:80"` y
  `"8443:443"`
- Los certificados auto-firmados expiran en 365 días
- El servicio está configurado con `restart: unless-stopped` implícito
