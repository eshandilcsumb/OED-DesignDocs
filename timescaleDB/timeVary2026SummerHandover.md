# Technical Handover Document

# TimescaleDB Migration for Meter Reading Aggregation

---

# Executive Summary

This document provides a technical handover for the TimescaleDB migration completed as part of the meter reading aggregation optimization project.

The objective of the project was to replace the existing PostgreSQL materialized-view-based reporting architecture with a TimescaleDB implementation using hypertables and continuous aggregates.

The migration was driven by the increasing cost of refreshing PostgreSQL materialized views as historical meter data continued to grow. The previous implementation recalculated large portions of historical data during every refresh, resulting in long execution times and duplicated work across hourly and daily aggregations.

The new implementation introduces:

- A TimescaleDB hypertable that stores precomputed hourly reading slices.
- Continuous aggregates for incremental hourly and daily aggregation.
- Cache tables that replace recursive views and runtime functions required for group aggregation.
- Updated application initialization and refresh workflows.
- Benchmarking and validation tools to verify correctness against the legacy implementation.

The migration successfully preserves analytical correctness while reducing aggregate refresh times by more than two orders of magnitude. Extensive benchmarking demonstrated approximately 250× faster hourly refreshes and approximately 340× faster daily refreshes compared to the previous implementation.

The implementation is functionally complete and ready for continued development.

---

# 1. Background

## Existing Architecture

Prior to this project, reporting was performed entirely using PostgreSQL materialized views.

```
readings
    │
    ▼
meter_hourly_readings_unit
    │
    ▼
meter_daily_readings_unit
    │
    ▼
group_hourly_readings_unit
    │
    ▼
group_daily_readings_unit
```

These materialized views were responsible for:

- Splitting readings across hourly boundaries
- Applying unit conversions
- Handling time-varying conversion factors
- Aggregating readings into hourly values
- Rolling hourly values into daily values
- Aggregating meters into groups

Although functionally correct, the design had several limitations.

### Expensive Refreshes

Materialized views refreshed by recomputing large portions of historical data.

As the database grew, refresh times increased proportionally.

### Duplicate Work

Hourly and daily materialized views independently repeated much of the same aggregation logic.

### Runtime Conversion Overhead

Every refresh repeatedly joined against conversion tables and recalculated overlap durations.

### Group Aggregation Limitations

Group aggregation depended on recursive views and runtime helper functions that are incompatible with TimescaleDB continuous aggregates.

---

# 2. Project Objectives

The migration was designed around four primary objectives.

## Performance

Replace expensive full refreshes with TimescaleDB's incremental aggregation model.

## Correctness

Maintain identical analytical results compared with the existing PostgreSQL implementation.

## Scalability

Support significantly larger datasets without proportional increases in refresh time.

## Maintainability

Move expensive calculations into predictable preprocessing and refresh stages rather than executing them repeatedly during aggregation.

---

# 3. Final Architecture

The implemented architecture is shown below.

```
                    readings
                       │
                    Trigger
                       │
                       ▼
            hypertable_hourly_split
                       │
        ┌──────────────┴──────────────┐
        ▼                             ▼
meter_hourly_readings_unit_cagg   meter_daily_readings_unit_cagg
        │                             │
        ▼                             ▼
group_hourly_readings_unit_cagg  group_daily_readings_unit_cagg
```

The intended aggregation hierarchy is:

1. Convert raw readings into hourly slices.
2. Aggregate hourly slices into meter hourly values.
3. Roll hourly values into daily values.
4. Aggregate meter values into group values.

This layered design minimizes repeated calculations and enables TimescaleDB to perform incremental refreshes efficiently.

---

# 4. Database Components

## 4.1 hypertable_hourly_split

File:

```
TimeScaleDB/create_prerequisites.sql
```

### Purpose

The hypertable stores hourly reading slices derived from the raw readings table.

Each row contains:

- hourly overlap duration
- weighted reading contribution
- unit metadata
- conversion metadata
- graphic unit information

Previously this information was calculated every time aggregation occurred.

The new implementation performs the calculation once during ingestion and stores the results for reuse.

### Benefits

- Eliminates repeated hourly splitting.
- Eliminates repeated conversion joins.
- Provides a stable source for continuous aggregates.
- Improves refresh performance.

---

## 4.2 Meter Hourly Continuous Aggregate

File:

```
TimeScaleDB/create_hourly_readings.sql
```

Created object:

```
meter_hourly_readings_unit_cagg
```

This replaces:

```
meter_hourly_readings_unit
```

Responsibilities include:

- hourly bucketing
- weighted aggregation
- minimum values
- maximum values
- unit conversion

Because the hourly split hypertable already contains precomputed overlap information, expensive calculations are not repeated.

---

## 4.3 Meter Daily Continuous Aggregate

File:

```
TimeScaleDB/create_daily_readings.sql
```

Created object:

```
meter_daily_readings_unit_cagg
```

The current implementation aggregates directly from:

```
hypertable_hourly_split
```

rather than from the hourly continuous aggregate.

While this maintains correctness, it does not yet realize the full benefits of hierarchical aggregation.

A future implementation should expose rollup state from the hourly aggregate (weighted sums, durations, minimums, and maximums) so that daily aggregates can be computed by summing intermediate states rather than reprocessing raw hourly slices.

Care must be taken not to average hourly averages, as this would produce incorrect weighted results.

---

## 4.4 Group Aggregation

### Challenge

The legacy implementation depended on:

- recursive views
- PL/pgSQL helper functions

Examples include:

- groups_deep_meters
- get_graphic_unit()

Continuous aggregates cannot depend on these objects.

### Solution

The project introduced cache tables that are refreshed before group aggregate refreshes.

```
groups_immediate_children
            │
            ▼
groups_deep_children
            │
            ▼
groups_deep_meters_cache
            │
            ▼
group_graphic_units_cache
            │
            ▼
group continuous aggregates
```

Implemented components include:

- cache tables
- cache refresh procedures
- hourly group continuous aggregates
- daily group continuous aggregates

This removes runtime recursion while remaining compatible with TimescaleDB.

---

# 5. Application Integration

The application initialization workflow was updated to create and maintain the new TimescaleDB objects.

Implemented in:

```
TimeScaleDB/Reading.js
```

Functions added include:

- createPrerequisites()
- createGroupDependencies()
- createHourlyReadings()
- createDailyReadings()
- createGroupHourlyReadings()
- createGroupDailyReadings()

Existing query functions were updated to reference the new continuous aggregates.

Meter queries now use:

- meter_hourly_readings_unit_cagg
- meter_daily_readings_unit_cagg

Group queries now use:

- group_hourly_readings_unit_cagg
- group_daily_readings_unit_cagg

---

# 6. Refresh Workflow

The implemented refresh order is:

1. Rebuild hypertable_hourly_split (when required).
2. Refresh hourly continuous aggregate.
3. Refresh daily continuous aggregate.
4. Refresh group dependency caches.
5. Refresh hourly group aggregate.
6. Refresh daily group aggregate.

This dependency order must be preserved because each stage depends on results produced by previous stages.

---

# 7. Testing and Validation

A comprehensive validation suite was created to compare the TimescaleDB implementation against the legacy PostgreSQL implementation.

Comparison scripts were created for:

- meter hourly
- meter daily
- group hourly
- group daily

Validation confirmed:

| Aggregate | Legacy | TimescaleDB | Mismatches |
|-----------|---------|-------------|------------|
| Hourly | 157,896 | 157,896 | 0 |
| Daily | 6,579 | 6,579 | 0 |

Comparison tolerance:

```
1 × 10^-11
```

The following values were validated:

- reading_rate
- minimum
- maximum
- timestamps
- missing rows

No analytical differences were detected.

---

# 8. Benchmark Results

## Hourly Refresh

Legacy:

```
~8.4 seconds
```

TimescaleDB:

```
~33 milliseconds
```

Approximately **250× faster**.

---

## Daily Refresh

Legacy:

```
~5.1 seconds
```

TimescaleDB:

```
~15 milliseconds
```

Approximately **340× faster**.

---

# 9. Storage Considerations

| Object | Size |
|---------|------|
| readings | 33 MB |
| hypertable_hourly_split | 1.3 GB |
| hourly continuous aggregate | 218 MB |
| daily continuous aggregate | 12 MB |

The storage increase is primarily due to the hourly split hypertable.

This is expected because each reading is decomposed into hourly slices before aggregation.

The additional storage represents an intentional trade-off that enables:

- dramatically faster refreshes
- reduced runtime computation
- improved scalability

---

# 10. Files Added and Modified

## SQL

- create_prerequisites.sql
- create_group_dependencies.sql
- create_hourly_readings.sql
- create_daily_readings.sql
- create_group_hourly_readings.sql
- create_group_daily_readings.sql
- CompareHourlyReadings.sql
- CompareDailyReadings.sql
- CompareGroupHourlyReadings.sql
- CompareGroupDailyReadings.sql

## Application

```
TimeScaleDB/Reading.js
```

Responsible for:

- object creation
- refresh workflow
- rebuild workflow
- integration with database setup

---

# 11. Known Limitations

The migration is complete but several areas remain for future optimization.

The current daily aggregate still processes the hourly split hypertable directly rather than rolling up hourly aggregate state.

Full hypertable rebuilds are still used more frequently than necessary.

Group cache tables are refreshed during every refresh cycle regardless of whether metadata has changed.

No explicit chunk interval has been configured.

Historical data compression has not yet been evaluated.

---

# 12. Future Work and Optimization Opportunities

## 12.1 Batch Reading Ingestion

The current implementation inserts readings sequentially.

Each inserted row triggers the hourly splitting logic independently.

The trigger performs joins against conversion tables and generates hourly slices using `generate_series()`.

Future work should investigate:

- multi-row INSERT statements
- COPY ingestion
- statement-level triggers using transition tables
- set-based generation of hourly slices

These approaches would significantly reduce ingestion overhead.

---

## 12.2 Avoid Full Hypertable Rebuilds

The refresh workflow currently rebuilds the entire split hypertable whenever no refresh window is supplied.

This process:

- deletes every split row
- regenerates every split row
- refreshes every continuous aggregate

Normal reading imports already maintain the hypertable through triggers.

Full rebuilds should therefore be reserved for:

- conversion changes
- meter-unit changes

Ordinary refreshes should instead use bounded refresh windows or TimescaleDB refresh policies.

---

## 12.3 Build Daily Aggregates from Hourly Aggregates

The current daily aggregate still processes the hourly split hypertable directly.

A true hierarchical implementation would expose rollup state from the hourly aggregate including:

- weighted sums
- durations
- minimum values
- maximum values

Daily aggregation could then process significantly fewer rows while maintaining correctness.

---

## 12.4 Improve Time Predicate Efficiency

Several graph functions create temporary `tsrange` values when filtering buckets.

Direct comparisons against the bucket timestamp are more likely to enable:

- chunk pruning
- index range scans

Some queries also call `time_bucket()` on values that are already bucketed.

Removing this unnecessary computation should improve query performance.

---

## 12.5 Refresh Group Dependency Caches Only When Required

Group cache tables are refreshed during every aggregate refresh.

Normal reading imports do not modify:

- group membership
- conversion compatibility
- meter metadata

Cache refreshes should therefore occur only when metadata changes.

The cache refresh procedures also recompute identical recursive queries multiple times.

Materializing these intermediate results once would reduce unnecessary work.

Additional indexes such as:

```
(meter_id, group_id)
```

should also be benchmarked.

---

## 12.6 Optimize Conversion Lookups

The ingestion trigger repeatedly searches the conversion table using overlapping time ranges.

The existing primary key is not optimized for this access pattern.

Future benchmarking should evaluate:

- B-tree indexes
- GiST indexes
- range indexes

to improve ingestion performance.

---

## 12.7 Tune Chunk Size and Historical Storage

The split hypertable currently uses TimescaleDB's default chunk interval.

Future work should determine an optimal chunk size based on:

- ingestion rate
- available memory
- workload characteristics

Historical chunks may also benefit from TimescaleDB columnstore compression once frequent rebuilds are eliminated.

---

## 12.8 Additional Benchmarking

Several optimizations remain worth investigating.

These include:

- composite continuous aggregate indexes
- materialized_only=true
- precomputed weighted values
- precomputed durations
- larger datasets
- higher-frequency readings
- additional meters
- production-scale workloads

---

# 13. Lessons Learned

## Continuous Aggregates Require Careful Dependency Planning

Objects used by continuous aggregates cannot depend upon:

- recursive views
- runtime helper functions
- dynamic calculations

Required metadata should be materialized before aggregation.

---

## Hierarchical Aggregation Improves Scalability

Building:

```
Raw
   │
Hourly
   │
Daily
```

is substantially more efficient than repeatedly aggregating directly from raw data.

---

## Correctness Must Be Verified

Performance improvements are only valuable if analytical correctness is preserved.

Every aggregate created during this project was validated against the legacy implementation before benchmarking.

---

# 14. Acknowledgements

Martin contributed the initial work integrating the group views and provided an important foundation for the final group aggregation implementation.

Dr. Huss-Lederman provided valuable guidance throughout testing, validation, benchmarking, and verification of the implementation.

Their support helped ensure both analytical correctness and significant performance improvements.

---

# 15. Current Status

The TimescaleDB migration has been:

- implemented
- benchmarked
- validated
- integrated into the application

Current status:

- Correctness: Passed
- Performance: Significantly improved
- Testing: Complete
- Integration: Complete

The project is considered feature complete and ready for future development.

Future contributors should use this document as the primary technical reference when extending, optimizing, or maintaining the TimescaleDB aggregation workflow.

# Recommended References and Further Reading

The following TimescaleDB documentation references provide additional background and guidance for future contributors working on optimization, maintenance, and further development of the aggregation architecture.

---

## Data Ingestion Optimization

TimescaleDB recommends using multi-row inserts or bulk loading methods such as COPY instead of inserting rows individually. Batch ingestion reduces transaction overhead and improves write performance, particularly for high-volume time-series workloads.

Reference:

https://www.tigerdata.com/docs/build/data-management/write-data/insert

Caption:

TimescaleDB documentation describing recommended approaches for efficient data ingestion, including multi-row INSERT operations and COPY-based loading.

---

## Continuous Aggregate Refresh Policies

Continuous aggregates should generally use bounded refresh windows or scheduled refresh policies rather than repeatedly performing open-ended full refreshes.

This is particularly relevant to this project because full rebuilds currently regenerate the entire hourly split hypertable and refresh all dependent continuous aggregates.

Reference:

https://www.tigerdata.com/docs/build/continuous-aggregates/refresh-policies

Caption:

TimescaleDB documentation explaining continuous aggregate refresh policies and recommended approaches for managing incremental materialization.

---

## Hypertable Query Performance and Chunk Pruning

Efficient time filtering is important for TimescaleDB hypertables because queries should allow TimescaleDB to exclude unnecessary chunks.

Future query optimization should ensure that time predicates directly constrain the hypertable time column whenever possible.

Reference:

https://www.tigerdata.com/docs/build/performance-optimization/secondary-indexes

Caption:

TimescaleDB documentation covering query performance optimization, indexing strategies, and efficient access patterns for hypertables.

---

## Hierarchical Continuous Aggregates

The current architecture uses layered aggregation. Future improvements should consider building daily aggregates from hourly aggregate states rather than recalculating from hourly split data.

When implementing hierarchical continuous aggregates, aggregate states must be designed carefully. For example, averages cannot simply be averaged again because this can produce incorrect results.

Reference:

https://www.tigerdata.com/docs/learn/continuous-aggregates/hierarchical-continuous-aggregates

Caption:

TimescaleDB documentation describing hierarchical continuous aggregates and techniques for building multi-level aggregation pipelines.

---

## Continuous Aggregate Limitations with Joined Tables

Group aggregation relies on cached dependency tables because continuous aggregates have limitations when tracking changes in joined tables.

Future contributors should consider these limitations when modifying group membership, metadata dependencies, or refresh workflows.

Reference:

https://www.tigerdata.com/docs/learn/continuous-aggregates

Caption:

TimescaleDB documentation explaining continuous aggregate behaviour, limitations, and considerations when using joins.

---

## Hypertable Chunk Sizing

The current implementation relies on TimescaleDB default chunk sizing.

Future optimization should evaluate explicit chunk intervals based on actual ingestion volume, memory availability, and query patterns.

Reference:

https://www.tigerdata.com/docs/learn/hypertables/understand-hypertables

Caption:

TimescaleDB documentation explaining hypertables, chunk sizing, and storage management considerations.

---

## Continuous Aggregate Indexing

Future benchmarking should evaluate additional composite indexes on continuous aggregates.

Potential candidates include:

- (meter_id, graphic_unit_id, bucket)
- (group_id, graphic_unit_id, bucket)

The optimal indexing strategy should be determined through workload benchmarking.

Reference:

https://www.tigerdata.com/docs/build/continuous-aggregates/create-index

Caption:

TimescaleDB documentation describing indexing strategies for continuous aggregates.

---

## Real-Time Aggregates

All continuous aggregates currently use real-time aggregation behaviour.

Future benchmarking should compare real-time aggregates against materialized-only queries to determine whether disabling real-time aggregation improves predictability and performance.

Reference:

https://www.tigerdata.com/docs/learn/continuous-aggregates/real-time-aggregates

Caption:

TimescaleDB documentation explaining real-time continuous aggregates and the relationship between materialized data and recent raw data.

---

## Summary

These references should be considered starting points for future optimization work. The current migration has established the TimescaleDB architecture, but additional improvements may be achieved through:

- Optimized batch ingestion.
- Improved refresh strategies.
- Better chunk and index tuning.
- Hierarchical aggregate design.
- Reduced unnecessary rebuild operations.
- Improved monitoring of continuous aggregate performance.
