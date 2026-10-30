Thanks for the detailed reports. I reproduced the stale widget header against the public Android Calendar source at commit `61f983b58d0abfb39d1672e8ee6a730fea094486` (v2.29.0/332).

In that snapshot, the event refresh path calls `notifyAppWidgetViewDataChanged()` for the collection but does not update the parent `RemoteViews` containing the date header. That permits event rows to refresh while the header remains on the previous day. The manifest also listens for `ACTION_DATE_CHANGED`, which is not a reliable manifest-delivered implicit broadcast on modern Android.

I prepared a narrowly scoped patch that:

- updates the header whenever the existing event-list refresh runs;
- schedules a self-rearming alarm for the next local midnight, using an exact alarm when permitted and an allow-while-idle fallback otherwise;
- restores that alarm after reboot and recalculates it after relevant widget/date/time-zone lifecycle events; and
- adds four regression tests.

The four focused tests pass and `assembleDevDebug` succeeds. The tests cover the header/list update contract, zone-aware next-midnight calculation, and exact/fallback alarm API selection; they are not end-to-end event-sync tests. On a KVM-accelerated API 35 emulator, the unmodified widget remained on Sunday, November 1 after virtual system time crossed into Monday, November 2. In a separate patched run, the widget advanced from Saturday, October 31 to Sunday, November 1 at midnight and scheduled the following midnight alarm. Reboot rescheduling also passed. Time-zone receiver behavior was verified when the broadcast was delivered, although the command-line time-zone change used by the AVD did not itself emit a deliverable broadcast.

The emulator used the logged-out dev flavor, so production authentication and end-to-end event synchronization were not tested. This was not tested on a physical device. Android may also defer the inexact fallback under Doze or OEM background restrictions. Finally, this analysis applies only to the cited public snapshot and does not indicate whether Proton's current production code already differs or contains another fix.

I can provide the complete Git patch, regression tests, technical report, and recorded emulator evidence for review.
