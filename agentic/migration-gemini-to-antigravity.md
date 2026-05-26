# Gemini CLI Target Migration

AICGEN no longer generates Gemini CLI target files.

Use Antigravity for Google-side agentic coding profiles:

- Gemini CLI target output removed: `.gemini/instructions.md`
- Antigravity target output kept: `.agent/rules/*.md`
- Antigravity workflows enabled from `standard` profile and above: `.agent/workflows/*.md`

Existing `.gemini` folders are not deleted by generation. Users can remove them manually or run the clear command if they want to clean all AI configuration files.
