# ascelerate

A command-line tool for the App Store Connect API, built with Swift.

## Build & Run

```bash
swift build                           # Debug build
swift build -c release                # Release build (slow — AppStoreAPI has ~2500 generated files)
swift run ascelerate <command>        # Run directly
swift run ascelerate --help           # Show all commands
```

Install globally:
```bash
strip .build/release/ascelerate              # Strip debug symbols (~175 MB → ~59 MB)
cp .build/release/ascelerate /usr/local/bin/
```

## Project Structure

```
Package.swift                         # SPM manifest (Swift 6.0, macOS 13+)
Sources/ascelerate/
  Ascelerate.swift                    # @main entry, root AsyncParsableCommand, central error handling
  Config.swift                        # ~/.ascelerate/config.json loader, ConfigError
  ClientFactory.swift                 # Creates authenticated AppStoreConnectClient
  Formatting.swift                    # Shared helpers: Table.print, ANSI colors, formatFieldName/formatState, formatDate, expandPath
  Aliases.swift                        # Alias storage (~/.ascelerate/aliases.json), resolveAlias()
  MediaUpload.swift                   # Media management: upload, download, retry screenshots/previews
  Commands/
    ConfigureCommand.swift            # Interactive credential setup, file permissions
    AppsCommand.swift                 # All app subcommands + findApp/findVersion helpers
    BuildsCommand.swift               # Build subcommands
    IAPCommand.swift                  # In-app purchase subcommands
    SubCommand.swift                 # Subscription subcommands
    CustomerReviewsCommand.swift      # Customer reviews + developer responses (list, info, respond, delete-response)
    AppEventsCommand.swift            # In-app events: CRUD + scheduling + localizations + media (screenshots/video clips); findAppEvent helper
    ProductPagesCommand.swift         # Custom product pages: CRUD (compound create) + localizations + media (screenshot/preview sets); findProductPage helper
    DevicesCommand.swift              # Device management subcommands + findDevice helper
    CertsCommand.swift                # Signing certificate subcommands + findCertificate helper
    BundleIDsCommand.swift            # Bundle identifier subcommands + findBundleID helper
    ProfilesCommand.swift             # Provisioning profile subcommands + findProfile helper
    AliasCommand.swift                # Alias management (add, remove, list) for bundle ID shortcuts
    RunWorkflowCommand.swift          # Sequential command runner from workflow files
    InstallCompletionsCommand.swift   # Shell completion installer with post-processing patches
    InstallSkillCommand.swift         # Claude Code skill installer (fetches from GitHub)
    RateLimitCommand.swift            # API rate limit status check
skills/
  ascelerate/SKILL.md                # AI coding skill (single source of truth)
  package.json                        # npm package for npx installer
  bin/install.js                      # npx installer (fetches SKILL.md from GitHub)
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
- **[swift-certificates](https://github.com/apple/swift-certificates)** (1.0.0+) — X.509 certificate and CSR generation (used by `certs create` auto-CSR flow)
- **[Yams](https://github.com/jpsim/Yams)** (5.0.0+) — YAML parsing (used by `screenshot` command for `ascelerate.yml`)

## Authentication

Config file at `~/.ascelerate/config.json`:
```json
{
    "keyId": "KEY_ID",
    "issuerId": "ISSUER_ID",
    "privateKeyPath": "/Users/.../.ascelerate/AuthKey_XXXXXXXXXX.p8"
}
```

- `configure` command copies the .p8 file into `~/.ascelerate/` and writes the config
- File permissions set to 700 (dir) and 600 (files) — owner-only access
- JWT tokens use ES256 (P256) signing, 20-minute expiry, auto-renewed by asc-swift
- Private key loaded via `JWT.PrivateKey(contentsOf: URL(fileURLWithPath: path))`

## Commands

```
ascelerate configure                                              # Interactive setup
ascelerate apps list                                              # List all apps
ascelerate apps info <bundle-id>                                  # App details
ascelerate apps versions <bundle-id>                              # List App Store versions
ascelerate apps localizations view <bundle-id> [--version X]      # View localizations
ascelerate apps localizations update <bundle-id> [--locale X]     # Update single locale via flags
ascelerate apps localizations import <bundle-id> [--file X]       # Bulk update from JSON file
ascelerate apps localizations export <bundle-id> [--version X]    # Export to JSON file
ascelerate apps review preflight <bundle-id> [--version X]           # Pre-submission checks (includes IAP/sub state and pricing)
ascelerate apps review status <bundle-id> [--version X]             # Review submission status
ascelerate apps create-version <bundle-id> <ver> [--platform X]   # Create new version
ascelerate apps build attach <bundle-id> [--version X]             # Interactively select and attach a build
ascelerate apps build attach-latest <bundle-id> [--version X]     # Attach the most recent build
ascelerate apps build detach <bundle-id> [--version X]            # Remove the attached build
ascelerate apps phased-release <bundle-id> [--version X]          # View/manage phased release
ascelerate apps app-info age-rating <bundle-id>                            # View age rating
ascelerate apps app-info age-rating export <bundle-id>                     # Export age rating to JSON
ascelerate apps app-info age-rating import <bundle-id> [--file X]          # Update age rating from JSON
ascelerate apps routing-coverage <bundle-id> [--file X]           # View/upload routing coverage
ascelerate apps review submit <bundle-id> [--version X]            # Submit version for App Review
ascelerate apps review resolve-issues <bundle-id>                 # Mark rejected items as resolved
ascelerate apps review cancel-submission <bundle-id>              # Cancel an active review submission
ascelerate apps review info <bundle-id> [--version X] [--contact-first-name X] [--contact-last-name X] [--contact-phone X] [--contact-email X] [--demo-account-name X] [--demo-account-password X] [--demo-account-required true|false] [--notes X] [-y]  # View/update App Review Information
ascelerate apps review attachment list <bundle-id> [--version X]              # List App Review attachment files
ascelerate apps review attachment upload <bundle-id> [--version X] <file> [-y]  # Upload an App Review attachment
ascelerate apps review attachment delete <attachment-id> [-y]               # Delete an App Review attachment
ascelerate apps media upload <bundle-id> [folder] [--version X] [--replace]  # Upload screenshots/previews
ascelerate apps media download <bundle-id> [--folder X] [--version X]        # Download screenshots/previews
ascelerate apps media verify <bundle-id> [--version X] [folder]              # Check media status, retry stuck
ascelerate apps app-info view <bundle-id>                         # View app info, categories, and localizations
ascelerate apps app-info view --list-categories                   # List available category IDs
ascelerate apps app-info update <bundle-id> [--name X] [--subtitle X] [--primary-category X] [-y]  # Update localization fields and/or categories
ascelerate apps app-info import <bundle-id> [--file X] [--verbose] [-y]  # Bulk update localizations from JSON
ascelerate apps app-info export <bundle-id> [--output X]          # Export localizations to JSON
ascelerate apps availability <bundle-id> [--add X] [--remove X]  # View/update territory availability
ascelerate apps encryption <bundle-id> [--create]                 # View/create encryption declarations
ascelerate apps eula <bundle-id> [--file X] [--delete]            # View/manage custom EULA
ascelerate apps subscription-grace-period <bundle-id> [--opt-in true|false] [--sandbox-opt-in true|false] [--duration X] [--renewal-type X] [-y]  # View/update app-level subscription grace period
ascelerate builds list [--bundle-id <id>] [--version X]           # List builds
ascelerate builds archive [--workspace X] [--scheme X] [--output X]  # Archive Xcode project
ascelerate builds upload [file]                                   # Upload build via altool
ascelerate builds validate [file]                                 # Validate build via altool
ascelerate builds await-processing <bundle-id> [--build-version X]  # Wait for build to finish processing
ascelerate iap list <bundle-id> [--type X] [--state X]            # List in-app purchases
ascelerate iap info <bundle-id> <product-id>                       # IAP details, localizations, missing-pricing warning
ascelerate iap promoted <bundle-id>                                # List promoted purchases
ascelerate iap create <bundle-id> [--type X] [--product-id X] [--name X] [--review-note X] [--family-sharable] [-y]  # Create IAP
ascelerate iap update <bundle-id> <product-id> [--name X] [--review-note X] [--family-sharable true|false] [-y]  # Update IAP
ascelerate iap delete <bundle-id> <product-id> [-y]               # Delete IAP
ascelerate iap submit <bundle-id> <product-id> [-y]               # Submit IAP for review
ascelerate iap localizations view <bundle-id> <product-id>        # View IAP localizations
ascelerate iap localizations export <bundle-id> <product-id> [--output X]  # Export IAP localizations to JSON
ascelerate iap localizations import <bundle-id> <product-id> [--file X] [--verbose] [-y]  # Import IAP localizations from JSON
ascelerate iap pricing show <bundle-id> <product-id>              # Show current price schedule (warns if missing)
ascelerate iap pricing tiers <bundle-id> <product-id> [--territory USA]  # List available price tiers for a territory
ascelerate iap pricing set <bundle-id> <product-id> --price 4.99 [--base-territory USA] [--start-date YYYY-MM-DD] [--remove-all-overrides] [-y]  # Set base price; preserves overrides (interactive menu to drop)
ascelerate iap pricing override <bundle-id> <product-id> --price 5.99 --territory FRA [--start-date YYYY-MM-DD] [-y]  # Add/update per-territory manual price override
ascelerate iap pricing remove <bundle-id> <product-id> --territory FRA [-y]  # Drop a per-territory override (revert to auto-equalize)
ascelerate iap availability <bundle-id> <product-id> [--add X] [--remove X] [--available-in-new-territories true|false] [-y]  # View/update per-IAP territory availability
ascelerate iap offer-code list <bundle-id> <product-id>           # List offer codes for an IAP
ascelerate iap offer-code info <bundle-id> <product-id> <offer-code-id>  # Show details + price/code counts
ascelerate iap offer-code create <bundle-id> <product-id> --name X --eligibility N,A,C --price 4.99 [--territory USA] [--equalize-all-territories] [-y]  # Create offer code with single-territory or fan-out prices
ascelerate iap offer-code toggle <bundle-id> <product-id> <offer-code-id> --active true|false [-y]  # Activate / deactivate
ascelerate iap offer-code gen-codes <bundle-id> <product-id> <offer-code-id> --count N --expires YYYY-MM-DD [--environment PRODUCTION|SANDBOX] [-y]  # Generate one-time-use code batch
ascelerate iap offer-code add-custom-codes <bundle-id> <product-id> <offer-code-id> --code X --count N [--expires YYYY-MM-DD] [-y]  # Add a custom redeem code
ascelerate iap offer-code view-codes <one-time-use-batch-id> [--output X]  # Fetch actual code values (async; may need retry)
ascelerate iap images list <bundle-id> <product-id>               # List uploaded promotional images
ascelerate iap images upload <bundle-id> <product-id> <file> [-y] # Upload image (3-step reserve/PUT/commit)
ascelerate iap images delete <bundle-id> <product-id> <image-id> [-y]  # Delete an image
ascelerate iap review-screenshot view <bundle-id> <product-id>    # Show current App Review screenshot
ascelerate iap review-screenshot upload <bundle-id> <product-id> <file> [-y]  # Upload (replaces existing)
ascelerate iap review-screenshot delete <bundle-id> <product-id> [-y]  # Delete
ascelerate sub groups <bundle-id>                                 # List subscription groups with subscriptions
ascelerate sub list <bundle-id>                                   # Flat list of all subscriptions
ascelerate sub info <bundle-id> <product-id>                      # Subscription details, localizations, missing-prices warning
ascelerate sub create <bundle-id> [--product-id X] [--name X] [--period X] [--group-level N] [--review-note X] [--family-sharable] [-y]  # Create subscription
ascelerate sub update <bundle-id> <product-id> [--name X] [--review-note X] [--group-level N] [--family-sharable true|false] [-y]  # Update subscription
ascelerate sub delete <bundle-id> <product-id> [-y]               # Delete subscription
ascelerate sub submit <bundle-id> <product-id> [-y]               # Submit subscription for review
ascelerate sub create-group <bundle-id> [--name X] [-y]           # Create subscription group
ascelerate sub update-group <bundle-id> [--name X] [-y]           # Update subscription group
ascelerate sub delete-group <bundle-id> [-y]                      # Delete subscription group
ascelerate sub localizations view <bundle-id> <product-id>        # View subscription localizations
ascelerate sub localizations export <bundle-id> <product-id> [--output X]  # Export subscription localizations to JSON
ascelerate sub localizations import <bundle-id> <product-id> [--file X] [--verbose] [-y]  # Import subscription localizations from JSON
ascelerate sub group-localizations view <bundle-id>               # View group localizations
ascelerate sub group-localizations export <bundle-id> [--output X]  # Export group localizations to JSON
ascelerate sub group-localizations import <bundle-id> [--file X] [--verbose] [-y]  # Import group localizations from JSON
ascelerate sub pricing show <bundle-id> <product-id>              # Show current prices per territory (warns if none)
ascelerate sub pricing tiers <bundle-id> <product-id> [--territory USA]  # List available price tiers for a territory
ascelerate sub pricing set <bundle-id> <product-id> --price 4.99 [--territory USA] [--start-date YYYY-MM-DD] [--preserve-current | --no-preserve-current] [--equalize-all-territories] [--confirm-decrease] [-y]  # Set price for one territory or fan out to all (equalized). Increases require --preserve-current/--no-preserve-current; decreases prompt interactively or require --confirm-decrease in -y mode.
ascelerate sub availability <bundle-id> <product-id> [--add X] [--remove X] [--available-in-new-territories true|false] [-y]  # View/update per-subscription territory availability
ascelerate sub intro-offer list <bundle-id> <product-id>          # List intro offers (free trials, intro discounts)
ascelerate sub intro-offer create <bundle-id> <product-id> --mode FREE_TRIAL|PAY_AS_YOU_GO|PAY_UP_FRONT --duration X --periods N [--territory USA] [--price 4.99] [--start-date Y] [--end-date Y] [-y]  # Create intro offer
ascelerate sub intro-offer update <bundle-id> <product-id> <offer-id> --end-date Y [-y]  # Update intro offer end date
ascelerate sub intro-offer delete <bundle-id> <product-id> <offer-id> [-y]  # Delete intro offer
ascelerate sub offer-code list <bundle-id> <product-id>           # List offer codes for a subscription
ascelerate sub offer-code info <bundle-id> <product-id> <offer-code-id>  # Show details + price/code counts
ascelerate sub offer-code create <bundle-id> <product-id> --name X --eligibility N,E,X --offer-eligibility STACK_WITH_INTRO_OFFERS|REPLACE_INTRO_OFFERS --mode FREE_TRIAL|PAY_AS_YOU_GO|PAY_UP_FRONT --duration X --periods N [--auto-renew | --no-auto-renew] --price 4.99 [--territory USA] [--equalize-all-territories] [-y]  # Create offer code
ascelerate sub offer-code toggle <bundle-id> <product-id> <offer-code-id> --active true|false [-y]  # Activate / deactivate
ascelerate sub offer-code gen-codes <bundle-id> <product-id> <offer-code-id> --count N --expires YYYY-MM-DD [--environment PRODUCTION|SANDBOX] [-y]  # Generate one-time-use code batch
ascelerate sub offer-code add-custom-codes <bundle-id> <product-id> <offer-code-id> --code X --count N [--expires YYYY-MM-DD] [-y]  # Add a custom redeem code
ascelerate sub offer-code view-codes <one-time-use-batch-id> [--output X]  # Fetch actual code values (async; may need retry)
ascelerate sub promo-offer list <bundle-id> <product-id>          # List promotional offers (server-signed)
ascelerate sub promo-offer info <bundle-id> <product-id> <offer-id>  # Show details
ascelerate sub promo-offer create <bundle-id> <product-id> --name X --code X --mode X --duration X --periods N --price 4.99 [--territory USA] [--equalize-all-territories] [-y]  # Create
ascelerate sub promo-offer update <bundle-id> <product-id> <offer-id> --price 4.99 [--territory USA] [--equalize-all-territories] [-y]  # Update prices only
ascelerate sub promo-offer delete <bundle-id> <product-id> <offer-id> [-y]  # Delete
ascelerate sub submit-group <bundle-id> [-y]                      # Submit a subscription group for review (mirror of `sub submit`)
ascelerate sub images list <bundle-id> <product-id>               # List uploaded promotional images
ascelerate sub images upload <bundle-id> <product-id> <file> [-y] # Upload image (3-step reserve/PUT/commit)
ascelerate sub images delete <bundle-id> <product-id> <image-id> [-y]  # Delete an image
ascelerate sub review-screenshot view <bundle-id> <product-id>    # Show current App Review screenshot
ascelerate sub review-screenshot upload <bundle-id> <product-id> <file> [-y]  # Upload (replaces existing)
ascelerate sub review-screenshot delete <bundle-id> <product-id> [-y]  # Delete
ascelerate devices list [--name X] [--platform X] [--status X]   # List registered devices
ascelerate devices info [name-or-udid]                            # Device details (interactive picker if omitted)
ascelerate devices register [--name X] [--udid X] [--platform X] [-y]  # Register a new device (interactive if omitted)
ascelerate devices update [name-or-udid] [--name X] [--status X] [-y]  # Update device (interactive if omitted)
ascelerate certs list [--type X] [--display-name X]               # List signing certificates
ascelerate certs info [serial-or-name]                            # Certificate details (interactive picker if omitted)
ascelerate certs create [--type X] [--csr <file>] [--output X] [-y]  # Create certificate (interactive type picker if omitted)
ascelerate certs revoke [serial-number] [-y]                      # Revoke a certificate (interactive picker if omitted)
ascelerate bundle-ids list [--platform X] [--identifier X]        # List bundle identifiers
ascelerate bundle-ids info [identifier]                           # Bundle ID details with capabilities (interactive picker if omitted)
ascelerate bundle-ids register [--name X] [--identifier X] [--platform X] [-y]  # Register a bundle ID (interactive if omitted)
ascelerate bundle-ids update [identifier] [--name X] [-y]         # Rename a bundle ID (interactive if omitted)
ascelerate bundle-ids delete [identifier] [-y]                    # Delete a bundle ID (interactive picker if omitted)
ascelerate bundle-ids enable-capability [identifier] [--type X] [-y]   # Enable a capability (interactive if omitted)
ascelerate bundle-ids disable-capability [identifier] [-y]        # Disable a capability (interactive picker)
ascelerate profiles list [--name X] [--type X] [--state X]       # List provisioning profiles
ascelerate profiles info [name]                                   # Profile details (interactive picker if omitted)
ascelerate profiles download [name] [--output X]                  # Download profile (interactive picker if omitted)
ascelerate profiles create [--name X] [--type X] [--bundle-id X] [--certificates X] [--devices X] [--output X] [-y]  # Create a profile (interactive if omitted; --certificates all = all of matching family)
ascelerate profiles delete [name] [-y]                            # Delete a profile (interactive picker if omitted)
ascelerate profiles reissue [name] [--all] [--all-invalid] [--to-certs X] [--all-devices] [-y]  # Reissue profiles with latest cert (or specific certs)
ascelerate alias add [name]                                       # Add/update an alias (interactive app picker if name omitted)
ascelerate alias remove [name] [-y]                               # Remove an alias (interactive picker if name omitted)
ascelerate alias list                                             # List all aliases
ascelerate events list <bundle-id> [--state X]                    # List in-app events
ascelerate events info <bundle-id> <ref-or-id>                    # Event details + schedules + localizations
ascelerate events create <bundle-id> --reference-name X [--badge X] [--priority X] [--purpose X] [--deep-link URL] [--primary-locale X] [--territories USA,GBR] [--publish-start X] [--event-start X] [--event-end X] [-y]  # Create event
ascelerate events update <bundle-id> <ref-or-id> [--reference-name X] [--badge X|NONE] [--priority X] [--purpose X] [--territories X] [--publish-start X] [--event-start X] [--event-end X] [-y]  # Update event/schedule
ascelerate events delete <bundle-id> <ref-or-id> [-y]            # Delete event
ascelerate events localizations view <bundle-id> <ref-or-id>     # View event localizations
ascelerate events localizations export <bundle-id> <ref-or-id> [--output X]  # Export event localizations to JSON
ascelerate events localizations import <bundle-id> <ref-or-id> [--file X] [--verbose] [-y]  # Import event localizations from JSON
ascelerate events media list <bundle-id> <ref-or-id>             # List event card screenshots + video clips
ascelerate events media upload <bundle-id> <ref-or-id> --locale X --asset-type EVENT_CARD|EVENT_DETAILS_PAGE [--preview-frame X] <file> [-y]  # Upload screenshot/video clip
ascelerate events media delete <bundle-id> <ref-or-id> <media-id> [-y]  # Delete a screenshot or video clip
ascelerate product-pages list <bundle-id>                         # List custom product pages
ascelerate product-pages info <bundle-id> <name-or-id>            # Page versions + localizations
ascelerate product-pages create <bundle-id> --name X --locale X [--promotional-text X] [-y]  # Create page (compound: version + first locale)
ascelerate product-pages update <bundle-id> <name-or-id> [--name X] [--visible true|false] [-y]  # Rename / toggle visibility
ascelerate product-pages delete <bundle-id> <name-or-id> [-y]    # Delete page
ascelerate product-pages localizations view <bundle-id> <name-or-id>    # View page localizations
ascelerate product-pages localizations export <bundle-id> <name-or-id> [--output X]  # Export to JSON
ascelerate product-pages localizations import <bundle-id> <name-or-id> [--file X] [--verbose] [-y]  # Import promotional text from JSON
ascelerate product-pages media list <bundle-id> <name-or-id>     # List page screenshots + app previews
ascelerate product-pages media upload <bundle-id> <name-or-id> --locale X [--display-type APP_IPHONE_67 | --preview-type APP_IPHONE_67] <file> [--preview-frame X] [-y]  # Upload screenshot/preview (creates set)
ascelerate product-pages media delete <bundle-id> <name-or-id> <media-id> [-y]  # Delete a screenshot or app preview
ascelerate reviews list <bundle-id> [--rating N] [--territory X] [--sort recent|oldest|critical|best] [--unanswered] [--limit N]  # List customer reviews
ascelerate reviews info <review-id>                               # Full review text + developer response
ascelerate reviews respond <review-id> --body "X" [-y]           # Publish/replace developer response
ascelerate reviews delete-response <review-id> [-y]              # Delete developer response
ascelerate run-workflow [file] [--yes]                            # Run commands from a workflow file
ascelerate rate-limit                                             # Show API rate limit status
ascelerate install-skill [--uninstall]                            # Install/remove Claude Code skill
ascelerate screenshot [--languages en-US,tr-TR]                   # Capture screenshots (optional language subset override)
ascelerate screenshot init                                        # Create ascelerate/screenshot.yml + ScreenshotHelper.swift
ascelerate screenshot create-helper [-o file]                     # Generate UITest helper file
ascelerate version                                                # Print version number (also: --version, -v)
```

## Key Patterns

### Adding a new subcommand
1. Add the command struct inside `AppsCommand` (or create a new command group)
2. Use `AsyncParsableCommand` for commands that call the API
3. Register in the appropriate `CommandGroup` in the parent's configuration (see below)
4. Use `findApp(bundleID:client:)` to resolve bundle ID to app ID
5. Use `findVersion(appID:versionString:platform:client:)` to resolve version (nil = prefers editable, prompts if multiple platforms)
6. Use shared helpers from Formatting.swift: `formatDate()`, `expandPath()`, `formatState()` for enum display, color helpers (`green()`, `red()`, `yellow()`, `bold()`)
7. Run `ascelerate install-completions` to regenerate completions after adding commands

### Subcommand grouping
`AppsCommand` uses `CommandGroup` (swift-argument-parser 1.7+) to organize subcommands into sections in `--help` output:
- **ungrouped** (`subcommands:`): list, info, versions — general browse commands
- **Version**: create-version, build (attach, attach-latest, detach), phased-release, routing-coverage
- **Info & Content**: app-info (view, update, import, export, age-rating (view, export, import)), localizations (view, update, import, export), media (upload, download, verify)
- **Configuration**: availability, encryption, eula
- **Review**: review (preflight, status, submit, resolve-issues, cancel-submission, info, attachment)

When adding a new subcommand, place it in the appropriate `CommandGroup` or create a new one. Shell completions are alphabetically sorted by zsh — don't try to force custom ordering there.

### App aliases
- Aliases map short names to bundle IDs, stored in `~/.ascelerate/aliases.json`
- `resolveAlias()` in `Aliases.swift` is the single resolution function — if input contains no dots, look up in aliases
- `findApp()` in `AppsCommand.swift` calls `resolveAlias()` at the top — this covers all app, IAP, subscription, and build commands automatically
- Alias names must match `^[a-zA-Z0-9_-]+$` — no dots (dots distinguish real bundle IDs from aliases)
- `bundle-ids`, `profiles`, `devices`, `certs` commands do NOT resolve aliases (different domain)

### Version management
- **No `version:` on `CommandConfiguration`** — intentionally omitted. ArgumentParser leaks a root `--version` flag into every subcommand's completion function, which conflicts with subcommands that define their own `--version` option (e.g. `builds list --version`, `apps review status --version`).
- Version is stored as `static let appVersion` in `ASC.swift`.
- `ascelerate version` subcommand prints just the version number. `--version` and `-v` are intercepted in `main()` before ArgumentParser and produce the same output.
- `install-completions` stamps `# ascelerate vX.Y.Z` into completion scripts (after `#compdef` line for zsh) and `install-skill` stamps `<!-- ascelerate vX.Y.Z -->` into the installed skill file.
- `checkForUpdates()` (non-interactive, API commands) and `checkForUpdatesInteractively()` (bare invocation) detect outdated completions and/or skill, offering a single Y/n prompt or NOTE line.
- Both `install-skill` and the npx installer (`npx ascelerate-skill`) fetch `SKILL.md` from GitHub — the skill content is NOT embedded in the binary. `skills/ascelerate/SKILL.md` in the repo is the single source of truth.

### Shell completions (`install-completions`)
- ArgumentParser's generated completion scripts need post-processing:
  - **`#compdef` must be line 1** in zsh completion files — never prepend content before it or compinit won't recognize the file.
  - `patchZshHelpCompletions` / `patchBashHelpCompletions` — fix `ascelerate help <tab>` to list subcommands (ArgumentParser generates a broken/empty help function).
  - `-V` flag removed from all `_describe` calls so zsh sorts completions alphabetically.
- **Argument-level completions** via ArgumentParser's `completion:` parameter:
  - `.file(extensions:)` — file path completion filtered by extension (e.g. `.json`, `.workflow`, `.ipa`)
  - `.shellCommand()` — dynamic completions from a shell command (used for alias names from `~/.ascelerate/aliases.json`)
  - Bundle ID arguments use `.shellCommand("grep ...")` to extract alias keys from the aliases JSON file
  - File arguments use `.file(extensions: ["json"])`, `.file(extensions: ["workflow", "txt"])`, etc.

### Interactive mode
- Most provisioning commands (devices, certs, bundle-ids, profiles) support interactive mode — arguments and options are optional.
- When omitted, commands prompt with numbered lists fetched from the API (e.g. bundle ID picker, certificate picker, profile type selection).
- Text inputs use a recursive `promptText()` that retries on empty input (same pattern as ConfigureCommand's `prompt()`).
- Selection lists use `[\(i + 1)]` numbering, `readLine()` input, `Int()` parsing, and range validation.
- `--yes` / `autoConfirm` is incompatible with interactive mode — commands throw `ValidationError` when required options are missing with `--yes`.
- `enable-capability` filters the type picker to exclude already-enabled capabilities; `disable-capability` only shows enabled ones.
- `enable-capability` and `disable-capability` offer to regenerate provisioning profiles after changes (delete + recreate with same settings) via `regenerateProfilesIfNeeded()` helper in BundleIDsCommand.swift.

### Output formatting
- **ANSI colors** — `red()`, `green()`, `yellow()` (orange 208), `bold()` in Formatting.swift. Auto-disabled when stdout is not a terminal (`isatty` check). `stderrRed()` uses a separate `isStderrTerminal` check for error messages.
- **Colored output conventions** — `green()` for success verbs ("Created", "Updated", "Deleted", etc.), `yellow()` for "Cancelled.", `stderrRed("Error:")` in central error handler. `red()` for failure indicators (e.g. preflight ✗).
- **`formatFieldName()`** — converts camelCase (`whatsNew` → "What's New") and SCREAMING_SNAKE_CASE (`PREPARE_FOR_SUBMISSION` → "Prepare for Submission") to human-readable titles. Has override map for special cases (URL suffixes, OS names, `CANCELED` → "Cancelled").
- **`formatState()`** — generic wrapper: `formatFieldName("\(value)")`. Use for any enum/state value displayed to the user (e.g. `.map { formatState($0) }`). Applied globally across all command files for platform, status, type, and state fields.
- **`localeName()`** — resolves locale codes to human-readable names via `Locale.current.localizedString(forIdentifier:)` (e.g. `en-US` → `en-US (English (United States))`). Applied to all locale display across commands.
- **`parseEnum()`** — validates a string against a `RawRepresentable & CaseIterable` enum, returning the matched case or throwing `ValidationError` with valid values list. Use instead of inline `guard let X = T(rawValue: .uppercased())` blocks. `parseFilter()` wraps `parseEnum()` for optional API filter values (returns `[T]?`).
- **`fetchAll()`** — collects all items from paginated API responses (`client.pages()`) into a single array with empty guard and optional sort. Used by `promptDevice()`, `promptCertificate()`, `promptBundleID()`, `promptProfile()`.
- **`resolveFile()`** — resolves a file path from an optional argument; lists matching files by extension in the current directory as a numbered picker, with manual path entry fallback. Used by `localizations import` and `app-info import`.
- **ANSI-aware Table** — `Table.print` uses `visibleLength()` (strips ANSI codes via regex) and `padToVisible()` for correct column alignment when cells contain colored text. All-empty rows (`["", ""]`) render as blank lines for visual grouping.

### Error handling
- `ASCClient.main()` overrides the default entry point to catch and format errors centrally.
- `ResponseError` (from asc-swift): handles rate limit (429), HTTP status codes (401/403/5xx), and empty responses.
- `URLError`: handles connectivity issues (no internet, DNS, timeout, connection lost, TLS).

### Workflow files (used by run-workflow)
- One command per line, without the `ascelerate` prefix
- Lines starting with `#` are comments, blank lines are ignored
- Quoted strings are respected for arguments with spaces (e.g. `--file "path with spaces.json"`)
- Without `--yes`: prompts once to confirm the workflow, then individual commands still prompt normally
- With `--yes`: sets `autoConfirm = true` globally, all prompts are skipped
- Commands are dispatched via `ASCClient.parseAsRoot(args)` — any registered subcommand works
- Nested workflows supported (`run-workflow` can call another workflow file) with circular reference detection via `activeWorkflows` path stack
- `builds upload` sets `lastUploadedBuildVersion` global — subsequent `await-processing` and `build attach-latest` automatically target the just-uploaded build, avoiding race conditions with API propagation delay

### Xcode signing
- Both `builds archive` and the `.xcarchive` → `.ipa` export pass `-allowProvisioningUpdates` to `xcodebuild`. Without this, `xcodebuild` only uses locally cached provisioning profiles and won't fetch updated ones from the Developer Portal (Xcode GUI does this automatically, CLI does not).
- Xcode no longer downloads profiles to `~/Library/MobileDevice/Provisioning Profiles/` — with automatic signing it manages them internally. That folder is legacy.
- `-allowProvisioningUpdates` authenticates via the Apple ID in Xcode > Settings > Accounts. For CI, pass `-authenticationKeyPath`/`-authenticationKeyID`/`-authenticationKeyIssuerID`.

### Build processing
- `awaitBuildProcessing()` is a shared helper in `AppsCommand.swift` (alongside `findApp`/`findVersion`) — used by both `builds await-processing` and `build attach-latest`
- Recently uploaded builds may take a few minutes to appear in the API — the helper polls with a dot-based progress indicator until the build is found
- `build attach-latest` prompts to wait if the latest build is still `PROCESSING`; with `--yes` it waits automatically

### API calls
- **`Certificate` type is ambiguous** — both `AppStoreAPI.Certificate` and `X509.Certificate` exist. In `CertsCommand.swift` (which imports both), use `AppStoreAPI.Certificate` explicitly for API response types.
- **`filterBundleID` does prefix matching** — `com.foo.Bar` also matches `com.foo.BarPro`. Always use `findApp()` which filters for exact `bundleID` match from results.
- **Null data in non-optional response fields** — Several GET sub-resource endpoints return `{"data": null}` when no related object exists (e.g. build on version, EULA on app), but generated response types have non-optional `data`. Catch `DecodingError` for these. For EULA, also catch `ResponseError` with 404 status.
- Builds don't have `filterBundleID` — look up app first, then use `filterApp: [appID]`
- **Encryption declarations use top-level endpoint** — `Resources.v1.apps.id(appID).appEncryptionDeclarations` returns 404 for some apps. Use `Resources.v1.appEncryptionDeclarations.get(filterApp: [appID])` instead.
- **Territory availability limit is 50** — The v1 `include: [.territoryAvailabilities]` has a max limit of 50. Use the v2 sub-resource endpoint `Resources.v2.appAvailabilities.id(availabilityID).territoryAvailabilities.get(limit: 50, include: [.territory])` with `client.pages()` pagination.
- **Multiple AppInfo objects per app** — `appInfos.get()` can return multiple objects (current + replaced). `pickActiveAppInfo()` handles selection: filters out `replacedWithNewInfo`, prefers editable state (prepareForSubmission/waitingForReview) over live. Used by both `findActiveAppInfo()` and `app-info view`. Included localizations must be filtered by the selected AppInfo's `relationships.appInfoLocalizations.data` IDs — back-references on included items aren't populated.
- **`findVersion()` prefers editable versions** — when `versionString` is nil, first queries for prepareForSubmission/waitingForReview versions. If multiple exist (multi-platform apps), prompts user to select by platform. Falls back to latest version if none are editable.
- **AppCategory has no name attribute** — The category `id` IS the human-readable name (e.g. `UTILITIES`, `GAMES_ACTION`). No separate name field exists.
- Localizations are per-version: get version ID first, then fetch/update localizations
- Updates are one API call per locale — no bulk endpoint in the API
- Only versions in editable states (`PREPARE_FOR_SUBMISSION` or `WAITING_FOR_REVIEW`) accept localization updates — except `promotionalText`, which can be updated in any state
- `create-version` `--release-type` is optional; omitting it uses the previous version's setting
- **`bundleIDCapabilities` sub-resource rejects `limit`** — despite the generated code accepting `limit: Int?`, the API returns an error if `limit` is passed. Use `.get()` with no arguments.
- Filter parameters vary per endpoint — check the generated PathsV1*.swift files for exact signatures
- **IAP price schedule fetches need fields[] AND include[] for relationships to populate** — Two related quirks discovered when wiring `iap pricing`:
  - `Resources.v2.inAppPurchases.id(iapID).iapPriceSchedule.get(...)` returns the schedule but `relationships.baseTerritory.data.id` is nil unless you pass `fieldsInAppPurchasePriceSchedules: [.baseTerritory, .manualPrices]` AND `include: [.baseTerritory]`. Both are required.
  - `Resources.v1.inAppPurchasePriceSchedules.id(scheduleID).manualPrices.get(...)` returns the price records but each record's `relationships.territory.data.id` and `relationships.inAppPurchasePricePoint.data.id` are nil unless you pass `include: [.inAppPurchasePricePoint, .territory]`. Without it you get just resource links.
  - Bottom line: for relationship `data` IDs, always pass both `include` and (where required) the matching `fields[]`. The schedule POST replaces the whole schedule wholesale, so you need these IDs to round-trip existing manualPrices through a read-modify-write.
- **IAP schedule POST is wholesale, but additive in practice** — `POST /v1/inAppPurchasePriceSchedules` fully replaces any existing schedule. To preserve manual overrides across a base-territory change or any edit, fetch the existing schedule first and include all entries you want kept in the new payload. Apple's web UI's "all manual prices will be deleted" warning is UI-conservatism — the API itself preserves whatever you send. The `iap pricing set/override/remove` commands all do this read-modify-write.
- **Subscription pricing is per-territory** — there's no `manualPrices`/`automaticPrices` split like IAPs have. Every entry in `subscription.prices` is an explicit `SubscriptionPrice` you POST. The `--equalize-all-territories` flag on `sub pricing set` mimics the web UI's auto-fill by walking `subscriptionPricePoints/{id}/equalizations` and POSTing one record per territory.
- **`isPreserveCurrentPrice` keeps existing subscribers on their old price** — On `SubscriptionPrice` POST, set `isPreserveCurrentPrice: true` to grandfather existing subscribers when raising prices. Apple's behavior:
  - **Decrease** (new < current): existing subscribers automatically get the lower price; `isPreserveCurrentPrice` is meaningless. `sub pricing set` warns and prompts interactively. Under `-y`, requires `--confirm-decrease` to acknowledge the revenue impact — plain `-y` is not enough.
  - **Increase** (new > current): the dev MUST decide. `--preserve-current` keeps existing subs at old price; `--no-preserve-current` pushes new price after Apple's notification period. The `sub pricing set` command errors if neither flag is passed for any increase (single territory or any territory in `--equalize-all-territories`).
  - **New territory** (no current price): no existing subs to consider; the flag is unnecessary.
  - **Unchanged**: `sub pricing set` skips silently. In equalize mode, only changed territories are POSTed.

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

### App info localization JSON format (used by app-info export/import)
```json
{
  "en-US": {
    "name": "My App",
    "subtitle": "Best app ever",
    "privacyPolicyURL": "https://example.com/privacy",
    "privacyChoicesURL": "https://example.com/choices"
  }
}
```

Same convention — only fields present get updated. The `AppInfoLocaleFields` struct in AppsCommand.swift defines the schema. The `app-info update` and `app-info import` commands check that the AppInfo is in an editable state (`PREPARE_FOR_SUBMISSION` or `WAITING_FOR_REVIEW`) before proceeding.

### Media upload folder structure (used by media upload)
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
- `media verify` checks all media status; with `--folder` retries stuck items: delete → upload → reorder
- File matching: server position N = Nth file alphabetically in local `locale/displayType/` folder

## Screenshot Module (`ascelerate screenshot`)

Captures App Store screenshots from iOS/iPadOS simulators. Replaces fastlane snapshot.

### Architecture

```
Sources/ascelerate/
  Screenshot/
    ScreenshotConfig.swift       # YAML config model (via Yams), validation
    ScreenshotRunner.swift       # Orchestrator: build → boot simulators → run tests → collect
    ScreenshotTestRunner.swift   # xcodebuild wrapper: build-for-testing + test-without-building
    ScreenshotCollector.swift    # Moves PNGs from per-device cache to output dir
    SimulatorManager.swift       # xcrun simctl: boot, shutdown, erase, localize, status bar, dark mode
    ScreenshotShell.swift        # Process wrapper: run (capture), stream (passthrough), runToLog (to file)
    ScreenshotError.swift        # Error types
  Commands/
    ScreenshotCommand.swift      # Subcommands: run, init, create-helper + embedded ScreenshotHelper.swift
```

### Config (`ascelerate/screenshot.yml`)

Config is decoded directly as `ScreenshotConfig` (no wrapper key). Created by `ascelerate screenshot init` in the `ascelerate/` directory.

```yaml
# project: App.xcodeproj
workspace: App.xcworkspace
scheme: AppUITests
devices:
  - simulator: iPhone 16 Pro Max
  - simulator: iPad Pro 13-inch (M4)
languages: [en-US, tr-TR]
outputDirectory: ./screenshots
clearPreviousScreenshots: true
eraseSimulator: false
localizeSimulator: true
# darkMode: false
# disableAnimations: false
# waitAfterBoot: 0
overrideStatusBar: true
# statusBarArguments: "--time '9:41' --dataNetwork wifi"
# configuration: Release
# testWithoutBuilding: true
# cleanBuild: false
# headless: false
# helperPath: AppUITests/ScreenshotHelper.swift
# testplan: MyTestPlan
# numberOfRetries: 0
# stopAfterFirstError: false
# reinstallApp: com.example.MyApp
# xcargs: -resultBundlePath ./results
```

### Flow

1. `build-for-testing` with `generic/platform=iOS Simulator` and `-configuration` (default Release)
2. xcodebuild writes to project's actual derived data (custom `-derivedDataPath` is ignored by Xcode workspace settings)
3. Resolve xctestrun file from `~/Library/Developer/Xcode/DerivedData/{ProjectName}[-hash]/Build/Products/`
4. For each language: boot all simulators → wait (`waitAfterBoot`) → set dark mode → localize → override status bar → uninstall app (`reinstallApp`) → `test-without-building` concurrently per device → collect screenshots
5. Each device gets isolated cache at `~/Library/Caches/tools.ascelerate/{UDID}/`
6. ScreenshotHelper.swift uses `SIMULATOR_UDID` env var to find its cache directory
7. Errors skip failing device/language, error logs saved as `{language}/{device}-error.log` (unless `stopAfterFirstError`)
8. Summary table printed at end

### Key decisions

- `-parallel-testing-enabled NO` prevents simulator cloning (needed for status bar override)
- `xcrun simctl bootstatus` waits for full boot before applying status bar override
- Test output goes to log files (not stdout) to prevent interleaved output from concurrent devices
- Helper version tracked via `// ScreenshotHelperVersion [X.Y]` comment in generated file
- `clearPreviousScreenshots` only clears per-language after all devices succeed
- `numberOfRetries` — retries failed languages by erasing the failed simulators, re-localizing, rebooting, and rerunning tests. Works around simulator localization bugs where translated labels aren't found. Only failed devices are retried.
- `testplan` and `xcargs` are passed to both build and test phases
- `disableAnimations` passes `-ASC_DISABLE_ANIMATIONS YES` as a launch argument; app must call `disableAnimationsIfNeeded()` to act on it
- SIGINT handler (via `sigaction`) forwards Ctrl-C to child xcodebuild processes
- **Helper uses `app.screenshot()`, not `XCUIScreen.main.screenshot()`.** XCUIScreen captures the simulator framebuffer via the render server, so if the target app has terminated at the moment of capture (seen intermittently on iPad when the last test tap triggers a locale-dependent crash in the app), the call silently returns a home-screen PNG and the test "passes" with a bad screenshot. `app.screenshot()` RPCs into the target process directly — if the process is gone, it throws `Lost connection to the application (pid N)`, XCTest marks the test failed, and `numberOfRetries` retries the device+language automatically (usually succeeds on retry since the crash is intermittent). Do not switch back to XCUIScreen without a replacement detection mechanism.

### Commands

```
ascelerate screenshot [-l en-US,tr-TR]        # Capture screenshots (optionally filter to a subset of configured languages)
ascelerate screenshot init                    # Create ascelerate/screenshot.yml + ScreenshotHelper.swift
ascelerate screenshot create-helper [-o file] # Generate ScreenshotHelper.swift (default: ascelerate/)
```

## Not Yet Implemented

asc-swift exposes the full App Store Connect surface (~185 top-level v1 resources). ascelerate wraps **63** of them — deep coverage where it exists (apps, versions, version/app-info localizations, screenshots/previews, full IAP + subscriptions, provisioning, review submissions + App Review Information (details + attachments), builds, territories/availability, encryption, EULA, age rating, routing coverage, customer reviews + responses, in-app events incl. localizations + media), but several whole product areas are untouched. Gap analysis last refreshed against **asc-swift 1.7.0**.

### Partially covered
- **Monetization** — IAP and subscriptions have CRUD + localizations + pricing (per-territory overrides for IAPs, equalize fan-out for subs with increase/decrease safety) + per-product availability + app-level grace period + subscription introductory offers + IAP/sub offer codes (with one-time-use + custom code generation) + subscription promotional offers + group submissions + IAP/sub promotional images + App Review screenshots. Remaining: **win-back offers** (blocked: asc-swift's `WinBackOfferPriceInlineCreate` doesn't expose territory/pricePoint relationships — can't reach the API correctly until the generator is updated. Confirmed still broken as of asc-swift 1.7.0: it remains the only `*PriceInlineCreate` type with just `type`+`id`, while all siblings — IAP/sub price, offer-code, promotional-offer — carry territory/pricePoint. The read-side `WinBackOfferPrice` does expose them, so it's specifically the CreateAPI-generated inline-create schema that's wrong), IAP hosted content (`inAppPurchaseContents`; read-only via API; low value).
- **Promoted purchases** — read-only (`iap promoted` lists via `apps/{id}/promotedPurchases`); no create/reorder/delete of the promoted-purchase set.
- **Build upload** — done via `altool` (binary upload); the API-native `buildUploads`/`buildUploadFiles`/`buildBundles` path is intentionally unused.
- **Custom product pages** — page CRUD (`product-pages list/info/create/update/delete`; create is a compound version+localization request using `${local-id}` inline references) + localizations (`product-pages localizations view/export/import`; promotional text per locale) + media (`product-pages media list/upload/delete`; per-locale screenshot sets by display type and app preview sets). Remaining: **per-locale search-keyword linkages** (blocked, same class of asc-swift codegen gap as win-back offers — as of 1.7.0 the `AppKeyword` entity exposes no attributes, only `type`+`id`+`links`, so the keyword *text* is unavailable, and there's no endpoint to create keywords. The relationship endpoints exist — `appCustomProductPageLocalizations/{id}/relationships/searchKeywords` GET/POST/DELETE link keyword IDs to a localization, sourced from the read-only app pool `apps/{id}/searchKeywords` — but a command could only shuffle opaque, undisplayable IDs. Not shippable until `AppKeyword` exposes the keyword text).
- **App metadata** — no commands for app tags, app categories CRUD, app clips, nominations (editorial).

### Missing entirely
Counts are approximate top-level resources from the 1.7.0 surface.
- **TestFlight / Beta** (~20) — beta groups, testers, tester invitations, beta app/build localizations, beta app review submissions/details, crash logs, feedback (crash + screenshot) submissions, recruitment criteria, prerelease versions, build beta details/notifications. *Largest single gap.*
- **Game Center** (~38) — achievements, leaderboards (+sets), challenges, activities, matchmaking (queues/rules/teams), groups, details, enabled versions, releases.
- **Xcode Cloud (CI/CD) + SCM** (~17) — `ci*` (products, workflows, build runs/actions, artifacts, issues, test results, macOS/Xcode versions) and `scm*` (providers, repositories, git refs, pull requests).
- **A/B experiments** (3) — App Store version experiments + treatments + treatment localizations.
- **App Clips** (7) — app clips, advanced/default experiences, header/advanced images, app-clip review details.
- **Background Assets** (6) — assets, versions, upload files, app-store/internal/external beta releases.
- **Alternative distribution / EU** (~8) — `alternativeDistribution*` (domains, keys, packages + variants/deltas/versions) and `marketplace*` (search details, webhooks).
- **Webhooks** (3) — webhooks + deliveries + pings (ASC event notifications; newer).
- **Analytics / Sales / Finance** (~7) — analytics reports (+requests/instances/segments), sales reports, finance reports, diagnostic signatures.
- **Users & access** (3) — users, user invitations, actors (team management).
- **App-level pricing & release** — `appPriceSchedules` (paid-app pricing), `subscriptionPlanAvailabilities`, `appStoreVersionPromotions`, `appStoreVersionReleaseRequests`, `endAppAvailabilityPreOrders`.
- **Provisioning extras** — `merchantIDs` (Apple Pay), `passTypeIDs` (Wallet).
- **Niche** — accessibility declarations, `appEncryptionDeclarationDocuments`, Android-to-iOS app mapping.

## Release build note

`swift build -c release` is very slow due to whole-module optimization of AppStoreAPI's ~2500 generated files. Debug builds are fast for development.


<claude-mem-context>
# Recent Activity

<!-- This section is auto-generated by claude-mem. Edit content outside the tags. -->

*No recent activity*
</claude-mem-context>