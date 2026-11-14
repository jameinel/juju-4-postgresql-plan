The installer checks the selected `golangci-lint` against the same 2.14.x
rule used by static analysis. It fails with a hint instead of replacing an
incompatible binary. The regression test covers older and newer versions,
invalid or failing version output, and a compatible GOBIN binary shadowed
by an incompatible one on PATH.

The focused compatibility test and shell checks passed locally. Full static
analysis was not run.
