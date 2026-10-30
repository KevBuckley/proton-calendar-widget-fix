# Proton Calendar Android widget date-header fix

This release bundle contains a narrowly scoped Git patch, regression tests,
technical analysis, a proposed GitHub issue comment, and sanitized evidence for
the Android home-screen widget date-header defect.

## Tested source

- Repository: `https://github.com/ProtonMail/android-calendar`
- Commit: `61f983b58d0abfb39d1672e8ee6a730fea094486`
- Commit subject: `v2.29.0 (332)`
- Commit date: 2025-09-24
- Patch SHA-256: `f9ee2f7c18394b05f4ed215b9690540a83f0f1c82afd214cba1277bfedbd0422`

The patch was independently checked against a clean archive of that commit.
`git apply --check` passed, and applying it changed exactly these files:

- `app/src/main/java/me/proton/android/calendar/CalendarWidget.kt`
- `app/src/main/java/me/proton/android/calendar/ProtonCalendarBroadcastReceiver.kt`
- `app/src/test/java/me/proton/android/calendar/CalendarWidgetTest.kt`

The applied files byte-match the modified working tree used for the recorded
tests.

## Bug and fix

The existing event-list refresh invalidates the widget collection without
submitting new parent `RemoteViews` for the date header. Event rows can
therefore refresh while the header remains stale. The manifest-declared
`ACTION_DATE_CHANGED` action is also not a dependable trigger for an app
targeting modern Android because it is not one of Android's exempt implicit
broadcasts.

The patch:

1. partially updates the weekday and month/day header whenever the existing
   event-list refresh runs, then preserves the existing list invalidation;
2. schedules a zone-aware one-shot alarm for the next local midnight;
3. re-arms that alarm after delivery, normal widget updates, and boot;
4. cancels it when the final widget is removed; and
5. uses an inexact allow-while-idle fallback when exact alarm access is not
   available.

It does not alter event loading, package identity, product flavors,
authentication, or signing configuration.

## Verification

The retained JUnit XML records four tests, zero failures, zero errors, and zero
skips:

1. `event list refresh also refreshes header()`
2. `next midnight uses the current local time zone across daylight saving change()`
3. `midnight alarm is exact when access is available()`
4. `midnight alarm falls back when exact alarm access is unavailable()`

The second test verifies the calculated next midnight in
`America/New_York` on the date daylight saving time begins. It does not test a
zone whose UTC offset changes exactly at midnight, nor does it simulate alarm
delivery through a DST transition.

The recorded commands completed successfully:

```sh
ANDROID_HOME=/path/to/Android/Sdk ./gradlew \
  :app:testDevDebugUnitTest \
  --tests me.proton.android.calendar.CalendarWidgetTest

ANDROID_HOME=/path/to/Android/Sdk ./gradlew :app:assembleDevDebug
```

The retained APK artifact and successful emulator installation corroborate the
`devDebug` build result. The APK is intentionally excluded from this bundle.
The complete repository test suite was not run.

On a KVM-accelerated API 35 Google APIs x86_64 AVD, the baseline widget stayed
on Sunday, November 1 after virtual system time crossed into Monday, November
2. In a separate patched run, the widget advanced from Saturday, October 31 to
Sunday, November 1 at midnight and scheduled the following midnight alarm.
Reboot rescheduling passed. Time-zone receiver behavior was verified only when
the broadcast was explicitly delivered in the rooted test AVD.

## Apply and test

```sh
git clone https://github.com/ProtonMail/android-calendar.git
cd android-calendar
git checkout 61f983b58d0abfb39d1672e8ee6a730fea094486
git apply --check /path/to/widget-header-midnight.patch
git apply /path/to/widget-header-midnight.patch
```

Then run the two verification commands shown above. See
`TECHNICAL_BUG_REPORT.md` for emulator testing instructions and the full scope
discussion.

## Bundle contents

- `widget-header-midnight.patch` — verified complete Git patch and tests.
- `README.md` — release overview and audit summary.
- `TECHNICAL_BUG_REPORT.md` — detailed findings and independent test steps.
- `PROPOSED_GITHUB_COMMENT.md` — neutral issue-comment draft.
- `evidence/EVIDENCE_SUMMARY.md` — sanitized evidence index and provenance.
- `evidence/CalendarWidgetTest.xml` — sanitized JUnit result.
- `evidence/*.png` — four conclusive emulator screenshots.

No checkout, APK, signing material, credentials, raw device logs, build cache,
or unrelated files are included.

## Limitations

- The emulator used the logged-out `devDebug` flavor. Production
  authentication and real event synchronization were not tested.
- The first unit test verifies the extracted header/list update contract with
  mocks; it is not an end-to-end synchronized-event test.
- Exact alarm access and delivery remain subject to Android policy. The
  fallback alarm can be delayed by Doze, app standby, or OEM restrictions.
- Force-stopped apps remain subject to Android's normal restart restrictions.
- Testing used an emulator, not a physical device.
- Results apply to the cited public source commit and do not establish the
  behavior of Proton's current production build.

## Release audit checklist

- **PASS** — patch applies cleanly to the stated commit.
- **PASS** — patch changes only the three documented files.
- **PASS** — patch content matches the tested modified files.
- **PASS** — patch SHA-256 matches the documented checksum.
- **PASS** — four focused JUnit tests are recorded as passing.
- **PASS** — successful `devDebug` build is recorded and corroborated by the
  installed emulator artifact.
- **PASS** — patch and release bundle contain no detected credentials, tokens,
  signing keys, email addresses, or host-user paths.
- **PASS** — evidence is limited to sanitized, relevant files.
- **PASS WITH LIMITATIONS** — emulator validation covers midnight, reboot, and
  explicit time-zone broadcast handling, but not production authentication,
  real event synchronization, a physical device, or the full test suite.

