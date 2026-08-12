# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

This is Corey's personal fork of [Subler](https://bitbucket.org/galad87/subler/ or
https://github.com/SublerApp/Subler), a macOS app for muxing and tagging mp4 files (chapters,
subtitles, closed captions, and metadata from TheMovieDB, TheTVDB, and the iTunes Store).

Repo: `https://github.com/KlassyCoder/Subler.git` (fork, not upstream). Remote is `origin`, branch
`main`.

The fork exists to carry fixes that either aren't upstreamed yet or are specific to how Corey uses
the app, and to produce Developer ID-signed, notarized builds independent of upstream's release
cadence. Fork-specific changes are logged in README.md under "Fork changes (KlassyCoder)" - update
that section (not just this file) whenever you land a behavioral fix here.

## Versioning

`MARKETING_VERSION` in `Subler.xcodeproj/project.pbxproj` carries a `.ck.N` suffix appended to
whatever upstream version it was forked from, e.g. `1.9.1.ck.1` off upstream `1.9.1`. This
distinguishes fork builds from upstream ones sharing the same base version. Bump the `.ck.N` counter
(not `CURRENT_PROJECT_VERSION`, the build number) for each new fork release; both `Debug` and
`Release` build configurations carry the same `MARKETING_VERSION` and must be updated together.

Tag each release `vX.Y.Z.ck.N` (matching `MARKETING_VERSION` with a `v` prefix) and push the tag
alongside the commit.

## Build

Submodules are not checked out by a plain clone:

```
git submodule update --init --recursive
```

This pulls in `MP42Foundation` (and its own `contrib` submodule). Sparkle is resolved separately via
Swift Package Manager on first build/archive.

Local iteration: open `Subler.xcodeproj` in Xcode and run the `Subler` scheme, or from the CLI:

```
xcodebuild -project Subler.xcodeproj -scheme Subler -configuration Debug \
  CODE_SIGN_IDENTITY="-" CODE_SIGN_STYLE=Manual DEVELOPMENT_TEAM="" \
  build
```

## Release build (signed + notarized)

**Do not use a plain `xcodebuild ... build` for a release artifact.** A plain `build` action (as
opposed to `archive`) injects the `com.apple.security.get-task-allow` debug entitlement and signs with
`--timestamp=none` even in the Release configuration - both are automatic notarization rejections
("signature does not include a secure timestamp" / "requests the get-task-allow entitlement"). It
also does not correctly re-sign Sparkle's nested helper bundles (`Updater.app`, `Autoupdate`, its XPC
services) - notarization will reject those as "not signed with a valid Developer ID certificate."

The correct path is `archive` followed by `-exportArchive` with `method: developer-id`, which does a
full re-sign pass across every nested binary:

```bash
rm -rf build
xcodebuild \
  -project Subler.xcodeproj -scheme Subler \
  -destination "generic/platform=macOS" -configuration Release \
  -archivePath build/Subler.xcarchive \
  CODE_SIGN_STYLE=Automatic DEVELOPMENT_TEAM=N2T53236WN \
  -allowProvisioningUpdates \
  archive

cat > build/ExportOptions.plist <<'PLIST'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key><string>developer-id</string>
  <key>teamID</key><string>N2T53236WN</string>
  <key>signingStyle</key><string>automatic</string>
  <key>allowProvisioningUpdates</key><true/>
</dict>
</plist>
PLIST

xcodebuild -exportArchive \
  -archivePath build/Subler.xcarchive -exportPath build/Export \
  -exportOptionsPlist build/ExportOptions.plist -allowProvisioningUpdates
```

`N2T53236WN` is the Klass Concepts LLC Developer ID team - the identity `Developer ID Application:
Klass Concepts LLC (N2T53236WN)` must already be in this Mac's keychain (`security find-identity -v
-p codesigning`). This is a deliberate choice, not the only option: this Mac also has a personal
Developer ID (`Corey Klass`, team `62CR6FVQ77`) available, use that instead by swapping the team ID
above if a given release should be signed personally rather than under the business entity.

Sanity-check the export before burning a notarization round trip on a build that would just get
rejected:

```bash
codesign --verify --deep --strict --verbose=2 build/Export/Subler.app
codesign -dvv build/Export/Subler.app          # expect a real Timestamp=, not none
codesign -d --entitlements - build/Export/Subler.app   # expect no get-task-allow
```

Notarize and staple:

```bash
cd build/Export
ditto -c -k --keepParent Subler.app Subler-notarize.zip
xcrun notarytool submit Subler-notarize.zip --keychain-profile "AC_NOTARY" --wait
xcrun stapler staple Subler.app
spctl -a -vv Subler.app   # expect: accepted, source=Notarized Developer ID
rm -f Subler-notarize.zip
```

The `AC_NOTARY` keychain profile (team `N2T53236WN`) is not specific to this repo - it's a
notarytool credential profile shared across Klass Concepts projects, set up via `xcrun notarytool
store-credentials AC_NOTARY`. It already exists in this Mac's keychain; don't try to recreate it, and
don't `security dump-keychain` to go hunting for it - if it's ever missing, check another project's
release script (e.g. `swift-klass-personal-os/scripts/release.sh`) for the exact `store-credentials`
invocation instead.

## Deploying a build to another Mac

To get a signed, notarized `Subler.app` onto a different Mac (not this one), use the
`klass-personal-os` MCP tools against that Mac's configured SFTP storage account (list them with
`Config_listStorageAccounts` - e.g. `sftp-ck-macmini05`).

**Zip with `ditto`, not the `Archive_create` MCP tool.** `Archive_create` skips symlinks entirely,
and `Subler.app`'s frameworks depend on internal symlinks (`Versions/Current`, top-level aliases into
`Sparkle.framework`, etc.) to be valid bundles - zipping through it silently produces a broken app.
`ditto -c -k --keepParent` preserves them correctly (it's the same mechanism used for the
notarization submission above, which only succeeds if the bundle round-trips intact):

```bash
cd build/Export
ditto -c -k --keepParent Subler.app /path/to/scratch/Subler-<version>.zip
```

Upload with `Storage_upload` (`accountName` = the target's SFTP account, `bucket` = its container,
`key` = a path relative to that container's root - never an absolute path or one starting with `~`).
Then unzip it remotely - this needs `Remote_exec`, which requires the account's `remoteExecPolicy` to
be `full` (check via `Config_listStorageAccounts`; it's `disabled` on some accounts and the call will
be rejected until that's changed):

```
Remote_exec: /usr/bin/ditto -x -k <uploaded zip path> <destination directory>
```

Verify the landed copy is intact and still carries its staple before considering the deploy done:

```
Remote_exec: /usr/bin/codesign -dvv <path>/Subler.app
```

Expect `Notarization Ticket=stapled` in the output.

## Fork changes

See README.md's "Fork changes (KlassyCoder)" section for the changelog of behavioral fixes made in
this fork versus upstream. Keep both in sync: this file covers *how to build and ship*, the README
covers *what changed and why*.
