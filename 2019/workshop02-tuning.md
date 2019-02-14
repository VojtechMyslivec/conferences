# Monitoring PostgreSQL

13.2.2019

Tomáš Vondra


## Shared buffers

- Cache datových stránek v RAM, spravované PostgreSQL na aplikační úrovni
- Cache filesystému je spravovaná OS
- Data se do *shared buffers* dostávají během práce s daty
    - Dotazy, *autovacuum*, ...
- Data se z *shared buffers* zapisují (a případně vyhazují)
    - Při *checkpointu*, průběžně *bg writer*
    - Při nedostatku místa
- Výchozí velikost (128 MB) je velmi konzervativní
    - Optimální velikost = vysoký *cache hit ratio* bez plýtvání paměti
    - Vhodné upravovat dle pozorování z monitoringu

### cache hit ratio

```sql
SELECT blks_hit, blks_read, blks_hit * 100.0 / (blks_hit + blks_read)
  FROM pg_stat_database
```

### Využití bufferů

```sql
CREATE EXTENSION pg_buffercache
SELECT COUNT(*) FROM pg_buffercache;
SELECT COUNT(*) FROM pg_buffercache WHERE reldatabase IS NULL;
```


## `work_mem`

- Výchozí konzervativní (4 MB), ale vejde se do cache procesoru
- Paměť pro jednu operaci v dotazu
- Při překročení se vytváří *dočasné* soubory na disku
- Optimální hodnota
    - Optimální velikost = databáze nevytváří mnoho dočasných souborů


## *checkpoint tuning*

- *Checkpoint* je okamžik ve *WAL*, kdy jsou všechna data zapsána do souborů na
  disk
- Zapisuje *dirty pages* z *shared buffer* na disk
- Nastává vypršením časového limitu (*timed*) nebo množstvím změn ve WAL
    - Většina checkpointů by měla být *timed*

```sql
SELECT checkpoints_timed, checkpoints_req
  FROM pg_stat_bgwriter;
```

### `bgwriter`

- Proces, který pravidelně prochází *shared buffers* a zapisuje do datových
  souborů
- Paralelně k checkpoint
- Zapisuje a řeší pouze nejméně zapisované stránky, neoptimalizuje zápisy dle
  souborů a pozic


## *autovacuum tuning*

- *Vacuum* proces prochází datové soubory a označuje již nedostupné řádky jako
  smazané, tedy k využití pro přepsání novými daty
- *Full vacuum* přesype celou tabulku do nových datových souborů
    - Většinou není potřeba, pouze pokud velikost dat je zlomkem velikosti celé
      tabulky
- *Autovacuum* se pouští automaticky, dle počtu změněných/smazaných řádek
- U velkých databází snížit `autovacuum_analyze_scale_factor` a `autovacuum_vacuum_scale_factor`
- Případně ladit `vacuum_cost_*`


```sql
SELECT relname, n_live_tup, n_dead_tup, 100.0*n_dead_tup/(n_live_tup+n_dead_tup+1) AS pct
  FROM pg_stat_all_tables
  WHERE schemaname = 'public'
  ORDER BY 4 DESC
  LIMIT 15;
```


# TODO

- OS nastavení pro optimalizaci
    - Kernel, FS, ...
