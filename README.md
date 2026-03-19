# Healthcare Admin Agents

> 51 specialized AI agents for US healthcare administration. MHA-level expertise in revenue cycle, compliance, quality, clinical operations, payer relations, health IT, and more.

Inspired by and built on the conventions of [The Agency](https://github.com/msitarzewski/agency-agents) by Michael Sitarzewski.

## What This Is

A complete healthcare administration department in your AI coding assistant. Each agent is a specialized expert with real regulatory knowledge, operational workflows, and deliverable templates — not a generic chatbot with a healthcare skin.

These agents know specific CFR sections, real EHR systems, actual CMS payment models, and the operational details that separate a textbook answer from practitioner-level advice.

**Target audience**: Healthcare professionals (MHA-level and above). These agents assume you know the domain and communicate peer-to-peer.

## Installation

### Claude Code

```bash
git clone https://github.com/ajhcs/healthcare-agents.git
cp healthcare-agents/agents/*.md ~/.claude/agents/
```

Agents are available immediately. Reference by name in conversation:

> "Activate the 340B Program Manager and help me assess our contract pharmacy compliance"

### Codex CLI (OpenAI)

Codex CLI uses TOML for agent definitions. A conversion script is provided:

```bash
git clone https://github.com/ajhcs/healthcare-agents.git
cd healthcare-agents

# Option 1: Install as read-only context (simplest)
mkdir -p ~/.codex/agents
cp agents/*.md ~/.codex/agents/

# Then add to ~/.codex/AGENTS.md:
echo "Refer to agent files in ~/.codex/agents/ for healthcare administration expertise." >> ~/.codex/AGENTS.md
```

Or reference the agents directory directly in your project's `AGENTS.md`:

```markdown
# Healthcare Agents
Load agent personality files from ./agents/ for healthcare administration expertise.
Each file defines a specialist. Adopt the identity described in the file when asked.
```

### Gemini CLI

```bash
git clone https://github.com/ajhcs/healthcare-agents.git
mkdir -p ~/.gemini/agents
cp healthcare-agents/agents/*.md ~/.gemini/agents/
```

Enable subagents in `~/.gemini/settings.json`:

```json
{
  "experimental": {
    "enableAgents": true
  }
}
```

Invoke agents with `@agent-name` or let Gemini auto-select based on task relevance.

### OpenClaw

```bash
git clone https://github.com/ajhcs/healthcare-agents.git
mkdir -p ~/.openclaw/skills/healthcare-agents

# Copy each agent as a skill
for f in healthcare-agents/agents/*.md; do
  name=$(basename "$f" .md)
  mkdir -p ~/.openclaw/skills/healthcare-agents/$name
  cp "$f" ~/.openclaw/skills/healthcare-agents/$name/SKILL.md
done
```

Or install at workspace scope by placing the skill directories in `./skills/` within your working directory.

### Claude Desktop — Cowork

1. Open Claude Desktop and navigate to the **Cowork** tab
2. Go to **Settings > Cowork > Global Instructions**
3. Reference agents as domain knowledge, or create a plugin:

```
your-plugin/
  .claude-plugin/plugin.json
  skills/
    # Copy each .md agent file here as a skill
```

For manual use, paste agent content into project instructions or add agent files to a **Claude Context** folder.

### Claude Web (Projects)

1. Open [claude.ai](https://claude.ai) and create a new **Project**
2. Click **Set custom instructions**
3. Paste the content of any agent file into the project instructions
4. Upload additional agent `.md` files as project knowledge

Each project supports one primary persona — choose the agent that matches your workflow.

### Cursor

```bash
git clone https://github.com/ajhcs/healthcare-agents.git
mkdir -p .cursor/rules
cp healthcare-agents/agents/*.md .cursor/rules/
```

Cursor rules support YAML frontmatter. The existing agent frontmatter (`name`, `description`) will be recognized. For automatic activation, you can optionally add `alwaysApply: false` to let Cursor intelligently select the right agent based on context.

### Windsurf

```bash
git clone https://github.com/ajhcs/healthcare-agents.git
mkdir -p .windsurf/rules
cp healthcare-agents/agents/*.md .windsurf/rules/
```

Optionally add `trigger: model_decision` to the YAML frontmatter of each file so Windsurf activates the right specialist automatically.

### Cline

```bash
git clone https://github.com/ajhcs/healthcare-agents.git
mkdir -p .clinerules
cp healthcare-agents/agents/*.md .clinerules/
```

### GitHub Copilot

```bash
git clone https://github.com/ajhcs/healthcare-agents.git
mkdir -p .github/instructions
cp healthcare-agents/agents/*.md .github/instructions/
```

Rename files to use the `.instructions.md` extension for path-specific activation:

```bash
for f in .github/instructions/*.md; do
  mv "$f" "${f%.md}.instructions.md"
done
```

Enable in VS Code settings: `github.copilot.chat.codeGeneration.useInstructionFiles: true`

### Amazon Q Developer

```bash
git clone https://github.com/ajhcs/healthcare-agents.git
mkdir -p .amazonq/rules
cp healthcare-agents/agents/*.md .amazonq/rules/
```

Rules are automatically applied as context in all Amazon Q chat sessions.

### Aider

```bash
git clone https://github.com/ajhcs/healthcare-agents.git
```

Add to `.aider.conf.yml`:

```yaml
read:
  - healthcare-agents/agents/revenue-340b-program-manager.md
  - healthcare-agents/agents/quality-compliance-officer.md
  # Add the specific agents you need
```

Or load in-session: `/read healthcare-agents/agents/revenue-340b-program-manager.md`

### Any Other Tool

Each agent is a self-contained Markdown file with YAML frontmatter. The format is:

```yaml
---
name: Agent Name
description: One-line specialty summary
color: "#hex"
emoji: 🏥
vibe: One-sentence personality
---

# Agent Name

System prompt and domain knowledge...
```

If your tool supports custom instructions, system prompts, or persona files — these agents work. Copy the `.md` file into wherever your tool reads custom instructions from, or paste the content directly into a system prompt field.

## The 10 Divisions

### Strategy & Advisory (5 agents)
| Agent | Specialty |
|-------|-----------|
| Healthcare Strategy Consultant | Service line planning, M&A, market analysis, CON strategy |
| Healthcare Operations Consultant | Lean/Six Sigma, throughput, benchmarking (MGMA/Vizient) |
| Clinical Operations Consultant | Clinical workflows, staffing models, ED/OR throughput |
| Structural Improvement Consultant | Org redesign, governance, change management, post-merger integration |
| Healthcare Actuarial Advisor | Risk adjustment (HCC/RAF), capitation, IBNR, MLR modeling |

### Clinical Operations (8 agents)
| Agent | Specialty |
|-------|-----------|
| Utilization Management Specialist | Medical necessity, InterQual/Milliman, Two-Midnight Rule |
| Care Management Specialist | Care coordination, TCM/CCM, readmission prevention, SDOH |
| Clinical Research Coordinator | IRB, ICH-GCP E6(R3), 21 CFR Part 11, trial management |
| Documentation Improvement Specialist | CDI queries, CC/MCC capture, DRG optimization |
| Prior Authorization Specialist | PA workflows, appeals, gold carding, ePA (CMS-0057-F) |
| Referral Specialist | Referral management, network navigation, care gap closure |
| Case Manager | Discharge planning, post-acute placement, LOS optimization |
| Infection Prevention Specialist | HAI surveillance (NHSN), antimicrobial stewardship, outbreak response |

### Quality, Safety & Compliance (7 agents)
| Agent | Specialty |
|-------|-----------|
| Quality Improvement Specialist | HEDIS, MIPS/QPP, CMS Stars, Leapfrog, eCQMs |
| Process Improvement Analyst | PDSA, Lean, Six Sigma DMAIC, value stream mapping |
| Patient Experience Coordinator | HCAHPS, service recovery, VBP patient experience domain |
| Patient Safety Officer | Sentinel events, RCA/FMEA, Just Culture, PSO reporting |
| Compliance Officer | HIPAA, Stark, Anti-Kickback, FCA, OIG compliance programs, EMTALA |
| Risk Manager | Enterprise/clinical risk, malpractice, claims management |
| Accreditation Specialist | Joint Commission, NCQA, URAC, AAAHC, DNV, survey readiness |

### Revenue Cycle & Finance (6 agents)
| Agent | Specialty |
|-------|-----------|
| Revenue Cycle Specialist | End-to-end RCM, denials (CARC/RARC), A/R optimization |
| Healthcare Finance Manager | Budgets, cost accounting, CMS-2552 cost reports, margin analysis |
| Healthcare Contract Analyst | Payer contracts, fee schedules, reimbursement modeling |
| Medical Coding Specialist | ICD-10-CM/PCS, CPT, DRG, HCC, E/M coding |
| 340B Program Manager | Covered entity compliance, contract pharmacy, split billing, HRSA audits |
| Chargemaster Analyst | CDM maintenance, price transparency, charge capture integrity |

### Payer & Managed Care (6 agents)
| Agent | Specialty |
|-------|-----------|
| Value-Based Care Manager | ACO operations (MSSP/ACO REACH), shared savings, risk contracts |
| Payer Relations Specialist | Network development, contract negotiation, No Surprises Act |
| Medicare & Medicaid Specialist | CMS regulations, CoPs, MAC requirements, dual-eligible programs |
| Managed Care Analyst | Capitation modeling, MLR, PMPM, network adequacy |
| Credentialing & Enrollment Coordinator | CAQH, PECOS, CMS-855, privileging, delegated credentialing |
| Medicare Outreach Coordinator | Beneficiary education, enrollment periods, LIS/Extra Help |

### Population Health & Community (3 agents)
| Agent | Specialty |
|-------|-----------|
| Population Health Manager | Risk stratification, care gaps, SDOH, chronic disease programs |
| Public Health Surveillance Coordinator | Reportable diseases, outbreak investigation, syndromic surveillance |
| Community Health Coordinator | CHNA, Schedule H, health equity, CHW programs, grant management |

### Health IT & Informatics (6 agents)
| Agent | Specialty |
|-------|-----------|
| Health Informatics Manager | Clinical informatics, USCDI/TEFCA, ONC HTI-1, data governance |
| Epic Applications Analyst | Epic build/config, Bridges, Caboodle/Cogito, certification |
| Health Information Manager | HIM operations, ROI, record retention, legal health record |
| Clinical Data Analyst | Registries, eCQMs, MIPS reporting, SQL/Python for healthcare data |
| Healthcare Interoperability Engineer | HL7v2, FHIR R4, C-CDA, X12 EDI, HIE connectivity |
| Telehealth Program Manager | Virtual care ops, licensure compacts, RPM/RTM billing |

### Operations & Administration (7 agents)
| Agent | Specialty |
|-------|-----------|
| Hospital Operations Administrator | Bed management, capacity planning, patient throughput |
| Physician Practice Manager | wRVU compensation, MGMA benchmarking, practice operations |
| Ambulatory Operations Manager | Clinic workflows, scheduling optimization, patient access |
| Home Health Administrator | CoPs (42 CFR 484), OASIS, PDGM, home health VBP |
| Long-Term Care Administrator | SNF CoPs (42 CFR 483), MDS, PDPM, Five-Star, survey readiness |
| Supply Chain Manager | GPO management, value analysis, OR supplies, FDA recalls |
| Healthcare Workforce Manager | Staffing models, scheduling, retention, burnout prevention |

### Pharmacy & Drug Programs (2 agents)
| Agent | Specialty |
|-------|-----------|
| Pharmacy Benefits Specialist | Formulary, PBM contracts, specialty pharmacy, biosimilars |
| Medication Safety Specialist | ISMP, LASA drugs, CPOE, BCMA, USP 797/800 |

### Emergency & Preparedness (1 agent)
| Agent | Specialty |
|-------|-----------|
| Emergency Preparedness Coordinator | HICS, CMS EP CoPs (42 CFR 482.15), surge planning, HVA |

## What Makes These Agents Different

**Real regulatory depth.** Every agent cites specific CFR sections, CMS transmittals, and Federal Register notices. Not "follow HIPAA" — the actual Privacy Rule sections (45 CFR 164.500-534) and breach notification requirements (45 CFR 164.400-414).

**Actual deliverables.** Each agent includes 2-4 fill-in-ready templates — compliance assessments, financial analyses, audit checklists, gap reports. Not descriptions of what a deliverable would look like. The deliverable itself.

**Real systems.** Agents know Epic, Cerner, MEDITECH, CAQH ProView, 340B OPAIS, PECOS, NHSN, and the actual tools healthcare administrators use. Including API capabilities and integration patterns where relevant.

**Expert audience.** These agents assume you know what a DRG is. They don't explain HIPAA from scratch. They discuss CC/MCC capture rates, PMPM decomposition, and F-tag citations because that's how healthcare professionals actually talk.

## Stats

- **51 agents** across 10 divisions
- **21,000+ lines** of domain knowledge
- **Average 420 lines/agent** of dense, real expertise
- Real regulatory citations throughout (42 CFR, USC, CMS transmittals, Federal Register)
- Actual deliverable templates with functional placeholders

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on creating new agents or improving existing ones.

## Validation

```bash
bash scripts/lint-agents.sh
```

## Attribution

This pack follows the conventions and structure established by [agency-agents](https://github.com/msitarzewski/agency-agents) by [Michael Sitarzewski](https://github.com/msitarzewski). The agency-agents project demonstrated that markdown-based agent personalities with strong voice, concrete deliverables, and measurable outcomes produce dramatically better AI assistance than generic prompts.

## Disclaimer

These agents provide healthcare administration knowledge for informational and operational purposes. They do not provide clinical advice (diagnosis, treatment, prescribing), legal opinions, or handle protected health information (PHI). Healthcare regulations change frequently — verify against primary sources (CMS.gov, Federal Register, state regulatory bodies) for current requirements.

## License

Apache 2.0 — see [LICENSE](LICENSE).
