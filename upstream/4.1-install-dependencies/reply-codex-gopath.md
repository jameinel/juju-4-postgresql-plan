With Go 1.27.3, the required version, `go env GOBIN` reports the
effective binary directory even when GOBIN is unset. I checked a
configured GOPATH with a space and multiple entries: it returned the
first entry's `bin` directory without truncating it.

The installer and static analysis now share that lookup, removing the
word-splitting fallback and the duplicate calculation. The existing
installer compatibility test and ShellCheck passed. Full static
analysis was not run locally.
