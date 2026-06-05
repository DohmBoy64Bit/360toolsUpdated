# Speed Fix: VdSwap Frame Limiter

## Problem
The game runs at half speed (~30 FPS game logic) despite reporting ~70 FPS in the overlay.

## Root Cause Found

### VdSwap Sleep(16) + Windows Timer Granularity

The VdSwap stub (frame swap function) often has a `Sleep(16)` call to cap the frame rate at ~60 FPS. However, Windows' default timer resolution is **15.6ms**, which means `Sleep(16)` actually sleeps for **~31ms** (rounds up to the next timer tick). This alone causes the game to run at ~32 FPS.

Additionally, the SDK's vsync worker thread independently fires `MarkVblank()` every 16ms. If the game waits on vblank interrupts AND VdSwap sleeps, you get double-throttling: ~16ms vblank wait + ~31ms Sleep = ~47ms per frame = ~21 FPS.

**Fix:** Replace `Sleep(16)` with a precise frame limiter using `QueryPerformanceCounter`:

```cpp
// In your VdSwap stub implementation:
static LARGE_INTEGER s_freq = {}, s_last = {};
if (s_freq.QuadPart == 0) {
    QueryPerformanceFrequency(&s_freq);
    QueryPerformanceCounter(&s_last);
}
const int64_t target_us = 16667; // 16.667ms = 60 Hz
LARGE_INTEGER now;
for (;;) {
    // pump Win32 messages here too if necessary
    QueryPerformanceCounter(&now);
    int64_t elapsed_us = (now.QuadPart - s_last.QuadPart) * 1000000 / s_freq.QuadPart;
    if (elapsed_us >= target_us) break;
    if (elapsed_us < target_us - 2000)
        Sleep(1);  // coarse yield if >2ms remain
}
QueryPerformanceCounter(&s_last);
```

This gives precise 60 Hz frame pacing regardless of Windows timer resolution.

## Timebase Scaling (No longer needed!)

Older recompilers like XenonRecomp translated the PowerPC `mftb` (move from timebase) instruction to the x86 `__rdtsc()` intrinsic. Since the Xbox 360 timebase runs at **49.875 MHz** and modern PC TSC counters run at **~3-4 GHz**, this caused massive timing inaccuracies unless manually overridden.

**The modern ReXGlue SDK handles timebase scaling natively.** When `rexglue codegen` encounters an `mftb` instruction, it automatically translates it to a call to `QueryGuestTickCount()`, which returns properly scaled guest ticks at 50 MHz. No manual overrides or patches are needed.
