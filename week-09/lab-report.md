# Week 09 — Lab Report: canBook as a Tool

**Name:** Uzakbayev Assylbek  
**Student ID:** 24B032082  
**GitHub:** Assylbek-Uzakbayev

## 1. Setup

| Item | Value |
| --- | --- |
| Assistant (frozen for the whole lab) | ChatGPT, signed-out default |
| Exact model | not displayed |
| Three fresh chats for R1, R2, R3? | yes |
| Python version | Python 3.12.14 |

The first R1 conversation finished without displaying a reply. R1 was repeated with the same prompt and default model setting in a fresh conversation. The saved R1 reply is the one displayed on that repeat.

## 2. Your tool definition — a reason for every constraint

```json
{
  "name": "check_room",
  "description": "Check whether a requested time slot for a room is available. Supply the requested start and end, the current time, whether the room is blocked, and its existing bookings. Use whole minutes since midnight in the inclusive range 0–1440 for all times. This tool does not save a booking. Use an empty bookings list only when the room has no existing bookings.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "start": { "type": "integer", "minimum": 0, "maximum": 1440 },
      "end": { "type": "integer", "minimum": 0, "maximum": 1440 },
      "now": { "type": "integer", "minimum": 0, "maximum": 1440 },
      "blocked": { "type": "boolean" },
      "bookings": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "start": { "type": "integer", "minimum": 0, "maximum": 1440 },
            "end": { "type": "integer", "minimum": 0, "maximum": 1440 }
          },
          "required": ["start", "end"],
          "additionalProperties": false
        }
      }
    },
    "required": ["start", "end", "now", "blocked", "bookings"],
    "additionalProperties": false
  }
}
```

| Field | Constraint | What goes wrong without it |
| --- | --- | --- |
| Tool name | Exact name `check_room` | The client needs the published name to route the call. |
| Tool description | Whole minutes since midnight, inclusive day range, all five inputs, and check-only semantics | A model could use clock strings, assume the room state, or tell the user that a reservation was saved. |
| Arguments | `type: object` | Positional arrays have no named slot for each input. |
| Arguments | Five declared properties and all five `required` | An incomplete request would pass and leave the function without current time, room status, or other required data. |
| Arguments | `additionalProperties: false` | Undeclared values such as `room` could pass and suggest the function accepts a room selector. |
| `start`, `end`, `now`, and both booking endpoints | `type: integer` | A value such as 600.5 or a string such as `"11:00"` would break the whole-minute contract. |
| Same five time fields | `minimum: 0` | Negative values describe time before midnight. |
| Same five time fields | `maximum: 1440` | Values after the end of the day would pass. The inclusive limit keeps 24:00 representable. |
| `blocked` | `type: boolean` | Strings such as `"false"` could pass as a room-status value even though they are not JSON booleans. |
| `bookings` | `type: array` with no minimum count | A list can hold multiple bookings. The empty array represents a room with no existing bookings. |
| Each booking in `bookings.items` | `type: object` with `start` and `end` both required | An unlabeled pair or a record missing an endpoint leaves the conflict interval underspecified. |
| Each booking in `bookings.items` | `additionalProperties: false` | Misspelled or unsupported nested keys would be silently accepted alongside the real endpoints. |

## 3. The twelve calls — first run and final run

I recorded the first checker run against the untouched starter before filling in the schema. The starter accepted all 12 objects, so it failed the nine cases that expected a refusal. The final run accepted three and refused nine.

| Case | First run | Final run | What changed |
| --- | --- | --- | --- |
| S01 | PASS | PASS | The supplied values already satisfy all five constraints. |
| S02 | PASS | PASS | An empty array remains valid. The schema sets no minimum item count. |
| S03 | FAIL | PASS | Required an integer for `end`, which excludes clock strings. |
| S04 | FAIL | PASS | Added `now` to the top-level required list. |
| S05 | FAIL | PASS | Closed the top-level object to undeclared keys such as `room`. |
| S06 | FAIL | PASS | Set the minimum for `start` to 0. |
| S07 | FAIL | PASS | Set the maximum for `end` to 1440. |
| S08 | FAIL | PASS | Made both endpoints required in existing booking objects. |
| S09 | FAIL | PASS | Required a JSON boolean for `blocked`. |
| S10 | FAIL | PASS | Required integer minutes for `start`, which excludes 600.5. |
| S11 | PASS | PASS | Kept 1440 as an inclusive endpoint so 24:00 can pass. |
| S12 | FAIL | PASS | Required each booking to be an object with named endpoints. |

## 4. Function calling — what the model wrote

### R1

```text
{"name":"check_room","arguments":{"start":600,"end":660,"now":540,"blocked":false,"bookings":[{"start":660,"end":720}]}}
```

The request gives `now=540`, a requested interval of 600–660, `blocked=false`, and one existing interval of 660–720. The arguments match. The slot is 60 minutes long, starts after 540, and touches the existing booking at minute 660. Under AC4, touching is allowed, so the expected function result is Available.

### R2

```text
{"name":"check_room","arguments":{"start":840,"end":990,"now":810,"blocked":true,"bookings":[]}}
```

The call matches the request. The model converted 13:30, 14:00 and 16:30 to 810, 840 and 990 respectively, set `blocked=true`, and supplied an empty bookings list. These are valid arguments. The expected function result is Unavailable under AC2 (150 minutes exceeds 120) and AC3 (maintenance block).

### R3

```text
What start and end times would you like tomorrow afternoon?
```

The model asked for the two endpoints. It made no call and invented no numbers. A reply with those endpoints would still leave `now`, room status, and existing bookings unresolved. Also, the schema has no date field for “tomorrow,” so the caller would need to establish the target day before sourcing the room state and applying same-day minute comparisons.

## 5. What the schema cannot check

| Criterion | Where it is enforced |
| --- | --- |
| AC1 | The schema enforces whole minutes from 0 to 1440 inclusive. `canBook` must compare `start` with both `end` and `now` to enforce `start < end` and `start > now`. |
| AC2 | `canBook` measures `end - start` and rejects durations over 120 minutes. The allowed keywords set limits on individual numbers. |
| AC3 | The schema accepts either boolean so the call can describe the room honestly. `canBook` returns Unavailable when `blocked` is true. |
| AC4 | `canBook` checks for overlap against the booking list. With strict comparisons, a conflict exists when the request starts before an existing booking ends and ends after that booking starts. Equal boundaries can touch. |
| AC5 | The implementation must return a result without modifying or persisting a booking. The schema only validates inputs, and the description documents the contract. |
| AC6 | The page is responsible for showing feedback based on the result. Input validation cannot inspect the page or confirm what the user sees. |

## 6. Permissions — allow, ask or block

| # | Action | Decision | Reason |
| --- | --- | --- | --- |
| A1 | Call `check_room` with the times the user just typed | allow | A complete call only checks availability. If room state is missing, ask for it or obtain it from an authorized source before supplying the arguments. |
| A2 | Read `week-08/app/src/booking.ts` | allow | The local file is relevant to verifying the booking rule. Scope the read to this file so unrelated project data stays out of the task. |
| A3 | Edit `booking.ts` to raise the 120-minute limit to 180 | ask | A three-hour booking request may be valid input without granting authority to change the course’s duration rule for every user. |
| A4 | Run `npm test` in `week-08/app` | allow | Local tests are appropriate for verifying booking behavior when the repository and test command are trusted. Inspect the test script if its behavior is unfamiliar because npm can execute project code. |
| A5 | Run `git reset --hard` to undo its own last three edits | ask | A hard reset also discards unrelated staged and unstaged tracked changes. Inspect the diff and confirm the target before using it. |
| A6 | Commit its change on the `week-09` branch | ask | A commit records a persistent snapshot and may include unrelated staged work or secrets. Review the staged diff and intended branch first. |
| A7 | Push to `main` | block | Direct pushes can expose unreviewed code and bypass the course’s required PR from `week-09` to `main`. |
| A8 | Send `.env` to an MCP server it found online | block | The file may contain API keys or credentials. An unverified remote server could store or misuse them. |

## 7. Checkpoint

The checkpoint is commit `820194c` (`Week 09: record the supplied starter as a checkpoint`). It was created on the local `week-09` branch before changing the tool definition. This checkpoint is in the SofwareENG repository's week-09 branch.

From the `week-09` branch in that local history, the rollback command would be:

```bash
git reset --hard 820194c
```

The command would move the branch back to the starter commit, discard staged and unstaged changes to tracked files, and remove the later Week 09 commits from the branch tip. Untracked files are generally kept unless they are in the way of a tracked path that Git restores. The command was documented only.

## 8. Checker output

```text
PASS  D1    tool/check_room.json exists and is valid JSON
PASS  D2    has name, description and inputSchema
PASS  D3    name is exactly "check_room"
PASS  D4    description: 25+ words, says minutes, 1440, and that it does not save
PASS  D5    inputSchema: an object with exactly the five fields, all required
PASS  D6    inputSchema uses only the keywords listed in README section 5
PASS  S01   the R1 call, valid -> accept
PASS  S02   no existing bookings: bookings = [] -> accept
PASS  S03   end as a string: "11:00" -> refuse
PASS  S04   now missing -> refuse
PASS  S05   an extra field: "room": "204" -> refuse
PASS  S06   start below the day: -30 -> refuse
PASS  S07   end beyond the day: 1500 -> refuse
PASS  S08   a booking with no end: {start: 660} -> refuse
PASS  S09   blocked as a string: "false" -> refuse
PASS  S10   start not a whole minute: 600.5 -> refuse
PASS  S11   end at the end of the day: 1440 -> accept
PASS  S12   a booking as a pair: [660, 720] -> refuse
PASS  F1    calls\r1.json: a call to check_room
PASS  F2    calls\r1.json: arguments valid against your schema
PASS  F3    calls\r1.json: arguments equal the expected ones (README section 5, Part 2)
PASS  F4    calls\r2.json: a call to check_room
PASS  F5    calls\r2.json: arguments valid against your schema
PASS  F6    calls\r2.json: arguments equal the expected ones (README section 5, Part 2)
PASS  F7    calls/r3: exactly one of r3.json (it called) or r3.txt (it asked), not empty
PASS  L1    section 5 names at least two acceptance criteria (AC1-AC6) the schema cannot check
PASS  L2    section 6: A1-A8 each have allow / ask / block and a reason of 5+ words
PASS  L3    section 7 names the checkpoint commit (a SHA)
PASS  L4    section 10: a conclusion of 80-150 words
PASS  L5    AI_USAGE.md filled in (a tool row, and signed)

SUMMARY: 30 PASS · 0 FAIL · 0 ERROR  (of 30 checks)
Paste this whole output into lab-report.md section 8, and these three numbers into submission.yml.
```

## 9. What still fails

None.

## 10. Conclusion (80–150 words)

The starter accepted S03 because it had no type constraint for the requested end time. Declaring integer minutes catches that error before the function runs. R2 shows the limit of that protection. All its arguments fit the schema, but a blocked room and a 150-minute duration each independently make the slot unavailable. R3 shows a separate problem for the caller. The model requested precise endpoints, yet even a reply containing those times would leave room status and existing bookings unknown. The date also needs to be resolved outside this tool. I would allow a complete availability check under A1 and require approval for A3 because editing the duration limit changes the rule for everyone.
