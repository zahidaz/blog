---
title: "Abusing the Companion BAL Exemption for Ads"
date: 2026-04-27
categories: [Android, Security]
tags: [android, bal, cdm, background-activity-launch]
---

Most Android abuse, in volume, is for ads. Adware [accounted for 62%
of mobile threat detections in 2025](https://securelist.com/mobile-threat-report-2025/119076/).
Most of those detections are the same thing: an app puts a full-screen
view on top of whatever the user is doing, long enough to register an
ad impression and get paid.

That full-screen view is an Android *Activity*, the framework's term
for a single UI screen. The app showing it is not in the foreground
when the launch happens. The capability to do this from the background
is called Background Activity Launch, or BAL, and it is the specific
thing Android has been trying to lock down. Without BAL restrictions,
any installed app could interrupt whatever the user is doing at any
time, with whatever screen it wanted. Android has tightened BAL across
five releases. The bypass below depends on what is left after all of
them.

## Six years of tightening

[Background Activity Launch (BAL)](https://developer.android.com/guide/components/activities/background-starts)
was introduced in Android 10 (API 29). The rule: an app in the
background cannot start an Activity. Before 10, background launches
were unrestricted; after 10, they became the exception.

Android 12 (API 31) closed several indirect launch paths. The
["notification trampoline" pattern](https://developer.android.com/about/versions/12/behavior-changes-12#notification-trampolines),
where a tap on a notification was quietly routed through a background
component that then called `startActivity`, was blocked. Foreground
services (a long-running background task with a persistent
notification) [could no longer be started from the background](https://developer.android.com/about/versions/12/behavior-changes-12#foreground-service-launch-restrictions).
And `PendingIntent` objects had to declare whether they were mutable.
A `PendingIntent` is a deferred action one app creates and hands to
another component, which fires it on the originator's behalf later; a
mutable handoff was a common way to launder activity-start permission.

Android 14 (API 34) closed the implicit-privilege paths. Earlier, an
app could effectively share its BAL permission with another app by
handing it a `PendingIntent`, or by letting that app connect to one of
its background services. In 14, neither transfer happens by default.
The granting app has to [set an explicit flag to allow it](https://developer.android.com/about/versions/14/behavior-changes-14#bal-restrictions).
Several older bypasses relied on the previous default-on behaviour,
and stopped working.

Android 15 [removed the last default-on case](https://developer.android.com/guide/components/activities/background-starts)
for `PendingIntent` creators. Android 16 added a developer-visible
runtime warning (via Android's StrictMode debug system) when a
background activity start happens.

After all of that, what remains is a list of explicit exceptions. The
official guide [enumerates thirteen of them](https://developer.android.com/guide/components/activities/background-starts).
Twelve are uncontroversial. An app can launch an Activity if it
already has a visible window, if a notification it issued was just
tapped, if the user granted it overlay permission (`SYSTEM_ALERT_WINDOW`),
if it is bound by the system as an accessibility or autofill service,
and a handful of similar cases. The thirteenth is the subject of this
post.

## The exemption

The thirteenth item, [quoted verbatim from the developer guide](https://developer.android.com/guide/components/activities/background-starts):

> The app is associated with a companion hardware device through the
> `CompanionDeviceManager` API. This API lets the app start activities
> in response to actions that the user performs on a paired device.

Companion Device Manager, or CDM, was added for legitimate use cases.
Wear OS watches need to open Activities on the phone when the user
taps a watch face. Smart glasses need to bring up controls or captions.
Fitness bands send notifications that, when acted on at the band, open
the phone's app. The user gesture occurs on the companion device; the
Activity has to appear on the phone. Without an exemption, that flow
cannot complete.

So CDM-associated apps are exempt from BAL. The exemption attaches to
the application UID, persists for the lifetime of the association, and
applies whether the companion device is connected, in range, or, as the
proof-of-concept demonstrates, present at all after the initial
pairing flow has completed. Internally, it is tracked as
`REASON_COMPANION_DEVICE_MANAGER` in
[`PowerExemptionManager`](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/os/PowerExemptionManager.java)
and consulted by
[`BackgroundActivityStartController`](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/core/java/com/android/server/wm/BackgroundActivityStartController.java)
in the window manager service.

## The minimum

A line in the app's manifest declaring use of the companion-device
feature:

```xml
<uses-feature android:name="android.software.companion_device_setup"
              android:required="false" />
```

A runtime call that asks the system to start a companion-device
association. The device filter can be empty, which makes any nearby
Bluetooth Low Energy (BLE) advertiser match:

```java
AssociationRequest request = new AssociationRequest.Builder()
        .addDeviceFilter(new BluetoothLeDeviceFilter.Builder().build())
        .setSingleDevice(false)
        .build();
cdm.associate(request, callback, null);
```

And one tap from the user, on a system dialog whose wording varies by
OEM and version but generally amounts to "choose a device to connect
to." The list shown contains whatever BLE is advertising in range:
earbuds, a neighbor's tracker, a smart scale, a passing fitness band.
Any selection works, because nothing was filtered.

After the tap, the UID is BAL-exempt for the lifetime of the
association.

## What the exemption gives

With the UID flagged, any background execution path can call
`startActivity()`:

- A scheduled `AlarmManager` wakeup at 3am
- A `BOOT_COMPLETED` receiver, run when the device finishes booting
- A `WorkManager` job that fires when the phone starts charging
- An FCM push notification from the app's backend
- A foreground service started for any plausible reason

The PoC uses a foreground service because it keeps the process alive
long enough for the delayed launch to fire. The same exemption serves
a push-triggered `BroadcastReceiver` just as well, with no notification
surfaced at all.

## The ad

Once an Activity can be launched from the background, what follows is
window configuration. A theme without a title bar. `FLAG_KEEP_SCREEN_ON`.
Immersive flags to hide the system bars. `android:excludeFromRecents="true"`
so the Activity does not appear in the task switcher. The Activity
reschedules itself on a timer, so dismissing it brings it back in five
seconds.

The platform never restricted any of those flags. BAL targets the call
to `startActivity()` from a background `Service`, and that call
succeeds because of one tap, weeks earlier, on a dialog the user does
not remember.

## What the user saw

A pairing dialog. The wording is a variation of "Allow this app to
access nearby devices?" with a list of advertised devices underneath.
The user picked their earbuds, or the first item in the list, or just
something to make the dialog go away. The dialog did not mention activities,
background behavior, or that the grant would, for the lifetime of the
association, place the app on a list of thirteen platform-wide
exceptions.

Dismissing the dialog does not record a "deny." There is no runtime
permission gating the call, and no system-enforced backoff. Any time
the app's Activity reaches the foreground, the dialog can be
re-presented. That includes user-initiated opens, notification taps,
and full-screen-intent notifications fired from a push.

After the user opens the app once, the app can start a foreground 
service while it is still in the foreground. That service survives the user backgrounding the app, 
swiping it from recents, or rebooting and re-launching it. From the service, the app
can launch Activities, each of which calls `associate()` and surfaces
the dialog again. On Android 13 and later, the user can deny the
notifications permission for the app and the foreground service keeps
running silently, with no visible notification and nothing in recents
to indicate the app is still alive. The dialog reappears minutes
or hours after the user thought the app was gone.

A freshly installed app, opened just once, can keep presenting the
dialog indefinitely until the user picks something. The user picks
eventually, because the alternative is uninstalling.

## The problem

The CDM exemption was added deliberately, for a class of legitimate
apps (Wear OS watches, smart glasses, automotive projection systems)
that need to launch UI in response to off-device events. The exemption
serves that audience correctly. The cost of joining that audience is
one BLE filter and one tap, and that is where the abuse begins.

A pairing grant currently unlocks background activity launch silently,
for the entire lifetime of the association. The dialog itself has no
deny-state and no rate limit, so an app can re-present it whenever its
Activity reaches the foreground, including from a foreground service
that survives the user closing the app. Safer behavior would scope the
unlock to the time the device is connected, treat a dismissed dialog
as a deny, and place the grant itself behind a second prompt that
names what is being granted.