# Memory Leak FAQ

## Purpose

Reference workflow for tracking down JVM memory leaks in the `2898-2025-CarrotChaos` robot codebase.

---

## 1. Quick Triage

- Watch Driver Station logs for “watchdog” or GC pauses.
- Add temporary heap telemetry (remove after debugging):

```kotlin
override fun robotPeriodic() {
    commandScheduler.run()
    val rt = Runtime.getRuntime()
    val usedMb = (rt.totalMemory() - rt.freeMemory()) / 1024.0 / 1024.0
    SmartDashboard.putNumber("Debug/HeapUsedMB", usedMb)
}
```

- If `Debug/HeapUsedMB` keeps climbing across mode changes (disable/enable), proceed to profiling.

---

## 2. Profiling Options

### Desktop Simulation (recommended first)

1. In `build.gradle`, set `includeDesktopSupport = true`.
1. Run with extra JVM diagnostics:

```bash
ORG_GRADLE_PROJECT_WPILIB_JAVA_OPTS="-Xmx512m -XX:+HeapDumpOnOutOfMemoryError -XX:+UnlockDiagnosticVMOptions -XX:NativeMemoryTracking=summary" ./gradlew simulateJava
```

1. Attach VisualVM, JProfiler, or Mission Control to the spawned `frc.robot.Main` JVM and monitor heap/live objects while reproducing the leak.

### On the roboRIO

1. `./gradlew deploy` and start the code.
1. SSH in: `ssh admin@roborio-2898-frc.local`.
1. List JVMs: `jcmd | grep frcUserProgram`.
1. Samples:
   - Histogram: `jcmd <pid> GC.class_histogram | head`
   - Heap dump: `jcmd <pid> GC.heap_dump /home/lvuser/leak.hprof`
1. Pull dumps with `scp` and inspect locally.

---

## 3. Usual Suspects in This Repo

- **`Drivetrain.periodic` telemetry**  
  Publishes pose, swerve states, and SmartDashboard numbers every cycle. If `swervelib` allocates new arrays per publish, NetworkTables may accumulate objects. Temporarily lower `SwerveDriveTelemetry.verbosity` or comment out the publishers to confirm.

- **`Tunnel.CarrotCounter` growth**  
  `carrots: MutableList<Carrot>` never shrinks unless `shootCarrot()` runs. If intake logic continually adds carrots without removing them, `Carrot` instances pile up. Watch the histogram for many `frc.robot.subsystems.Carrot` objects.

- **Vision listeners**  
  `Vision.listeners.add("UpdateOdometry", ...)` runs inside `Drivetrain` init. If `Drivetrain` or `Vision` reinitializes in tests, duplicate lambdas accumulate. Ensure listener registration happens once or provide a remove method.

- **SmartDashboard keys with dynamic names**  
  Most current keys are static (e.g., `"vX"`), but avoid generating unique keys per iteration. NetworkTables never deletes entries, so dynamic keys will leak.

---

## 4. Investigation Workflow

1. **Reproduce** the leak in sim or on the robot until memory clearly trends upward.
2. **Gather histograms** periodically (`jcmd GC.class_histogram`) to see the hottest classes.
3. **Map classes to code** (e.g., `Carrot`, `Transform2d`, `SwerveModuleState`). Search the repo for where they’re stored and verify they’re removed/reused.
4. **Add guards** once the culprit is known:
   - Bound list sizes (`while (carrots.size > N) carrots.removeFirst()`).
   - Reuse publishers/buffers instead of allocating per loop.
   - Deregister listeners when subsystems shut down.
5. **Verify fix** by rerunning the profiler. Heap trends should level off after the change.
6. **Remove temporary instrumentation** (heap dashboards, extra logs) before competition builds.

---

## 5. Useful Commands Cheat Sheet

| Purpose | Command |
| --- | --- |
| Start sim with diagnostics | `ORG_GRADLE_PROJECT_WPILIB_JAVA_OPTS="..." ./gradlew simulateJava` |
| List JVMs (roboRIO) | `jcmd` |
| Class histogram | `jcmd <pid> GC.class_histogram` |
| Heap dump | `jcmd <pid> GC.heap_dump /home/lvuser/leak.hprof` |
| Monitor GC | `jstat -gc <pid> 1000` |

---

## 6. When to Call It Fixed

- Heap usage stabilizes (oscillates) during multiple enable/disable cycles.
- `GC.class_histogram` shows object counts that plateau instead of increasing monotonically.
- NetworkTables memory footprint does not grow without bound when logged via `Shuffleboard`/`AdvantageScope`.

---

## 7. Next Steps If Still Stuck

- Share the histogram top entries and telemetry trends with the programming team.
- Capture a heap dump and analyze in Eclipse MAT “Leak Suspects” report.
- Double-check vendor libraries for known issues (e.g., `swervelib`, `PhotonVision`). Updating vendordeps can sometimes fix allocator leaks.

Good luck, and remember to remove any temporary diagnostics once the leak is resolved.

---

## Appendix: Beaverlib Hotspots

- **`beaverlib.misc.Symbol` global maps**  
  Every `Symbol` instance inserts its id into `symbolRegistry`/`descriptionMap` and nothing ever removes those entries. Code that creates Symbols per event will leak map entries. `Symbol.forDescription` is worse: it constructs new Symbols for existing ids, and each reconstruction re-registers another random id before overwriting it, so lookups leak too.  
  _Mitigation_: add a constructor that skips registration for existing ids, reuse Symbols instead of creating new ones, or replace the registry with weak references/UUIDs if uniqueness doesn’t need bookkeeping.

- **`BeaverPhotonVision.listeners` (via `BiSignal`)**  
  Listeners are stored in a plain `MutableMap` that only shrinks if callers remember to call `remove`. Any subsystem/command that registers a callback without removal leaks the lambda plus whatever it captures, and every leak also increases per-frame work because `periodic()` iterates all listeners each camera result.  
  _Mitigation_: provide lifecycle helpers (e.g., auto-remove on command end), enforce unique keys so re-adding replaces old listeners, or switch to weak references keyed to subsystem instances.

- **General tip**: when using Beaverlib utilities, audit any `mutableListOf`/`mutableMapOf` fields inside singletons (`object` declarations). They persist for the entire JVM lifetime, so add explicit size limits or cleanup hooks when storing runtime data.
