# docs: add agent guidance and use British English

Add `AGENTS.md` to help contributors and coding agents find the repository's
contribution requirements, code structure, invariants, validation commands
and documentation conventions. It points to CONTRIBUTING.md for the existing
contribution requirements rather than replacing them.

The documentation spelling checker uses British English. Make that requirement
explicit in the guidance, update the security page to use British spelling and
add the valid terms `cryptographic`, `Cryptographic`, `TLS` and `sanitisation`
to the spelling word list. This also fixes the existing spelling check failure.

## Functional QA

From the repository root, inspect the guidance:

```bash
grep -n -E 'British English|en_GB|behaviour' AGENTS.md
```

The output includes the British English requirement, the spelling checker's
`en_GB` configuration and examples of British spelling.

Build the documentation and inspect the rendered security page:

```bash
cd docs
make html
grep -n -E 'Argument sanitisation|Struct tag sanitisation|Transport Layer Security' \
    _build/explanation/security/index.html
make spelling
```

The rendered page contains both British-spelled headings and the TLS guidance.
The spelling check passes.
