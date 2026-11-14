The installer probes the selected `yq` for JSON input and JSON output
syntax and fails with a hint if either check fails. It leaves existing
binaries untouched. The regression test covers both unsupported forms
and commands that print the expected output but fail.

The focused compatibility test and shell checks passed locally. The probes
cover representative syntax rather than every expression used by the tests.
