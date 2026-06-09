#!/usr/bin/env node
/**
 * Regenerates the auto-updated "Latest Repositories" block in README.md.
 * Pulls live repos from the GitHub API, skips forks and the profile repo,
 * and writes a table between the AUTO-GENERATED markers.
 *
 * Runs in GitHub Actions with GITHUB_TOKEN provided automatically.
 */

const fs = require("fs");

const USER = "prometheus-18";
const README = "README.md";
const START = "<!-- AUTO-GENERATED:START -->";
const END = "<!-- AUTO-GENERATED:END -->";
const MAX_REPOS = 8; // how many recently-pushed repos to show

async function main() {
  const res = await fetch(
    `https://api.github.com/users/${USER}/repos?per_page=100&sort=pushed&type=owner`,
    {
      headers: {
        "Accept": "application/vnd.github+json",
        "User-Agent": "readme-updater",
        ...(process.env.GITHUB_TOKEN
          ? { Authorization: `Bearer ${process.env.GITHUB_TOKEN}` }
          : {}),
      },
    }
  );

  if (!res.ok) {
    throw new Error(`GitHub API error: ${res.status} ${await res.text()}`);
  }

  const repos = await res.json();

  const visible = repos
    .filter((r) => !r.fork && !r.archived && r.name !== USER)
    .slice(0, MAX_REPOS);

  const rows = visible
    .map((r) => {
      const desc = (r.description || "—").replace(/\|/g, "\\|");
      const lang = r.language || "—";
      const updated = new Date(r.pushed_at).toISOString().slice(0, 10);
      return `| [${r.name}](${r.html_url}) | ${desc} | ${lang} | ${updated} |`;
    })
    .join("\n");

  const block = [
    "### 🔄 Latest Repositories",
    "",
    "_Auto-updated from GitHub on every push._",
    "",
    "| Repo | Description | Language | Last push |",
    "| --- | --- | --- | --- |",
    rows,
  ].join("\n");

  let readme = fs.readFileSync(README, "utf8");
  const pattern = new RegExp(`${START}[\\s\\S]*?${END}`);
  const replacement = `${START}\n${block}\n${END}`;

  if (!pattern.test(readme)) {
    throw new Error("AUTO-GENERATED markers not found in README.md");
  }

  const updated = readme.replace(pattern, replacement);

  if (updated !== readme) {
    fs.writeFileSync(README, updated);
    console.log(`Updated README with ${visible.length} repos.`);
  } else {
    console.log("No changes needed.");
  }
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
