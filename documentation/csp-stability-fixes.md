# Wine 11.4 CSP / TourBox stability fixes

This branch is based on the upstream `wine-11.4` tag, commit
`cc893ef9cb17b994bfd1f1a1f7355be55e615623`. It contains two changes validated
with CLIP STUDIO PAINT and TourBox Console on a Plasma Wayland/Xwayland
session, plus these validation notes.

## Serial polling callback lifetime

In `server/serial.c`, `async_wait_timeout()` used to wake the serial wait
queue before rearming its timeout. Waking a client that has just exited can
synchronously complete the async request through `thread_apc_destroy()` and
`async_set_result()`, which calls `free_wait_req()`. The callback then wrote
the next timeout through the freed request.

The callback now rearms before waking the queue. Completion can cancel the
new timeout, and the callback does not access the request after wakeup.

Validation used a separate Wine prefix and a pseudoterminal, without opening
the physical TourBox device. A small Win32 client opened the serial port and
issued an overlapped `WaitCommEvent()`. With an AddressSanitizer server, GDB
paused at the serial timeout callback while the test client was terminated;
after its Unix process had exited, the callback resumed.

- Before: ASan reported a heap-use-after-free write in `async_wait_timeout()`,
  with the free originating from `free_wait_req()` during queue wakeup.
- After: the same sequence completed without an ASan error; the server
  remained alive and successfully ran another client.
- A non-ASan build also passed 1,000 iterations of serial waits, cancellation,
  and worker-thread termination.

The original production core showed a corrupted timeout list, but did not
include allocation/free history. The controlled reproduction establishes a
real defect in this callback; it does not prove that it was the sole source
of that earlier corruption.

## Stale RandR output snapshots

In `dlls/winex11.drv/xrandr.c`, an output ID can become invalid between
obtaining screen resources and calling `XRRGetOutputInfo()`. An unhandled
`BadRROutput` is fatal to the requesting process. Losing Wine's explorer
process also destroys its shared X11 resources, including the cursor-clipping
window cached by other applications.

Output-info calls now use Wine's existing expected-error mechanism. If the
request fails, the helper returns `NULL` to the existing failure paths.
RandR support and monitor enumeration remain enabled.

Validation used a separate Xvfb display and a test-only dynamic-linker audit
library that replaced the first output ID passed to `XRRGetOutputInfo()` with
an invalid one. Both sides used the same upstream source and build settings.

- Before: Wine reported `BadRROutput / RRGetOutputInfo`; the test client could
  not create its windows and returned 2.
- After: the client returned 0 and completed 837 focus/cursor-clipping cycles.

This change handles the stale-output query. It does not recreate every Wine
desktop resource if explorer exits for an unrelated reason.

## Build and real-session acceptance

The release components were built in a separate build directory with:

```sh
../wine/configure --enable-win64 --without-wayland
make -j8 server/wineserver dlls/winex11.drv/winex11.so
```

The ASan server used `-fsanitize=address -fno-omit-frame-pointer -O1 -g` and
`ASAN_OPTIONS=detect_leaks=0` for the test runs. No fault-injection hooks were
added to either production source file.

The modified server and X11 module were installed in a dedicated copy of the
existing Wine 11.4 runtime. The applications' existing Wine prefix, display
recovery helpers, clipboard integration, and TourBox shortcut settings were
preserved. The shared original runtime was retained for rollback.

On 2026-10-28, the user verified drawing and pen pressure on a blank canvas,
switching to another VT and back to Plasma, switching to TourBox, and returning
to CSP. The reported result was that everything worked normally. Afterwards,
CSP, TourBox, explorer, and wineserver retained their process IDs, and the
new TourBox session log contained no `BadWindow` or `BadRROutput` failures.
Both TourBox hardware handshake rounds and the CSP companion connection were
also confirmed.

This is acceptance for the tested Wine 11.4 session and switching sequence.
A fresh login and extended use were not covered. The original core lacked
sufficient history to establish the complete incident chain.
