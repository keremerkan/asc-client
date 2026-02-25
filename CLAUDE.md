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
  ASCClient.swift                     # @main entry, root AsyncParsableCommand, central error handling
  Config.swift                        # ~/.asc-client/config.json loader, ConfigError
  ClientFactory.swift                 # Creates authenticated AppStoreConnectClient
  Formatting.swift                    # Shared helpers: Table.print, formatDate, expandPath
  MediaUpload.swift                   # Media management: upload, download, retry screenshots/previews
  Commands/
    ConfigureCommand.swift            # Interactive credential setup, file permissions
    AppsCommand.swift                 # All app subcommands + findApp/findVersion helpers
    BuildsCommand.swift               # Build subcommands
    IAPCommand.swift                  # In-app purchase subcommands (read-only)
    SubCommand.swift                 # Subscription subcommands (read-only)
    RunWorkflowCommand.swift          # Sequential command runner from workflow files
    InstallCompletionsCommand.swift   # Shell completion installer with post-processing patches
    RateLimitCommand.swift            # API rate limit status check
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
asc-client apps review-status <bundle-id> [--version X]            # Review submission status
asc-client apps create-version <bundle-id> <ver> [--platform X]   # Create new version
asc-client apps select-build <bundle-id> [--version X]            # Attach a build to a version
asc-client apps phased-release <bundle-id> [--version X]          # View/manage phased release
asc-client apps age-rating <bundle-id> [--version X] [--file X]   # View/update age rating
asc-client apps routing-coverage <bundle-id> [--file X]           # View/upload routing coverage
asc-client apps submit-for-review <bundle-id> [--version X]       # Submit version for App Review
asc-client apps resolve-issues <bundle-id>                        # Mark rejected items as resolved
asc-client apps cancel-submission <bundle-id>                     # Cancel an active review submission
asc-client apps update-localization <bundle-id> [--locale X]      # Update single locale via flags
asc-client apps update-localizations <bundle-id> [--file X]       # Bulk update from JSON file
asc-client apps export-localizations <bundle-id> [--version X]    # Export to JSON file
asc-client apps upload-media <bundle-id> [--folder X] [--version X] [--replace]  # Upload screenshots/previews
asc-client apps download-media <bundle-id> [--folder X] [--version X]            # Download screenshots/previews
asc-client apps verify-media <bundle-id> [--version X] [--folder X]                               # Check media status, retry stuck
asc-client apps app-info <bundle-id> [--primary-category X]       # View/update app info and categories
asc-client apps app-info --list-categories                        # List available category IDs
asc-client apps availability <bundle-id> [--add X] [--remove X]  # View/update territory availability
asc-client apps encryption <bundle-id> [--create]                 # View/create encryption declarations
asc-client apps eula <bundle-id> [--file X] [--delete]            # View/manage custom EULA
asc-client builds list [--bundle-id <id>] [--version X]           # List builds
asc-client builds archive [--workspace X] [--scheme X] [--output X]  # Archive Xcode project
asc-client builds upload [file]                                   # Upload build via altool
asc-client builds validate [file]                                 # Validate build via altool
asc-client builds await-processing <bundle-id> [--build-version X]  # Wait for build to finish processing
asc-client iap list <bundle-id> [--type X] [--state X]            # List in-app purchases
asc-client iap info <bundle-id> <product-id>                       # IAP details with localizations
asc-client iap promoted <bundle-id>                                # List promoted purchases
asc-client sub groups <bundle-id>                                 # List subscription groups with subscriptions
asc-client sub list <bundle-id>                                   # Flat list of all subscriptions
asc-client sub info <bundle-id> <product-id>                      # Subscription details with localizations
asc-client run-workflow [file] [--yes]                            # Run commands from a workflow file
asc-client rate-limit                                             # Show API rate limit status
asc-client version                                                # Print version number (also: --version, -v)
```

## Key Patterns

### Adding a new subcommand
1. Add the command struct inside `AppsCommand` (or create a new command group)
2. Use `AsyncParsableCommand` for commands that call the API
3. Register in the appropriate `CommandGroup` in the parent's configuration (see below)
4. Use `findApp(bundleID:client:)` to resolve bundle ID to app ID
5. Use `findVersion(appID:versionString:client:)` to resolve version (nil = latest)
6. Use shared `formatDate()` and `expandPath()` from Formatting.swift
7. Run `asc-client install-completions` to regenerate completions after adding commands

### Subcommand grouping
`AppsCommand` uses `CommandGroup` (swift-argument-parser 1.7+) to organize subcommands into sections in `--help` output:
- **ungrouped** (`subcommands:`): list, info, versions — general browse commands
- **Version**: create-version, attach-build, attach-latest-build, detach-build, phased-release, age-rating, routing-coverage
- **Localization**: localizations, export-localizations, update-localization, update-localizations
- **Media**: download-media, upload-media, verify-media
- **Review**: review-status, submit-for-review, resolve-issues, cancel-submission
- **Configuration**: app-info, availability, encryption, eula

When adding a new subcommand, place it in the appropriate `CommandGroup` or create a new one. Shell completions are alphabetically sorted by zsh — don't try to force custom ordering there.

### Version management
- **No `version:` on `CommandConfiguration`** — intentionally omitted. ArgumentParser leaks a root `--version` flag into every subcommand's completion function, which conflicts with subcommands that define their own `--version` option (e.g. `builds list --version`, `apps review-status --version`).
- Version is stored as `static let appVersion` in `ASCClient.swift`.
- `asc-client version` subcommand prints just the version number. `--version` and `-v` are intercepted in `main()` before ArgumentParser and produce the same output.
- `install-completions` stamps `# asc-client vX.Y.Z` into completion scripts (after `#compdef` line for zsh) so `checkCompletionsVersion()` can detect outdated completions.

### Shell completions (`install-completions`)
- ArgumentParser's generated completion scripts need post-processing:
  - **`#compdef` must be line 1** in zsh completion files — never prepend content before it or compinit won't recognize the file.
  - `patchZshHelpCompletions` / `patchBashHelpCompletions` — fix `asc-client help <tab>` to list subcommands (ArgumentParser generates a broken/empty help function).
  - `-V` flag removed from `_describe` so zsh sorts subcommands alphabetically.

### Error handling
- `ASCClient.main()` overrides the default entry point to catch and format errors centrally.
- `ResponseError` (from asc-swift): handles rate limit (429), HTTP status codes (401/403/5xx), and empty responses.
- `URLError`: handles connectivity issues (no internet, DNS, timeout, connection lost, TLS).

### Workflow files (used by run-workflow)
- One command per line, without the `asc-client` prefix
- Lines starting with `#` are comments, blank lines are ignored
- Quoted strings are respected for arguments with spaces (e.g. `--file "path with spaces.json"`)
- Without `--yes`: prompts once to confirm the workflow, then individual commands still prompt normally
- With `--yes`: sets `autoConfirm = true` globally, all prompts are skipped
- Commands are dispatched via `ASCClient.parseAsRoot(args)` — any registered subcommand works
- Nested workflows supported (`run-workflow` can call another workflow file) with circular reference detection via `activeWorkflows` path stack
- `builds upload` sets `lastUploadedBuildVersion` global — subsequent `await-processing` and `attach-latest-build` automatically target the just-uploaded build, avoiding race conditions with API propagation delay

### Build processing
- `awaitBuildProcessing()` is a shared helper in `AppsCommand.swift` (alongside `findApp`/`findVersion`) — used by both `builds await-processing` and `attach-latest-build`
- Recently uploaded builds may take a few minutes to appear in the API — the helper polls with a dot-based progress indicator until the build is found
- `attach-latest-build` prompts to wait if the latest build is still `PROCESSING`; with `--yes` it waits automatically

### API calls
- **`filterBundleID` does prefix matching** — `com.foo.Bar` also matches `com.foo.BarPro`. Always use `findApp()` which filters for exact `bundleID` match from results.
- **Null data in non-optional response fields** — Several GET sub-resource endpoints return `{"data": null}` when no related object exists (e.g. build on version, EULA on app), but generated response types have non-optional `data`. Catch `DecodingError` for these. For EULA, also catch `ResponseError` with 404 status. Never use bare `try?` — it swallows network/auth errors too.
- Builds don't have `filterBundleID` — look up app first, then use `filterApp: [appID]`
- **Encryption declarations use top-level endpoint** — `Resources.v1.apps.id(appID).appEncryptionDeclarations` returns 404 for some apps. Use `Resources.v1.appEncryptionDeclarations.get(filterApp: [appID])` instead.
- **Territory availability limit is 50** — The v1 `include: [.territoryAvailabilities]` has a max limit of 50. Use the v2 sub-resource endpoint `Resources.v2.appAvailabilities.id(availabilityID).territoryAvailabilities.get(limit: 50, include: [.territory])` with `client.pages()` pagination.
- **Multiple AppInfo objects per app** — `appInfos.get()` can return multiple objects (current + replaced). Filter by `state != .replacedWithNewInfo`. Included localizations must be filtered by the selected AppInfo's `relationships.appInfoLocalizations.data` IDs — back-references on included items aren't populated.
- **AppCategory has no name attribute** — The category `id` IS the human-readable name (e.g. `UTILITIES`, `GAMES_ACTION`). No separate name field exists.
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
- **Monetization**: price points, in-app purchase management (create/update/delete), subscription management (create/update/delete)
- **Feedback**: customer reviews, review summarizations
- **Analytics**: analytics reports, performance power metrics
- **Configuration**: app events, app clips, custom product pages, A/B experiments

## Release build note

`swift build -c release` is very slow due to whole-module optimization of AppStoreAPI's ~2500 generated files. Debug builds are fast for development.


<claude-mem-context>
# Recent Activity

<!-- This section is auto-generated by claude-mem. Edit content outside the tags. -->

*No recent activity*
</claude-mem-context>