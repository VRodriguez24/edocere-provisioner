# Etapa 02 — Comparación técnica: Moodle nativo vs Docker

## 1. Objetivo

El objetivo de esta etapa fue evaluar experimentalmente la viabilidad de ejecutar una instancia Moodle mediante Docker y compararla con una instalación nativa equivalente sobre Ubuntu.

La etapa **no busca decidir todavía la arquitectura multitenant definitiva de Edocere**. Su propósito es caracterizar, con evidencia práctica, las ventajas, costos y dificultades de ambos enfoques antes de diseñar la topología multitenant de la siguiente etapa.

La pregunta evaluada fue:

> ¿Docker es técnicamente viable para ejecutar Moodle y qué diferencias introduce respecto de una instalación nativa equivalente?

La decisión final sobre **dónde usar Docker**, qué componentes contenerizar y qué recursos compartir entre tenants queda pendiente para la Etapa 03 — Multitenant.

---

## 2. Control del experimento

Para reducir diferencias ajenas al mecanismo de despliegue, ambas instalaciones utilizaron el mismo código Moodle.

### Moodle

- Versión funcional observada: **Moodle 4.5.13+**
- Rama: `MOODLE_405_STABLE`
- Commit utilizado:

```text
c7cedd4c4a16d68b42b0db7c357e84764ea525f3
```

### Versiones de infraestructura

| Componente | Nativo | Docker |
|---|---|---|
| PHP | 8.1.2-1ubuntu2.25 | 8.1.34 |
| MariaDB | 10.6.23 | 10.6.28 |
| Moodle | 4.5.13+ | 4.5.13+ |
| Commit Moodle | mismo | mismo |

Las versiones major/minor de PHP y MariaDB son equivalentes, pero los patches no son idénticos. Esta diferencia se considera una limitación experimental y se declara explícitamente.

---

## 3. Instalación nativa utilizada como baseline

La instalación nativa fue implementada directamente sobre Ubuntu 22.04.

Su organización general fue:

```text
Ubuntu
├── Apache
├── PHP
├── MariaDB
│   └── moodle_tenant1
├── Moodle
│   └── /var/www/moodle-tenant1
├── moodledata
│   └── /var/moodledata/tenant1
└── cron
    └── /etc/cron.d/moodle-tenant1
```

Cada tenant mantiene una base de datos independiente y un directorio `moodledata` independiente.

La instancia nativa quedó validada con HTTP funcional, 494 tablas en la base de datos, `config.php` válido, cron operativo, código no escribible por el usuario web, `moodledata` escribible por `www-data` y un curso funcional de prueba.

---

## 4. Instalación Docker experimental

La instalación Docker utilizó Docker Compose con tres servicios:

```text
Docker Compose
├── web
│   ├── Apache
│   ├── PHP
│   └── Moodle
├── cron
│   └── Moodle CLI
└── db
    └── MariaDB
```

### Persistencia

El servicio web utiliza:

```text
./moodle                 -> /var/www/html
moodledata_data          -> /var/moodledata
```

La base de datos utiliza:

```text
mariadb_data             -> /var/lib/mysql
```

El código Moodle se mantuvo como bind mount para conservar exactamente el mismo código utilizado en la instalación nativa.

### Red

Docker Compose creó una red privada para los servicios.

Moodle se conecta a MariaDB mediante:

```text
db:3306
```

y no mediante `localhost`.

La resolución DNS interna fue comprobada experimentalmente:

```text
172.22.0.2      db
```

---

## 5. Validación funcional Docker

| Prueba | Resultado |
|---|---|
| Servicio `db` | healthy |
| Servicio `web` | operativo |
| Servicio `cron` | operativo |
| HTTP | 200 OK |
| `config.php` | sin errores de sintaxis |
| Host DB en Moodle | `db` |
| Base de datos | `moodle_docker` |
| Tablas | 494 |
| `moodledata` | escribible |
| Código Moodle | no escribible por `www-data` |
| `config.php` | legible, no escribible por `www-data` |
| Curso funcional | correcto |
| DNS interno | correcto |

La instancia Docker quedó funcionalmente equivalente a la instalación nativa para los objetivos del laboratorio.

---

## 6. Cron

### 6.1 Cron Docker

Se decidió ejecutar el cron como un servicio independiente y no instalar un daemon cron dentro del contenedor web.

El comportamiento configurado fue:

```text
cron.php --keep-alive=55
sleep 5
repetir
```

El proceso Moodle se ejecuta como `www-data`, mientras que la imagen puede completar previamente su inicialización con los permisos necesarios.

La ejecución fue validada mediante mensajes:

```text
Cron run completed correctly
```

### 6.2 Problema encontrado con permisos Docker

Inicialmente se configuró el servicio completo como:

```yaml
user: "33:33"
```

Esto provocó el error:

```text
/usr/local/etc/php/conf.d/20-local.ini: Permission denied
```

La causa fue que el entrypoint de la imagen necesitaba permisos para preparar configuración PHP antes de ejecutar el proceso final.

La solución fue permitir la inicialización normal del contenedor y posteriormente ejecutar Moodle cron como `www-data`.

Este problema demuestra que Docker agrega una capa operacional adicional asociada a entrypoints, usuarios internos, permisos y ciclo de vida del contenedor.

### 6.3 Problema encontrado con cron nativo

La configuración nativa inicial ejecutaba:

```text
* * * * * www-data /usr/bin/php /var/www/moodle-tenant1/admin/cli/cron.php
```

Sin embargo, Moodle mantenía cada proceso vivo aproximadamente 180 segundos. Como el sistema iniciaba una nueva ejecución cada minuto, se observaron varias instancias simultáneas de `cron.php`.

La configuración fue corregida a:

```text
* * * * * www-data /usr/bin/php /var/www/moodle-tenant1/admin/cli/cron.php --keep-alive=55 >/dev/null 2>&1
```

Después de la corrección se observó una sola ejecución nativa activa por ciclo.

Este hallazgo es relevante para el futuro Provisioner: no basta con crear una entrada cron; también debe controlarse la duración y concurrencia del proceso.

---

## 7. Hardening y permisos

### Nativo

El código Moodle fue dejado con permisos equivalentes a:

```text
directorios      root:root       755
archivos         root:root       644
config.php       root:www-data   640
moodledata       www-data        escribible
```

### Docker

Se aplicó el mismo principio sobre el bind mount del código.

La comprobación como `www-data` produjo:

```text
www-data puede leer config.php: OK
www-data no puede modificar config.php: OK
www-data no puede modificar codigo: OK
www-data puede escribir moodledata: OK
```

Por lo tanto, ambos enfoques permiten implementar el mismo principio básico:

> El proceso web puede escribir en `moodledata`, pero no debe poder modificar el código Moodle ni `config.php`.

---

## 8. Prueba de persistencia Docker

Se creó un archivo de prueba dentro de `/var/moodledata` y posteriormente se ejecutó:

```text
docker compose down
```

Esto eliminó los contenedores web, cron y MariaDB, además de la red Docker.

Los volúmenes permanecieron:

```text
edocere-moodle-docker_mariadb_data
edocere-moodle-docker_moodledata_data
```

Luego se ejecutó:

```text
docker compose up -d
```

Los tres servicios fueron recreados.

Después de la recreación se comprobó:

```text
MariaDB healthy                OK
494 tablas                     OK
moodledata persistente         OK
Moodle seguía instalado        OK
HTTP funcional                 OK
cron nuevamente operativo      OK
```

Esto demuestra experimentalmente la separación entre runtime reemplazable y estado persistente.

---

## 9. Mediciones

### 9.1 Código, datos y base

| Métrica | Nativo | Docker |
|---|---:|---:|
| Código Moodle | 401 MB | 401 MB |
| moodledata | 35 MB | 34 MB |
| Base de datos | 15.66 MB | 14.00 MB |
| Tablas | 494 | 494 |

Las pequeñas diferencias de tamaño en `moodledata` y base de datos son esperables debido a diferencias de actividad realizadas durante las pruebas.

### 9.2 Imágenes Docker

| Imagen | Tamaño |
|---|---:|
| `moodlehq/moodle-php-apache:8.1` | 1.19 GB |
| `mariadb:10.6` | 309 MB |

Estos tamaños no deben interpretarse como costo por tenant. Las capas de imagen son reutilizadas por Docker entre múltiples contenedores que utilizan la misma imagen.

### 9.3 Memoria y CPU Docker

Se tomaron cinco muestras consecutivas mediante `docker stats`.

| Servicio | RAM promedio | CPU promedio |
|---|---:|---:|
| web | 99.05 MiB | 0.016 % |
| cron | 31.50 MiB | 0.794 % |
| db | 126.78 MiB | 0.906 % |
| **Total experimental** | **257.33 MiB** | **1.716 %** |

La RAM total observada fue estable, aproximadamente entre 256.6 y 257.9 MiB durante las muestras.

Estos valores representan **la topología experimental completa web + cron + DB para una instancia**, no una estimación definitiva del costo por tenant.

### 9.4 Memoria nativa

Los servicios nativos son servicios del host que pueden ser compartidos por múltiples instancias Moodle, por lo que no se consideran directamente equivalentes al consumo Docker por tenant.

Además, la medición inicial de `cron.service` incluía varias ejecuciones Moodle simultáneas debido al problema de `keep-alive`.

Después de corregir el cron nativo se observó:

```text
cron.service ≈ 123.82 MiB
TasksCurrent = 4
```

Este dato se conserva como evidencia del entorno, pero no se utiliza para afirmar un costo marginal por tenant.

---

## 10. Comparación cualitativa

| Criterio | Nativo | Docker |
|---|---|---|
| Moodle funcional | Sí | Sí |
| Mismo código Moodle | Sí | Sí |
| DB independiente | Sí | Sí |
| `moodledata` independiente | Sí | Sí |
| Cron funcional | Sí | Sí |
| Dependencias PHP | Host | Encapsuladas |
| Configuración | Distribuida en SO | Más declarativa |
| Aislamiento de procesos | Menor | Mayor |
| Red privada entre servicios | Configuración manual | Integrada en Compose |
| Persistencia | Filesystem/DB del host | Volúmenes + bind mounts |
| Recreación del runtime | Más manual | Probada con `down/up` |
| Portabilidad del runtime | Menor | Mayor |
| Complejidad inicial | Menor | Mayor |
| Permisos | Linux/host | Linux + contenedor |
| Networking | Host | Host + red Docker |
| EntryPoints | No aplica | Sí |
| Volúmenes | No aplica | Sí |
| Imágenes | No aplica | Sí |
| Automatización futura | Viable | Viable y más declarativa |

---

## 11. Ventajas observadas de Docker

**Reproducibilidad.** La estructura de servicios puede declararse en `compose.yaml`.

**Encapsulamiento de dependencias.** El entorno PHP/Apache no depende directamente de los paquetes instalados en el host.

**Separación de procesos.** Web, cron y base de datos pueden operar como unidades independientes.

**Networking interno.** Los servicios pueden comunicarse mediante nombres lógicos como `db`.

**Recreación del runtime.** Los contenedores pudieron eliminarse y reconstruirse conservando los datos.

**Portabilidad.** La descripción del entorno puede utilizarse en otro host compatible con Docker con menos configuración manual del sistema operativo.

---

## 12. Costos y dificultades observados de Docker

**Permisos y usuarios.** Fue necesario comprender la interacción entre usuario del host, usuario del contenedor y entrypoint.

**Volúmenes y bind mounts.** El operador debe distinguir claramente entre datos persistentes y runtime reemplazable.

**Networking.** `localhost` deja de representar necesariamente la base de datos; aparecen redes y DNS internos.

**EntryPoints.** La imagen ejecuta lógica propia antes del comando definido por el usuario.

**Almacenamiento de imágenes.** Las imágenes agregan almacenamiento adicional, aunque sus capas pueden compartirse.

**Mayor número de conceptos operacionales.** El equipo debe comprender imágenes, contenedores, redes, volúmenes y Compose además de Moodle, PHP, Apache y MariaDB.

---

## 13. Limitaciones del experimento

La topología Docker probada utiliza:

```text
web + cron + MariaDB
```

para una sola instancia experimental.

Esto **no implica** que la arquitectura final vaya a utilizar una base MariaDB por tenant.

Edocere ha planteado un modelo donde un mismo motor MariaDB puede mantener una base independiente por Moodle. Por ello, una posible arquitectura futura podría compartir MariaDB y contenerizar únicamente determinados componentes por tenant.

Ejemplo conceptual:

```text
                MariaDB compartido
             ┌──────────────────────┐
             │ DB tenant1           │
             │ DB tenant2           │
             │ DB tenant3           │
             └──────▲────▲────▲─────┘
                    │    │    │
              ┌─────┘    │    └─────┐
              │          │          │
         Moodle T1   Moodle T2   Moodle T3
         web/cron    web/cron    web/cron
```

Por este motivo, las métricas del Docker experimental no deben extrapolarse directamente a 10, 100 o 200 tenants.

La topología multitenant y la ubicación exacta de Docker dentro de la arquitectura se evaluarán en la siguiente etapa.

---

## 14. Conclusión

El experimento demuestra que **Docker es técnicamente viable para ejecutar Moodle** y que puede entregar ventajas comprobables en aislamiento, encapsulamiento de dependencias, configuración declarativa, networking interno, persistencia y reconstrucción del runtime.

Al mismo tiempo, Docker introduce una complejidad operacional adicional asociada a imágenes, redes, volúmenes, entrypoints, permisos y administración del ciclo de vida de los contenedores.

Por lo tanto, esta etapa **no concluye que Docker deba adoptarse obligatoriamente para toda la arquitectura**.

La conclusión de la Etapa 02 es:

> Docker es una alternativa técnicamente viable para Moodle y sus principales trade-offs fueron caracterizados experimentalmente. La decisión final sobre su adopción y ubicación en la arquitectura debe considerar la topología multitenant, especialmente qué componentes serán dedicados por tenant y cuáles serán compartidos.

La decisión arquitectónica se continuará en:

```text
Etapa 03 — Multitenant
```

donde se evaluará la organización práctica de múltiples tenants y se comparará una topología nativa frente a alternativas Docker o híbridas más cercanas al escenario real de Edocere.
