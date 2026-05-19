# CHP Screening Tool v2.0 - Design Notes

**Status:** Approved design, ready to build
**Source conversation:** chp-screening-spec__1_.md (v1.1) + CHP_Screening_Tool_Demo_Changes.docx
**Calculation reference:** CHP_Calculations_forWebDemo.xlsx
**Data tables:** chp-screening-data.js (seeded)

---

## 1. Architectural decisions

### Two-phase flow
- **Phase 1:** 7 screening questions → preliminary fit tier
- **Phase 2:** Results page with adjustable inputs (rates, consumption, runtime, utilization) → refined savings

Phase 2 is not a separate screen. It's the savings panel on the results page. User sees a savings number immediately based on state defaults and facility-type defaults, then can refine.

### Product routing
- E200 (180 kW, available) is the default model
- E100 (100 kW, planned 2026) added to PRODUCTS now, status `planned_2026`
- Routing logic respects status. Until E100 flips to `available`, all qualified visitors model E200
- When annual kWh is provided in Phase 2, base load is computed: `annual_kwh / run_hours`. Used to message "an E100 may be a closer match" but doesn't change the modeled product in v2

### Bill caps
Applied in dollars on both legs:
- `electric_savings = MIN(usable_kwh * elec_rate, current_annual_electric_bill)`
- `heat_savings = MIN(displaced_therms * fuel_rate, current_annual_fuel_bill)`

This is mathematically equivalent to the v1.1 kWh-side cap on the electric leg but more intuitive and consistent across legs.

### Operating point: not adjustable
Fixed at 180 kW for E200, 100 kW for E100. Load-follow handles below that range in real operation. Electrical utilization knob (50-100%, default 100%) captures partial-load operation over the year.

### Savings demotion
If modeled net savings < $20,000/year, demote tier from Strong to Potential. This handles the case where structural fit signals look good but the local spark spread doesn't support attractive economics.

### Under-$5K bill: no longer disqualifies
With E100 in the catalog, small sites are watch-list candidates. Result-page copy routes them to "E100 launch announcement list" until E100 ships.

---

## 2. Question flow (Phase 1)

| # | ID | Question | Purpose | Input type |
|---|---|---|---|---|
| 1 | q1_gas | Natural gas availability | Go/no-go disqualifier first | radio (4 options) |
| 2 | q2_state | Facility state | Drives EIA rate defaults | dropdown (51 states) |
| 3 | q3_facility | Facility type | Drives run-hours, thermal-util, vertical score | radio (10 types) |
| 4 | q4_elec_bill | Monthly electric bill | Fit signal + dollar cap on elec savings | dropdown, $1K-$100K in $1K + $100K+ + not sure (102 options) |
| 5 | q5_fuel_bill | Monthly fuel bill | Fit signal + dollar cap on heat savings | dropdown, $500-$20K in $500 + $20K+ + not sure (42 options) |
| 6 | q6_thermal | Year-round heat/hot water | Thermal demand modifier | radio (5 options) |

Project timing moved to lead form (sales/qualification field, not fit field).

**v2.1 bill answer storage:** The bill answer is the raw monthly $ value as a string (e.g., `'45000'`) or a sentinel (`'over_100k'`, `'over_20k'`, `'not_sure'`). Score banding uses `billPoints()`. Savings caps use the monthly value × 12. See section 4a for product selection based on bill.

Dropped from v1.1:
- Q3 operating hours (replaced by facility-type default + Phase 2 override)
- Q5 heat-uses multi-select (replaced by facility-type default + Phase 2 override)
- Q7 reason-for-interest (moved to lead form)
- Q8 project timing (moved to lead form, per Dustin's instruction)

---

## 3. Phase 2 adjustable inputs (results page)

All shown with defaults derived from Phase 1 answers. All validated.

| Input | Default source | Validation |
|---|---|---|
| Annual electricity consumption (kWh) | Optional, blank by default | Integer, 0 to 100,000,000 |
| Electric blended rate ($/kWh) | RATES_BY_STATE[state].elecRate | Currency, 0.01 to 1.00 |
| Natural gas rate ($/therm) | RATES_BY_STATE[state].gasRate | Currency, 0.10 to 5.00 |
| Boiler efficiency (%) | 80 | Integer, 50 to 98 |
| Annual runtime (hours) | FACILITY_TYPES[type].runHours | Integer, 500 to 8700 |
| Electrical utilization (%) | 100 | Integer, 50 to 100 |
| Thermal utilization (%) | facility_default × thermal_modifier | Integer, 0 to 100 |

Operating point is **not** adjustable. Fixed at 180 kW for E200 / 100 kW for E100.

---

## 4. Calculation engine

```
chp_kw_output        = product.outputKW                  // 180 (E|200) or 100 (E|100)
chp_heat_btu_hr      = product.heatBtuHr
chp_fuel_therms_hr   = product.fuelThermsPerHr

annual_kwh_gross     = chp_kw_output × run_hours
usable_kwh           = annual_kwh_gross × electrical_utilization

// Load-follow operating mode: fuel and heat output scale with electric output.
// When the engine dials back to match site load, fuel input and heat both
// drop proportionally. This is the E-series operating behavior.
annual_fuel_therms   = chp_fuel_therms_hr × run_hours × electrical_utilization
annual_heat_btu      = chp_heat_btu_hr × run_hours × electrical_utilization × thermal_utilization
boiler_therms_disp   = annual_heat_btu / 100,000 / boiler_efficiency

raw_elec_savings     = usable_kwh × elec_rate
elec_savings         = MIN(raw_elec_savings, current_annual_electric_bill)

raw_heat_savings     = boiler_therms_disp × fuel_rate
heat_savings         = MIN(raw_heat_savings, current_annual_fuel_bill)

chp_fuel_cost        = annual_fuel_therms × fuel_rate
om_cost              = usable_kwh × om_per_kwh                    // $0.025/kWh

net_annual_savings   = elec_savings + heat_savings - chp_fuel_cost - om_cost

low  = MAX(0, net × 0.75)
mid  = MAX(0, net)
high = MAX(0, net × 1.25)
```

Where `current_annual_electric_bill = billValueToMonthly(answer, 'elec') × 12`, same for fuel.

**Load-follow scaling note (v2.1):** Previous versions treated CHP fuel consumption as fixed at full rated input regardless of electric utilization. This produced unrealistic $0 results for undersized sites where the bill cap bound tight on electric savings while full fuel was still charged. The E-series uses load-follow, meaning the engine dials back when site load is below rated output. Fuel and heat output now scale with electric utilization, matching real operation. This is a divergence from the original Excel model and should be verified against engineering-team assumptions before launch.

**Default electric utilization (v2.1):** The questionnaire-default `elec_util` is now derived from estimated annual kWh divided by gross CHP capacity, clamped to [0.5, 1.0]. Estimated kWh comes from `monthly_bill × 12 / state_elec_rate`. When the user picks "Not sure" for the bill, default utilization stays at 100%. User can override on the results page.

---

## 4a. Product selection (E|200 vs E|100)

```
baseLoad = annual_kwh_estimate / run_hours

if baseLoad >= 150 kW → E|200 (180 kW operating, good utilization)
else                  → E|100 (100 kW operating, better match for smaller sites)
```

`annual_kwh_estimate` priority:
1. User-provided annual kWh on results page (highest accuracy)
2. Monthly bill × 12 / state-average elec rate (fallback)
3. If bill is "Not sure", default to E|200

Effect by market:
- High-rate state (CA at $0.247/kWh): cutoff bill ≈ $27K/month
- Low-rate state (TX at $0.089/kWh): cutoff bill ≈ $10K/month

Same base load, different bills because rates differ. This is the right behavior — the product fit is driven by load, not by bill size alone.

---

## 4b. Multi-unit disclaimer

For sites where `monthly_bill ≥ $40,000` and selected product is E|200, the results page appends a note explaining that multi-unit configurations may add proportional savings and that Application Engineering will size the optimal unit count. The screening tool models a single unit only.

---

## 5. Scoring algorithm

Two independent scores. The fit tier is what the user sees. The lead score is internal-only and used by Inside Sales for prioritization in HubSpot.

### 5a. Fit tier (user-facing, technical fit only)

```
Hard rules:
  q1_gas = 'no' → tier='notfit', reason='no_gas'

Soft signals (additive, max 15):
  + GAS_AVAILABILITY[q1_gas].points         (0-3)
  + FACILITY_TYPES[q3_facility].verticalScore (0-3)
  + billPoints(q4_elec_bill, 'elec')          (0-3, see ELEC_BILL_BANDS)
  + billPoints(q5_fuel_bill, 'fuel')          (0-3, see FUEL_BILL_BANDS)
  + Thermal points:
      yes=3, mostly=2, seasonal=-1, limited=-2, dont_know=0

Tier from raw score:
  >= 13 → strong       (87% of max)
  >= 9  → potential
  >= 4  → review
  < 4   → notfit

Savings-floor adjustment (applied after Phase 2 savings calc):
  If tier = 'strong' AND mid_savings < $20,000 → demote to 'potential'
```

Bill scoring uses `billPoints()` which maps the granular monthly $ value to a band:
- Electric: <$5K→0, $5K-$15K→2, $15K-$40K→3, $40K-$100K→2, >$100K→1, not_sure→0
- Gas: <$1K→0, $1K-$3K→1, $3K-$8K→2, $8K-$20K→3, >$20K→2, not_sure→0

Timing, financing preference, and evaluator role are deliberately excluded from fit. They reflect commercial readiness, not whether the equipment makes technical sense at the facility.

### 5b. Lead score (internal only, commercial readiness)

Computed at submission, sent to HubSpot, never displayed to the user.

```
Soft signals (additive, max 13):
  + LEAD_SCORE.timing[project_timing]       (0-4)
      exploring=0, no_timeline=0, 12_24=2, 0_12=4
  + LEAD_SCORE.financing[financing_pref]    (0-3)
      direct=3, lease=2, ppa=2, eaas=2, open=1
  + LEAD_SCORE.role[evaluator_role]         (0-3)
      end_user=3, esco_consultant=2, developer=2, investor=1
  + reasons_count × 0.5, capped at 3

Priority:
  >= 10 → hot
  >= 6  → warm
  < 6   → cold
```

Sales can sort HubSpot by `chp_screening_tier` (technical fit), by `lead_priority` (sales readiness), or by both. A "Strong + Hot" lead is the highest-value combination.

---

## 6. UI changes from v1.1

| Change | Implementation |
|---|---|
| Back button on every screen | Standard. Disabled on screen 1. |
| Restart button on every screen | Immediate reset, clears DOM and state (no confirm dialog, v2.1) |
| Progress indicator updated for 6-question count | "Question N of 6" |
| Lead form gains: timing, reason-for-interest, financing-preference, evaluator-role | New fields in form-grid |
| Savings card: inputs always visible above results (v2.1) | Pre-filled calculator look, no toggle |
| Bill questions: granular dropdowns instead of bands (v2.1) | $1K-$100K elec, $500-$20K gas |
| Submit decision: send defaults vs send edits | Modal before form submission if any Phase 2 field was edited |

---

## 7. Lead form fields (final)

Existing (keep):
- Full name, Company, Email, Phone, City, State (auto-filled from Q2), Facility type (auto-filled from Q3), Project description, Preferred follow-up, Consent checkbox

New:
- **Project timing** (single-select, moved from screening)
- **Reason for interest** (multi-select, was a screening question in v1.1)
- **Financing preference** (single-select: Direct / PPA / EaaS / Open. Lease removed in v2.1)
- **Evaluator role** (single-select: End user / Developer / ESCO / Investor)

---

## 8. HubSpot payload (final)

```javascript
{
  // Contact identification
  name, company, email, phone, city, state,

  // Screening tier
  chp_screening_tier,           // strong | potential | review | notfit
  chp_screening_reason_code,    // no_gas | below_floor | etc., or null
  chp_screening_score,
  chp_screening_raw_answers,    // JSON of all Q1-Q7

  // Phase 2 questionnaire defaults (what the tool calculated initially)
  chp_questionnaire_run_hours,
  chp_questionnaire_elec_util,
  chp_questionnaire_thermal_util,
  chp_questionnaire_elec_rate,
  chp_questionnaire_gas_rate,
  chp_questionnaire_boiler_eff,
  chp_questionnaire_savings_low,
  chp_questionnaire_savings_mid,
  chp_questionnaire_savings_high,

  // Phase 2 user-edited values (what the user submitted)
  chp_submitted_run_hours,
  chp_submitted_elec_util,
  chp_submitted_thermal_util,
  chp_submitted_elec_rate,
  chp_submitted_gas_rate,
  chp_submitted_boiler_eff,
  chp_submitted_annual_kwh,     // null unless user provided
  chp_submitted_savings_low,
  chp_submitted_savings_mid,
  chp_submitted_savings_high,

  // Which set did the user choose to send?
  chp_submission_basis,         // 'questionnaire' | 'edited'

  // Marketing/sales fields
  reason_for_interest,          // array, multi-select
  financing_preference,         // string
  evaluator_role,               // string
  project_description,
  preferred_followup,

  // System fields
  submitted_at, source_url, utm_source, utm_medium, utm_campaign
}
```

---

## 9. Open items for Dustin to confirm during/after build

- HubSpot portal ID and form GUID for the Forms API call (not in this build, wire-up step)
- Final brand color values if `#1F2329` and `#2D7A3E` are approximations
- Whether to add a tooltip/info icon on Phase 2 fields explaining what each does
- Whether the "Submit defaults vs edits" choice gets a modal or an inline radio above the submit button (UX-only decision)
- Whether to add a "Forward to a colleague" or "Save as PDF" option on the results page

---

## 10. Data freshness commitments

- **EIA rate tables:** refresh annually each spring after EIA publishes February EPM data. Update `RATES_VINTAGE` string when refreshed.
- **Product catalog:** update when new products launch (flip status to 'available') or when performance data revisions are issued from engineering.
- **Bill range bands:** revisit if monthly electric/gas bill distributions shift materially.

---

## 11. Known limitations (carry-overs from v1.1, plus new)

- State average rates are weighted averages across utilities; visitors on cheaper or more expensive tariffs will see numbers that diverge from reality unless they override.
- Industrial gas rates used as proxy; small-commercial CHP customers may pay slightly more.
- DC, FL, IN, KS, NM gas rates are estimated (flagged with `gasEstimated: true` in data file). Verify against local sources before launch.
- Calculation does not model demand charges, standby tariffs, interconnection fees, incentives, ITC/MACRS, financing structure, or seasonal heat utilization curves. These belong in Application Engineering follow-up, not the screening tool.
- English-only at launch.
- WCAG 2.1 AA audit required before public launch.
- No portfolio mode (one site at a time). Multi-property prospects flagged via evaluator role field for sales handoff.

---

**Next session opening cue:** "Build the v2 demo HTML using chp-screening-data.js and these design notes. Match v1.1 styling and brand tokens. Replace v1.1 scoring, calc engine, and screen flow with v2 per these specs."
