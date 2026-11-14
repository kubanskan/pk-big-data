# Aircraft Fuel Efficiency and CO2 Estimation

**Big Data Project – Topic 7, Group 7, Team No. 7**
**Authors:** Izabela S., Natalia K.
**Platform:** Microsoft Azure + Databricks (+ backup / failover instance)

---

## 1. Goal

Build a batch data pipeline on Azure Databricks that:

1. measures **fuel efficiency** of commercial aircraft (fuel per km, per flight hour, per aircraft type),
2. **estimates CO2 emissions** from fuel burn,
3. (optional) trains a simple **ML model** to predict fuel burn from flight trajectories,
4. looks at **fleet age** if aircraft metadata allows it,
5. keeps working if the main environment fails (backup / failover instance).

### Research questions

- Which aircraft types are the most fuel-efficient (kg fuel / CO2 per km)?
- How do fuel burn and CO2 depend on flight phase, distance and aircraft type?
- Can fuel burn be predicted from trajectory data (altitude, speed, vertical rate)?
- *(Optional)* Does fleet age affect fuel efficiency?

---

## 2. Data

> Status: sources chosen, nothing loaded yet. Row counts and sizes must be verified after loading.

### Core dataset: Aircraft Fuel Burn Estimation Data Challenge (EUROCONTROL PRC 2025)

- Source: https://zenodo.org/records/19184662 (CC BY 4.0)
- **Measured** fuel burn (from ACARS fuel-on-board reports) fused with ADS-B trajectories (OpenSky).
- 15,761 flights, Apr–Oct 2025, 27 ICAO aircraft types, 193,275 fuel intervals, ~4.9 GB.
- Format: Parquet (trajectories: one Parquet file per flight, inside zip archives).

| File | Content | Main columns |
|---|---|---|
| `flightlist_*.parquet` | flight metadata | `flight_id`, `flight_date`, `aircraft_type`, `takeoff`, `landed`, `origin_icao`, `destination_icao` |
| `fuel_*.parquet` | fuel per time interval | `flight_id`, `start`, `end`, `fuel_kg` |
| `flights_*.zip` | trajectories | `timestamp`, `latitude`, `longitude`, `altitude`, `groundspeed`, `track`, `vertical_rate`, `mach`, `TAS`, `CAS`, `typecode`, `source` |
| `airports.parquet` | airports | `icao`, `latitude`, `longitude`, `elevation` |

**Train / rank / final split** (from the original data challenge):

| Part | Period | Flights |
|---|---|---|
| train | 12 Apr – 31 Aug 2025 | 11,037 |
| rank | 1 – 27 Sep 2025 | 1,888 |
| final | 1 – 31 Oct 2025 | 2,836 |

Files `*_submit.parquet` are empty submission templates and are not needed.
For ML we keep the time-based split (train on train, tune on rank, evaluate on final).

### Optional extensions

| Dataset | Use |
|---|---|
| **OpenSky flight list 2019–2022** (https://zenodo.org/records/7923702, ~8.1 GB, non-commercial licence) | Scale up: millions of flights, trends over time (incl. COVID-19), `icao24`/`registration`/`typecode` for fleet age |
| **OpenSky aircraft metadata** (https://opensky-network.org/data/aircraft) | Aircraft details; check whether build year is available |

### Limitations

- Core dataset covers one period (Apr–Oct 2025) and 27 types; A320 family is ~60% of flights.
- Fuel values are derived from differences of ACARS reports; some noise is possible.
- Core dataset has no `icao24`/registration, so aircraft age cannot be joined to it directly.

---

## 3. CO2 estimation

```
CO2 [kg] = fuel [kg] × emission factor
emission factor ≈ 3.16 kg CO2 per kg jet fuel   (commonly used value – verify and cite)
```

Efficiency metrics: kg fuel (or CO2) per km, per flight hour, and per aircraft type.
The emission factor is a configuration parameter.

---

## 4. Architecture

```
 Zenodo / BTS files
        │  download + upload
        ▼
 Azure Data Lake Storage Gen2   (raw / bronze / silver / gold)
        │
        ▼
 Azure Databricks (Spark + Delta Lake)
   Bronze – raw files loaded as Delta tables
   Silver – cleaned flights, fuel intervals, trajectories (joined by flight_id)
   Gold   – fuel and CO2 per flight / type / route, ML features
        │
        ▼
 Databricks SQL dashboard
```

### Backup / failover

- Geo-redundant storage (GRS) or copy to a second storage account in another region.
- Periodic copy of Delta tables (e.g. `DEEP CLONE`) to the standby region.
- Second Databricks workspace in another region with the same code (from this repo).
- **Failover test:** switch to the standby, check that results are identical, report recovery time (RTO) and data loss (RPO).

---

## 5. Steps

### 5.1 Ingest (bronze)
Download files, store them unchanged in `raw/`, load into bronze Delta tables (flight list, fuel intervals, airports, trajectories). Unzip the trajectory archives (one Parquet file per flight).

### 5.2 Clean (silver)
Unify types and timestamps (UTC), remove duplicates, join fuel intervals with flights and trajectories by `flight_id`, attach airport coordinates.

### 5.3 EDA (exploratory data analysis)
Done on the silver tables, results documented in `docs/`.

1. **Size and structure:** number of flights, fuel intervals and trajectory rows per phase (train / rank / final); total size in Delta.
2. **Data quality:**
   - nulls and duplicates in each table,
   - flights that have fuel labels but no trajectory (and the reverse),
   - interval lengths (should be 5–60 min according to the dataset description),
   - share of ADS-B vs ACARS rows in trajectories (`mach`, `TAS`, `CAS` are empty for ADS-B rows by design).
3. **Distributions:**
   - flights per aircraft type (the A320 family is ~60% of flights),
   - flight duration and distance,
   - `fuel_kg` per interval and interval length,
   - altitude, ground speed and vertical rate.
4. **Outliers and sanity checks:**
   - zero, negative or extreme `fuel_kg` values,
   - fuel rate (kg per hour) per aircraft type compared with plausible values,
   - speed spikes, altitude jumps and gaps in trajectories.
5. **Relationships:**
   - fuel vs interval length, per aircraft type,
   - fuel rate vs altitude, speed and flight phase (climb / cruise / descent, derived from `vertical_rate` and altitude),
   - correlation matrix of candidate features.
6. **Train / rank / final comparison:** check that distributions and aircraft types are similar across phases (no big drift).
7. **Decisions:** list of filters and fixes applied after EDA (e.g. removed flights, capped values) and a first set of candidate features for the model.

### 5.4 Aggregate (gold): fuel, CO2, efficiency
Fuel and CO2 (`fuel_kg × 3.16`) per flight, route and aircraft type; distance from trajectories (sum of segments); efficiency metrics (kg fuel and kg CO2 per km and per flight hour).

### 5.5 Fuel burn prediction (ML, optional)
Goal: predict `fuel_kg` per interval from trajectory data and aircraft type. CO2 is then computed from the predicted fuel with the same formula.

- **Features** (computed per interval from trajectories): aircraft type, interval duration and distance, mean and change of altitude, mean ground speed (and TAS / Mach where available), mean vertical rate, flight phase.
- **Models** (Spark MLlib), from simplest to more advanced:
  1. **Baseline:** average fuel rate per aircraft type × interval duration,
  2. **Linear regression**,
  3. **Random Forest / Gradient Boosted Trees** (expected best),
  4. *(optional)* **OpenAP** physical model, either as a comparison or as an extra feature (hybrid); check its documentation for required inputs.
- **Evaluation:** time-based split – train on `train`, tune on `rank`, final score on `final`. Metric: RMSE in kg, compared with the baseline and (optionally) with results from the data challenge paper.
- **Use:** apply the model to flights without measured fuel (e.g. OpenSky flight list) to estimate fuel and CO2 at larger scale.

### 5.6 Extensions (optional)
OpenSky flight list 2019–2022 for scale and trends, fleet age if aircraft metadata allows it.

### 5.7 Dashboard and report
Databricks SQL dashboard (fuel, CO2 and efficiency by aircraft type, route, month) and the final report.

---

## 6. Repository structure proposal

```
.
├── README.md
├── docs/            # report, diagrams
├── notebooks/
│   ├── 01_ingest_bronze
│   ├── 02_clean_silver
│   ├── 03_gold_fuel_co2
│   └── 04_ml_fuel_burn       # optional
├── src/             # shared helpers and constants
├── infra/           # storage, workspaces, failover
└── tests/           # data quality checks
```

---

## 7. Plan

| Stage | Task | Owner (proposal) |
|---|---|---|
| 1 | Azure setup: ADLS Gen2, Databricks, repo linked | Izabela |
| 2 | Download and ingest data (bronze) | Natalia |
| 3 | Silver layer and data quality checks | both |
| 4 | Gold layer: fuel, CO2, efficiency | Izabela |
| 5 | Failover setup and test (RTO/RPO) | Izabela |
| 6 | ML model, dashboard | Natalia |
| 7 | Report and presentation | both |

---

## 8. Open questions

- [ ] Do we include the optional datasets?
- [ ] Azure budget and regions for the standby instance.

---

## 9. References

- Sun, J., Spinielli, E., Strohmeier, M. (2026). *Aircraft Fuel Burn Estimation: The EUROCONTROL PRC 2025 Data Challenge.* Journal of Open Aviation Science, 4(3). https://doi.org/10.59490/joas.2026.8750
- Dataset: https://zenodo.org/records/19184662
- Strohmeier et al. (2021). *Crowdsourced air traffic data from the OpenSky Network 2019–2020.* ESSD 13(2). https://doi.org/10.5194/essd-13-357-2021
