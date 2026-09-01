# Etapa 0 — Preparación del laboratorio

## Objetivo

Preparar y validar un entorno Linux capaz de ejecutar Moodle antes de comenzar con la instalación manual de la primera instancia.

Esta etapa busca conocer las dependencias reales del sistema y dejar un laboratorio reproducible sobre el cual experimentar durante las siguientes fases del proyecto.

---

## Entorno utilizado

| Componente | Versión / configuración |
|---|---|
| Sistema operativo | Ubuntu 22.04.5 LTS (Jammy Jellyfish) |
| Apache | 2.4.52 |
| PHP | 8.1.2 |
| MariaDB | 10.6.23 |
| `max_input_vars` | 5000 |

Para esta primera instalación se utilizará **Moodle 4.5**, evitando modificar innecesariamente el stack base de Ubuntu durante la exploración inicial.

---

## Validaciones realizadas

### 1. Sistema operativo

```bash
lsb_release -a
cat /etc/os-release
```

Se confirmó que el laboratorio utiliza Ubuntu 22.04.5 LTS.

### 2. Versiones de los servicios principales

```bash
php -v
mariadb --version
apache2 -v
```

Esto permitió conocer el stack disponible antes de seleccionar una versión de Moodle compatible.

### 3. Extensiones de PHP

```bash
php -m | sort
```

Se verificó la disponibilidad de extensiones necesarias para Moodle, entre ellas:

- `curl`
- `dom`
- `fileinfo`
- `gd`
- `intl`
- `mbstring`
- `mysqli`
- `soap`
- `sodium`
- `xml`
- `zip`

### 4. Configuración de `max_input_vars`

El valor inicial era:

```text
max_input_vars = 1000
```

Se modificaron los archivos:

```text
/etc/php/8.1/apache2/php.ini
/etc/php/8.1/cli/php.ini
```

dejando:

```ini
max_input_vars = 5000
```

Luego se reinició Apache:

```bash
sudo systemctl restart apache2
```

y se validó tanto PHP CLI como PHP ejecutado mediante Apache:

```bash
php -i | grep max_input_vars
curl http://localhost/check.php
```

Resultado:

```text
5000
```

> Durante esta configuración se detectó que una línea iniciada con `;` en `php.ini` permanece comentada y, por lo tanto, PHP ignora su valor. La configuración solo se aplicó después de eliminar ese carácter.

### 5. Estado de Apache y MariaDB

```bash
sudo systemctl status apache2 --no-pager
sudo systemctl status mariadb --no-pager
```

Ambos servicios quedaron activos y funcionando correctamente.

También se comprobó que Apache tiene cargado el módulo de PHP:

```bash
apache2ctl -M | grep php
```

---

## Resultado de la etapa

El laboratorio quedó preparado para comenzar la instalación manual de Moodle:

```text
Ubuntu 22.04.5
│
├── Apache 2.4.52        ✓
├── PHP 8.1.2            ✓
│   ├── extensiones      ✓
│   └── max_input_vars   ✓ 5000
└── MariaDB 10.6.23      ✓
```

---

## Relevancia para el provisioner

Esta etapa permitió identificar una primera responsabilidad del futuro sistema: **validar el entorno antes de intentar crear un tenant**.

Antes del aprovisionamiento, la solución deberá poder comprobar condiciones como:

- disponibilidad del motor de base de datos;
- disponibilidad del servidor web;
- compatibilidad de PHP;
- presencia de dependencias necesarias;
- permisos y configuración del sistema.

Estas comprobaciones pueden formar parte de una etapa de *pre-flight checks* previa a la creación de una instancia.

---

## Siguiente etapa

**Etapa 1 — Instalación manual de una instancia Moodle**

El siguiente objetivo es crear desde cero una primera instancia y documentar los recursos que requiere:

```text
Base de datos
    ↓
Usuario de base de datos
    ↓
Código Moodle
    ↓
moodledata
    ↓
Servidor web
    ↓
Instalación
    ↓
cron
    ↓
Verificación
```

Cada paso se documentará indicando qué se hizo, por qué fue necesario, cuál fue el resultado esperado y qué implicancia tiene para la futura automatización.
