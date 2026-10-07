---
name: firebase-crashlytics
description: Access Firebase Crashlytics and investigate missing iOS dSYMs. Use for Firebase console access, missing-symbol alerts, UUID matching, and checking symbol uploads for Cue OS or another Firebase app.
---

# Firebase Crashlytics

Use an existing authenticated browser session to inspect Crashlytics. For Cue
OS, open [the iOS dashboard](https://console.firebase.google.com/project/cue-os/crashlytics/app/ios:ai.cueos.app/issues).
Confirm the project is `cue-os` and the app is `ai.cueos.app`; the app selector
shows **Cue OS**. On the workspace Mac mini x, the existing Firebase session
is in Chrome’s **Zhaolong** profile. Browser profile and Google account
selection are local machine facts: inspect the available browser sessions and the account displayed
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

## Sources

- [Firebase: readable Apple crash reports](https://firebase.google.com/docs/crashlytics/ios/get-deobfuscated-reports)
- [Apple: locating a missing debug symbol file](https://developer.apple.com/documentation/xcode/locating-a-missing-debug-symbol-file)
