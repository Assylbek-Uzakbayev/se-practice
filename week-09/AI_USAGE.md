# AI Usage Disclosure — Week 09

AI-use level D. This is a disclosure prepared for review by Uzakbayev Assylbek.

| Tool | Exact model / plan | Used for | Which files it touched |
| --- | --- | --- | --- |
| ChatGPT | not displayed (signed-out default) | Part 2 — producing the R1, R2 and R3 replies in separate chats | Model responses captured as `calls/r1.json`, `calls/r2.json`, and `calls/r3.txt` |
| Codex | not displayed | Drafted the tool definition and lab report, recorded test runs, and prepared the declaration and Git handoff | `tool/check_room.json`, `lab-report.md`, `AI_USAGE.md`, `submission.yml` |

**I wrote `tool/check_room.json` myself:** no. Codex drafted the entire definition. The supplied checker verified D1–D6 and S01–S12 after the starter run was recorded.

**I used an AI to explain a JSON Schema keyword or a check:** yes. Codex drafted the rationale for the required fields, closed objects, array items and inclusive integer bounds, and explained how schema validation differs from the function’s checks.

**Every reply in `calls/` and in lab-report.md section 4 is exactly what the model returned:** yes. Each visible reply was copied verbatim.

**Every terminal output in lab-report.md was pasted from a real run, unedited:** yes. The complete final checker output is reproduced in Section 8.

**Anything I accepted from the AI without fully understanding it:** Student review pending. This draft does not make a claim about the student's understanding.

Signed: Uzakbayev Assylbek
Date: 2026-11-15