# Parent-Subsidiary Bridge 
 
The bridge provides an annual mapping of U.S. ultimate parent firms to global subsidiaries, with the following identifiers. 

 
- US parent company:
   - `gvkey`: Compustat firm identifier for the ultimate parent firm
    - `ultimate_parent_rcid`: Revelio firm identifier for the ultimate parent firm
    - `ultimate_parent_name`: Name of the ultimate parent firm
    - `ultimate_parent_cusip`: CUSIP  of the ultimate parent firm
    - `ultimate_parent_cik`: SEC identifier CIK of the ultimate parent firm
- Subsidiary company: 
    - `subsidiary_rcid`: Revelio firm identifier for the subsidiary firm
    - `subsidiary_name`: Name of the subsidiary firm
 
Please cite xxx if using the data.

## Data  
#### Data Versions
The version released on xxxxx is the latest data that updates until the end of 2024.

 
#### Data Source
We use the following data to build the bridge:

1. Revelio Company Mapping data from WRDS (obtained on Dec 15, 2025)
2. Subsidiary structure files from Revelio team (2025 May version)
3. Compustat North America from WRDS (obtained on October 28, 2025)

 
 
## Why building the bridge?

The bridge serves to handle the following issues of using Revelio data, especially when merging with other datasets such as Compustat.



#### Issue 1: Inconsistent firm unit
The default unit of firm is not consistent between Revelio and Compustat data.
- **Compustat**: The unit of observation in Compustat is the SEC registrant (reporting parent). Firms are defined at the level of the consolidated parent, and financial variables reflect the accounting boundary of the reporting entity, including its consolidated subsidiaries.  Compustat variables are consistent with the firm’s reporting scope in SEC filings.

- **Revelio**: Each `rcid` corresponds to an organizational entity, which may represent either a parent firm or an individual subsidiary within a corporate group. Entities are observed separately and are not automatically consolidated to align with the accounting boundary used in Compustat.

This mismatch in firm scope and boundary creates inconsistencies when conducting analysis at the parent-firm level using Compustat data. Without an explicit parent–subsidiary bridge, employment records, worker flows, and firm characteristics in Revelio may be measured at a narrower or different organizational scope than the corresponding Compustat firm. As a result, combining the two datasets without reconciliation can lead to inconsistent measurement of scope and boundary of firms.  The parent–subsidiary bridge is to align Revelio entities with Compustat’s consolidated firm scope and ensure consistent measurement across datasets.
 
#### Issue 2: Incomplete firm identifiers 
 
In the Company Mapping of Revelio data, some firms are missing the `gvkey` identifier, which is the primary firm identifier used in Compustat. This issue arises in particular for firms involved in mergers and acquisitions. For example, Continental Airlines (`rcid` = 796214), which was acquired by United Airlines in 2010, does not have an available `gvkey` in the Revelio Company Mapping.

To maximize coverage of the crosswalk between `ultimate_parent_rcid` and `gvkey`, and to improve matching rate with Compustat data, we supplement missing gvkeys using alternative firm identifiers, including `cusip` (using the first eight digits) and `cik` from the SEC. Details of this imputation and matching procedure are described in Step 2 of the bridge construction process.


#### Issue 3: Time-invariant ultimate parent assignment

In the Company Mapping data provided by Revelio, the default `rcid`–`ultimate_parent_rcid` relationship is time-invariant and reflects the most recent parent–subsidiary structure. However, corporate ownership structures evolve over time due to mergers, acquisitions, spin-offs, and other restructurings. As a result, when a time-invariant parent mapping is applied to historical worker–firm observations, employees may be incorrectly assigned to the wrong ultimate parent in different periods.

For example, the Company Mapping indicates that Continental Airlines (`rcid` = 796214) belongs to the ultimate parent United Airlines (`rcid` = 22214941), without specifying that this parent–subsidiary relationship only begins after their merger in 2010. Consequently, in the individual job position data, positions held at Continental Airlines prior to 2010 may be incorrectly recorded as being associated with United Airlines as the ultimate parent, even though the merger had not yet occurred.

To address this issue, we construct a time-varying bridge between  `rcid` and `ultimate_parent_rcid` to ensure that workers are consistently matched to the correct parent firm at each point in time.


#### Issue 4: Spurious new entities from corporate restructurings
Following mergers and acquisitions, post-merger consolidated firms may appear as new entities in Revelio data. This gives rise to two related problems:
  1. a mismatch between individual job position records and entity existence over time, whereby worker–firm links may reference a consolidated entity in periods when that entity did not yet exist; and
2. inconsistent treatment of predecessor and successor firms relative to Compustat and SEC data.

For example, in Revelio’s parent–subsidiary data, American Airlines Group (`rcid` = 21007446) is recorded as existing only from 2013 onward, reflecting its formation following the merger. However, in Revelio’s individual job position data, positions held prior to 2013 may still list American Airlines Group as the ultimate parent, despite the entity’s recorded start date. In Compustat, by contrast, American Airlines Group is treated as the successor of AMR Corp (`rcid` = 589078), changing its name on 12/9/2013 following bankruptcy while retaining the same `gvkey` (001045). Compustat therefore reflects continuity of the reporting entity rather than the creation of a new firm. As a result, the creation of a post-merger entity in Revelio mechanically generates an artificial spike in worker counts and flows in the merger year, even though the pre- and post-merger entities correspond to the same firm in Compustat and SEC filings.

To address this inconsistency, the bridge backfills the crosswalk between `rcid`/`ultimate_parent_rcid` and `gvkey`, preventing the spurious creation of new entities following corporate events and aligning the workforce data with Compustat and SEC treatments of corporate renaming and restructuring.
 
---
## Procedures of building the bridge 


#### Step 1. Build U.S. ultimate parent firm sample from Revelio
 

- **1.1 Process parent–subsidiary structure files from Revelio**
  -  Use Revelio’s subsidiary structure files as the raw source and keep relationship records if they has any overlap with the analysis window. This forms the universe of potential parent–subsidiary relationships for the sample period


- **1.2 Identify U.S. ultimate parent firms**
  - Query Revelio’s company mapping table to obtain a list of U.S.-headquartered firms
  - Restrict to firms with at least one non-missing identifier among `gvkey`, `cusip` and `cik`
  <!-- - These identifiers are needed for downstream merges with Compustat  -->
 

- **1.3 Restrict relationships to U.S. ultimate parents**
  - Keep only relationship records where `ultimate_parent_rcid` belongs to the U.S. firm list
  - Drop relationship entries whose ultimate parent is not identified as U.S.-based
 
  - Save the filtered parent–subsidiary relationships for later steps
  - Build a  panel indicating when each U.S. ultimate parent is active
 
- **1.4 Construct the parent × quarter panel**
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
- **2.1 Prepare mappings from Step 1 (Revelio parent Q4 snapshots)**
  - Keep only Q4 observations to proxy for annual relationship 
  - Build three annual mappings between firm identifiers:
    - (`gvkey`, `year`) → `ultimate_parent_rcid`
    - (`cusip8`, `year`) → `ultimate_parent_rcid`
    - (`cik`, `year`) → `ultimate_parent_rcid`

 

- **2.2 Merge Revelio ultimate-parent rcid using prioritized identifier matches**
  - Merge in Revelio-based `ultimate_parent_rcid` using the following priority order:

    - Priority 1: (`gvkey`, `year`) 
    - Priority 2: (`cusip`, `year`) 
      -  Use this only if the `gvkey`-based `ultimate_parent_rcid` is missing. `cusip8` captures the issuer and can recover cases where gvkey mapping is missing in Revelio for some years.
   -  Priority 3: (`cik`, `year`)
      -  CIK is not guaranteed to be one-to-one with firms over time (reassignment/reuse), so we avoid propagating it across years
 
- **2.3 Propagate `ultimate_parent_rcid` within firm identifiers to reduce missingness**
  - Some firms have a valid `ultimate_parent_rcid` in some years but not others. To maximize coverage, propagate within available identifiers
    - Propagation within `gvkey`: For each `gvkey`, identify its associated `ultimate_parent_rcid`, then use this value to backfill or forward-fill `ultimate_parent_rcid` to other years with the same `gvkey` where it is missing. 
 
    - Propagation within `cusip`: for the remaining missings, repeat the same procedure using 8-digit `cusip`
 
  - *Note*: `cik` is NOT propagated, because `cik`-to-firm mappings can change, and cik is not guaranteed to uniquely identify the same firm over time
 
    
- **2.4 Final output of Step 2**
  - A Compustat-based firm-year panel (U.S. universe) containing:
    - `gvkey`, `cusip`, `cusip8`, `cik`, `conm`, `year`, `ultimate_parent_rcid`
  - This file is used in Step 3 to:
    - align Compustat parent coverage with Revelio parent–subsidiary relationships
    - attach parent identifiers when building the annual bridge panel



 
#### Step 3. Build an annual parent–subsidiary panel (2000–2024) 

Construct an annual panel linking each U.S. ultimate parent firm to all subsidiaries active in a given year. Final unit of observation: **year × ultimate parent × subsidiary**
 
- **3.1 Clean and align relationship dates** 
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
 

- **3.2 Identify parent-subsidiary relationship using year-end snapshots**
  - Use December 31 of each year to define point-in-time parent-subsidiary relationship
  - For each year from **2000 to 2024**, for each parent entity, identify point-in-item active subsidiary if:
    - the (revised) relationship start date is on or before December 31, and
    - the relationship end date is on or after December 31 (or missing, which means ongoing)
  - Assign each active relationship to that calendar year

 

- **3.3 Construct the bridge panel**
  - Stack year-end snapshots to form a panel with one row per:
    - `year`
    - `ultimate_parent_rcid`
    - `subsidiary_rcid`
 

  - Merge in ultimate parent identifiers using the intermediate data from step 2 to finalize the bridge 
  
- **3.4 Final restrictions and output**
  - Restrict the sample to years **2000–2024**
  - Keep only observations with a valid ultimate parent `gvkey` 
---
### Contact

 
