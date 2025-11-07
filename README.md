# Reverse Proxy - Runpath

Reverse proxy implementation using Nginx for the Runpath software architecture
system.

## 📋 Descripción

El reverse proxy actúa como punto de entrada único para todas las peticiones del
sistema Runpath. Proporciona **comunicación segura HTTPS exclusivamente** a través
del puerto 443 hacia el frontend, garantizando que todo el tráfico externo esté
cifrado.

### Funcionalidades principales:

- **Comunicación segura**: Solo HTTPS en puerto 443 hacia el frontend
- **SSL/TLS**: Utiliza certificados firmados por una CA propia para desarrollo
- **Routing centralizado**: Enruta peticiones cifradas hacia el frontend
- **Seguridad**: Protege servicios internos y asegura la confidencialidad de datos

## 🛠️ Tecnología

- **Nginx**: Servidor web y reverse proxy de alto rendimiento
- **OpenSSL**: Generación de certificados SSL para desarrollo
- **Docker & Docker Compose**: Contenedorización y orquestación

## 📁 Estructura

```
reverse-proxy/
├── docker-compose.yml          # Configuración del contenedor
├── nginx.conf                  # Configuración de Nginx
├── certs/                      # Certificados SSL/TLS
│   ├── ca.crt                  # Certificado de la Autoridad Certificadora (CA)
│   ├── ca.key                  # Clave privada de la CA
│   ├── ca.srl                  # Número de serie para certificados firmados
│   ├── nginx.crt               # Certificado del servidor firmado por la CA
│   ├── nginx.key               # Clave privada del servidor Nginx
│   ├── nginx.csr               # Certificate Signing Request (solicitud)
│   └── extfile.cnf             # Extensiones para el certificado
└── README.md
```

### Documentos en `certs/`

- **`ca.crt`**: Certificado público de la Autoridad Certificadora. Este archivo debe instalarse en el sistema operativo para que los navegadores confíen en los certificados firmados por esta CA.
- **`ca.key`**: Clave privada de la CA. Se utiliza para firmar certificados de servidor. **Debe mantenerse seguro y privado**.
- **`ca.srl`**: Archivo de número de serie que OpenSSL usa para llevar registro de los certificados firmados.
- **`nginx.crt`**: Certificado del servidor Nginx firmado por la CA. Contiene la clave pública del servidor.
- **`nginx.key`**: Clave privada del servidor Nginx. Se usa para cifrar/descifrar el tráfico HTTPS.
- **`nginx.csr`**: Certificate Signing Request, la solicitud que se envía a la CA para obtener el certificado firmado.
- **`extfile.cnf`**: Archivo de configuración con extensiones adicionales para el certificado (ej: Subject Alternative Names).

## 🚀 Cómo ejecutar

### Prerrequisitos

- Docker y Docker Compose instalados
- Redes Docker creadas: `frontend_net` y `orchestration_net`
- OpenSSL (para generar certificados si no existen)

### 1. Instalar el certificado de la CA en tu sistema

Para que tu navegador y aplicaciones confíen en los certificados del servidor, debes instalar el certificado de la CA (`ca.crt`) en tu sistema operativo.

#### **Windows**

1. **Abrir el certificado**:
   - Navega a la carpeta `certs/`
   - Haz doble clic en `ca.crt`

2. **Instalar el certificado**:
   - Click en **"Instalar certificado..."**
   - Selecciona **"Máquina local"** (requiere privilegios de administrador)
   - Click en **"Siguiente"**

3. **Seleccionar el almacén**:
   - Marca **"Colocar todos los certificados en el siguiente almacén"**
   - Click en **"Examinar..."**
   - Selecciona **"Entidades de certificación raíz de confianza"**
   - Click en **"Aceptar"** y luego **"Siguiente"**

4. **Finalizar**:
   - Click en **"Finalizar"**
   - Confirma la advertencia de seguridad con **"Sí"**

**Alternativa por PowerShell (como Administrador)**:

```powershell
# Importar el certificado de la CA
Import-Certificate -FilePath ".\certs\ca.crt" -CertStoreLocation Cert:\LocalMachine\Root
```

#### **Linux (Ubuntu/Debian)**

```bash
# Copiar el certificado al directorio de CAs confiables
sudo cp certs/ca.crt /usr/local/share/ca-certificates/runpath-ca.crt

# Actualizar los certificados del sistema
sudo update-ca-certificates

# Verificar que se agregó correctamente
ls -l /etc/ssl/certs/ | grep runpath
```

#### **macOS**

```bash
# Abrir Keychain Access y agregar el certificado
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain certs/ca.crt

# O manualmente:
# 1. Abre "Keychain Access" (Acceso a llaveros)
# 2. Arrastra ca.crt a "System" o "login"
# 3. Haz doble clic en el certificado
# 4. Expande "Trust" (Confianza)
# 5. Selecciona "Always Trust" (Confiar siempre) para SSL
```

#### **Navegadores (si es necesario)**

- **Firefox**: Tiene su propio almacén de certificados
  1. Configuración → Privacidad y seguridad → Certificados → Ver certificados
  2. Pestaña "Autoridades" → Importar → Selecciona `ca.crt`
  3. Marca "Confiar en esta CA para identificar sitios web"

- **Chrome/Edge**: Usan el almacén del sistema operativo (ya configurado arriba)

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

**HTTPS (puerto 443):**

```bash
curl https://localhost/health
```

Respuesta esperada: Respuesta del frontend sin errores de certificado

**Desde el navegador**:
- Visita `https://localhost`
- **No debería** aparecer ninguna advertencia de seguridad si instalaste correctamente el certificado de la CA

### Comandos útiles

```bash
# Ver logs
docker-compose logs -f

# Detener el servicio
docker-compose down

# Reiniciar después de cambios en nginx.conf
docker-compose restart

# Verificar que el certificado está instalado (Windows PowerShell)
Get-ChildItem -Path Cert:\LocalMachine\Root | Where-Object {$_.Subject -like "*Runpath*"}

# Verificar que el certificado está instalado (Linux)
awk -v cmd='openssl x509 -noout -subject' '/BEGIN/{close(cmd)};{print | cmd}' < /etc/ssl/certs/ca-certificates.crt | grep -i runpath
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
