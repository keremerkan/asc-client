# asc-client

A command-line tool for the App Store Connect API, built with Swift.

## Build & Run

```bash
swift build                           # Debug build
swift build -c release                # Release build (slow — AppStoreAPI has ~2500 generated files)
swift run asc-client <command>        # Run directly
swift run asc-client --help           # Show all commands
```

Install globally:
```bash
strip .build/release/asc-client              # Strip debug symbols (~175 MB → ~59 MB)
cp .build/release/asc-client /usr/local/bin/
```

## Project Structure

```
Package.swift                         # SPM manifest (Swift 6.0, macOS 13+)
Sources/asc-client/
  ASCClient.swift                     # @main entry, root AsyncParsableCommand
  Config.swift                        # ~/.asc-client/config.json loader, ConfigError
  ClientFactory.swift                 # Creates authenticated AppStoreConnectClient
  Formatting.swift                    # Shared helpers: Table.print, formatDate, expandPath
  MediaUpload.swift                   # Media management: upload, download, retry screenshots/previews
  Commands/
    ConfigureCommand.swift            # Interactive credential setup, file permissions
    AppsCommand.swift                 # All app subcommands + findApp/findVersion helpers
    BuildsCommand.swift               # Build subcommands
```

## Dependencies

- **[asc-swift](https://github.com/aaronsky/asc-swift)** (1.0.0+) — App Store Connect API client
  - Product used: `AppStoreConnect` (bundles both `AppStoreConnect` core and `AppStoreAPI` endpoints)
  - `AppStoreAPI` is a target, NOT a separate product — do not add it to Package.swift dependencies
  - API path pattern: `Resources.v1.apps.get()`, `Resources.v1.apps.id("ID").appStoreVersions.get()`
  - Sub-resource access: `Resources.v1.appStoreVersions.id("ID").appStoreVersionLocalizations.get()`
  - Client is a Swift actor: `AppStoreConnectClient`
  - Pagination: `for try await page in client.pages(request)`
  - Resolved version: 1.5.0 (with swift-crypto, URLQueryEncoder, swift-asn1 as transitive deps)
- **[swift-argument-parser](https://github.com/apple/swift-argument-parser)** (1.3.0+) — CLI framework

## Authentication

Config file at `~/.asc-client/config.json`:
```json
{
    "keyId": "KEY_ID",
    "issuerId": "ISSUER_ID",
    "privateKeyPath": "/Users/.../.asc-client/AuthKey_XXXXXXXXXX.p8"
}
```

- `configure` command copies the .p8 file into `~/.asc-client/` and writes the config
- File permissions set to 700 (dir) and 600 (files) — owner-only access
- JWT tokens use ES256 (P256) signing, 20-minute expiry, auto-renewed by asc-swift
- Private key loaded via `JWT.PrivateKey(contentsOf: URL(fileURLWithPath: path))`

## Commands

```
asc-client configure                                              # Interactive setup
asc-client apps list                                              # List all apps
asc-client apps info <bundle-id>                                  # App details
asc-client apps versions <bundle-id>                              # List App Store versions
asc-client apps localizations <bundle-id> [--version X]           # View localizations
asc-client apps review-status <bundle-id>                         # Review submission status
asc-client apps create-version <bundle-id> <ver> [--platform X]   # Create new version
asc-client apps select-build <bundle-id> [--version X]            # Attach a build to a version
asc-client apps submit-for-review <bundle-id> [--version X]       # Submit version for App Review
asc-client apps update-localization <bundle-id> [--locale X]      # Update single locale via flags
asc-client apps update-localizations <bundle-id> [--file X]       # Bulk update from JSON file
asc-client apps export-localizations <bundle-id> [--version X]    # Export to JSON file
asc-client apps upload-media <bundle-id> [--folder X] [--version X] [--replace]  # Upload screenshots/previews
asc-client apps download-media <bundle-id> [--folder X] [--version X]            # Download screenshots/previews
asc-client apps verify-media <bundle-id> [--version X] [--folder X]                               # Check media status, retry stuck
asc-client builds list [--bundle-id <id>]                         # List builds
```

## Key Patterns

### Adding a new subcommand
1. Add the command struct inside `AppsCommand` (or create a new command group)
2. Use `AsyncParsableCommand` for commands that call the API
3. Register in parent's `subcommands` array
4. Use `findApp(bundleID:client:)` to resolve bundle ID to app ID
5. Use `findVersion(appID:versionString:client:)` to resolve version (nil = latest)
6. Use shared `formatDate()` and `expandPath()` from Formatting.swift

### API calls
- **`filterBundleID` does prefix matching** — `com.foo.Bar` also matches `com.foo.BarPro`. Always use `findApp()` which filters for exact `bundleID` match from results.
- **Build relationship returns null when unattached** — `GET /v1/appStoreVersions/{id}/build` returns `{"data": null}` when no build is attached, but `BuildWithoutIncludesResponse.data` is non-optional. Use `try?` to handle the decoding failure gracefully.
- Builds don't have `filterBundleID` — look up app first, then use `filterApp: [appID]`
- Localizations are per-version: get version ID first, then fetch/update localizations
- Updates are one API call per locale — no bulk endpoint in the API
- Only versions in editable states (e.g. `PREPARE_FOR_SUBMISSION`) accept localization updates
- `create-version` `--release-type` is optional; omitting it uses the previous version's setting
- Filter parameters vary per endpoint — check the generated PathsV1*.swift files for exact signatures

### Localization JSON format (used by export/update-localizations)
```json
{
  "en-US": {
    "description": "App description",
    "whatsNew": "- Bug fixes\n- New feature",
    "keywords": "keyword1,keyword2",
    "promotionalText": "Promo text",
    "marketingURL": "https://example.com",
    "supportURL": "https://example.com/support"
  }
}
```

Only fields present in the JSON get updated — omitted fields are left unchanged. The `LocaleFields` struct in AppsCommand.swift defines the schema.

### Media upload folder structure (used by upload-media)
```
media/
├── en-US/
│   ├── APP_IPHONE_67/
│   │   ├── 01_home.png
│   │   ├── 02_settings.png
│   │   └── preview.mp4
│   └── APP_IPAD_PRO_3GEN_129/
│       └── 01_home.png
└── de-DE/
    └── APP_IPHONE_67/
        └── 01_home.png
```

- Level 1: locale, Level 2: display type (ScreenshotDisplayType raw values), Level 3: files
- Images (`.png`, `.jpg`, `.jpeg`) → screenshot sets; Videos (`.mp4`, `.mov`) → preview sets
- Files sorted alphabetically = upload order
- Preview types derived by stripping `APP_` prefix; Watch/iMessage types are screenshots-only
- Upload flow: POST reserve → PUT chunks to presigned URLs → PATCH commit with MD5 checksum
- `--replace` deletes existing assets in matching sets before uploading
- Download filenames are prefixed with `01_`, `02_` etc. to avoid collisions (same name can appear multiple times in a set)
- `ImageAsset.templateURL` uses `{w}x{h}bb.{f}` placeholders — resolve with actual width/height/format for download
- `AppPreview.videoURL` provides direct download URL for preview videos
- Reorder screenshots via `PATCH /v1/appScreenshotSets/{id}/relationships/appScreenshots` with `AppScreenshotSetAppScreenshotsLinkagesRequest`
- `AppMediaAssetState.State` values: `.awaitingUpload`, `.uploadComplete`, `.complete`, `.failed` — stuck items show `uploadComplete`
- `verify-media` checks all media status; with `--folder` retries stuck items: delete → upload → reorder
- File matching: server position N = Nth file alphabetically in local `locale/displayType/` folder

## Not Yet Implemented

API endpoints available but not yet added (43 app sub-resources + 9 top-level resources):
- **TestFlight**: beta groups, beta testers, pre-release versions, beta app localizations
- **Provisioning**: devices, bundle IDs, certificates, profiles
- **Monetization**: in-app purchases, subscriptions, price points, promoted purchases
- **Feedback**: customer reviews, review summarizations
- **Analytics**: analytics reports, performance power metrics
- **Configuration**: app info/categories, availability/territories, encryption declarations, EULA, app events, app clips, custom product pages, A/B experiments

## Release build note

`swift build -c release` is very slow due to whole-module optimization of AppStoreAPI's ~2500 generated files. Debug builds are fast for development.


<claude-mem-context>
# Recent Activity

<!-- This section is auto-generated by claude-mem. Edit content outside the tags. -->

*No recent activity*
</claude-mem-context>