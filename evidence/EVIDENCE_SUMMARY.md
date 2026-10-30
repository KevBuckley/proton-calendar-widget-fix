# Sanitized verification evidence

This directory contains a minimal public evidence set derived from the
preserved test and emulator artifacts. The original files were not modified.
Raw logcat captures and full UI hierarchy dumps were excluded because they are
large, contain unrelated Android diagnostics, and are unnecessary to review
the conclusions below.

## Environment

- Source commit: `61f983b58d0abfb39d1672e8ee6a730fea094486`
- AVD: `ProtonCalendarWidgetApi35`
- Android: API 35, Google APIs, x86_64
- Emulator: `36.6.11.0`
- Acceleration: KVM (`-accel on`)
- Package: `me.proton.android.calendar.dev`
- Widget state: logged out (`Sign in to see your events`)
- Physical device: none connected or accessed

## Focused unit tests

`CalendarWidgetTest.xml` is a copy of Gradle's retained JUnit result with only
the host name replaced by `sanitized-host`. It records:

- tests: 4
- failures: 0
- errors: 0
- skipped: 0

The tests verify the extracted header/list update call contract, local
next-midnight calculation on the New York DST-start date, and exact/fallback
alarm API selection. They do not verify production authentication, event
synchronization, launcher rendering, boot delivery, or time-zone broadcast
delivery.

## Patched midnight run

The AVD was moved to 2026-10-31 23:59:40 UTC. Before midnight the widget showed
Saturday, October 31, and `dumpsys alarm` showed an exact `RTC_WAKEUP` for
2026-11-01 00:00:00 with action
`me.proton.android.calendar.action.WIDGET_MIDNIGHT_UPDATE`.

At midnight the retained raw log records the app-widget service updating widget
ID 2, including a partial widget update and collection invalidation. The widget
then showed Sunday, November 1, and the next midnight alarm was scheduled for
2026-11-02 00:00:00.

`patched-after-midnight.png` shows the resulting widget. Pixel Launcher's own
date surface lagged after the aggressive virtual-clock change, so it must not
be used as the system-time authority; system command/log output, the widget UI
dump, and alarm state were evaluated together.

## Unmodified comparison

The unmodified commit was built in a separate temporary checkout and installed
over the emulator-only dev package. At virtual system time 2026-11-01 23:59 UTC
the baseline widget showed Sunday, November 1. After virtual system time reached
Monday, November 2 at 00:00:12 UTC, it still showed Sunday, November 1.

`baseline-after-midnight.png` shows the stale header. As above, the launcher
date surface is not treated as the system-time authority after clock rebasing.
The baseline and patched observations are separate controlled runs on the same
API 35 AVD, not simultaneous tests and not the same calendar transition.

## Reboot and restoration

After an AVD reboot, the widget binding persisted, the original log recorded
`BOOT_COMPLETED` delivery to `ProtonCalendarBroadcastReceiver`, and an exact
next-midnight widget alarm was present. `patched-after-reboot.png` records the
post-reboot widget. Reinstalling the patched APK after the baseline comparison
updated the widget to Monday, November 2 and armed the November 3 alarm;
`patched-restored.png` records that state.

## Time-zone scope

`cmd alarm set-timezone` changed the emulator framework zone but did not emit a
deliverable `TIMEZONE_CHANGED` broadcast in this setup. A root-only explicit
broadcast to the provider caused the header to recalculate and the next alarm
to be scheduled in UTC. This verifies receiver behavior when delivery occurs;
it does not verify the complete Android Settings UI flow or physical-device
behavior.

## Authentication and event-list limitation

The `devDebug` flavor targets `proton.black` and was built without an Atlas
proxy token. Production authentication and event synchronization were not
attempted. The emulator therefore validates header rendering and alarm/widget
lifecycle behavior in the supported logged-out state, while the event-driven
header/list call contract is covered only by the focused mock-based unit test.

## Evidence checksums

The checksums below are for files in this sanitized directory:

```text
CalendarWidgetTest.xml       ee442cd258d6a99c3212d78ef4f7765fe1ac05956bd41dcb02d3edf829b37ef6
baseline-after-midnight.png  e1d893550e977d8592d3a9e6a735b7d80eadae7c4ea8353eebd500076e9f20ee
patched-after-midnight.png   50d4ac5606c0796c4f924018417f59f5ecccec78713905f96a0ea92fa7b2461d
patched-after-reboot.png     1a4d7618ac4db101a3832177589464c5e7c27de847490555c99f848a1622169f
patched-restored.png         8610546544fcaa6af6dcb543e0903995831184bbd053ce070ecdab9ac28f56c8
```

