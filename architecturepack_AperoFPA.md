# Architecture Pack — Apero FP&A
> Version: 3.0 | Ngay: 2026-03-09 | Dua tren: PD260 (Report/Drilldown), PD510 (All Filters), PD530 (KRF), PD200 (Runner Guide), PD220 (Plan Calculate)
> Stack: Google Sheets -> Google Colab (ETL) -> BigQuery -> FastAPI Backend -> Lovable Frontend

---

## Muc luc

1. [Entity Relationship Description](#1-entity-relationship-description)
2. [Process Description](#2-process-description)
   - 2.7 [Plan Calculate (PD220)](#27-process-plan-calculate-tu-pd220)
3. [UI Wireframe](#3-ui-wireframe)
4. [User Flows](#4-user-flows)
5. [Complex Logic](#5-complex-logic)
6. [Goi y hoan thien System Design](#6-goi-y-hoan-thien-system-design)
7. [Diem khong dong bo can xac nhan](#7-diem-khong-dong-bo-can-xac-nhan)

---

## 1. Entity Relationship Description

### 1.1 Tong quan Layer

```
+-------------------------------------------------------------------+
|  CONFIG LAYER                                                      |
|  ZBlock1 | AllocationALT | RepTemp | RepTempBlock | AllocToItem   |
|  RepTempBlockToBlock | AllocationListType                          |
+-----------------------------+-------------------------------------+
                              |
+-----------------------------v-------------------------------------+
|  DATA LAYER                                                        |
|  SOCell (so_cell_raw_full) | SOCell_SOCell                         |
+-----------------------------+-------------------------------------+
                              |
+-----------------------------v-------------------------------------+
|  REPORT LAYER                                                      |
|  RepPage | RepCell | RepFilterItem (temp)                          |
+-----------------------------+-------------------------------------+
                              |
+-----------------------------v-------------------------------------+
|  SECURITY & AUDIT LAYER                                            |
|  User | UserSecConfig | ActivityLog | PlanEventConfig               |
+-------------------------------------------------------------------+
```

---

### 1.2 Danh sach Entity & Schema

#### **[E1] ZBlock1** *(Report_config.ZBlock1_NativeTable)*
Dinh nghia cac "Run" cua du lieu (Plan / Forecast / Actual)

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `id` | STRING | PK -- composite key |
| `Z_BLOCK_ZBlock1_Source` | STRING | SRC: ACT, PLN |
| `Z_BLOCK_ZBlock1_Pack` | STRING | PCK: Pack code (vd: CA, CB) |
| `Z_BLOCK_ZBlock1_Scenario` | STRING | SCN: OPT, REAL, PESS; hoac "Actual", "Future" |
| `Z_BLOCK_ZBlock1_Run` | STRING | RUN: code cua lan chay (vd: CAC, R2025DEC11) |
| `Z_BLOCK_ZBlock1_FF` | STRING | FF: Frequency of Forecast (YF, QF, MF, WF, DF, HF) |
| `YNumber` | INTEGER | So thu tu duy nhat cho moi ZBlock combo -- dung trong YNumber1 |
| `name` | STRING | Ten mo ta |
| `created_at` | TIMESTAMP | |

> **Ghi chu tu PD510**: ZBlock1 gom 6 thanh phan con: SRC (Source), PCK (Pack), SCN (Scenario), RUN (Run), FF (Frequency). Cac thanh phan nay dung de tao cascading dropdown trong UI V2.
> **Ghi chu tu PD260**: ZBlock1 duoc dung cho ca Plan lan Forecast; chon theo tham so khi query.

---

#### **[E2] AllocationALT** *(Report_config.AllocationALT_NativeTable)* **[CAN TAO MOI]**
Mapping AllocationTypeCode -> YNumber (phuc vu cong thuc YNumber1)

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `ALT_code` | STRING | PK -- vd: CFA1, CFA2, CFA3, PLA1, PLA1T, PLA2, PLA2C, PLA3, PLA4, PLA5 |
| `ALT_type1` | STRING | CFA hoac PLA |
| `ALT_type2` | STRING | Sub-type |
| `YNumber` | INTEGER | CFA: 1xx, PLA: 2xx; Tax & CO hau to 1, con lai hau to 0 |
| `name` | STRING | Ten day du |
| `description` | STRING | Mo ta cach phan bo |

> **[BO SUNG tu PD260]** Bang nay can duoc tao moi. PD260 row 21 de cap:
> - CFA: 1xx (CFA1->100, CFA2->101, CFA3->102)
> - PLA: 2xx (PLA1->200, PLA1T->201, PLA2->202, PLA2C->203, PLA3->204, PLA4->205, PLA5->206)
> - Tax, CO hau to la 1; con lai hau to la 0
> **Lien ket PD200**: AllocationRun su dung ALT ranges: Actual ALT 100-299, Plan MKT-X ALT 300-399, Plan All ALT 400-499

---

#### **[E3] RepTemp** *(Report_config.RepTemp_NativeTable)*
Template bao cao -- dinh nghia cau truc tong the cua 1 loai bao cao

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `REP_TEMP_TYPE` | STRING | PK -- vd: KRF-L4.CDT0, KRF-L4.CDT1, KRF-L8-GI.CDT0 |
| `REP_TEMP_NUMBER` | INTEGER | So thu tu (REP1110, REP1120...) |
| `ZNumber` | INTEGER | Z-level cua template nay |
| `name` | STRING | Ten hien thi |
| `audience` | STRING | BOD, CXO, MM, TL |
| `drill_level` | STRING | DL1, DL2, DL3 |
| `created_at` | TIMESTAMP | |

> **Ghi chu tu PD530**: RepTemp lien ket voi KRF hierarchy. Cac RepTemp nhu KRF-L4.CDT0 su dung Level 4 cua KRF (Gross Inflow -> BAC -> OPX -> sub-categories), KRF-L8-GI.CDT0 di xuong Level 8.

---

#### **[E4] RepTempBlock** *(Report_config.RepTempBlock_NativeTable)*
Cac khoi dong trong 1 RepTemp. Moi block = 1 to hop KR x Filter.

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `id` | STRING | PK |
| `FK1` | STRING | FK -> RepTemp.REP_TEMP_TYPE |
| `YNumber2` | INTEGER | Thu tu block trong template (tang dan) |
| `NOW_Y_BLOCK_FNF_FNF` | STRING | KRF hoac KRN |
| `NOW_Y_BLOCK_KR_Item_Code_KR1` | STRING | KR level 1 (vd: GI, BAC, CP) |
| `NOW_Y_BLOCK_KR_Item_Code_KR2` | STRING | KR level 2 (vd: OPX, OMC, BHR, PRM) |
| `NOW_Y_BLOCK_KR_Item_Code_KR3` | STRING | KR level 3 (vd: CAC, IMP, INS, MTCH) |
| `NOW_Y_BLOCK_KR_Item_Code_KR4` | STRING | KR level 4 (vd: MAC, eCPM, CPI, DC) |
| `NOW_Y_BLOCK_KR_Item_Code_KR5` | STRING | KR level 5 (vd: OMC, P1K, NP, PPC) |
| `NOW_Y_BLOCK_KR_Item_Code_KR6` | STRING | KR level 6 |
| `NOW_Y_BLOCK_KR_Item_Code_KR7` | STRING | KR level 7 |
| `NOW_Y_BLOCK_KR_Item_Code_KR8` | STRING | KR level 8 |
| `NOW_Y_BLOCK_CDT_CDT1` | STRING | Budget Owner L1 filter |
| `NOW_Y_BLOCK_CDT_CDT2` | STRING | Budget Owner L2 filter |
| `NOW_Y_BLOCK_CDT_CDT3` | STRING | Budget Owner L3 filter |
| `NOW_Y_BLOCK_CDT_CDT4` | STRING | Budget Owner L4 filter |
| `NOW_Y_BLOCK_PTNow_PT1` | STRING | Product Type Now L1 |
| `NOW_Y_BLOCK_PTNow_PT2` | STRING | Product Type Now L2 |
| `NOW_Y_BLOCK_PTNow_Duration` | STRING | Duration |
| `NOW_Y_BLOCK_PTPrev_PT1` | STRING | Product Type Prev L1 |
| `NOW_Y_BLOCK_PTPrev_PT2` | STRING | Product Type Prev L2 |
| `NOW_Y_BLOCK_PTPrev_Duration` | STRING | Duration Prev |
| `NOW_Y_BLOCK_PTFix_OwnType` | STRING | In-house / Partnership |
| `NOW_Y_BLOCK_PTFix_AIType` | STRING | AI / Non-AI |
| `NOW_Y_BLOCK_PTSub_CTY1` | STRING | Country Tier |
| `NOW_Y_BLOCK_PTSub_CTY2` | STRING | Country code |
| `NOW_Y_BLOCK_PTSub_OSType` | STRING | iOS / Android |
| `NOW_Y_BLOCK_Funnel_FU1` | STRING | Funnel stage L1 (PS, PI, PA) |
| `NOW_Y_BLOCK_Funnel_FU2` | STRING | Funnel stage L2 (R, N) |
| `NOW_Y_BLOCK_Channel_CH` | STRING | Channel |
| `NOW_Y_BLOCK_Employee_EGT1` | STRING | Employee Group L1 |
| `NOW_Y_BLOCK_Employee_EGT2` | STRING | Employee Group L2 |
| `NOW_Y_BLOCK_Employee_EGT3` | STRING | Employee Group L3 |
| `NOW_Y_BLOCK_Employee_EGT4` | STRING | Employee Group L4 |
| `NOW_Y_BLOCK_HR_HR1` | STRING | HR type |
| `NOW_Y_BLOCK_HR_HR2` | STRING | HR sub-type |
| `NOW_Y_BLOCK_HR_HR3` | STRING | HR detail |
| `NOW_Y_BLOCK_SEC` | STRING | Security level |
| `NOW_Y_BLOCK_Period_MX` | STRING | Month Relative filter (vd: DX-15) |
| `NOW_Y_BLOCK_Period_DX` | STRING | Day filter |
| `NOW_Y_BLOCK_Period_PPC` | STRING | PrevPeriodCohort (vd: DC-365, MC-12) |
| `NOW_Y_BLOCK_Period_NP` | STRING | NowPeriod |
| `NOW_Y_BLOCK_LE_LE1` | STRING | Legal Entity L1 |
| `NOW_Y_BLOCK_LE_LE2` | STRING | Legal Entity L2 |
| `NOW_Y_BLOCK_UNIT` | STRING | Unit |
| `NOW_Y_BLOCK_TD_BU` | STRING | TopDown / BottomUp (TD, BU) |
| `KRTypeFull` | STRING | Composite KR code (parsed tu KR1-KR8) |
| `FilterTypeFull` | STRING | Composite filter code, vd: "CDT-S1-50_PT-S2-40_LE-20" |

> **Ghi chu tu PD260 row 70**: FilterTypeFull duoc tach thanh cac MyFilterType le, vd: CDT-PT-LE -> MaxNumber = 3. MyFilterType lay tu cac cot NOW_Y_BLOCK_*.
> **Ghi chu tu PD260 row 80**: KRTypeFull lay tu cac cot NOW_Y_BLOCK_FNF_FNF va KR1-KR8.

---

#### **[E5] RepTempBlockToBlock** *(Report_config.RepTempBlockToBlock_NativeTable)*
Mapping Drill-Down giua cac Block: 1 NowRepTempBlock (O) = N PrevRepTempBlock (I)

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `id` | STRING | PK |
| `now_rep_temp_block_id` | STRING | FK -> RepTempBlock.id (Block O / To) |
| `prev_rep_temp_block_id` | STRING | FK -> RepTempBlock.id (Block I / From) |
| `kr_type_code` | STRING | KR code da khu NoOfRows |
| `filter_type_code` | STRING | Filter code da khu NoOfRows |
| `drill_direction` | STRING | DrillX / DrillY / DrillZ |

> **Ghi chu tu PD260 Drill-Down**: Khu NoOfRows tu KRTypeCode va FilterTypeCode:
> - Cat cac FilterType bang "_"
> - Cat ra cac Segment bang "-"
> - Neu moi Segment toan SO thi bo di (NoOfRows)
> - So o cuoi Segment thu 2 chinh la Level
> - Chi giu lai Segment dau tien: MyFilterType (Main)

---

#### **[E6] AllocationListType** *(allocation_config.AllocationListType_NativeTable)* **[BO SUNG]**
Master list cac Filter Type -- theo PD510

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `id` | STRING | PK |
| `CAT1` | STRING | Category 1: ZB / YB / XB |
| `CAT2` | STRING | Category 2: ZB1, ZB2, LIST, PRD |
| `CAT3` | STRING | Category 3: SRC, PCK, SCN, KR, FLT |
| `CAT4` | STRING | Category 4: CDT, PTNow, PTPrev, PTFix, PTSub, Funnel, Channel, Employee, HR, Period, LE, UNIT, TDBU |
| `TypeMainCode` | STRING | Main code: CDT, PT, FU, CH, EGT, HR, LE, UNIT, TDBU |
| `TypeSubCode` | STRING | Sub code: CDT-S, CDT-CS, PT-S, PT-CS |
| `TypeLevel` | INTEGER | Level so |
| `TypeLevelCode` | STRING | vd: CDT-S1, CDT-CS2, PT-CS1 |
| `name` | STRING | Ten hien thi |

> **Ghi chu tu PD510**: Day la bang goc de dinh nghia he thong phan loai filters. Moi font Blue trong PD510 la TypeLevelCode cua 1 dong.

---

#### **[E7] AllocationToItem** *(allocation_config.AllocationToItem_NativeTable)*
Danh sach item cho tung filter type -- dung de build filter item list

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `id` | STRING | PK |
| `TO_Y_BLOCK_ToType` | STRING | Filter type, vd: "CDT-S1", "PT-S2", "LE" |
| `TO_Y_BLOCK_ToItem` | STRING | Filter item code |
| `YNumber` | INTEGER | Thu tu cua item trong type |
| `ParentType` | STRING | Parent type code (dung cho IsSub check) |
| `ParentItem` | STRING | Parent item code |
| `name` | STRING | Ten hien thi |
| `level` | INTEGER | Level trong hierarchy |

> **Ghi chu tu PD260 row 100**: Query: SELECT TO_Y_BLOCK_ToItem FROM AllocationToItem WHERE TO_Y_BLOCK_ToType = "CDT-S1"

---

#### **[E8] SOCell** *(alloc_stage.so_cell_raw_full)*
**Bang data chinh** -- luu gia tri so lieu tai moi to hop (ZBlock x YBlock x XPeriod)

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `id` | STRING | PK |
| `z_block_zblock1_source` | STRING | Source (ACT/PLN) |
| `z_block_zblock1_pack` | STRING | Pack code |
| `z_block_zblock1_scenario` | STRING | Scenario |
| `z_block_zblock1_run` | STRING | Run code |
| `now_zblock2_alt` | STRING | Allocation Type (CFA1, PLA4, ...) |
| `now_y_block_fnf_fnf` | STRING | KRF / KRN |
| `now_y_block_kr_item_code_kr1..kr8` | STRING | KR Item codes (8 levels) |
| `now_y_block_kr_item_name` | STRING | KR Item name |
| `now_y_block_cdt_cdt1..cdt4` | STRING | Budget Owner (4 levels) |
| `now_y_block_ptnow_pt1` | STRING | Product Type Now L1 |
| `now_y_block_ptnow_pt2` | STRING | Product Type Now L2 |
| `now_y_block_ptnow_duration` | STRING | Duration |
| `now_y_block_ptprev_pt1..pt2` | STRING | Product Type Prev |
| `now_y_block_ptprev_duration` | STRING | Duration Prev |
| `now_y_block_ptfix_owntype` | STRING | Ownership Type (In-house/Partnership) |
| `now_y_block_ptfix_aitype` | STRING | AI Type (AI/Non-AI) |
| `now_y_block_ptsub_cty1..cty2` | STRING | Country (Tier, Code) |
| `now_y_block_ptsub_ostype` | STRING | OS Type (iOS/Android) |
| `now_y_block_funnel_fu1..fu2` | STRING | Funnel (PS/PI/PA, R/N) |
| `now_y_block_channel_ch` | STRING | Channel |
| `now_y_block_employee_egt1..egt4` | STRING | Employee Group (4 levels) |
| `now_y_block_hr_hr1..hr3` | STRING | HR (3 levels) |
| `now_y_block_sec` | STRING | Security level |
| `now_y_block_period_mx` | STRING | Month Relative |
| `now_y_block_period_dx` | STRING | Day |
| `now_y_block_period_ppc` | STRING | PrevPeriodCohort |
| `now_y_block_period_np` | STRING | NowPeriod |
| `now_y_block_le_le1..le2` | STRING | Legal Entity |
| `now_y_block_unit` | STRING | Unit |
| `now_y_block_td_bu` | STRING | TopDown / BottomUp |
| `now_np` | STRING | Period (vd: M2512) -- X-Block |
| `now_value` | FLOAT | Gia tri so lieu |
| `uploaded_at` | TIMESTAMP | Thoi gian upload -- ORDER BY DESC LIMIT 1 de lay latest |

> **Ghi chu tu PD260 row 160-180**: Khi query SOCell, dieu kien loc gom: z_block_*, now_y_block_* (KR fields + Filter fields), now_np (Period), now_zblock2_alt. Neu MyFilterItem = null thi dieu kien la "IS NULL".

---

#### **[E9] SOCell_SOCell** *(alloc_stage.socell_socell)*
Join table de thuc hien Drill-Down (from -> to relationship)

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `id` | STRING | PK |
| `to_cell_id` | STRING | FK -> SOCell.id (cell dich / tong hop) |
| `from_cell_id` | STRING | FK -> SOCell.id (cell nguon / chi tiet) |
| `relationship_type` | STRING | "AllocationRun" / "AggregateBy" / "Offset" |
| `allocation_run_id` | STRING | FK -> AllocationRun log |

> **Quan he tu PD260 V5.1-V5.2**:
> - AllocationRun: 1 From -> N To -> ghi 1 record moi SOCell To
> - AggregateBy: N From -> 1 To -> ghi N records moi SOCell To
> - Offset: 1 From -> 1 To -> ghi 1 record
> **Trang thai**: PD260 V5.2 ghi "Sua code de ghi bang SOCell_SOCell" - can xac nhan da implement chua.

---

#### **[E10] RepPage** *(Report_data.RepPage)*
Instance cua 1 bao cao duoc build -- cache ket qua build

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `id` | STRING | PK |
| `YNumber1` | INTEGER | Composite key: ZBlockPlan.YNumber x 1M + ZBlockForecast.YNumber x 1K + ALT.YNumber |
| `FK1` | STRING | FK -> RepTemp.REP_TEMP_TYPE |
| `Z_BLOCK_ZBlockPlan_Source` | STRING | Plan ZBlock1 components |
| `Z_BLOCK_ZBlockPlan_Pack` | STRING | |
| `Z_BLOCK_ZBlockPlan_Scenario` | STRING | |
| `Z_BLOCK_ZBlockPlan_Run` | STRING | |
| `Z_BLOCK_ZBlockForecast_Source` | STRING | Forecast ZBlock1 components |
| `Z_BLOCK_ZBlockForecast_Pack` | STRING | |
| `Z_BLOCK_ZBlockForecast_Scenario` | STRING | |
| `Z_BLOCK_ZBlockForecast_Run` | STRING | |
| `NOW_ZBlock2_ALT` | STRING | FK -> AllocationALT.ALT_code |
| `last_report_month` | STRING | vd: M2512 |
| `last_actual_month` | STRING | vd: M2510 |
| `latest_build_time` | TIMESTAMP | NULL neu chua build; trigger re-build |
| `created_at` | TIMESTAMP | |

> **Ghi chu tu PD260 row 20**: FindOrCreate logic -- neu chua co RepPage thi Insert roi goi BuildReport.

---

#### **[E11] RepCell** *(Report_data.RepCell)*
Du lieu tung o cua bao cao sau khi build

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `id` | STRING | PK |
| `FK_rep_page_id` | STRING | FK -> RepPage.id |
| `FK_rep_temp_block` | STRING | FK -> RepTempBlock.id |
| `MyRepTempBlock` | STRING | RepTemp type code |
| `ZNumber` | INTEGER | Z-level |
| `YNumber1` | INTEGER | = RepPage.YNumber1 |
| `YNumber2` | INTEGER | = RepTempBlock.YNumber2 (thu tu block) |
| `YNumber3` | INTEGER | Thu tu period (L = 0..119) |
| `Z_BLOCK_TYPE` | STRING | "Plan" / "ActualForecast" / "Delta" |
| `NOW_NP` | STRING | X-Period (vd: M2512) |
| `NOW_VALUE` | FLOAT | Gia tri |
| `NOW_Y_BLOCK_FNF_FNF` | STRING | Copy tu SOCell |
| `NOW_Y_BLOCK_KR_Item_Code_KR1..KR8` | STRING | Copy tu SOCell |
| `NOW_Y_BLOCK_KR_Item_Name` | STRING | Copy tu SOCell |
| *(tat ca cac cot Filter tu MyFilterItem)* | STRING | CDT1-4, PT, Funnel, Channel, EGT1-4, HR1-3, SEC, LE1-2, UNIT, TD_BU |
| `created_at` | TIMESTAMP | |

> **Ghi chu tu PD260 row 24**: LoadReport query RepCell voi: YNumber1, MyRepTempBlock, NOW_NP IN MyXPeriodLoadList, ORDER BY YNumber2, YNumber3, Z_BLOCK_TYPE.

---

#### **[E12] User** *(security)*

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `username` | STRING | PK |
| `display_name` | STRING | |
| `email` | STRING | |
| `role` | STRING | FCOO / BOD / MGT / MKT |
| `sec_level` | STRING | SEC1 / SEC2 / SEC3 / SEC4 |
| `created_at` | TIMESTAMP | |

> **Ghi chu tu PD200**: Security Levels:
> - Level 1: Build full reports nhung load/view chi CDT_allowed va PT_allowed
> - Level 2: View only allowed by username
> - Level 3: QA security features nang cao

---

#### **[E13] UserSecConfig** *(security)*
CDT-allowed va PT-allowed theo tung user

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `id` | STRING | PK |
| `username` | STRING | FK -> User.username |
| `allowed_type` | STRING | "CDT" / "PT" |
| `allowed_code` | STRING | Code cu the duoc xem |
| `level` | STRING | CDT1/CDT2/CDT3/CDT4 hoac PT1/PT2 |
| `created_at` | TIMESTAMP | |

---

#### **[E14] ActivityLog** *(audit)* **[BO SUNG]**

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `id` | STRING | PK |
| `username` | STRING | Nguoi thuc hien |
| `event_type` | STRING | "Upload" / "Calculate" / "AllocationRun" / "BuildReport" / "LoadReport" / "DrillDown" |
| `event_detail` | JSON | Chi tiet tham so: ZBlock, ALT, file, sheet, step_code |
| `status` | STRING | "SUCCESS" / "FAILED" / "IN_PROGRESS" |
| `created_at` | TIMESTAMP | |
| `completed_at` | TIMESTAMP | |

> **Ghi chu tu PD200**: Moi step trong Runner (110, 120, 130, 141-149, 150, 211-213, ...) deu can log.

---

#### **[E15] PlanEventConfig** *(workflow)* **[BO SUNG]**
Deadline tracking cho FPA Runner

| Cot | Kieu | Mo ta |
|-----|------|-------|
| `id` | STRING | PK |
| `z_block_id` | STRING | FK -> ZBlock1 |
| `runner_type` | STRING | "GH" / "COH" / "CEH" / "DH" / "HQ" / "MKT-X" |
| `step_code` | STRING | "110" / "120" / "149" / "150" / ... |
| `deadline_date` | DATE | Deadline cho step nay |
| `assigned_username` | STRING | FK -> User.username |
| `actual_done_at` | TIMESTAMP | NULL neu chua xong |
| `status` | STRING | "PENDING" / "DONE" / "LATE" |

---

### 1.3 Entity Relationship Diagram (Text)

```
ZBlock1 (SRC/PCK/SCN/RUN/FF) ----+
                                   |  YNumber
AllocationALT (ALT_code/YNumber) -+---> RepPage (YNumber1 composite)
                                   |         |
RepTemp <----FK1--- RepTempBlock   |         | BuildReport
   |                    |          |         |
   |                    | FilterTypeFull     |
   |               AllocationToItem          |
   |               AllocationListType        |
   |                    |                    v
   |                    +--- query ---> SOCell --> RepCell
   |                                      |     (Plan/AF/Delta)
   |                                 SOCell_SOCell
   |                                 (Drill-Down)
   |
   +-- RepTempBlockToBlock (Drill-Down by RepTemp)

User --- UserSecConfig --- sec_level, CDT/PT allowed
         ActivityLog --- audit trail
         PlanEventConfig --- workflow deadlines
```

---

### 1.4 Dimension Reference (tu PD510)

#### Z-BLOCK Dimensions
| Block | SubBlock | Code | Mo ta | Vi du Items |
|-------|----------|------|-------|-------------|
| ZB1 | SRC | SRC1-SRC3 | Source | ACT, PLN; YF, QF, MF; G, CO, CE, D, P |
| ZB1 | PCK | PCK | Pack | Pack9B-25Jan01 |
| ZB1 | SCN | SCN | Scenario | OPT, REAL, PESS |
| ZB1 | RUN | RUN | Run | R2025DEC11 |
| ZB1 | FF | FF | Frequency of Forecast | YF, QF, MF, WF, DF, HF |
| ZB2 | ALT | ALT | Allocation Type | CFA1-3, PLA1-5 |
| ZB2 | FI | FI | File (Database) | |
| ZB2 | SH | SH | Sheet (Table) | |

#### Y-BLOCK Dimensions -- KR (Key Result)
| Code | Mo ta | Levels |
|------|-------|--------|
| FNF | Finance/Non-Finance | KRF, KRN |
| KR1 | Key Result Level 1 | GI, BAC, CP, ARP, NO, RATE, ROAS, PRM, TD, CPEM, NOEM, CPI, ARPP, BOC, CPM |
| KR2 | Key Result Level 2 | OPX, GI, OMC, BHR, PRM, OR, COS, CUL, NNEW, REC, CPX, FAC, OTH, BAC, PL2-PL7, SAL, BON, INS, PIT, CAC, REP |
| KR3 | Key Result Level 3 | CAC, IMP, INS, MTCH, RQ, DAU, CLK, AOMC, PL2, BU, COS, PPS, GAC, PAR, INF, BIZ, DEV, OTH, INT, TAX, NOEM, SAL, GI, PAY, MAC, PER, ADD |
| KR4 | Key Result Level 4 | MAC, eCPM, CPI, DC, D0, MTCH, RQ, DAU, DX, INS, CLK, IMP, PL2, ALLOCATE, OTH, OPS, FAC, MGM, DEV, SET, EXP, OMC |
| KR5 | Key Result Level 5 | OMC, P1K, NP, PPC, PER, OTH, PPS, TESE, DEV, RNT, BRC, TAL, REC, TRA, OVH, D0, INS, CLK, IMP, PAY |
| KR6-KR8 | Key Result Level 6-8 | (expansion) |

#### Y-BLOCK Dimensions -- FILTER
| Filter | SubCode | Levels | Mo ta | Vi du Items |
|--------|---------|--------|-------|-------------|
| CDT | CDT, CDT-S, CDT-CS | CDT1-CDT4 | Budget Owner (Chu Du Toan) | Company/Venture, Center, Department, Team |
| PTNow | PT, PT-S, PT-CS | PT1-PT2, DU | Product Type Now | Game, Fitness; Duration |
| PTPrev | PT | PT1-PT2, DU | Product Type Previous | (tuong tu PTNow) |
| PTFix | OWT, AIT | OWT, AIT | Ownership & AI Type | In-house/Partnership, AI/Non-AI |
| PTSub | CTY, OST | CTY1-CTY2, OST | Country & OS | Tier1-3, US/France; iOS/Android |
| Funnel | FU | FU1-FU2 | Funnel | PS/PI/PA, R/N |
| Channel | CH | CH | Channel | |
| Employee | EGT | EGT1-EGT4 | Employee Group Type | (combined list tu 2 list goc) |
| HR | HR | HR1-HR2 | Human Resource | |
| Period | PX, PPC, NP | PX, PPC, NP | Period filters | DX-15, DC-365/MC-12 |
| LE | LE | LE1-LE2 | Legal Entity | |
| UNIT | UNIT | UNIT | Unit | |
| TDBU | TDBU | TDBU | Top Down - Bottom Up | TD, BU |

#### X-BLOCK Dimensions -- Period
| Code | Mo ta | Vi du |
|------|-------|-------|
| Y | Year | 2025, 2026, ... |
| Q | Quarter | Q1-Q4 |
| M | Month | M1-M12 |
| W | Week | W1-W5 |
| D | Day | D1-D365 |

---

### 1.5 KRF Hierarchy Reference (tu PD530)

```
KRF (Key Result Financial)
|
+-- L1: GI (Gross Inflow) / BAC (Business Activities Cost) / BOC (Business Ops Cost)
|   |
|   +-- L2: Revenue streams, Cost categories
|   |   |  GI > Revenue
|   |   |  BAC > OPX (Operating Expense)
|   |   |  BAC > CAC (Customer Acquisition Cost)
|   |   |  BAC > COS (Cost of Service)
|   |   |  BAC > GAC (General Admin Cost)
|   |   |  BAC > CDC (Capital Development Cost)
|   |   |
|   |   +-- L3: Sub-categories (CAC, IMP, INS, MTCH, ...)
|   |   |   |
|   |   |   +-- L4: Operational subdivisions
|   |   |   |   |  L4.HR1: HR vs NHR (2 row variants)
|   |   |   |   |  L4.HR2: SAL+BON+INS+PIT+NHR (5 row expansion)
|   |   |   |   |
|   |   |   |   +-- L5-L8: Deep operational breakdowns
|   |   |   |       (OMC, P1K, NP, PPC, PER, OTH, ...)

RepTemp mapping:
  KRF-L4.CDT0  -> Level 4, no CDT filter (tong quan)
  KRF-L4.CDT1  -> Level 4, CDT Level 1 filter
  KRF-L8-GI.CDT0 -> Level 8 (chi GI), no CDT filter
```

> **Ghi chu tu PD530**: KRF v1.7 co 50 base rows voi provisions for expansion to 42+ rows. Supports 11 nam du lieu (2020-2030) voi multiple period granularities.

---

## 2. Process Description

### 2.1 Tong quan quy trinh (tu PD200 Runner Guide)

```
ACTUAL FLOW (Last 1 Month):
Step 110: Copy-paste Last 1 Month data -> Folder RAW
   -> Step 120: Import RAW -> APR-SEC (HQ/Non-HQ variants)
   -> Step 130: APR Calculate (formula auto-link)
   -> Steps 141-147: ACA file allocations:
      141: CFA1-ToLE
      142: CFA2-ToCDT
      143: CFA3-ToKRF
      144: PLA1-ToLE
      145: PLA2-ToCDT
      146: PLA3-ToKRF
      147: PLA4-ToPT
   -> Step 148: Calculate on ACA (PD221)
   -> Step 149: Upload to SO-330
   -> Step 150: AllocationRunActual (ALT 100-299)

PLAN FLOW (All Months 2026-2030):
Step 311-312: PPR Planning (R20-R100) -> Input, Upload
   -> Steps 211-213: MKT-X Planning (PCA-2100-PLA5_MKT-X) -- xem PD220 PC_3
      211: Input R20
      212: Calculate (PD220 PC_3 formulas: NO, RATE, GI, OMC)
      213: Upload
   -> Step 220: AllocationRunPlan_MKT-X (ALT 300-399)
   -> Steps 321-323: Subscription Planning (PCA-2200-PLA5) -- xem PD220 PC_3
      321: Download SI-110
      322: Calculate (PD220 PC_3 subscription formulas)
      323: Upload BU
   -> Steps 331-333: PLA4-ToPT Planning (PCA-2300-PLA4) -- xem PD220 PC_1, PC_2
      331: Download SI-110
      332: Calculate (PD220 PC_1 PL formulas, PC_2 CDT aggregation)
      333: Upload
   -> Step 340: AllocationRunPlan_All (ALT 400-499)
   -> Step 350: Download PPR IMP

REPORT FLOW:
Step 400: Select RepTemp, ZBlock, ALT
   -> LoadReport() -> (if no RepPage) -> BuildReport()
   -> RepCell array -> Render UI (Plan / ActualForecast / Delta)
   -> Drill-Down (by RepTemp / by SOCellToCell)
```

---

### 2.2 Process: BuildReport() (tu PD260)

**API**: `POST /api/report/build` -> tra ve `task_id` (async background job)
**Theo doi**: `GET /api/task/{task_id}` -> polling trang thai task

**Input**: `MyRepTemp`, `MyZBlockPlan`, `MyZBlockForecast`, `MyALT`, `MyLastReportMonth`, `MyLastActualMonth`
**Output**: `RepPage` (moi hoac da ton tai), `RepCell` (N records)

| Step | Mo ta | Chi tiet |
|------|-------|----------|
| **30** | Query RepTemp | `SELECT * FROM RepTemp WHERE REP_TEMP_TYPE = MyRepTemp` -> lay ZNumber |
| **31** | Lay MyZNumber | `MyZNumber = RepTemp.ZNumber` |
| **32** | Query RepPage | `SELECT * FROM RepPage WHERE FK1=MyRepTemp AND ZBlockPlan=... AND ZBlockForecast=... AND ALT=MyALT` |
| **33** | Lay MyYNumber1 | `MyYNumber1 = RepPage.YNumber1` |
| **21** | Tinh YNumber1 | `= ZBlockPlan.YNumber x 1,000,000 + ZBlockForecast.YNumber x 1,000 + ALT.YNumber` |
| **21A** | Neu chua co | `RepPage.LatestBuildTime = NULL` -> Insert RepPage, goi BuildReport |
| **21B** | Neu da co | Load 6 thang gan nhat (MyXPeriodLoadList) |
| **40** | Query RepTempBlock | `SELECT * FROM RepTempBlock WHERE FK1 = MyRepTemp` -> MyRepTempBlockList |
| **50** | ForEach block | `ForEach MyRepTempBlock IN MyRepTempBlockList (YNumber2 Increasing)` |
| **60** | Lay YNumber2 | `MyYNumber2 = MyRepTempBlock.YNumber2` |
| **70** | Parse FilterTypeFull | `MyRepTempBlock.FilterTypeFull -> MyFilterTypeList (MaxNumber=N)` |
| **80** | Parse KRTypeFull | `MyRepTempBlock.KRTypeFull -> MyKRTypeFull` |
| **90** | Clear temp | `Clear RepFilterItem Table` |
| **100** | Build FilterItems | `BuildFilterItemList()` -- xem Complex Logic 5.1 |
| **120-130** | Nested loops | `ForEach MyFilterItem` x `ForEach L=0 to 119` |
| **140** | Tinh period | `MyXPeriod = MyLastReportMonth - L` |
| **150** | Tinh YNumber3 | `MyYNumber3 = L` |
| **160** | Query Plan | `SOCell1 = query SOCell(MyZBlockPlan, MyKRTypeFull, MyFilterItem, MyXPeriod, MyALT)` |
| **170** | Query Actual | `SOCell2 = query SOCell(ZBlock="ACT"/Actual, MyKRTypeFull, MyFilterItem, MyXPeriod, MyALT)` |
| **180** | Query Forecast | `SOCell3 = query SOCell(MyZBlockForecast, MyKRTypeFull, MyFilterItem, MyXPeriod, MyALT)` |
| **190** | Write Plan | `RepCell(ZBlockType="Plan", value=SOCell1.now_value)` |
| **200-210** | Write AF-Actual | `IF LastActualMonth >= MyXPeriod: RepCell(ZBlockType="ActualForecast", value=SOCell2.now_value)` |
| **220-230** | Write AF-Forecast | `ELSE: RepCell(ZBlockType="ActualForecast", value=SOCell3.now_value)` |
| **240** | Tinh Delta | `DeltaValue = AF.now_value - Plan.now_value` (NULL handling) |
| **250** | Write Delta | `RepCell(ZBlockType="Delta", value=DeltaValue)` -- Copy tat ca KR + Filter fields |

---

### 2.3 Process: LoadReport() (tu PD260)

**API**: `POST /api/report/load` -- Diem vao chinh (smart entry point)
**Behavior**: Tra report neu da co; neu chua thi tu dong trigger build va tra `task_id`

**Input**: `MyRepTemp`, `MyZBlockPlan`, `MyZBlockForecast`, `MyALT`, `MyLastReportMonth`, `MyLastActualMonth`
**Output**: Array `RepCell`(N) HOAC `task_id` (neu can build)

```
1. FindOrCreate RepPage:
   - Tim RepPage voi (MyRepTemp, MyZBlockPlan, MyZBlockForecast, MyALT)
   - Neu RepPage.LatestBuildTime != NULL -> da co data, nhay buoc 2
   - Neu RepPage chua co HOAC LatestBuildTime = NULL:
       -> Tu dong trigger POST /api/report/build (internal)
       -> Return { status: "building", task_id: "xxx" }
       -> Frontend polling GET /api/task/{task_id} cho den khi done

2. Tao MyXPeriodLoadList = 6 thang gan nhat:
   For M = 0 to 5:
     MyXPeriodLoad = MyLastReportMonth - M
   VD: {M2512, M2511, M2510, M2509, M2508, M2507}

3. Query RepCell:
   SELECT * FROM RepCell
   WHERE YNumber1 = RepPage.YNumber1
     AND MyRepTempBlock = MyRepTemp
     AND NOW_NP IN MyXPeriodLoadList
   ORDER BY YNumber2, YNumber3, Z_BLOCK_TYPE

4. Apply Security Filter (CDT_allowed, PT_allowed theo username)

5. Return { status: "ready", data: Array RepCell }
```

---

### 2.4 Process: AllocationRun (tu PD200)

**Input**: `MinALT`, `MaxALT`, `Period`
**Output**: Ghi vao `SOCell`; ghi quan he vao `SOCell_SOCell`

| Sub-process | ALT Range | Steps | Trigger |
|-------------|-----------|-------|---------|
| AllocationRunActual | 100-299 | Step 150 | Sau Upload SO-330 |
| AllocationRunPlan_MKT-X | 300-399 | Step 220 | Sau Upload PCA-2100 |
| AllocationRunPlan_All | 400-499 | Step 340 | Sau Upload PCA-2200, PCA-2300 |

> **Ghi chu tu PD200**: Moi AllocationRun ghi SOCell_SOCell de phuc vu Drill-Down. Quan he: AllocationRun (1 From -> N To), AggregateBy (N From -> 1 To), Offset (1-1).

---

### 2.5 Process: Drill-Down (tu PD260)

#### Type A -- Drill-Down by RepTemp (PD241) [CURRENT - dang dung hien tai]

```
1. User click [+] o 1 row/block trong report
2. Lay MyKR, MyFilterItemList tu row do
3. Voi moi MyFilterItem:
   -> Xac dinh MyFilterType, MyFilterLevel
   -> Khu NoOfRows: cat bang "-", bo Segment toan so, giu Segment dau = Main
4. Query RepTempBlockToBlock:
   (MyKR, MyFilterType, MyFilterLevel) -> NowRepTempBlock -> PrevRepTempBlock(N)
5. Tao RepTemp moi: NowRepTempBlock + PrevRepTempBlock(N)
6. Goi BuildReport() voi RepTemp moi
7. Hien thi sub-rows inline
```

> **Ghi chu tu PD260**: MyRepTemp = (MyRepTempX, MyRepTempY, MyRepTempZ)
> DrillX: xuong X, giu nguyen Y, Z
> Tuong tu cho DrillY, DrillZ

#### Type B -- Drill-Down by SOCellToCell (V5.1/V5.2) [TARGET - dang mong muon]

```
1. User click vao 1 dong du lieu trong UI
2. Frontend object -> tim RepCell
3. Tu RepCell.PrimaryKey(X,Y,Z) -> tim SOCell (To)
4. Tu SOCell(To) -> query SOCell_SOCell -> lay N SOCell(From)
5. Hien thi cac SOCell(From) duoi dong do (Collapse/Expand)
```

#### Type C -- Drill-Down by SOCellToCell (Prev = APR) [FAR FUTURE - tuong lai xa]
- Dung cho ACTUAL data voi CFA1-3, PLA1-3
- Upload APR len SOCell, Query tu KRF + LE/CDT/PT Item ra
- HR rieng: a) Bao mat b) nguoi cu (luong cu the) + nguoi moi (CPEM theo EGT4)

---

### 2.6 Process: Upload/Download (tu PD200)

| Step | Process | Input | Output |
|------|---------|-------|--------|
| 110 | Upload RAW | Copy-paste Last 1 Month data | Folder RAW |
| 120 | Import APR-SEC | RAW folder | APR-SEC (HQ/Non-HQ) |
| 130 | APR Calculate | APR file | APR with formulas |
| 141-148 | ACA Allocations | APR, APR-SEC | ACA with CFA1-3, PLA1-4, Calculate |
| 149 | Upload SO-330 | ACA calculated | SOCell in BigQuery |
| 211-213 | MKT-X Input | R20-R100 | PCA-2100-PLA5_MKT-X |
| 311-312 | PPR Input | Manual | PPR R20-R100 |
| 321-323 | Subscription | Download SI-110 | PCA-2200-PLA5 |
| 331-333 | PLA4-ToPT | Download SI-110 | PCA-2300-PLA4 |
| 350 | Download PPR | AllocationRun result | PPR IMP file |

---

### 2.7 Process: Plan Calculate (tu PD220)

PD220 dinh nghia cau truc va cong thuc tinh toan tren cac sheet PCA (Plan Calculate Actual) trong Google Sheets. Gom 4 phan: PC_1, PC_2, PC_3 (Calculate), va RP (Report).

#### 2.7.1 Tong quan PCA Sheets

```
PCA-PLA4 (Plan Allocation Level 4):
  PCA290-KRF    -- KR Hierarchy master (GI, BAC, FIN, PRM)
  PCA280-BYKR   -- Conversion/penetration ratios
  PCA240-COS    -- Cost of Service breakdown by CDT
  PCA230-CAC    -- Customer Acquisition Cost breakdown by CDT
  PCA210-GI     -- Gross Inflow breakdown by CDT
  PCA250-GAC    -- General Admin Cost breakdown by CDT
  PCA260-CPX    -- Capex breakdown by CDT

PCA-PLA5 (Plan Allocation Level 5 -- MKT-X):
  PCA-PLA5-GI    -- Gross Inflow by PT x CTY
  PCA-PLA5-OMC   -- Online Marketing Cost by CDT x PT x CTY
  PCA-PLA5-RATE  -- Rates (CPI, ARPP, Retention, Subscription)
  PCA-PLA5-NO    -- Number metrics (Install, Click, Impression, PaidApp, PaidSub)
```

#### 2.7.2 KRF Hierarchy -- Full Tree (tu PD220 PC_1)

```
KRF (Key Result Financial)
|
+-- GI (Gross Inflow / Revenue / Cash Inflow from Ops)
|   +-- GI > GI > GI (leaf)
|
+-- BAC (Business Activities Cost / Cash Outflow)
|   +-- OPX (Operation Expense)
|   |   +-- CAC (Customer Acquisition Cost)
|   |   |   +-- MAC (Marketing Cost)
|   |   |   |   +-- OMC (Online Marketing Cost)
|   |   |   |   +-- OTH (Other)
|   |   |   +-- OTH (Other)
|   |   +-- COS (Cost of Service)
|   |   |   +-- OPS (Operations)
|   |   |   |   +-- PPS (Partner Profit Share of PL2)
|   |   |   |   +-- TESE (Tech Service Cost)
|   |   |   |   |   +-- API | SERV | LIC | OTH
|   |   |   |   +-- DEV (Development / Maintain Apps)
|   |   |   |   +-- OTH
|   |   +-- GAC (General Admin Cost)
|   |   |   +-- FAC (Facilities)
|   |   |   |   +-- RNT (Office Rental) | OTH
|   |   |   +-- MGM (Management)
|   |   |   |   +-- BRC (Branding) | CUL (Talent & Culture)
|   |   |   |   +-- REC (Recruitment) | TRA (Travel) | OTH
|   |   |   +-- DEV (In-House Apps) | OTH
|   +-- CPX (Capex & Depreciation / Cash Outflow from Investing)
|   |   +-- CPX
|   |   |   +-- PAR (Partnership Dev) | INF (Infrastructure)
|   |   |   |   +-- SET (Office Setup) | OTH
|   |   |   +-- BIZ (Biz Dev) | DEV (Build Apps)
|   |   +-- OTH
|   +-- ITX (Interest & Tax)
|   |   +-- ITX > INT (Interest) | TAX (CIT)
|   |   +-- OTH
|   +-- OTH > OTH > COMP (Compensation) | OTH
|
+-- FIN (Financing)
|   +-- STOCK (Net Proceeds from Issuance/Repurchase)
|   |   +-- INF (Inflow) | OUTF (Outflow) | COST
|   +-- DEBT (Net Proceeds from Debt)
|   |   +-- INF | OUTF | COST
|   +-- DIV (Dividend)
|       +-- INF | OUTF | COST
|
+-- PRM (Profit / Margin)
    +-- BHR (Before HR)
    |   +-- AOMC > PL2 (After Marketing)
    |   +-- ATESE > PL3 (After Tech Services)
    |   +-- APPS > PL4 (After Partner Profit Share)
    +-- AHR (After HR)
    |   +-- ABON > PL5 (After Marketer Bonus)
    |   +-- AVHR > PL6 (After All Venture HR)
    +-- AVAR (After Variable Costs)
    |   +-- CM1 > GM (Gross Margin / Contribution Margin 1)
    |   +-- CM2 | CM3
    +-- AFIX (After Fix Costs)
    |   +-- EBITDA > NCFO (Net Cashflow from Ops)
    |   +-- ADA > EBIT (Earnings Before Int & Tax)
    +-- AINT > EBT > PL7 (Earnings Before Tax)
    +-- ATAX > NP > NCFOI (Net Profit / Net CF from Ops & Investing)
    +-- AFIN > NCF (Net Cashflow After Financing)
```

#### 2.7.3 P&L Waterfall Formulas (tu PD220 PC_1 TECHNICAL BLOCK)

```
BAC-[L7-50]     = HRC + NHRC                (HR Cost + Non-HR Cost)

PL2 (PRM-BHR-AOMC-PL2)
  = GI - BAC-OPX-CAC-MAC-OMC              [theo CDT-S1]

PL3 (PRM-BHR-ATESE-PL3)
  = PL2 - BAC-OPX-COS-OPS-TESE           [theo CDT-S1, after allocation]

PL4 (PRM-BHR-APPS-PL4)
  = PL3 - BAC-OPX-COS-OPS-PPS            [theo CDT-S1, after allocation]

PL5 (PRM-AHR-ABON-PL5)
  = PL4 - BAC-OTH-COMP-HRC-BON-PL2       [theo CDT-S1]

PL6 (PRM-AHR-AVHR-PL6)
  = PL5 - SAL - INS - PIT                 [theo CDT-S1]
  (SAL = BAC-OTH-COMP-HRC-SAL, INS = BAC-OTH-COMP-HRC-INS, PIT = BAC-OTH-COMP-HRC-PIT)

PL7 (PRM-AINT-EBT-PL7)
  = PL6 - Total HQ Cost                   [theo CDT-S1, after allocation]
```

#### 2.7.4 CDT Aggregation Formulas (tu PD220 PC_2)

Moi muc chi phi duoc breakdown theo CDT cascading:

```
Aggregation Chain:
  KR.CDT0     = Aggregate(KR.CDT-S1)        -- Total = sum of CDT Level 1
  KR.CDT-S1   = Aggregate(KR.CDT-CS2)       -- CDT L1 = sum of CDT Level 2
  KR.CDT-CS2  = sum of child KR items        -- CDT L2 = sum of sub-categories

Vi du COS (Cost of Service):
  BAC-OPX-COS.CDT0       = Aggregate(BAC-OPX-COS.CDT-S1)
  BAC-OPX-COS.CDT-S1     = Aggregate(BAC-OPX-COS.CDT-CS2)
  BAC-OPX-COS.CDT-CS2    = BAC-OPX-COS-OPS.CDT-CS2
  BAC-OPX-COS-OPS.CDT-CS2 = PPS + TESE + DEV + OTH  [tat ca .CDT-CS2]

Vi du CAC (Customer Acquisition Cost):
  BAC-OPX-CAC.CDT0       = Aggregate(BAC-OPX-CAC.CDT-S1)
  BAC-OPX-CAC.CDT-S1     = Aggregate(BAC-OPX-CAC.CDT-CS3)
  BAC-OPX-CAC.CDT-CS3    = MAC + OTH
  BAC-OPX-CAC-MAC.CDT-CS3 = OMC + OTH

COS-PPS Formula:
  BAC-OPX-COS-PPS.CDT-CS2 = PRM-BHR-ATESE-PL3 * RATE-COS-SHA.CDT-CS2

OMC Formula:
  BAC-OPX-CAC-MAC-OMC.CDT-CS3 = GI.CDT-CS3 / PRM-BHR-AOMC-PL2.CDT-CS
```

**I/O Pattern cho CDT sheets**:
| Sheet | IO | FilterTypeFull | Rows | CDT Level |
|-------|-----|---------------|------|-----------|
| PCA240-COS | SHO (Sheet Output) | NHRC | 1 (CDT0), 8 (CDT-S1, CDT-CS2) | CDT0 -> CDT-S1 -> CDT-CS2 |
| PCA230-CAC | SHO (Sheet Output) | NHRC | 1 (CDT0), 8 (CDT-S1, CDT-CS3) | CDT0 -> CDT-S1 -> CDT-CS3 |
| PCA210-GI | SHI (Sheet Input) | NHRC | 5 (CDT-S1) | GI.CDT-CS2 = Input from 330-SO |

Leaf items (API, SERV, LIC, OTH, DEV, ...) co: `I/O = I` (Input) va `Source = SHI` (Sheet Input) hoac `INPUT by`.

#### 2.7.5 BYKR -- Conversion Ratios (tu PD220 PC_1 PCA280-BYKR)

```
IO: PCA-PLA4 | PCA280-BYKR
Cac ratio dung de phan bo (BY = "By", ratio de convert):

Product Type ratios:
  BY-INS-TO-PTS2-FROM-PT0   = NO-INS-PTS2 / NO-INS-PT0
  BY-DAU-TO-PTS2-FROM-PT0   = NO-DAU-PTS2 / NO-DAU-PT0
  BY-GI-TO-PTS2-FROM-PT0    = GI-PTS2 / GI-PT0

CDT ratios (NOEM = Number of Employees):
  BY-NOEM-TO-CDTx-FROM-CDT0  = NO-NOEM-CDTx / NO-NOEM-CDT0
  (x = 1, 2, 3, 4)

CDT ratios (NNEM = Non-Employee):
  BY-NNEM-TO-CDTx-FROM-CDT0  = NO-NNOEM-CDTx / NO-NNOEM-CDT0
  (x = 1, 2, 3, 4)

DAU/DU pricing:
  BY-DAU-TO-PX-DP15-FROM-NP-BU = KRN-RATE-GI-DAU-DX-PER-D0
  BY-DU-TO-PX-MP12-FROM-NP     = Input
  BY-DU-TO-PX-DP15-FROM-NP     = Input

Source: Numerators & Denominators tu SHI (SheetInput)
```

#### 2.7.6 MKT-X Planning Metrics (tu PD220 PC_3 -- PCA-PLA5)

```
IO: PCA-PLA5 (MKT-X specific)
Filters: PT-CS-30 (30 products), CTY2-17 (17 countries),
         IAPS/PA (In-App Purchase Subscription / Paid App),
         DU-3 (3 durations: 1W/4W/52W), Funnel (R1/R2/N)

--- NUMBER metrics (KRN > NO) ---
NO-PS     (Paid Subscription)     = NO-OMC-INS * RATE-PS-PER-INS
NO-PA     (Paid App)              = NO-OMC-INS * RATE-PA-PER-INS
NO-OMC-INS (Install)              = BAC-CAC-MAC-OMC / CPI
NO-OMC-CLK (Click)                = NO-OMC-INS / RATE-OMC-INS-CLK
NO-OMC-IMP (Impression)           = NO-OMC-CLK / RATE-OMC-CLK-IMP

--- RATE metrics (KRN > RATE) ---
RATE-REP-PER-EXP    (Retention Rate)        = Input (per R1/R2)
RATE-PS-DU-PER-PS   (Duration/Subscription)  = Input (per 1W/4W/52W)
RATE-PS-PER-INS     (Subscription/Install)   = Input
RATE-PA-PER-INS     (Paid App/Install)        = Input
RATE-OMC-INS-PER-CLK (Install/Click)          = Input
RATE-OMC-CLK-PER-IMP (Click/Impression)       = Input

--- COST metrics (KRN/KRF) ---
CPI        (Cost Per Install)                 = Input
ARPP.IAPS  (Avg Revenue Per Package - IAPS)   = Input (per DU)
ARPP.PA    (Avg Revenue Per Package - PA)     = Input

--- REVENUE (KRF > GI) ---
GI = NO-PS * ARPP.IAPS + NO-PA * ARPP.PA    (tinh tu metrics)

--- COST (KRF > BAC-CAC-MAC-OMC) ---
BAC-CAC-MAC-OMC = Input (40 CDT files x PT x CTY)
Rows: CDT-CS4-40 x PT-CS-30 x CTY2-17 x IAPS = 20,400 max rows
```

#### 2.7.7 GI Calculation (tu PD220 PC_3 -- PCA-PLA5-GI)

```
Gross Inflow calculation chain:

GI-GI-GI (total by PT x CTY)
  = GI-GI-GI.IAPS + GI-GI-GI.PA         (In-App Subscription + Paid App)

GI-GI-GI.IAPS (per PT-CS, CTY2, DU)
  = NO-PS * ARPP.IAPS                     (Number of Paid Sub x Avg Revenue)

GI-GI-GI.PA (per PT-CS, CTY2)
  = NO-PA * ARPP.PA                        (Number of Paid App x Avg Revenue)

Subscription flow:
  New Install -> Paid Sub (RATE-PS-PER-INS)
    -> Retention R1 -> R2 -> ... (RATE-REP-PER-EXP)
    -> Each has Duration split (1W/4W/52W via RATE-PS-DU-PER-PS)
    -> Revenue = count * ARPP per duration
```

#### 2.7.8 Report Display (tu PD220 RP)

PD220 RP dinh nghia cach hien thi ket qua Plan Calculate tren Report. Cung KR hierarchy nhu PC nhung chi co Output (O), khong co formula.

**8 Report Template columns x 3 View Types = 24 columns:**

| # | RepTemp Column | Mo ta |
|---|---------------|-------|
| 1 | KRF-L4 | Level 4 tong hop (GI, CAC, COS, GAC, CPX, PL2..PL7) |
| 2 | KRF-L8-GI | Level 8 chi tiet GI |
| 3 | KRF-L8-CAC | Level 8 chi tiet CAC (MAC->OMC/OTH, OTH) |
| 4 | KRF-L8-COS | Level 8 chi tiet COS (OPS->PPS/TESE/DEV/OTH) |
| 5 | KRF-L8-GAC | Level 8 chi tiet GAC (FAC, MGM, DEV, OTH) |
| 6 | KRF-L8-CPX | Level 8 chi tiet CPX (PAR, INF, BIZ, DEV, OTH) |
| 7 | KRF-L8-PL2 | Level 8 chi tiet PL2 (Profit After Marketing) |
| 8 | KRF-L8-PL7 | Level 8 chi tiet PL7 (Earnings Before Tax) |

Moi column lap lai cho 3 view: **CDT0** (tong), **CDT1** (by CDT level 1), **PT-S2** (by PT sub-level 2).

**RepTemp Position Mapping (KR row -> vi tri so trong report):**

```
KRF-L4 (tong hop - 11 dong):
  1: GI          6: PL2 (PRM-BHR-AOMC)
  2: CAC         7: PL3 (PRM-BHR-ATESE)
  3: COS         8: PL4 (PRM-BHR-APPS)
  4: GAC         9: PL5 (PRM-AHR-ABON)
  5: CPX        10: PL6 (PRM-AHR-AVHR)
                11: PL7 (PRM-AINT-EBT)

KRF-L8-GI (3 dong, chi co PT-S2):
  1: GI (header)     2: GI-GI     3: GI-GI-GI

KRF-L8-CAC (5 dong):
  1: CAC (header)    2: MAC    3: OMC    4: MAC-OTH    5: CAC-OTH

KRF-L8-COS (10 dong):
  1: COS (header)    2: OPS       3: PPS      4: TESE     5: API
  6: SERV            7: LIC       8: TESE-OTH 9: DEV     10: OPS-OTH

KRF-L8-GAC (12 dong):
  1: GAC (header)    2: FAC       3: RNT      4: FAC-OTH  5: MGM
  6: BRC             7: CUL       8: REC      9: TRA     10: MGM-OTH
  11: DEV           12: GAC-OTH

KRF-L8-CPX (9 dong):
  1: CPX (header)    2: CPX-CPX   3: PAR      4: INF      5: SET
  6: INF-OTH         7: BIZ       8: DEV      9: CPX-OTH

KRF-L8-PL2: 1 dong (PL2)
KRF-L8-PL7: 1 dong (PL7) -- PL7 cung xuat hien tai KRF-L4 pos 11
```

**Luu y:**
- FIN (Financing), ITX (Interest & Tax), BAC-OTH: **khong co RepTemp position** (khong hien thi tren report)
- Dong 105 PD220 RP: ghi "PRT" thay vi "PRM" (co the la typo trong PD goc)
- PL3, PL4, PL5, PL6: chi co trong KRF-L4, khong co L8 drill-down rieng

**Drill-Down by RepTemp:**
- REP1110: Report Level 4 (KRF-L4) -- view tong hop
- REP1120: Report Level 8 (KRF-L8-*) -- drill-down chi tiet theo tung nhom (GI, CAC, COS, GAC, CPX, PL2, PL7)
- Khi user click vao dong GI (pos 1) trong REP1110 -> mo REP1120 voi KRF-L8-GI
- Tuong tu: click CAC -> KRF-L8-CAC, click COS -> KRF-L8-COS, v.v.

#### 2.7.9 Sheet I/O Flow

```
Input Sources:
  SHI (SheetInput)  -- Data nhap truc tiep tu user
  330-SO             -- Data tu SO-330 (upload tu ACA)
  Input              -- Manual input fields

Output Targets:
  SHO (SheetOutput) -- Data output de cac sheet khac su dung
  O                  -- Output column (data tinh duoc)

Flow:
  330-SO (GI data) --> PCA210-GI (input)
  PCA290-KRF (master formulas) <-- PCA240-COS, PCA230-CAC, PCA250-GAC, PCA260-CPX
  PCA280-BYKR (ratios) <-- SheetInput (user input ratios)
  PCA-PLA5 (MKT-X) <-- SheetInput (CPI, ARPP, Rates)
                    --> OMC, Install, Revenue calculations
```

---

## 3. UI Wireframe

### 3.1 Report UI -- Version 1 (Don gian, ban dau)

```
+------------------------------------------------------------------+
|  APERO FP&A -- REPORT                                             |
+------------------------------------------------------------------+
|  Report Template: [v Select MyRepTemp          ]                  |
|  Last Reporting Month: [v Select ]  Last Actual Month: [v ]      |
+------------------------------------------------------------------+
|  [  Load Report  ]  [  Re-Build Report  ]                         |
+-----------------------+------------+--------------+---------------+
|  Metric / Filter      |   Plan     | Actual/Fcst  |   Delta       |
+-----------------------+------------+--------------+---------------+
|  v [RepTempBlock 1]   |  100       |  90          |  -10          |
|  v [RepTempBlock 2]   |  200       |  220         |  +20          |
|  v [RepTempBlock 3]   |   50       |   55         |   +5          |
|  ...                  |  ...       |  ...         |  ...          |
+-----------------------+------------+--------------+---------------+
|  *Hien thi 6 thang gan nhat; Plan/ActualForecast/Delta columns*  |
+------------------------------------------------------------------+
```

---

### 3.2 Report UI -- Version 2 (Day du filter ZBlock, trung gian)

```
+------------------------------------------------------------------------+
|  APERO FP&A -- REPORT V2                                                |
+------------------------------------------------------------------------+
|  Report Template: [v Select MyRepTemp     ]                             |
|                                                                         |
|  PLAN:     Pack[v] Scenario[v] Source[v] FF[v] Run[v]                  |
|  FORECAST: Pack[v] Scenario[v] Source[v] FF[v] Run[v]                  |
|                                                                         |
|  ALT: [v PLA4 ]  Last Report Month: [v M2512 ]  Last Actual: [v M2510] |
|  PT:  [v All  ]  CDT: [v All       ]                                   |
+------------------------------------------------------------------------+
|  [  Load Report  ]  [  Re-Build Report  ]                               |
+---------------------+----------+--------------+---------+--------------+
|  Metric / Filter    |  Plan    | Actual/Fcst  |  Delta  | Delta%       |
+---------------------+----------+--------------+---------+--------------+
| > GI                |  1,000   |  900         |  -100   | -10%         |
|   > CDT1-A          |    400   |  380         |   -20   |  -5%         |
|     > CDT2-A1  [+]  |    200   |  190         |   -10   |  -5%         |
|     > CDT2-A2  [+]  |    200   |  190         |   -10   |  -5%         |
| > BAC               |    800   |  850         |   +50   |  +6%         |
|   > OPX             |    500   |  520         |   +20   |  +4%         |
|   > CAC             |    300   |  330         |   +30   | +10%         |
| ...                 |  ...     |  ...         |  ...    |  ...         |
+---------------------+----------+--------------+---------+--------------+
|  [+] = Drill-Down by RepTemp | Click row = Drill-Down by SOCell        |
+------------------------------------------------------------------------+

Plan Table: Hien thi cac RepCell co ZBlockType = Plan
ActualForecast: Hien thi cac RepCell co ZBlockType = ActualForecast
Delta Table: Hien thi cac RepCell co ZBlockType = Delta
```

> **V2 Cascade Dropdown Logic (tu PD260)**:
> - "PLN" -> PCKList
> - "PLN", MyPCK -> SCNList
> - "PLN", MyPCK, MySCN -> SRCList
> - "PLN", MyPCK, MySCN, MySRC -> FFList
> - "PLN", MyPCK, MySCN, MySRC, MyFF -> RUNList
> - Tuong tu cho FORECAST

---

### 3.3 Report UI -- Version 3 (V3 - Hien tai, tu Screenshot)

```
+============================================================================================+
| [Li] Financial Metrics Report · KRF-L4.CDT0                    [Gear] [V1] [V2] [V3*]     |
+============================================================================================+
| Template: [v KRF-L4.CDT0]  Report: [v Mar 2026]  Actual: [v Dec 2025]                     |
| Unit: [v mVND]  1 USD = [25000] VND  [To USD btn]                          [Load][Rebuild] |
+--------------------------------------------------------------------------------------------+
| PLAN:     PCK:[v planfullver1] SCN:[v REAL] SRC:[v G] FF:[v TD-YF] RUN:[v R26FEB25]       |
| FORECAST: PCK:[v planfullver1] SCN:[v REAL] SRC:[v G] FF:[v TD-YF] RUN:[v R26FEB25]       |
|                                                        ALT:[v PLA4] PT:[v Select] CDT:[v]  |
+------+------------------------------+-------------------+--+------------------+--+---------+
| CDT  | Metrics                      |       PLAN        |  | ACTUAL/FORECAST  |  |  DELTA  |
|      |                              | Plan R26FEB25     |  | Actual | Forecast|  | Act/Fcs |
|      |                              |                   |  |        |         |  | - Plan  |
|      |                              | Oct Nov Dec Jan   |  | Oct Nov| Jan Feb |  | Oct ... |
|      |                              |  25  25  25  26   |  |  25  25|  26  26 |  |  25     |
|      |                              | Feb Mar           |  | Dec    | Mar     |  | ... Mar |
|      |                              |  26  26           |  |  25    |  26     |  |      26 |
+------+------------------------------+-------------------+--+--------+---------+--+---------+
|> Grp | GI (Revenue, Cash Inflow)    | 0  0  0  85K 93K  |  | 0  0  0| 0  0  0 |  | 0 ... -102K|
|> Grp | CAC (Customer Acquisition)   | 0  0  0   0   0   |  | 0  0  0| 0  0  0 |  | 0 ...    0 |
|> Grp | COS (Cost of Service)        | 0  0  0   0   0   |  | 0  0  0| 0  0  0 |  | 0 ...    0 |
|> Grp | GAC (General Admin Cost)     | 0  0  0   0   0   |  | 0  0  0| 0  0  0 |  | 0 ...    0 |
|> Grp | CPX (Capex & Depreciation)   | 0  0  0   0   0   |  | 0  0  0| 0  0  0 |  | 0 ...    0 |
|> Grp | PL2 (Margin Bef HR-Mkt)     | 0  0  0  21K 23K  |  | 0  0  0| 0  0  0 |  | 0 ... -25K |
|> Grp | PL3 (Margin Bef HR-Tech)    | 0  0  0   0   0   |  | 0  0  0| 0  0  0 |  | 0 ...    0 |
|> Grp | PL4 (Margin Bef HR-Partner) | 0  0  0   0   0   |  | 0  0  0| 0  0  0 |  | 0 ...    0 |
|> Grp | PL5 (Margin Aft HR-MktBonus)| 0  0  0   0   0   |  |11K 10K 26K| 0 0 0|  |+11K...   0 |
|> Grp | PL6 (Margin Aft HR-AllVent) | 0  0  0   0   0   |  | 4K  3K 19K| 0 0 0|  |+4K ... +19K|
|> Grp | PL7 (EBT - Earn Bef Tax)   | 0  0  0   0   0   |  | 0  0  0| 0  0  0 |  | 0 ...    0 |
+------+------------------------------+-------------------+--+--------+---------+--+---------+
|                            TM+ Financial Reporting (V3)                                    |
+============================================================================================+
```

#### V3 Layout Notes:
- **Header**: Title + RepTemp name; Gear icon (settings); V1/V2/V3 tabs (switch version)
- **Filter Row 1**: Template (RepTemp dropdown), Report Month, Actual Month, Unit (mVND/VND), USD conversion rate + button
- **Filter Row 2**: PLAN ZBlock (PCK/SCN/SRC/FF/RUN), FORECAST ZBlock (same 5 dropdowns), ALT/PT/CDT filters
- **Buttons**: [Load] = Load cached report; [Rebuild] = Re-build report from scratch
- **Table 3 sections mau**:
  - **PLAN** (header xanh duong / nen vang nhat): Hien thi 6 thang Plan data
  - **ACTUAL/FORECAST** (header cam + xanh la):
    - Actual (header cam / nen cam nhat): Thang co data thuc te (Oct25-Dec25)
    - Forecast (header xanh la / nen xanh nhat): Thang chua co actual, dung forecast (Jan26-Mar26)
  - **DELTA** (header xanh dam / nen xanh nhat): = Actual/Forecast - Plan
    - So duong: mau xanh, prefix "+" (vd: +11,324)
    - So am: mau do, prefix "-" (vd: -85,000)
- **Rows**: Moi row co CDT="Group" + icon ">" (expand/collapse drill-down)
- **KRF Metrics hien thi**: GI, CAC, COS, GAC, CPX, PL2, PL3, PL4, PL5, PL6, PL7 (11 dong Level 1-2)
- **Period logic**: 6 thang lien tiep; split Actual vs Forecast tai Actual Month boundary
  - Vd: Actual = Dec 2025 -> Oct25/Nov25/Dec25 = Actual, Jan26/Feb26/Mar26 = Forecast
- **Footer**: "TM+ Financial Reporting (V3)"

---

### 3.4 Config Management UI

```
+------------------------------------------------------+
|  CONFIG MANAGEMENT                                    |
+------------------+-----------------------------------+
|  [AllocALT]      |  ALT Code | Type | YNumber | Desc |
|  [AllocListType] +-----------------------------------+
|  [AllocToItem]   |  CFA1     | CFA  | 100     | ...  |
|  [RepTemp]       |  PLA4     | PLA  | 205     | ...  |
|  [RepTempBlock]  |  ...      | ...  | ...     | ...  |
|  [BlockToBlock]  |                                   |
|  [UserSec]       |  [ + Add ] [ Edit ] [ Delete ]    |
+------------------+-----------------------------------+
```

---

### 3.5 FPA Runner UI (tu PD200)

```
+------------------------------------------------------------------+
|  FPA RUNNER -- WORKFLOW                                            |
+------------------------------------------------------------------+
|  ZBlock: [v ACTUAL M2512 ]   Status: 2/5 buoc hoan thanh         |
+------------------------------------------------------------------+
|  ACTUAL FLOW                                                      |
|  Step | Ten                      | Deadline   | Status  | Action  |
|  110  | Upload RAW               | 2025-12-05 | Done    | [View]  |
|  120  | Import APR-SEC (HQ)      | 2025-12-07 | Done    | [View]  |
|  130  | APR Calculate            | 2025-12-08 | Pending | [Run]   |
|  149  | Upload SO-330            | 2025-12-09 | Pending | [Upload]|
|  150  | AllocationRun ACT        | 2025-12-10 | Locked  | [Run]   |
+------------------------------------------------------------------+
|  PLAN FLOW                                                        |
|  Step | Ten                      | Deadline   | Status  | Action  |
|  311  | PPR Input (R20-R100)     | 2026-01-05 | Pending | [Input] |
|  211  | MKT-X Input              | 2026-01-07 | Pending | [Input] |
|  220  | AllocationRun MKT-X      | 2026-01-08 | Locked  | [Run]   |
|  321  | Subscription Download    | 2026-01-09 | Pending | [DL]    |
|  340  | AllocationRun Plan All   | 2026-01-12 | Locked  | [Run]   |
+------------------------------------------------------------------+
|  [AllocationRunActual]  [AllocationRunPlan_MKT-X]  [Plan_All]    |
+------------------------------------------------------------------+
```

---

### 3.6 Drill-Down Expanded View

```
+---------------------+----------+--------------+---------+
| > GI                |  1,000   |  900         |  -100   |
|   v CDT1-Astronex   |    400   |  380         |   -20   |  << EXPANDED
|     | CDT2-MKT [+]  |    200   |  190         |   -10   |  << sub-row
|     | CDT2-PRD [+]  |    100   |   95         |    -5   |  << sub-row
|     | CDT2-OPS [+]  |    100   |   95         |    -5   |  << sub-row
|   > CDT1-Terasofts  |    600   |  520         |   -80   |  << COLLAPSED
+---------------------+----------+--------------+---------+

Drill-Down by SOCell (click vao 1 dong):
+---------------------+----------+--------------+---------+
|   > CDT2-MKT        |    200   |  190         |   -10   |
|     [From] CAC-MAC   |     80   |   85         |    +5   |  << SOCell From
|     [From] CAC-OMC   |     70   |   65         |    -5   |  << SOCell From
|     [From] Other     |     50   |   40         |   -10   |  << SOCell From
+---------------------+----------+--------------+---------+
```

---

## 4. User Flows

### 4.1 FPA Viewer -- Xem bao cao

```
[User] Truy cap /report
    |
    v
[Chon filters]
    RepTemplate ---- GET /api/rep-temp/list
    Plan ZBlock  ---- Cascade:
      PCK ----------- GET /api/zblock/pck?src=PLN
      SCN ----------- GET /api/zblock/scn?src=PLN&pck={pck}
      SRC ----------- GET /api/zblock/src?src=PLN&pck={pck}&scn={scn}
      FF ------------ GET /api/zblock/ff?...
      RUN ----------- GET /api/zblock/run?...
    Forecast ZBlock -- (tuong tu cascade)
    ALT -------------- GET /api/alt/list
    Last Report Month -- (input hoac default)
    Last Actual Month -- (input hoac default)
    PT Filter --------- GET /api/pt/allowed?user=xxx
    CDT Filter -------- GET /api/cdt/allowed?user=xxx
    |
    v
[Bam "Load Report"]
    POST /api/report/load
    { repTemp, zBlockPlan, zBlockForecast, alt,
      lastReportMonth, lastActualMonth }
    |
    +-- [RepPage da ton tai] ---- return RepCell array
    |
    +-- [RepPage chua co] ----> auto-trigger POST /api/report/build (internal)
                                        |
                                        return { status: "building", task_id: "abc-123" }
                                        |
                                        v
                               [polling GET /api/task/{task_id}] wait...
                                        -> { status: "pending" | "running" | "completed" }
                                        |
                                        v (khi completed)
                               return RepCell array
    |
    v
[Render bang bao cao]
Plan | ActualForecast | Delta columns
6 thang gan nhat (filter boi MyXPeriodLoadList)
    |
    v
[Click [+] tren 1 row -- Drill-Down by RepTemp]
    POST /api/drilldown/by-rep-temp
    { repCellId, username }
    -> BuildReport voi RepTemp moi (drilled)
    -> Insert drilled rows inline
    |
    v
[Click 1 dong -- Drill-Down by SOCell]
    POST /api/drilldown/by-socell
    { repCellId }
    -> GET SOCell(From) list via SOCell_SOCell
    -> Insert sub-rows inline (Collapse/Expand)
```

---

### 4.2 FPA Runner -- Upload va chay Allocation (tu PD200)

```
[Runner] Truy cap /runner
    |
    v
[Chon ZBlock (thang can chay)]
    |
    v
=== ACTUAL FLOW ===
[Step 110: Upload RAW]
    -> Nhan "Upload" tren tung file RAW
    -> POST /api/upload { file, step: "110", zblock }
    -> ActivityLog ghi nhan
    |
    v
[Step 120: Import APR-SEC]
    -> Bam "Import from RAW" trong file APR-SEC
    (HQ: FPA Runner bam; Non-HQ: tu bam)
    |
    v
[Step 130: APR Calculate -- tu dong link formula]
    |
    v
[Step 141-148: ACA Calculate]
    -> CFA1-ToLE, CFA2-ToCDT, CFA3-ToKRF
    -> PLA1-ToLE, PLA2-ToCDT, PLA3-ToKRF, PLA4-ToPT
    -> Calculate (PD221)
    -> POST /api/calculate -> ActivityLog
    |
    v
[Step 149: Upload SO-330]
    -> POST /api/upload/so330
    |
    v
[Step 150: AllocationRunActual]
    -> POST /api/allocation/run { minALT: 100, maxALT: 299, period: "M2512" }
    -> [polling status] -> Complete
    -> ActivityLog ghi nhan
    |
    v
=== PLAN FLOW ===
[Step 311: PPR Input R20-R100]
    -> Manual input HQ/DH/CEH
    |
    v
[Steps 211-213: MKT-X Planning]
    -> Input -> Calculate -> Upload
    |
    v
[Step 220: AllocationRunPlan_MKT-X]
    -> POST /api/allocation/run { minALT: 300, maxALT: 399 }
    |
    v
[Steps 321-323: Subscription Planning]
    -> Download SI-110 -> Calculate -> Upload BU
    |
    v
[Steps 331-333: PLA4-ToPT Planning]
    -> Download SI-110 -> Calculate -> Upload
    |
    v
[Step 340: AllocationRunPlan_All]
    -> POST /api/allocation/run { minALT: 400, maxALT: 499 }
    |
    v
[Step 350: Download PPR IMP]
```

---

### 4.3 BOD / Manager -- Xem bao cao cap cao

```
[BOD/Manager] Truy cap /report
    |
    v
[RepTemp duoc pre-select theo role]
    BOD -> KRF-L4.CDT0 (tong hop, khong drill sau)
    MGT -> KRF-L4.CDT1 hoac CDT2 (theo CDT-allowed)
    |
    v
[Load Report voi default ZBlock (latest build)]
    -> Security filter tu dong ap theo username
    -> Chi hien thi CDT/PT trong allowed list
    |
    v
[Xem bang Plan vs Actual/Forecast vs Delta]
    [Drill-Down neu co quyen]
```

---

### 4.4 Admin -- Config Management

```
[Admin/FCOO] Truy cap /config
    |
    v
[Tab: AllocALT]
    -> CRUD AllocationALT records
    -> Xac dinh YNumber mapping
    |
    v
[Tab: RepTemp]
    -> UploadRepTemp() -- tu PD260
    -> CRUD RepTemp records
    |
    v
[Tab: RepTempBlock]
    -> UploadRepTempBlock() -- tu PD260
    -> CRUD RepTempBlock records
    -> Dinh nghia KRTypeFull va FilterTypeFull
    |
    v
[Tab: UserSec]
    -> Quan ly CDT/PT-allowed per user
    -> Gan sec_level
```

---

## 5. Complex Logic

### 5.1 BuildFilterItemList -- Xay dung danh sach filter items da tang

**Muc dich**: Tu `FilterTypeFull` (vd: "CDT-S1-50_PT-S2-40_LE-20") -> sinh ra bang to hop tat ca filter items (cartesian product co dieu kien)

```
Input: FilterTypeFull = "CDT-S1-50_PT-S2-40_LE-20"
  -> parse:
     [0] = { type: "CDT-S1", maxRows: 50 }
     [1] = { type: "PT-S2",  maxRows: 40 }
     [2] = { type: "LE",     maxRows: 20 }

MaxNumber = 3 (so tang)
Multiple   = 1,000,000,000,000 (ban dau)

Parse rule:
  - Tach cac FilterType bang "_" separator
  - Bo phan -Number cuoi (CDT-S1-50 --> CDT-S1, maxRows=50)

Algorithm (iterative, toi da 5 tang):

BuildFilterItemList(FilterTypeList, CurrentTypeNumber, MaxTypeNumber,
                     Multiple, MomFilterItem, MomYNumber):

  items = query AllocationToItem
    WHERE TO_Y_BLOCK_ToType = FilterTypeList[CurrentTypeNumber].type

  ForEach item IN items (KidYNumber = YNumber increasing):

    // SubFilter check:
    IF FilterType.Main = "PT" AND IsSub(MainToItem, MyPT):
        include
    ELIF FilterType.Main = "CDT" AND IsSub(MainToItem, MyCDT):
        include
    ELIF FilterType.Main <> "PT" AND FilterType.Main <> "CDT":
        include  // cac filter khac lay het
    ELSE:
        skip

    KidYNumber3 = MomYNumber * Multiple + KidYNumber * (Multiple / 1000)
    KidFilterBlock = Merge(MomFilterItem, item)

    Write to RepFilterItem Table: { KidYNumber3, KidFilterBlock }

    IF CurrentTypeNumber < MaxTypeNumber:
        Call BuildFilterItemList(
          FilterTypeList, CurrentTypeNumber+1, MaxTypeNumber,
          Multiple/1000, KidFilterItem, KidYNumber)

Output: RepFilterItem temp table co structure YNumber ma hoa vi tri trong tree
```

**Vi du YNumber3 structure** (CDT=20 items, PT=12 items, LE=5 items):
```
YNumber3 cua CDT[i] x PT[j] x LE[k]
  = i x 1,000,000 + j x 1,000 + k x 1
  Tong so records = 20 x 12 x 5 = 1,200 filter items
```

> **Ghi chu**: PD260 row 101 co "// Comment Out" -- can xac nhan dang dung iterative hay recursive. De xuat: iterative voi nested loop (max 5 tang), fallback recursive neu can.

---

### 5.2 YNumber1 Composite Key

```
YNumber1 = ZBlockPlan.YNumber   x 1,000,000
         + ZBlockForecast.YNumber x 1,000
         + ALT.YNumber

Rang buoc:
  - ZBlockPlan.YNumber: 1..999 (toi da 999 Plan runs)
  - ZBlockForecast.YNumber: 1..999 (toi da 999 Forecast runs)
  - ALT.YNumber: 100..299 (CFA) / 200..299 (PLA)

Vi du:
  ZBlockPlan.YNumber = 5 (PLN-CA-Future-CAC)
  ZBlockForecast.YNumber = 3 (FRC-CA-Future-CAC)
  ALT.YNumber = 205 (PLA4)
  -> YNumber1 = 5,000,000 + 3,000 + 205 = 5,003,205

Query YNumber tu BigQuery (PD260 row 21):
  #ZBlockPlan.YNumber:
  SELECT YNumber FROM ZBlock1_NativeTable
  WHERE Z_BLOCK_ZBlockPlan_Source = 'PLAN'
  AND Z_BLOCK_ZBlockPlan_Pack = 'CA'
  AND Z_BLOCK_ZBlockPlan_Scenario = 'Future'
  AND Z_BLOCK_ZBlockPlan_Run = 'CAC'

  #ALT.YNumber:
  SELECT YNumber FROM AllocationALT_NativeTable
  WHERE ALT_code = 'PLA4'
```

---

### 5.3 Delta Calculation

```
DeltaValue = ActualForecast.NOW_VALUE - Plan.NOW_VALUE

Logic chon ActualForecast:
  IF LastActualMonth >= MyXPeriod:
    ActualForecast = SOCell2 (ZBlock scenario = "Actual")
  ELSE:
    ActualForecast = SOCell3 (ZBlock = MyZBlockForecast)

  // Delta = ActualForecast - Plan

NULL handling:
  IF SOCell1 (Plan) IS NULL OR SOCell2/3 (AF) IS NULL:
    DeltaValue = NULL

Write RepCell (Delta):
  Copy all KR fields from SOCell (FNF, KR1-KR8, KR_Item_Name)
  Copy all Filter fields from MyFilterItem (CDT1-4, PT, Funnel, ...)
```

---

### 5.4 Security Filter trong LoadReport

```
Khi LoadReport tra ve RepCell array:

SEC Level 1 (CDT/PT filter):
  -> Lay CDT_allowed list tu UserSecConfig WHERE username = xxx
  -> Lay PT_allowed list tu UserSecConfig WHERE username = xxx
  -> Filter: RepCell.NOW_Y_BLOCK_CDT_CDT4 IN CDT_allowed
       AND RepCell.NOW_Y_BLOCK_PTNow_PT2 IN PT_allowed
  (chi loc neu allowed list khong rong)

SEC Level 2 (Row-level):
  -> An hoan toan cac dong khong thuoc CDT/PT allowed

SEC Level 3+ (nang cap tuong lai):
  -> Row-level security trong BigQuery hoac API layer

GI-OMC business units: Filter theo TD/BU
Other: All
```

---

### 5.5 Period Arithmetic

```
MyXPeriod = MyLastReportMonth - L   (L = 0..119)

Period format: M{YY}{MM}
  - YY = year (25 = 2025, 26 = 2026)
  - MM = 01..12

Arithmetic:
  Subtract(M2512, 0) = M2512
  Subtract(M2512, 1) = M2511
  Subtract(M2512, 5) = M2507
  Subtract(M2501, 1) = M2412  // cross-year rollover

BuildReport: 120 months (L=0..119) = tu M2501 den M3012
LoadReport: 6 months (L=0..5) = tu M2507 den M2512
```

---

### 5.6 Cascade Dropdown Logic (V2 UI)

```
Frontend cascade cho Plan ZBlock:
  Step 1: User chon SRC = "PLN"
    -> GET /api/zblock/pck?src=PLN -> PCKList
  Step 2: User chon PCK
    -> GET /api/zblock/scn?src=PLN&pck={pck} -> SCNList
  Step 3: User chon SCN
    -> GET /api/zblock/src?src=PLN&pck={pck}&scn={scn} -> SRCList
  Step 4: User chon SRC detail
    -> GET /api/zblock/ff?... -> FFList
  Step 5: User chon FF
    -> GET /api/zblock/run?... -> RUNList
  Step 6: User chon RUN
    -> MyZBlockPlan = composite {SRC}-{PCK}-{SCN}-{SRC}-{FF}-{RUN}

Tuong tu cho Forecast ZBlock.
MyZBlock = (MyRepTemp, MyZBlockPlan, MyZBlockForecast, MyALT) -> MyRepPage
```

---

### 5.7 FilterTypeFull Parse Rule (tu PD260 row 70)

```
Input: FilterTypeFull = "CDT-S1-50_PT-S2-40_CTY-S2-20"

Parse steps:
  1. Tach bang "_": ["CDT-S1-50", "PT-S2-40", "CTY-S2-20"]
  2. Voi moi phan tu:
     a. Tach bang "-": ["CDT", "S1", "50"]
     b. Segment cuoi la so nguyen -> maxRows = 50
     c. Ghep lai bo so: "CDT-S1" -> typeCode
  3. Output: [{typeCode: "CDT-S1", maxRows: 50},
              {typeCode: "PT-S2",  maxRows: 40},
              {typeCode: "CTY-S2", maxRows: 20}]

De xuat: Tao ham parseFilterTypeFull() chuan hoa.

MyFilterType lay tu cac cot trong RepTempBlock (PD260 row 70):
  NOW-Y-BLOCK_CDT_CDT1..CDT4
  NOW-Y-BLOCK_PTNow_PT1..PT2, Duration
  NOW-Y-BLOCK_PTPrev_PT1..PT2, Duration
  NOW-Y-BLOCK_PTFix_OwnType, AIType
  NOW-Y-BLOCK_PTSub_CTY1..CTY2, OSType
  NOW-Y-BLOCK_Funnel_FU1..FU2
  NOW-Y-BLOCK_Channel_CH
  NOW-Y-BLOCK_Employee_EGT1..EGT4
  NOW-Y-BLOCK_HR_HR1..HR3
  NOW-Y-BLOCK_SEC
  NOW-Y-BLOCK_Period_MX, DX, PPC, NP
  NOW-Y-BLOCK_LE_LE1..LE2
  NOW-Y-BLOCK_UNIT
  NOW-Y-BLOCK_TD_BU
```

---

### 5.8 Drill-Down NoOfRows Parsing (tu PD260)

```
Input: KRTypeCode = "GI-CAC-MAC-3"
       FilterTypeCode = "CDT-S1-50_PT-S2-40"

Parse steps (khu NoOfRows):
  1. Cat cac FilterType bang "_"
  2. Cat ra cac Segment bang "-"
  3. Neu moi Segment toan SO thi bo di (NoOfRows)
     - "CDT-S1-50" -> bo "50" -> "CDT-S1"
  4. So o cuoi Segment thu 2 chinh la Level
     - "CDT-S1" -> Level = 1
     - "PT-S2" -> Level = 2
  5. Chi giu lai Segment dau tien: MyFilterType (Main)
     - "CDT-S1" -> Main = "CDT"

Output:
  MyFilterType = "CDT", MyFilterLevel = 1
  -> Dung de query RepTempBlockToBlock
```

---

## 6. Goi y hoan thien System Design

### 6.1 Entity / Data Model

| # | Goi y | Muc do | Chi tiet |
|---|-------|--------|----------|
| S1 | **Tao bang AllocationALT** | QUAN TRONG | PD260 row 21 de cap nhung chua co schema. Xem [E2]. Can dong bo ALT code voi AllocationRun. |
| S2 | **Tao bang AllocationListType** | BO SUNG | PD510 dinh nghia he thong phan loai filters. Xem [E6]. Giup quan ly taxonomy filter tap trung. |
| S3 | **Them cot vao RepPage** | BO SUNG | Them `last_built_period` de ho tro incremental build. Them `build_duration_seconds` de monitoring. |
| S4 | **Them cot vao RepCell** | BO SUNG | Them `delta_percent` (= Delta/Plan * 100) de UI khong phai tinh lai. |
| S5 | **AllocationToItem bo sung ParentType/ParentItem** | QUAN TRONG | Can cho IsSub() check trong BuildFilterItemList. Xac nhan da co hay chua. |

### 6.2 Logic / Process Synchronization

| # | Goi y | Muc do | Chi tiet |
|---|-------|--------|----------|
| S6 | **Dong bo ALT range giua PD200 va PD260** | QUAN TRONG | PD200: ACT=ALT100-299, MKT-X=ALT300-399, PlanAll=ALT400-499. PD260 dung ALT code (PLA4, CFA1). Can mapping ro rang. |
| S7 | **Dong bo FilterTypeFull format** | QUAN TRONG | PD260 row 70 dung "-" lam separator (CDT-PT-LE), nhung vi du row 100 lai dung format khac (CDT-S1-50). Can chuan hoa. |
| S8 | **Xac nhan IsSub() function** | QUAN TRONG | PD260 row 101 su dung nhung chua dinh nghia logic cu the. Can implement: IsSub(childItem, parentItem) -> boolean. |
| S9 | **SOCell_SOCell populate** | QUAN TRONG | PD260 V5.2 de cap "Sua code de ghi bang". Can xac nhan trang thai implement va test. |
| S10 | **BuildReport rebuild trigger** | TRUNG BINH | Khi nao can rebuild? Hien tai: LatestBuildTime=NULL hoac manual. De xuat: them auto-invalidate khi SOCell data thay doi. |

### 6.3 Performance & Scalability

| # | Goi y | Muc do | Chi tiet |
|---|-------|--------|----------|
| S11 | **RepCell partitioning** | QUAN TRONG | BuildReport tao: FilterItems x 120 periods x 3 ZBlockTypes records. VD: 40x100x120x3 = 1.44M records/RepPage. Can partition theo FK_rep_page_id + NOW_NP. |
| S12 | **Incremental Build** | TRUNG BINH | Hien tai rebuild toan bo. De xuat: luu last_built_period, chi build thang moi. |
| S13 | **SOCell query optimization** | QUAN TRONG | Moi cell cua BuildReport query SOCell 3 lan (Plan/Actual/Forecast). Voi 1,200 filter items x 120 periods = 432,000 queries. De xuat: batch query, pre-aggregate, hoac materialized view. |
| S14 | **BigQuery clustering** | TRUNG BINH | Cluster SOCell theo (z_block_zblock1_source, now_zblock2_alt, now_np). Cluster RepCell theo (YNumber1, YNumber2, NOW_NP). |
| S15 | **RepFilterItem persistence** | THAP | Hien la temp table / memory array. Neu filter items co dinh theo RepTempBlock, luu persistent de khong rebuild moi lan. |

### 6.4 API Endpoints

#### 6.4.1 DA IMPLEMENTED (4 endpoints)

| API | Method | Mo ta | Ghi chu |
|-----|--------|-------|---------|
| `/health` | GET | Health check -- kiem tra API co dang hoat dong khong | Tra ve status OK |
| `/api/report/build` | POST | Day task tao bao cao tai chinh vao hang doi xu ly nen | Tra ve `task_id`; async background job |
| `/api/task/{task_id}` | GET | Theo doi trang thai task dang chay nen | Generic task tracker, dung cho build va cac task khac |
| `/api/report/load` | POST | Diem vao chinh: tra report neu da co, neu chua thi tu dong trigger build va tra `task_id` | Smart entry point: cache-first, auto-build |

#### Flow giua cac API da implemented:

```
[Frontend]
    |
    |--(1)-- POST /api/report/load { repTemp, zBlockPlan, zBlockForecast, alt, ... }
    |            |
    |            +-- [RepPage DA CO, LatestBuildTime != NULL]
    |            |       -> return { status: "ready", data: RepCell[] }
    |            |
    |            +-- [RepPage CHUA CO hoac can rebuild]
    |                    -> tu dong goi POST /api/report/build (internal)
    |                    -> return { status: "building", task_id: "abc-123" }
    |
    |--(2)-- GET /api/task/{task_id}    (polling cho den khi done)
    |            -> return { status: "pending" | "running" | "completed" | "failed",
    |                        progress: 0-100,
    |                        result: RepCell[] (khi completed) }
    |
    |--(3)-- GET /health                (optional: kiem tra truoc khi goi API)
    |            -> return { status: "ok" }
```

#### 6.4.2 CAN TAO THEM (planned endpoints)

| API | Method | Mo ta | Muc do |
|-----|--------|-------|--------|
| `/api/rep-temp/list` | GET | Lay danh sach RepTemp | CAN THIET |
| `/api/zblock/pck` | GET | Cascade: lay PCK list theo SRC | CAN THIET |
| `/api/zblock/scn` | GET | Cascade: lay SCN list | CAN THIET |
| `/api/zblock/src` | GET | Cascade: lay SRC detail list | CAN THIET |
| `/api/zblock/ff` | GET | Cascade: lay FF list | CAN THIET |
| `/api/zblock/run` | GET | Cascade: lay RUN list | CAN THIET |
| `/api/alt/list` | GET | Lay danh sach AllocationALT | CAN THIET |
| `/api/pt/allowed` | GET | Lay PT-allowed list theo user | CAN THIET |
| `/api/cdt/allowed` | GET | Lay CDT-allowed list theo user | CAN THIET |
| `/api/drilldown/by-rep-temp` | POST | Drill-Down by RepTemp hierarchy | TRUNG BINH |
| `/api/drilldown/by-socell` | POST | Drill-Down by SOCell relationship | TRUNG BINH |
| `/api/allocation/run` | POST | AllocationRun (minALT, maxALT) | TRUNG BINH |
| `/api/upload/so330` | POST | Upload data to SOCell | TRUNG BINH |
| `/api/validate/socell` | POST | Validate SOCell data quality | THAP |
| `/api/plan-events/status` | GET | Runner workflow status | THAP |
| `/api/activity-log/list` | GET | Audit log viewer | THAP |

### 6.5 Security & Access Control

| # | Goi y | Muc do | Chi tiet |
|---|-------|--------|----------|
| S16 | **Mapping SEC level -> role** | QUAN TRONG | SEC1-SEC4 chua map ro voi role (FCOO/BOD/MGT/MKT). Can tai lieu hoa day du. |
| S17 | **BigQuery Row Access Policy** | TRUNG BINH | De xuat: ap dung BigQuery Row Access Policy cho SOCell, RepCell thay vi filter o API layer. |
| S18 | **HR data bao mat** | QUAN TRONG | PD260 de cap: HR data can security rieng (nguoi cu - luong cu the, nguoi moi - CPEM theo EGT4). |
| S19 | **Audit trail cho config changes** | THAP | Moi thay doi RepTemp, RepTempBlock, UserSecConfig can ghi ActivityLog. |

### 6.6 UX Improvements

| # | Goi y | Chi tiet |
|---|-------|----------|
| S20 | **Real-time Build Progress** | BuildReport co the mat vai phut. Can WebSocket hoac polling endpoint voi progress bar. |
| S21 | **Default values cho filters** | Last Report Month / Last Actual Month nen co default thong minh (thang hien tai -1). |
| S22 | **Export bao cao** | Ho tro export RepCell ra Excel/CSV de BOD review offline. |
| S23 | **Bookmark/Save report config** | Cho phep user save bo filter da chon de nhanh chong load lai. |

---

## 7. Diem khong dong bo can xac nhan

| # | Van de | File nguon | De xuat |
|---|--------|-----------|---------|
| **Q1** | **AllocationALT.YNumber mapping** chua co bang chinh thuc. PD260 row 21 ghi "CFA: 1xx, PLA: 2xx" nhung chua xac nhan gia tri cu the cho tung ALT code (CFA1T, PLA1T, ...). | PD260 row 21, PD510 | Xac nhan va tao bang AllocationALT theo schema [E2] |
| **Q2** | **BuildFilterItemList**: Co doan "// Comment Out" o ham de quy. Thuc te dang dung iterative hay recursive? PD260 ghi "[Thu theo cach foreach truoc, max 5 tang, khong duoc thi de quy]" | PD260 row 101, 206 | Xac nhan approach hien tai |
| **Q3** | **MyLastReportMonth = "M3012"** o row 30A cua PD260 -- day la gia tri mac dinh (build full 10 nam) hay loi typo? (So sanh voi "M2512" o cac cho khac) | PD260 row 30A | Xac nhan: M3012 la default cho BuildReport (full range) con M2512 la vi du LoadReport? |
| **Q4** | **SOCell_SOCell** hien chua ro cach populate trong AllocationRun. PD260 V5.2 ghi "Sua code de ghi bang SOCell_SOCell" -- da implement chua? | PD260 V5.2 | Xac nhan status cua feature nay |
| **Q5** | **FilterTypeFull separator**: PD260 row 70 vi du "CDT-PT-LE" (dung "-"), nhung cac vi du khac dung format "CDT-S1-50_PT-S2-40" (dung "_" va "-"). Cau hoi: "-" separator giua types hay giua type va level? | PD260 row 70, 100 | Chuan hoa: dung "_" giua types, "-" giua type components |
| **Q6** | **Security SEC1-SEC4 mapping**: PD200 noi 3 levels, PD260 dung SEC1-SEC4. Mapping cu the la gi? | PD200, PD260 | Can mapping day du SEC -> role -> permission |
| **Q7** | **LoadReport period count**: Hien load 6 thang (M-0 den M-5). Co configurable khong? BuildReport thi tinh 120 thang. | PD260 row 21B, 130 | Xac nhan: co dinh 6 hay cho phep user chon? |
| **Q8** | **YNumber1 overflow**: Formula dung range 1K cho ZBlockForecast.YNumber. Neu >999 forecast runs thi overflow. | PD260 row 21 | Xac nhan gioi han so luong ZBlock runs thuc te |
| **Q9** | **IsSub() function**: Dung trong BuildFilterItemList nhung chua dinh nghia. Can ParentType/ParentItem trong AllocationToItem? | PD260 row 214-216 | Dinh nghia va implement IsSub() |
| **Q10** | **Drill-Down ALT compatibility**: PD260 Drill-Down table chi ra cac ALT type (CFA1-3, PLA1-5) ho tro drill-down khac nhau (RepTemp vs SOCellToCell vs SOCellToCell Prev=APR). Can xac nhan logic chinh xac. | PD260 Drill-Down section | Xac nhan matrix ALT x DrillDown type |

---

| **Q11** | **PD220 PC_1 BAC-[L7-50] formula**: `= HRC + NHRC` -- "[L7-50]" co phai la placeholder cho 50 row expansion o Level 7 khong? | PD220 PC_1 line 16 | Xac nhan y nghia cua "[L7-50]" notation |
| **Q12** | **PD220 CDT aggregation consistency**: PC_2 dung CDT-CS2 cho COS nhung CDT-CS3 cho CAC. Day la co y (khac level) hay loi? | PD220 PC_2 | Xac nhan CDT level cho tung cost category |
| **Q13** | **PD220 MKT-X row count**: PC_3 OMC section co toi da 20,400 rows (40 CDT x 30 PT x 17 CTY). Day co phai la gioi han thuc te? | PD220 PC_3 line 188 | Xac nhan max row count va performance impact |
| **Q14** | **PD220 RP vs PC**: RP sheet co cung formula nhu PC hay chi la output/display? Can lam ro moi quan he giua RP va PC sheets. | PD220 RP | Xac nhan: RP chi doc data tu PC hay co logic rieng? |

---

*Tai lieu nay duoc tong hop tu: PD260 (Report/Drilldown), PD510 (All Filters Dimensions), PD530 (KRF Hierarchy), PD200 (Runner Guide), PD220 (Plan Calculate PC_1/PC_2/PC_3/RP). Version 3.0 bo sung: Plan Calculate process (PD220), full KRF hierarchy tree, P&L waterfall formulas, CDT aggregation logic, BYKR ratios, MKT-X planning metrics, GI calculation chain, Sheet I/O flow.*
