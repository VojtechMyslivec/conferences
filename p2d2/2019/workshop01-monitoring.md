# Monitoring PostgreSQL

13.2.2019

Aleš Zelený, Pavel Stěhule


## Monitoring OS

- IO
- Podívat se na `sar`, `sadc?`, `vmstat`


## Monitoring PG

- Sbírat data
    - Velikost tabulek (1x za pár minut, rychlá retence – ve dnech)
    - Velikost databáze (např. 1x za den, archivovat kvůli měsíčním, ročním trendům)
- Twaeks pro monitoring klienta
    - Nastavit maximální prováděcí čas
    - Nastavit maximální počet spojení


## Off-topic

### Procesy

- `postmaster`, nově `postgresql`
    - Hlavní proces PG služby
    - Poslouchá na portu 5432
    - Vyřizuje připojení klientů a spouštění dalších procesů
- `checkpointer`
    - Vyřizuje operace při *checkpointu*
- `bg writer`
    - Zapisuje data z RAM (*shared buffers*) do datových souborů
- `wal writer`
    - Zapisuje *write ahead log*
- `autovacuum launcher`
    - Hlídá a vyřizuje *autovacuum*
- `stats collector`
    - Sbírá a ukládá statistiky
    - Sloupcové a ...
    - Některé perzistentní (především *delete* a *update* pro *autovacuum*)


### Shared buffers

- Cache datových bloků v paměti
- Součástí je i `wal buffer`
    - 1/32 velikosti *shared buffers*, maximálně velikost *wal block*


## Monitoring konfigurace

### Track IO

- Benchmark `pg_test_timing`
- Pokud je 99 % do 2 µs, zapnout`track_io_timing`
    - Jinak je to veliká režie


### Slow queries

- Parametr `log_min_duration_statement`
- Nastavit tak, aby log nebyl zahlcen
- Stovky milisekund, v závislosti na typu aplikace a požadavkům


### Funkce

- `track_functions`, pokud se používají vlastní funkce, procedury a triggery
- `track_functions = all`


### Autovacuum

- `log_autovacuum_min_duration`
- Začínat na `0`, zaloguje všechny autovacuum akce
    - *pgBadger* poté zobrazí data


## pgBadger

- Je možné generovat denně a ukazovat vývojářům
- Pozor na citlivá data

### pgBadger performance test

Nastavení pouze během výkonnostního testu:

```
log_duration = on
log_min_duration_statement = 0
log_statement = 'all'
log_temp_files = 0
log_autovacuum_min_duration = 0
```


## Statistiky

- Aktuální stav
    - `pg_stat_activity`
    - `pg_stat_replication`
    - `pg_stat_ssl`
- Kumulativní
    - Všechny ostatní
    - Dle instance, dle databáze


### `pg_stat_activity`

- Přehled aktuálně připojených session
- Informace
    - Trvání dotazu, trvání dotazu
    - Stav
        - active, idle, ... ok
        - *idle in transaction*
            - **důležité**
            - Nesmí běžet dlouho
            - Blokuje autovacuum
            - Volba `idle_in_transaction_session_timeout` může zabít takové dotay (nastavit řádově v minutách, desítkách minut)


### `pg_stat_archiver`

- Informace
    - Počet
    - Poslední archivace
    - Počet selhání



### `pg_stat_bgwriter`

- Checkpoint timeout
    - Výchozí 5 minut
    - Vhodné trošku delší
- Informace
    - Počet *timed* – žádoucí
    - Počet *requested* – dochází místo v *shared buffers*, *wal bloku* apod.
    - Počet bufferů
    - Počet *clean* bufferů – žádoucí
    - Počet *dirty* bufferů (`buffers_backend`)– nežádoucí, kvůli docházejícímu místu


### `pg_stat_database`

- Informace
    - Počet session
    - Počet commitů, počet rollbacků (kumulativní)
    - Počet čtení bloků, počet hit bloků (efektivita *shared buffers*)
    - Počet deadlocků
- Monitorovat
    - Vytížení počtu spojení
    - Počet transakcí (commit+rollback)
    - Poměr commit:rollback
    - *Cache hit ratio*
    - Počet a velikost temp souborů


### `pg_stat_all_tables`

- Sloučený pohled na `pg_stat_sys_tables` a `pg_stat_user_tables`
- Informace
    - Čtení indexu
    - Sekvenční sken tabulky
    - Počet přečtených řádků při sekvenčním čtení
    - Počet hot update (viz FILLFACTOR, pro tabulky bez UPDATE)
- Monitorovat
    - Poměr sekvenčních čtení ku indexovaných čtení (pokud je velký, může chybět index)
    - Poměr přečtených řádků ku sekvenčních čtení (pokud je v desítkách procent, je to v pořádku)


### `pg_stat_all_indexes`

- Opět sloučený `sys` a `user` indexy
- Informace
    - Počet čtení indexu
- Monitorovat
    - Využívání indexu, jestli není, je zbytečný
- Extenze `pg_buffercache` zjistí, jestli se bloky indexu drží v *shared buffers*


### `pg_stat_user_functions`

- Informace
    - Počet volání
    - Délka provádění
- Monitorovat
    - Ne/linearita času provádění k objemu dat


### Extenze `pg_stat_statements`

- Informace (Per dotaz)
    - Počet volání
    - Součet časů
    - Počet afektovaných řádek
    - Statistiky ohledně *shared buffers*, ...
- Monitoring, ladění
    - Vypsat nejčastější, nejdéle trvající, datově nejnáročnější dotazy
    - Ty řešit


### *Bloat* monitoring

- Nabobtnané tabulky (fragmentované)
- Monitoring
    - Opět monitoring trendu
    - *check_postgres* stačí
    - *pgstattuple* detailnější informace při řešení problémů



## TODO

- Zeptat se na parametry ohledně wal archive (`max_wal_size`, `min_wal_size`, `archive_timeout`, ...
- `COUNT(*)` sekvenční sken?
- powa extenze ()
- Telegraf agent (data do influxu)
- Jednorázové zálohování pomocí archivace
