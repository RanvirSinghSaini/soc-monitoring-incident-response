# Image Evidence Placement

Add reviewed screenshots to this folder only after they pass the sanitization checklist in the main README.

## Filename format

```text
EVID-XX-short-description.png
```

Use PNG for readable interface and terminal evidence. Keep the original aspect ratio and avoid excessive compression.

## Add an image to the main README

```markdown
![EVID-XX — Clear description of what the screenshot proves](docs/evidence/images/EVID-XX-short-description.png)
```

## Before committing

- Confirm the label matches the evidence register.
- Crop unrelated tabs, notifications, and desktop content.
- Redact secrets and personal data.
- Keep the time range, query, host, and relevant result visible.
- Verify that redaction does not change the conclusion.
- Check the staged file one final time with `git diff --cached`.

The absence of screenshots in this directory means the associated lab phase is not yet evidenced; it must not be marked complete.
