# Fixing AzuHoverStats After the Valheim 1.0 Update

Valheim 1.0 (shipped Sept 9, 2026) upgraded the Unity engine and changed several game APIs. AzuHoverStats uses Harmony patches that target those APIs directly, so a handful of signature/visibility changes broke the mod's `_harmony.PatchAll()` call entirely — since Harmony aborts the whole patch batch if any single patch fails to bind, this means **none** of the mod's features work until every broken patch is fixed, not just the one causing the visible error.

This guide covers patching `AzuHoverStats.dll` directly with dnSpy. No Visual Studio or build toolchain needed — just dnSpy.

## What broke, and why

1. **`ItemDrop.ItemData.GetTooltip`** (static overload) gained a new trailing parameter, `bool appending = false`. AzuHoverStats' Harmony attribute hardcodes the full parameter list to find this specific overload, so it stopped matching.
2. **`WearNTear.GetSupport()`, `GetMaxSupport()`, `GetSupportColorValue()`** became `private`. The mod calls them directly from an external patch class, which no longer compiles/resolves.
3. **`Hud.OnHoverPiece`** gained two new overloads (previously there was only one). AzuHoverStats' patch attribute names the method with no parameter type list, so Harmony's lookup became ambiguous once multiple overloads existed with the same name.
4. **`Localization`** moved out of `assembly_valheim.dll` into a separate `assembly_utils.dll`. Not a code fix, just a reference-loading issue while editing.

## Prerequisites

- [dnSpy](https://github.com/dnSpyEx/dnSpy) (or dnSpyEx, the maintained fork)
- Your Valheim `Managed` folder: `Valheim\valheim_Data\Managed\`
- Your BepInEx `core` folder: `BepInEx\core\`

## Step 1 — Load reference assemblies into dnSpy

dnSpy's built-in C# editor ("Edit Class") only resolves types from assemblies it has loaded into its own Assembly Explorer — it won't pull them from disk automatically. Before editing anything, use **File → Open** in dnSpy and load these as separate modules:

| File | Location |
|---|---|
| `0Harmony.dll` | `BepInEx\core\` |
| `BepInEx.dll` | `BepInEx\core\` |
| `assembly_valheim.dll` | `Valheim\valheim_Data\Managed\` |
| `UnityEngine.CoreModule.dll` | `Valheim\valheim_Data\Managed\` |
| `Unity.TextMeshPro.dll` | `Valheim\valheim_Data\Managed\` |
| `assembly_guiutils.dll` | `Valheim\valheim_Data\Managed\` |
| `assembly_utils.dll` | `Valheim\valheim_Data\Managed\` |

## Step 2 — Open the mod DLL directly, in place

Open the **actual file r2modman/your mod manager loads at runtime** — not a copy, not a re-zipped/re-imported version:

```
<your profile>\BepInEx\plugins\Azumatt-AzuHoverStats\AzuHoverStats.dll
```

Editing a copy elsewhere and expecting it to take effect won't work — you'll edit and save successfully, but the game will keep loading the original.

## Step 3 — Apply the fixes

In the Assembly Explorer, navigate to `AzuHoverStats` → `HoverTextPatches`, right-click the class → **Edit Class (C#)**. (Editing one nested patch class recompiles the whole containing class, so all sibling patch classes need to compile together — this is why all the reference assemblies above are needed even though any single fix only touches one small patch.)

**Fix 1 — `GetTooltip`:** add a 6th entry to the `Type[]` array:

```csharp
[HarmonyPatch(typeof(ItemData), "GetTooltip", new Type[]
{
    typeof(ItemData),
    typeof(int),
    typeof(bool),
    typeof(float),
    typeof(int),
    typeof(bool)   // <-- new
})]
private static class ItemDropItemDataGetTooltipPatch
{
    private static void Postfix(ItemData item, ref string __result)
    {
        // unchanged
    }
}
```

**Fix 2 — `WearNTear` private members:** replace direct calls with Harmony's `Traverse`, and replace `ShaderProps` field references with `Shader.PropertyToID(...)`:

```csharp
[HarmonyPatch(typeof(WearNTear), "Highlight")]
private static class WearNTearHighlightPatch
{
    private static void Postfix(WearNTear __instance)
    {
        if (AzuHoverStatsPlugin.customIntegrityColors.Value == AzuHoverStatsPlugin.Toggle.Off)
        {
            return;
        }

        Traverse t = Traverse.Create(__instance);
        float support = t.Method("GetSupport").GetValue<float>();
        float maxSupport = t.Method("GetMaxSupport").GetValue<float>();

        if (support < 0f || maxSupport < support) return;

        Color val;
        float colorValue = t.Method("GetSupportColorValue").GetValue<float>();
        if (colorValue >= 0f)
        {
            val = (support / maxSupport >= 0.5f)
                ? Color.Lerp(AzuHoverStatsPlugin.midIntegrityColor.Value, AzuHoverStatsPlugin.highIntegrityColor.Value, (support / maxSupport - 0.5f) * 2f)
                : Color.Lerp(AzuHoverStatsPlugin.lowIntegrityColor.Value, AzuHoverStatsPlugin.midIntegrityColor.Value, support / maxSupport * 2f);
        }
        else
        {
            val = new Color(0.6f, 0.8f, 1f);
        }

        MaterialMan.instance.SetValue(__instance.gameObject, Shader.PropertyToID("_EmissionColor"), val);
        MaterialMan.instance.SetValue(__instance.gameObject, Shader.PropertyToID("_Color"), val);
        __instance.CancelInvoke("ResetHighlight");
        __instance.Invoke("ResetHighlight", 0.2f);
    }
}
```

**Fix 3 — `OnHoverPiece` ambiguity and `m_hoveredPiece` privatization:**

```csharp
[HarmonyPatch(typeof(Hud), "OnHoverPiece", new Type[] { typeof(Piece) })]
private static class PlayerGetPiecePatch
{
    private static void Postfix(Hud __instance)
    {
        Piece hoveredPiece = Traverse.Create(__instance).Field("m_hoveredPiece").GetValue<Piece>();

        if (hoveredPiece == null || HudAwakePatch.ClonedSelectedInfoGo == null || hoveredPiece.m_comfort <= 0)
        {
            return;
        }

        RectTransform component = Hud.instance.m_buildHud.transform.Find("SelectedInfo").GetComponent<RectTransform>();
        float num = 1f;
        ConfigEntry<float> val = default;
        if (HudAwakePatch.MinimalUIInstance != null &&
            HudAwakePatch.MinimalUIInstance.Config.TryGetEntry<float>("2 - Inventory", "Selected Build Panel Scale", out val))
        {
            num = val.Value;
        }

        float num2 = 75f * num;
        float x = component.anchoredPosition.x;
        Rect rect = component.rect;
        float num3 = x + rect.width + num2;
        HudAwakePatch.ClonedSelectedInfoRt.anchoredPosition = new Vector2(num3, component.anchoredPosition.y);
        Vector3 localScale = component.transform.localScale;
        HudAwakePatch.ClonedSelectedInfoRt.transform.localScale = new Vector3(localScale.x, localScale.y, localScale.z);
        HudAwakePatch.ClonedSelectedInfoGo.SetActive(true);
        HudAwakePatch.ClonedSelectedInfoTmp.text = $"Comfort Level: {hoveredPiece.m_comfort}\nComfort Group: {hoveredPiece.m_comfortGroup}\n";
    }
}
```

`ConfigEntry<T>` requires `using BepInEx.Configuration;` at the top of the file if not already present.

## Step 4 — Compile and save

- Click **Compile**. If the window closes with no error list, it succeeded.
- Select the **module** node (top level, not the class) in the Assembly Explorer → **File → Save Module...** — this is a required, separate step from compiling; compiling only updates dnSpy's in-memory copy.
- Confirm the save path matches the exact file your mod manager loads from.

## Step 5 — Verify

- Fully close Valheim (check no lingering process in Task Manager).
- Relaunch through your mod manager.
- Check `BepInEx/LogOutput.log` for a clean load — no `AccessTools.DeclaredMethod` warnings, no `ArgumentException`/`AmbiguousMatchException` from Harmony.
- In-game: hover an inventory item (tooltip type/tier text), damage a building piece (integrity highlight color), and hover a comfort-giving piece in build mode (comfort level/group text).

## Ongoing caution

Back up your patched `AzuHoverStats.dll` outside the mod manager's profile folder. If your mod manager updates mods automatically and Thunderstore is still serving the pre-1.0 build, an "update all" can silently overwrite your fix. Once Azumatt ships an official 1.0-compatible release, drop your patched copy and go back to the normal Thunderstore version.
