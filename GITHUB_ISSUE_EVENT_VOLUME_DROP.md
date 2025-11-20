# Issue: Event volume drop (~30-44%) and missing super properties after Unity SDK v3.5.3 migration

## Issue Summary

After migrating from native embedded SDKs to Unity SDK v3.5.3, production games are experiencing:
1. **Event volume drop of ~30-44%** for `App Start` and `App Exit` events on Android (Google Play) only
2. **Missing super properties** on the first event of every session (`App Start`) on both iOS and Android

## Customer Information

- **Customer**: Sago Mini (Production environment)
- **Affected Games**: School, World
- **Migration**: Native SDKs → Unity SDK v3.5.3
  - **Previous Android SDK**: 5.3.0 via Gradle
  - **Previous iOS SDK**: Custom wrapper at commit [0b982aab58](https://github.com/mixpanel/mixpanel-iphone/tree/0b982aab58e832ad85a8d9dca4d9729aa3948b53)
- **Rollout Date**: October 1st, 2025
- **Zendesk Case**: [#674175](https://mixpanelsupport.zendesk.com/agent/tickets/674175)

## Detailed Symptoms

### 1. Event Volume Drop (Android Only)

- **Platform Affected**: Android (Google Play) only
- **Events Affected**: `[School] App Start` and `[School] App Exit`
- **Magnitude**:
  - ~30% drop in total events
  - ~44% drop in unique users for `App Exit`
- **Data**: [Mixpanel Insights Report](https://mixpanel.com/project/210340/view/23451/app/insights#fTLhAYFBMnRZ)
- **iOS**: Event counts appear normal
- **Important**: No data loss for events between App Start and App Exit
- **DAU**: Not affected (confirmed by App Store/Play Store data)

### 2. Missing Super Properties

- **Platform Affected**: Both iOS and Android
- **Events Affected**: First event in session (`App Start`)
- **Behavior**: All super properties are missing from the first tracked event
- **Subsequent Events**: Super properties present on all other events in the session

## Expected Behavior

- Event volumes should remain consistent with pre-migration levels
- Super properties should be present on ALL events, including the first event in a session
- Games not yet migrated continue to report expected volumes with super properties present

## Technical Context

### Customer Initialization Flow

```csharp
// Bootstrap scene - early in game lifecycle
Mixpanel.Init();
// ... immediately followed by ...
RegisterSuperProperties(); // Register super properties
TrackAppStart(); // Fires "App Start" event
```

### Observations from Code Review

**Potential Race Condition:**

1. `Mixpanel.Init()` appears synchronous (not a coroutine/Task)
2. Controller's `Start()` method runs asynchronously via Unity lifecycle
3. Super properties are lazy-loaded from disk on first access
4. If `App Start` fires before super properties are loaded from disk, they will be missing

**Relevant Code Locations:**

- `Controller.cs:48-52` - `Initialize()` method
- `Controller.cs:90-95` - `Start()` coroutine (runs migration, starts auto-flush)
- `Controller.cs:325-352` - `DoTrack()` merges super properties (line 334)
- `Storage.cs:275-293` - `SuperProperties` lazy-loading from PlayerPreferences

**Critical Code Path:**

```csharp
// Controller.cs:48-52
internal static void Initialize() {
    // Copy over any runtime changes that happened before initialization from settings instance to the config.
    MixpanelSettings.Instance.ApplyToConfig();
    GetInstance(); // Creates singleton
}

// Controller.cs:64-73
internal static Controller GetInstance() {
    if (_instance == null) {
        GameObject g = new GameObject ("Mixpanel");
        _instance = g.AddComponent<Controller>();
        DontDestroyOnLoad(g);
    }
    return _instance;
}

// Controller.cs:90-95
private void Start() {
    MigrateFrom1To2();
    Mixpanel.Log($"Mixpanel Component Started");
    StartCoroutine(WaitAndFlush());
}

// Controller.cs:325-352 - DoTrack merges super properties
internal static void DoTrack(string eventName, Value properties) {
    if (!MixpanelStorage.IsTracking) return;
    if (properties == null) properties = new Value();
    properties.Merge(GetEventsDefaultProperties());
    // These auto properties can change in runtime so we don't bake them into AutoProperties
    properties["$screen_width"] = Screen.width;
    properties["$screen_height"] = Screen.height;
    properties.Merge(MixpanelStorage.OnceProperties);
    properties.Merge(MixpanelStorage.SuperProperties); // <--- Line 334: Super properties merged here
    // ...
}

// Storage.cs:275-293 - SuperProperties getter (lazy loads from disk)
internal static Value SuperProperties {
    get {
        if (_superProperties != null) return _superProperties;
        if (!PreferencesSource.HasKey(SuperPropertiesName)) SuperProperties = new Value();
        else {
            _superProperties = new Value();
            JsonUtility.FromJsonOverwrite(PreferencesSource.GetString(SuperPropertiesName), _superProperties);
        }
        return _superProperties;
    }
    set {
        _superProperties = value;
        PreferencesSource.SetString(SuperPropertiesName, JsonUtility.ToJson(_superProperties));
    }
}
```

### Customer Code Flow

Customer provided sequence diagram showing:
1. Bootstrap scene initializes early
2. `Mixpanel.Init()` called
3. Super properties registered via wrapper
4. `App Start` event fired immediately after
5. Auto-flush happens every 60 seconds

### App Exit Event Handling

- SDK relies on Unity lifecycle events
- Auto-flush occurs every 60 seconds (`Config.FlushInterval`)
- If app exits between flush cycles, queued events should be sent on next launch
- Historical issue: `OnApplicationQuit` race condition was fixed in v3.0.0 (CHANGELOG line 164)

### Initialization Lifecycle Analysis

**Current Behavior:**

1. `Mixpanel.Init()` is called
2. `Controller.Initialize()` runs synchronously:
   - Applies config settings
   - Creates GameObject with Controller component
3. Control returns to caller (appears "ready")
4. Unity lifecycle eventually calls `Controller.Start()`:
   - Runs migration logic
   - Starts auto-flush coroutine
5. Meanwhile, customer code:
   - Registers super properties (writes to PlayerPreferences)
   - Fires `App Start` event immediately
6. `DoTrack()` attempts to merge super properties:
   - Lazy-loads from PlayerPreferences
   - **Race condition**: Properties might not be fully persisted/loaded yet

## Reproduction

### Steps Attempted by Customer

1. Download and install School app from [Google Play](https://play.google.com/store/apps/details?id=com.sagosago.School.googleplay)
2. Open app, log into Piknik account, pass title screen, close app
3. Reopen app, pass title screen, wait 1 min for flush
4. Close app
5. Check events in Mixpanel

**Result**: Unable to reproduce locally; issue only visible in production data with live traffic

### Testing Requirements

- Production-scale traffic patterns
- Real device testing (not emulator)
- Multiple session lifecycles
- Android focus (primary issue platform)
- Various device specifications (low-end to high-end)

## What Customer Has Checked

✅ Initialization paths appear to run (`Mixpanel.Init` + subsequent calls)
✅ Super property registration code still executes (same location as before)
✅ No recent logic changes that would gate or suppress tracking
✅ No obvious token or environment mismatches
✅ Migration kept existing C# wrapper/util classes
✅ Debug logs provided (with `ShowDebug = true`)
✅ Verified DAU from App/Play Store (not affected)
✅ Confirmed no data loss for mid-session events

## Investigation Status

**Latest Update** (from Donna Chow, Mixpanel Support - Nov 19, 2025):
> "We may have narrowed down the cause of this issue. We are actively working on a fix/pr for the Unity SDK."

**Key Support Engineer Comments:**

**Lin Yee Koh** (Nov 5, 2025):
> "Mixpanel Unity (v3.5.3) initializes asynchronously. It is possible that the first event can fire before the instance is ready or before the super-properties file is loaded from disk."

**Customer Response** (Guilherme Quadros da Silva - Nov 6, 2025):
> "The Mixpanel.Init() call seems sync. It is not a coroutine or Task method. It just applies the configs from the scriptable object and then initialize a singleton instance. How can we wait for the initialization finish? I didn't find an event or something that would tell the call finished."

## Requested Features/Fixes

### 1. Initialization Callback/Event

Provide a way to know when SDK is fully initialized and ready to track events:

```csharp
// Option A: Callback
Mixpanel.Init(onComplete: () => {
    RegisterSuperProperties();
    TrackAppStart();
});

// Option B: Async/await
await Mixpanel.InitAsync();
RegisterSuperProperties();
TrackAppStart();

// Option C: Event
Mixpanel.OnInitialized += () => {
    RegisterSuperProperties();
    TrackAppStart();
};
Mixpanel.Init();

// Option D: IsReady check
Mixpanel.Init();
while (!Mixpanel.IsReady()) {
    yield return null;
}
RegisterSuperProperties();
TrackAppStart();
```

### 2. Race Condition Fix

Ensure super properties are reliably available on first event:
- Load super properties synchronously during `Init()` before returning
- Preload super properties in `InitializeBeforeSceneLoad()`
- Queue events until SDK is fully initialized
- Document timing requirements clearly

### 3. App Exit Event Reliability

Investigate Android-specific event loss:
- Review Android lifecycle handling vs iOS
- Improve event persistence/queuing for app termination scenarios
- Consider force-flush on `OnApplicationQuit` or `OnApplicationPause(true)`
- Add platform-specific lifecycle handling if needed
- Test on various Android OS versions and OEMs

### 4. Documentation

Add clear documentation about:
- When it's safe to call `Track()` after `Init()`
- Best practices for registering super properties
- Initialization lifecycle and async considerations
- Platform-specific differences in lifecycle handling

## Additional Resources

- **Customer logs**: Provided via Slack (adb logcat with `ShowDebug = true`)
- **Customer wrapper class**: `SagoMixpanelBinding` (provided to support team)
- **Sequence diagram**: Initialization and event flow (provided to support team)
- **Mixpanel data**: Charts showing event volume drop over time

## Priority

**High** - Affecting production games with significant user base, causing:
- 30-44% data loss for critical session boundary events
- Missing super properties impacting analytics accuracy and data quality
- Customer blocked on migrating other titles until resolved
- Potential impact on other customers experiencing similar issues

## Impact Assessment

- **Data Quality**: Critical analytics data missing or incomplete
- **Customer Trust**: Production issue affecting business metrics
- **Migration Risk**: Blocks adoption of Unity SDK for customers with similar patterns
- **User Base**: Large-scale production games affected
- **Revenue Impact**: Customer relies on this data for product decisions

## Suggested Labels

`bug`, `production`, `high-priority`, `android`, `ios`, `initialization`, `super-properties`, `event-loss`, `race-condition`, `v3.5.3`

## Related Issues

- [#69](https://github.com/mixpanel/mixpanel-unity/issues/69) - `OnApplicationQuit` possible race condition (fixed in v3.0.0)
- [#119](https://github.com/mixpanel/mixpanel-unity/pull/119) - Tracking persistence layer refactor (v3.0.0)

## Contacts

- **Customer**: Guilherme Quadros da Silva, Edwin Jo Mathew, Brennan Clark (Sago Mini)
- **Support Team**: Lin Yee Koh, Zach Bates, Donna Chow, Argenis Ferrer Mora
- **Engineering**: Jared McFarland (Mixpanel Mobile Engineering)
