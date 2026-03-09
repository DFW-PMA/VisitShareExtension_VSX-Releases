# VV Share Extension

A universal Share Extension for the JustMacApps healthcare suite. One extension, multiple target apps.

## Architecture

```
VisitShareExtension.app (Helper/Container App)
└── VisitShareExtensionAppEx.appex (Share Extension)
        ↓
    Picker UI: "Send to which app?"
        ↓
    ┌─────────────────┬─────────────────┬─────────────────┐
    │  VisitReportingApp     │  VisitManagementApp   │    VisitSchedulingApp   │
    │  (Tickets)      │  (Management)   │    (MetaData)   │
    └─────────────────┴─────────────────┴─────────────────┘
```

## How It Works

1. User selects text in Messages (or any app)
2. Taps Share → VV Share Extension
3. Extension shows picker with available target apps
4. User selects destination (e.g., "Create Ticket in VisitVerify")
5. Data written to shared App Group container
6. Target app launches via URL scheme
7. Target app reads data and processes it

## App Group Strategy

All JustMacApps products share a single App Group:
```
group.com.PreferredMobileApplications.sharedVisitApps1
```

This allows:
- Single container for all shared data
- Any app can read handoffs intended for it
- Simpler entitlement management

## Project Structure

```
VisitShareExtension/
├── Shared/                         # Add to ALL targets
│   ├── VVSharedConfig.swift       # App Group, URL schemes, app definitions
│   ├── VVMessageHandoff.swift     # Data model for transfers
│   └── VVSharedTargetApps.swift          # Target app enumeration
├── HelperApp/                      # Helper app target only
│   ├── VisitShareExtensionApp.swift     # App entry point
│   ├── HelperContentView.swift     # Main UI with instructions
│   ├── AppStatusView.swift         # Shows installed apps
│   └── Info.plist
├── ShareExtension/                 # Extension target only
│   ├── ShareViewController.swift   # Extension entry point
│   ├── AppPickerView.swift         # SwiftUI picker UI
│   ├── ShareExtension.entitlements
│   └── Info.plist
└── TargetAppIntegration/           # Reference code for target apps
    ├── TargetAppIntegration.swift  # Drop-in handler for target apps
    └── README.md                   # Integration instructions
```

## Xcode Setup

### 1. Create the Helper App Project

1. File → New → Project → iOS App
2. Product Name: `VisitShareExtension`
3. Organization Identifier: `com.PreferredMobileApplications`
4. Interface: SwiftUI

### 2. Add Share Extension Target

1. File → New → Target → Share Extension
2. Product Name: `VisitShareExtensionAppEx`
3. Uncheck "Include UI Extension"
4. Activate the 'extension'...

### 3. Configure App Group (All Targets)

**Apple Developer Portal:**
1. Identifiers → App Groups → Add
2. Create: `group.com.PreferredMobileApplications.sharedVisitApps1`

**In Xcode (Helper App + Extension):**
1. Select target → Signing & Capabilities
2. + Capability → App Groups
3. Check `group.com.PreferredMobileApplications.sharedVisitApps1`

**In each Target App (VisitVerify, VMA, etc.):**
1. Add the same App Group capability
2. Add the integration code from `TargetAppIntegration/`

### 4. Add Source Files

**Shared folder** → Add to BOTH Helper App and Extension targets

**HelperApp folder** → Helper App target only

**ShareExtension folder** → Extension target only

## URL Schemes

Each target app must register its URL scheme:

| App | URL Scheme | Example URL |
|-----|------------|-------------|
| VisitReportingApp | `visitreportingapp` | `visitreportingapp://ticket?source=share&id=...` |
| VisitManagementApp | `visitmanagementapp` | `visitmanagementapp://management?source=share&id=...` |
| VisitSchedulingApp | `visitschedulingapp` | `visitschedulingapp://metadata?source=share&id=...` |

## Adding a New Target App

1. Add entry to `VisitShareExtension.swift`
2. Register App Group in new app's entitlements
3. Add URL scheme to new app's Info.plist
4. Integrate `VisitShareExtensionAppEx` in new app
5. Rebuild the Share Extension

## Testing

1. Build and run VisitShareExtension (installs extension)
2. Build and run at least one target app (VisitManagementApp, etc.)
3. Open Messages, select text, tap Share
4. Choose "VisitShareExtension"
5. Select target app from picker
6. Verify app launches and receives data

## Troubleshooting

**Extension doesn't appear:**
- Check NSExtensionActivationRule in Info.plist
- Ensure extension is enabled in Settings
- Restart device (share sheet caches aggressively)

**"App not installed" in picker:**
- Target app must be built and run at least once
- URL scheme must be registered in target app's Info.plist
- Use `canOpenURL` check (requires LSApplicationQueriesSchemes)

**Handoff data not received:**
- Verify App Group identifier matches exactly everywhere
- Check that target app has App Group entitlement
- Look for errors in Console.app

## Build and Test - Appendix:

# Build & Test Workflow

## How It Works

When you build VisitShareExtension, Xcode automatically:
1. Builds VisitShareExtensionAppEx (it's a dependency)
2. Embeds the `.appex` inside the helper app bundle
3. Code signs both

You just select the **helper app scheme** and hit Run.

## Xcode Setup to Verify

1. **Check the embed setting:**
   - Select VisitShareExtension target
   - Go to **General** tab → **Frameworks, Libraries, and Embedded Content**
   - VisitShareExtensionAppEx.appex should be listed with **Embed & Sign**

2. **If it's not there:**
   - Click the **+** button
   - Select VisitShareExtensionAppEx.appex from the list
   - Set to "Embed & Sign"

3. **Scheme selection:**
   - Use the scheme dropdown (next to the device selector)
   - Select **VisitShareExtension** (not the extension)
   - The extension scheme exists but is mainly for debugging the extension in isolation

## Testing Flow

```
1. Select scheme: VisitShareExtension
2. Select device: Your iPhone (or Simulator)
3. Build & Run (⌘R)
4. Helper app installs (extension comes along embedded)
5. Open Messages, select text, tap Share
6. "VisitShareExtension" should appear in share sheet
```

## Debugging the Extension

If you need to debug the extension code specifically:

1. Set breakpoints in ShareViewController.swift or AppPickerView.swift
2. Build & Run the helper app as normal
3. When you invoke the share sheet and select your extension, Xcode will hit those breakpoints

Alternatively, you can run the extension scheme directly:

1. Select **VisitShareExtensionAppEx** scheme
2. Run → Xcode asks "Choose an app to run"
3. Select Messages (or any app with sharing)
4. Xcode launches that app, waits for you to invoke the extension

## Verifying Extension is Embedded

After building, verify the extension is embedded:

```bash
# In Terminal, after building to a device
# Check the app bundle structure
ls -la ~/Library/Developer/Xcode/DerivedData/VisitShareExtension-*/Build/Products/Debug-iphoneos/VisitShareExtension.app/PlugIns/
```

You should see `VisitShareExtensionAppEx.appex` in there.

## Summary

Just build the helper app target—the extension comes along for the ride. No archive needed for testing.

