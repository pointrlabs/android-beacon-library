# Pointr's build of AltBeacon

AltBeacon [android-beacon-library](https://github.com/AltBeacon/android-beacon-library) from its master branch at
`72d2c6b` (2.21.1 and 18 commits, 27 Oct 2026), on the branch `pointr` of Pointr's fork
[pointrlabs/android-beacon-library](https://github.com/pointrlabs/android-beacon-library). core-android-sdk has it as
the git submodule `android-beacon-library`, builds it as the module `:altbeacon` (`lib/`) and releases it as
`com.pointrlabs:altbeacon` (see `../altbeacon-release.gradle`). `:PointrSDK` depends on it instead of
`org.altbeacon:android-beacon-library`.

## Working on it

Remotes in the submodule: `origin` is Pointr's fork, `upstream` is AltBeacon's repo. Only AltBeacon's maintainers
push to it, so changes reach it as pull requests from the fork.

- A change for every AltBeacon user: branch from `upstream/master`, commit, push the branch to `origin` and open a
  pull request: `gh pr create -R AltBeacon/android-beacon-library --head pointrlabs:<branch>`. Pick it onto `pointr`
  too, so core-android-sdk need not wait for the merge.
- A change for Pointr only: commit it on `pointr` and push.
- After AltBeacon merges or releases: `git fetch upstream`, rebase `pointr` on `upstream/master`, push it with
  `--force-with-lease`, and commit the new submodule commit in core-android-sdk.
- In core-android-sdk, `git submodule update --init` checks out the commit core-android-sdk records. Commit on
  `pointr` (not on that detached commit), push, then commit the submodule change in core-android-sdk.

Master already restarts a long running scan every 4 minutes, before Android's Bluetooth scan timeout
(`scan_timeout_millis`, 5 minutes on Pixel and Samsung phones), and uses the background between scan period of the
settings (#1221). Pointr's changes, one commit each:

- For AltBeacon too, each on its own branch of the fork, to be sent as a pull request:
  - `fix-nullable-requested-permissions-flags`: `PermissionsInspector` reads `requestedPermissionsFlags` as nullable,
    as compile SDK 35 and later declare it.
  - `fix-bluetooth-scan-permission-check`: `PermissionsInspector` checks `BLUETOOTH_SCAN` for apps that target
    Android 12 or later on Android 12 or later, and `BLUETOOTH` and `BLUETOOTH_ADMIN` otherwise. Master asks for those
    two in every case, so in a debug build ranging threw "AndroidManifest.xml is misconfigured for beacon scanning" in
    apps that declare them with `maxSdkVersion="30"`, as `:PointrSDK` does.
  - `optional-foreground-service-notification`: the foreground service strategy needs no notification:
    `Settings.ForegroundServiceScanStrategy(notificationId)` and
    `BeaconManager.enableForegroundServiceScanning(null, notificationId)`. `BeaconService` then shows the notification
    the app already shows with that id, or a plain one with the app's name and icon on a channel of its own
    (`altbeacon_foreground_service_channel_name`). It needs the fine location permission only, While In Use is
    enough.
- For Pointr only: `lib/build.gradle` fits core-android-sdk: compile SDK 37, min SDK 28, Java 11 and Kotlin 2.0 like
  `:PointrSDK`, AltBeacon's own dependencies, no Dokka or Maven Central publishing. AltBeacon's own tests are not
  built here: run them with AltBeacon's build, `./gradlew test` on a branch from `upstream/master`.
- Sent to AltBeacon as #1257 (branch `android-17-scan-allowance`), on the branch `RD-1589-altbeacon-background-scanning`
  here: Android 17's scan allowance
  (`CycledLeScannerForLollipop`). The Pixel 9's Bluetooth stack gives an app 6 minutes of scanning an hour while it
  is not on screen, so on Android 17 and later the foreground service scans off screen with a low power batch scan,
  which costs 10% of that time and lasts the hour. It fetches the results every second, and isn't restarted every 4
  minutes: batch scans get no scan timeout. It starts the scan again after a gap of more than 25 s, at most once a
  minute, for a Bluetooth stack bug. Found with SimpleBeaconScanner.
