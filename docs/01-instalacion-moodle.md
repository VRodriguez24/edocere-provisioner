# Etapa 1 — Instalación manual de una instancia Moodle

## Objetivo

Instalar manualmente una primera instancia Moodle para comprender qué recursos, configuraciones, validaciones y operaciones serán necesarias antes de automatizar su aprovisionamiento.

Esta etapa utiliza una instalación **nativa** sobre Ubuntu/WSL. La comparación con Docker se realizará en la Etapa 2.

---

## Instancia de prueba

| Recurso | Valor |
|---|---|
| Tenant | `tenant1` |
| Moodle instalado | **4.5.13+** — Build `20260818` |
| Rama utilizada | `MOODLE_405_STABLE` |
| Base de datos | `moodle_tenant1` |
| Usuario DB | `moodle_tenant1` |
| Código desplegado | `/var/www/moodle-tenant1` |
| Datos | `/var/moodledata/tenant1` |
| Servidor web | Apache 2.4.52 |
| PHP | 8.1.2 |
| MariaDB | 10.6.23 |
| URL del laboratorio | `http://172.26.45.117` |

> Las contraseñas utilizadas en el laboratorio no se almacenan en el repositorio.

---

## Estado de la etapa

```text
[✅] 1. Crear base de datos y usuario
[✅] 2. Obtener código de Moodle
[✅] 3. Desplegar código del tenant
[✅] 4. Crear y validar moodledata
[✅] 5. Configurar Apache
[✅] 6. Instalar Moodle
[✅] 7. Configurar administrador y sitio
[✅] 8. Configurar cron
[✅] 9. Endurecer permisos
[✅] 10. Validar funcionalmente la instancia
```

**Resultado: Etapa 1 completada.**

---

## 1. Base de datos

Se creó una base independiente para el tenant y un usuario limitado exclusivamente a ella:

```sql
CREATE DATABASE moodle_tenant1
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'moodle_tenant1'@'localhost'
  IDENTIFIED BY '<PASSWORD_LOCAL>';

GRANT ALL PRIVILEGES
ON moodle_tenant1.*
TO 'moodle_tenant1'@'localhost';
```

### Verificación

```sql
SHOW GRANTS FOR 'moodle_tenant1'@'localhost';
```

También se verificó iniciando sesión como `moodle_tenant1`: el usuario puede trabajar con su propia base, pero no administrar usuarios ni otras bases del servidor.

### Implicancia para el provisioner

El provisioner deberá crear la base y credenciales de forma consistente, limitar los privilegios al tenant correspondiente, validar la conexión antes de continuar y evitar que secretos aparezcan en logs o en Git.

---

## 2. Código de Moodle

El repositorio oficial de Moodle se mantuvo separado del proyecto:

```text
~/edocere-provisioner/   → proyecto Edocere
~/moodle/                → repositorio oficial Moodle
```

Se obtuvo el código y se seleccionó la rama 4.5:

```bash
cd ~
git clone https://github.com/moodle/moodle.git
cd ~/moodle
git checkout MOODLE_405_STABLE
```

Durante el instalador se identificó la versión concreta:

```text
Moodle 4.5.13+ (Build: 20260818)
```

### Implicancia para el provisioner

La versión de Moodle debe ser explícita y reproducible. Para una implementación productiva será preferible trabajar con una versión o tag fijado en lugar de depender de una rama flotante.

---

## 3. Despliegue del tenant

```bash
sudo mkdir -p /var/www/moodle-tenant1
sudo rsync -a --delete --exclude='.git' ~/moodle/ /var/www/moodle-tenant1/
```

Se comprobó que `.git` no fue desplegado y que Apache podía leer el código sin modificarlo.

```bash
sudo -u www-data test -r /var/www/moodle-tenant1/index.php
sudo -u www-data test -w /var/www/moodle-tenant1/index.php
```

### Implicancia para el provisioner

El proceso deberá desplegar una versión conocida, excluir archivos innecesarios, aplicar permisos controlados y validar el acceso con el usuario real del servidor web.

---

## 4. `moodledata`

```bash
sudo mkdir -p /var/moodledata/tenant1
sudo chown -R www-data:www-data /var/moodledata/tenant1
sudo chmod 770 /var/moodledata/tenant1
```

Se verificó lectura y escritura ejecutando la prueba como `www-data`.

### Aprendizaje

```text
Código Moodle   → Apache debe leer, pero no modificar
moodledata      → Apache debe leer y escribir
```

### Implicancia para el provisioner

Cada tenant deberá contar con un `moodledata` aislado y con permisos distintos a los del código.

---

## 5. Apache

Se creó:

```text
/etc/apache2/sites-available/moodle-tenant1.conf
```

con:

```apache
<VirtualHost *:80>
    ServerName moodle1.local
    DocumentRoot /var/www/moodle-tenant1

    <Directory /var/www/moodle-tenant1>
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/moodle-tenant1-error.log
    CustomLog ${APACHE_LOG_DIR}/moodle-tenant1-access.log combined
</VirtualHost>
```

Luego:

```bash
sudo a2ensite moodle-tenant1.conf
sudo a2enmod rewrite
sudo apache2ctl configtest
sudo systemctl reload apache2
```

La configuración respondió:

```text
Syntax OK
```

Antes de la instalación Moodle respondió:

```text
HTTP/1.1 302 Found
Location: install.php
```

### Particularidad del laboratorio WSL

Apache se ejecuta en WSL mientras el navegador se ejecuta en Windows. La resolución de `moodle1.local` entre ambos entornos generó complejidad adicional.

Se comprobó que el VirtualHost funcionaba enviando explícitamente el encabezado `Host`, pero para completar esta primera exploración se utilizó directamente la IP de WSL.

Se deshabilitó el sitio por defecto para que la IP resolviera al tenant:

```bash
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

La IP utilizada durante esta ejecución fue:

```text
172.26.45.117
```

> La IP de WSL es temporal y puede cambiar. Esta dificultad pertenece al laboratorio y no al diseño de producción. Los dominios por tenant se retomarán durante la Etapa 3 — Multitenant.

---

## 6. Instalación de Moodle

### 6.1 Permiso temporal para crear `config.php`

El instalador web necesitaba crear `config.php` en la raíz. Para permitirlo se cambió temporalmente el propietario **solo del directorio raíz**:

```bash
sudo chown www-data:www-data /var/www/moodle-tenant1
```

Los archivos existentes continuaron protegidos.

### 6.2 Parámetros utilizados

| Campo | Valor |
|---|---|
| Dirección web | `http://172.26.45.117` |
| Directorio Moodle | `/var/www/moodle-tenant1` |
| Directorio de datos | `/var/moodledata/tenant1` |
| Controlador DB | MariaDB nativo (`mariadb`) |
| Host DB | `localhost` |
| Base de datos | `moodle_tenant1` |
| Usuario DB | `moodle_tenant1` |
| Contraseña DB | credencial local |
| Prefijo | `mdl_` |
| Puerto | valor por defecto |
| Socket Unix | valor por defecto |

Moodle validó el entorno y confirmó:

```text
Su entorno de servidor cumple todos los requerimientos mínimos.
```

La única advertencia fue el uso de HTTP en lugar de HTTPS, aceptado únicamente para este laboratorio local.

### 6.3 `config.php` generado

Se verificaron solo los campos no sensibles:

```bash
sudo grep -E "dbtype|dbname|wwwroot|dataroot" /var/www/moodle-tenant1/config.php
```

Resultado:

```text
$CFG->dbtype    = 'mariadb';
$CFG->dbname    = 'moodle_tenant1';
$CFG->wwwroot   = 'http://172.26.45.117';
$CFG->dataroot  = '/var/moodledata/tenant1';
```

### Implicancia para el provisioner

El provisioner deberá separar claramente parámetros entregados por el portal, valores derivados a partir del tenant, configuración fija de infraestructura y secretos generados o recibidos.

---

## 7. Administrador y configuración inicial del sitio

Se creó el administrador principal del laboratorio y se configuró el sitio como:

```text
Nombre completo: Moodle Tenant 1 - Edocere
Nombre corto:    Tenant1
Zona horaria:    America/Santiago
```

Las credenciales administrativas no se almacenan en Git.

### Implicancia para el provisioner

Parte de estos valores probablemente llegará desde el portal o la API; otros podrán definirse mediante defaults o valores derivados por Edocere.

---

## 8. Cron

Moodle requiere tareas programadas para sus procesos en segundo plano.

Se creó:

```bash
sudo tee /etc/cron.d/moodle-tenant1 >/dev/null <<'CRON'
* * * * * www-data /usr/bin/php /var/www/moodle-tenant1/admin/cli/cron.php >/dev/null 2>&1
CRON

sudo chmod 644 /etc/cron.d/moodle-tenant1
sudo systemctl enable --now cron
```

### Validación manual

```bash
sudo -u www-data /usr/bin/php /var/www/moodle-tenant1/admin/cli/cron.php
```

La ejecución finalizó correctamente:

```text
Cron run completed correctly
```

### Validación automática

```bash
sudo journalctl -u cron --since "5 minutes ago" --no-pager | grep moodle-tenant1
```

Se observaron ejecuciones consecutivas cada minuto como `www-data`.

### Implicancia para el provisioner

Cada tenant deberá quedar con sus tareas periódicas configuradas y verificadas antes de declararse operativo.

---

## 9. Endurecimiento de permisos

Después de la instalación se revirtieron los permisos temporales:

```bash
sudo chown -R root:root /var/www/moodle-tenant1
sudo find /var/www/moodle-tenant1 -type d -exec chmod 755 {} \;
sudo find /var/www/moodle-tenant1 -type f -exec chmod 644 {} \;

sudo chown root:www-data /var/www/moodle-tenant1/config.php
sudo chmod 640 /var/www/moodle-tenant1/config.php

sudo chown -R www-data:www-data /var/moodledata/tenant1
sudo chmod -R 770 /var/moodledata/tenant1
```

### Estado final observado

```text
/var/www/moodle-tenant1
→ root:root, 755

index.php
→ root:root, 644

config.php
→ root:www-data, 640

/var/moodledata/tenant1
→ www-data:www-data, 770
```

### Verificación funcional de permisos

Se comprobó que:

```text
Apache puede leer config.php:              OK
Apache no puede modificar config.php:      OK
Apache no puede modificar el código:       OK
Apache puede escribir en moodledata:       OK
```

### Implicancia para el provisioner

Los permisos finales forman parte del aprovisionamiento y no deben quedar como una tarea manual posterior.

---

## 10. Validación final

La instancia no se consideró terminada únicamente porque apareciera el dashboard.

### Servicios

```bash
systemctl is-active apache2
systemctl is-active mariadb
systemctl is-active cron
```

Resultado:

```text
active
active
active
```

### Apache

```bash
sudo apache2ctl configtest
```

Resultado:

```text
Syntax OK
```

### HTTP

```bash
curl -I http://172.26.45.117
```

Resultado:

```text
HTTP/1.1 200 OK
```

### Base de datos

```bash
sudo mariadb -e "
SELECT COUNT(*) AS tables_count
FROM information_schema.tables
WHERE table_schema='moodle_tenant1';
"
```

Resultado:

```text
tables_count
494
```

La cantidad concreta depende de la versión/build; lo importante es que el esquema fue creado correctamente.

### `config.php`

```bash
sudo -u www-data php -l /var/www/moodle-tenant1/config.php
```

Resultado:

```text
No syntax errors detected in /var/www/moodle-tenant1/config.php
```

### Cron automático

El journal confirmó ejecuciones consecutivas cada minuto:

```text
(www-data) CMD (/usr/bin/php /var/www/moodle-tenant1/admin/cli/cron.php ...)
```

### Prueba funcional

Se creó y abrió correctamente un curso:

```text
Nombre completo: Curso de prueba Tenant 1
Nombre corto:    TEST-T1
```

Esto valida una operación real a través de toda la cadena:

```text
Navegador
   ↓
Apache / PHP
   ↓
Moodle
   ↓
MariaDB
   +
moodledata
```

---

## Resultado de la Etapa 1

La instancia `tenant1` quedó funcional y validada:

```text
[✅] Apache activo
[✅] MariaDB activo
[✅] Cron activo
[✅] Configuración Apache válida
[✅] HTTP 200
[✅] Base de datos inicializada con 494 tablas
[✅] config.php válido y protegido
[✅] Código no escribible por Apache
[✅] moodledata escribible por Apache
[✅] Cron manual correcto
[✅] Cron automático cada minuto
[✅] Administrador funcional
[✅] Curso de prueba creado y accesible
```

Por lo tanto:

```text
tenant1 → ACTIVE
```

---

## Flujo aprendido para el futuro provisioner

La instalación manual permitió obtener una primera especificación concreta del ciclo de aprovisionamiento:

```text
Validar entorno
      ↓
Validar parámetros
      ↓
Crear DB y credenciales
      ↓
Desplegar código
      ↓
Crear moodledata
      ↓
Configurar servidor web
      ↓
Instalar Moodle
      ↓
Configurar administrador y sitio
      ↓
Configurar cron
      ↓
Aplicar permisos finales
      ↓
Validar infraestructura
      ↓
Validar funcionalmente
      ↓
ACTIVE
```

### Criterio preliminar de `ACTIVE`

Un tenant puede considerarse `ACTIVE` solamente después de verificar:

```text
Sitio HTTP responde
DB inicializada
config.php válido
moodledata escribible
cron operativo
permisos seguros
operación funcional básica
```

---

## Aprendizajes principales

1. La creación de una instancia Moodle involucra varios subsistemas: filesystem, Apache, PHP, MariaDB, Moodle y cron.
2. Estos subsistemas no comparten una única transacción; la automatización deberá controlar estados parciales, validaciones y recuperación ante fallas.
3. Los secretos deben mantenerse fuera del repositorio y fuera de logs.
4. Código y `moodledata` requieren políticas de permisos diferentes.
5. Los permisos temporales de instalación deben revertirse automáticamente.
6. Un tenant no está listo solo porque cargue la interfaz web.
7. La configuración de red y dominios del laboratorio WSL no representa directamente el ambiente productivo.
8. El proceso manual ya permite identificar inputs, valores derivados y validaciones que más adelante deberán formalizarse en el provisioner.

---

## Siguiente etapa

**Etapa 2 — Comparación instalación nativa vs. Docker**

Se repetirá una instalación equivalente utilizando Docker para comparar ambos enfoques con evidencia y criterios como:

- aislamiento;
- complejidad operativa;
- reproducibilidad;
- gestión de dependencias;
- consumo de recursos;
- seguridad;
- facilidad de aprovisionamiento;
- mantenimiento y actualización.

Después de esa comparación se avanzará a la convivencia de múltiples instancias en la Etapa 3.
