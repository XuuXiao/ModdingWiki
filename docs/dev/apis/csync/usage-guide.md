---
prev: false
next: true
description: The main guide to using CSync (v5).
---

# Guide to using CSync

::: warning

## CSync 5 is unreleased.
Please use the [CSync 4 usage guide](/dev/apis/csync/v4-usage-guide) for the time being.

:::

## 1. Creating your config class

Create a new class extending `SyncedConfig2`.

```csharp
public class MyConfig : SyncedConfig2<Config>
```

Within this class, we declare fields for each config entry we would like to sync.
Optionally, you can also declare non-synced config entries too.
Each `SyncedEntry` must be annotated with the `[SyncedEntryField]` attribute.

```csharp
public class MyConfig : SyncedConfig2<Config> {
    public ConfigEntry<float> DebugLevel { get; private set; }
    
    [SyncedEntryField] public SyncedEntry<float> MovementSpeed; // fields may be annotated directly
    [field: SyncedEntryField] public SyncedEntry<float> ClimbSpeed { get; private set; } // properties must specify 'field' as attribute target
}
```

::: warning
When using client side and synced entries in the same class, any instance of `ConfigEntry` should **NOT** be marked with `[SyncedEntryField]`!
:::

## 2. Binding entries

We add a constructor to our config class that calls the base constructor.

```csharp
public class MyConfig : SyncedConfig2<Config> {
    public MyConfig(ConfigFile configFile) : base("My.Plugin.Guid") { } // [!code focus]
}
```

We bind our entries to the BepInEx config file like usual.
However, we will use the `BindSyncedEntry` extension method to bind `SyncedEntry` instances.
```csharp
public MyConfig(ConfigFile cfg) : base("My.Plugin.Guid") { 
    DebugLevel = cfg.Bind(
        new ConfigDefinition("General", "Debug Level"),
        0,
        new ConfigDescription("Debug logging level for the mod.")

    MovementSpeed = cfg.BindSyncedEntry(
        new ConfigDefinition("Movement", "Movement Speed"),
        4.1f,
        new ConfigDescription("The base speed at which the player moves.")
    );

    ClimbSpeed = cfg.BindSyncedEntry(
        new ConfigDefinition("Movement", "Climb Speed"),
        3.9f,
        new ConfigDescription("The base speed at which the player climbs.")
    );
}
```

After binding, we add the following line at the end of the constructor.
```csharp
ConfigManager.Register(this);
```

```csharp
public MyConfig(ConfigFile cfg) : base("My.Plugin.Guid") { 
    DebugLevel = cfg.Bind(
        new ConfigDefinition("General", "Debug Level"),
        0,
        new ConfigDescription("Debug logging level for the mod.")

    MovementSpeed = cfg.BindSyncedEntry(
        new ConfigDefinition("Movement", "Movement Speed"),
        4.1f,
        new ConfigDescription("The base speed at which the player moves.")
    );

    ClimbSpeed = cfg.BindSyncedEntry(
        new ConfigDefinition("Movement", "Climb Speed"),
        3.9f,
        new ConfigDescription("The base speed at which the player climbs.")
    );
    
    ConfigManager.Register(this); // [!code ++]
}
```

## 3. Instantiating the config class

We declare an `internal static` field to hold our config instance so that it is accessible
from anywhere in our project. Note the `new` modifier is only necessary when naming the field
`Config`.

We instantiate the config class in the `Awake` method of our BepInEx plugin and assign our
field to the new instance.
```csharp
internal static new MyConfig Config; // [!code ++]

static void Awake() {
    Config = new MyConfig(base.Config); // [!code ++]
}    
```

## 4. Declaring dependency

We declare CSync a dependency of our BepInEx plugin by adding a `BepInDependency` attribute that
specifies CSync's plugin GUID.

```csharp
[BepInPlugin(PluginInfo.PLUGIN_GUID, PluginInfo.PLUGIN_NAME, PluginInfo.PLUGIN_VERSION)]
[BepInDependency("com.sigurd.csync", "5.0.0")] // [!code ++]
public class MyPlugin : BaseUnityPlugin
```

If we plan to upload our mod to **Thunderstore**, we must ensure we specify the dependency within our
`manifest.json` file by adding CSync's **Thunderstore** package ID to the dependency array.
```json
{
  "dependencies": ["BepInEx-BepInExPack-5.4.2100", "Sigurd-CSync-5.0.0"] // [!code focus]
}
```

::: info NOTE
Please ensure your manifest contains the latest version, the one seen above may be outdated!
:::

## 5. Using your config

We may reference our config entries from the field we declared in our BepInEx plugin.
```csharp
SetMovementSpeed(MyPlugin.Config.MovementSpeed.Value);
```

Try not to cache any entries' values in a variable; doing so means it is possible
that you are not using the most up-to-date value.

You can get (or set) the local value of a `SyncedEntry`:

```csharp
SyncedEntry<int> myEntry;
DoSomethingWith(myEntry.LocalValue);
myEntry.LocalValue = 5;
```

Note that you **cannot** set the `Value` of a `SyncedEntry` - you can only assign to
`.LocalValue`.

## 6. Events

The following events are available for convenience:

### `SyncedConfig2<T>.InitialSyncComplete`

Fires:
- on the host, immediately after initially packing all the entries' values
- on the client, immediately after initially unpacking all the entries' values

```csharp
ManualLogSource logger;
MyConfig Config;

Config.InitialSyncComplete += (sender, args) => {
    logger.LogInfo("Initial sync complete!"); 
};
```

### `SyncedEntry<T>.Changed`

Fires:
- on the host, when the local value is changed
- on the client, when the host's local value is changed
- on the client, when disconnecting from a session

```csharp
ManualLogSource logger;
SyncedEntry<float> myEntry;

myEntry.Changed += (sender, args) => {
    logger.LogInfo($"The old value was {args.OldValue}");
    logger.LogInfo($"The new value is {args.NewValue}");
}
```
