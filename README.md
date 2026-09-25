# Cross-Scene-Object-Guide

# TLD modding: bringing objects from one scene into another /by KanarieWilfried



### Overview



Hi tld modding community, since there seemed to be quite some interest, this guide shows you how to take an object from one scene of The Long Dark and place it in another. For example placing a plane from Forsaken Airfield in Coastal Highway, or a prison wall from Blackrock into Pleasant valley, etc... This guide covers simple objects like 3D models, materials, colliders and LOD levels. Objects that need the game's own scripts to work (doors, containers, stoves) is much harder, and depends on the object, but if there is enough interest I might make a guide as well.



Before we start, many of the game's assets are Addressables, and can be loaded by their original file path from any scene. For example, Fuar's Improved Trader loads Assets/ArtAssets/Env/Objects/OBJ\_TradeBox/OBJ\_TradeBoxA\_Prefab.prefab, and Waltz's Safehouse Customization Plus loads Assets/Prefabs/Interactive/INTERACTIVE\_PotBellyStove.prefab

So check that first. This guide is for prefabs, meshes and material that have no addressable key, so they only exist inside a particular scene.





#### Tools needed:

* Unity Explorer (I use DZ's UnityExplorer with Waltztools)
* Assetriper
* Unity hub + the exact unity version the game uses. (for the last tld update its 6000.0.60f1) (you can find this easily at the top of your melonloader latest.log)
* .NET SDK + Visual Studio + Melonloader (like any other mod)





**In short this is the process:**

1\. Find the object in the game with UnityExplorer: its scene and its full path.

2\. List the bundle files that contain that scene, with some helper code.

3\. Extract those files with AssetRipper into a Unity project.

4\. Open that project in the Unity and make a prefab of the object and build an asset bundle from it.

5\. Load the bundle in a mod and place the object in the target scene.

6\. Fine-tune its position in game with UnityExplorer.



### Step 1: Find the object in the game



Go to the object in game and with unity explorer find the object you want to move. Write down 3 things:

* The scene's name, for example "AirfieldRegion"
* The object's full path, for example "Art/Helicopters/OBJ\_PLaneBeaver\_A\_Prefab Variant"
* Its Position and Rotation, which you'll need if the object is made of several parts.



##### 

### Step 2: Find the bundle files that hold the scene





The game's scenes and assets are stored in bundle files, under tld\_Data\\StreamingAssets\\aa\\. A scene is spread over its own bundle plus several shared ones because materials, textures and shaders often live in other bundles.

Here is code for a helper that finds the bundles your object needs. Add it to your mod temporarily, call it once while a save is loaded (for example from OnSceneWasInitialized), and remove it afterwards. (the helper was made with inspiration the way Safehouse Customization Plus uses Il2CppSystem.Linq.Enumerable.ToList, thank you Waltz<3)

You call it with the scene name, for example ListBundles("AirfieldRegion")

```
// Needs: using System.Linq; using UnityEngine.AddressableAssets;

//        using UnityEngine.ResourceManagement.ResourceLocations; using MelonLoader.Utils;

static void ListBundles(string sceneKey)

{

&#x20;   var bundles = new HashSet<string>();

&#x20;   var locations = Addressables.LoadResourceLocationsAsync(sceneKey, null).WaitForCompletion();

&#x20;   var list = Il2CppSystem.Linq.Enumerable.ToList(

&#x20;       locations.Cast<Il2CppSystem.Collections.Generic.IEnumerable<IResourceLocation>>());

&#x20;   for (int i = 0; i < list.Count; i++)

&#x20;       CollectBundles(list\[i], bundles, 0);



&#x20;   File.WriteAllLines(Path.Combine(MelonEnvironment.UserDataDirectory, sceneKey + "\_bundles.txt"), bundles.OrderBy(b => b));

}



// A scene's location lists the bundle files it depends on; those can depend on further bundles

static void CollectBundles(IResourceLocation loc, HashSet<string> bundles, int depth)

{

&#x20;   if (loc == null || depth > 10) return;

&#x20;   if (loc.ProviderId != null \&\& loc.ProviderId.Contains("AssetBundle") \&\& loc.InternalId != null)

&#x20;       bundles.Add(loc.InternalId.Replace("{UnityEngine.AddressableAssets.Addressables.RuntimePath}", Addressables.RuntimePath));

&#x20;   if (loc.Dependencies == null) return;

&#x20;   var deps = Il2CppSystem.Linq.Enumerable.ToList(

&#x20;       loc.Dependencies.Cast<Il2CppSystem.Collections.Generic.IEnumerable<IResourceLocation>>());

&#x20;   for (int i = 0; i < deps.Count; i++)

&#x20;       CollectBundles(deps\[i], bundles, depth + 1);

}```



Copy every file in the list from your game folder into an empty folder. Here's a simple powershell command that does that.



```

$gameDir   = "YOUR\_TLD\_DIRECTORY" # Where your tld.exe is

$scene     = "SCENE\_NAME"      # the scene you passed to ListBundles

$targetDir = "AN\_EMPTY\_FOLDER"

$listFile  = Join-Path $gameDir "UserData\\$($scene)\_bundles.txt"



\# Create the destination folder if it doesn't exist yet

if (-not (Test-Path $targetDir)) {

&#x20;   New-Item -ItemType Directory -Path $targetDir -Force | Out-Null

}



\# The helper writes one full file path per line, with forward slashes

$paths = @(Get-Content $listFile | Where-Object { $\_ -match '\\S' } | ForEach-Object { $\_.Trim() -replace '/', '\\' })

Write-Host "$($paths.Count) bundle files listed in $listFile"



\# Copy each file straight from its listed path

$copied  = 0

$missing = @()

foreach ($path in $paths) {

&#x20;   if (Test-Path -LiteralPath $path -PathType Leaf) {

&#x20;       Copy-Item -LiteralPath $path -Destination $targetDir -Force

&#x20;       $copied++

&#x20;   } else {

&#x20;       $missing += Split-Path $path -Leaf

&#x20;   }

}



\# A line that isn't a file on disk: search the game folder for that file name instead

if ($missing.Count -gt 0) {

&#x20;   Write-Host "Searching the game folder for $($missing.Count) file(s) not found at their listed path..."

&#x20;   Get-ChildItem -Path $gameDir -File -Recurse | Where-Object { $missing -contains $\_.Name } | ForEach-Object {

&#x20;       Copy-Item -LiteralPath $\_.FullName -Destination $targetDir -Force

&#x20;       $name = $\_.Name

&#x20;       $missing = @($missing | Where-Object { $\_ -ne $name })

&#x20;       $copied++

&#x20;   }

}



Write-Host "Done: copied $copied of $($paths.Count) files to $targetDir"

if ($missing.Count -gt 0) { Write-Host "Not found anywhere: $($missing -join ', ')" -ForegroundColor Yellow }

```





### Step 3: Extract the scene with AssetRipper



Download the free edition of [AssetRipper](https://assetripper.github.io/AssetRipper/), unzip it and run AssetRipper.GUI.Free.exe



Load your folder with the copied bundle files and Choose Export â†’ Export Unity Project into an empty folder





### Step 4: Open the export in Unity, make the prefab and build the bundle



In Unity Hub, choose Add â†’ Add project from disk, select the ExportedProject folder AssetRipper just created, open it with the exact Unity version as the current game version. (6000.0.60f1 for TLD 2.55). The first import takes ages, be patient lol.



Create a folder Assets/Editor in the Project window and place BundleTools.cs with this code in it. A Bundle Tools menu should appears in Unity's top menu bar.



```

// BundleTools.cs - put this file in Assets/Editor/ of the project AssetRipper exported.

// Adds a "Bundle Tools" menu to Unity's top menu bar.

\#if UNITY\_EDITOR

using System.IO;

using System.Text.RegularExpressions;

using UnityEditor;

using UnityEngine;



public static class BundleTools

{

&#x20;   const string DummyPrefix = "ModDummy/";           // the mod looks for this prefix

&#x20;   const string PrefabFolder = "Assets/ModBundle";   // every prefab in here goes into the bundle

&#x20;   const string BundleName = "mybundle";             // becomes mybundle.bundle

&#x20;   const string OutputFolder = "ModBundleBuild";     // created next to the Assets folder



&#x20;   // Renderers with these materials are editor-only helpers in TLD scenes

&#x20;   static readonly string\[] HelperMaterials = { "FX\_PlacePoint", "Default-Material" };



&#x20;   // Step 1: give AssetRipper's placeholder shaders a unique name, so the mod can tell

&#x20;   // them apart from the game's real shaders (which have the original names)

&#x20;   \[MenuItem("Bundle Tools/1  Rename dummy shaders")]

&#x20;   static void RenameDummyShaders()

&#x20;   {

&#x20;       foreach (string guid in AssetDatabase.FindAssets("t:Shader", new\[] { "Assets" }))

&#x20;       {

&#x20;           string path = AssetDatabase.GUIDToAssetPath(guid);

&#x20;           if (!path.EndsWith(".shader")) continue;



&#x20;           string text = File.ReadAllText(path);

&#x20;           Match m = Regex.Match(text, "Shader\\\\s+\\"(\[^\\"]+)\\"");

&#x20;           if (!m.Success || m.Groups\[1].Value.StartsWith(DummyPrefix)) continue;



&#x20;           // Shader "Shader Forge/TLD\_StandardDiffuse" -> Shader "ModDummy/Shader Forge/TLD\_StandardDiffuse"

&#x20;           File.WriteAllText(path, text.Substring(0, m.Groups\[1].Index) + DummyPrefix + text.Substring(m.Groups\[1].Index));

&#x20;       }

&#x20;       AssetDatabase.Refresh();

&#x20;       Debug.Log("\[Bundle Tools] Dummy shaders renamed");

&#x20;   }



&#x20;   // Step 2: select an object in the Hierarchy, then run this. It saves a cleaned-up copy as a prefab.

&#x20;   \[MenuItem("Bundle Tools/2  Make prefab from selected object")]

&#x20;   static void MakePrefab()

&#x20;   {

&#x20;       GameObject original = Selection.activeGameObject;

&#x20;       if (original == null)

&#x20;       {

&#x20;           EditorUtility.DisplayDialog("Bundle Tools", "Select the object in the Hierarchy first.", "OK");

&#x20;           return;

&#x20;       }



&#x20;       // A copy at the origin: where it goes in the game is decided by the mod

&#x20;       GameObject copy = Object.Instantiate(original);

&#x20;       copy.name = original.name;

&#x20;       copy.transform.SetPositionAndRotation(Vector3.zero, Quaternion.identity);

&#x20;       copy.transform.localScale = original.transform.lossyScale;



&#x20;       // Keep the looks and collision (meshes, materials, LODs, colliders); remove what only works in the game

&#x20;       foreach (Transform t in copy.GetComponentsInChildren<Transform>(true))

&#x20;       {

&#x20;           if (t == null) continue;                                   // removed together with its parent

&#x20;           GameObject go = t.gameObject;



&#x20;           if (go != copy \&\& go.GetComponent<ParticleSystem>() != null)

&#x20;           {

&#x20;               Object.DestroyImmediate(go);

&#x20;               continue;

&#x20;           }



&#x20;           GameObjectUtility.RemoveMonoBehavioursWithMissingScript(go);

&#x20;           foreach (MonoBehaviour mb in go.GetComponents<MonoBehaviour>())

&#x20;               if (mb != null) Object.DestroyImmediate(mb);

&#x20;           foreach (AudioSource a in go.GetComponents<AudioSource>())

&#x20;               if (a != null) Object.DestroyImmediate(a);



&#x20;           MeshRenderer mr = go.GetComponent<MeshRenderer>();

&#x20;           if (mr != null \&\& UsesHelperMaterial(mr))

&#x20;               Object.DestroyImmediate(mr);

&#x20;       }



&#x20;       if (!AssetDatabase.IsValidFolder(PrefabFolder)) AssetDatabase.CreateFolder("Assets", "ModBundle");

&#x20;       string path = $"{PrefabFolder}/{original.name}.prefab";

&#x20;       PrefabUtility.SaveAsPrefabAsset(copy, path);

&#x20;       Object.DestroyImmediate(copy);

&#x20;       Debug.Log("\[Bundle Tools] Saved " + path);

&#x20;   }



&#x20;   // Step 3: builds every prefab in Assets/ModBundle (plus everything they use) into one bundle

&#x20;   \[MenuItem("Bundle Tools/3  Build bundle")]

&#x20;   static void BuildBundle()

&#x20;   {

&#x20;       foreach (string name in AssetDatabase.GetAllAssetBundleNames())

&#x20;           AssetDatabase.RemoveAssetBundleName(name, true);

&#x20;       foreach (string guid in AssetDatabase.FindAssets("t:Prefab", new\[] { PrefabFolder }))

&#x20;           AssetImporter.GetAtPath(AssetDatabase.GUIDToAssetPath(guid)).assetBundleName = BundleName;



&#x20;       Directory.CreateDirectory(OutputFolder);

&#x20;       BuildPipeline.BuildAssetBundles(OutputFolder, BuildAssetBundleOptions.ChunkBasedCompression, BuildTarget.StandaloneWindows64);



&#x20;       string final = Path.Combine(OutputFolder, BundleName + ".bundle");

&#x20;       File.Copy(Path.Combine(OutputFolder, BundleName), final, true);

&#x20;       Debug.Log("\[Bundle Tools] Built " + Path.GetFullPath(final));

&#x20;       EditorUtility.RevealInFinder(final);

&#x20;   }



&#x20;   static bool UsesHelperMaterial(Renderer r)

&#x20;   {

&#x20;       foreach (Material m in r.sharedMaterials)

&#x20;           if (m != null \&\& System.Array.IndexOf(HelperMaterials, m.name) >= 0) return true;

&#x20;       return false;

&#x20;   }

}

\#endif

```



Time to create the bundle. Bu first load the scene the object is in. In the Project window, search t:Scene plus your scene name, and double-click the scene. 


1. Click on the bundle tools menu in the top bar and choose: 1 Rename dummy shaders. Run this once per project. It changes each placeholder shader's name from, for example, Shader Forge/TLD\_StandardDiffuse to ModDummy/Shader Forge/TLD\_StandardDiffuse. Without this, the mod's Shader.Find could find your placeholder instead of the game's real shader, since they have the same name.
2. Select your object in the Hierarchy, then run Bundle Tools â†’ 2 Make prefab from selected object. The prefab appears in Assets/ModBundle. Repeat for any other objects you want in the bundle.
3. Bundle Tools menu â†’ 3 Build bundle. File Explorer opens ModBundleBuild with mybundle.bundle. You can ignore the other files there.



Note: If your object is made of several parts (most likely), put the parts under one parent first, so they keep their places relative to each other:

1\. Create an empty object (GameObject â†’ Create Empty) and give it the main part's Position and Rotation.

2\. Drag each part onto it in the Hierarchy. Unity keeps their world positions when you do that.

3\. Select the empty object and run step 2 on it.

4\. If you want part of an object gone, delete it from the prefab: double-click the prefab in Assets/ModBundle, delete the part, save, and build again.





### Step 5: The mod that loads the bundle



Now we are back to a normal MelonLoader project. Here is a dummy mod that loads the bundle.



```

using System;

using System.Collections.Generic;

using System.IO;

using MelonLoader;

using MelonLoader.Utils;

using UnityEngine;

using UnityEngine.AddressableAssets;

using UnityEngine.AddressableAssets.ResourceLocators;

using UnityEngine.Rendering;

using UnityEngine.SceneManagement;



\[assembly: MelonInfo(typeof(ObjectImporter.ObjectImporterMod), "ObjectImporter", "1.0.0", "YourName")]

\[assembly: MelonGame("Hinterland", "TheLongDark")]



namespace ObjectImporter

{

&#x20;   public class ObjectImporterMod : MelonMod

&#x20;   {

&#x20;       // Settings

&#x20;       const string TargetScene = "TARGET\_SCENE";                 // Your target scene

&#x20;       const string BundleFile = "ObjectImporter/mybundle.bundle";  // inside the Mods folder

&#x20;       const string DummyPrefix = "ModDummy/";                      // must match BundleTools.cs



&#x20;       // Prefab name in the bundle, then world position, rotation and scale (from UnityExplorer)

&#x20;       static readonly (string prefab, Vector3 position, Vector3 rotation, Vector3 scale)\[] Placements =

&#x20;       {

&#x20;           ("PREFAB\_NAME", new Vector3(0f, 0f, 0f), new Vector3(0f, 0f, 0f), new Vector3(1f, 1f, 1f)),

&#x20;       };



&#x20;       // Placing the objects



&#x20;       // Once the scene has been set up - not in OnSceneWasLoaded, which can be too early on the game's first load from the main menu

&#x20;       public override void OnSceneWasInitialized(int buildIndex, string sceneName)

&#x20;       {

&#x20;           if (sceneName != TargetScene) return;

&#x20;           Scene scene = SceneManager.GetSceneByName(sceneName);



&#x20;           foreach (var (prefabName, position, rotation, scale) in Placements)

&#x20;           {

&#x20;               GameObject go = GameObject.Instantiate<GameObject>(LoadPrefab(prefabName), position, Quaternion.Euler(rotation));

&#x20;               SceneManager.MoveGameObjectToScene(go, scene);          // unloaded together with the scene

&#x20;               go.transform.localScale = scale;

&#x20;               FixMaterials(go);

&#x20;           }

&#x20;       }



&#x20;       // Loading from the bundle



&#x20;       static AssetBundle \_bundle;

&#x20;       static Il2CppSystem.IO.MemoryStream \_bundleStream;              // must stay alive while the bundle is loaded



&#x20;       static GameObject LoadPrefab(string name)

&#x20;       {

&#x20;           // Once per game session: Unity refuses to load the same bundle twice

&#x20;           if (\_bundle == null)

&#x20;           {

&#x20;               string path = Path.Combine(MelonEnvironment.ModsDirectory, BundleFile);

&#x20;               \_bundleStream = new Il2CppSystem.IO.MemoryStream(File.ReadAllBytes(path));

&#x20;               \_bundle = AssetBundle.LoadFromStream(\_bundleStream);

&#x20;           }

&#x20;           // Fetched every time: the game clears unused assets from memory between loads

&#x20;           return \_bundle.LoadAsset<GameObject>(name);

&#x20;       }



&#x20;       // Materials: the game's own where possible, otherwise the game's real shader



&#x20;       static readonly Dictionary<string, Material> \_gameMaterials = new Dictionary<string, Material>();



&#x20;       static void FixMaterials(GameObject go)

&#x20;       {

&#x20;           Shader standard = Shader.Find("Shader Forge/TLD\_StandardDiffuse");



&#x20;           foreach (Renderer r in go.GetComponentsInChildren<Renderer>(true))

&#x20;           {

&#x20;               Material\[] mats = r.sharedMaterials;

&#x20;               for (int i = 0; i < mats.Length; i++)

&#x20;               {

&#x20;                   Material m = mats\[i];

&#x20;                   if (m == null) continue;

&#x20;                   string name = m.name.Replace(" (Instance)", "");



&#x20;                   // 1. The game has a material with this name: use it (exactly right, and already loaded)

&#x20;                   Material game = GetGameMaterial(name);

&#x20;                   if (game != null)

&#x20;                   {

&#x20;                       mats\[i] = game;

&#x20;                       continue;

&#x20;                   }



&#x20;                   // 2. Otherwise keep our copy, but give it the game's real shader instead of the dummy

&#x20;                   string shaderName = m.shader.name;

&#x20;                   if (!shaderName.StartsWith(DummyPrefix)) continue;       // already fixed

&#x20;                   Shader real = Shader.Find(shaderName.Substring(DummyPrefix.Length));

&#x20;                   m.shader = real != null ? real : standard;

&#x20;               }

&#x20;               r.sharedMaterials = mats;

&#x20;               r.lightProbeUsage = LightProbeUsage.BlendProbes;             // bundles carry no baked lighting

&#x20;           }

&#x20;       }



&#x20;       static Material GetGameMaterial(string name)

&#x20;       {

&#x20;           if (!\_gameMaterials.TryGetValue(name, out Material material))

&#x20;           {

&#x20;               string key = ResolveKey(name + ".mat");

&#x20;               material = key == null ? null : Addressables.LoadAssetAsync<Material>(key).WaitForCompletion();

&#x20;               \_gameMaterials\[name] = material;

&#x20;           }

&#x20;           return material;

&#x20;       }



&#x20;       // The game's Addressables keys: its assets, addressed by their original file path



&#x20;       static List<string> \_allKeys;



&#x20;       // "GLB\_WallWoodNatural\_N02.mat" -> "Assets/ArtAssets/Materials/Global/GLB\_WallWoodNatural\_N02.mat"

&#x20;       static string ResolveKey(string file)

&#x20;       {

&#x20;           \_allKeys ??= ReadAllKeys();

&#x20;           foreach (string key in \_allKeys)

&#x20;               if (key.EndsWith("/" + file, StringComparison.OrdinalIgnoreCase)) return key;

&#x20;           return null;

&#x20;       }



&#x20;       static List<string> ReadAllKeys()

&#x20;       {

&#x20;           var all = new List<string>();

&#x20;           var locators = Il2CppSystem.Linq.Enumerable.ToList(Addressables.ResourceLocators);

&#x20;           for (int i = 0; i < locators.Count; i++)

&#x20;           {

&#x20;               IResourceLocator locator = locators\[i];

&#x20;               if (locator == null || locator.Keys == null) continue;

&#x20;               var keys = Il2CppSystem.Linq.Enumerable.ToList(locator.Keys);

&#x20;               for (int k = 0; k < keys.Count; k++)

&#x20;                   if (keys\[k] != null) all.Add(keys\[k].ToString());

&#x20;           }

&#x20;           return all;

&#x20;       }

&#x20;   }

}

```



### Step 6: Place it and fine-tune in game



From here on out just place the objects where you want. I recommend looking at [MooseMeat's mods](https://github.com/moosemeat817) on how to do that. (that's how I did it)





### Credits

**This wouldn't have been possible without other modders published work:**

* DigitalzombieTLD (UnityExplorer for TLD and his many guides)
* Waltz (loading a bundle with LoadFromStream, and reading the game's Addressables keys.
* Fuar11 (loading the game's own prefabs by their Addressables path.)
* moosemeat817 (moving and copying scene objects, picking objects by child index)
* Everyone at the AssetRipper Project





&#x20;






