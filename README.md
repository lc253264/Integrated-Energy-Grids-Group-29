
# Integrated-Energy-Grids-Group-29

Group 29 repository for assignment 1 and 2

## Project Overview

This repository contains the coursework for DTU Course 46770: Integrated Energy Grids, focusing on optimal energy system design and analysis for Belgium and an interconnected European network (Belgium, France, Netherlands, Germany).

### Course Project Part 1 (Assignment 1)
**Deadline:** March 25, 2026

**Tasks:**
- **a)** Optimal capacity calculation for renewable and non-renewable generators in Belgium
- **b)** Sensitivity analysis of interannual variability in solar and wind generation
- **c)** Storage technology integration and impact analysis
- **d)** Multi-country interconnected network with HVAC transmission lines (400kV)
- **e)** PTDF matrix calculation and manual power flow verification

### Course Project Part 2 (Assignment 2)
**Deadline:** May 1, 2026

**Tasks:**
- **f)** CO₂ constraint sensitivity analysis on generation mix
- **g)** Gas pipeline network modeling (CH₄ transport)
- **h)** CO₂ pricing and decarbonization target analysis
- **i)** Sector coupling (electricity + heating/transport)
- **j)** Regional policy impact experiment




## Data Sources

### 1. Electricity Load Data
- **Source:** ENTSOE (European Network of Transmission System Operators)
- **File:** `data/ods001.csv`
- **Coverage:** Belgium hourly load data (2014-2025)
- **Processing:** 
  - Filtered to hourly resolution (PT15M → PT1H)
  - Recent 3 years extracted (2023-2025)
  - Missing values handled with forward/backward fill

### 2. Renewable Capacity Factors
- **Source:** ERA5 Reanalysis Historical Weather Data (1979-2017)
- **Files:** 
  - `onshore_wind_1979-2017.csv` - Onshore wind CF
  - `offshore_wind_1979-2017.csv` - Offshore wind CF
  - `pv_optimal.csv` - Solar PV CF
- **Coverage:** Belgium and neighboring countries
- **Methodology:** Mapped to 3 most recent historical years (inter-annual variability analysis)

### 3. Technology Costs & Parameters
- **Source:** PyPSA Technology Data v0.11.0 (2025 assumptions)
- **Repository:** https://github.com/PyPSA/technology-data
- **File Generated:** `data/df_techno.csv`
- **Included Parameters:**
  - Capital costs (€/MW)
  - Variable O&M costs (€/MWh)
  - Thermal efficiency
  - Lifetime (years)
  - CO₂ intensity (tCO₂/MWh)

### 4. Economic Assumptions
- **Discount Rate:** 3% (NPV analysis)
- **CO₂ Price (ETS):** €62/tCO₂ (2025 baseline)
- **Fuel Prices:** Natural gas €25.92/MWh (from PyPSA data)
- **Reference:** EnergyVille PATHS2050 (https://perspective2050.energyville.be/)

## Model Structure

### Part 1: Single-Country Model (Belgium)

**File:** `Belgium_model_task_abcf.ipynb`

**Model Type:** Linear optimization (LP)

**Decision Variables:**
- `capacity[tech]` - Installed capacity per technology (MW)
- `generation[tech, time]` - Hourly generation per technology (MWh)

**Objective Function:**

Minimize: ∑ᵢ(Capacityᵢ × AnnualizedCAPEXᵢ) + ∑ᵢ,ₜ(Generationᵢ,ₜ × MarginalCostᵢ)
Code


**Constraints:**
1. **Energy Balance:** ∑ᵢ Generationᵢ,ₜ = Demandₜ (for all timesteps t)
2. **Capacity Limit:** Generationᵢ,ₜ ≤ Capacityᵢ × CFᵢ,ₜ (for renewable sources)
3. **Non-negativity:** All variables ≥ 0

**Key Calculations:**

Annualized CAPEX (Capital Recovery Factor):

Annualized CAPEX = CAPEX × [r(1+r)ⁿ / ((1+r)ⁿ - 1)]

Where: r = discount rate (0.03) n = technical lifetime (25-30 years)
Code


Marginal Cost (Thermal Plants):

MC = VOM + (Fuel Price + CO₂ Intensity × CO₂ Price) / Efficiency

Example - CCGT: MC = 0 + (25.92 + 0.202 × 62) / 0.57 = €69.62/MWh
Code


**Technologies Included:**
| Tech | Annualized CAPEX | Marginal Cost | Efficiency |
|------|------------------|---------------|-----------|
| CCGT | €82,172/MW/yr | €69.62/MWh | 57% |
| OCGT | €35,386/MW/yr | €96.35/MWh | 40.5% |
| Solar | €41,985/MW/yr | €0.01/MWh | 100% |
| Onwind | €74,138/MW/yr | €1.51/MWh | 100% |
| Offwind | €132,260/MW/yr | €0.02/MWh | 100% |
| Nuclear | €480,948/MW/yr | €14.01/MWh | 32.6% |
| Coal | €215,708/MW/yr | €88.63/MWh | 35.6% |

**Solver:** HiGHS (open-source LP solver)

**Optimization Libraries:** Linopy + Xarray

---

### Part 2: Multi-Country Interconnected Network

**File:** `interconnected_model_task_deghij.ipynb`

**Model Framework:** PyPSA (Python Power System Analysis)

**Countries:** Belgium, France, Netherlands, Germany

**Model Type:** 
- AC network optimization
- DC power flow approximation (linearized)
- Multi-carrier sector coupling (electricity + gas)

**Network Topology:**

Buses (AC, 400kV):

Belgium ←→ France ←→ Germany ↓ ↓ ↑ Netherlands ←→ --------+
Code


**Transmission Lines (HVAC):**
| Link | Capacity | Reactance | Type |
|------|----------|-----------|------|
| Belgium-France | 3,550 MW | 0.1 | Fixed |
| Belgium-Netherlands | 3,400 MW | 0.1 | Fixed |
| Belgium-Germany | 1,000 MW | 0.1 | Fixed |
| France-Germany | 3,000 MW | 0.1 | Fixed |
| Netherlands-Germany | 3,950 MW | 0.1 | Fixed |

**Power Flow Equations (DC Approximation):**

Pₗ = (θ₀ - θ₁) / xₗ

Where: Pₗ = power flow on line l θ = bus voltage angle (radians) x = line reactance
Code


**Generators by Country:**

Belgium:
- CCGT, OCGT, Solar, Onshore Wind, Offshore Wind, Nuclear, Coal

France:
- Nuclear, Onshore Wind

Germany:
- Onshore Wind, Gas (CCGT), Coal, Solar

Netherlands:
- Gas (CCGT), Wind, Solar, Coal

**Objective:** Minimize total annualized system cost (capital + operational)

---

### Part 2b: Multi-Carrier Network (Task g - Gas Pipelines)

**Extension:** Gas transport network (CH₄)

**Gas Network Components:**

Buses:
- Belgium_gas, France_gas, Netherlands_gas, Germany_gas

**Gas Pipelines (Bidirectional Links):**
| Link | Distance | CAPEX | Loss Rate |
|------|----------|-------|-----------|
| Belgium-Netherlands | 100 km | €10/MW/km | 0.015/1000/km |
| Belgium-France | 300 km | €10/MW/km | 0.015/1000/km |
| France-Germany | 500 km | €10/MW/km | 0.015/1000/km |
| Netherlands-Germany | 300 km | €10/MW/km | 0.015/1000/km |
| Belgium-Germany | 300 km | €10/MW/km | 0.015/1000/km |

**Gas Prices by Country:**
- Netherlands: €35/MWh
- Belgium: €37/MWh
- Germany: €40/MWh
- France: €40/MWh

**Conversion Units (CCGT):**
- Gas → Electricity conversion
- Efficiency: 57%
- Installed in: Belgium, Germany, Netherlands

**Key Results:**
- Total electricity flow: 95.47 TWh/year
- Total gas flow: 419.17 TWh/year
- Gas network critically important for seasonal storage and flexibility

---

### Part 2c: CO₂ Constraint Analysis (Task h)

**Global Constraint:**

∑ᵢ,ₜ(Generationᵢ,ₜ × CO₂ Intensityᵢ) ≤ CO₂ Limit

Example: 101 MtCO₂ allowance for entire system
Code


**CO₂ Intensities:**
| Source | CO₂ Intensity |
|--------|--------------|
| Coal | 1.000 tCO₂/MWh |
| Gas (CCGT/OCGT) | 0.430 tCO₂/MWh |
| Wind | 0.012 tCO₂/MWh |
| Solar | 0.037 tCO₂/MWh |
| Nuclear | 0.005 tCO₂/MWh |

**Marginal Abatement Cost (MAC):**
- Calculated from shadow prices (dual values) of CO₂ constraint
- Indicates required CO₂ price to achieve decarbonization targets
- Compared with existing ETS price (62€/tCO₂)


## Tools & Software Stack

| Tool | Purpose | Version |
|------|---------|---------|
| Jupyter Notebook | Interactive development & visualization | - |
| Python | Programming language | 3.x |
| Pandas | Data manipulation & processing | ≥1.3.0 |
| NumPy | Numerical computing | ≥1.21.0 |
| Xarray | N-dimensional arrays | ≥0.19.0 |
| Linopy | Linear optimization | ≥0.2.0 |
| PyPSA | Power system analysis | ≥0.20.0 |
| HiGHS | LP/MIP solver | latest |
| Matplotlib | Visualization | ≥3.4.0 |
| Seaborn | Statistical visualization | ≥0.11.0 |

---

## Workflow & Data Processing

    Raw Data Import └─ Load: ENTSOE CSV → ods001.csv └─ Wind/Solar: ERA5 NetCDF → CSV files

    Data Cleaning & Filtering └─ Hourly resolution (15-min → 1-hour) └─ Extract recent 3 years (2023-2025) └─ Handle missing values (forward/backward fill)

    Technology Data Preparation └─ Download PyPSA costs_2025.csv └─ Calculate annualized CAPEX └─ Calculate marginal costs with CO₂ pricing └─ Save as df_techno.csv

    Optimization Setup (Linopy/PyPSA) └─ Define decision variables (capacity, generation) └─ Add constraints (balance, capacity, CO₂) └─ Define objective function └─ Solve with HiGHS

    Results Analysis & Visualization └─ Extract capacities & dispatch └─ Calculate annual generation by technology └─ Plot time series & capacity mixes └─ Sensitivity comparisons

Code


---

## Key Outputs & Results

### Single-Country Model (Belgium)
- Optimal capacity mix by technology
- Annual generation dispatch profiles
- Cost breakdown (capital vs operational)
- Renewable penetration rate
- Storage contribution to flexibility

### Multi-Country Model
- Cross-border power flows (TWh/year)
- National generation mixes
- Transmission line utilization
- Gas pipeline flows vs electricity flows
- System-wide decarbonization costs

### Sensitivity Analysis
- Year-to-year capacity variations
- Wind & solar inter-annual variability
- Cost sensitivity to weather conditions

### CO₂ Pricing
- Required CO₂ price for target emissions
- Comparison with EU ETS carbon price
- Decarbonization pathway analysis

---

## References & Data Sources

1. **PyPSA Technology Data:**
   https://github.com/PyPSA/technology-data
   - Latest costs for generation technologies
   - Lifetimes and efficiency assumptions

2. **ENTSOE Transparency Platform:**
   https://transparency.entsoe.eu/
   - Actual/forecast electricity load
   - Generation data by technology
   - Cross-border flows

3. **EnergyVille PATHS2050:**
   https://perspective2050.energyville.be/
   - Belgian energy transition scenarios
   - Technology assumptions
   - Fuel price projections

4. **EU ETS Carbon Pricing:**
   https://ec.europa.eu/clima/ets/
   - Allowance prices
   - Emissions trading scheme rules

5. **ERA5 Climate Reanalysis:**
   https://www.ecmwf.int/en/forecasts/datasets/reanalysis-datasets/era5
   - Historical weather data
   - Wind & solar resource assessment

6. **NOWTRICITY**
   https://www.nowtricity.com/
   - Belgium carbon emissions
   - CO2 emissions factors 

---

## How to Use This Repository

### Running the Notebooks
```bash
# Install dependencies
pip install pandas numpy xarray linopy pypsa matplotlib seaborn

# Start Jupyter
jupyter notebook

# Open and run:
# - Belgium_model_task_abcf.ipynb (Part 1)
# - interconnected_model_task_deghij.ipynb (Part 2)

Modifying Scenarios

    Change technology costs in data/df_techno.csv
    Modify interconnector capacities in the notebook
    Adjust CO₂ limits or fuel prices
    Re-run optimization to see impact

Team & Submission

Group 29 - DTU Course 46770: Integrated Energy Grids Submission Date: May 1, 2026 Report Pages: 12 pages (including revised Part 1)
License

This project is academic coursework for DTU. Data sources are publicly available and properly attributed.
Code