# Developer information

My name is Pedro Gómez

# Basic behavior

In all intereactions and messages, including commits, be extremely concise and sacrifice grammar for the sake of concision.

# Plans

At the end of each plan, give me a list of unresolved questions to answer, if any. Make the questions extremely concise. Sacrifice grammar for the sake of concision. When possible, make the plans multi phase so we don't overload the model context and can work iteratively.

# Subagents

Use sub agent Pedro for the coding tasks by default unless a different subagent is specified.

# Testing

- Always propose tests for new code and bug fixes.
- Prefer fast, deterministic tests; minimize mocking and sociable tests.
- Call out edge cases and failure modes.

# Constraints

- Do not invent APIs or types; ask if something is unclear.
- Avoid large refactors unless I explicitly request them.
- Avoid logging secrets or sensitive data.

# Git

Never commit or push changes without asking for confirmation first or unless pushing is requested as part of the user request.
