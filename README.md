<div align="center">

# Edocere Provisioner

**Automatización del aprovisionamiento de instancias Moodle en un entorno multitenant**

`Moodle` · `Linux` · `MariaDB` · `Python` · `Docker`

</div>

---

## 🎯 Objetivo

Edocere ofrece **Moodle as a Service** sobre una infraestructura compartida.  
Este proyecto busca automatizar la creación de nuevas instancias Moodle de forma **reproducible, segura y aislada**, reduciendo el trabajo manual necesario para escalar el servicio.

Antes de automatizar, primero necesitamos entender qué implica realmente instalar y operar Moodle. Por eso el desarrollo parte desde instalaciones manuales, avanza hacia un entorno multitenant y termina convirtiendo ese conocimiento en un provisioner.

```text
Entender → Documentar → Experimentar → Comparar → Diseñar → Automatizar
```

## 🧭 Plan técnico

| Etapa | Objetivo |
|---|:---:|
| **0 · Laboratorio** | Preparar Linux, Apache, PHP y MariaDB |
| **1 · Primer Moodle** | Instalar una instancia desde cero y documentar el proceso |
| **2 · Multitenant** | Ejecutar varias instancias y estudiar su aislamiento |
| **3 · Aprovisionamiento** | Definir inputs, pasos, estados, validaciones y errores |
| **4 · Alternativas** | Comparar instalación nativa vs. Docker |
| **5 · Automatización** | Construir el provisioner a partir del proceso aprendido |

## ⚙️ Flujo esperado

```text
Solicitud de creación
        │
        ▼
Validación de parámetros
        │
        ▼
Creación y configuración de recursos
        │
        ├── Base de datos
        ├── Filesystem / moodledata
        ├── Moodle
        ├── Servidor web
        └── Tareas programadas
        │
        ▼
Validación final
        │
        ▼
Tenant operativo
```

## 🧪 El laboratorio local utiliza

- Ubuntu **22.04.5 LTS**
- Apache **2.4.52**
- PHP **8.1.2**
- MariaDB **10.6.23**
- `max_input_vars = 5000`


## 📚 Documentación

La exploración técnica se mantiene en `docs/`:

```text
docs/
├── 00-entorno.md
├── 01-instalacion-moodle.md
├── 02-multitenant.md
├── 03-proceso-aprovisionamiento.md
├── 04-comparacion-native-docker.md
└── 05-automatizacion.md
```

Cada documento debe dejar claro **qué hicimos, por qué fue necesario, qué resultado esperábamos y qué implica para la futura automatización**.

---

> La meta no es automatizar rápidamente un procedimiento manual, sino entender el problema y construir una solución que Edocere pueda operar y escalar con confianza.
