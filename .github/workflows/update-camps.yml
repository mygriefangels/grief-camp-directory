"""
Grief Camp Directory — monthly auto-update script
==================================================
Runs once a month (via GitHub Actions). Asks Claude, with web search,
to research currently-operating grief camps across the USA, then writes
the results to camps.json — which the MyGriefAngels.org directory
embed reads automatically.

COST GUARDS (in order of importance):
  1. The hard ceiling is the monthly spend limit you set in the
     Anthropic Console (see SETUP-GUIDE.md, Step 3). Nothing in this
     script can exceed that.
  2. MAX_WEB_SEARCHES caps how many searches one run may perform.
  3. MAX_OUTPUT_TOKENS caps the response size.
  4. The script refuses to run if camps.json is fresher than
     MIN_DAYS_BETWEEN_RUNS (protects against accidental re-runs).
  5. Runs only once a month by schedule.
"""

import json
import os
import re
import sys
from datetime import datetime, timezone

from anthropic import Anthropic

# ----------------------- Tunable settings -----------------------
MODEL = os.environ.get("CAMP_MODEL", "claude-sonnet-4-6")
MAX_WEB_SEARCHES = int(os.environ.get("CAMP_MAX_SEARCHES", "12"))
MAX_OUTPUT_TOKENS = int(os.environ.get("CAMP_MAX_TOKENS", "8000"))
MIN_DAYS_BETWEEN_RUNS = int(os.environ.get("CAMP_MIN_DAYS", "20"))
MIN_CAMPS_TO_ACCEPT = 5   # if research returns fewer, keep the old file
OUTPUT_FILE = "camps.json"
REVIEW_FILE = "latest-run-review.md"
# -----------------------------------------------------------------

LOSS_KEYS = {"child", "sibling", "parent", "spouse", "suicide",
             "overdose", "infant", "friend", "multiple"}
AGE_KEYS = {"children", "teens", "youngAdults", "adults", "families"}
FAMILY_KEYS = {"parents", "siblings", "spouses", "grandparents"}

PROMPT = """You are helping MyGriefAngels.org, a volunteer-run non-profit, maintain a
free public directory of grief camps and grief retreats in the United States.

Use web search to find grief camps and bereavement retreats that are CURRENTLY
OPERATING in the USA, with sessions scheduled now or in the future where possible.
Prioritize well-established programs. Aim for 15-30 camps covering a variety of:
loss types (child, sibling, parent, spouse/partner, suicide loss, overdose loss,
pregnancy/infant loss), regions of the country, free and paid options, and
in-person plus virtual programs.

STRICT ACCURACY RULES:
- Only include a camp if you found it on its own official website or its
  sponsoring organization's website during this research session.
- Use ONLY contact details (email, phone) shown on those official pages.
  If you cannot verify an email or phone, use an empty string "" — never guess.
- If dates for the next session aren't published yet, describe what the site
  says (e.g. "Annual, summer — 2027 dates TBA").
- Do not invent camps, details, or contact information under any circumstances.

OUTPUT FORMAT — respond with ONLY a JSON object inside <json></json> tags,
no other commentary. Schema:

<json>
{
  "camps": [
    {
      "name": "string (required)",
      "org": "string — sponsoring organization",
      "founded": 2009,
      "email": "string or \\"\\"",
      "phone": "string or \\"\\"",
      "website": "string — official URL you verified",
      "lossTypes": ["child"],
      "format": "in-person | virtual | hybrid",
      "city": "string ('Online' for virtual)",
      "state": "2-letter code, or '—' for virtual",
      "dates": "string as families should read it",
      "startDate": "YYYY-MM-DD or null",
      "durationDays": 3,
      "ageGroups": ["adults"],
      "gender": "all | women | men",
      "familyStatus": ["parents"],
      "recentLossOk": true,
      "religious": "faith-based | secular | interfaith",
      "languages": ["English"],
      "animalTherapy": false,
      "recoveryFocused": false,
      "cost": 0,
      "costNote": "string, e.g. 'Free, including meals'",
      "financialAid": true
    }
  ]
}
</json>

Allowed values: lossTypes from [child, sibling, parent, spouse, suicide,
overdose, infant, friend, multiple]; ageGroups from [children, teens,
youngAdults, adults, families]; familyStatus from [parents, siblings,
spouses, grandparents]. founded: best estimate from the site, or the
current year if unknown. cost: number, 0 = free."""


def fail(msg: str) -> None:
    print(f"ERROR: {msg}", file=sys.stderr)
    sys.exit(1)


def file_age_days(path: str) -> float:
    try:
        with open(path) as f:
            updated = json.load(f).get("updatedAt", "")
        dt = datetime.fromisoformat(updated.replace("Z", "+00:00"))
        return (datetime.now(timezone.utc) - dt).total_seconds() / 86400
    except Exception:
        return 9999


def clean_camp(c: dict, idx: int) -> dict | None:
    name = str(c.get("name", "")).strip()
    if not name:
        return None
    def lst(key, allowed):
        vals = c.get(key) or []
        if isinstance(vals, str):
            vals = [vals]
        return [v for v in vals if v in allowed]
    fmt = c.get("format", "in-person")
    if fmt not in ("in-person", "virtual", "hybrid"):
        fmt = "in-person"
    rel = c.get("religious", "secular")
    if rel not in ("faith-based", "secular", "interfaith"):
        rel = "secular"
    gender = c.get("gender", "all")
    if gender not in ("all", "women", "men"):
        gender = "all"
    try:
        cost = max(0, float(c.get("cost", 0) or 0))
    except (TypeError, ValueError):
        cost = 0
    try:
        founded = int(c.get("founded") or datetime.now().year)
    except (TypeError, ValueError):
        founded = datetime.now().year
    start = c.get("startDate")
    if start and not re.match(r"^\d{4}-\d{2}-\d{2}$", str(start)):
        start = None
    return {
        "id": f"auto{idx}",
        "name": name,
        "org": str(c.get("org", "")).strip(),
        "founded": founded,
        "email": str(c.get("email", "")).strip(),
        "phone": str(c.get("phone", "")).strip(),
        "website": str(c.get("website", "")).strip(),
        "lossTypes": lst("lossTypes", LOSS_KEYS) or ["multiple"],
        "format": fmt,
        "city": str(c.get("city", "")).strip(),
        "state": str(c.get("state", "")).strip() or "—",
        "dates": str(c.get("dates", "")).strip(),
        "startDate": start,
        "durationDays": int(c.get("durationDays") or 1),
        "ageGroups": lst("ageGroups", AGE_KEYS) or ["adults"],
        "gender": gender,
        "familyStatus": lst("familyStatus", FAMILY_KEYS),
        "recentLossOk": bool(c.get("recentLossOk", False)),
        "religious": rel,
        "languages": [str(x) for x in (c.get("languages") or ["English"])],
        "animalTherapy": bool(c.get("animalTherapy", False)),
        "recoveryFocused": bool(c.get("recoveryFocused", False)),
        "cost": cost,
        "costNote": str(c.get("costNote", "")).strip() or ("Free" if cost == 0 else f"${cost:g}"),
        "financialAid": bool(c.get("financialAid", False)),
    }


def main() -> None:
    if not os.environ.get("ANTHROPIC_API_KEY"):
        fail("ANTHROPIC_API_KEY is not set (add it as a GitHub Actions secret).")

    age = file_age_days(OUTPUT_FILE)
    if age < MIN_DAYS_BETWEEN_RUNS and os.environ.get("FORCE_RUN") != "yes":
        print(f"camps.json is only {age:.0f} days old "
              f"(< {MIN_DAYS_BETWEEN_RUNS}); skipping to save cost. "
              "Set FORCE_RUN=yes to override.")
        return

    client = Anthropic()
    print(f"Researching with {MODEL} "
          f"(max {MAX_WEB_SEARCHES} searches, {MAX_OUTPUT_TOKENS} output tokens)…")
    resp = client.messages.create(
        model=MODEL,
        max_tokens=MAX_OUTPUT_TOKENS,
        messages=[{"role": "user", "content": PROMPT}],
        tools=[{
            "type": "web_search_20250305",
            "name": "web_search",
            "max_uses": MAX_WEB_SEARCHES,
        }],
    )

    text = "".join(b.text for b in resp.content if getattr(b, "type", "") == "text")
    m = re.search(r"<json>\s*(\{.*\})\s*</json>", text, re.DOTALL)
    if not m:
        m = re.search(r"(\{.*\})", text, re.DOTALL)
    if not m:
        fail("No JSON found in the model response; keeping the existing camps.json.")
    try:
        data = json.loads(m.group(1))
    except json.JSONDecodeError as e:
        fail(f"Could not parse JSON ({e}); keeping the existing camps.json.")

    camps = []
    for i, c in enumerate(data.get("camps", []), start=1):
        cleaned = clean_camp(c, i)
        if cleaned:
            camps.append(cleaned)

    if len(camps) < MIN_CAMPS_TO_ACCEPT:
        fail(f"Only {len(camps)} usable camps returned "
             f"(< {MIN_CAMPS_TO_ACCEPT}); keeping the existing camps.json.")

    out = {"updatedAt": datetime.now(timezone.utc).isoformat(), "camps": camps}
    with open(OUTPUT_FILE, "w") as f:
        json.dump(out, f, indent=2, ensure_ascii=False)

    usage = getattr(resp, "usage", None)
    with open(REVIEW_FILE, "w") as f:
        f.write(f"# Camp directory update — {out['updatedAt'][:10]}\n\n")
        f.write(f"{len(camps)} camps published. A volunteer should skim this list "
                f"and spot-check a few websites/phone numbers.\n\n")
        for c in camps:
            f.write(f"- **{c['name']}** — {c['org']} · {c['city']}, {c['state']} · "
                    f"{c['dates'] or 'dates TBA'} · "
                    f"{'Free' if c['cost'] == 0 else '$%g' % c['cost']} · "
                    f"{c['website'] or 'no website captured'}\n")
        if usage:
            f.write(f"\n_Usage this run: {usage.input_tokens} input / "
                    f"{usage.output_tokens} output tokens._\n")

    print(f"Wrote {OUTPUT_FILE} with {len(camps)} camps. "
          f"Review summary in {REVIEW_FILE}.")


if __name__ == "__main__":
    main()
