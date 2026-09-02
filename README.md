<div align="center">

# Edocere Provisioner

**Automatización del aprovisionamiento de instancias Moodle en un entorno multitenant**

`Moodle` · `Linux` · `MariaDB` · `Python` · `Docker`

</div>

---

## 🎯 Objetivo

Edocere ofrece **Moodle as a Service** sobre una infraestructura compartida.  
Este proyecto busca automatizar la creación de nuevas instancias Moodle de forma **reproducible, segura y aislada**, reduciendo el trabajo manual necesario para escalar el servicio.

Antes de automatizar, primero necesitamos entender qué implica realmente instalar y operar Moodle. Por eso el desarrollo parte desde instalaciones manuales, compara alternativas de despliegue, estudia la convivencia de múltiples instancias y recién después formaliza e implementa el aprovisionamiento.

```text
Entender → Documentar → Experimentar → Comparar → Diseñar → Automatizar
```

## 🧭 Plan técnico

| Etapa | Objetivo | Estado |
|---|---|:---:|
| **0 · Laboratorio** | Preparar Linux, Apache, PHP y MariaDB | ✅ |
| **1 · Primer Moodle** | Instalar una instancia nativa desde cero y documentar el proceso | ✅ |
| **2 · Alternativas** | Repetir una instalación equivalente con Docker y comparar ambos enfoques | ✅ |
| **3 · Multitenant** | Ejecutar varias instancias y estudiar aislamiento y recursos compartidos | ⬜ |
| **4 · Aprovisionamiento** | Definir inputs, pasos, estados, validaciones, errores e idempotencia | ⬜ |
| **5 · Automatización** | Construir el provisioner a partir del proceso validado | ⬜ |

## ✅ Resultado de la Etapa 1

La primera instalación nativa quedó operativa y validada funcionalmente.

| Recurso / validación | Resultado |
|---|---|
| Tenant | `tenant1` |
| Moodle | **4.5.13+** — Build `20260818` |
| Rama utilizada | `MOODLE_405_STABLE` |
| Base de datos | `moodle_tenant1` |
| Usuario DB | `moodle_tenant1` |
| Código | `/var/www/moodle-tenant1` |
| Datos | `/var/moodledata/tenant1` |
| URL de laboratorio | `http://172.26.45.117` |
| Tablas creadas | **494** |
| Apache | activo / `Syntax OK` |
| MariaDB | activo |
| Cron | activo y ejecutándose cada minuto |
| HTTP | `200 OK` |
| Curso de prueba | `Curso de prueba Tenant 1` / `TEST-T1` |
| Permisos | código protegido y `moodledata` escribible por Apache |

La IP corresponde únicamente al laboratorio WSL y puede cambiar. La configuración de dominios por tenant se retomará al estudiar el escenario multitenant.

## ✅ Resultado de la Etapa 2

Se implementó una segunda instancia Moodle equivalente utilizando Docker Compose y se comparó experimentalmente con la instalación nativa de la Etapa 1.

| Recurso / validación | Nativo | Docker |
|---|:---:|:---:|
| Moodle 4.5.13+ | ✅ | ✅ |
| Mismo commit Moodle | ✅ | ✅ |
| Tablas creadas | 494 | 494 |
| HTTP funcional | ✅ | ✅ |
| Cron operativo | ✅ | ✅ |
| Código protegido | ✅ | ✅ |
| `moodledata` escribible | ✅ | ✅ |
| Curso de prueba | ✅ | ✅ |
| Persistencia tras recrear runtime | — | ✅ |
| DNS interno entre servicios | — | ✅ |

La prueba Docker utilizó tres servicios: `web`, `cron` y `db`. También se comprobó que los contenedores pueden eliminarse y recrearse sin perder la base de datos ni `moodledata`.

Docker mostró ventajas en aislamiento, encapsulamiento de dependencias, configuración declarativa y reconstrucción del runtime, pero también introdujo complejidad adicional asociada a redes, volúmenes, permisos y entrypoints.

La Etapa 2 **no decide todavía que Docker sea la arquitectura final**. La ubicación y organización de los contenedores se evaluará en la Etapa 3 al estudiar el escenario multitenant y distinguir recursos dedicados de recursos compartidos.

Los resultados completos se encuentran en [`docs/02-comparacion-native-docker.md`](docs/02-comparacion-native-docker.md).

## ⚙️ Flujo de aprovisionamiento aprendido

La Etapa 1 permitió transformar una instalación manual en una primera especificación del proceso que deberá automatizarse:

```text
Solicitud de creación
        │
        ▼
Validar entorno y parámetros
        │
        ▼
Crear base de datos y credenciales
        │
        ▼
Desplegar código Moodle
        │
        ▼
Crear moodledata
        │
        ▼
Configurar servidor web
        │
        ▼
Instalar y configurar Moodle
        │
        ▼
Configurar tareas programadas
        │
        ▼
Aplicar permisos finales
        │
        ▼
Validar funcionamiento
        │
        ▼
Tenant ACTIVE
```

Un tenant no debe considerarse operativo únicamente porque cargue el dashboard. La validación final debe comprobar, al menos, conectividad HTTP, base de datos inicializada, configuración válida, escritura en `moodledata`, cron operativo y permisos seguros.

## 🧪 Laboratorio local

- Ubuntu **22.04.5 LTS**
- Apache **2.4.52**
- PHP **8.1.2**
- MariaDB **10.6.23**
- Moodle **4.5.13+** para la primera exploración
- `max_input_vars = 5000`

## 📚 Documentación

```text
docs/
├── 00-entorno.md
├── 01-instalacion-moodle.md
├── 02-comparacion-native-docker.md
├── 03-multitenant.md
├── 04-proceso-aprovisionamiento.md
└── 05-automatizacion.md
```

Cada documento debe dejar claro **qué hicimos, por qué fue necesario, cómo lo verificamos, qué problemas aparecieron y qué implica para la futura automatización**.

---

> La meta no es automatizar rápidamente un procedimiento manual, sino entender el problema, validar alternativas y construir una solución que Edocere pueda operar y escalar con confianza.
