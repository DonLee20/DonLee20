import os
import requests
from datetime import date, timedelta
from pathlib import Path

USERNAME = "DonLee20"

YEAR = 2026

TOKEN = os.environ["GITHUB_TOKEN"]

GRAPHQL_URL = "https://api.github.com/graphql"

HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json",
}

START = f"{YEAR}-01-01T00:00:00Z"
END = f"{YEAR}-12-31T23:59:59Z"

QUERY = """
query($login: String!, $from: DateTime!, $to: DateTime!) {
  user(login: $login) {
    contributionsCollection(
      from: $from
      to: $to
    ) {
      contributionCalendar {
        totalContributions
        weeks {
          contributionDays {
            date
            contributionCount
            weekday
          }
        }
      }
    }
  }
}
"""

response = requests.post(
    GRAPHQL_URL,
    headers=HEADERS,
    json={
        "query": QUERY,
        "variables": {
            "login": USERNAME,
            "from": START,
            "to": END,
        },
    },
)

response.raise_for_status()

data = response.json()

if "errors" in data:
    raise RuntimeError(data["errors"])

calendar = (
    data["data"]["user"]
    ["contributionsCollection"]
    ["contributionCalendar"]
)

weeks = calendar["weeks"]

total = calendar["totalContributions"]

# ---------------------------------------------------------
# SVG SETTINGS
# ---------------------------------------------------------

CELL = 13
GAP = 3
STEP = CELL + GAP

LEFT = 45
TOP = 55

WIDTH = LEFT + (len(weeks) * STEP) + 20
HEIGHT = TOP + (7 * STEP) + 45

# GitHub-like contribution levels
LEVELS = [
    "#161b22",
    "#0e4429",
    "#006d32",
    "#26a641",
    "#39d353",
]

def get_level(count):
    if count == 0:
        return 0

    if count <= 2:
        return 1

    if count <= 5:
        return 2

    if count <= 9:
        return 3

    return 4


# ---------------------------------------------------------
# SVG
# ---------------------------------------------------------

svg = []

svg.append(
    f'''<svg xmlns="http://www.w3.org/2000/svg"
    width="100%"
    viewBox="0 0 {WIDTH} {HEIGHT}"
    role="img"
    aria-label="{USERNAME} GitHub contributions for {YEAR}">
'''
)

svg.append(
    f'''
<rect width="100%" height="100%" fill="#0d1117" rx="10"/>
'''
)

svg.append(
    f'''
<text
    x="{LEFT}"
    y="25"
    fill="#f0f6fc"
    font-size="16"
    font-family="Arial, sans-serif"
    font-weight="600">
    {YEAR} Contributions
</text>
'''
)

svg.append(
    f'''
<text
    x="{LEFT}"
    y="43"
    fill="#8b949e"
    font-size="11"
    font-family="Arial, sans-serif">
    {total:,} contributions in {YEAR}
</text>
'''
)

# ---------------------------------------------------------
# Day labels
# ---------------------------------------------------------

day_names = {
    1: "Mon",
    3: "Wed",
    5: "Fri",
}

for weekday, label in day_names.items():

    y = TOP + (weekday * STEP) + 10

    svg.append(
        f'''
<text
    x="5"
    y="{y}"
    fill="#8b949e"
    font-size="10"
    font-family="Arial, sans-serif">
    {label}
</text>
'''
    )


# ---------------------------------------------------------
# Contribution cells
# ---------------------------------------------------------

for week_index, week in enumerate(weeks):

    x = LEFT + week_index * STEP

    for day in week["contributionDays"]:

        weekday = day["weekday"]

        y = TOP + weekday * STEP

        count = day["contributionCount"]

        level = get_level(count)

        color = LEVELS[level]

        tooltip = f'{day["date"]}: {count} contributions'

        svg.append(
            f'''
<rect
    x="{x}"
    y="{y}"
    width="{CELL}"
    height="{CELL}"
    rx="2"
    fill="{color}">
    <title>{tooltip}</title>
</rect>
'''
        )


# ---------------------------------------------------------
# Legend
# ---------------------------------------------------------

legend_y = HEIGHT - 25

svg.append(
    f'''
<text
    x="{LEFT}"
    y="{legend_y + 10}"
    fill="#8b949e"
    font-size="10"
    font-family="Arial, sans-serif">
    Less
</text>
'''
)

legend_start = LEFT + 35

for i, color in enumerate(LEVELS):

    x = legend_start + i * STEP

    svg.append(
        f'''
<rect
    x="{x}"
    y="{legend_y}"
    width="{CELL}"
    height="{CELL}"
    rx="2"
    fill="{color}"/>
'''
    )

svg.append(
    f'''
<text
    x="{legend_start + (len(LEVELS) * STEP) + 5}"
    y="{legend_y + 10}"
    fill="#8b949e"
    font-size="10"
    font-family="Arial, sans-serif">
    More
</text>
'''
)

svg.append("</svg>")

# ---------------------------------------------------------
# Write file
# ---------------------------------------------------------

output = Path("assets/contributions.svg")

output.parent.mkdir(parents=True, exist_ok=True)

output.write_text(
    "\n".join(svg),
    encoding="utf-8"
)

print(
    f"Generated {output} with {total:,} contributions for {YEAR}."
)
