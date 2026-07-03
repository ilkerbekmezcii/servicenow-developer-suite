🌍 [English](README.md) | [Türkçe](README.tr.md) | [Español](README.es.md)

# 🛠️ ServiceNow Developer Suite — Fix Scripts & Actualizaciones Masivas

> **Una colección seleccionada de Fix Scripts de ServiceNow listos para producción, mejores prácticas de rendimiento y guías de migración de datos por lotes.**

<div align="center">

![Plataforma](https://img.shields.io/badge/platform-ServiceNow-brightgreen)
![Licencia](https://img.shields.io/badge/license-MIT-green)
![Compatibilidad](https://img.shields.io/badge/compatibility-Utah%20%7C%20Vancouver%20%7C%20Washington-blue)

</div>

---

## 📚 Tabla de Contenidos

- [Introducción](#introducción)
- [Mejores Prácticas para Fix Scripts](#mejores-prácticas-para-fix-scripts)
- [Ejemplo 1: Actualización Masiva Segura con Simulación (Dry-Run)](#ejemplo-1-actualización-masiva-segura-con-simulación-dry-run)
- [Ejemplo 2: Migración por Lotes Fragmentados (Chunked)](#ejemplo-2-migración-por-lotes-fragmentados-chunked)
- [Ejemplo 3: GlideMultipleUpdate de Alto Rendimiento](#ejemplo-3-glidemultipleupdate-de-alto-rendimiento)
- [Tarjeta de Referencia Rápida](#tarjeta-de-referencia-rápida)

---

## Introducción

En el desarrollo empresarial de ServiceNow, los **Fix Scripts** son herramientas críticas utilizadas para la inicialización de datos de aplicaciones, migraciones de datos masivas, limpiezas y despliegues de parches. Sin embargo, la ejecución de scripts en tablas con millones de registros puede provocar tiempos de espera de transacciones (timeouts), agotamiento de memoria o ejecución accidental de reglas de negocio (business rules).

Este repositorio describe plantillas estándar y mejores prácticas para garantizar actualizaciones de datos seguras, eficientes y reproducibles en ServiceNow.

---

## Mejores Prácticas para Fix Scripts

1. **Utilice siempre setWorkflow(false)** (cuando sea apropiado) para evitar la activación de Reglas de Negocio, Flujos de Trabajo, motores de Flow Designer y auditorías, lo que acelera significativamente la ejecución.
2. **Utilice siempre autoSysFields(false)** para preservar los campos de auditoría del sistema originales como `sys_updated_on`, `sys_updated_by` y `sys_mod_count` durante las migraciones técnicas.
3. **Implemente el modo de simulación (Dry-Run):** Incluya siempre una bandera `dryRun` para previsualizar los cambios (registrar recuentos y sys_ids de registros afectados) antes de ejecutar las actualizaciones reales.
4. **Utilice fragmentación (Chunking) para conjuntos de datos grandes:** Evite consultas que carguen millones de registros en memoria a la vez. Procéselos en lotes de 5.000 a 10.000 utilizando `setLimit()` y seguimiento de compensación.

---

## Ejemplo 1: Actualización Masiva Segura con Simulación (Dry-Run)

Utilice esta plantilla para actualizar registros de forma segura con un modo de vista previa de simulación integrado.

```javascript
var dryRun = true; // Establecer en false para aplicar los cambios
var targetTable = 'incident';
var encodedQuery = 'active=true^state=3'; // Incidentes en espera (On Hold)

var gr = new GlideRecord(targetTable);
gr.addEncodedQuery(encodedQuery);
gr.query();

var count = 0;
gs.info('Iniciando Fix Script. Dry Run: ' + dryRun);

while (gr.next()) {
    count++;
    if (!dryRun) {
        gr.setWorkflow(false);
        gr.autoSysFields(false);
        gr.work_notes = "Mantenimiento del sistema: Actualización de comentarios en espera.";
        gr.update();
    } else {
        gs.info('[DRY RUN] Actualizaría el incidente: ' + gr.number);
    }
}
gs.info('Ejecución completada. Registros afectados: ' + count);
```

---

## Ejemplo 2: Migración por Lotes Fragmentados (Chunked)

Para conjuntos de datos grandes, utilice este bucle para consultar y actualizar registros en lotes, evitando errores de memoria y tiempos de espera de transacciones.

```javascript
var BATCH_SIZE = 5000;
var targetTable = 'u_custom_data';
var encodedQuery = 'u_processed=false';

var recordsLeft = true;
var totalUpdated = 0;

while (recordsLeft) {
    var gr = new GlideRecord(targetTable);
    gr.addEncodedQuery(encodedQuery);
    gr.setLimit(BATCH_SIZE);
    gr.query();
    
    var batchCount = 0;
    if (!gr.hasNext()) {
        recordsLeft = false;
        break;
    }
    
    while (gr.next()) {
        gr.setWorkflow(false);
        gr.autoSysFields(false);
        gr.u_processed = true;
        gr.update();
        batchCount++;
    }
    
    totalUpdated += batchCount;
    gs.info('Lote actualizado de ' + batchCount + ' registros. Total: ' + totalUpdated);
    
    // Evitar bucles infinitos en caso de que los criterios de consulta no cambien
    if (batchCount < BATCH_SIZE) {
        recordsLeft = false;
    }
}
gs.info('Migración completada. Total actualizado: ' + totalUpdated);
```

---

## Ejemplo 3: GlideMultipleUpdate de Alto Rendimiento

Cuando necesite actualizar una sola columna al mismo valor en millones de registros, `GlideMultipleUpdate` es el método más rápido porque ejecuta una sentencia SQL `UPDATE` directa en la base de datos, omite la memoria de scripts y evita el bucle GlideRecord estándar.

> ⚠️ **Precaución:** Esta API no está documentada y solo debe usarse en scripts con ámbito (scoped) o globales cuando se requiera el máximo rendimiento. Omite todos los flujos de trabajo y campos del sistema automáticamente.

```javascript
var targetTable = 'incident';
var mu = new GlideMultipleUpdate(targetTable);
mu.addQuery('active', 'true');
mu.addQuery('priority', '1');
mu.setValue('work_notes', 'Actualización masiva ejecutada a través de GlideMultipleUpdate.');

gs.info('Ejecutando actualización masiva de base de datos...');
mu.execute();
gs.info('Actualización masiva completada con éxito.');
```

---

## Tarjeta de Referencia Rápida

```
┌──────────────────────────────────────────────────────────┐
│             🛠️ Referencia Rápida ServiceNow              │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  GLIDERECORD API           BEST PRACTICES                │
│  gr.query()      .......  Ejecutar consulta setWorkflow()│
│  gr.next()       .......  Siguiente reg.  autoSysFields()│
│  gr.update()     .......  Guardar cambios setLimit()     │
│                                                          │
│  MÉTODOS ACTUALIZACIÓN MAS. RENDIMIENTO                  │
│  GlideMultipleUpd.......  Actualiz. rápidaChooseColumns()│
│  GlideQuery      .......  API Query modernisInteractive()│
│                                                          │
│  EJECUCIÓN SEGURA                                        │
│  Dry Run         .......  Prueba de simulac              │
│  Chunking        .......  Evitar timeout                 │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

> **ServiceNow Developer Suite** — Licencia MIT — Creado con ❤️
