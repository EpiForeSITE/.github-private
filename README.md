# EpiForeSITE Copilot Custom Agents[^original]

[^original]: This repository is based on the original template found at <https://github.com/docs/custom-agents-template>.

This private repository stores GitHub Copilot custom agent profiles used across the EpiForeSITE organization. Each agent lives as a standalone markdown file inside the `agents/` directory with front matter declaring its `name` and `description`, followed by clear, actionable operating guidelines.

## Current Agents

| Agent Name | Purpose |
|------------|---------|
| [`r_package_developer`](/agents/r_package_developer.agent.md) | Expert assistant for epidemiology‐focused R package development (code, docs, tests, scientific validation). |


## Creating a New Agent

Before adding a new agent profile, please review the section on [Contributing](#contributing) below.

1. Create a new file in `agents/` named `<agent-name>.agent.md` (use kebab_case or snake_case; keep it short and specific).
2. Add a YAML front matter block:
	 ```yaml
	 ---
	 name: short_machine_friendly_name
	 description: One sentence describing the agent's scope.
	 ---
	 ```
3. Write the agent guidance in imperative style ("You do…", "Always…", "Never…").
4. Cover: domain scope, priorities, coding standards, review/testing expectations, escalation/clarification guidance.
5. Keep instructions concrete; avoid ambiguous adjectives ("clean", "simple") without criteria.
6. If the agent should be temporarily disabled, comment out the front matter or wrap the body in HTML comments.
7. Open an Issue first describing: purpose, scope boundaries (what it should NOT do), example prompts, and any compliance or data constraints. Use the "New Agent Proposal" issue template if available.
8. After issue triage/approval, open a PR referencing the issue (e.g., "Closes #NN"). Include the agent file, plus any supporting assets. Keep the PR focused—one agent per PR.
9. Ensure at least one domain maintainer and one governance reviewer approve before merge.

Reference: GitHub Docs – [Creating custom agents](https://docs.github.com/copilot/how-tos/use-copilot-agents/coding-agent/create-custom-agents).

## Style & Content Guidelines

When authoring agent profiles:
- Be explicit about tooling (e.g., `roxygen2`, `tinytest`, `devtools::check()`).
- List decision heuristics (e.g., "Prefer vectorized operations over loops unless readability suffers").
- State compliance or organizational requirements (badges, documentation, testing levels, data privacy constraints).
- Encourage clarification for domain‐specific or uncertain scientific claims.

## Highlight: R Package Developer Agent

Key directives the `r_package_developer` agent follows:
- Optimize for modern, efficient R; minimize unnecessary dependencies.
- Use `roxygen2` for documentation; update docs with every function change (`devtools::document()`).
- Prefer `tinytest` unless a project already uses `testthat`.
- Include scientific validation tests (statistical properties, epidemiology relevance) beyond basic unit coverage.
- Maintain `NEWS.md`; keep rendered `README` synced (e.g., re-knit `README.Rmd` / `README.qmd`).
- Ensure packages pass `R CMD check` (or `devtools::check()`).
- Use public health & epidemiology–relevant examples in docs/vignettes.
- Add the ForeSITE badge to package READMEs if missing:
	```
	[![ForeSITE Group](https://github.com/EpiForeSITE/software/raw/e82ed88f75e0fe5c0a1a3b38c2b94509f122019c/docs/assets/foresite-software-badge.svg)](https://github.com/EpiForeSITE)
	```

## Reviewing and Updating Agents

Perform periodic (quarterly or pre–major cycle) reviews:
- Verify all external links and badge URLs.
- Confirm tooling practices reflect current organization standards.
- Sunset or archive agents no longer in active use (move to `agents/archive/` or add a deprecation note at top).

## Contributing

Workflow summary for new or changed agents:
- Always start with an Issue (proposal or change rationale) before a PR.
- Link PR to Issue (`Closes #issue_number`).
- Tag at least one domain maintainer and one governance reviewer.
- Provide example scenarios or prompts to validate intended behavior.
- Avoid bundling multiple agents or unrelated changes in one PR.

## Security & Privacy

Do not include proprietary data samples, unpublished results, or PII in agent instructions. Use synthetic, anonymized, or public examples only.

---
Questions or suggestions? Create an internal issue or start a discussion thread referencing the agent file path.
