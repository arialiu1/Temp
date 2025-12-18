# Parent-Subsidiary Bridge (Temporary)
 

### Overview

The bridge provides an annual mapping of U.S. ultimate parent firms to subsidiaries, with the following identifiers: 

 

- US parent company:
   - `gvkey` (ultimate parent)
    - `ultimate_parent_rcid`
    - `ultimate_parent_name`
    - `ultimate_parent_cusip`
    - `ultimate_parent_cik`
- Subsidiary company: 
    - `subsidiary_rcid`
    - `subsidiary_name`
 
 
---
### Procedures of building the bridge 


#### Step 1. Build U.S. ultimate parent firm sample from Revelio
 
- **Process parent–subsidiary structure files from Revelio**
  - Use Revelio’s subsidiary structure files as the raw source and keep relationship records if they has any overlap with the analysis window. This forms the universe of potential parent–subsidiary relationships for the sample period


- **Identify U.S. ultimate parent firms**
  - Query Revelio’s company mapping table to obtain a list of U.S.-headquartered firms
  - Restrict to firms with at least one non-missing identifier among `gvkey`, `cusip` and `cik`
 

- **Restrict relationships to U.S. ultimate parents**
  - Keep only relationship records where `ultimate_parent_rcid` belongs to the U.S. firm list
  - Drop relationship entries whose ultimate parent is not identified as U.S.-based
 
  - Save the filtered parent–subsidiary relationships for later steps
  - Build a  panel indicating when each U.S. ultimate parent is active
 
- **Construct the parent × quarter panel**
  - Stack all quarter-level snapshots into a single panel:
    - one row per `(ultimate_parent_rcid, quarter-end)`, and the associated firm identifiers (`gvkey`, `cusip`, `cik`) from Revelio
  
 

#### Step 2. Maximize coverage of `ultimate_parent_rcid` for the U.S. parent-firm universe 
Create a **parent-year panel** (Compustat universe) with the best-possible coverage of `ultimate_parent_rcid`
 
 

- **Inputs**
  - From **Step 1** (Revelio-based parent quarterly panel, Q4 only as year proxy):
    - `ultimate_parent_rcid`, `gvkey`, `cusip`, `cik`, `qtr_end`
  - From Compustat (U.S. firm-year sample):
    - `gvkey`, `cusip`, `cik`, `conm`, `year`

- **Key identifiers**
  - `gvkey` = Compustat firm identifier (most reliable for Compustat merges)
  - `cusip8` = first 8 digits of CUSIP (issuer-level identifier; more stable than 9-digit security-level CUSIP)
  - `cik` = SEC identifier (can be reused and not one-to-one over time; use cautiously)
  - `ultimate_parent_rcid` = Revelio ultimate parent firm identifier (target variable)

 
###### Steps
- **Prepare mappings from Step 1 (Revelio parent Q4 snapshots)**
  - Keep only Q4 observations to proxy for annual relationship 
  - Build three year-specific mapping tables from Revelio:
    - (`gvkey`, `year`) → `ultimate_parent_rcid`
    - (`cusip8`, `year`) → `ultimate_parent_rcid`
    - (`cik`, `year`) → `ultimate_parent_rcid`

 

- **Merge Revelio ultimate-parent RCID using prioritized identifier matches**
  - Merge in Revelio-based `ultimate_parent_rcid` using the following priority order:

    - Priority 1: (`gvkey`, `year`) 
    - Priority 2: (`cusip`, `year`) 
      -  Use this only if the `gvkey`-based `ultimate_parent_rcid` is missing. `cusip8` captures the issuer and can recover cases where gvkey mapping is missing in Revelio for some years.
   -  Priority 3: (`cik`, `year`)
      -  CIK is not guaranteed to be one-to-one with firms over time (reassignment/reuse), so we avoid propagating it across years
 
- **Propagate `ultimate_parent_rcid` within firm identifiers to reduce missingness**
  - Some firms have a valid `ultimate_parent_rcid` in some years but not others. To maximize coverage, propagate within available identifiers
    - Propagation within `gvkey`: For each `gvkey`, identify its associated `ultimate_parent_rcid`, then use this value to backfill or forward-fill `ultimate_parent_rcid` to other years with the same `gvkey` where it is missing. 
    - Propagation within `cusip`: for the remaining missings, repeat the same procedure using 8-digit `cusip`
 
  - *Note*: `cik` is NOT propagated, because `cik`-to-firm mappings can change, and cik is not guaranteed to uniquely identify the same firm over time
 
    
- **Final output of Step 2**
  - A Compustat-based firm-year panel (U.S. universe) containing:
    - `gvkey`, `cusip`, `cusip8`, `cik`, `conm`, `year`, `ultimate_parent_rcid`
  - This file is used in Step 3 to:
    - align Compustat parent coverage with Revelio parent–subsidiary relationships
    - attach parent identifiers when building the annual bridge panel

 
#### Step 3. Build an annual parent–subsidiary panel (2000–2024) 

Construct an annual panel linking each U.S. ultimate parent firm to all subsidiaries active in a given year. Final unit of observation: **year × ultimate parent × subsidiary**
 
- **Clean and align relationship dates** 
  - Start from Revelio parent–subsidiary relationship data. Parse relationship start and end dates into a consistent date format
  - From the Compustat parent–year panel, identify:
    - the earliest year each ultimate parent appears in Compustat
    - the latest year of Compustat coverage

        *Note*: This is because Revelio may record relationship start dates later than Compustat coverage even when the structure already existed. This can happen when Revelio creates a new entity due to M&A, such as American Airline Group (More illustrative details for this case in the appendix).
  - Backward-extend relationship start dates if appropriate
    - For each ultimate parent:
      - Identify the earliest relationship start date in Revelio
      - Compare it with the Compustat coverage start year
    - Reset the relationship start date to the Compustat start year if meeting two condition:
      - The relationship start date is the first observed date for that parent entity, and
      - Compustat coverage begins earlier
*Note*: This fixing is backward-looking only; relationship end dates are not modified
 

- **Identify parent-subsidiary relationship using year-end snapshots**
  - Use December 31 of each year to define point-in-time parent-subsidiary relationship
  - For each year from **2000 to 2024**, for each parent entity, identify point-in-item active subsidiary if:
    - the (revised) relationship start date is on or before December 31, and
    - the relationship end date is on or after December 31 (or missing, which means ongoing)
  - Assign each active relationship to that calendar year

 

- **Construct the bridge panel**
  - Stack year-end snapshots to form a panel with one row per:
    - `year`
    - `ultimate_parent_rcid`
    - `subsidiary_rcid`
 

  - Merge in ultimate parent identifiers using the intermediate data from step 2 to finalize the bridge 
  
- **Final restrictions and output**
  - Restrict the sample to years **2000–2024**
  - Keep only observations with a valid ultimate parent `gvkey` 
---



### Tricky case and examples



#### 1. Missing standalone identity
 The default RCID–parent RCID relationship on WRDS is time-invariant—for example, Continental Airlines is never treated as an independent entity and is always associated with United as of the time we were checking. Our goal is to treat Continental as a separate entity before the merger and as part of United afterwards. The bridge is therefore built to make the subsidiary firm exist as a standalone entity before the merger, with the parent-subsidiary relationship starting only after the merger occurred.

#### 2. New entity due to company restructuring

American Airlines Group
 
--- 


## Contact
