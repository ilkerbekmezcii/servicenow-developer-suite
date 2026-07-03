🌍 [English](README.md) | [Türkçe](README.tr.md) | [Español](README.es.md)

# 🛠️ ServiceNow Developer Suite — Fix Scripts & Mass Updates

> **A curated collection of production-ready ServiceNow Fix Scripts, performance best practices, and batch data migration guides.**

<div align="center">

![Platform](https://img.shields.io/badge/platform-ServiceNow-brightgreen)
![License](https://img.shields.io/badge/license-MIT-green)
![Compatibility](https://img.shields.io/badge/compatibility-Utah%20%7C%20Vancouver%20%7C%20Washington-blue)

</div>

---

## 📚 Table of Contents

- [Introduction](#introduction)
- [Best Practices for Fix Scripts](#best-practices-for-fix-scripts)
- [Example 1: Safe Dry-Run Mass Update](#example-1-safe-dry-run-mass-update)
- [Example 2: Chunked Batch Migration](#example-2-chunked-batch-migration)
- [Example 3: High-Performance GlideMultipleUpdate](#example-3-high-performance-glidemultipleupdate)
- [Quick Reference Card](#quick-reference-card)

---

## Introduction

In ServiceNow enterprise development, **Fix Scripts** are critical tools used for application data initialization, mass data migrations, cleanups, and patch deployments. However, executing scripts on tables with millions of records can result in transaction timeouts, memory exhaustion, or accidental business rule execution.

This repository outlines standard templates and best practices to ensure safe, performant, and reproducible data updates in ServiceNow.

---

## Best Practices for Fix Scripts

1. **Always Use setWorkflow(false)** (when appropriate) to prevent triggering Business Rules, Workflows, Flow Designer engines, and auditing, which significantly speeds up execution.
2. **Always Use autoSysFields(false)** to preserve original system audit fields like `sys_updated_on`, `sys_updated_by`, and `sys_mod_count` during technical migrations.
3. **Implement Dry-Run Mode:** Always include a `dryRun` flag to preview changes (log counts and affected record sys_ids) before executing actual updates.
4. **Use Chunking/Windowing for Large Datasets:** Avoid queries that load millions of records into memory at once. Batch them in chunks of 5,000 to 10,000 using `setLimit()` and offset tracking.

---

## Example 1: Safe Dry-Run Mass Update

Use this template to update records safely with a built-in dry-run preview mode.

```javascript
(function() {
    var dryRun = true; // Set to false to apply changes
    var targetTable = 'incident';
    var encodedQuery = 'active=true^state=3'; // On Hold incidents
    
    var gr = new GlideRecord(targetTable);
    gr.addEncodedQuery(encodedQuery);
    gr.query();
    
    var count = 0;
    gs.info('Starting Fix Script. Dry Run: ' + dryRun);
    
    while (gr.next()) {
        count++;
        if (!dryRun) {
            gr.setWorkflow(false);
            gr.autoSysFields(false);
            gr.work_notes = "System maintenance: Updated On Hold state comments.";
            gr.update();
        } else {
            gs.info('[DRY RUN] Would update incident: ' + gr.number);
        }
    }
    gs.info('Completed execution. Affected records: ' + count);
})();
```

---

## Example 2: Chunked Batch Migration

For large datasets, use this loop to query and update records in batches, avoiding memory heap errors and transaction timeouts.

```javascript
(function() {
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
        gs.info('Updated batch of ' + batchCount + ' records. Total: ' + totalUpdated);
        
        // Prevent infinite loops in case query criteria doesn't change
        if (batchCount < BATCH_SIZE) {
            recordsLeft = false;
        }
    }
    gs.info('Migration complete. Total updated: ' + totalUpdated);
})();
```

---

## Example 3: High-Performance GlideMultipleUpdate

When you need to update a single column to the same value across millions of records, `GlideMultipleUpdate` is the fastest method because it executes a direct SQL `UPDATE` statement in the database, bypasses scripting memory, and avoids standard GlideRecord looping.

> ⚠️ **Caution:** This API is undocumented and should only be used in scoped or global scripts when maximum performance is required. It bypasses all workflows and system fields automatically.

```javascript
(function() {
    var targetTable = 'incident';
    var mu = new GlideMultipleUpdate(targetTable);
    mu.addQuery('active', 'true');
    mu.addQuery('priority', '1');
    mu.setValue('work_notes', 'Critical mass update executed via GlideMultipleUpdate.');
    
    gs.info('Executing mass database update...');
    mu.execute();
    gs.info('Mass update completed successfully.');
})();
```

---

## Quick Reference Card

```
┌──────────────────────────────────────────────────────────┐
│              🛠️ ServiceNow Quick Reference               │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  GLIDERECORD API           BEST PRACTICES                │
│  gr.query()      .......  Execute query   setWorkflow()  │
│  gr.next()       .......  Next record     autoSysFields()│
│  gr.update()     .......  Save changes    setLimit()     │
│                                                          │
│  MASS UPDATE METHODS       PERFORMANCE                   │
│  GlideMultipleUpd.......  Fast mass updateChooseColumns()│
│  GlideQuery      .......  Modern query APIisInteractive()│
│                                                          │
│  SAFE EXECUTION                                          │
│  Dry Run (Preview.......  Test script                    │
│  Chunking        .......  Prevent timeout                │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

> **ServiceNow Developer Suite** — MIT Licensed — Created with ❤️
