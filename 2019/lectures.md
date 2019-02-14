# Konference p2d2 2019

14.2.2019


## PostgreSQL 11

Magnus Hagander


**Novinky v administraci**
- Změna velikosti WAL segmentu při inicializaci (dříve *compile-time*)
- *INCLUDE* dalších sloupečků do indexu
    - Pomalejší, ale v případě `SELECT` dalších sloupečků se nemusí koukat do
      data souborů
    - Horší než *id-only* index, lepší než *(id,sloupček)* index – kvůli
      velikosti indexu
- Automatický *prewarm*
    - Každých 5 minut ukládá stav *shared buffers*
    - Při startu PG je rovnou načítá
- Zrychlení přidání sloupečku s *ne-NULL* hodnotou
    - `ALTER TABLE ... ADD COLUMN ... DEFAULT ...`


**Novinky v SQL**
- Full-text vyhledávání
    - Nová funkce `websearch_to_tsquery`
    - To, co je očekáváno od Full-text search
    - Ruší významy klíčových slov apod. by default (`AND`, `OR`, ...)
- **Dotazy s _plovoucím oknem_**
    - Např. pro součet/operaci okolních řádek
    - Nebo dle *timestamp* (`"15 minutes PRECEDING"`) postupný průměr, ...)
    - `OVER`
- Uložené procedury
    - Umožňuje `COMMIT` v proceduře


**Novinky v zálohách a replikaci**
- Pokročilé replikační sloty
- Logická replikace `TRUNCATE`
- 
- *unlogged* a *temp* tabulky vyloučeny z *basebackup*
- *basebackup* provádí checksum validaci by default

**Novinky ve výkonu**
- Zlepšení paralelizmu
    - Paralelní `CREATE INDEX`
    - `max_parallel_maintenance_workers` (zvýšit na 8 apod.)
- *Partitioning*
    - Výchozí *partitioning*: vytváří *partition* při `INSERT`
    - Umožňuje `UPDATE`, který mění umístění řádku mezi *partition*
    - Index na *partition* dle hlavní tabulky (původní i nové)
    - `UNIQUE` a agregované funkce napříč *partition*
    - Umožňuje `INSERT ON CONFLICT` napříč *partition*
    - Zlepšení pročišťování (*prune*) *partition*
    - *Partitioning* podle *hash* hodnoty
    - *Partition*-znalý `JOIN` (nutné zapnout v konfiguraci)
- *JIT* kompilace


## Logická replikace PostgreSQL

Aleš Zelený

- Replikace mezi DC v různých kontinentech
    - Logická, asynchronní
    - Použití pro distribuci dat
- Logická replikace
    - Transakční, ctí pořadí commitů
    - Vhodné na aplikační úrovni vkládat data nejdřív do závislých tabulek
    - Nereplikují se DDL a sekvence
    - `wal_level = logical`
    - Replikuje objekty dle *full qualified name* (jméno schéma i tabulky musí
      byt shodne)
- **Monitoring**
    - Kdy, ne jestli, se to stane
    - Pohled `pg_stat_replication` – replikační zpoždění
    - Pohled`pg_replication_slots` – aktivní *pid*
- Problémy
    - Řešení konfliktů
        - Řve do logu
        - ~~pg_replication_origin_advance()~~
        - Spíše smazat konfliktní řádky na *standby*
    - Problémy s replikací tabulek bez primárních klíčů
    - Pozor na *subtransakce* a *Exception handling*


## Práce s datem a časem v PostgreSQL

Pavel Stěhule

- Základní typy
    - `date`
        - Interně `int`
        - Jen počet dnů od 1.1.2000
        - Vhodné na rychlý odkaz na dny: `current_date - 10`
        - Vhodné použít ISO zápis `YYYY-MM-DD`, ostatní jsou nejednoznačné
    - `time`
        - Nedoporučuje se používat, jen z důvodu historické kompatibility
        - Nevhodné kombinovat s časovou zónou (chybí vazba na datum)
    - `timestamp`
        - Interně `int8`, včetně µs
        - Sémanticky *date* + *time* dohromady
        - Existuje s časovou zónou nebo ne – pouze přepočítávání při uložení a
          zobrazení
        - Výpis v sekundách `SELECT current_timestamp::timestamp(0);`
- Speciální konstanty
    - `epoch`
    - `infinity`, `-infinity`
        - `SELECT 'infinity'::timestamp`
    - `now`, `today`, `tomorrow`, `yesterday`
        - `SELECT 'now'::timestamp`
    - Pozor na 
- Funkce
    - `current_timestamp()` vrací čas zahájení transakce – emuluje *freeze*
    - `clock_timestamp()` vrací aktuální čas
- Typ interval
    - Nejsložitější struktura, interně složena z měsíců, dnů, a vteřin
    - Pozor na operace s intervaly a `EXTRACT` např. dní z intervalu
        - Trochu pomáhá `justify_interval()` funkce
    - Umí i zkratkovité definice, z ANSI SQL apod... velmi krkolomné
    - Vhodné použít: `1 year 1 month 42 minutes`
- Změna časového pásma
    - „čas je pořád stejný”, mění se zobrazení času
- Operátor `AT TIME ZONE`, uff...


## Patroni

Jan Tomsa

- Motivace
    - Automatizace *failover* mezi *stanby* a *master*
- Patroni
    - *Python controller* postgresu
    - Používá *HAProxy* pro směrování klientů
    - *etcd* pro interní metadata a dynamickou konfiguraci
- Nástroje
    - Patroni poskytuje *REST API* (využívá ho *HAProxy*)
    - CLI `patronictl` pro ovládání clusteru
- Poznámky
    - HAProxy vynucuje timeout na spojení – pozor na dlouhé dotazy


## Život PG serveru bez ručních zásahů

Jakub Jedelský

- Automatizace správy serverů
    - Puppet
    - OpenStack
    - Foreman
    - Vlastní klient pro Foreman
- Monitoring
    - *Icinga* + *Grafana*
    - [grafana-dashboard-builder](https://github.com/jakubplichta/grafana-dashboard-builder)


## Professional PostgreSQL monitoring made easy

Kaarel Moppel

- Důvody k monitorování
    - Výpadek
    - Výkon
    - Predikce chování
    - Sledování vytížení a využití
- Úroveň monitoringu
    - Dostupnost služby
        - Např. cron skript, jednoduché řešení, cron script, ...
    - Systémový monitoring
        - CPU, RAM, swap, IO
    - Analýza logů
        - Pro případ potřeby analýzy problému
        - Archivace logů na centrální místo
    - ...

- Parametry logování
    - `log_destination`, varianta *csv*
    - Pohledy na statistiky – dynamické a kumulativní
- Extenze
    - `pg_stat_statements` – must have 🙂
    - `pgstattuple` – *bloat tables*
    - `pg_buffercache` – efektivita *shared buffers*
    - `pg_autoexplain` – pro ladění


- Nástroje pro monitoring
    - [Seznam na wiki](https://wiki.postgresql.org/wiki/Monitoring)
- Ad hox monitoring
    - pg_activity „interaktivní `pg_stat_statements`”
    - pghero
        - Rychlý náhled na stav databáze
        - backup, index, vacuum. ... problémy
- Kontinuální monitoring
    - Obecné
        - Icinga, Zabbix, ...
    - Specifické pro PostgreSQL
        - pghero, powa, pgwatch2

**pgwatch2**

- Používá `pg_stat_*` metriky
    - Nepotřebuje admina na všechno
- Používá Grafanu
    - Dynamické dashboardy
- Volitelně i posílá oznámení (dle )


## pglogical

Matěj Klonfar

- Extenze `pglogical3`
- Logická replikace pomocí pro PostgreSQL 9.4+
- Podobně jako nativní logická replikace v PostgreSQL 10+
- Dekóduje *WAL* a konstruuje SQL dotazy na standby
    - Má i „klienty” do jiných systémů, např. RabbitMQ, Kafka apod.
- Má jisté vlastnosti, které nativní nemá
    - Možnost řešit konflikty


## PostgeSQL v praxi

Vratislav Beneš

- PostgreSQL v korporátu
    - Automotive
    - Data z měření, kontroly kvality výroby
    - Velká data, dlouho uchovávaná
    - Různorodá (signály, stav senzorů, videa, ...)
- Služba pro sběr a analýzu dat z výroby
