# Commission Review Skills

Claude skills for Spot AI's sales commission review workflows.

---

## `spot-commission-quarterly-review`

Runs a quarter of sales commission end to end: pulls payouts and earnings from Visdum,
reverse-engineers quota attainment and accelerator tiers, builds a deduplicated live-ARR base,
reconciles every paid deal against Salesforce and NetSuite, and produces an interactive
dashboard, a multi-tab Excel workbook and an executive one-pager.

### Why this exists

Pulling commission numbers is easy. Knowing which ones are wrong by construction is not. This
skill encodes three traps that cost real money if you miss them.

**Attainment is not in the attainment field.** Visdum's `achievement` column on
`quota_allotments` returns **zero for every allotment**. Attainment has to be reverse-engineered
from the `earning_calculation` string, which carries every input as a literal — credited ARR,
quota, accelerator tier, multipliers.

**Credited ARR double-counts three separate ways.** A single deal is credited to the AE, their
manager and the supporting solution engineer, so summing across payees multiplies the same ARR.
Contracted-ARR parents release into the quarter while the child shipment carries the same
`arr__c`. Pre-shipped and partner pass-through deals earn commission on $0 of recurring revenue.
Miss any of these and the cost-of-ARR figure is wrong by tens of points.

**Salesforce disagrees with the commission engine for two legitimate reasons.** Pre-shipped
orders carry $0 `arr__c` by design. Renewals differ because Visdum credits full renewal value
while `arr__c` holds only net-new ARR. Reporting these as variances buries the handful of deals
that genuinely need a decision — which is the entire point of the reconciliation.

### What it produces

| Deliverable | Audience |
|---|---|
| Interactive HTML dashboard | The working artifact — attainment, payout vs variable, motion, per-seller, register, exceptions |
| Excel workbook | Evidence pack — deal register, Salesforce reconciliation, double-count controls, one tab per seller |
| Executive one-pager | The call, what's working, what's broken, decisions with owners and amounts |

### Layout

```
spot-commission-quarterly-review/
├── SKILL.md                       workflow, aggregation rules, recurring failure modes
├── references/
│   ├── visdum.md                  data model, query mechanics, accelerator table, validation gates
│   ├── arr-base.md                the three double-counts and how to remove each
│   ├── validation.md              Salesforce severity classification, NetSuite fulfilment tiering
│   └── deliverables.md            structure of the dashboard, workbook and one-pager
└── scripts/
    ├── parse_earning_calc.py      parse earning rules into attainment, tier, rate, multipliers
    ├── build_live_arr.py          deduplicated live-ARR base and payout rate on ARR
    └── recon_salesforce.py        deal-by-deal reconciliation, classified by severity
```

`SKILL.md` is loaded whenever the skill triggers. The `references/` files are read only when the
relevant step needs them, which keeps the working context small.

### Systems it touches

- **Visdum** — payouts, earnings, quota allotments, plan components
- **Salesforce via BigQuery** (`jf-hevo.core`) — opportunity stage, close date, ARR, owner
- **NetSuite** — sales orders and item fulfilments, to corroborate that hardware actually shipped

### Getting started

See [INSTRUCTIONS.md](INSTRUCTIONS.md) for installation, a walkthrough of a full quarter review,
and how to run each script standalone.

### A note on the scripts

Each one refuses to pass silently. `parse_earning_calc.py` reports any formula shape it could not
parse and checks that `variable × accelerator_rate × blended_multiplier` reproduces actual Visdum
earning before you rely on the output. `recon_salesforce.py` warns when the Salesforce pull is
missing opportunities rather than reconciling against a partial set. If a validation gate fails,
fix the parse — do not proceed to the deliverables.

