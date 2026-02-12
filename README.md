# Parent-Subsidiary Bridge 
<!-- would maybe number the sub-steps within step 1, step 2 etc as 1.1 etc 
I would clarify the source dataset at the start and the time of download
related to the above, the final example I would specify the date (instead of saying “at the time we were checking”). Not sure if there are other instances like this but the basic idea is just to be specific with the dates and source data (imagining someone reading this 5 yrs from now and trying to understand the procedure exactly)  -->

The bridge provides an annual mapping of U.S. ultimate parent firms to subsidiaries, with the following identifiers: 

<!-- **Company identifiers**: -->
<!-- - US-based parent company: `gvkey`, `ultimate_parent_cusip`, `ultimate_parent_cik`, `ultimate_parent_rcid`, `ultimate_parent_name`
- Subsidiary company: `subsidiary_rcid`, `subsidiary_name` -->

- US parent company:
   - `gvkey`: Compustat firm identifier for the ultimate parent firm
    - `ultimate_parent_rcid`: Revelio firm identifier for the ultimate parent firm
    - `ultimate_parent_name`: Name of the ultimate parent firm
    - `ultimate_parent_cusip`: CUSIP  of the ultimate parent firm
    - `ultimate_parent_cik`:  CIK of the ultimate parent firm
- Subsidiary company: 
    - `subsidiary_rcid`: Revelio firm identifier for the subsidiary firm
    - `subsidiary_name`: Name of the subsidiary firm
<!-- - **Final dataset**
  - Variables:
    - `year`
    - `gvkey` (ultimate parent)
    - `ultimate_parent_rcid`
    - `ultimate_parent_name`
    - `ultimate_parent_cusip`
    - `ultimate_parent_cik`
    - `subsidiary_rcid`
    - `subsidiary_name`
  - Provides a consistent annual mapping of U.S. ultimate parent firms to subsidiaries -->

Please cite xxx if using the data.

## Data  
#### Data Versions
The version released on xxxxx is the latest data that updates until the end of 2024.

 
#### Data Source
We use the following input data to build the bridge:

1. Revelio Company Mapping data from WRDS (obtained on Dec 15, 2025)
2. Subsidiary structure files from Revelio team (2025 May version)
3. Compustat North America from WRDS (obtained on October 28, 2025)

 
 
## Why building the bridge?

The bridge serves to handle the following issues of using Revelio data, especially when merging with other datasets such as Compustat.



#### Issue 1: Inconsistent firm scope and boundary
The unit of firm is not consistent between Revelio and Compustat data.
- **Compustat**: The unit of observation in Compustat is  at the `gvkey` level for the consolidated parent, whose financial data account for the consolidated subsidiaries.

- **Revelio**: Each `rcid` corresponds to an organizational entity, which may represent either a parent firm or an individual subsidiary within a corporate group. 

<!-- This inconsistency in firm scope and boundary induces inconsistencies when conducting analysis at the parent-firm level using Compustat data. Without an explicit parent–subsidiary bridge, employment records, worker flows, and firm characteristics in Revelio may be measured at a narrower or different organizational scope than the corresponding Compustat firm. As a result, combining the two datasets without reconciliation can lead to inconsistent measurement of scope and boundary of firms.   -->

The parent–subsidiary bridge allows one to link all subsidiaries to their parent firm at each point in time, and then construct Revelio-based measures at the parent-firm level.
<!-- is to align Revelio entities with Compustat’s consolidated firm scope and ensure consistent measurement across datasets. -->


<!-- 
#### Issue 1: Missing inactive firms


The Company Mapping data by Revelio omit some firms that are no longer active or have been delisted, introducing survivorship bias and creating inconsistencies with datasets such as Compustat, which include both active and inactive publicly held companies. As a result, firms that existed as standalone entities prior to corporate events may be absent from the Revelio mapping.

For example, Continental Airlines (`rcid` = 796214), which was acquired by United Airlines in 2010, does not appear as a standalone entity in the Revelio Company Mapping data, whereas it is observed as a standalone firm in Compustat prior to the merger. Without correction, workers and firm attributes associated with Continental Airlines before the merger cannot be properly linked to the correct firm.

The parent–subsidiary bridge is therefore constructed to recover such missing firms, allowing Continental Airlines to exist as a standalone parent firm prior to the merger and to become a subsidiary of United Airlines only after the merger. This ensures consistency in firm identification over time and alignment with Compustat’s historical firm coverage. -->

  
#### Issue 2: Incomplete firm identifiers 
 
In the Company Mapping file of Revelio data, some firms are missing the `gvkey` identifier, which is the primary firm identifier used in Compustat. This issue arises in particular for firms involved in mergers and acquisitions. For example, Continental Airlines (`rcid` = 796214), which was acquired by United Airlines in 2010, does not have an available `gvkey` in the Revelio Company Mapping file.

To maximize coverage of the crosswalk between `ultimate_parent_rcid` and `gvkey`, and to improve matching rate with Compustat data, we fill missing gvkeys using alternative firm identifiers, including `cusip` (the first eight digits) and `cik` from the SEC. Details of this imputation and matching procedure are described in Step 2 of the bridge construction process.


#### Issue 3: Time-invariant ultimate parent assignment

In the Company Mapping data provided by Revelio, the default `rcid`–`ultimate_parent_rcid` linkage is time-invariant and reflects the most recent parent–subsidiary structure. However, corporate ownership structures can change over time (e.g., M&A, spin-offs). If relying on the given static, most-recent linkage, employees can be incorrectly assigned to the wrong ultimate parent firm in earlier periods.

For example, the Company Mapping data indicates that Continental Airlines (`rcid` = 796214) belongs to the ultimate parent United Airlines (`rcid` = 22214941). However, this parent–subsidiary relationship only began after their merger in 2010. Without accounting for the merger year, job positions held at Continental Airlines prior to 2010 may be incorrectly recorded in the individual job position data as being associated with United Airlines as the ultimate parent, even though the merger had not yet occurred.

The time-varying bridge between  `rcid` and `ultimate_parent_rcid` is to ensure that workers are consistently matched to the correct parent firm at each point in time.


#### Issue 4: Spurious new entities from corporate restructurings
Following mergers and acquisitions, post-merger consolidated firms may appear as new entities in Revelio data. This gives rise to two related problems:
  1. a mismatch between individual job position records and entity existence over time, whereby worker–firm links may reference a consolidated entity in periods when that entity did not yet exist; and
2. inconsistent treatment of predecessor and successor firms relative to Compustat and SEC data.

For example, in Revelio’s parent–subsidiary data, American Airlines Group (`rcid` = 21007446) is recorded as existing only from 2013 onward, reflecting its formation following the merger. However, in Revelio’s individual job position data, positions held prior to 2013 may still list American Airlines Group as the ultimate parent, despite the entity’s recorded start date. In Compustat, by contrast, American Airlines Group is treated as the successor of AMR Corp (`rcid` = 589078), changing its name on 12/9/2013 following bankruptcy while retaining the same `gvkey` (001045). Compustat therefore reflects continuity of the reporting entity rather than the creation of a new firm. As a result, the creation of a post-merger entity in Revelio mechanically generates an artificial spike in worker counts and flows in the merger year, even though the pre- and post-merger entities correspond to the same firm in Compustat and SEC filings.

To address this inconsistency, the bridge backfills the crosswalk between `rcid`/`ultimate_parent_rcid` and `gvkey`, preventing the spurious creation of new entities following corporate events and aligning the workforce data with Compustat and SEC treatments of corporate renaming and restructuring.
<!-- 
Following mergers and acquisitions, post-merger consolidated firms are sometimes treated as new entities  by Revelio. Because the mapping does not explicitly account for the timing of restructuring events, worker–firm observations from pre-merger periods may be incorrectly assigned to the post-merger consolidated entity as the ultimate parent.

This generates internal inconsistencies: job positions are linked to a consolidated parent that did not yet exist, and the consolidated entity exhibits an artificial spike in worker inflows in the merger year. Such patterns are inconsistent with firm definitions and reporting boundaries in SEC filings and Compustat.

For example, American Airlines Group, the post-merger entity formed from American Airlines and US Airways, may appear as an ultimate parent for job positions held prior to the merger.

To address this issue, the bridge treats post-merger consolidated firms as new entities, with parent–subsidiary relationships beginning only after the merger date. -->

<!-- 
Due to Merger & Acquisitions, sometimes the post-merger consolidated entity is treated as a new entity in Revelio data, and the post-merger consolidated entity only exists after the merger. This causes the inconsistency that a job position can be marked with the ultimate parent firm as the consolidated entity before the merger, but the consolidated entity does not exist before the merger, according to the parent-subsidiary structure data. As a result, the consolidated entity has a huge worker inflow in the year of the merger, which is inconsistent with the SEC and Compustat data.

For example, American Airlines Group is a post-merger consolidated entity of American Airlines and US Airways. 

The bridge is therefore built to treat American Airlines Group as a new entity, with the parent-subsidiary relationship starting only after the merger occurred. -->
 
---
## Procedures of building the bridge 


#### Step 1. Build U.S. ultimate parent firm sample from Revelio

<!-- - **Goal**
  - Identify U.S.-headquartered ultimate parent firms using Revelio data
  - Construct a clean parent–subsidiary structure with reliable time coverage
  - Produce a quarterly parent panel with core firm identifiers (`gvkey`, `cusip`, `cik`) -->
 

- **1.1 Process parent–subsidiary structure files from Revelio**
  -  Use Revelio’s subsidiary structure files as the raw source and keep relationship records if they has any overlap with the analysis window. This forms the universe of potential parent–subsidiary relationships for the sample period


- **1.2 Identify U.S. ultimate parent firms**
  - Query Revelio’s company mapping table to obtain a list of U.S.-headquartered firms
  - Restrict to firms with at least one non-missing identifier among `gvkey`, `cusip` and `cik`
  <!-- - These identifiers are needed for downstream merges with Compustat  -->
 

- **1.3 Restrict relationships to U.S. ultimate parents**
  - Keep only relationship records where `ultimate_parent_rcid` belongs to the U.S. firm list
  - Drop relationship entries whose ultimate parent is not identified as U.S.-based
  <!-- - Retain the following variables:  `rcid`, `company_name`     - `ultimate_parent_rcid`
    - `ultimate_parent_company_name`
    - `startdate`
    - `enddate` -->
  - Save the filtered parent–subsidiary relationships for later steps
  - Build a  panel indicating when each U.S. ultimate parent is active
 
- **1.4 Construct the parent × quarter panel**
  - Stack all quarter-level snapshots into a single panel:
    - one row per `(ultimate_parent_rcid, quarter-end)`, and the associated firm identifiers (`gvkey`, `cusip`, `cik`) from Revelio
 

<!--  

- **1.5 Attach firm identifiers**
  - Merge the quarterly panel with Revelio’s U.S. company mapping
  - Attach:
    - `gvkey`
    - `cusip`
    - `cik`
    - company name
  - Do not propagate identifiers beyond what is directly observed in Revelio

---

- **1.6 Final output of Step 1**
  - A quarterly panel of U.S. ultimate parent firms from **1999–2024**
  - Each observation corresponds to:
    - an ultimate parent rcid
    - a quarter-end date
    - available firm identifiers (`gvkey`, `cusip`, `cik`)
  - This panel serves as the foundation for:
    - identifier harmonization (Step 2)
    - annual parent–subsidiary bridge construction (Step 3) -->


<!-- Use Revelio Company Mapping data from WRDS to find the identifiers of firms whose whose headquarters are in the U.S. Raw ffirm identfiers from Revelio in use: `rcid`, `gvkey`, `cusip`, `cik`.  -->


<!-- **1. Build U.S. parent firm sample**  
   Use Revelio company mapping and parent–subsidiary structure data to identify *ultimate parent* firms whose headquarters are in the U.S.  
   Raw firm identifiers from Revelio in use:
   
   - `gvkey` (Compustat identifier),
   - `cusip` (Securityty identifier, using the first 8 digits),
   - `cik` (SEC identifier).   
   - `rcid` (Revelio firm identifier)
   - `company` (company name) -->
 

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

<!-- - **Step 2.2 Start from Compustat firm-year universe**
  - Load Compustat firm-year sample (`Compustat_out`)
  - Keep core variables needed for merging and verification:
    - `gvkey`, `cusip`, `cik`, `conm`, `year`
  - Create `cusip8` and standardize `cik`:
    - `cusip8 = substr(cusip, 1, 8)`
    - convert `cik` to numeric where possible

--- -->

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
  <!-- - This assumes the ultimate parent remains constant within a gvkey over time unless explicitly observed otherwise  -->
    <!-- - Interpretation:
      - if a firm has a known ultimate parent at least once under the same gvkey, treat that as the parent across all gvkey-years where missing -->
    - Propagation within `cusip`: for the remaining missings, repeat the same procedure using 8-digit `cusip`
  <!-- - **Propagation within CUSIP8**
    - For remaining missing values, repeat the same idea using `cusip8`
    - For each `cusip8`, take any non-missing `ultimate_parent_rcid`
    - Merge back and fill remaining missings
    - Interpretation:
      - if the issuer-level CUSIP maps to an ultimate parent at least once, reuse it for other years where the same issuer appears -->
  - *Note*: `cik` is NOT propagated, because `cik`-to-firm mappings can change, and cik is not guaranteed to uniquely identify the same firm over time
 
    
- **2.4 Final output of Step 2**
  - A Compustat-based firm-year panel (U.S. universe) containing:
    - `gvkey`, `cusip`, `cusip8`, `cik`, `conm`, `year`, `ultimate_parent_rcid`
  - This file is used in Step 3 to:
    - align Compustat parent coverage with Revelio parent–subsidiary relationships
    - attach parent identifiers when building the annual bridge panel



<!-- 
**2. Find the maximized coverage of `ultimate_parent_rcid` for the universe of U.S. parent firmss.**  
   - Start from Compustat data for U.S. firms and take common firm identifiers from Compustat: `gvkey`, `cusip`, `cik`, `conm`, `year`. 
   - Sequentially merge Revelio-based mappings to merge in the `ultimate_parent_rcid` associated with each identifier-year pair:
     1. Merge on (`gvkey`, `year`), first priority
     2. Merge on (`cusip8`, `year`), second priority
   - Populate ultimate_parent_rcid 
      1. Extend (backfill/forward-fill) the `ultimate_parent_rcid`:  For each `gvkey`, identify a non-missing `ultimate_parent_rcid` and propagate it to all firm-years with the same `gvkey`  if a firm has non-missing `ultimate_parent_rcid` in some years but not full years  
     1. If still missing, repeat the same procedure using 8-digit `cusip` when `gvkey` backfilling does not fix all missings.
    - Use CIK as a last-resort, year-specific match. If still missing, introduce the `ultimate_parent_rcid`  based on the pair (`cik`, `year`). (`cik` is not prioritized because `cik`-`ultimate_parent_rcid` is not a one-to-one mapping due to the reusage of cik per SEC Edgar.) Did not propagate CIK-based matches across years.
   -->
 
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



<!-- ### Tricky case and examples
 -->


<!-- 2. Continental Airlines and United Airlines  -->
 

 
<!-- 
   

### 1.3 Identify U.S. ultimate parent firms

Identify parent–subsidiary relationship entries where the **ultimate parent** is a U.S. firm (using `ultimate_parent_rcid`).

---

### 1.4 Harmonize start dates within parent–subsidiary pairs

For each `rcid`–`ultimate_parent_rcid` pair:

- Identify the **earliest start date** across all positions  
- Use this earliest date to modify the start date of each relationship entry

---

### 1.5 Output

A firm–quarter panel of:

- `ultimate_parent_rcid`  
- × quarter-end dates  

An entry exists if there is an active parent–subsidiary relationship on that quarter-end date.

---

## 2. Populate parent firm identifiers using Compustat 
_(Fill missing `gvkey` in Revelio data)_

### 2.1 Build a Compustat bridge

Use Compustat to build a bridge across:

- `gvkey`
- `cik`
- `cusip`
- year

---

### 2.2 Fill missing `gvkey`s in Revelio data

- Take `pid_qtr_panel.csv` from Step 1  
- Fill missing `gvkey`s using `cusip8` and `cik` from Compustat  

> Disagreement check:  
> Less than **0.1%** of observations show disagreement where both sources report a `gvkey` but values differ.

---

### 2.3 Restrict to active public firms

Merge with the bridge (from 2.1) by `gvkey × year` to ensure all entries appear in `compustat_out.dta` (i.e., are public in that year).

This filters out firms that are no longer public but still appear active due to missing `enddate`.

**Example:**  
Tranzonic went private after its 2022 acquisition by Peak Rock Capital but still appears active in Revelio.


<!-- 
This dataset provides a consistent, year-by-year mapping of U.S. ultimate parent firms to their subsidiaries and is designed for merging with financial, labor, and organizational datasets.

 
1. For the US-head quartered firms, find the subsidiaries that have ever associated with each firm, using the start date and end date of parent-subsidairy relationships. The annual bridge is based on year-end relationships.  

  - input: parent-subsidiary structure package
    - output:  ultiamte parent rcid  × quarter-end × all associated rcids -->


 
