# Contributing

This repository is maintained as a generated mirror.

## Preferred change path

1. Open an issue or pull request describing the bug, migration case, or
   validation gap.
2. Keep changes focused on migration behavior, validation behavior, or public
   documentation.
3. Include a before/after sample when changing a migration transform.
4. Run the validator before submitting:

```bash
python3 tools/migration_support/validate_names.py --scan-dir .codex/skills
```

Large upstream syncs are reviewed manually before publication. Do not include
private project details, local filesystem paths, customer data, deployment
names, or internal incident narratives in examples or tests.

