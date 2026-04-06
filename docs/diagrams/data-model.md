# Data Model — ER Diagrams

> **Materialization strategy:** Staging tables are materialized as `table` (physical). Raw vault, business vault, and marts are materialized as `view` to minimize storage on the free tier.

---

## 1. Staging Layer (Materialized Tables)

Raw data landed from Warcraft Logs API. These are the only physical tables besides seeds.

```mermaid
erDiagram
    stg_wcl_reports {
        varchar report_code PK
        varchar title
        varchar owner
        timestamp start_time
        timestamp end_time
        int zone_id
        varchar zone_name
        varchar record_source
        timestamp load_date
    }

    stg_wcl_fights {
        varchar report_code FK
        int fight_id
        int encounter_id
        varchar encounter_name
        int difficulty
        boolean kill
        float start_time_offset
        float end_time_offset
        float fight_duration
        varchar record_source
        timestamp load_date
    }

    stg_wcl_players {
        varchar player_name
        varchar server_slug
        varchar server_region
        int class_id
        int guild_id
        varchar guild_name
        varchar guild_server_slug
        varchar guild_server_region
        varchar record_source
        timestamp load_date
    }

    stg_wcl_rankings {
        varchar player_name
        varchar server_slug
        varchar server_region
        varchar report_code
        int encounter_id
        int difficulty
        int spec_id
        float dps
        float hps
        float parse_percentile
        int ilvl
        int deaths
        float duration
        varchar record_source
        timestamp load_date
    }

    stg_wcl_guilds {
        varchar guild_name
        varchar server_slug
        varchar server_region
        varchar faction
        varchar record_source
        timestamp load_date
    }

    stg_wcl_zones {
        int zone_id PK
        varchar zone_name
        varchar expansion
        varchar zone_type
        varchar record_source
        timestamp load_date
    }

    stg_wcl_reports ||--o{ stg_wcl_fights : "contains"
    stg_wcl_reports ||--o{ stg_wcl_rankings : "report_code"
    stg_wcl_fights ||--o{ stg_wcl_rankings : "encounter"
    stg_wcl_players ||--o{ stg_wcl_rankings : "player"
    stg_wcl_guilds ||--o{ stg_wcl_players : "guild"
    stg_wcl_zones ||--o{ stg_wcl_reports : "zone_id"
```

---

## 2. Raw Vault Layer (Views)

Data Vault 2.0 modeled as views over staging tables. Hash keys computed at query time.

### Hubs

```mermaid
erDiagram
    hub_player {
        char_32 player_hk PK "md5(player_name || server_slug || server_region)"
        varchar player_name
        varchar server_slug
        varchar server_region
        timestamp load_date
        varchar record_source
    }

    hub_guild {
        char_32 guild_hk PK "md5(guild_name || server_slug || server_region)"
        varchar guild_name
        varchar server_slug
        varchar server_region
        timestamp load_date
        varchar record_source
    }

    hub_encounter {
        char_32 encounter_hk PK "md5(encounter_id || difficulty)"
        int encounter_id
        int difficulty
        timestamp load_date
        varchar record_source
    }

    hub_zone {
        char_32 zone_hk PK "md5(zone_id)"
        int zone_id
        timestamp load_date
        varchar record_source
    }

    hub_report {
        char_32 report_hk PK "md5(report_code)"
        varchar report_code
        timestamp load_date
        varchar record_source
    }
```

### Links

```mermaid
erDiagram
    hub_player {
        char_32 player_hk PK
    }
    hub_guild {
        char_32 guild_hk PK
    }
    hub_encounter {
        char_32 encounter_hk PK
    }
    hub_report {
        char_32 report_hk PK
    }

    link_player_guild {
        char_32 player_guild_hk PK "md5(player_hk || guild_hk)"
        char_32 player_hk FK
        char_32 guild_hk FK
        timestamp load_date
        varchar record_source
    }

    link_report_encounter {
        char_32 report_encounter_hk PK "md5(report_hk || encounter_hk)"
        char_32 report_hk FK
        char_32 encounter_hk FK
        timestamp load_date
        varchar record_source
    }

    link_player_encounter {
        char_32 player_encounter_hk PK "md5(player_hk || encounter_hk || report_hk)"
        char_32 player_hk FK
        char_32 encounter_hk FK
        char_32 report_hk FK
        int spec_id "dependent child key"
        timestamp load_date
        varchar record_source
    }

    link_player_report {
        char_32 player_report_hk PK "md5(player_hk || report_hk)"
        char_32 player_hk FK
        char_32 report_hk FK
        timestamp load_date
        varchar record_source
    }

    hub_player ||--o{ link_player_guild : "player_hk"
    hub_guild ||--o{ link_player_guild : "guild_hk"
    hub_report ||--o{ link_report_encounter : "report_hk"
    hub_encounter ||--o{ link_report_encounter : "encounter_hk"
    hub_player ||--o{ link_player_encounter : "player_hk"
    hub_encounter ||--o{ link_player_encounter : "encounter_hk"
    hub_report ||--o{ link_player_encounter : "report_hk"
    hub_player ||--o{ link_player_report : "player_hk"
    hub_report ||--o{ link_player_report : "report_hk"
```

### Satellites

```mermaid
erDiagram
    hub_player {
        char_32 player_hk PK
    }
    hub_guild {
        char_32 guild_hk PK
    }
    hub_encounter {
        char_32 encounter_hk PK
    }
    hub_report {
        char_32 report_hk PK
    }
    hub_zone {
        char_32 zone_hk PK
    }
    link_player_encounter {
        char_32 player_encounter_hk PK
    }

    sat_player_details {
        char_32 player_hk FK
        char_32 hashdiff "md5(player_name || server_slug || class_id || ilvl)"
        varchar player_name
        varchar server_slug
        varchar server_region
        int class_id
        int ilvl
        timestamp load_date
        varchar record_source
    }

    sat_guild_details {
        char_32 guild_hk FK
        char_32 hashdiff "md5(guild_name || server_slug || faction)"
        varchar guild_name
        varchar server_slug
        varchar server_region
        varchar faction
        timestamp load_date
        varchar record_source
    }

    sat_encounter_details {
        char_32 encounter_hk FK
        char_32 hashdiff "md5(encounter_name || kill || fight_duration)"
        varchar encounter_name
        boolean kill
        float fight_duration
        timestamp load_date
        varchar record_source
    }

    sat_report_details {
        char_32 report_hk FK
        char_32 hashdiff "md5(title || owner || start_time || end_time || zone_id)"
        varchar title
        varchar owner
        timestamp start_time
        timestamp end_time
        int zone_id
        timestamp load_date
        varchar record_source
    }

    sat_player_encounter_performance {
        char_32 player_encounter_hk FK
        char_32 hashdiff "md5(dps || hps || parse_pct || deaths || ilvl || spec_id)"
        float dps
        float hps
        float parse_percentile
        int deaths
        int ilvl
        int spec_id
        float duration
        timestamp load_date
        varchar record_source
    }

    sat_zone_details {
        char_32 zone_hk FK
        char_32 hashdiff "md5(zone_name || expansion || zone_type)"
        varchar zone_name
        varchar expansion
        varchar zone_type
        timestamp load_date
        varchar record_source
    }

    hub_player ||--o{ sat_player_details : "player_hk"
    hub_guild ||--o{ sat_guild_details : "guild_hk"
    hub_encounter ||--o{ sat_encounter_details : "encounter_hk"
    hub_report ||--o{ sat_report_details : "report_hk"
    hub_zone ||--o{ sat_zone_details : "zone_hk"
    link_player_encounter ||--o{ sat_player_encounter_performance : "player_encounter_hk"
```

---

## 3. Business Vault Layer (Views)

Resolved current-state views that pick the latest satellite record per business key.

```mermaid
erDiagram
    bv_current_player {
        char_32 player_hk PK
        varchar player_name
        varchar server_slug
        varchar server_region
        int class_id
        int ilvl
        varchar class_name "joined from seed"
        varchar guild_name "joined from link + hub"
        timestamp effective_from
    }

    bv_current_guild {
        char_32 guild_hk PK
        varchar guild_name
        varchar server_slug
        varchar server_region
        varchar faction
        int member_count "count from link_player_guild"
        timestamp effective_from
    }

    bv_current_encounter {
        char_32 encounter_hk PK
        int encounter_id
        int difficulty
        varchar encounter_name
        varchar zone_name "joined from hub_zone + sat_zone"
        varchar expansion
        timestamp effective_from
    }

    bv_player_encounter_latest {
        char_32 player_encounter_hk PK
        char_32 player_hk FK
        char_32 encounter_hk FK
        char_32 report_hk FK
        float dps
        float hps
        float parse_percentile
        int deaths
        int ilvl
        int spec_id
        varchar spec_name "joined from seed"
        float duration
        timestamp effective_from
    }

    bv_current_player ||--o{ bv_player_encounter_latest : "player_hk"
    bv_current_encounter ||--o{ bv_player_encounter_latest : "encounter_hk"
```

---

## 4. Mart Layer (Views)

Consumption-ready views for the frontend. Queried by Next.js Server Actions with filters.

```mermaid
erDiagram
    mart_player_rankings {
        varchar player_name
        varchar server_slug
        varchar server_region
        varchar class_name
        varchar guild_name
        int total_encounters
        float avg_parse_percentile
        float best_parse_percentile
        float avg_dps
        float max_dps
        float avg_hps
        float max_hps
        int total_kills
        int total_deaths
    }

    mart_encounter_statistics {
        varchar encounter_name
        int difficulty
        varchar zone_name
        varchar expansion
        varchar spec_name
        varchar class_name
        int player_count
        float avg_dps
        float median_dps
        float p95_dps
        float avg_hps
        float median_hps
        float p95_hps
        float avg_duration
        int total_kills
        int total_wipes
    }

    mart_class_performance {
        varchar class_name
        varchar spec_name
        varchar encounter_name
        int difficulty
        float avg_parse_percentile
        float avg_dps
        float avg_hps
        int sample_size
        int rank_by_dps
        int rank_by_hps
    }

    mart_guild_progression {
        varchar guild_name
        varchar server_slug
        varchar server_region
        varchar faction
        varchar encounter_name
        int difficulty
        varchar zone_name
        timestamp first_kill_date
        int total_kills
        float fastest_kill_duration
        float avg_kill_duration
        int progression_rank
    }

    mart_recent_reports {
        varchar report_code
        varchar title
        varchar owner
        timestamp start_time
        timestamp end_time
        varchar zone_name
        int fight_count
        int player_count
        int kill_count
        int wipe_count
    }
```

---

## 5. Full Data Flow

```mermaid
flowchart LR
    subgraph API["Warcraft Logs API"]
        A1[Reports]
        A2[Fights]
        A3[Players]
        A4[Rankings]
        A5[Guilds]
        A6[Zones]
    end

    subgraph STG["Staging (Tables)"]
        S1[stg_wcl_reports]
        S2[stg_wcl_fights]
        S3[stg_wcl_players]
        S4[stg_wcl_rankings]
        S5[stg_wcl_guilds]
        S6[stg_wcl_zones]
    end

    subgraph RV["Raw Vault (Views)"]
        direction TB
        subgraph Hubs
            H1[hub_player]
            H2[hub_guild]
            H3[hub_encounter]
            H4[hub_zone]
            H5[hub_report]
        end
        subgraph Links
            L1[link_player_guild]
            L2[link_report_encounter]
            L3[link_player_encounter]
            L4[link_player_report]
        end
        subgraph Satellites
            SA1[sat_player_details]
            SA2[sat_guild_details]
            SA3[sat_encounter_details]
            SA4[sat_report_details]
            SA5[sat_player_encounter_perf]
            SA6[sat_zone_details]
        end
    end

    subgraph BV["Business Vault (Views)"]
        B1[bv_current_player]
        B2[bv_current_guild]
        B3[bv_current_encounter]
        B4[bv_player_encounter_latest]
    end

    subgraph MART["Marts (Views)"]
        M1[mart_player_rankings]
        M2[mart_encounter_statistics]
        M3[mart_class_performance]
        M4[mart_guild_progression]
        M5[mart_recent_reports]
    end

    subgraph SEEDS["Seeds"]
        SD1[class_spec_mapping]
    end

    A1 --> S1
    A2 --> S2
    A3 --> S3
    A4 --> S4
    A5 --> S5
    A6 --> S6

    S1 & S2 & S3 & S4 & S5 & S6 --> RV
    SEEDS --> BV
    RV --> BV
    BV --> MART
```

---

## 6. Seed Data

```mermaid
erDiagram
    seed_class_spec_mapping {
        int class_id PK
        varchar class_name
        int spec_id PK
        varchar spec_name
        varchar role "tank / healer / dps"
    }
```
