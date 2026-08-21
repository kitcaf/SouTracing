<!-- ARIS-COPILOT:BEGIN -->
## ARIS Copilot CLI Skill Scope
ARIS mainline skills installed in this project for GitHub Copilot CLI.
Managed entries: 85
Manifest: `.aris/installed-skills-copilot.txt`
ARIS repo root: `/d/project/research/SouTracing/aris_repo`
Project skill path: `.github/skills/<skill-name>`
For ARIS workflows, prefer the project-local skills under `.github/skills/`.
When a skill needs ARIS helper scripts, resolve the repo root from the manifest or set it explicitly:
`ARIS_REPO=$(awk -F'\t' '$1=="repo_root"{print $2; exit}' "/d/project/research/SouTracing/.aris/installed-skills-copilot.txt")`
Do not edit or delete symlinked skills in place; update upstream or rerun:
`bash /d/project/research/SouTracing/aris_repo/tools/install_aris_copilot.sh "/d/project/research/SouTracing" --reconcile`
For copied installs, use:
`bash /d/project/research/SouTracing/aris_repo/tools/smart_update_copilot.sh --project "/d/project/research/SouTracing"`
<!-- ARIS-COPILOT:END -->
