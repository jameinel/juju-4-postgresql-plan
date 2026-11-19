The immediate broken-connection test can assert that the connection is
disconnected before the worker records that state. Its second send only
proves that the worker received the first notification, not that it
processed it.

Model the connection's broken notification as a closed channel and wait
for unit-presence cleanup before asserting the disconnected state. Keep
coverage of a break immediately after the initial `IsBroken` check and
the normal later-break path. The cleanup callback is reached after the
worker records the disconnected state. In the immediate-break case, the
mock closes the channel as it returns the initial healthy result.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~Integration tests~
- [ ] ~doc.go added or updated in changed packages~

## QA steps

Use a disposable checkout with Go installed:

```sh
git clone https://github.com/jameinel/juju.git
cd juju
git switch --detach d380e5e4fba4c9918a0774638ffa3138aeece73e
```

Save and apply this temporary diff to delay the worker after it receives
the notification:

```sh
cat > /tmp/juju-controller-presence-delay.patch <<'PATCH'
diff --git a/internal/worker/controllerpresence/worker.go b/internal/worker/controllerpresence/worker.go
--- a/internal/worker/controllerpresence/worker.go
+++ b/internal/worker/controllerpresence/worker.go
@@ -245,7 +245,8 @@ func (w *connectionTracker) loop() error {
 		select {
 		case <-w.tomb.Dying():
 			return nil
 		case <-c.Broken():
+			time.Sleep(500 * time.Millisecond)
 			w.connected.Store(false)
 			return w.handleBrokenConnection(ctx)
 		}
PATCH
git apply /tmp/juju-controller-presence-delay.patch
go test -race -count=1 -timeout=30s \
  -run 'TestWorker/TestNewConnectionTrackerBrokenImmediately$' \
  ./internal/worker/controllerpresence
```

The original test fails because `connected` is still true. Keep the delay
in place, switch to the proposed branch, and run both break scenarios:

```sh
git switch 4.1-controller-presence-test
go test -race -count=5 -timeout=30s \
  -run 'TestWorker/TestNewConnectionTrackerBroken($|Immediately$)' \
  ./internal/worker/controllerpresence
```

Both break scenarios pass despite the delay, including the presence-cleanup
expectations. Remove the temporary delay after the check:

```sh
git apply -R /tmp/juju-controller-presence-delay.patch
```

## Documentation changes

None; this changes tests only.

## Links

~None.~
