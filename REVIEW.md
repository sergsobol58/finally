# Review

Scope: last commit (`HEAD~1..HEAD`), because the working tree had no uncommitted changes.

## Findings

1. **High - Plugin identity is inconsistent, so the enabled plugin entry can point at the wrong plugin.**

   `.claude/settings.json:6` enables `independendent-reviewer@serhii-tools`, and `.claude-plugin/marketplace.json:9` publishes the same misspelled plugin name. However, `independent-reviewer/.claude-plugin/plugin.json:2` declares the actual plugin name as `independent-reviewer`. Plugin installation/enabling normally keys off the marketplace/plugin manifest identity, so this mismatch can leave the plugin unavailable even though settings say it is enabled. Use one canonical name everywhere, likely `independent-reviewer`.

2. **High - The Stop hook runs a Codex review after every assistant stop.**

   `independent-reviewer/hooks/hooks.json:3-8` registers a project `Stop` hook with no matcher or guard, and the command mutates `REVIEW.md` by running `codex exec "Review changes since last commit and write results to a file named REVIEW.md"`. Once enabled, normal Claude conversations in this project can unexpectedly spawn another agent, consume tokens, slow down every response cycle, and overwrite review output even when no review was requested. Make the review an explicit slash command or add a guard script/matcher that exits unless the user asked for this workflow.

3. **Medium - `/doc-review` is empty.**

   `.claude/commands/doc-review.md` is a zero-byte file. If this is intended to be the explicit entry point for the review workflow, invoking it will not provide the needed prompt or instructions. Populate it with the intended command text, or remove it until it has behavior.

4. **Medium - The new reviewer agents do not match the advertised review workflow.**

   `.claude/agents/reviewer.md:8` reviews only `planning/PLAN.md` and writes `planning/REVIEW.md`. `.claude/agents/codex-reviewer.md:6-8` does the same via a nested `codex exec` command. That conflicts with the plugin description and hook command, which promise a review of all changes since the last commit written to root `REVIEW.md`. Users can get different review scopes and output paths depending on whether they invoke the agent or the hook/plugin. Align these prompts with the all-changes workflow, or rename them so they are clearly PLAN-specific.

## Notes

- The README and `planning/PLAN.md` documentation edits look internally consistent with the market data docs and existing backend paths.
- I did not run the backend test suite because the reviewed changes are documentation plus Claude/Codex plugin configuration, not backend runtime code.
