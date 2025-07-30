# 🏗️ FASE 1 DETALLADA: CONFIGURACIÓN DE INFRAESTRUCTURA BASE

## 🎯 OBJETIVOS DE LA FASE 1
- Configurar PostgreSQL para multi-tenancy
- Crear schemas por empresa y compartidos
- Configurar seguridad y accesos
- Preparar subdominio para desarrollo
- Establecer backup básico
- Configurar PHP y extensiones

**Duración estimada:** 2-3 días  
**Estado actual:** PostgreSQL 13.20 ✅ Instalado y funcionando

---

## 📋 PASO 1.1: CONFIGURACIÓN BÁSICA POSTGRESQL
**Tiempo estimado:** 30-45 minutos

### **1.1.1 Crear usuario administrador del ERP**
```bash
# Conectar como usuario postgres
sudo -u postgres psql

# Dentro de PostgreSQL
CREATE USER fixum_admin WITH ENCRYPTED PASSWORD 'FixumERP2025!@#';
ALTER USER fixum_admin CREATEDB;
ALTER USER fixum_admin CREATEROLE;
GRANT ALL PRIVILEGES ON DATABASE postgres TO fixum_admin;

# Verificar que se creó correctamente
\du

# Salir de PostgreSQL
\q
```

### **1.1.2 Crear base de datos principal**
```bash
# Crear la base de datos principal del ERP
sudo -u postgres createdb fixum_erp -O fixum_admin

# Verificar que se creó
sudo -u postgres psql -l | grep fixum_erp
```

### **1.1.3 Probar conexión con el nuevo usuario**
```bash
# Conectar con el usuario creado
psql -h localhost -U fixum_admin -d fixum_erp

# Si conecta correctamente, salir
\q
```

**✅ Checkpoint:** Usuario y base de datos creados correctamente

---

## 🔐 PASO 1.2: CONFIGURACIÓN DE SEGURIDAD
**Tiempo estimado:** 45-60 minutos

### **1.2.1 Configurar postgresql.conf**
```bash
# Editar configuración principal
sudo nano /var/lib/pgsql/data/postgresql.conf

# Modificar estas líneas:
listen_addresses = 'localhost'          # Escuchar solo localhost por seguridad
port = 5432                            # Puerto por defecto
max_connections = 200                  # Aumentar conexiones para multi-tenant
shared_buffers = 256MB                 # Optimizar memoria para tu VPS
effective_cache_size = 1GB             # Aprovechar RAM disponible
work_mem = 4MB                         # Memoria por operación
maintenance_work_mem = 64MB            # Memoria para mantenimiento
checkpoint_completion_target = 0.9     # Optimización de escritura
wal_buffers = 16MB                     # Buffer de logs
default_statistics_target = 100        # Estadísticas para queries
logging_collector = on                 # Habilitar logs
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_statement = 'mod'                  # Log modificaciones
log_min_duration_statement = 1000      # Log queries lentas (>1s)
```

### **1.2.2 Configurar pg_hba.conf (autenticación)**
```bash
# Editar configuración de autenticación
sudo nano /var/lib/pgsql/data/pg_hba.conf

# Agregar al final del archivo (antes de las líneas existentes):
# Database administrative login by Unix domain socket
local   all             postgres                                peer

# "local" is for Unix domain socket connections only
local   all             fixum_admin                             md5
local   fixum_erp       fixum_admin                             md5

# IPv4 local connections:
host    fixum_erp       fixum_admin     127.0.0.1/32            md5
host    fixum_erp       fixum_admin     ::1/128                 md5

# Denegar todo lo demás
local   all             all                                     reject
host    all             all             127.0.0.1/32            reject
host    all             all             ::1/128                 reject
```

### **1.2.3 Reiniciar PostgreSQL y verificar**
```bash
# Reiniciar servicio
sudo systemctl restart postgresql

# Verificar que esté funcionando
sudo systemctl status postgresql

# Probar conexión con nueva configuración
psql -h localhost -U fixum_admin -d fixum_erp
```

**✅ Checkpoint:** PostgreSQL configurado con seguridad básica

---

## 🏢 PASO 1.3: CREACIÓN DE SCHEMAS MULTI-TENANT
**Tiempo estimado:** 30-45 minutos

### **1.3.1 Conectar a la base de datos**
```bash
psql -h localhost -U fixum_admin -d fixum_erp
```

### **1.3.2 Crear schemas por empresa**
```sql
-- Schema para cada empresa del Grupo Fixum
CREATE SCHEMA IF NOT EXISTS kreis;
CREATE SCHEMA IF NOT EXISTS gosan;
CREATE SCHEMA IF NOT EXISTS altaguarda;
CREATE SCHEMA IF NOT EXISTS dtc_coatzacoalcos;
CREATE SCHEMA IF NOT EXISTS maha;
CREATE SCHEMA IF NOT EXISTS milenio;
CREATE SCHEMA IF NOT EXISTS acierta;
CREATE SCHEMA IF NOT EXISTS jaguar;

-- Schema para recursos compartidos
CREATE SCHEMA IF NOT EXISTS shared;

-- Schema para configuraciones del sistema
CREATE SCHEMA IF NOT EXISTS system;

-- Verificar que se crearon todos los schemas
\dn
```

### **1.3.3 Configurar permisos por schema**
```sql
-- Dar permisos completos al usuario fixum_admin en todos los schemas
GRANT ALL ON SCHEMA kreis TO fixum_admin;
GRANT ALL ON SCHEMA gosan TO fixum_admin;
GRANT ALL ON SCHEMA altaguarda TO fixum_admin;
GRANT ALL ON SCHEMA dtc_coatzacoalcos TO fixum_admin;
GRANT ALL ON SCHEMA maha TO fixum_admin;
GRANT ALL ON SCHEMA milenio TO fixum_admin;
GRANT ALL ON SCHEMA acierta TO fixum_admin;
GRANT ALL ON SCHEMA jaguar TO fixum_admin;
GRANT ALL ON SCHEMA shared TO fixum_admin;
GRANT ALL ON SCHEMA system TO fixum_admin;

-- Configurar search_path por defecto
ALTER USER fixum_admin SET search_path TO system, public;
```

### **1.3.4 Crear tablas de prueba**
```sql
-- Tabla de prueba en schema system
SET search_path TO system, public;

CREATE TABLE IF NOT EXISTS test_connection (
    id SERIAL PRIMARY KEY,
    message VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO test_connection (message) VALUES ('PostgreSQL Multi-Tenant configurado correctamente');

-- Verificar que funciona
SELECT * FROM test_connection;

-- Tabla de prueba en schema de empresa
SET search_path TO kreis, public;

CREATE TABLE IF NOT EXISTS test_tenant (
    id SERIAL PRIMARY KEY,
    company VARCHAR(50),
    message VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO test_tenant (company, message) VALUES ('Kreis', 'Schema empresarial funcionando');

-- Verificar
SELECT * FROM test_tenant;
```

**✅ Checkpoint:** Schemas multi-tenant creados y funcionando

---

## 🌐 PASO 1.4: CONFIGURACIÓN DEL SUBDOMINIO
**Tiempo estimado:** 45-60 minutos

### **1.4.1 Acceder a CyberPanel**
```bash
# Obtener password de CyberPanel si no la tienes
sudo cat .litespeed_password
```
Accede a: `https://31.97.141.107:8090`

### **1.4.2 Crear subdominio en CyberPanel**
1. **Login** con las credenciales de root
2. **Websites → Create Website**
3. **Configuración:**
   - **Domain Name:** `erp.tudominio.com` (reemplaza con tu dominio real)
   - **Admin Email:** tu email
   - **Package:** Default
   - **Open SSL:** YES (importante para HTTPS)
4. **Create Website**

### **1.4.3 Configurar DNS (si no está configurado)**
Si el dominio es tuyo, agrega en tu proveedor de DNS:
```
Tipo: A
Nombre: erp
Valor: 31.97.141.107
TTL: 300
```

### **1.4.4 Configurar SSL**
1. En CyberPanel: **SSL → Hostname SSL**
2. Seleccionar el dominio `erp.tudominio.com`
3. **Issue SSL**
4. Esperar confirmación

### **1.4.5 Verificar configuración**
```bash
# Verificar que el directorio se creó
ls -la /home/erp.tudominio.com/

# Crear archivo de prueba
echo "<?php phpinfo(); ?>" > /home/erp.tudominio.com/public_html/info.php

# Probar en navegador
# https://erp.tudominio.com/info.php
```

**✅ Checkpoint:** Subdominio configurado con SSL

---

## 🐘 PASO 1.5: CONFIGURAR PHP Y EXTENSIONES
**Tiempo estimado:** 30-45 minutos

### **1.5.1 Verificar versión de PHP**
```bash
# Ver versiones disponibles
sudo dnf list available | grep php

# Verificar versión actual
php -v
```

### **1.5.2 Instalar extensiones necesarias para Laravel**
```bash
# Instalar extensiones esenciales
sudo dnf install -y php-pdo php-pgsql php-mbstring php-xml php-curl php-zip php-gd php-intl php-bcmath php-soap php-xmlrpc php-redis php-memcached

# Verificar extensiones instaladas
php -m | grep -E "(pdo|pgsql|mbstring|xml|curl|zip|gd|intl|bcmath)"
```

### **1.5.3 Configurar PHP para Laravel**
```bash
# Editar configuración de PHP
sudo nano /etc/php.ini

# Modificar estas configuraciones:
max_execution_time = 300
max_input_time = 300
memory_limit = 512M
post_max_size = 100M
upload_max_filesize = 100M
max_file_uploads = 50
date.timezone = America/Mexico_City
```

### **1.5.4 Instalar Composer**
```bash
# Descargar Composer
curl -sS https://getcomposer.org/installer | php

# Mover a directorio global
sudo mv composer.phar /usr/local/bin/composer

# Verificar instalación
composer --version
```

### **1.5.5 Instalar Node.js y npm**
```bash
# Instalar Node.js 18+ (LTS)
curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo bash -
sudo dnf install -y nodejs

# Verificar instalación
node --version
npm --version
```

**✅ Checkpoint:** PHP, Composer y Node.js configurados

---

## 💾 PASO 1.6: CONFIGURAR BACKUP BÁSICO
**Tiempo estimado:** 30 minutos

### **1.6.1 Crear directorio de backups**
```bash
sudo mkdir -p /backups/postgresql
sudo chown postgres:postgres /backups/postgresql
```

### **1.6.2 Crear script de backup**
```bash
sudo nano /usr/local/bin/backup_erp.sh
```

```bash
#!/bin/bash
# Script de backup para ERP Grupo Fixum

# Variables
DB_NAME="fixum_erp"
DB_USER="fixum_admin"
BACKUP_DIR="/backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/fixum_erp_$DATE.sql"

# Crear backup
export PGPASSWORD="FixumERP2025!@#"
pg_dump -h localhost -U $DB_USER -d $DB_NAME > $BACKUP_FILE

# Comprimir backup
gzip $BACKUP_FILE

# Eliminar backups más antiguos de 7 días
find $BACKUP_DIR -name "fixum_erp_*.sql.gz" -mtime +7 -delete

echo "Backup completado: $BACKUP_FILE.gz"
```

### **1.6.3 Hacer ejecutable y probar**
```bash
sudo chmod +x /usr/local/bin/backup_erp.sh

# Probar el script
sudo -u postgres /usr/local/bin/backup_erp.sh

# Verificar que se creó el backup
ls -la /backups/postgresql/
```

### **1.6.4 Configurar cron para backup automático**
```bash
sudo crontab -e

# Agregar esta línea (backup diario a las 2:00 AM)
0 2 * * * /usr/local/bin/backup_erp.sh
```

**✅ Checkpoint:** Sistema de backup configurado

---

## 🔍 PASO 1.7: TESTING Y VERIFICACIÓN FINAL
**Tiempo estimado:** 30 minutos

### **1.7.1 Test de conexión completo**
```bash
# Crear script de prueba
nano /tmp/test_db_connection.php
```

```php
<?php
// Test de conexión PostgreSQL
$host = 'localhost';
$port = '5432';
$dbname = 'fixum_erp';
$user = 'fixum_admin';
$password = 'FixumERP2025!@#';

try {
    $pdo = new PDO("pgsql:host=$host;port=$port;dbname=$dbname", $user, $password);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    
    echo "✅ Conexión exitosa a PostgreSQL\n";
    
    // Test de schemas
    $schemas = ['system', 'kreis', 'gosan', 'altaguarda', 'dtc_coatzacoalcos', 'maha', 'milenio', 'acierta', 'jaguar', 'shared'];
    
    foreach ($schemas as $schema) {
        $stmt = $pdo->query("SELECT schema_name FROM information_schema.schemata WHERE schema_name = '$schema'");
        if ($stmt->rowCount() > 0) {
            echo "✅ Schema '$schema' existe\n";
        } else {
            echo "❌ Schema '$schema' NO existe\n";
        }
    }
    
    // Test de tabla system
    $pdo->exec("SET search_path TO system, public");
    $stmt = $pdo->query("SELECT * FROM test_connection LIMIT 1");
    $result = $stmt->fetch(PDO::FETCH_ASSOC);
    
    if ($result) {
        echo "✅ Tabla de prueba system funciona: " . $result['message'] . "\n";
    }
    
    // Test de tabla tenant
    $pdo->exec("SET search_path TO kreis, public");
    $stmt = $pdo->query("SELECT * FROM test_tenant LIMIT 1");
    $result = $stmt->fetch(PDO::FETCH_ASSOC);
    
    if ($result) {
        echo "✅ Tabla de prueba tenant funciona: " . $result['message'] . "\n";
    }
    
    echo "\n🎉 Todos los tests pasaron correctamente!\n";
    
} catch (PDOException $e) {
    echo "❌ Error de conexión: " . $e->getMessage() . "\n";
}
?>
```

```bash
# Ejecutar test
php /tmp/test_db_connection.php
```

### **1.7.2 Verificar servicios**
```bash
# Verificar que todos los servicios estén corriendo
sudo systemctl status postgresql
sudo systemctl status lsws  # LiteSpeed

# Verificar puertos
netstat -tlnp | grep :5432  # PostgreSQL
netstat -tlnp | grep :80    # HTTP
netstat -tlnp | grep :443   # HTTPS
```

### **1.7.3 Test de performance básico**
```bash
# Test básico de performance PostgreSQL
psql -h localhost -U fixum_admin -d fixum_erp -c "SELECT version();"
psql -h localhost -U fixum_admin -d fixum_erp -c "SHOW shared_buffers;"
psql -h localhost -U fixum_admin -d fixum_erp -c "SHOW max_connections;"
```

**✅ Checkpoint:** Todos los tests pasan correctamente

---

## 📋 CHECKLIST FINAL FASE 1

### **✅ Base de Datos:**
- [ ] PostgreSQL 13.20 funcionando
- [ ] Usuario `fixum_admin` creado
- [ ] Base de datos `fixum_erp` creada
- [ ] 10 schemas creados (8 empresas + shared + system)
- [ ] Permisos configurados correctamente
- [ ] Tablas de prueba funcionando

### **✅ Seguridad:**
- [ ] `postgresql.conf` optimizado
- [ ] `pg_hba.conf` configurado para seguridad
- [ ] Conexiones limitadas a localhost
- [ ] Passwords encriptados

### **✅ Infraestructura Web:**
- [ ] Subdominio configurado
- [ ] SSL habilitado
- [ ] PHP 8.2+ con extensiones
- [ ] Composer instalado
- [ ] Node.js instalado

### **✅ Backup:**
- [ ] Script de backup creado
- [ ] Cron job configurado
- [ ] Directorio de backups creado
- [ ] Backup de prueba realizado

### **✅ Testing:**
- [ ] Conexión PostgreSQL funciona
- [ ] Todos los schemas accesibles
- [ ] Switch entre schemas funciona
- [ ] Servicios corriendo correctamente

---

## 🎯 ENTREGABLES DE LA FASE 1

1. **PostgreSQL Multi-Tenant** completamente configurado
2. **Subdominio con SSL** listo para desarrollo
3. **Script de backup** automático funcionando
4. **Documentación** de configuración
5. **Scripts de testing** para verificación

---

## 🚀 PRÓXIMO PASO: FASE 2

Una vez completada la Fase 1, estaremos listos para:
- Instalar Laravel 12 en el subdominio
- Configurar la arquitectura hexagonal
- Conectar Laravel con PostgreSQL multi-tenant
- Configurar el sistema de migraciones por schema

**¿Empezamos con el Paso 1.1 de configuración de PostgreSQL?**