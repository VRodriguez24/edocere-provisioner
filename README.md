# Edocere Provisioner

## Contexto

Edocere ofrece Moodle como servicio utilizando múltiples instancias Moodle dentro de una infraestructura compartida. El proyecto busca automatizar la creación de nuevas instancias Moodle de forma reproducible, segura y aislada.

Antes de implementar la automatización, se realizará una exploración técnica del proceso de instalación de Moodle para comprender qué recursos, configuraciones y validaciones son necesarias.

## Objetivo técnico

Diseñar e implementar un sistema capaz de aprovisionar una nueva instancia Moodle a partir de un conjunto de parámetros, manteniendo aislamiento entre tenants y compatibilidad con la infraestructura de Edocere.

## Plan técnico

### Etapa 0 — Preparación del laboratorio
Preparar y validar un entorno Linux con Apache, PHP y MariaDB.

### Etapa 1 — Instalación manual de un Moodle
Instalar una instancia Moodle completa y documentar cada paso.

### Etapa 2 — Instalación de múltiples instancias
Ejecutar varias instancias Moodle en el mismo servidor y estudiar aislamiento y recursos compartidos.

### Etapa 3 — Formalización del aprovisionamiento
Definir inputs, pasos, estados, validaciones, errores y dependencias.

### Etapa 4 — Comparación de alternativas
Comparar instalación nativa y Docker en términos de aislamiento, consumo, mantenibilidad y escalabilidad.

### Etapa 5 — Automatización
Construir el provisioner a partir del proceso documentado.

## Estado actual

### Etapa 0 — Completada

Entorno utilizado:

- Ubuntu 22.04.5 LTS
- Apache 2.4.52
- PHP 8.1.2
- MariaDB 10.6.23
- `max_input_vars = 5000`

Próximo paso: comenzar la instalación manual de la primera instancia Moodle.

## Documentación

La documentación técnica de cada etapa se encuentra en `docs/`.