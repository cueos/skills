---
name: firebase-crashlytics
description: Access Firebase Crashlytics, investigate missing iOS dSYMs, and read Android crash alerts. Use for Firebase console access, missing-symbol alerts, UUID matching, checking symbol uploads, and Android alerts with unreadable r8-map-id titles, for Cue OS or another Firebase app.
---

# Firebase Crashlytics

Use an existing authenticated browser session to inspect Crashlytics. For Cue
OS, open [the iOS dashboard](https://console.firebase.google.com/project/cue-os/crashlytics/app/ios:ai.cueos.app/issues)
or [the Android dashboard](https://console.firebase.google.com/project/cue-os/crashlytics/app/android:ai.cueos.app/issues).
Confirm the project is `cue-os` and the app is `ai.cueos.app`; the app selector
shows **Cue OS**. Browser profile and Google account selection are local
machine facts: inspect the available browser sessions and the account displayed
in the console. An authenticated session in a different profile may have
different project access.

## Read a missing-symbol alert

Open the **dSYMs** tab. Read the row for the alert's exact UUID, including its
version, status, and event count. A required missing dSYM blocks crash
processing; an optional one affects report quality. Use the rendered table
text when the accessibility tree omits UUIDs.

Match symbols by UUID, not just the version/build label. Several development
binaries can report the same version and build number.

## Find the original symbols

On a Mac with the original build, start with Spotlight:

```sh
mdfind "com_apple_xcode_dsym_uuids == $missing_uuid"
```

Check candidate bundles directly:

```sh
xcrun dwarfdump --uuid "$dsym"
```

When Spotlight returns no match, inspect the original build's DerivedData
`Build/Products` and an archive's `dSYMs` directory. In the Cue OS workspace,
check the build folders in the iOS checkout and its worktrees. For a
self-hosted GitHub build, locate the runner from its `Runner.Listener` process
and inspect its `_work` folder too; it is separate from the development
workspace. Actions can put the checkout under a second `ios` directory. Then
check another known build host when the original host is uncertain.

A copied Simulator app can identify a surviving binary, but it does not
include the original object files needed to recover its symbols.
Check the app executable and, for Xcode debug builds, its `.debug.dylib` too:

```sh
xcrun dwarfdump --uuid "$binary"
```

A rebuilt app can have a different UUID. Uploading its symbols does not repair
an older UUID. If the original symbols and recoverable build products cannot
be found, leave the original alert unresolved and report which hosts and
artifact stores were checked. Hiding its warning does not restore the crashes.

## Upload and verify

Read [Firebase's Apple symbol guide](https://firebase.google.com/docs/crashlytics/ios/get-deobfuscated-reports)
when configuring a build phase or investigating an upload failure. Cue OS's
release workflow already calls `ios/scripts/upload-dsyms.sh` with its archive
and DerivedData paths. Development builds also need dSYM generation and the
Firebase SDK's `Crashlytics/run` phase.

The SDK run phase validates synchronously and uploads in the background; a
successful build alone does not prove Firebase processed its symbols. Confirm
the exact UUID in the live dSYMs table after an upload, allowing processing to
finish, and retain the before/after status with the task's evidence.

For Cue OS, run the Swift package uploader from the iOS checkout with the
original matching dSYM. Set `derived_data` to the build's actual DerivedData
folder and `dsym` to the matched bundle:

```sh
"$derived_data/SourcePackages/checkouts/firebase-ios-sdk/Crashlytics/upload-symbols" \
  --debug -gsp chatapp/GoogleService-Info.plist -p ios "$dsym"
```

The debug output names each submitted UUID. Retain the relevant success or
failure lines; distinguish SDK submission from the console finishing crash
processing. If the SDK is absent from that cache, use a resolved Firebase
package checkout from a known iOS build on the same Mac.

The exercised route uses an existing Firebase browser login and a resolved
Firebase Swift package. First-time Google sign-in and manual browser ZIP
uploads have not been walked.

## Read an Android crash alert

Crashlytics emails such as "Trending stability issues for <date> - Android
<app id>" name the app, the version and build number, the crash and user
counts, and one console link per issue. Note whether each issue is a crash or
a non-fatal error: digests report both. These checks narrow an alert down even
when nobody can open the console yet.

- **Compare the build with what the store serves.** Development builds that
  keep a default build number, often `1`, report to the same Crashlytics app
  as store builds. But `1` is also a normal first release, so the number alone
  can't tell test devices from customers. Compare it with the store's release
  tracks (for Google Play, the Play Developer API's
  `applications/<package>/tracks/<track>/releases`), the app ID, and the
  project's build settings. Until the crashing build is found, treat test
  traffic as a suspicion, not a finding.
- **An `r8-map-id-<64 hex>` title suggests Crashlytics couldn't restore the
  real names for that event.** Release builds shorten class and method names
  with R8 to make the app smaller, and the build's mapping file lists the real
  names. The title alone doesn't say why the mapping wasn't used: an upload
  turned off for that build (`mappingFileUploadEnabled = false`), a failed
  upload, or a build whose mapping never left the machine it was built on.
- **Match a crash to its build.** Crash lines from the app's own code,
  including the libraries built into it, name the build's map ID, as in
  `at a.b.c(r8-map-id-<hex>:10)`; Android system lines keep normal file
  names. The same ID is inside the app file and in the mapping file's
  `# pg_map_id:` header line:

  ```sh
  unzip -p app.apk 'classes*.dex' | LC_ALL=C grep -a -o 'r8-map-id-[0-9a-f]\{64\}' | sort -u
  ```

  Emails cut the ID short: search for the part you can see, then confirm the
  full ID before using a mapping. Search build outputs and saved test
  evidence. A later build in the same folder replaces the earlier one, so a
  missing ID doesn't prove the build never existed.
- **Compare shortened names only within one mapping.** The same bug showed
  different shortened names in two builds. To match crashes across builds,
  compare the real class and method names from each build's own mapping, the
  user action, and whether the build includes the fix.
- **Look for earlier work.** Search the team's task records and recently
  merged changes for the exception name. A crash that test builds hit is
  often reproduced and fixed before the daily alert arrives.
- **Try the API when the console is out of reach.** Crashlytics has a read
  API at `firebasecrashlytics.googleapis.com` (`v1alpha` issues). It takes the
  Firebase app ID, not the package name. A 403 `SERVICE_DISABLED` means the API
  isn't turned on for the Google Cloud project behind your credentials.

This section was walked on 2026-11-19 and 2026-11-20 for one alert, without
console access: the build comparison (including the Play releases call), the
map ID search, the search for earlier work, and the Crashlytics API call, which
returned 403 `SERVICE_DISABLED`. Not walked: reading Android issues in the
console or through a working API, Firebase's MCP tools for Crashlytics, and
turning crash lines back into real names with R8's `retrace` tool.

## Sources

- [Firebase: readable Apple crash reports](https://firebase.google.com/docs/crashlytics/ios/get-deobfuscated-reports)
- [Apple: locating a missing debug symbol file](https://developer.apple.com/documentation/xcode/locating-a-missing-debug-symbol-file)
- [Firebase: readable Android crash reports](https://firebase.google.com/docs/crashlytics/android/get-deobfuscated-reports)
