# infra-topology-generator

A Python CLI tool that generates structured, validated network topology JSON for
critical infrastructure systems from plain-English descriptions.

Powered by the Claude API. Compliance validation aligned with **IEC 62443** security
zone design principles and **IEC 61850** substation communication standards.

---

## What it does

1. Takes a plain-English description of a network (typed or from a file)
2. Calls Claude to generate a structured JSON topology (nodes, connections, zones)
3. Runs rule-based IEC 62443 compliance checks against the output
4. Prints the topology JSON and a compliance report

This mirrors the core workflow of an AI-assisted infrastructure design platform:
natural language in → validated, deployment-ready architecture out.

---

## Architecture

<p align="center"><img src="docs/architecture.svg" width="100%" alt="One run: a plain-English description, typed or from a file, goes into a prompt with the JSON structure and IEC 62443 design rules. Claude (claude-opus-4-5) returns topology JSON. main.py strips code fences and parses it; invalid JSON stops the run. validator.py runs three rule checks and prints a compliance report and the topology JSON. The checks use the IEC 62443 zones corporate_it, dmz, supervisory, control and field: OT nodes never in corporate_it, a DMZ when IT and OT zones are both present, and every cross-zone connection encrypted."></p>

**Why this architecture**

| Choice | Why |
|---|---|
| LLM drafts, plain code checks | The model can break rules it was told to follow (see Results). The validator applies the same rules the same way every time. |
| Rules in both the prompt and the validator | The prompt steers the draft toward a compliant design; the validator confirms it instead of trusting it. |
| Fixed JSON structure with fixed node types and zone names | Output that code can check, save and pass on to other tools, not free text. |
| IEC 62443 zone levels (corporate IT → field) | The standard zone model for industrial control systems, so each rule maps to a known requirement. |
| Small CLI, one dependency (`anthropic`) | Easy to run and to read. Scenarios are text files, so the same input can be run again. |

## Results

There are no automated tests or CI yet. The results below are the two example runs in `examples/` (screenshots taken 2 May 2026).

| Input | Rules passed | What the validator reported |
|---|---|---|
| `examples/substation.txt` | 2 of 3 | **Failed** `IEC62443-ENCRYPTED-CROSS-ZONE`: 3 unencrypted links from the SCADA server to the HMI and two PLCs (IEC 61850 MMS), even though the description says cross-zone links are encrypted. |
| `examples/water_treatment.txt` | 3 of 3 | Compliant. |

## What worked and what didn't

| | What | Detail |
|---|---|---|
| ✅ | The validator caught a model mistake | In the substation run the model marked three cross-zone links as unencrypted, against both the input and its own instructions. The check flagged each link by name. |
| ✅ | Structured output | Both runs returned valid JSON in the expected structure, which the validator could check without changes. |
| ❌ | Same input, different output | The substation run under *Example output* (all 3 rules passed, node `plc-01`) and the screenshot (1 rule failed, node `plc-01a`) come from different runs of the same file. |
| ⚠️ | Only 3 rules | The prompt also asks for a firewall at every zone boundary; the validator does not check that yet. |
| ⚠️ | No schema check | `schema.py` defines the data model, but the LLM output is used as a plain dict. A node without a `zone` field would crash the validator. Invalid JSON stops the run with no retry. |

---

## Screenshots

**Substation network — compliance violation detected:**
![Substation output](examples/substation-output.png)

**Water treatment network — fully compliant:**
![Water treatment output](examples/water-treatment-output.png)

---

## Example output

Input (`examples/substation.txt`):
```
A power utility substation network connecting corporate IT systems to OT equipment.
The OT side includes PLCs, RTUs, an HMI, and a SCADA server...
```

Output (truncated):
```json
{
  "name": "Power Utility Substation Network",
  "description": "Segmented IT/OT network with DMZ, IEC 61850 OT communication, and redundant PLCs.",
  "nodes": [
    { "id": "fw-01", "label": "Perimeter Firewall", "type": "firewall", "zone": "corporate_it", "redundant": false },
    { "id": "plc-01", "label": "Breaker PLC Primary", "type": "plc", "zone": "control", "redundant": true },
    { "id": "rtu-01", "label": "Field RTU 1", "type": "rtu", "zone": "field", "redundant": false }
  ],
  "connections": [
    { "source": "scada-01", "target": "plc-01", "protocol": "IEC 61850 MMS", "encrypted": true }
  ]
}
```

Compliance report:
```
── Compliance Report (IEC 62443) ────────────────────────────────
  ✓ [PASS] IEC62443-OT-ISOLATION
      All OT nodes are correctly isolated from the corporate IT zone.
  ✓ [PASS] IEC62443-DMZ-REQUIRED
      DMZ zone is present or not required for this topology.
  ✓ [PASS] IEC62443-ENCRYPTED-CROSS-ZONE
      All cross-zone connections are encrypted.

  Result: 3 passed, 0 failed, 0 warnings
  ✓ Topology is IEC 62443 compliant
─────────────────────────────────────────────────────────────────
```

---

## Setup

```bash
git clone https://github.com/kchimbodza/infra-topology-generator.git
cd infra-topology-generator
pip install anthropic
export ANTHROPIC_API_KEY=your_key_here
```

---

## Usage

**Interactive mode** (type your description):
```bash
python main.py
```

**File input:**
```bash
python main.py --input examples/substation.txt
```

**Save topology to file:**
```bash
python main.py --input examples/substation.txt --output topology.json
```

**Skip compliance validation:**
```bash
python main.py --input examples/substation.txt --no-validate
```

---

## Project structure

```
infra-topology-generator/
├── main.py          # CLI entry point — orchestrates generation and validation
├── schema.py        # Topology data model (Node, Connection, Topology dataclasses)
├── prompts.py       # System prompt and user prompt templates for the LLM
├── validator.py     # Rule-based IEC 62443 compliance checks
├── docs/
│   └── architecture.svg     # Diagram above
└── examples/
    ├── substation.txt       # Power utility substation scenario
    └── water_treatment.txt  # Municipal water treatment facility scenario
```

---

## Compliance rules

| Rule | Standard | Description |
|------|----------|-------------|
| `IEC62443-OT-ISOLATION` | IEC 62443 | OT nodes (PLC, RTU, HMI, historian) must not reside in the corporate IT zone |
| `IEC62443-DMZ-REQUIRED` | IEC 62443 | A DMZ must exist when both corporate IT and OT zones are present |
| `IEC62443-ENCRYPTED-CROSS-ZONE` | IEC 62443 | All connections crossing zone boundaries must be encrypted |

---

## Standards reference

- **IEC 62443** — Security for industrial automation and control systems. Defines security levels and zone/conduit model used for node zone classification in this tool.
- **IEC 61850** — Communication networks and systems in substations. Protocol names (GOOSE, MMS, SAMPLED VALUES) used in generated connection labels.
