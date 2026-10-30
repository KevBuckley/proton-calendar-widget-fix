# Proton Calendar Android widget date-header refresh defect

## Summary

The Android home-screen widget can continue refreshing its event rows while its date header remains on the previous day. The defect was reproduced against the public Proton Calendar Android source snapshot at commit `61f983b58d0abfb39d1672e8ee6a730fea094486` (version `2.29.0`, build `332`).

The immediate verified cause is that the event-list refresh path invalidates only the collection view. It does not recompute or push the header `RemoteViews`. A second verified issue is that the widget relies in part on a manifest-declared `ACTION_DATE_CHANGED` receiver, although modern Android does not deliver that implicit broadcast to manifest receivers. Consequently, event data can refresh independently while no reliable operation advances the date header at local midnight.

The proposed narrow patch makes every event-list refresh also update the header and installs a self-rescheduling, local-midnight `AlarmManager` trigger. Four focused unit tests pass, the `devDebug` APK builds successfully, and an API 35/KVM emulator comparison showed the unmodified widget remaining stale across midnight while the patched widget advanced and scheduled the following midnight.

This report concerns the cited public source snapshot. It does not establish the state of Proton's current production code, and the patch has not been tested on a physical device.

## User-visible defect and reproduction

### Symptom

After the local date changes, the widget may display event data that has been refreshed while the large header still shows yesterday's weekday and date. Removing and re-adding the widget, reinstalling, or another full-widget update can repair the display temporarily. Reopening the app alone was not tested as a repair mechanism.

### Normal user reproduction

1. Sign in to Proton Calendar and place the Calendar widget on the Android home screen.
2. Leave the widget installed across local midnight, particularly while the app is backgrounded or the device is idle.
3. After midnight, allow an event-driven or scheduled event-list refresh to occur.
4. Observe that the event collection may update while the header can remain on the preceding day.

The timing is device- and launcher-dependent because periodic work, widget update intervals, idle modes, and OEM background policies are not exact.

### Deterministic emulator reproduction used here

The comparison used the dedicated `ProtonCalendarWidgetApi35` AVD, API 35 Google APIs x86_64, emulator `36.6.11.0`, KVM acceleration, and package `me.proton.android.calendar.dev`. With the emulator clock placed immediately before midnight, the widget was observed on both sides of the date boundary. The baseline APK retained the prior day's header; the patched APK advanced at midnight and armed its next alarm.

## Tested source and affected files

- Repository: `ProtonMail/android-calendar`
- Tested commit: `61f983b58d0abfb39d1672e8ee6a730fea094486`
- Public snapshot version: `2.29.0` (`332`)
- Patch file: `widget-header-midnight.patch`
- Recorded patch SHA-256: `f9ee2f7c18394b05f4ed215b9690540a83f0f1c82afd214cba1277bfedbd0422`

The patch is limited to these files:

1. `app/src/main/java/me/proton/android/calendar/CalendarWidget.kt`
   - Event-driven header refresh: patched lines 77-115.
   - midnight broadcast handling and lifecycle scheduling: patched lines 122-171.
   - next-midnight calculation, alarm scheduling, and cancellation: patched lines 256-332.
2. `app/src/main/java/me/proton/android/calendar/ProtonCalendarBroadcastReceiver.kt`
   - Re-arms the widget midnight alarm after boot: patched line 50.
3. `app/src/test/java/me/proton/android/calendar/CalendarWidgetTest.kt`
   - Four regression tests covering header refresh and midnight scheduling.

Related pre-patch behavior was traced through:

- `app/src/main/AndroidManifest.xml`, lines 174-189: manifest receiver registration includes `android.intent.action.DATE_CHANGED`.
- `app/src/main/res/xml/calendar_widget_info.xml`, line 5: periodic widget update interval.
- `app/src/main/java/me/proton/android/calendar/CalendarWidget.kt`, pre-patch lines 75-80 and 181-185: collection invalidation and header computation are separate paths.
- `app/src/main/java/me/proton/android/calendar/CalendarWidget.kt`, pre-patch lines 220-237: scheduled widget refresh work.
- `app/src/main/java/me/proton/android/calendar/common/worker/PeriodicCalendarWorker.kt`, lines 39-49 and 69-82: periodic widget refresh behavior and scheduling.
- `app/src/main/java/me/proton/android/calendar/presentation/account/AccountViewModel.kt`, lines 151-159: account cleanup cancels WorkManager work.
- `app/src/main/java/me/proton/android/calendar/presentation/main/MainActivity.kt`, lines 322-323: a widget refresh call during activity creation is commented out in this snapshot.

Line numbers above refer to the inspected commit and patched working tree; later source revisions may differ.

## Root-cause analysis

### Verified findings

1. **Collection refresh does not update the header.** The existing event-list refresh calls `AppWidgetManager.notifyAppWidgetViewDataChanged(...)`. This asks the launcher to refresh the `RemoteViewsService`-backed list, but it does not recreate or submit the parent widget's header views. The header text is calculated only during a full `RemoteViews` update. This directly explains how current event rows and a stale header can coexist.

2. **The manifest-declared date-change receiver is not a reliable modern-Android midnight trigger.** The snapshot declares `ACTION_DATE_CHANGED` in the manifest. Android 8.0 and later restrict manifest delivery of most implicit broadcasts; `ACTION_DATE_CHANGED` does not appear in Android's published list of exempt broadcasts. With target SDK 35, it cannot be treated as a dependable midnight refresh mechanism. Android does exempt `TIME_SET` and `TIMEZONE_CHANGED`, but those actions do not normally represent the daily midnight boundary. See [Android's implicit-broadcast exceptions](https://developer.android.com/develop/background-work/background-tasks/broadcasts/broadcast-exceptions).

3. **There was no dedicated next-local-midnight alarm.** Before the patch, the widget depended on lifecycle updates, event-driven refreshes, the provider update interval, WorkManager, and broadcasts. None constituted an exact, self-rescheduling local-midnight trigger for the header.

4. **The baseline and patched APKs behaved differently in comparable API 35 emulator runs.** In the baseline run, the unmodified header stayed on Sunday, November 1 after virtual system time reached Monday, November 2. In a separate patched run, the widget changed from Saturday, October 31 to Sunday, November 1 at midnight and scheduled the subsequent midnight alarm.

### Plausible contributing factors, not proven as the primary defect

- Doze, app standby, launcher scheduling, and OEM battery policies can defer provider updates, WorkManager jobs, and inexact alarms.
- `AccountViewModel.cleanUser()` cancels WorkManager jobs. Depending on subsequent process and lifecycle events, the hourly fallback may not be restored immediately.
- The commented widget refresh in `MainActivity.onCreate()` removes one possible activity-start repair opportunity. It is not an `onResume()` callback, and its historical intent was not established.
- Exceptions during worker setup or execution may be logged without producing a visible widget repair.
- Full and partial `RemoteViews` updates could theoretically race and briefly restore older content, but this was not established by static inspection or the recorded emulator run.

These hypotheses can increase the window in which the defect appears, but the missing header update and lack of a reliable midnight trigger are sufficient verified explanations.

## Patch design and why it works

### 1. Refresh the header whenever the event list is refreshed

`CalendarWidget.refreshEventList(...)` now derives the current local date, formats the weekday and date header strings, and sends a partial widget update before retaining the existing call to `notifyAppWidgetViewDataChanged(...)`.

This is the smallest direct correction for the split-refresh defect:

- It preserves the existing event-list refresh and adapter behavior.
- It does not force a complete widget reconstruction or change event loading.
- Any path that already refreshes event data also repairs the header using the current local date.
- A small helper centralizes assignment of `tv_header_text` and `tv_main_text`, keeping full and partial updates consistent.

### 2. Schedule and re-arm the next local midnight

The widget provider adds an app-specific, explicit `WIDGET_MIDNIGHT_UPDATE` action and schedules it for the next local day boundary. When received, the provider updates the widgets and schedules the following midnight.

The trigger time is calculated with:

`now.toLocalDate().plusDays(1).atStartOfDay(now.zone)`

Calculating the next day at the start of day in the active zone, rather than adding a fixed 24 hours, uses the zone rules for that local date. The focused test covers the New York date on which daylight saving time begins, but it does not simulate delivery across the offset change or test a zone whose offset changes exactly at midnight.

The alarm behavior is deliberately compatible with modern Android:

- If exact-alarm access is available, it uses an exact allow-while-idle alarm.
- Otherwise it falls back to an allow-while-idle inexact alarm instead of failing or requiring a new permission flow.
- `onUpdate()` and `onEnabled()` arm the alarm.
- `onDisabled()` cancels it when the final widget is removed.
- receipt of the midnight action re-arms the next occurrence.
- the existing boot receiver re-arms the alarm after `BOOT_COMPLETED`, because alarms do not survive reboot.
- timezone/date-triggered widget updates recalculate the next boundary in the current zone.

No event query/loading logic, authentication logic, application ID, product flavor, or signing configuration is changed. The existing collection invalidation remains in place after the new header update.

## Regression tests and build result

The focused test task completed successfully:

```text
ANDROID_HOME=/path/to/Android/Sdk ./gradlew \
  :app:testDevDebugUnitTest \
  --tests me.proton.android.calendar.CalendarWidgetTest

BUILD SUCCESSFUL
```

All four regression tests passed:

1. `event list refresh also refreshes header()`
2. `next midnight uses the current local time zone across daylight saving change()`
3. `midnight alarm is exact when access is available()`
4. `midnight alarm falls back when exact alarm access is unavailable()`

These are focused unit tests. The first uses mocks to verify the extracted helper's header-update, partial-update, and collection-invalidation calls. The time calculation test checks the resulting next midnight in `America/New_York` on the date DST begins; it is not an alarm-delivery test. The remaining tests verify selection of the exact and fallback `AlarmManager` APIs.

The application APK build also completed successfully:

```text
ANDROID_HOME=/path/to/Android/Sdk ./gradlew :app:assembleDevDebug

BUILD SUCCESSFUL
```

The resulting development application retained package identity `me.proton.android.calendar.dev`, version `2.29.0` (`332`). No package identity or signing changes were made.

## API 35 emulator results

### Before-and-after midnight comparison

| Build | Before midnight | After midnight | Result |
|---|---|---|---|
| Unmodified commit | Sunday, November 1 | System time reached Monday, November 2 at 00:00:12; widget still showed Sunday, November 1 | **Stale header reproduced** |
| Patched working tree | Saturday, October 31 | Widget showed Sunday, November 1 after crossing midnight | **Header advanced correctly** |

For the patched run, an `RTC_WAKEUP` alarm with action `WIDGET_MIDNIGHT_UPDATE` was present for November 1 midnight before the transition. After delivery, the next alarm was present for November 2 midnight. The baseline package replacement initially left a previously created pending intent visible, but the unmodified provider does not recognize the patch action; it was consumed before the tested November 2 transition and did not update the baseline header. Because aggressive virtual-clock changes can leave Pixel Launcher's own date surface temporarily stale, the conclusion uses the controlled system-time command/log, widget UI dump, and alarm state together rather than treating the launcher date label as the system-time authority.

The widget was logged out and displayed “Sign in to see your events.” This is sufficient to exercise the header `RemoteViews` and alarm lifecycle but not production event synchronization.

### Reboot

- The API 35 AVD completed a virtual-device reboot.
- The widget remained placed on the launcher.
- `ProtonCalendarBroadcastReceiver` received `BOOT_COMPLETED`.
- A next-midnight alarm was present afterward.
- The header matched the reboot-reset local date in the emulator's restored Eastern time zone.

This verifies reboot rescheduling in the emulator. It is not a physical-device test.

### Time-zone handling

Changing the emulator time zone with `cmd alarm set-timezone` changed the system zone but did not emit a deliverable time-zone broadcast in that setup. A root-only, emulator-scoped explicit `TIMEZONE_CHANGED` delivery caused the header to change from Friday, October 30 to Saturday, October 31 and rescheduled the alarm for midnight in UTC.

Therefore, receiver handling and rescheduling are verified when the broadcast is delivered. A complete user-facing time-zone change through the Android Settings UI was not recorded and remains an independent test recommendation.

### Alarm repair after reinstall

Reinstalling the patched APK repaired the stale baseline header to Monday, November 2 and armed the November 3 midnight alarm. This confirms that the standard widget update path both repairs the header and restores scheduling.

## Limitations and scope

- **Production authentication was not tested.** The `dev` flavor targets the Proton Black development environment and the test had no Atlas proxy token. Production login was neither available nor attempted.
- **Production event synchronization was not tested end to end.** The emulator widget remained logged out. Event-driven header behavior is covered by the focused unit test, while the visual emulator run validates header and alarm behavior.
- **The current production app may differ.** The test is tied to public commit `61f983b58d0abfb39d1672e8ee6a730fea094486`. Proton's closed/release branches and current store binaries may contain later changes. No conclusion is made about whether the currently distributed production version is fixed.
- **Android does not guarantee exact background execution in every state.** Exact delivery depends on exact-alarm access. The fallback alarm is intentionally inexact and may be deferred by Doze, app standby, OEM power management, or launcher behavior. The additional event-list refresh correction ensures that the next ordinary data refresh also repairs the header.
- **Force-stop behavior remains subject to Android rules.** A force-stopped application generally cannot receive alarms or broadcasts until the user launches it again.
- **No physical device was used.** Results are from a KVM-accelerated API 35 Google APIs x86_64 emulator only.
- **A separately installed dev APK is not a production-authentication test.** The `.dev` application ID allows side-by-side installation with the official app, but it uses a different app identity, signing context, storage, OAuth redirect identity, and development backend configuration. Changing only the package name would not safely preserve production authentication.

## Applying the patch

Start from the exact public commit and keep the patch file outside any unrelated working changes:

```bash
git clone https://github.com/ProtonMail/android-calendar.git
cd android-calendar
git checkout 61f983b58d0abfb39d1672e8ee6a730fea094486
git apply --check /path/to/widget-header-midnight.patch
git apply /path/to/widget-header-midnight.patch
```

Confirm that only the three files listed above differ, then run the focused tests and build:

```bash
ANDROID_HOME=/path/to/Android/Sdk ./gradlew \
  :app:testDevDebugUnitTest \
  --tests me.proton.android.calendar.CalendarWidgetTest

ANDROID_HOME=/path/to/Android/Sdk ./gradlew :app:assembleDevDebug
```

The patch in the tested checkout can be verified before application with:

```bash
sha256sum widget-header-midnight.patch
```

Expected digest:

```text
f9ee2f7c18394b05f4ed215b9690540a83f0f1c82afd214cba1277bfedbd0422
```

## Independent emulator verification

1. Use an API 35 Google APIs x86_64 AVD with hardware acceleration.
2. Install the patched `app-dev-debug.apk` on the emulator only.
3. Add the Proton Calendar widget to the launcher and record its app-widget ID.
4. Confirm via `dumpsys alarm` that `WIDGET_MIDNIGHT_UPDATE` is scheduled for the next local midnight.
5. On a disposable emulator snapshot with automatic time disabled, move the virtual clock to shortly before midnight.
6. Observe the widget across the boundary and confirm both the new header date and a newly scheduled following-midnight alarm.
7. Reboot the AVD; confirm the widget persists, `BOOT_COMPLETED` reaches the receiver, and the alarm is restored.
8. Change the time zone through Android Settings when possible; verify that the displayed local date and next alarm use the new zone.
9. For the baseline comparison, use a clean snapshot or reinstall the unmodified APK from the same commit, ensuring that no pending alarm created by the patched version remains relevant to the transition under test.

Clock manipulation, protected-broadcast injection, and root commands should be confined to a disposable emulator. They should not be used on a personal or production device.

## Evidence provenance

The conclusions above were assembled from the existing repository diff, patch, retained JUnit result, prior build record, and the sanitized evidence included under `evidence/`. No additional build or emulator run was performed while preparing or auditing this publication report, and no original evidence was modified.

The release audit also verified that the patch applies cleanly to commit `61f983b58d0abfb39d1672e8ee6a730fea094486`, changes only the three listed files, produces files byte-identical to the tested working tree, and has SHA-256 `f9ee2f7c18394b05f4ed215b9690540a83f0f1c82afd214cba1277bfedbd0422`.

