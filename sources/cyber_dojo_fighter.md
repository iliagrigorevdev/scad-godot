### cyber_dojo_fighter/addons/scad_importer/plugin.cfg
```text
[plugin]

name="OpenSCAD GLTF Importer"
description="Imports .scad files directly as 3D scenes using scad-gltf"
author="scad-gltf"
version="0.2"
script="scad_plugin.gd"

```

### cyber_dojo_fighter/addons/scad_importer/scad_importer.gd
```gdscript
@tool
extends EditorSceneFormatImporter

const SCAD_CACHE_DIR := "res://__scad_glbs/"

func _get_extensions():
	return PackedStringArray(["scad"])

func _get_import_flags():
	return EditorSceneFormatImporter.IMPORT_SCENE

func _get_cached_glb_path(path: String) -> String:
	var rel := path
	if rel.begins_with("res://"):
		rel = rel.substr(6)
	rel = rel.trim_prefix("/")
	return SCAD_CACHE_DIR + rel + ".glb"

func _is_scad_gltf_available() -> bool:
	var check_output = []
	if OS.get_name() == "Windows":
		return OS.execute("where", PackedStringArray(["scad-gltf"]), check_output, true) == 0
	return OS.execute("sh", PackedStringArray(["-c", "command -v scad-gltf"]), check_output, true) == 0

func _import_scene(path: String, flags: int, options: Dictionary) -> Object:
	var cached_glb_path := _get_cached_glb_path(path)
	var temp_glb_path := ""
	var glb_to_load := ""
	var should_remove_temp := false

	if _is_scad_gltf_available():
		var global_source = ProjectSettings.globalize_path(path)
		var unique_id = str(hash(path))
		temp_glb_path = ProjectSettings.globalize_path("user://scad_cache_" + unique_id + ".glb")

		var args = PackedStringArray()
		args.append("convert")
		args.append(global_source)
		args.append(temp_glb_path)

		var output = []
		print("Importing %s via scad-gltf..." % path.get_file())

		var exit_code = -1
		if OS.get_name() == "Windows":
			var win_args = PackedStringArray(["/c", "scad-gltf"])
			win_args.append_array(args)
			exit_code = OS.execute("cmd.exe", win_args, output, true)
		else:
			exit_code = OS.execute("scad-gltf", args, output, true)

		if exit_code != 0 or not FileAccess.file_exists(temp_glb_path):
			push_error("Failed to compile SCAD file: %s. Ensure scad-gltf is installed." % path.get_file())
			push_error("scad-gltf convert output: ", "\n".join(output))
			return null

		glb_to_load = temp_glb_path
		should_remove_temp = true
	elif FileAccess.file_exists(cached_glb_path):
		print("scad-gltf not found; using cached GLB fallback for %s (%s)." % [path.get_file(), cached_glb_path])
		glb_to_load = ProjectSettings.globalize_path(cached_glb_path)
	else:
		push_error("Failed to import SCAD file: %s. scad-gltf is not installed and no cached GLB was found at %s." % [path.get_file(), cached_glb_path])
		return null

	var gltf_doc = GLTFDocument.new()
	var gltf_state = GLTFState.new()
	var err = gltf_doc.append_from_file(glb_to_load, gltf_state)

	if should_remove_temp and FileAccess.file_exists(temp_glb_path):
		DirAccess.remove_absolute(temp_glb_path)

	if err != OK:
		push_error("Failed to parse the generated GLB for %s." % path.get_file())
		return null

	var generated_scene = gltf_doc.generate_scene(gltf_state)
	if generated_scene:
		generated_scene.name = path.get_file().get_basename()

	return generated_scene

```

### cyber_dojo_fighter/addons/scad_importer/scad_plugin.gd
```gdscript
@tool
extends EditorPlugin

var import_plugin

func _enter_tree():
	import_plugin = preload("res://addons/scad_importer/scad_importer.gd").new()
	add_scene_format_importer_plugin(import_plugin)

func _exit_tree():
	remove_scene_format_importer_plugin(import_plugin)
	import_plugin = null

```

### cyber_dojo_fighter/assets/models/fighter_striker.scad
```openscad
// Cyber Striker Fighter (~1.78m tall)
// +Z is UP, +Y is Forward, +X is Right, -X is Left

$fn = 24;
$asa = 35.0;

// Shared martial arts animations
striker_animations = [
  ["idle", [
    ["Pelvis", [
      [0.0, [0, 0, -12], [0, 0, 0.90]],
      [0.45, [0, 0, -12], [0, 0, 0.87]],
      [0.9, [0, 0, -12], [0, 0, 0.90]]
    ]],
    ["Torso", [
      [0.0, [-6, 0, 6]],
      [0.45, [-9, 0, 6]],
      [0.9, [-6, 0, 6]]
    ]],
    ["Head", [
      [0.0, [4, 0, 6]],
      [0.45, [6, 0, 6]],
      [0.9, [4, 0, 6]]
    ]],
    ["LeftUpperArm", [
      [0.0, [42, 18, -10]],
      [0.45, [46, 18, -10]],
      [0.9, [42, 18, -10]]
    ]],
    ["LeftForearm", [
      [0.0, [78, 0, 0]],
      [0.45, [82, 0, 0]],
      [0.9, [78, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [28, -16, 10]],
      [0.45, [32, -16, 10]],
      [0.9, [28, -16, 10]]
    ]],
    ["RightForearm", [
      [0.0, [85, 0, 0]],
      [0.45, [90, 0, 0]],
      [0.9, [85, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [22, 8, 0]],
      [0.45, [26, 8, 0]],
      [0.9, [22, 8, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-26, 0, 0]],
      [0.45, [-32, 0, 0]],
      [0.9, [-26, 0, 0]]
    ]],
    ["RightThigh", [
      [0.0, [-18, -8, 0]],
      [0.45, [-22, -8, 0]],
      [0.9, [-18, -8, 0]]
    ]],
    ["RightShin", [
      [0.0, [-18, 0, 0]],
      [0.45, [-22, 0, 0]],
      [0.9, [-18, 0, 0]]
    ]]
  ]],

  ["walk_forward", [
    ["Pelvis", [
      [0.0, [0, 0, -6], [0, 0, 0.90]],
      [0.2, [0, 0, -6], [0, 0, 0.88]],
      [0.4, [0, 0, -6], [0, 0, 0.90]],
      [0.6, [0, 0, -6], [0, 0, 0.88]],
      [0.8, [0, 0, -6], [0, 0, 0.90]]
    ]],
    ["Torso", [
      [0.0, [-8, 0, 8]],
      [0.2, [-8, 0, 0]],
      [0.4, [-8, 0, -8]],
      [0.6, [-8, 0, 0]],
      [0.8, [-8, 0, 8]]
    ]],
    ["LeftUpperArm", [
      [0.0, [38, 12, 0]],
      [0.4, [48, 12, 0]],
      [0.8, [38, 12, 0]]
    ]],
    ["LeftForearm", [
      [0.0, [82, 0, 0]],
      [0.8, [82, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [44, -12, 0]],
      [0.4, [32, -12, 0]],
      [0.8, [44, -12, 0]]
    ]],
    ["RightForearm", [
      [0.0, [86, 0, 0]],
      [0.8, [86, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [32, 4, 0]],
      [0.2, [5, 4, 0]],
      [0.4, [-24, 4, 0]],
      [0.6, [12, 4, 0]],
      [0.8, [32, 4, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-14, 0, 0]],
      [0.2, [-18, 0, 0]],
      [0.4, [-28, 0, 0]],
      [0.6, [-48, 0, 0]],
      [0.8, [-14, 0, 0]]
    ]],
    ["RightThigh", [
      [0.0, [-24, -4, 0]],
      [0.2, [12, -4, 0]],
      [0.4, [32, -4, 0]],
      [0.6, [5, -4, 0]],
      [0.8, [-24, -4, 0]]
    ]],
    ["RightShin", [
      [0.0, [-28, 0, 0]],
      [0.2, [-48, 0, 0]],
      [0.4, [-14, 0, 0]],
      [0.6, [-18, 0, 0]],
      [0.8, [-28, 0, 0]]
    ]]
  ]],

  ["walk_backward", [
    ["Pelvis", [
      [0.0, [0, 0, -10], [0, 0, 0.89]],
      [0.2, [0, 0, -10], [0, 0, 0.87]],
      [0.4, [0, 0, -10], [0, 0, 0.89]],
      [0.6, [0, 0, -10], [0, 0, 0.87]],
      [0.8, [0, 0, -10], [0, 0, 0.89]]
    ]],
    ["Torso", [
      [0.0, [2, 0, 8]],
      [0.4, [2, 0, 14]],
      [0.8, [2, 0, 8]]
    ]],
    ["LeftUpperArm", [
      [0.0, [54, 10, -8]],
      [0.8, [54, 10, -8]]
    ]],
    ["LeftForearm", [
      [0.0, [88, 0, 0]],
      [0.8, [88, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [48, -10, 8]],
      [0.8, [48, -10, 8]]
    ]],
    ["RightForearm", [
      [0.0, [92, 0, 0]],
      [0.8, [92, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [-18, 4, 0]],
      [0.4, [26, 4, 0]],
      [0.8, [-18, 4, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-24, 0, 0]],
      [0.4, [-16, 0, 0]],
      [0.8, [-24, 0, 0]]
    ]],
    ["RightThigh", [
      [0.0, [26, -4, 0]],
      [0.4, [-18, -4, 0]],
      [0.8, [26, -4, 0]]
    ]],
    ["RightShin", [
      [0.0, [-16, 0, 0]],
      [0.4, [-24, 0, 0]],
      [0.8, [-16, 0, 0]]
    ]]
  ]],

  ["crouch", [
    ["Pelvis", [
      [0.0, [0, 0, -24], [0, 0, 0.68]],
      [0.4, [0, 0, -24], [0, 0, 0.67]]
    ]],
    ["Torso", [
      [0.0, [-28, 0, 20]],
      [0.4, [-28, 0, 20]]
    ]],
    ["Head", [
      [0.0, [22, 0, 6]],
      [0.4, [22, 0, 6]]
    ]],
    ["LeftUpperArm", [
      [0.0, [54, 16, -12]],
      [0.4, [54, 16, -12]]
    ]],
    ["LeftForearm", [
      [0.0, [96, 0, 0]],
      [0.4, [96, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [46, -16, 14]],
      [0.4, [46, -16, 14]]
    ]],
    ["RightForearm", [
      [0.0, [98, 0, 0]],
      [0.4, [98, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [66, 12, 0]],
      [0.4, [66, 12, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-64, 0, 0]],
      [0.4, [-64, 0, 0]]
    ]],
    ["RightThigh", [
      [0.0, [-46, -14, 0]],
      [0.4, [-46, -14, 0]]
    ]],
    ["RightShin", [
      [0.0, [-18, 0, 0]],
      [0.4, [-18, 0, 0]]
    ]]
  ]],

  ["jump", [
    ["Pelvis", [
      [0.0, [0, 0, 0], [0, 0, 0.96]],
      [0.4, [0, 0, 0], [0, 0, 0.96]]
    ]],
    ["Torso", [
      [0.0, [-10, 0, 0]],
      [0.4, [-10, 0, 0]]
    ]],
    ["LeftUpperArm", [
      [0.0, [55, 25, 0]],
      [0.4, [55, 25, 0]]
    ]],
    ["LeftForearm", [
      [0.0, [75, 0, 0]],
      [0.4, [75, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [30, -25, 0]],
      [0.4, [30, -25, 0]]
    ]],
    ["RightForearm", [
      [0.0, [85, 0, 0]],
      [0.4, [85, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [58, 6, 0]],
      [0.4, [58, 6, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-82, 0, 0]],
      [0.4, [-82, 0, 0]]
    ]],
    ["RightThigh", [
      [0.0, [25, -6, 0]],
      [0.4, [25, -6, 0]]
    ]],
    ["RightShin", [
      [0.0, [-55, 0, 0]],
      [0.4, [-55, 0, 0]]
    ]]
  ]],

  ["punch_light", [
    ["Pelvis", [
      [0.0, [0, 0, -10], [0, 0, 0.90]],
      [0.08, [0, 0, 22], [0, 0.08, 0.89]],
      [0.24, [0, 0, -10], [0, 0, 0.90]]
    ]],
    ["Torso", [
      [0.0, [-6, 0, -10]],
      [0.08, [-14, 0, 28]],
      [0.24, [-6, 0, 6]]
    ]],
    ["Head", [
      [0.0, [4, 0, 6]],
      [0.08, [10, 0, -24]],
      [0.24, [4, 0, 6]]
    ]],
    ["LeftUpperArm", [
      [0.0, [42, 18, -10]],
      [0.08, [25, 22, -25]],
      [0.24, [42, 18, -10]]
    ]],
    ["LeftForearm", [
      [0.0, [78, 0, 0]],
      [0.08, [95, 0, 0]],
      [0.24, [78, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [30, -14, 0]],
      [0.08, [88, 0, -12]],
      [0.16, [88, 0, -12]],
      [0.24, [28, -16, 10]]
    ]],
    ["RightForearm", [
      [0.0, [85, 0, 0]],
      [0.08, [8, 0, 0]],
      [0.16, [8, 0, 0]],
      [0.24, [85, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [22, 8, 0]],
      [0.08, [32, 8, 0]],
      [0.24, [22, 8, 0]]
    ]],
    ["RightThigh", [
      [0.0, [-18, -8, 0]],
      [0.08, [-28, -8, 0]],
      [0.24, [-18, -8, 0]]
    ]]
  ]],

  ["punch_light_left", [
    ["Pelvis", [
      [0.0, [0, 0, -10], [0, 0, 0.90]],
      [0.08, [0, 0, -26], [0, 0.08, 0.89]],
      [0.24, [0, 0, -10], [0, 0, 0.90]]
    ]],
    ["Torso", [
      [0.0, [-6, 0, 6]],
      [0.08, [-14, 0, -24]],
      [0.24, [-6, 0, 6]]
    ]],
    ["Head", [
      [0.0, [4, 0, 6]],
      [0.08, [10, 0, 26]],
      [0.24, [4, 0, 6]]
    ]],
    ["LeftUpperArm", [
      [0.0, [42, 18, -10]],
      [0.08, [88, 0, 14]],
      [0.16, [88, 0, 14]],
      [0.24, [42, 18, -10]]
    ]],
    ["LeftForearm", [
      [0.0, [78, 0, 0]],
      [0.08, [8, 0, 0]],
      [0.16, [8, 0, 0]],
      [0.24, [78, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [28, -16, 10]],
      [0.08, [34, -20, 22]],
      [0.24, [28, -16, 10]]
    ]],
    ["RightForearm", [
      [0.0, [85, 0, 0]],
      [0.08, [95, 0, 0]],
      [0.24, [85, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [22, 8, 0]],
      [0.08, [28, 8, 0]],
      [0.24, [22, 8, 0]]
    ]],
    ["RightThigh", [
      [0.0, [-18, -8, 0]],
      [0.08, [-26, -8, 0]],
      [0.24, [-18, -8, 0]]
    ]]
  ]],

  ["kick_heavy", [
    ["Pelvis", [
      [0.0, [0, 0, -10], [0, 0, 0.90]],
      [0.16, [12, -16, 35], [0, 0.12, 0.94]],
      [0.32, [12, -16, 35], [0, 0.12, 0.94]],
      [0.48, [0, 0, -10], [0, 0, 0.90]]
    ]],
    ["Torso", [
      [0.0, [-6, 0, 6]],
      [0.16, [18, 16, -24]],
      [0.32, [18, 16, -24]],
      [0.48, [-6, 0, 6]]
    ]],
    ["LeftUpperArm", [
      [0.0, [42, 18, -10]],
      [0.16, [60, 35, 0]],
      [0.48, [42, 18, -10]]
    ]],
    ["RightUpperArm", [
      [0.0, [28, -16, 10]],
      [0.16, [-30, -35, 0]],
      [0.48, [28, -16, 10]]
    ]],
    ["RightThigh", [
      [0.0, [-18, -8, 0]],
      [0.10, [65, -8, 0]],
      [0.18, [92, -6, 12]],
      [0.32, [92, -6, 12]],
      [0.48, [-18, -8, 0]]
    ]],
    ["RightShin", [
      [0.0, [-18, 0, 0]],
      [0.10, [-85, 0, 0]],
      [0.18, [-6, 0, 0]],
      [0.32, [-6, 0, 0]],
      [0.48, [-18, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [22, 8, 0]],
      [0.18, [-8, 10, 0]],
      [0.48, [22, 8, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-26, 0, 0]],
      [0.18, [-12, 0, 0]],
      [0.48, [-26, 0, 0]]
    ]]
  ]],

  ["kick_heavy_left", [
    ["Pelvis", [
      [0.0, [0, 0, -10], [0, 0, 0.90]],
      [0.16, [12, 16, -35], [0, 0.12, 0.94]],
      [0.32, [12, 16, -35], [0, 0.12, 0.94]],
      [0.48, [0, 0, -10], [0, 0, 0.90]]
    ]],
    ["Torso", [
      [0.0, [-6, 0, 6]],
      [0.16, [18, -16, 24]],
      [0.32, [18, -16, 24]],
      [0.48, [-6, 0, 6]]
    ]],
    ["LeftUpperArm", [
      [0.0, [42, 18, -10]],
      [0.16, [-30, 35, 0]],
      [0.48, [42, 18, -10]]
    ]],
    ["RightUpperArm", [
      [0.0, [28, -16, 10]],
      [0.16, [60, -35, 0]],
      [0.48, [28, -16, 10]]
    ]],
    ["LeftThigh", [
      [0.0, [22, 8, 0]],
      [0.10, [65, 8, 0]],
      [0.18, [92, 6, -12]],
      [0.32, [92, 6, -12]],
      [0.48, [22, 8, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-26, 0, 0]],
      [0.10, [-85, 0, 0]],
      [0.18, [-6, 0, 0]],
      [0.32, [-6, 0, 0]],
      [0.48, [-26, 0, 0]]
    ]],
    ["RightThigh", [
      [0.0, [-18, -8, 0]],
      [0.18, [-8, -10, 0]],
      [0.48, [-18, -8, 0]]
    ]],
    ["RightShin", [
      [0.0, [-18, 0, 0]],
      [0.18, [-12, 0, 0]],
      [0.48, [-18, 0, 0]]
    ]]
  ]],

  ["special_strike", [
    ["Pelvis", [
      [0.0, [0, 0, 0], [0, -0.05, 0.88]],
      [0.15, [0, 0, 0], [0, -0.10, 0.84]],
      [0.28, [0, 0, 0], [0, 0.16, 0.88]],
      [0.52, [0, 0, -12], [0, 0, 0.90]]
    ]],
    ["Torso", [
      [0.0, [8, 0, 0]],
      [0.15, [14, 0, 0]],
      [0.28, [-22, 0, 0]],
      [0.52, [-6, 0, 6]]
    ]],
    ["LeftUpperArm", [
      [0.0, [20, 14, 0]],
      [0.15, [-18, 12, 0]],
      [0.28, [86, 6, -12]],
      [0.40, [86, 6, -12]],
      [0.52, [42, 18, -10]]
    ]],
    ["LeftForearm", [
      [0.0, [80, 0, 0]],
      [0.15, [95, 0, 0]],
      [0.28, [6, 0, 0]],
      [0.40, [6, 0, 0]],
      [0.52, [78, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [20, -14, 0]],
      [0.15, [-18, -12, 0]],
      [0.28, [86, -6, 12]],
      [0.40, [86, -6, 12]],
      [0.52, [28, -16, 10]]
    ]],
    ["RightForearm", [
      [0.0, [80, 0, 0]],
      [0.15, [95, 0, 0]],
      [0.28, [6, 0, 0]],
      [0.40, [6, 0, 0]],
      [0.52, [85, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [28, 10, 0]],
      [0.28, [42, 10, 0]],
      [0.52, [22, 8, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-36, 0, 0]],
      [0.28, [-48, 0, 0]],
      [0.52, [-26, 0, 0]]
    ]],
    ["RightThigh", [
      [0.0, [-24, -10, 0]],
      [0.28, [-36, -10, 0]],
      [0.52, [-18, -8, 0]]
    ]]
  ]],

  ["block", [
    ["Pelvis", [
      [0.0, [0, 0, -6], [0, 0, 0.84]],
      [0.3, [0, 0, -6], [0, 0, 0.84]]
    ]],
    ["Torso", [
      [0.0, [-12, 0, 12]],
      [0.3, [-12, 0, 12]]
    ]],
    ["Head", [
      [0.0, [-8, 0, -6]],
      [0.3, [-8, 0, -6]]
    ]],
    ["LeftUpperArm", [
      [0.0, [68, 8, -28]],
      [0.3, [68, 8, -28]]
    ]],
    ["LeftForearm", [
      [0.0, [95, 0, 0]],
      [0.3, [95, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [74, -8, 28]],
      [0.3, [74, -8, 28]]
    ]],
    ["RightForearm", [
      [0.0, [98, 0, 0]],
      [0.3, [98, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [32, 10, 0]],
      [0.3, [32, 10, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-42, 0, 0]],
      [0.3, [-42, 0, 0]]
    ]],
    ["RightThigh", [
      [0.0, [-24, -10, 0]],
      [0.3, [-24, -10, 0]]
    ]],
    ["RightShin", [
      [0.0, [-32, 0, 0]],
      [0.3, [-32, 0, 0]]
    ]]
  ]],

  ["hit_stun", [
    ["Pelvis", [
      [0.0, [14, 0, 0], [0, -0.08, 0.88]],
      [0.25, [6, 0, 0], [0, -0.02, 0.90]]
    ]],
    ["Torso", [
      [0.0, [26, 0, -10]],
      [0.25, [10, 0, 0]]
    ]],
    ["Head", [
      [0.0, [28, 0, 12]],
      [0.25, [8, 0, 0]]
    ]],
    ["LeftUpperArm", [
      [0.0, [15, 38, 0]],
      [0.25, [35, 20, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [15, -38, 0]],
      [0.25, [30, -20, 0]]
    ]]
  ]],

  ["knockdown", [
    ["Pelvis", [
      [0.0, [18, 0, 0], [0, -0.1, 0.84]],
      [0.22, [58, 0, 0], [0, -0.35, 0.45]],
      [0.45, [88, 0, 0], [0, -0.60, 0.16]],
      [0.85, [88, 0, 0], [0, -0.60, 0.16]]
    ]],
    ["Torso", [
      [0.0, [14, 0, 0]],
      [0.45, [4, 0, 0]],
      [0.85, [4, 0, 0]]
    ]],
    ["LeftUpperArm", [
      [0.0, [20, 35, 0]],
      [0.45, [10, 55, 0]],
      [0.85, [10, 55, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [20, -35, 0]],
      [0.45, [10, -55, 0]],
      [0.85, [10, -55, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [15, 10, 0]],
      [0.45, [6, 18, 0]],
      [0.85, [6, 18, 0]]
    ]],
    ["RightThigh", [
      [0.0, [10, -10, 0]],
      [0.45, [4, -18, 0]],
      [0.85, [4, -18, 0]]
    ]]
  ]],

  ["victory", [
    ["Pelvis", [
      [0.0, [0, 0, 0], [0, 0, 0.90]],
      [0.4, [0, 0, 0], [0, 0, 0.92]],
      [0.8, [0, 0, 0], [0, 0, 0.90]]
    ]],
    ["Torso", [
      [0.0, [4, 0, 0]],
      [0.4, [6, 0, 0]],
      [0.8, [4, 0, 0]]
    ]],
    ["LeftUpperArm", [
      [0.0, [35, 22, 0]],
      [0.4, [15, 18, 0]],
      [0.8, [15, 18, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [85, -18, 0]],
      [0.4, [158, -14, 0]],
      [0.8, [158, -14, 0]]
    ]],
    ["RightForearm", [
      [0.0, [65, 0, 0]],
      [0.4, [12, 0, 0]],
      [0.8, [12, 0, 0]]
    ]]
  ]]
];

// Material helpers
module suit_dark() {
  color([0.12, 0.14, 0.18], roughness=0.55, metalness=0.25) children();
}

module armor_blue() {
  color([0.14, 0.42, 0.88], roughness=0.28, metalness=0.65) children();
}

module armor_silver() {
  color([0.78, 0.82, 0.88], roughness=0.22, metalness=0.88) children();
}

module neon_cyan() {
  color([0.1, 0.9, 1.0], roughness=0.2, metalness=0.1, emissive=[0.1, 0.85, 1.0], emissiveIntensity=2.4) children();
}

armature(animations=striker_animations) {
  // Soft-skinned waist flex connector between Pelvis (z=0.92) and Torso (z=1.04)
  skin(bones=["Pelvis", "Torso"], weights=function(x,y,z) [1.0 - min(1.0, max(0.0, (z - 0.92) / 0.14)), min(1.0, max(0.0, (z - 0.92) / 0.14))]) {
    suit_dark()
      translate([0, 0, 0.91])
        scale([1.15, 0.85, 1.0])
          cylinder(h=0.16, r=0.115, $fn=32);
  }

  bone(name="Pelvis", t=[0, 0, 0.92], r=[0, 0, 0]) {
    // Belt & armored waist
    armor_blue()
      scale([1.22, 0.9, 1.0])
        cylinder(h=0.12, r=0.135, center=true);
    neon_cyan()
      translate([0, 0.12, 0.01])
        cube([0.08, 0.03, 0.05], center=true);

    // Torso
    bone(name="Torso", t=[0, 0, 0.12], r=[0, 0, 0]) {
      // Chest under-suit
      suit_dark()
        translate([0, 0, 0.20])
          scale([1.22, 0.82, 1.0])
            cylinder(h=0.34, r1=0.13, r2=0.175, center=true);
      // Raised chest breastplate
      armor_blue()
        translate([0, 0.035, 0.24])
          cube([0.32, 0.24, 0.21], center=true);
      // Glowing chest core
      neon_cyan()
        translate([0, 0.155, 0.25])
          rotate([90, 0, 0])
            cylinder(h=0.03, r=0.048, center=true);
      // Shoulder blades / collar
      armor_silver()
        translate([0, -0.02, 0.36])
          cube([0.36, 0.14, 0.06], center=true);

      // Head
      bone(name="Head", t=[0, 0, 0.46], r=[0, 0, 0]) {
        // Neck
        suit_dark()
          translate([0, 0, -0.04])
            cylinder(h=0.10, r=0.055, center=true);
        // Helmet / Head
        armor_silver()
          translate([0, 0, 0.10])
            scale([0.92, 1.05, 1.15])
              sphere(r=0.115);
        // Visor
        neon_cyan()
          translate([0, 0.095, 0.11])
            scale([1.0, 0.65, 0.45])
              sphere(r=0.085);
        // Headband crest & flowing tails
        armor_blue()
          translate([0, 0, 0.17])
            cylinder(h=0.04, r=0.116, center=true);
        neon_cyan()
          translate([-0.03, -0.14, 0.14])
            rotate([-28, 12, 0])
              cube([0.035, 0.18, 0.015], center=true);
        neon_cyan()
          translate([0.03, -0.14, 0.12])
            rotate([-38, -12, 0])
              cube([0.035, 0.16, 0.015], center=true);
      }

      // Left Arm (-X)
      bone(name="LeftUpperArm", t=[-0.24, 0, 0.34], r=[0, 0, 0]) {
        armor_blue()
          sphere(r=0.075);
        suit_dark()
          translate([0, 0, -0.14])
            cylinder(h=0.26, r1=0.05, r2=0.062, center=true);

        bone(name="LeftForearm", t=[0, 0, -0.28], r=[0, 0, 0]) {
          armor_silver()
            sphere(r=0.052);
          armor_blue()
            translate([0, 0, -0.12])
              cylinder(h=0.22, r1=0.046, r2=0.056, center=true);
          neon_cyan()
            translate([-0.045, 0.02, -0.13])
              cube([0.025, 0.04, 0.14], center=true);
          // Fist
          armor_silver()
            translate([0, 0.01, -0.25])
              sphere(r=0.058);
        }
      }

      // Right Arm (+X)
      bone(name="RightUpperArm", t=[0.24, 0, 0.34], r=[0, 0, 0]) {
        armor_blue()
          sphere(r=0.075);
        suit_dark()
          translate([0, 0, -0.14])
            cylinder(h=0.26, r1=0.05, r2=0.062, center=true);

        bone(name="RightForearm", t=[0, 0, -0.28], r=[0, 0, 0]) {
          armor_silver()
            sphere(r=0.052);
          armor_blue()
            translate([0, 0, -0.12])
              cylinder(h=0.22, r1=0.046, r2=0.056, center=true);
          neon_cyan()
            translate([0.045, 0.02, -0.13])
              cube([0.025, 0.04, 0.14], center=true);
          // Fist
          armor_silver()
            translate([0, 0.01, -0.25])
              sphere(r=0.058);
        }
      }
    }

    // Left Leg (-X)
    bone(name="LeftThigh", t=[-0.12, 0, -0.05], r=[0, 0, 0]) {
      suit_dark()
        translate([0, 0, -0.20])
          cylinder(h=0.40, r1=0.068, r2=0.088, center=true);
      armor_blue()
        translate([-0.045, 0.035, -0.18])
          cube([0.09, 0.13, 0.24], center=true);

      bone(name="LeftShin", t=[0, 0, -0.42], r=[0, 0, 0]) {
        armor_silver()
          translate([0, 0.02, 0])
            sphere(r=0.062);
        armor_blue()
          translate([0, 0, -0.20])
            cylinder(h=0.38, r1=0.052, r2=0.068, center=true);
        neon_cyan()
          translate([-0.045, 0.03, -0.20])
            cube([0.025, 0.035, 0.24], center=true);
        // Boot
        armor_silver()
          translate([0, 0.05, -0.41])
            cube([0.11, 0.22, 0.08], center=true);
      }
    }

    // Right Leg (+X)
    bone(name="RightThigh", t=[0.12, 0, -0.05], r=[0, 0, 0]) {
      suit_dark()
        translate([0, 0, -0.20])
          cylinder(h=0.40, r1=0.068, r2=0.088, center=true);
      armor_blue()
        translate([0.045, 0.035, -0.18])
          cube([0.09, 0.13, 0.24], center=true);

      bone(name="RightShin", t=[0, 0, -0.42], r=[0, 0, 0]) {
        armor_silver()
          translate([0, 0.02, 0])
            sphere(r=0.062);
        armor_blue()
          translate([0, 0, -0.20])
            cylinder(h=0.38, r1=0.052, r2=0.068, center=true);
        neon_cyan()
          translate([0.045, 0.03, -0.20])
            cube([0.025, 0.035, 0.24], center=true);
        // Boot
        armor_silver()
          translate([0, 0.05, -0.41])
            cube([0.11, 0.22, 0.08], center=true);
      }
    }
  }
}
```

### cyber_dojo_fighter/assets/models/fighter_titan.scad
```openscad
// Crimson Titan Heavy Brawler (~1.92m tall)
// +Z is UP, +Y is Forward, +X is Right, -X is Left

$fn = 24;
$asa = 30.0;

titan_animations = [
  ["idle", [
    ["Pelvis", [
      [0.0, [0, 0, -10], [0, 0, 0.94]],
      [0.5, [0, 0, -10], [0, 0, 0.91]],
      [1.0, [0, 0, -10], [0, 0, 0.94]]
    ]],
    ["Torso", [
      [0.0, [-5, 0, 6]],
      [0.5, [-8, 0, 6]],
      [1.0, [-5, 0, 6]]
    ]],
    ["Head", [
      [0.0, [4, 0, 4]],
      [0.5, [6, 0, 4]],
      [1.0, [4, 0, 4]]
    ]],
    ["LeftUpperArm", [
      [0.0, [38, 20, -8]],
      [0.5, [42, 20, -8]],
      [1.0, [38, 20, -8]]
    ]],
    ["LeftForearm", [
      [0.0, [74, 0, 0]],
      [0.5, [78, 0, 0]],
      [1.0, [74, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [24, -18, 8]],
      [0.5, [28, -18, 8]],
      [1.0, [24, -18, 8]]
    ]],
    ["RightForearm", [
      [0.0, [82, 0, 0]],
      [0.5, [86, 0, 0]],
      [1.0, [82, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [20, 10, 0]],
      [0.5, [24, 10, 0]],
      [1.0, [20, 10, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-24, 0, 0]],
      [0.5, [-28, 0, 0]],
      [1.0, [-24, 0, 0]]
    ]],
    ["RightThigh", [
      [0.0, [-16, -10, 0]],
      [0.5, [-20, -10, 0]],
      [1.0, [-16, -10, 0]]
    ]],
    ["RightShin", [
      [0.0, [-16, 0, 0]],
      [0.5, [-20, 0, 0]],
      [1.0, [-16, 0, 0]]
    ]]
  ]],

  ["walk_forward", [
    ["Pelvis", [
      [0.0, [0, 0, -6], [0, 0, 0.94]],
      [0.22, [0, 0, -6], [0, 0, 0.91]],
      [0.44, [0, 0, -6], [0, 0, 0.94]],
      [0.66, [0, 0, -6], [0, 0, 0.91]],
      [0.88, [0, 0, -6], [0, 0, 0.94]]
    ]],
    ["Torso", [
      [0.0, [-8, 0, 6]],
      [0.44, [-8, 0, -6]],
      [0.88, [-8, 0, 6]]
    ]],
    ["LeftUpperArm", [
      [0.0, [34, 14, 0]],
      [0.44, [44, 14, 0]],
      [0.88, [34, 14, 0]]
    ]],
    ["LeftForearm", [
      [0.0, [78, 0, 0]],
      [0.88, [78, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [42, -14, 0]],
      [0.44, [30, -14, 0]],
      [0.88, [42, -14, 0]]
    ]],
    ["RightForearm", [
      [0.0, [82, 0, 0]],
      [0.88, [82, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [28, 6, 0]],
      [0.44, [-22, 6, 0]],
      [0.88, [28, 6, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-14, 0, 0]],
      [0.44, [-28, 0, 0]],
      [0.88, [-14, 0, 0]]
    ]],
    ["RightThigh", [
      [0.0, [-22, -6, 0]],
      [0.44, [28, -6, 0]],
      [0.88, [-22, -6, 0]]
    ]],
    ["RightShin", [
      [0.0, [-28, 0, 0]],
      [0.44, [-14, 0, 0]],
      [0.88, [-28, 0, 0]]
    ]]
  ]],

  ["walk_backward", [
    ["Pelvis", [
      [0.0, [0, 0, -8], [0, 0, 0.93]],
      [0.44, [0, 0, -8], [0, 0, 0.91]],
      [0.88, [0, 0, -8], [0, 0, 0.93]]
    ]],
    ["LeftUpperArm", [
      [0.0, [50, 14, -8]],
      [0.88, [50, 14, -8]]
    ]],
    ["LeftForearm", [
      [0.0, [88, 0, 0]],
      [0.88, [88, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [45, -14, 8]],
      [0.88, [45, -14, 8]]
    ]],
    ["RightForearm", [
      [0.0, [90, 0, 0]],
      [0.88, [90, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [-16, 6, 0]],
      [0.44, [22, 6, 0]],
      [0.88, [-16, 6, 0]]
    ]],
    ["RightThigh", [
      [0.0, [22, -6, 0]],
      [0.44, [-16, -6, 0]],
      [0.88, [22, -6, 0]]
    ]]
  ]],

  ["crouch", [
    ["Pelvis", [
      [0.0, [0, 0, -24], [0, 0, 0.72]],
      [0.4, [0, 0, -24], [0, 0, 0.71]]
    ]],
    ["Torso", [
      [0.0, [-24, 0, 18]],
      [0.4, [-24, 0, 18]]
    ]],
    ["Head", [
      [0.0, [20, 0, 6]],
      [0.4, [20, 0, 6]]
    ]],
    ["LeftUpperArm", [
      [0.0, [52, 16, -12]],
      [0.4, [52, 16, -12]]
    ]],
    ["LeftForearm", [
      [0.0, [94, 0, 0]],
      [0.4, [94, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [44, -16, 14]],
      [0.4, [44, -16, 14]]
    ]],
    ["RightForearm", [
      [0.0, [96, 0, 0]],
      [0.4, [96, 0, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [66, 12, 0]],
      [0.4, [66, 12, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-64, 0, 0]],
      [0.4, [-64, 0, 0]]
    ]],
    ["RightThigh", [
      [0.0, [-46, -14, 0]],
      [0.4, [-46, -14, 0]]
    ]],
    ["RightShin", [
      [0.0, [-18, 0, 0]],
      [0.4, [-18, 0, 0]]
    ]]
  ]],

  ["jump", [
    ["Pelvis", [
      [0.0, [0, 0, 0], [0, 0, 1.00]],
      [0.4, [0, 0, 0], [0, 0, 1.00]]
    ]],
    ["LeftUpperArm", [
      [0.0, [50, 26, 0]],
      [0.4, [50, 26, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [26, -26, 0]],
      [0.4, [26, -26, 0]]
    ]],
    ["LeftThigh", [
      [0.0, [54, 8, 0]],
      [0.4, [54, 8, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-78, 0, 0]],
      [0.4, [-78, 0, 0]]
    ]],
    ["RightThigh", [
      [0.0, [22, -8, 0]],
      [0.4, [22, -8, 0]]
    ]],
    ["RightShin", [
      [0.0, [-50, 0, 0]],
      [0.4, [-50, 0, 0]]
    ]]
  ]],

  ["punch_light", [
    ["Pelvis", [
      [0.0, [0, 0, -10], [0, 0, 0.94]],
      [0.09, [0, 0, 24], [0, 0.08, 0.93]],
      [0.25, [0, 0, -10], [0, 0, 0.94]]
    ]],
    ["Torso", [
      [0.0, [-5, 0, -8]],
      [0.09, [-14, 0, 30]],
      [0.25, [-5, 0, 6]]
    ]],
    ["RightUpperArm", [
      [0.0, [24, -18, 8]],
      [0.09, [88, 0, -10]],
      [0.17, [88, 0, -10]],
      [0.25, [24, -18, 8]]
    ]],
    ["RightForearm", [
      [0.0, [82, 0, 0]],
      [0.09, [10, 0, 0]],
      [0.17, [10, 0, 0]],
      [0.25, [82, 0, 0]]
    ]]
  ]],

  ["punch_light_left", [
    ["Pelvis", [
      [0.0, [0, 0, -10], [0, 0, 0.94]],
      [0.09, [0, 0, -26], [0, 0.08, 0.93]],
      [0.25, [0, 0, -10], [0, 0, 0.94]]
    ]],
    ["Torso", [
      [0.0, [-5, 0, 6]],
      [0.09, [-14, 0, -24]],
      [0.25, [-5, 0, 6]]
    ]],
    ["Head", [
      [0.0, [4, 0, 4]],
      [0.09, [10, 0, 24]],
      [0.25, [4, 0, 4]]
    ]],
    ["LeftUpperArm", [
      [0.0, [38, 20, -8]],
      [0.09, [88, 0, 12]],
      [0.17, [88, 0, 12]],
      [0.25, [38, 20, -8]]
    ]],
    ["LeftForearm", [
      [0.0, [74, 0, 0]],
      [0.09, [10, 0, 0]],
      [0.17, [10, 0, 0]],
      [0.25, [74, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [24, -18, 8]],
      [0.09, [32, -22, 20]],
      [0.25, [24, -18, 8]]
    ]],
    ["RightForearm", [
      [0.0, [82, 0, 0]],
      [0.09, [92, 0, 0]],
      [0.25, [82, 0, 0]]
    ]]
  ]],

  ["kick_heavy", [
    ["Pelvis", [
      [0.0, [0, 0, -10], [0, 0, 0.94]],
      [0.16, [10, -14, 32], [0, 0.12, 0.98]],
      [0.32, [10, -14, 32], [0, 0.12, 0.98]],
      [0.48, [0, 0, -10], [0, 0, 0.94]]
    ]],
    ["Torso", [
      [0.0, [-5, 0, 6]],
      [0.16, [16, 14, -20]],
      [0.32, [16, 14, -20]],
      [0.48, [-5, 0, 6]]
    ]],
    ["RightThigh", [
      [0.0, [-16, -10, 0]],
      [0.10, [62, -10, 0]],
      [0.18, [88, -6, 10]],
      [0.32, [88, -6, 10]],
      [0.48, [-16, -10, 0]]
    ]],
    ["RightShin", [
      [0.0, [-16, 0, 0]],
      [0.10, [-82, 0, 0]],
      [0.18, [-6, 0, 0]],
      [0.32, [-6, 0, 0]],
      [0.48, [-16, 0, 0]]
    ]]
  ]],

  ["kick_heavy_left", [
    ["Pelvis", [
      [0.0, [0, 0, -10], [0, 0, 0.94]],
      [0.16, [10, 14, -32], [0, 0.12, 0.98]],
      [0.32, [10, 14, -32], [0, 0.12, 0.98]],
      [0.48, [0, 0, -10], [0, 0, 0.94]]
    ]],
    ["Torso", [
      [0.0, [-5, 0, 6]],
      [0.16, [16, -14, 20]],
      [0.32, [16, -14, 20]],
      [0.48, [-5, 0, 6]]
    ]],
    ["LeftUpperArm", [
      [0.0, [38, 20, -8]],
      [0.16, [-24, 34, 0]],
      [0.48, [38, 20, -8]]
    ]],
    ["RightUpperArm", [
      [0.0, [24, -18, 8]],
      [0.16, [54, -30, 0]],
      [0.48, [24, -18, 8]]
    ]],
    ["LeftThigh", [
      [0.0, [20, 10, 0]],
      [0.10, [64, 10, 0]],
      [0.18, [88, 6, -10]],
      [0.32, [88, 6, -10]],
      [0.48, [20, 10, 0]]
    ]],
    ["LeftShin", [
      [0.0, [-24, 0, 0]],
      [0.10, [-82, 0, 0]],
      [0.18, [-6, 0, 0]],
      [0.32, [-6, 0, 0]],
      [0.48, [-24, 0, 0]]
    ]],
    ["RightThigh", [
      [0.0, [-16, -10, 0]],
      [0.18, [-8, -10, 0]],
      [0.48, [-16, -10, 0]]
    ]],
    ["RightShin", [
      [0.0, [-16, 0, 0]],
      [0.18, [-12, 0, 0]],
      [0.48, [-16, 0, 0]]
    ]]
  ]],

  ["special_strike", [
    ["Pelvis", [
      [0.0, [0, 0, 0], [0, -0.05, 0.92]],
      [0.15, [0, 0, 0], [0, -0.10, 0.88]],
      [0.28, [0, 0, 0], [0, 0.16, 0.92]],
      [0.52, [0, 0, -10], [0, 0, 0.94]]
    ]],
    ["Torso", [
      [0.0, [8, 0, 0]],
      [0.15, [14, 0, 0]],
      [0.28, [-20, 0, 0]],
      [0.52, [-5, 0, 6]]
    ]],
    ["LeftUpperArm", [
      [0.0, [20, 16, 0]],
      [0.15, [-16, 16, 0]],
      [0.28, [86, 8, -10]],
      [0.40, [86, 8, -10]],
      [0.52, [38, 20, -8]]
    ]],
    ["LeftForearm", [
      [0.0, [74, 0, 0]],
      [0.28, [6, 0, 0]],
      [0.40, [6, 0, 0]],
      [0.52, [74, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [20, -16, 0]],
      [0.15, [-16, -16, 0]],
      [0.28, [86, -8, 10]],
      [0.40, [86, -8, 10]],
      [0.52, [24, -18, 8]]
    ]],
    ["RightForearm", [
      [0.0, [82, 0, 0]],
      [0.28, [6, 0, 0]],
      [0.40, [6, 0, 0]],
      [0.52, [82, 0, 0]]
    ]]
  ]],

  ["block", [
    ["Pelvis", [
      [0.0, [0, 0, -6], [0, 0, 0.88]],
      [0.3, [0, 0, -6], [0, 0, 0.88]]
    ]],
    ["Torso", [
      [0.0, [-10, 0, 10]],
      [0.3, [-10, 0, 10]]
    ]],
    ["LeftUpperArm", [
      [0.0, [66, 10, -28]],
      [0.3, [66, 10, -28]]
    ]],
    ["LeftForearm", [
      [0.0, [94, 0, 0]],
      [0.3, [94, 0, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [72, -10, 28]],
      [0.3, [72, -10, 28]]
    ]],
    ["RightForearm", [
      [0.0, [96, 0, 0]],
      [0.3, [96, 0, 0]]
    ]]
  ]],

  ["hit_stun", [
    ["Pelvis", [
      [0.0, [14, 0, 0], [0, -0.08, 0.92]],
      [0.25, [4, 0, 0], [0, -0.02, 0.94]]
    ]],
    ["Torso", [
      [0.0, [24, 0, -8]],
      [0.25, [8, 0, 0]]
    ]],
    ["Head", [
      [0.0, [24, 0, 10]],
      [0.25, [6, 0, 0]]
    ]]
  ]],

  ["knockdown", [
    ["Pelvis", [
      [0.0, [18, 0, 0], [0, -0.1, 0.88]],
      [0.24, [58, 0, 0], [0, -0.35, 0.48]],
      [0.48, [88, 0, 0], [0, -0.62, 0.18]],
      [0.85, [88, 0, 0], [0, -0.62, 0.18]]
    ]],
    ["LeftUpperArm", [
      [0.0, [20, 35, 0]],
      [0.48, [10, 55, 0]],
      [0.85, [10, 55, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [20, -35, 0]],
      [0.48, [10, -55, 0]],
      [0.85, [10, -55, 0]]
    ]]
  ]],

  ["victory", [
    ["Pelvis", [
      [0.0, [0, 0, 0], [0, 0, 0.94]],
      [0.45, [0, 0, 0], [0, 0, 0.96]],
      [0.9, [0, 0, 0], [0, 0, 0.94]]
    ]],
    ["LeftUpperArm", [
      [0.0, [85, 18, 0]],
      [0.45, [152, 16, 0]],
      [0.9, [152, 16, 0]]
    ]],
    ["RightUpperArm", [
      [0.0, [85, -18, 0]],
      [0.45, [152, -16, 0]],
      [0.9, [152, -16, 0]]
    ]]
  ]]
];

module chassis_gunmetal() {
  color([0.16, 0.15, 0.17], roughness=0.5, metalness=0.5) children();
}

module armor_crimson() {
  color([0.78, 0.12, 0.16], roughness=0.32, metalness=0.72) children();
}

module armor_gold() {
  color([0.92, 0.72, 0.22], roughness=0.25, metalness=0.92) children();
}

module magma_glow() {
  color([1.0, 0.42, 0.08], roughness=0.2, metalness=0.1, emissive=[1.0, 0.35, 0.05], emissiveIntensity=2.6) children();
}

armature(animations=titan_animations) {
  bone(name="Pelvis", t=[0, 0, 0.94], r=[0, 0, 0]) {
    chassis_gunmetal()
      scale([1.3, 0.96, 1.0])
        cylinder(h=0.14, r=0.16, center=true);
    armor_gold()
      translate([0, 0.145, 0.0])
        cube([0.12, 0.04, 0.08], center=true);

    bone(name="Torso", t=[0, 0, 0.13], r=[0, 0, 0]) {
      // Solid waist hydraulic core bridging Pelvis and Chest
      chassis_gunmetal()
        translate([0, 0, -0.02])
          scale([1.15, 0.85, 1.0])
            cylinder(h=0.22, r=0.135, center=true);
      // Heavy chest block
      chassis_gunmetal()
        translate([0, 0, 0.22])
          scale([1.28, 0.90, 1.0])
            cylinder(h=0.38, r1=0.15, r2=0.21, center=true);
      // Raised crimson chest plate
      armor_crimson()
        translate([0, 0.05, 0.24])
          cube([0.36, 0.26, 0.25], center=true);
      // Reactor core
      magma_glow()
        translate([0, 0.185, 0.25])
          rotate([90, 0, 0])
            cylinder(h=0.04, r=0.058, center=true);
      // High collar shield
      armor_gold()
        translate([0, -0.05, 0.41])
          cube([0.34, 0.14, 0.10], center=true);

      bone(name="Head", t=[0, 0, 0.47], r=[0, 0, 0]) {
        armor_crimson()
          translate([0, 0, 0.10])
            cube([0.21, 0.22, 0.22], center=true);
        magma_glow()
          translate([0, 0.112, 0.10])
            cube([0.15, 0.025, 0.045], center=true);
        // Samurai mech horns
        armor_gold()
          translate([-0.10, 0.06, 0.22])
            rotate([15, -28, 0])
              cylinder(h=0.16, r1=0.024, r2=0.004, center=true);
        armor_gold()
          translate([0.10, 0.06, 0.22])
            rotate([15, 28, 0])
              cylinder(h=0.16, r1=0.024, r2=0.004, center=true);
      }

      // Left Arm (-X)
      bone(name="LeftUpperArm", t=[-0.28, 0, 0.36], r=[0, 0, 0]) {
        // Huge pauldron
        armor_crimson()
          translate([-0.03, 0, 0.03])
            scale([1.2, 1.15, 1.0])
              sphere(r=0.115);
        armor_gold()
          translate([-0.06, 0, 0.10])
            cube([0.14, 0.16, 0.03], center=true);
        chassis_gunmetal()
          translate([0, 0, -0.15])
            cylinder(h=0.26, r1=0.065, r2=0.075, center=true);

        bone(name="LeftForearm", t=[0, 0, -0.29], r=[0, 0, 0]) {
          armor_crimson()
            translate([0, 0, -0.13])
              cube([0.14, 0.14, 0.24], center=true);
          magma_glow()
            translate([-0.072, 0.02, -0.13])
              cube([0.02, 0.05, 0.14], center=true);
          armor_gold()
            translate([0, 0.01, -0.27])
              cube([0.125, 0.125, 0.10], center=true);
        }
      }

      // Right Arm (+X)
      bone(name="RightUpperArm", t=[0.28, 0, 0.36], r=[0, 0, 0]) {
        armor_crimson()
          translate([0.03, 0, 0.03])
            scale([1.2, 1.15, 1.0])
              sphere(r=0.115);
        armor_gold()
          translate([0.06, 0, 0.10])
            cube([0.14, 0.16, 0.03], center=true);
        chassis_gunmetal()
          translate([0, 0, -0.15])
            cylinder(h=0.26, r1=0.065, r2=0.075, center=true);

        bone(name="RightForearm", t=[0, 0, -0.29], r=[0, 0, 0]) {
          armor_crimson()
            translate([0, 0, -0.13])
              cube([0.14, 0.14, 0.24], center=true);
          magma_glow()
            translate([0.072, 0.02, -0.13])
              cube([0.02, 0.05, 0.14], center=true);
          armor_gold()
            translate([0, 0.01, -0.27])
              cube([0.125, 0.125, 0.10], center=true);
        }
      }
    }

    // Left Leg (-X)
    bone(name="LeftThigh", t=[-0.14, 0, -0.05], r=[0, 0, 0]) {
      chassis_gunmetal()
        translate([0, 0, -0.21])
          cylinder(h=0.42, r1=0.082, r2=0.10, center=true);
      armor_crimson()
        translate([-0.04, 0.035, -0.19])
          cube([0.13, 0.16, 0.26], center=true);

      bone(name="LeftShin", t=[0, 0, -0.44], r=[0, 0, 0]) {
        armor_gold()
          translate([0, 0.065, 0])
            cube([0.12, 0.08, 0.12], center=true);
        armor_crimson()
          translate([0, 0, -0.21])
            cylinder(h=0.40, r1=0.075, r2=0.088, center=true);
        chassis_gunmetal()
          translate([0, 0.05, -0.42])
            cube([0.15, 0.26, 0.10], center=true);
      }
    }

    // Right Leg (+X)
    bone(name="RightThigh", t=[0.14, 0, -0.05], r=[0, 0, 0]) {
      chassis_gunmetal()
        translate([0, 0, -0.21])
          cylinder(h=0.42, r1=0.082, r2=0.10, center=true);
      armor_crimson()
        translate([0.04, 0.035, -0.19])
          cube([0.13, 0.16, 0.26], center=true);

      bone(name="RightShin", t=[0, 0, -0.44], r=[0, 0, 0]) {
        armor_gold()
          translate([0, 0.065, 0])
            cube([0.12, 0.08, 0.12], center=true);
        armor_crimson()
          translate([0, 0, -0.21])
            cylinder(h=0.40, r1=0.075, r2=0.088, center=true);
        chassis_gunmetal()
          translate([0, 0.05, -0.42])
            cube([0.15, 0.26, 0.10], center=true);
      }
    }
  }
}
```

### cyber_dojo_fighter/assets/models/dojo_arena.scad
```openscad
// Cyber Dojo Fighting Arena
// Floor top surface is at Z = 0.0
// Camera looks from Godot +Z toward -Z, so OpenSCAD +Y is the background of the arena!

$fn = 24;
$asa = 25.0;

arena_anims = [
  ["ambient", [
    ["HoloRing", [
      [0.0, [0, 0, 0]],
      [2.0, [0, 90, 0]],
      [4.0, [0, 180, 0]],
      [6.0, [0, 270, 0]],
      [8.0, [0, 360, 0]]
    ]],
    ["LeftLanternCore", [
      [0.0, [0, 0, 0], [-6.8, 3.2, 1.95]],
      [2.0, [0, 0, 180], [-6.8, 3.2, 2.08]],
      [4.0, [0, 0, 360], [-6.8, 3.2, 1.95]]
    ]],
    ["RightLanternCore", [
      [0.0, [0, 0, 0], [6.8, 3.2, 1.95]],
      [2.0, [0, 0, 180], [6.8, 3.2, 2.08]],
      [4.0, [0, 0, 360], [6.8, 3.2, 1.95]]
    ]]
  ]]
];

module dark_slate() {
  color([0.08, 0.09, 0.13], roughness=0.72, metalness=0.20) children();
}

module tatami_panel() {
  color([0.14, 0.16, 0.22], roughness=0.80, metalness=0.10) children();
}

module crimson_lacquer() {
  color([0.65, 0.10, 0.12], roughness=0.30, metalness=0.35) children();
}

module bronze_trim() {
  color([0.75, 0.58, 0.25], roughness=0.30, metalness=0.85) children();
}

module neon_border() {
  color([0.15, 0.75, 1.0], roughness=0.2, emissive=[0.15, 0.75, 1.0], emissiveIntensity=1.9) children();
}

module lantern_amber() {
  color([1.0, 0.65, 0.15], roughness=0.2, emissive=[1.0, 0.55, 0.1], emissiveIntensity=2.8) children();
}

module lantern_pillar(px, py) {
  translate([px, py, 0]) {
    dark_slate()
      translate([0, 0, 0.25])
        cube([0.9, 0.9, 0.5], center=true);
    crimson_lacquer()
      translate([0, 0, 1.15])
        cylinder(h=1.4, r1=0.24, r2=0.20, center=true);
    bronze_trim()
      translate([0, 0, 2.45])
        cube([0.85, 0.85, 0.18], center=true);
    crimson_lacquer()
      translate([0, 0, 2.75])
        cylinder(h=0.45, r1=0.65, r2=0.10, $fn=4, center=true);
  }
}

armature(animations=arena_anims) {
  bone(name="ArenaRoot", t=[0, 0, 0], r=[0, 0, 0]) {
    // Main fighting platform (top at Z = 0)
    dark_slate()
      translate([0, 0, -0.45])
        cube([17.0, 8.6, 0.9], center=true);

    // Outer stepped base
    dark_slate()
      translate([0, 0, -1.15])
        cube([20.0, 11.0, 0.5], center=true);

    // Tatami deck grid inlays (flush at Z = 0)
    for (ix = [-6 : 2 : 6]) {
      tatami_panel()
        translate([ix, 0, -0.02])
          cube([1.86, 6.4, 0.05], center=true);
    }

    // Glowing arena boundary strips
    neon_border()
      translate([0, 3.35, 0.01])
        cube([14.8, 0.12, 0.04], center=true);
    neon_border()
      translate([0, -3.35, 0.01])
        cube([14.8, 0.12, 0.04], center=true);
    neon_border()
      translate([-7.4, 0, 0.01])
        cube([0.12, 6.82, 0.04], center=true);
    neon_border()
      translate([7.4, 0, 0.01])
        cube([0.12, 6.82, 0.04], center=true);

    // Center fighting emblem ring
    bronze_trim()
      translate([0, 0, 0.005])
        difference() {
          cylinder(h=0.02, r=2.0, center=true, $fn=48);
          cylinder(h=0.04, r=1.82, center=true, $fn=48);
        }

    // Background Cyber-Torii Gate (at +Y so it sits behind the fighters)
    translate([0, 4.4, 0]) {
      // Pillars
      crimson_lacquer()
        translate([-3.4, 0, 2.7])
          cylinder(h=5.4, r=0.28, center=true);
      crimson_lacquer()
        translate([3.4, 0, 2.7])
          cylinder(h=5.4, r=0.28, center=true);
      // Pillar bases
      dark_slate()
        translate([-3.4, 0, 0.35])
          cylinder(h=0.7, r=0.42, center=true);
      dark_slate()
        translate([3.4, 0, 0.35])
          cylinder(h=0.7, r=0.42, center=true);
      // Lower crossbeam (Nuki)
      crimson_lacquer()
        translate([0, 0, 4.2])
          cube([8.4, 0.32, 0.34], center=true);
      // Upper roof beam (Kasagi)
      dark_slate()
        translate([0, 0, 5.45])
          cube([9.8, 0.65, 0.38], center=true);
      crimson_lacquer()
        translate([0, 0, 5.15])
          cube([9.0, 0.45, 0.28], center=true);
      // Center plaque with neon trim
      bronze_trim()
        translate([0, -0.12, 4.7])
          cube([1.1, 0.24, 0.85], center=true);
      neon_border()
        translate([0, -0.25, 4.7])
          cube([0.75, 0.04, 0.50], center=true);
    }

    // Background Lantern Pillars
    lantern_pillar(-6.8, 3.2);
    lantern_pillar(6.8, 3.2);

    // Animated Vertical Holographic Dojo Ring behind the Torii gate
    bone(name="HoloRing", t=[0, 4.8, 2.6], r=[0, 0, 0]) {
      rotate([90, 0, 0]) {
        neon_border()
          difference() {
            cylinder(h=0.06, r=2.1, center=true, $fn=48);
            cylinder(h=0.10, r=1.98, center=true, $fn=48);
          }
        for (a = [0 : 45 : 315]) {
          rotate([0, 0, a])
            translate([2.15, 0, 0])
              bronze_trim()
                cube([0.24, 0.08, 0.08], center=true);
        }
      }
    }

    // Floating Lantern Cores
    bone(name="LeftLanternCore", t=[-6.8, 3.2, 1.95], r=[0, 0, 0]) {
      lantern_amber()
        cylinder(h=0.42, r=0.18, $fn=6, center=true);
    }
    bone(name="RightLanternCore", t=[6.8, 3.2, 1.95], r=[0, 0, 0]) {
      lantern_amber()
        cylinder(h=0.42, r=0.18, $fn=6, center=true);
    }
  }
}

```

### cyber_dojo_fighter/assets/models/energy_wave.scad
```openscad
// Spinning Energy Wave Projectile
$fn = 24;
$asa = 30.0;

wave_anims = [
  ["spin", [
    ["WaveCore", [
      [0.0, [0, 0, 0]],
      [0.15, [0, 90, 0]],
      [0.30, [0, 180, 0]],
      [0.45, [0, 270, 0]],
      [0.60, [0, 360, 0]]
    ]]
  ]]
];

armature(animations=wave_anims) {
  bone(name="WaveCore", t=[0, 0, 0], r=[0, 0, 0]) {
    color([0.4, 0.95, 1.0], roughness=0.1, emissive=[0.2, 0.9, 1.0], emissiveIntensity=3.5)
      sphere(r=0.22);

    color([1.0, 1.0, 1.0], roughness=0.1, emissive=[0.6, 0.98, 1.0], emissiveIntensity=4.0)
      rotate([90, 0, 0])
        difference() {
          cylinder(h=0.06, r=0.36, center=true);
          cylinder(h=0.10, r=0.28, center=true);
        }

    for (a = [0 : 90 : 270]) {
      rotate([0, a, 0])
        translate([0.32, 0, 0])
          color([0.1, 0.8, 1.0], roughness=0.15, emissive=[0.1, 0.85, 1.0], emissiveIntensity=3.0)
            rotate([0, 90, 0])
              cylinder(h=0.20, r1=0.08, r2=0.0, center=true);
    }
  }
}

```

### cyber_dojo_fighter/project.godot
```ini
; Engine configuration file.
; It's best edited using the editor UI and not directly,
; since the parameters that go here are not all obvious.
;
; Format:
;   [section] ; section goes between []
;   param=value ; assign values to parameters

config_version=5

[application]

config/name="Cyber Dojo Fighter"
run/main_scene="res://scenes/main.tscn"
config/features=PackedStringArray("4.3", "Forward Plus")

[display]

window/size/viewport_width=1280
window/size/viewport_height=720
window/stretch/mode="canvas_items"

[editor_plugins]

enabled=PackedStringArray("res://addons/scad_importer/plugin.cfg")

[input]

p1_left={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":65,"physical_keycode":0,"key_label":0,"unicode":97,"location":0,"echo":false,"script":null)
]
}
p1_right={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":68,"physical_keycode":0,"key_label":0,"unicode":100,"location":0,"echo":false,"script":null)
]
}
p1_jump={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":87,"physical_keycode":0,"key_label":0,"unicode":119,"location":0,"echo":false,"script":null)
]
}
p1_crouch={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":83,"physical_keycode":0,"key_label":0,"unicode":115,"location":0,"echo":false,"script":null)
]
}
p1_light={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":70,"physical_keycode":0,"key_label":0,"unicode":102,"location":0,"echo":false,"script":null)
]
}
p1_heavy={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":71,"physical_keycode":0,"key_label":0,"unicode":103,"location":0,"echo":false,"script":null)
]
}
p1_special={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":72,"physical_keycode":0,"key_label":0,"unicode":104,"location":0,"echo":false,"script":null)
]
}
p2_left={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":4194319,"physical_keycode":0,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
p2_right={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":4194321,"physical_keycode":0,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
p2_jump={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":4194320,"physical_keycode":0,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
p2_crouch={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":4194322,"physical_keycode":0,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
p2_light={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":74,"physical_keycode":0,"key_label":0,"unicode":106,"location":0,"echo":false,"script":null)
]
}
p2_heavy={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":75,"physical_keycode":0,"key_label":0,"unicode":107,"location":0,"echo":false,"script":null)
]
}
p2_special={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":76,"physical_keycode":0,"key_label":0,"unicode":108,"location":0,"echo":false,"script":null)
]
}

[rendering]

anti_aliasing/quality/msaa_3d=2

```

### cyber_dojo_fighter/scripts/sound_fx.gd
```gdscript
extends Node

const SAMPLE_RATE := 44100
const POOL_SIZE := 12

var _players: Array[AudioStreamPlayer] = []
var _next_idx := 0
var _streams: Dictionary = {}

func _ready() -> void:
	for i in POOL_SIZE:
		var p := AudioStreamPlayer.new()
		p.volume_db = -5.0
		add_child(p)
		_players.append(p)

	_streams["whoosh"] = _build_whoosh(0.14, 340.0, 1750.0, 280.0, 0.45)
	_streams["whoosh_heavy"] = _build_whoosh(0.22, 190.0, 1150.0, 160.0, 0.65)
	_streams["hit_light"] = _build_hit(0.16, 185.0, 46.0, 1.8, 0.55)
	_streams["hit_heavy"] = _build_hit(0.28, 150.0, 28.0, 2.8, 0.85)
	_streams["block_light"] = _build_block(false)
	_streams["block_heavy"] = _build_block(true)
	_streams["special_wave"] = _build_special_wave()
	_streams["special_dash"] = _build_special_dash()
	_streams["jump"] = _build_jump()
	_streams["land"] = _build_land()
	_streams["ko"] = _build_ko()
	_streams["round"] = _build_round_announce()
	_streams["fight"] = _build_fight_start()
	_streams["victory"] = _build_victory()
	_streams["ui_click"] = _build_ui_click()

func play_whoosh() -> void:
	_play_cached("whoosh", -2.0, 0.08)

func play_whoosh_heavy() -> void:
	_play_cached("whoosh_heavy", -0.5, 0.06)

func play_hit_light() -> void:
	_play_cached("hit_light", 0.0, 0.07)

func play_hit_heavy() -> void:
	_play_cached("hit_heavy", 1.5, 0.05)

func play_block(is_heavy := false) -> void:
	if is_heavy:
		_play_cached("block_heavy", 0.5, 0.05)
	else:
		_play_cached("block_light", 0.0, 0.06)

func play_special(has_wave := true) -> void:
	if has_wave:
		_play_cached("special_wave", 1.0, 0.03)
	else:
		_play_cached("special_dash", 0.0, 0.05)

func play_jump() -> void:
	_play_cached("jump", -3.0, 0.06)

func play_land() -> void:
	_play_cached("land", -4.0, 0.08)

func play_ko() -> void:
	_play_cached("ko", 2.0, 0.0)

func play_round_announce() -> void:
	_play_cached("round", -1.5, 0.0)

func play_fight_start() -> void:
	_play_cached("fight", 0.5, 0.0)

func play_victory() -> void:
	_play_cached("victory", -1.0, 0.0)

func play_ui_click() -> void:
	_play_cached("ui_click", -3.0, 0.04)

func _play_cached(key: String, volume_offset_db := 0.0, pitch_jitter := 0.0) -> void:
	var stream := _streams.get(key) as AudioStreamWAV
	if not stream:
		return
	var player := _players[_next_idx]
	_next_idx = (_next_idx + 1) % _players.size()
	player.stream = stream
	player.volume_db = -5.0 + volume_offset_db
	player.pitch_scale = randf_range(1.0 - pitch_jitter, 1.0 + pitch_jitter) if pitch_jitter > 0.0 else 1.0
	player.play()

func _soft_clip(x: float, drive := 1.0) -> float:
	var v := x * drive
	return v / (1.0 + absf(v))

func _lp_alpha(cutoff_hz: float) -> float:
	var dt := 1.0 / float(SAMPLE_RATE)
	var rc := 1.0 / (TAU * maxf(cutoff_hz, 20.0))
	return dt / (rc + dt)

func _build_whoosh(duration: float, f_start: float, f_peak: float, f_end: float, gain: float) -> AudioStreamWAV:
	var n := int(duration * SAMPLE_RATE)
	var buf := PackedFloat32Array()
	buf.resize(n)

	var lp := 0.0
	var hp := 0.0
	var prev_in := 0.0
	for i in n:
		var t := float(i) / float(n)
		var env := sin(PI * pow(t, 0.72))
		var cutoff := lerpf(f_start, f_peak, t / 0.35) if t < 0.35 else lerpf(f_peak, f_end, (t - 0.35) / 0.65)
		var noise := randf_range(-1.0, 1.0)
		var a := _lp_alpha(cutoff)
		lp = lp + a * (noise - lp)
		hp = 0.96 * (hp + lp - prev_in)
		prev_in = lp
		buf[i] = _soft_clip(hp * env * 2.2, 1.2) * gain

	return _to_wav(buf)

func _build_hit(duration: float, f_start: float, f_end: float, drive: float, gain: float) -> AudioStreamWAV:
	var n := int(duration * SAMPLE_RATE)
	var buf := PackedFloat32Array()
	buf.resize(n)

	var phase := 0.0
	var lp := 0.0
	var hp := 0.0
	var prev_noise := 0.0

	for i in n:
		var t := float(i) / float(n)
		var sec := float(i) / float(SAMPLE_RATE)

		# Sub-bass punch with exponential pitch drop
		var freq := f_end + (f_start - f_end) * exp(-sec * 42.0)
		phase += TAU * freq / float(SAMPLE_RATE)
		var sub := sin(phase) * exp(-sec * 16.0)

		# Crisp impact transient + filtered crunch noise
		var snap_env := exp(-sec * 65.0)
		var tail_env := exp(-sec * 18.0)
		var noise := randf_range(-1.0, 1.0)
		var a := _lp_alpha(lerpf(3600.0, 320.0, t))
		lp = lp + a * (noise - lp)
		hp = 0.92 * (hp + noise - prev_noise)
		prev_noise = noise

		var mixed := sub * 0.9 + lp * tail_env * 0.85 + hp * snap_env * 0.75
		buf[i] = _soft_clip(mixed, drive) * gain

	return _to_wav(buf)

func _build_block(is_heavy: bool) -> AudioStreamWAV:
	var duration := 0.21 if is_heavy else 0.15
	var n := int(duration * SAMPLE_RATE)
	var buf := PackedFloat32Array()
	buf.resize(n)

	var p_thud := 0.0
	var p_s1 := 0.0
	var p_s2 := 0.0
	var p_mod := 0.0
	var lp := 0.0
	var hp := 0.0
	var prev_lp := 0.0

	var base_shield_f := 240.0 if is_heavy else 300.0
	var thud_decay := 30.0 if is_heavy else 42.0
	var shield_decay := 18.0 if is_heavy else 24.0
	var snap_decay := 48.0 if is_heavy else 70.0

	for i in n:
		var t := float(i) / float(n)
		var sec := float(i) / float(SAMPLE_RATE)

		# 1. Tight kinetic guard thud (absorbs physical force)
		var thud_f := 58.0 + (155.0 if is_heavy else 185.0) * exp(-sec * 52.0)
		p_thud += TAU * thud_f / float(SAMPLE_RATE)
		var thud := sin(p_thud) * exp(-sec * thud_decay)

		# 2. Bandpass-filtered deflection transient
		var noise := randf_range(-1.0, 1.0)
		lp = lp + _lp_alpha(lerpf(3200.0, 650.0, t)) * (noise - lp)
		hp = 0.86 * (hp + lp - prev_lp)
		prev_lp = lp
		var crunch := hp * exp(-sec * snap_decay) * 1.4

		# 3. Warm cyber-shield harmonic fifth with subtle impact sag
		var sag := 1.0 - 0.14 * (1.0 - exp(-sec * 24.0))
		var f_root := base_shield_f * sag
		p_s1 += TAU * f_root / float(SAMPLE_RATE)
		p_s2 += TAU * (f_root * 1.498) / float(SAMPLE_RATE)
		p_mod += TAU * (f_root * 3.0) / float(SAMPLE_RATE)

		var mod_amt := 0.65 * exp(-sec * 35.0)
		var shield_osc := sin(p_s1) * 0.55 + sin(p_s2) * 0.35 + sin(p_s1 * 2.0 + sin(p_mod) * mod_amt) * 0.25
		var shield := shield_osc * minf(sec / 0.004, 1.0) * exp(-sec * shield_decay)

		var mixed := thud * (0.95 if is_heavy else 0.78) + crunch * 0.85 + shield * 0.55
		buf[i] = _soft_clip(mixed, 1.9 if is_heavy else 1.6) * (0.78 if is_heavy else 0.70)

	return _to_wav(buf)

func _build_special_wave() -> AudioStreamWAV:
	var duration := 0.56
	var release_sec := 0.22
	var n := int(duration * SAMPLE_RATE)
	var buf := PackedFloat32Array()
	buf.resize(n)

	var p_chg := 0.0
	var p_sub := 0.0
	var p_b1 := 0.0
	var p_b2 := 0.0
	var p_bmod := 0.0
	var lp := 0.0
	var hp := 0.0
	var prev_lp := 0.0

	for i in n:
		var sec := float(i) / float(SAMPLE_RATE)
		var noise := randf_range(-1.0, 1.0)
		var sample := 0.0

		if sec < release_sec:
			# Phase 1: Rising cyber-energy charge gather
			var u := sec / release_sec
			var f_chg := 150.0 * pow(4.2, u)
			p_chg += TAU * f_chg / float(SAMPLE_RATE)

			var pulse := 0.72 + 0.28 * sin(TAU * (18.0 + 26.0 * u) * sec)
			var chg_osc := (sin(p_chg) + 0.5 * sin(p_chg * 2.0 + sin(p_chg * 3.0) * 0.6) + 0.25 * sin(p_chg * 3.0)) * pulse

			lp = lp + _lp_alpha(lerpf(400.0, 2600.0, u)) * (noise - lp)
			hp = 0.90 * (hp + lp - prev_lp)
			prev_lp = lp

			var chg_env := minf(sec / 0.02, 1.0) * pow(u, 0.85)
			sample = (chg_osc * 0.42 + hp * 0.55) * chg_env
		else:
			# Phase 2: Plasma wave cannon release
			var r := sec - release_sec
			var sub_f := 36.0 + 145.0 * exp(-r * 34.0)
			p_sub += TAU * sub_f / float(SAMPLE_RATE)
			var sub := sin(p_sub) * exp(-r * 13.0)

			var beam_f := 180.0 + 740.0 * exp(-r * 9.5)
			p_b1 += TAU * beam_f / float(SAMPLE_RATE)
			p_b2 += TAU * (beam_f * 1.012) / float(SAMPLE_RATE)
			p_bmod += TAU * (beam_f * 2.5) / float(SAMPLE_RATE)

			var fm := 1.4 * exp(-r * 12.0)
			var beam := (sin(p_b1) + 0.6 * sin(p_b2 * 1.5) + 0.4 * sin(p_b1 * 2.0 + sin(p_bmod) * fm)) * exp(-r * 7.5)

			lp = lp + _lp_alpha(420.0 + 3800.0 * exp(-r * 8.0)) * (noise - lp)
			hp = 0.88 * (hp + lp - prev_lp)
			prev_lp = lp
			var blast := hp * exp(-r * 9.0) * 1.15

			sample = sub * 0.75 + beam * 0.52 + blast * 0.65

		buf[i] = _soft_clip(sample, 1.75) * 0.80

	return _to_wav(buf)

func _build_special_dash() -> AudioStreamWAV:
	var duration := 0.44
	var release_sec := 0.21
	var n := int(duration * SAMPLE_RATE)
	var buf := PackedFloat32Array()
	buf.resize(n)

	var p_wind := 0.0
	var p_sub := 0.0
	var lp := 0.0
	var hp := 0.0
	var prev_lp := 0.0

	for i in n:
		var sec := float(i) / float(SAMPLE_RATE)
		var noise := randf_range(-1.0, 1.0)
		var sample := 0.0

		if sec < release_sec:
			var u := sec / release_sec
			var f_w := lerpf(130.0, 310.0, u * u)
			p_wind += TAU * f_w / float(SAMPLE_RATE)
			lp = lp + _lp_alpha(lerpf(300.0, 1400.0, u)) * (noise - lp)
			hp = 0.90 * (hp + lp - prev_lp)
			prev_lp = lp
			sample = (sin(p_wind) * 0.32 + hp * 0.45) * minf(sec / 0.02, 1.0) * pow(u, 0.7)
		else:
			var r := sec - release_sec
			var sub_f := 44.0 + 130.0 * exp(-r * 36.0)
			p_sub += TAU * sub_f / float(SAMPLE_RATE)
			var sub := sin(p_sub) * exp(-r * 22.0)

			lp = lp + _lp_alpha(380.0 + 2600.0 * exp(-r * 12.0)) * (noise - lp)
			hp = 0.88 * (hp + lp - prev_lp)
			prev_lp = lp
			var tear := hp * exp(-r * 12.0) * 1.4

			sample = sub * 0.75 + tear * 0.85

		buf[i] = _soft_clip(sample, 1.65) * 0.72

	return _to_wav(buf)

func _build_jump() -> AudioStreamWAV:
	var duration := 0.14
	var n := int(duration * SAMPLE_RATE)
	var buf := PackedFloat32Array()
	buf.resize(n)

	var phase := 0.0
	var lp := 0.0
	for i in n:
		var t := float(i) / float(n)
		var freq := lerpf(170.0, 470.0, pow(t, 0.7))
		phase += TAU * freq / float(SAMPLE_RATE)
		var tri := 2.0 * absf(2.0 * (phase / TAU - floorf(phase / TAU + 0.5))) - 1.0
		var noise := randf_range(-1.0, 1.0)
		lp = lp + _lp_alpha(1200.0) * (noise - lp)
		var env := minf(t / 0.08, 1.0) * pow(1.0 - t, 1.3)
		buf[i] = _soft_clip((tri * 0.45 + lp * 0.45) * env, 1.1) * 0.5

	return _to_wav(buf)

func _build_land() -> AudioStreamWAV:
	var duration := 0.11
	var n := int(duration * SAMPLE_RATE)
	var buf := PackedFloat32Array()
	buf.resize(n)

	var phase := 0.0
	var lp := 0.0
	for i in n:
		var sec := float(i) / float(SAMPLE_RATE)
		var freq := 36.0 + 78.0 * exp(-sec * 48.0)
		phase += TAU * freq / float(SAMPLE_RATE)
		var thud := sin(phase) * exp(-sec * 30.0)
		var noise := randf_range(-1.0, 1.0)
		lp = lp + _lp_alpha(480.0) * (noise - lp)
		buf[i] = _soft_clip(thud * 0.8 + lp * exp(-sec * 45.0) * 0.6, 1.4) * 0.55

	return _to_wav(buf)

func _build_ko() -> AudioStreamWAV:
	var duration := 0.72
	var n := int(duration * SAMPLE_RATE)
	var buf := PackedFloat32Array()
	buf.resize(n)

	var p_sub := 0.0
	var p_ring := 0.0
	var lp := 0.0

	for i in n:
		var t := float(i) / float(n)
		var sec := float(i) / float(SAMPLE_RATE)

		var sub_f := 24.0 + 135.0 * exp(-sec * 11.0)
		p_sub += TAU * sub_f / float(SAMPLE_RATE)
		var sub := sin(p_sub) * exp(-sec * 4.5)

		var ring_f := lerpf(420.0, 95.0, pow(t, 0.6))
		p_ring += TAU * ring_f / float(SAMPLE_RATE)
		var ring := (sin(p_ring) + 0.5 * sin(p_ring * 1.49)) * exp(-sec * 6.5)

		var noise := randf_range(-1.0, 1.0)
		lp = lp + _lp_alpha(lerpf(2800.0, 180.0, t)) * (noise - lp)
		var blast := lp * exp(-sec * 6.0)

		buf[i] = _soft_clip(sub * 0.95 + ring * 0.4 + blast * 0.75, 2.2) * 0.85

	return _to_wav(buf)

func _build_round_announce() -> AudioStreamWAV:
	var duration := 0.35
	var n := int(duration * SAMPLE_RATE)
	var buf := PackedFloat32Array()
	buf.resize(n)

	var p1 := 0.0
	var p2 := 0.0
	for i in n:
		var sec := float(i) / float(SAMPLE_RATE)
		p1 += TAU * 440.0 / float(SAMPLE_RATE)
		p2 += TAU * 660.0 / float(SAMPLE_RATE)
		var tone := sin(p1) * 0.55 + sin(p2) * 0.45 + 0.2 * sin(p1 * 2.0)
		var env := minf(sec / 0.012, 1.0) * exp(-sec * 8.5)
		buf[i] = _soft_clip(tone * env, 1.1) * 0.55

	return _to_wav(buf)

func _build_fight_start() -> AudioStreamWAV:
	var duration := 0.45
	var n := int(duration * SAMPLE_RATE)
	var buf := PackedFloat32Array()
	buf.resize(n)

	var p1 := 0.0
	var p2 := 0.0
	var p3 := 0.0
	var lp := 0.0
	for i in n:
		var sec := float(i) / float(SAMPLE_RATE)
		var f_base := 110.0 + 70.0 * exp(-sec * 25.0)
		p1 += TAU * f_base / float(SAMPLE_RATE)
		p2 += TAU * 330.0 / float(SAMPLE_RATE)
		p3 += TAU * 495.0 / float(SAMPLE_RATE)

		var gong := sin(p1) * 0.75 + sin(p2) * 0.45 + sin(p3) * 0.35 + sin(p2 * 2.41) * 0.2
		var noise := randf_range(-1.0, 1.0)
		lp = lp + _lp_alpha(1800.0) * (noise - lp)
		var impact := lp * exp(-sec * 28.0) * 0.6

		var env := minf(sec / 0.008, 1.0) * exp(-sec * 6.0)
		buf[i] = _soft_clip(gong * env + impact, 1.5) * 0.7

	return _to_wav(buf)

func _build_victory() -> AudioStreamWAV:
	var duration := 0.65
	var n := int(duration * SAMPLE_RATE)
	var buf := PackedFloat32Array()
	buf.resize(n)

	var notes := [523.25, 659.25, 783.99, 1046.5]
	var note_len := 0.11
	var phase := 0.0
	var harm_phase := 0.0

	for i in n:
		var sec := float(i) / float(SAMPLE_RATE)
		var idx := mini(int(sec / note_len), notes.size() - 1)
		var local_sec := sec - float(idx) * note_len
		var freq := float(notes[idx])

		phase += TAU * freq / float(SAMPLE_RATE)
		harm_phase += TAU * (freq * 1.5) / float(SAMPLE_RATE)

		var decay := 4.5 if idx == notes.size() - 1 else 11.0
		var env := minf(local_sec / 0.008, 1.0) * exp(-local_sec * decay)
		var tone := sin(phase) + 0.35 * sin(phase * 2.0) + 0.18 * sin(harm_phase)
		buf[i] = _soft_clip(tone * env, 1.1) * 0.55

	return _to_wav(buf)

func _build_ui_click() -> AudioStreamWAV:
	var duration := 0.055
	var n := int(duration * SAMPLE_RATE)
	var buf := PackedFloat32Array()
	buf.resize(n)

	var phase := 0.0
	for i in n:
		var t := float(i) / float(n)
		var freq := lerpf(880.0, 1420.0, t)
		phase += TAU * freq / float(SAMPLE_RATE)
		var env := minf(t / 0.08, 1.0) * pow(1.0 - t, 1.8)
		buf[i] = sin(phase) * env * 0.45

	return _to_wav(buf)

func _to_wav(samples: PackedFloat32Array) -> AudioStreamWAV:
	var n := samples.size()
	var fade_samples := mini(int(0.003 * SAMPLE_RATE), n / 4)
	var data := PackedByteArray()
	data.resize(n * 2)

	for i in n:
		var s := samples[i]
		if i < fade_samples:
			s *= float(i) / float(fade_samples)
		elif i >= n - fade_samples:
			s *= float(n - 1 - i) / float(fade_samples)
		data.encode_s16(i * 2, clampi(int(s * 32767.0), -32768, 32767))

	var wav := AudioStreamWAV.new()
	wav.format = AudioStreamWAV.FORMAT_16_BITS
	wav.mix_rate = SAMPLE_RATE
	wav.stereo = false
	wav.data = data
	return wav
```

### cyber_dojo_fighter/scripts/fighting_camera.gd
```gdscript
extends Camera3D

@export var min_distance := 6.6
@export var max_distance := 10.6
@export var height_offset := 1.85
@export var look_height := 1.10
@export var smooth_speed := 7.0

var fighter_a: Node3D
var fighter_b: Node3D

var _shake_amount := 0.0

func add_shake(amount: float) -> void:
	_shake_amount = maxf(_shake_amount, amount)

func _process(delta: float) -> void:
	if not is_instance_valid(fighter_a) or not is_instance_valid(fighter_b):
		return

	var mid_x := (fighter_a.global_position.x + fighter_b.global_position.x) * 0.5
	var mid_y := maxf(fighter_a.global_position.y, fighter_b.global_position.y) * 0.35
	var sep := absf(fighter_a.global_position.x - fighter_b.global_position.x)

	var target_z := clampf(5.4 + sep * 0.72, min_distance, max_distance)
	var target_pos := Vector3(clampf(mid_x, -3.2, 3.2), height_offset + mid_y, target_z)
	global_position = global_position.lerp(target_pos, clampf(smooth_speed * delta, 0.0, 1.0))

	var look_target := Vector3(clampf(mid_x, -3.5, 3.5), look_height + mid_y * 0.5, 0.0)
	look_at(look_target, Vector3.UP)

	if _shake_amount > 0.001:
		h_offset = randf_range(-_shake_amount, _shake_amount)
		v_offset = randf_range(-_shake_amount, _shake_amount)
		_shake_amount = lerpf(_shake_amount, 0.0, clampf(12.0 * delta, 0.0, 1.0))
	else:
		h_offset = 0.0
		v_offset = 0.0

```

### cyber_dojo_fighter/scripts/projectile.gd
```gdscript
extends Node3D

signal hit_fighter(target: Node3D, damage: float, knockback: Vector2, is_heavy: bool)

var direction := 1.0
var speed := 11.5
var damage := 22.0
var owner_fighter: Node3D
var opponent_fighter: Node3D

var _lifetime := 2.2
var _model: Node3D

func setup(p_owner: Node3D, p_opponent: Node3D, dir: float, tint_color: Color) -> void:
	owner_fighter = p_owner
	opponent_fighter = p_opponent
	direction = dir
	# Orient forward (-Z) along flight direction (+X or -X)
	look_at(global_position + Vector3(direction, 0.0, 0.0), Vector3.UP)
	var light := get_node_or_null("GlowLight") as OmniLight3D
	if light:
		light.light_color = tint_color
	_spawn_trail_particles(tint_color)

func _ready() -> void:
	var scene := load("res://assets/models/energy_wave.scad") as PackedScene
	if scene:
		_model = scene.instantiate()
		add_child(_model)
		var anim := _find_animation_player(_model)
		if anim and anim.has_animation("spin"):
			anim.get_animation("spin").loop_mode = Animation.LOOP_LINEAR
			anim.play("spin")

func _process(delta: float) -> void:
	_lifetime -= delta
	if _lifetime <= 0.0 or absf(global_position.x) > 8.5:
		queue_free()
		return

	# Move along local forward (-basis.z)
	global_position += -global_transform.basis.z * speed * delta

	if is_instance_valid(opponent_fighter) and opponent_fighter.is_alive():
		var dx := absf(global_position.x - opponent_fighter.global_position.x)
		var dy := absf(global_position.y - (opponent_fighter.global_position.y + 0.95))
		if dx < 0.72 and dy < 0.95:
			emit_signal("hit_fighter", opponent_fighter, damage, Vector2(direction * 5.2, 2.2), true)
			queue_free()

func _spawn_trail_particles(tint_color: Color) -> void:
	var trail := CPUParticles3D.new()
	trail.emitting = false
	trail.local_coords = false
	trail.amount = 24
	trail.lifetime = 0.28
	trail.emission_shape = CPUParticles3D.EMISSION_SHAPE_SPHERE
	trail.emission_sphere_radius = 0.22
	trail.direction = Vector3(0.0, 0.0, 1.0)
	trail.spread = 35.0
	trail.initial_velocity_min = 0.6
	trail.initial_velocity_max = 1.8
	trail.gravity = Vector3.ZERO

	var scale_curve := Curve.new()
	scale_curve.add_point(Vector2(0.0, 1.0))
	scale_curve.add_point(Vector2(1.0, 0.0))
	trail.scale_amount_curve = scale_curve

	var grad := Gradient.new()
	var hdr_tint := Color(tint_color.r * 2.0, tint_color.g * 2.0, tint_color.b * 2.0, 0.95)
	grad.offsets = PackedFloat32Array([0.0, 1.0])
	grad.colors = PackedColorArray([hdr_tint, Color(tint_color.r, tint_color.g, tint_color.b, 0.0)])
	trail.color_ramp = grad

	var mesh := SphereMesh.new()
	mesh.radius = 0.055
	mesh.height = 0.11
	var mat := StandardMaterial3D.new()
	mat.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	mat.transparency = BaseMaterial3D.TRANSPARENCY_ALPHA
	mat.blend_mode = BaseMaterial3D.BLEND_MODE_ADD
	mat.vertex_color_use_as_albedo = true
	mesh.material = mat
	trail.mesh = mesh

	add_child(trail)
	trail.restart()
	trail.emitting = true

func _find_animation_player(root: Node) -> AnimationPlayer:
	if root is AnimationPlayer:
		return root
	for child in root.get_children():
		var found := _find_animation_player(child)
		if found:
			return found
	return null
```

### cyber_dojo_fighter/scripts/fighter.gd
```gdscript
extends Node3D

signal hit_landed(attacker: Node3D, defender: Node3D, damage: float, blocked: bool, is_heavy: bool, hit_pos: Vector3)
signal meter_changed(fighter: Node3D, new_meter: float)
signal health_changed(fighter: Node3D, new_health: float)
signal fighter_ko(fighter: Node3D)

enum State {
	IDLE,
	WALK,
	CROUCH,
	JUMP,
	ATTACK_LIGHT,
	ATTACK_HEAVY,
	ATTACK_SPECIAL,
	BLOCK,
	BLOCK_STUN,
	HIT_STUN,
	KNOCKDOWN,
	VICTORY
}

@export var fighter_name := "STRIKER"
@export var model_path := "res://assets/models/fighter_striker.scad"
@export var accent_color := Color(0.15, 0.85, 1.0)
@export var max_health := 100.0
@export var walk_speed := 4.4
@export var jump_velocity := 9.2
@export var gravity := 24.0
@export var is_player_one := true
@export var is_ai := false

var health := 100.0
var super_meter := 0.0
var state := State.IDLE
var facing_dir := 1.0 # +1 faces right (+X), -1 faces left (-X)
var velocity := Vector3.ZERO
var opponent: Node3D
var input_enabled := false

var _model_root: Node3D
var _anim_player: AnimationPlayer
var _current_anim := ""

var _state_timer := 0.0
var _attack_hit_done := false
var _use_left_punch := true
var _use_left_kick := false
var _ai_think_timer := 0.0
var _ai_move_dir := 0.0
var _ai_wants_block := false
var _ai_wants_crouch := false

const STAGE_LIMIT_X := 6.7

func _ready() -> void:
	load_fighter_model(model_path)
	reset_for_round(global_position.x, facing_dir)

func load_fighter_model(path: String) -> void:
	model_path = path
	if is_instance_valid(_model_root):
		_model_root.queue_free()
		_model_root = null
		_anim_player = null

	var scene := load(path) as PackedScene
	if scene:
		_model_root = scene.instantiate()
		add_child(_model_root)
		_anim_player = _find_animation_player(_model_root)
		_configure_loops()

func reset_for_round(start_x: float, start_facing: float) -> void:
	health = max_health
	state = State.IDLE
	velocity = Vector3.ZERO
	facing_dir = start_facing
	global_position = Vector3(start_x, 0.0, 0.0)
	_state_timer = 0.0
	_attack_hit_done = false
	_use_left_punch = true
	_use_left_kick = false
	_update_facing_rotation()
	_play_anim("idle", 0.05)
	emit_signal("health_changed", self, health)
	emit_signal("meter_changed", self, super_meter)

func is_alive() -> bool:
	return health > 0.0 and state != State.KNOCKDOWN

func set_victory() -> void:
	state = State.VICTORY
	velocity = Vector3.ZERO
	_play_anim("victory", 0.1)

func _physics_process(delta: float) -> void:
	# Apply gravity & vertical motion
	if global_position.y > 0.0 or velocity.y > 0.0:
		velocity.y -= gravity * delta

	global_position += velocity * delta

	if global_position.y <= 0.0:
		global_position.y = 0.0
		if velocity.y < 0.0:
			var impact_vy := -velocity.y
			velocity.y = 0.0
			if state == State.JUMP:
				state = State.IDLE
				velocity.x = 0.0
				if impact_vy > 2.5:
					get_node("/root/Main/SoundFX").play_land()

	global_position.x = clampf(global_position.x, -STAGE_LIMIT_X, STAGE_LIMIT_X)

	# Keep pushbox separation when both fighters are near ground
	if is_instance_valid(opponent) and absf(global_position.y - opponent.global_position.y) < 0.8:
		var dx := global_position.x - opponent.global_position.x
		var min_sep := 0.88
		if absf(dx) < min_sep:
			var push_dir := 1.0 if dx >= 0.0 else -1.0
			if absf(dx) < 0.01:
				push_dir = -facing_dir
			global_position.x = clampf(global_position.x + push_dir * (min_sep - absf(dx)) * 0.5, -STAGE_LIMIT_X, STAGE_LIMIT_X)

	if state == State.KNOCKDOWN or state == State.VICTORY:
		velocity.x = lerpf(velocity.x, 0.0, clampf(8.0 * delta, 0.0, 1.0))
		return

	if not input_enabled:
		velocity.x = lerpf(velocity.x, 0.0, clampf(10.0 * delta, 0.0, 1.0))
		if state != State.IDLE and global_position.y <= 0.01:
			state = State.IDLE
			_play_anim("idle")
		return

	# Auto-face opponent while grounded and neutral
	if is_instance_valid(opponent) and global_position.y <= 0.01 and state in [State.IDLE, State.WALK, State.CROUCH, State.BLOCK]:
		var desired := 1.0 if opponent.global_position.x >= global_position.x else -1.0
		if desired != facing_dir:
			facing_dir = desired
			_update_facing_rotation()

	match state:
		State.IDLE, State.WALK, State.CROUCH, State.BLOCK:
			_handle_neutral_input(delta)
		State.JUMP:
			_handle_air_input()
		State.ATTACK_LIGHT:
			_process_attack(delta, 0.07, 0.16, 0.28, 1.48, 8.0, Vector2(facing_dir * 2.2, 0.0), false)
		State.ATTACK_HEAVY:
			_process_attack(delta, 0.15, 0.30, 0.48, 1.85, 16.0, Vector2(facing_dir * 4.8, 2.5), true)
		State.ATTACK_SPECIAL:
			_process_special_attack(delta)
		State.BLOCK_STUN:
			velocity.x = lerpf(velocity.x, 0.0, clampf(12.0 * delta, 0.0, 1.0))
			_state_timer -= delta
			if _state_timer <= 0.0:
				state = State.IDLE
				_handle_neutral_input(delta)
		State.HIT_STUN:
			velocity.x = lerpf(velocity.x, 0.0, clampf(9.0 * delta, 0.0, 1.0))
			_state_timer -= delta
			if _state_timer <= 0.0:
				state = State.IDLE if global_position.y <= 0.01 else State.JUMP
				_play_anim("idle" if state == State.IDLE else "jump")

func _handle_neutral_input(delta: float) -> void:
	var move_x := 0.0
	var want_jump := false
	var want_crouch := false
	var want_light := false
	var want_heavy := false
	var want_special := false

	if is_ai:
		var ai_cmds: Dictionary = _compute_ai(delta)
		move_x = float(ai_cmds["move_x"])
		want_jump = bool(ai_cmds["jump"])
		want_crouch = bool(ai_cmds["crouch"])
		want_light = bool(ai_cmds["light"])
		want_heavy = bool(ai_cmds["heavy"])
		want_special = bool(ai_cmds["special"])
	else:
		var prefix := "p1_" if is_player_one else "p2_"
		if Input.is_action_pressed(prefix + "left"):
			move_x -= 1.0
		if Input.is_action_pressed(prefix + "right"):
			move_x += 1.0
		want_jump = Input.is_action_just_pressed(prefix + "jump")
		want_crouch = Input.is_action_pressed(prefix + "crouch")
		want_light = Input.is_action_just_pressed(prefix + "light")
		want_heavy = Input.is_action_just_pressed(prefix + "heavy")
		want_special = Input.is_action_just_pressed(prefix + "special")

	if want_special:
		_start_special()
		return
	if want_heavy:
		_start_heavy()
		return
	if want_light:
		_start_light()
		return

	if want_jump and global_position.y <= 0.01:
		state = State.JUMP
		velocity.y = jump_velocity
		velocity.x = move_x * walk_speed
		_play_anim("jump", 0.05)
		get_node("/root/Main/SoundFX").play_jump()
		return

	if _ai_wants_block and is_ai:
		state = State.BLOCK
		velocity.x = 0.0
		_play_anim("block", 0.05)
		return

	if want_crouch:
		state = State.CROUCH
		velocity.x = 0.0
		_play_anim("crouch", 0.06)
		return

	if absf(move_x) > 0.1:
		state = State.WALK
		var forward_ratio := move_x * facing_dir
		var spd := walk_speed if forward_ratio > 0.0 else walk_speed * 0.78
		velocity.x = move_x * spd
		_play_anim("walk_forward" if forward_ratio > 0.0 else "walk_backward", 0.08)
	else:
		state = State.IDLE
		velocity.x = 0.0
		_play_anim("idle", 0.1)

func _handle_air_input() -> void:
	if is_ai:
		return
	var prefix := "p1_" if is_player_one else "p2_"
	if Input.is_action_just_pressed(prefix + "light"):
		_start_light()
	elif Input.is_action_just_pressed(prefix + "heavy"):
		_start_heavy()

func _start_light() -> void:
	state = State.ATTACK_LIGHT
	_state_timer = 0.0
	_attack_hit_done = false
	velocity.x *= 0.25
	var anim_name := "punch_light_left" if _use_left_punch else "punch_light"
	_use_left_punch = not _use_left_punch
	if _anim_player and not _anim_player.has_animation(anim_name):
		anim_name = "punch_light"
	_play_anim(anim_name, 0.03)
	get_node("/root/Main/SoundFX").play_whoosh()

func _start_heavy() -> void:
	state = State.ATTACK_HEAVY
	_state_timer = 0.0
	_attack_hit_done = false
	velocity.x *= 0.2
	var anim_name := "kick_heavy_left" if _use_left_kick else "kick_heavy"
	_use_left_kick = not _use_left_kick
	if _anim_player and not _anim_player.has_animation(anim_name):
		anim_name = "kick_heavy"
	_play_anim(anim_name, 0.04)
	get_node("/root/Main/SoundFX").play_whoosh_heavy()

func _start_special() -> void:
	state = State.ATTACK_SPECIAL
	_state_timer = 0.0
	_attack_hit_done = false
	velocity.x = 0.0
	_play_anim("special_strike", 0.04)
	get_node("/root/Main/SoundFX").play_special(super_meter >= 40.0)

func _process_attack(delta: float, active_start: float, active_end: float, total_dur: float, reach: float, dmg: float, kb: Vector2, is_heavy: bool) -> void:
	_state_timer += delta
	if global_position.y <= 0.01:
		velocity.x = lerpf(velocity.x, 0.0, clampf(12.0 * delta, 0.0, 1.0))

	if not _attack_hit_done and _state_timer >= active_start and _state_timer <= active_end:
		if _check_melee_hit(reach):
			_attack_hit_done = true
			_add_meter(12.0 if not is_heavy else 18.0)
			var contact_x := lerpf(global_position.x, opponent.global_position.x, 0.72)
			var contact_y := opponent.global_position.y + (1.05 if is_heavy else 1.25)
			opponent.receive_hit(self, dmg, kb, is_heavy, Vector3(contact_x, contact_y, 0.0))

	if _state_timer >= total_dur:
		state = State.IDLE if global_position.y <= 0.01 else State.JUMP
		_play_anim("idle" if state == State.IDLE else "jump", 0.08)

func _process_special_attack(delta: float) -> void:
	_state_timer += delta
	velocity.x = lerpf(velocity.x, 0.0, clampf(10.0 * delta, 0.0, 1.0))

	if not _attack_hit_done and _state_timer >= 0.24:
		_attack_hit_done = true
		if super_meter >= 40.0:
			_add_meter(-40.0)
			_spawn_energy_wave()
		else:
			# Dash Palm Strike when meter < 40
			velocity.x = facing_dir * 7.5
			if _check_melee_hit(2.0):
				_add_meter(15.0)
				var contact_x := lerpf(global_position.x, opponent.global_position.x, 0.72)
				opponent.receive_hit(self, 12.0, Vector2(facing_dir * 5.0, 1.8), true, Vector3(contact_x, opponent.global_position.y + 1.1, 0.0))

	if _state_timer >= 0.54:
		state = State.IDLE
		_play_anim("idle", 0.08)

func _spawn_energy_wave() -> void:
	var main := get_node("/root/Main")
	if main and main.has_method("spawn_projectile"):
		main.spawn_projectile(self, opponent, global_position + Vector3(facing_dir * 0.75, 1.0, 0.0), facing_dir, accent_color)

func _check_melee_hit(reach: float) -> bool:
	if not is_instance_valid(opponent) or not opponent.is_alive():
		return false
	var dx := (opponent.global_position.x - global_position.x) * facing_dir
	var dy := absf(opponent.global_position.y - global_position.y)
	return dx >= -0.35 and dx <= reach and dy <= 1.25

func is_attacking() -> bool:
	return state in [State.ATTACK_LIGHT, State.ATTACK_HEAVY, State.ATTACK_SPECIAL]

func is_blocking() -> bool:
	if state in [State.BLOCK, State.BLOCK_STUN, State.CROUCH]:
		return true
	if state == State.WALK and (velocity.x * facing_dir) < -0.1:
		return true
	return false

func receive_hit(attacker: Node3D, damage: float, knockback: Vector2, is_heavy: bool, hit_pos: Vector3) -> void:
	if not is_alive():
		return

	var blocked: bool = is_blocking() and ((attacker.global_position.x - global_position.x) * facing_dir >= -0.2)
	var final_damage: float = damage * 0.2 if blocked else damage

	health = maxf(0.0, health - final_damage)
	emit_signal("health_changed", self, health)
	emit_signal("hit_landed", attacker, self, final_damage, blocked, is_heavy, hit_pos)
	_add_meter(6.0 if blocked else 10.0)

	if health <= 0.0:
		state = State.KNOCKDOWN
		velocity.x = knockback.x * 1.1
		velocity.y = maxf(knockback.y, 3.2)
		_play_anim("knockdown", 0.04)
		emit_signal("fighter_ko", self)
		return

	if blocked:
		velocity.x = knockback.x * 0.45
		state = State.BLOCK_STUN
		_state_timer = 0.20 if is_heavy else 0.13
		_play_anim("block", 0.03)
	else:
		state = State.HIT_STUN
		_state_timer = 0.28 if is_heavy else 0.17
		velocity.x = knockback.x * 0.75
		if knockback.y > 0.5 and is_heavy:
			velocity.y = knockback.y * 0.65
		_play_anim("hit_stun", 0.02)

func _add_meter(amount: float) -> void:
	super_meter = clampf(super_meter + amount, 0.0, 100.0)
	emit_signal("meter_changed", self, super_meter)

func _update_facing_rotation() -> void:
	# OpenSCAD +Y faces Godot -Z. look_at aligns -Z toward target!
	look_at(global_position + Vector3(facing_dir, 0.0, 0.0), Vector3.UP)

func _compute_ai(delta: float) -> Dictionary:
	var cmds := {
		"move_x": 0.0,
		"jump": false,
		"crouch": false,
		"light": false,
		"heavy": false,
		"special": false
	}
	if not is_instance_valid(opponent):
		return cmds

	var dist := absf(opponent.global_position.x - global_position.x)
	var dir_to_opp := 1.0 if opponent.global_position.x >= global_position.x else -1.0

	_ai_think_timer -= delta
	if _ai_think_timer <= 0.0:
		_ai_think_timer = randf_range(0.14, 0.30)
		_ai_wants_block = false
		_ai_wants_crouch = false

		if bool(opponent.is_attacking()) and dist < 2.3 and randf() < 0.55:
			_ai_wants_block = true
			_ai_move_dir = -dir_to_opp
		elif dist > 3.6:
			_ai_move_dir = dir_to_opp
			if super_meter >= 40.0 and randf() < 0.32:
				cmds["special"] = true
		elif dist > 1.7:
			var roll := randf()
			if roll < 0.55:
				_ai_move_dir = dir_to_opp
			elif roll < 0.78:
				_ai_move_dir = 0.0
				cmds["heavy"] = true
			elif roll < 0.90:
				_ai_move_dir = -dir_to_opp
			else:
				cmds["jump"] = true
				_ai_move_dir = dir_to_opp
		else:
			var close_roll := randf()
			_ai_move_dir = 0.0
			if close_roll < 0.45:
				cmds["light"] = true
			elif close_roll < 0.72:
				cmds["heavy"] = true
			elif close_roll < 0.85 and super_meter >= 40.0:
				cmds["special"] = true
			else:
				_ai_move_dir = -dir_to_opp

	cmds["move_x"] = _ai_move_dir
	cmds["crouch"] = _ai_wants_crouch
	return cmds

func _configure_loops() -> void:
	if not _anim_player:
		return
	for loop_name in ["idle", "walk_forward", "walk_backward", "crouch", "jump", "block", "victory"]:
		if _anim_player.has_animation(loop_name):
			_anim_player.get_animation(loop_name).loop_mode = Animation.LOOP_LINEAR

func _play_anim(anim_name: String, blend := 0.08) -> void:
	if not _anim_player:
		return
	if _current_anim == anim_name and _anim_player.is_playing():
		return
	if _anim_player.has_animation(anim_name):
		_current_anim = anim_name
		_anim_player.play(anim_name, blend)

func _find_animation_player(root: Node) -> AnimationPlayer:
	if root is AnimationPlayer:
		return root
	for child in root.get_children():
		var found := _find_animation_player(child)
		if found:
			return found
	return null
```

### cyber_dojo_fighter/scripts/hud.gd
```gdscript
extends CanvasLayer

signal start_1p_requested
signal start_2p_requested
signal rematch_requested
signal menu_requested

@onready var p1_health_bar: ProgressBar = $TopHUD/P1Box/P1HealthBar
@onready var p1_trail_bar: ProgressBar = $TopHUD/P1Box/P1HealthBar/P1TrailBar
@onready var p2_health_bar: ProgressBar = $TopHUD/P2Box/P2HealthBar
@onready var p2_trail_bar: ProgressBar = $TopHUD/P2Box/P2HealthBar/P2TrailBar

@onready var p1_name_label: Label = $TopHUD/P1Box/P1Header/P1Name
@onready var p2_name_label: Label = $TopHUD/P2Box/P2Header/P2Name
@onready var p1_pips: HBoxContainer = $TopHUD/P1Box/P1Header/P1Pips
@onready var p2_pips: HBoxContainer = $TopHUD/P2Box/P2Header/P2Pips
@onready var timer_label: Label = $TopHUD/TimerPanel/TimerLabel

@onready var p1_meter_bar: ProgressBar = $BottomHUD/P1MeterBox/P1MeterBar
@onready var p2_meter_bar: ProgressBar = $BottomHUD/P2MeterBox/P2MeterBar

@onready var p1_combo_label: Label = $P1ComboLabel
@onready var p2_combo_label: Label = $P2ComboLabel
@onready var announcer_label: Label = $AnnouncerLabel

@onready var title_panel: PanelContainer = $TitlePanel
@onready var result_panel: PanelContainer = $ResultPanel
@onready var winner_label: Label = $ResultPanel/ResultVBox/WinnerLabel
@onready var touch_controls: Control = $TouchControls

var _p1_combo_timer := 0.0
var _p2_combo_timer := 0.0
var _announce_timer := 0.0

func _ready() -> void:
	p1_combo_label.visible = false
	p2_combo_label.visible = false
	announcer_label.visible = false
	result_panel.visible = false
	title_panel.visible = true

	$TitlePanel/TitleVBox/Btn1P.pressed.connect(func(): emit_signal("start_1p_requested"))
	$TitlePanel/TitleVBox/Btn2P.pressed.connect(func(): emit_signal("start_2p_requested"))
	$ResultPanel/ResultVBox/BtnRematch.pressed.connect(func(): emit_signal("rematch_requested"))
	$ResultPanel/ResultVBox/BtnMenu.pressed.connect(func(): emit_signal("menu_requested"))
	touch_controls.menu_pressed.connect(func(): emit_signal("menu_requested"))

func _process(delta: float) -> void:
	p1_trail_bar.value = lerpf(p1_trail_bar.value, p1_health_bar.value, clampf(4.5 * delta, 0.0, 1.0))
	p2_trail_bar.value = lerpf(p2_trail_bar.value, p2_health_bar.value, clampf(4.5 * delta, 0.0, 1.0))

	if _p1_combo_timer > 0.0:
		_p1_combo_timer -= delta
		if _p1_combo_timer <= 0.0:
			p1_combo_label.visible = false

	if _p2_combo_timer > 0.0:
		_p2_combo_timer -= delta
		if _p2_combo_timer <= 0.0:
			p2_combo_label.visible = false

	if _announce_timer > 0.0:
		_announce_timer -= delta
		if _announce_timer <= 0.0:
			announcer_label.visible = false

func show_title() -> void:
	title_panel.visible = true
	result_panel.visible = false
	announcer_label.visible = false
	touch_controls.set_gameplay_active(false, false)

func hide_menus() -> void:
	title_panel.visible = false
	result_panel.visible = false

func set_gameplay_active(active: bool, is_two_player: bool) -> void:
	touch_controls.set_gameplay_active(active, is_two_player)

func show_result(winner_text: String) -> void:
	winner_label.text = winner_text
	result_panel.visible = true

func set_fighter_names(p1_name: String, p2_name: String) -> void:
	p1_name_label.text = p1_name
	p2_name_label.text = p2_name

func update_health(is_p1: bool, val: float) -> void:
	if is_p1:
		p1_health_bar.value = val
	else:
		p2_health_bar.value = val

func update_meter(is_p1: bool, val: float) -> void:
	if is_p1:
		p1_meter_bar.value = val
	else:
		p2_meter_bar.value = val
	touch_controls.set_special_ready(is_p1, val >= 40.0)

func update_timer(seconds_left: int) -> void:
	timer_label.text = "%02d" % clampi(seconds_left, 0, 99)

func update_round_pips(p1_wins: int, p2_wins: int) -> void:
	for i in p1_pips.get_child_count():
		var pip := p1_pips.get_child(i) as ColorRect
		pip.color = Color(1.0, 0.82, 0.15) if i < p1_wins else Color(0.25, 0.28, 0.35)
	for i in p2_pips.get_child_count():
		var pip := p2_pips.get_child(i) as ColorRect
		pip.color = Color(1.0, 0.82, 0.15) if i < p2_wins else Color(0.25, 0.28, 0.35)

func show_combo(is_p1_attacker: bool, count: int) -> void:
	if count < 2:
		return
	if is_p1_attacker:
		p1_combo_label.text = "%d HIT COMBO!" % count
		p1_combo_label.visible = true
		_p1_combo_timer = 1.4
	else:
		p2_combo_label.text = "%d HIT COMBO!" % count
		p2_combo_label.visible = true
		_p2_combo_timer = 1.4

func announce(text: String, duration := 1.2) -> void:
	announcer_label.text = text
	announcer_label.visible = true
	_announce_timer = duration
```

### cyber_dojo_fighter/scripts/game_manager.gd
```gdscript
extends Node3D

enum MatchState {
	MENU,
	ROUND_INTRO,
	FIGHTING,
	ROUND_OVER,
	MATCH_OVER
}

@onready var fighter_p1: Node3D = $FighterP1
@onready var fighter_p2: Node3D = $FighterP2
@onready var fighting_camera: Camera3D = $FightingCamera
@onready var hud: CanvasLayer = $HUD
@onready var sound_fx: Node = $SoundFX
@onready var arena_holder: Node3D = $ArenaHolder
@onready var effects_holder: Node3D = $EffectsHolder

var _projectile_scene: PackedScene
var _match_state := MatchState.MENU
var _round_number := 1
var _p1_wins := 0
var _p2_wins := 0
var _round_timer := 99.0
var _two_player_mode := false

var _p1_combo := 0
var _p2_combo := 0
var _p1_combo_window := 0.0
var _p2_combo_window := 0.0

func _ready() -> void:
	_projectile_scene = load("res://scenes/projectile.tscn") as PackedScene

	# Load the procedural .scad dojo arena
	var arena_scene := load("res://assets/models/dojo_arena.scad") as PackedScene
	if arena_scene:
		var arena := arena_scene.instantiate()
		arena_holder.add_child(arena)
		var anim := _find_animation_player(arena)
		if anim and anim.has_animation("ambient"):
			anim.get_animation("ambient").loop_mode = Animation.LOOP_LINEAR
			anim.play("ambient")

	# Setup fighters
	fighter_p1.fighter_name = "CYBER STRIKER"
	fighter_p1.is_player_one = true
	fighter_p1.is_ai = false
	fighter_p1.accent_color = Color(0.15, 0.85, 1.0)
	fighter_p1.opponent = fighter_p2

	fighter_p2.fighter_name = "CRIMSON TITAN"
	fighter_p2.is_player_one = false
	fighter_p2.is_ai = true
	fighter_p2.accent_color = Color(1.0, 0.35, 0.1)
	fighter_p2.load_fighter_model("res://assets/models/fighter_titan.scad")
	fighter_p2.opponent = fighter_p1

	fighting_camera.fighter_a = fighter_p1
	fighting_camera.fighter_b = fighter_p2

	fighter_p1.health_changed.connect(_on_health_changed)
	fighter_p2.health_changed.connect(_on_health_changed)
	fighter_p1.meter_changed.connect(_on_meter_changed)
	fighter_p2.meter_changed.connect(_on_meter_changed)
	fighter_p1.hit_landed.connect(_on_hit_landed)
	fighter_p2.hit_landed.connect(_on_hit_landed)
	fighter_p1.fighter_ko.connect(_on_fighter_ko)
	fighter_p2.fighter_ko.connect(_on_fighter_ko)

	hud.start_1p_requested.connect(_on_start_1p)
	hud.start_2p_requested.connect(_on_start_2p)
	hud.rematch_requested.connect(_on_rematch)
	hud.menu_requested.connect(_on_return_menu)

	hud.set_fighter_names(str(fighter_p1.fighter_name), str(fighter_p2.fighter_name))
	hud.update_round_pips(0, 0)
	hud.update_timer(99)
	hud.show_title()

func _process(delta: float) -> void:
	if _p1_combo_window > 0.0:
		_p1_combo_window -= delta
		if _p1_combo_window <= 0.0:
			_p1_combo = 0

	if _p2_combo_window > 0.0:
		_p2_combo_window -= delta
		if _p2_combo_window <= 0.0:
			_p2_combo = 0

	if _match_state == MatchState.FIGHTING:
		_round_timer -= delta
		hud.update_timer(ceili(_round_timer))
		if _round_timer <= 0.0:
			_round_timer = 0.0
			_handle_time_over()

	if Input.is_action_just_pressed("ui_cancel") and _match_state != MatchState.MENU:
		_on_return_menu()

func _on_start_1p() -> void:
	sound_fx.play_ui_click()
	_two_player_mode = false
	fighter_p2.is_ai = true
	fighter_p2.fighter_name = "CRIMSON TITAN (CPU)"
	_start_new_match()

func _on_start_2p() -> void:
	sound_fx.play_ui_click()
	_two_player_mode = true
	fighter_p2.is_ai = false
	fighter_p2.fighter_name = "CRIMSON TITAN (P2)"
	_start_new_match()

func _on_rematch() -> void:
	sound_fx.play_ui_click()
	_start_new_match()

func _on_return_menu() -> void:
	sound_fx.play_ui_click()
	_match_state = MatchState.MENU
	fighter_p1.input_enabled = false
	fighter_p2.input_enabled = false
	_clear_effects()
	fighter_p1.reset_for_round(-2.6, 1.0)
	fighter_p2.reset_for_round(2.6, -1.0)
	hud.set_gameplay_active(false, _two_player_mode)
	hud.show_title()

func _start_new_match() -> void:
	_round_number = 1
	_p1_wins = 0
	_p2_wins = 0
	fighter_p1.super_meter = 20.0
	fighter_p2.super_meter = 20.0
	hud.set_fighter_names(str(fighter_p1.fighter_name), str(fighter_p2.fighter_name))
	hud.update_round_pips(0, 0)
	hud.hide_menus()
	_begin_round()

func _begin_round() -> void:
	_match_state = MatchState.ROUND_INTRO
	_round_timer = 99.0
	_p1_combo = 0
	_p2_combo = 0
	_clear_effects()
	fighter_p1.input_enabled = false
	fighter_p2.input_enabled = false
	fighter_p1.reset_for_round(-2.6, 1.0)
	fighter_p2.reset_for_round(2.6, -1.0)
	hud.update_timer(99)
	hud.set_gameplay_active(true, _two_player_mode)

	sound_fx.play_round_announce()
	hud.announce("ROUND %d" % _round_number, 1.1)
	await get_tree().create_timer(1.2).timeout
	if _match_state != MatchState.ROUND_INTRO:
		return
	sound_fx.play_fight_start()
	hud.announce("FIGHT!", 0.9)
	_match_state = MatchState.FIGHTING
	fighter_p1.input_enabled = true
	fighter_p2.input_enabled = true

func spawn_projectile(p_owner: Node3D, p_opponent: Node3D, pos: Vector3, dir: float, tint: Color) -> void:
	if not _projectile_scene:
		return
	var proj := _projectile_scene.instantiate()
	effects_holder.add_child(proj)
	proj.global_position = pos
	proj.setup(p_owner, p_opponent, dir, tint)
	proj.hit_fighter.connect(func(target, dmg, kb, heavy):
		if is_instance_valid(target) and is_instance_valid(p_owner):
			target.receive_hit(p_owner, dmg, kb, heavy, target.global_position + Vector3(0.0, 1.0, 0.0))
	)

func _on_health_changed(fighter: Node3D, new_health: float) -> void:
	hud.update_health(fighter == fighter_p1, new_health)

func _on_meter_changed(fighter: Node3D, new_meter: float) -> void:
	hud.update_meter(fighter == fighter_p1, new_meter)

func _on_hit_landed(attacker: Node3D, defender: Node3D, _damage: float, blocked: bool, is_heavy: bool, hit_pos: Vector3) -> void:
	var hit_dir := 1.0
	if is_instance_valid(attacker) and is_instance_valid(defender):
		hit_dir = 1.0 if defender.global_position.x >= attacker.global_position.x else -1.0

	_spawn_hit_spark(hit_pos, blocked, is_heavy, hit_dir)
	if blocked:
		sound_fx.play_block(is_heavy)
		fighting_camera.add_shake(0.06 if is_heavy else 0.04)
	else:
		if is_heavy:
			sound_fx.play_hit_heavy()
			fighting_camera.add_shake(0.14)
		else:
			sound_fx.play_hit_light()
			fighting_camera.add_shake(0.06)

		if attacker == fighter_p1:
			_p1_combo += 1
			_p1_combo_window = 1.25
			hud.show_combo(true, _p1_combo)
		else:
			_p2_combo += 1
			_p2_combo_window = 1.25
			hud.show_combo(false, _p2_combo)

func _on_fighter_ko(ko_fighter: Node3D) -> void:
	if _match_state != MatchState.FIGHTING:
		return
	_match_state = MatchState.ROUND_OVER
	fighter_p1.input_enabled = false
	fighter_p2.input_enabled = false
	sound_fx.play_ko()
	fighting_camera.add_shake(0.28)
	_spawn_hit_spark(ko_fighter.global_position + Vector3(0.0, 1.1, 0.0), false, true, -ko_fighter.facing_dir, true)

	var p1_won_round: bool = (ko_fighter == fighter_p2)
	if p1_won_round:
		_p1_wins += 1
		fighter_p1.set_victory()
		hud.announce("PERFECT!" if float(fighter_p1.health) >= float(fighter_p1.max_health) else "K.O.!", 1.8)
	else:
		_p2_wins += 1
		fighter_p2.set_victory()
		hud.announce("PERFECT!" if float(fighter_p2.health) >= float(fighter_p2.max_health) else "K.O.!", 1.8)

	hud.update_round_pips(_p1_wins, _p2_wins)
	await get_tree().create_timer(2.3).timeout
	if _match_state != MatchState.ROUND_OVER:
		return
	_advance_round_or_finish()

func _handle_time_over() -> void:
	_match_state = MatchState.ROUND_OVER
	fighter_p1.input_enabled = false
	fighter_p2.input_enabled = false
	sound_fx.play_round_announce()
	hud.announce("TIME OVER!", 1.8)

	if float(fighter_p1.health) >= float(fighter_p2.health):
		_p1_wins += 1
		fighter_p1.set_victory()
	else:
		_p2_wins += 1
		fighter_p2.set_victory()

	hud.update_round_pips(_p1_wins, _p2_wins)
	await get_tree().create_timer(2.0).timeout
	if _match_state != MatchState.ROUND_OVER:
		return
	_advance_round_or_finish()

func _advance_round_or_finish() -> void:
	if _p1_wins >= 2 or _p2_wins >= 2:
		_match_state = MatchState.MATCH_OVER
		var winner_name: String = str(fighter_p1.fighter_name) if _p1_wins >= 2 else str(fighter_p2.fighter_name)
		sound_fx.play_victory()
		hud.set_gameplay_active(false, _two_player_mode)
		hud.show_result("%s WINS!" % winner_name)
	else:
		_round_number += 1
		_begin_round()

func _spawn_hit_spark(pos: Vector3, blocked: bool, is_heavy: bool, hit_dir := 1.0, is_ko := false) -> void:
	var effect_root := Node3D.new()
	effects_holder.add_child(effect_root)
	effect_root.global_position = pos + Vector3(0.0, 0.0, 0.22)

	var core_color: Color
	var mid_color: Color
	var light_color: Color
	if blocked:
		core_color = Color(1.6, 2.2, 2.6, 1.0)
		mid_color = Color(0.2, 0.85, 1.0, 0.9)
		light_color = Color(0.25, 0.85, 1.0)
	elif is_ko or is_heavy:
		core_color = Color(2.6, 2.1, 1.2, 1.0)
		mid_color = Color(1.8, 0.55, 0.12, 0.95)
		light_color = Color(1.0, 0.5, 0.15)
	else:
		core_color = Color(2.4, 2.2, 1.3, 1.0)
		mid_color = Color(1.6, 1.15, 0.25, 0.9)
		light_color = Color(1.0, 0.85, 0.3)

	var scale_curve := Curve.new()
	scale_curve.add_point(Vector2(0.0, 1.0))
	scale_curve.add_point(Vector2(0.45, 0.65))
	scale_curve.add_point(Vector2(1.0, 0.0))

	var spark_grad := Gradient.new()
	spark_grad.offsets = PackedFloat32Array([0.0, 0.35, 1.0])
	spark_grad.colors = PackedColorArray([core_color, mid_color, Color(mid_color.r, mid_color.g, mid_color.b, 0.0)])

	# 1. High-velocity directional spark streaks
	var streaks := CPUParticles3D.new()
	streaks.emitting = false
	streaks.local_coords = true
	streaks.one_shot = true
	streaks.explosiveness = 1.0
	streaks.amount = 28 if is_ko else (20 if is_heavy else 12)
	streaks.lifetime = 0.28 if is_heavy else 0.20
	streaks.particle_flag_align_y = true
	streaks.direction = Vector3(0.0, 0.15, 0.15) if blocked else Vector3(hit_dir * 0.85, 0.25, 0.15)
	streaks.spread = 85.0 if blocked else 68.0
	streaks.initial_velocity_min = 5.0 if is_heavy else 4.0
	streaks.initial_velocity_max = 12.0 if is_ko else (9.5 if is_heavy else 6.8)
	streaks.damping_min = 8.0
	streaks.damping_max = 14.0
	streaks.gravity = Vector3(0.0, -2.5, 0.0)
	streaks.scale_amount_min = 0.85
	streaks.scale_amount_max = 1.45 if is_heavy else 1.15
	streaks.scale_amount_curve = scale_curve
	streaks.color_ramp = spark_grad

	var streak_mesh := BoxMesh.new()
	streak_mesh.size = Vector3(0.028, 0.24 if is_heavy else 0.17, 0.028)
	var streak_mat := StandardMaterial3D.new()
	streak_mat.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	streak_mat.transparency = BaseMaterial3D.TRANSPARENCY_ALPHA
	streak_mat.blend_mode = BaseMaterial3D.BLEND_MODE_ADD
	streak_mat.vertex_color_use_as_albedo = true
	streak_mesh.material = streak_mat
	streaks.mesh = streak_mesh

	effect_root.add_child(streaks)
	streaks.restart()
	streaks.emitting = true

	# 2. Secondary glowing energy motes
	var motes := CPUParticles3D.new()
	motes.emitting = false
	motes.local_coords = true
	motes.one_shot = true
	motes.explosiveness = 0.95
	motes.amount = 14 if is_heavy else 8
	motes.lifetime = 0.38 if is_heavy else 0.28
	motes.emission_shape = CPUParticles3D.EMISSION_SHAPE_SPHERE
	motes.emission_sphere_radius = 0.12
	motes.direction = Vector3(hit_dir * 0.4, 0.5, 0.2)
	motes.spread = 180.0
	motes.initial_velocity_min = 2.0
	motes.initial_velocity_max = 5.5 if is_heavy else 3.8
	motes.damping_min = 4.5
	motes.damping_max = 7.5
	motes.gravity = Vector3(0.0, 1.2, 0.0)
	motes.scale_amount_min = 0.7
	motes.scale_amount_max = 1.3
	motes.scale_amount_curve = scale_curve
	motes.color_ramp = spark_grad

	var mote_mesh := SphereMesh.new()
	mote_mesh.radius = 0.042 if is_heavy else 0.032
	mote_mesh.height = 0.084 if is_heavy else 0.064
	var mote_mat := StandardMaterial3D.new()
	mote_mat.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	mote_mat.transparency = BaseMaterial3D.TRANSPARENCY_ALPHA
	mote_mat.blend_mode = BaseMaterial3D.BLEND_MODE_ADD
	mote_mat.vertex_color_use_as_albedo = true
	mote_mesh.material = mote_mat
	motes.mesh = mote_mesh

	effect_root.add_child(motes)
	motes.restart()
	motes.emitting = true

	# 3. Impact core flash sphere
	var flash := MeshInstance3D.new()
	var flash_mesh := SphereMesh.new()
	flash_mesh.radius = 0.22
	flash_mesh.height = 0.44
	var flash_mat := StandardMaterial3D.new()
	flash_mat.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	flash_mat.transparency = BaseMaterial3D.TRANSPARENCY_ALPHA
	flash_mat.blend_mode = BaseMaterial3D.BLEND_MODE_ADD
	flash_mat.albedo_color = core_color
	flash_mesh.material = flash_mat
	flash.mesh = flash_mesh
	flash.scale = Vector3.ONE * 0.35
	effect_root.add_child(flash)

	# 4. Expanding shockwave / guard ring
	var ring := MeshInstance3D.new()
	var ring_mesh := TorusMesh.new()
	ring_mesh.inner_radius = 0.34
	ring_mesh.outer_radius = 0.42
	ring_mesh.rings = 24
	ring_mesh.ring_segments = 10
	var ring_mat := StandardMaterial3D.new()
	ring_mat.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	ring_mat.transparency = BaseMaterial3D.TRANSPARENCY_ALPHA
	ring_mat.blend_mode = BaseMaterial3D.BLEND_MODE_ADD
	ring_mat.albedo_color = mid_color
	ring_mesh.material = ring_mat
	ring.mesh = ring_mesh
	ring.rotation_degrees.x = 90.0
	ring.scale = Vector3.ONE * 0.35
	effect_root.add_child(ring)

	# 5. Brief impact OmniLight3D flash
	var flash_light := OmniLight3D.new()
	flash_light.light_color = light_color
	flash_light.light_energy = 4.5 if is_ko else (3.0 if is_heavy else 1.8)
	flash_light.omni_range = 3.2 if is_heavy else 2.2
	effect_root.add_child(flash_light)

	var flash_end_scale := 2.4 if is_ko else (1.55 if is_heavy else 1.05)
	var ring_end_scale := 2.8 if is_ko else (1.85 if is_heavy else 1.25)
	var ring_dur := 0.26 if is_heavy else 0.18

	var tween := effect_root.create_tween()
	tween.set_parallel(true)
	tween.tween_property(flash, "scale", Vector3.ONE * flash_end_scale, 0.11).set_trans(Tween.TRANS_QUAD).set_ease(Tween.EASE_OUT)
	tween.tween_property(flash_mat, "albedo_color:a", 0.0, 0.12)
	tween.tween_property(ring, "scale", Vector3.ONE * ring_end_scale, ring_dur).set_trans(Tween.TRANS_CUBIC).set_ease(Tween.EASE_OUT)
	tween.tween_property(ring_mat, "albedo_color:a", 0.0, ring_dur)
	tween.tween_property(flash_light, "light_energy", 0.0, 0.16)

	get_tree().create_timer(0.55).timeout.connect(func():
		if is_instance_valid(effect_root):
			effect_root.queue_free()
	)

func _clear_effects() -> void:
	for child in effects_holder.get_children():
		child.queue_free()

func _find_animation_player(root: Node) -> AnimationPlayer:
	if root is AnimationPlayer:
		return root
	for child in root.get_children():
		var found := _find_animation_player(child)
		if found:
			return found
	return null
```

### cyber_dojo_fighter/scenes/projectile.tscn
```text
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/projectile.gd" id="1_proj"]

[node name="Projectile" type="Node3D"]
script = ExtResource("1_proj")

[node name="GlowLight" type="OmniLight3D" parent="."]
light_color = Color(0.2, 0.9, 1, 1)
light_energy = 2.5
omni_range = 3.5

```

### cyber_dojo_fighter/scenes/fighter.tscn
```text
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/fighter.gd" id="1_fighter"]

[node name="Fighter" type="Node3D"]
script = ExtResource("1_fighter")

```

### cyber_dojo_fighter/scenes/hud.tscn
```text
[gd_scene load_steps=8 format=3]

[ext_resource type="Script" path="res://scripts/hud.gd" id="1_hud"]
[ext_resource type="Script" path="res://scripts/touch_controls.gd" id="2_touch"]

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_bg"]
bg_color = Color(0.08, 0.1, 0.15, 0.85)
border_width_left = 2
border_width_top = 2
border_width_right = 2
border_width_bottom = 2
border_color = Color(0.25, 0.7, 0.95, 0.8)
corner_radius_top_left = 4
corner_radius_top_right = 4
corner_radius_bottom_right = 4
corner_radius_bottom_left = 4

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_p1_hp"]
bg_color = Color(0.15, 0.78, 0.98, 1)
corner_radius_top_left = 3
corner_radius_top_right = 3
corner_radius_bottom_right = 3
corner_radius_bottom_left = 3

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_trail"]
bg_color = Color(0.96, 0.82, 0.2, 0.9)

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_p2_hp"]
bg_color = Color(0.95, 0.25, 0.22, 1)
corner_radius_top_left = 3
corner_radius_top_right = 3
corner_radius_bottom_right = 3
corner_radius_bottom_left = 3

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_meter"]
bg_color = Color(0.95, 0.72, 0.12, 1)
corner_radius_top_left = 3
corner_radius_top_right = 3
corner_radius_bottom_right = 3
corner_radius_bottom_left = 3

[node name="HUD" type="CanvasLayer"]
script = ExtResource("1_hud")

[node name="TopHUD" type="HBoxContainer" parent="."]
anchors_preset = 10
anchor_right = 1.0
offset_left = 28.0
offset_top = 20.0
offset_right = -28.0
offset_bottom = 96.0
grow_horizontal = 2
theme_override_constants/separation = 16

[node name="P1Box" type="VBoxContainer" parent="TopHUD"]
layout_mode = 2
size_flags_horizontal = 3
theme_override_constants/separation = 6

[node name="P1Header" type="HBoxContainer" parent="TopHUD/P1Box"]
layout_mode = 2

[node name="P1Name" type="Label" parent="TopHUD/P1Box/P1Header"]
layout_mode = 2
size_flags_horizontal = 3
theme_override_colors/font_color = Color(0.75, 0.94, 1, 1)
theme_override_font_sizes/font_size = 18
text = "CYBER STRIKER"

[node name="P1Pips" type="HBoxContainer" parent="TopHUD/P1Box/P1Header"]
layout_mode = 2
theme_override_constants/separation = 6

[node name="Pip1" type="ColorRect" parent="TopHUD/P1Box/P1Header/P1Pips"]
custom_minimum_size = Vector2(16, 16)
layout_mode = 2
color = Color(0.25, 0.28, 0.35, 1)

[node name="Pip2" type="ColorRect" parent="TopHUD/P1Box/P1Header/P1Pips"]
custom_minimum_size = Vector2(16, 16)
layout_mode = 2
color = Color(0.25, 0.28, 0.35, 1)

[node name="P1HealthBar" type="ProgressBar" parent="TopHUD/P1Box"]
custom_minimum_size = Vector2(0, 28)
layout_mode = 2
theme_override_styles/background = SubResource("StyleBoxFlat_bg")
theme_override_styles/fill = SubResource("StyleBoxFlat_p1_hp")
value = 100.0
fill_mode = 1
show_percentage = false

[node name="P1TrailBar" type="ProgressBar" parent="TopHUD/P1Box/P1HealthBar"]
show_behind_parent = true
layout_mode = 1
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
theme_override_styles/fill = SubResource("StyleBoxFlat_trail")
value = 100.0
fill_mode = 1
show_percentage = false

[node name="TimerPanel" type="PanelContainer" parent="TopHUD"]
custom_minimum_size = Vector2(88, 72)
layout_mode = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_bg")

[node name="TimerLabel" type="Label" parent="TopHUD/TimerPanel"]
layout_mode = 2
theme_override_colors/font_color = Color(1, 0.95, 0.72, 1)
theme_override_font_sizes/font_size = 38
text = "99"
horizontal_alignment = 1
vertical_alignment = 1

[node name="P2Box" type="VBoxContainer" parent="TopHUD"]
layout_mode = 2
size_flags_horizontal = 3
theme_override_constants/separation = 6

[node name="P2Header" type="HBoxContainer" parent="TopHUD/P2Box"]
layout_mode = 2

[node name="P2Pips" type="HBoxContainer" parent="TopHUD/P2Box/P2Header"]
layout_mode = 2
theme_override_constants/separation = 6

[node name="Pip1" type="ColorRect" parent="TopHUD/P2Box/P2Header/P2Pips"]
custom_minimum_size = Vector2(16, 16)
layout_mode = 2
color = Color(0.25, 0.28, 0.35, 1)

[node name="Pip2" type="ColorRect" parent="TopHUD/P2Box/P2Header/P2Pips"]
custom_minimum_size = Vector2(16, 16)
layout_mode = 2
color = Color(0.25, 0.28, 0.35, 1)

[node name="P2Name" type="Label" parent="TopHUD/P2Box/P2Header"]
layout_mode = 2
size_flags_horizontal = 3
theme_override_colors/font_color = Color(1, 0.78, 0.72, 1)
theme_override_font_sizes/font_size = 18
text = "CRIMSON TITAN"
horizontal_alignment = 2

[node name="P2HealthBar" type="ProgressBar" parent="TopHUD/P2Box"]
custom_minimum_size = Vector2(0, 28)
layout_mode = 2
theme_override_styles/background = SubResource("StyleBoxFlat_bg")
theme_override_styles/fill = SubResource("StyleBoxFlat_p2_hp")
value = 100.0
show_percentage = false

[node name="P2TrailBar" type="ProgressBar" parent="TopHUD/P2Box/P2HealthBar"]
show_behind_parent = true
layout_mode = 1
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
theme_override_styles/fill = SubResource("StyleBoxFlat_trail")
value = 100.0
show_percentage = false

[node name="BottomHUD" type="HBoxContainer" parent="."]
anchors_preset = 12
anchor_top = 1.0
anchor_right = 1.0
anchor_bottom = 1.0
offset_left = 28.0
offset_top = -62.0
offset_right = -28.0
offset_bottom = -18.0
grow_horizontal = 2
grow_vertical = 0
theme_override_constants/separation = 160

[node name="P1MeterBox" type="VBoxContainer" parent="BottomHUD"]
layout_mode = 2
size_flags_horizontal = 3

[node name="P1MeterLabel" type="Label" parent="BottomHUD/P1MeterBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 13
text = "SUPER METER (H: ENERGY WAVE at 40+)"

[node name="P1MeterBar" type="ProgressBar" parent="BottomHUD/P1MeterBox"]
custom_minimum_size = Vector2(0, 16)
layout_mode = 2
theme_override_styles/background = SubResource("StyleBoxFlat_bg")
theme_override_styles/fill = SubResource("StyleBoxFlat_meter")
value = 20.0
show_percentage = false

[node name="P2MeterBox" type="VBoxContainer" parent="BottomHUD"]
layout_mode = 2
size_flags_horizontal = 3

[node name="P2MeterLabel" type="Label" parent="BottomHUD/P2MeterBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 13
text = "SUPER METER (L: ENERGY WAVE at 40+)"
horizontal_alignment = 2

[node name="P2MeterBar" type="ProgressBar" parent="BottomHUD/P2MeterBox"]
custom_minimum_size = Vector2(0, 16)
layout_mode = 2
theme_override_styles/background = SubResource("StyleBoxFlat_bg")
theme_override_styles/fill = SubResource("StyleBoxFlat_meter")
value = 20.0
fill_mode = 1
show_percentage = false

[node name="P1ComboLabel" type="Label" parent="."]
offset_left = 40.0
offset_top = 140.0
offset_right = 340.0
offset_bottom = 185.0
theme_override_colors/font_color = Color(0.3, 0.92, 1, 1)
theme_override_colors/font_outline_color = Color(0, 0, 0, 1)
theme_override_constants/outline_size = 6
theme_override_font_sizes/font_size = 28
text = "2 HIT COMBO!"

[node name="P2ComboLabel" type="Label" parent="."]
anchors_preset = 1
anchor_left = 1.0
anchor_right = 1.0
offset_left = -340.0
offset_top = 140.0
offset_right = -40.0
offset_bottom = 185.0
grow_horizontal = 0
theme_override_colors/font_color = Color(1, 0.52, 0.2, 1)
theme_override_colors/font_outline_color = Color(0, 0, 0, 1)
theme_override_constants/outline_size = 6
theme_override_font_sizes/font_size = 28
text = "2 HIT COMBO!"
horizontal_alignment = 2

[node name="AnnouncerLabel" type="Label" parent="."]
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -320.0
offset_top = -60.0
offset_right = 320.0
offset_bottom = 60.0
grow_horizontal = 2
grow_vertical = 2
theme_override_colors/font_color = Color(1, 0.88, 0.25, 1)
theme_override_colors/font_outline_color = Color(0.05, 0.05, 0.1, 1)
theme_override_constants/outline_size = 10
theme_override_font_sizes/font_size = 64
text = "FIGHT!"
horizontal_alignment = 1
vertical_alignment = 1

[node name="TitlePanel" type="PanelContainer" parent="."]
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -270.0
offset_top = -180.0
offset_right = 270.0
offset_bottom = 180.0
grow_horizontal = 2
grow_vertical = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_bg")

[node name="TitleVBox" type="VBoxContainer" parent="TitlePanel"]
layout_mode = 2
theme_override_constants/separation = 14
alignment = 1

[node name="GameTitle" type="Label" parent="TitlePanel/TitleVBox"]
layout_mode = 2
theme_override_colors/font_color = Color(0.35, 0.9, 1, 1)
theme_override_font_sizes/font_size = 38
text = "CYBER DOJO FIGHTER"
horizontal_alignment = 1

[node name="ControlsInfo" type="Label" parent="TitlePanel/TitleVBox"]
layout_mode = 2
theme_override_colors/font_color = Color(0.85, 0.88, 0.95, 1)
theme_override_font_sizes/font_size = 14
text = "P1: A/D Move | W Jump | S Crouch/Block | F Punch | G Kick | H Special
P2: Arrows Move/Jump/Crouch | J Punch | K Kick | L Special
Touch: Virtual Stick Move/Jump/Crouch | Action Buttons Attack
Hold Back or Crouch to Guard Incoming Strikes!"
horizontal_alignment = 1

[node name="Btn1P" type="Button" parent="TitlePanel/TitleVBox"]
custom_minimum_size = Vector2(280, 48)
layout_mode = 2
size_flags_horizontal = 4
theme_override_font_sizes/font_size = 20
text = "1P VS CPU"

[node name="Btn2P" type="Button" parent="TitlePanel/TitleVBox"]
custom_minimum_size = Vector2(280, 48)
layout_mode = 2
size_flags_horizontal = 4
theme_override_font_sizes/font_size = 20
text = "2P LOCAL VERSUS"

[node name="ResultPanel" type="PanelContainer" parent="."]
visible = false
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -220.0
offset_top = -120.0
offset_right = 220.0
offset_bottom = 120.0
grow_horizontal = 2
grow_vertical = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_bg")

[node name="ResultVBox" type="VBoxContainer" parent="ResultPanel"]
layout_mode = 2
theme_override_constants/separation = 14
alignment = 1

[node name="WinnerLabel" type="Label" parent="ResultPanel/ResultVBox"]
layout_mode = 2
theme_override_colors/font_color = Color(1, 0.88, 0.25, 1)
theme_override_font_sizes/font_size = 30
text = "CYBER STRIKER WINS!"
horizontal_alignment = 1

[node name="BtnRematch" type="Button" parent="ResultPanel/ResultVBox"]
custom_minimum_size = Vector2(220, 44)
layout_mode = 2
size_flags_horizontal = 4
theme_override_font_sizes/font_size = 18
text = "REMATCH"

[node name="BtnMenu" type="Button" parent="ResultPanel/ResultVBox"]
custom_minimum_size = Vector2(220, 44)
layout_mode = 2
size_flags_horizontal = 4
theme_override_font_sizes/font_size = 18
text = "MAIN MENU"

[node name="TouchControls" type="Control" parent="."]
visible = false
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
mouse_filter = 2
script = ExtResource("2_touch")
```

### cyber_dojo_fighter/scenes/main.tscn
```text
[gd_scene load_steps=9 format=3]

[ext_resource type="Script" path="res://scripts/game_manager.gd" id="1_gm"]
[ext_resource type="Script" path="res://scripts/fighting_camera.gd" id="2_cam"]
[ext_resource type="PackedScene" path="res://scenes/fighter.tscn" id="3_fighter"]
[ext_resource type="PackedScene" path="res://scenes/hud.tscn" id="4_hud"]
[ext_resource type="Script" path="res://scripts/sound_fx.gd" id="5_sfx"]

[sub_resource type="ProceduralSkyMaterial" id="ProceduralSkyMaterial_dojo"]
sky_top_color = Color(0.05, 0.08, 0.16, 1)
sky_horizon_color = Color(0.18, 0.12, 0.28, 1)
ground_bottom_color = Color(0.04, 0.04, 0.08, 1)
ground_horizon_color = Color(0.14, 0.1, 0.2, 1)

[sub_resource type="Sky" id="Sky_dojo"]
sky_material = SubResource("ProceduralSkyMaterial_dojo")

[sub_resource type="Environment" id="Environment_dojo"]
background_mode = 2
sky = SubResource("Sky_dojo")
ambient_light_source = 2
ambient_light_color = Color(0.42, 0.48, 0.65, 1)
ambient_light_energy = 0.9
glow_enabled = true
glow_intensity = 0.75
glow_bloom = 0.25

[node name="Main" type="Node3D"]
script = ExtResource("1_gm")

[node name="WorldEnvironment" type="WorldEnvironment" parent="."]
environment = SubResource("Environment_dojo")

[node name="KeyLight" type="DirectionalLight3D" parent="."]
transform = Transform3D(0.866025, -0.353553, 0.353553, 0, 0.707107, 0.707107, -0.5, -0.612372, 0.612372, 4, 8, 6)
light_color = Color(1, 0.96, 0.9, 1)
light_energy = 1.6
shadow_enabled = true

[node name="RimLight" type="DirectionalLight3D" parent="."]
transform = Transform3D(-0.866025, 0.25, -0.433013, 0, 0.866025, 0.5, 0.5, 0.433013, -0.75, -4, 5, -6)
light_color = Color(0.25, 0.75, 1, 1)
light_energy = 0.9

[node name="FightingCamera" type="Camera3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 0.990268, 0.139173, 0, -0.139173, 0.990268, 0, 2.0, 8.2)
fov = 48.0
script = ExtResource("2_cam")

[node name="ArenaHolder" type="Node3D" parent="."]

[node name="EffectsHolder" type="Node3D" parent="."]

[node name="FighterP1" parent="." instance=ExtResource("3_fighter")]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, -2.6, 0, 0)

[node name="FighterP2" parent="." instance=ExtResource("3_fighter")]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 2.6, 0, 0)

[node name="HUD" parent="." instance=ExtResource("4_hud")]

[node name="SoundFX" type="Node" parent="."]
script = ExtResource("5_sfx")

```

### cyber_dojo_fighter/scripts/touch_controls.gd
```gdscript
extends Control

signal menu_pressed

class VirtualStick extends Control:
	var action_left: StringName
	var action_right: StringName
	var action_jump: StringName
	var action_crouch: StringName
	var accent_color := Color(0.25, 0.88, 1.0)
	var ring_radius := 72.0
	var knob_radius := 28.0
	var active := false
	var knob_offset := Vector2.ZERO
	var dir_flags := Vector4i.ZERO # x=left, y=right, z=jump, w=crouch

	func _init(left_act: StringName, right_act: StringName, jump_act: StringName, crouch_act: StringName, color: Color) -> void:
		action_left = left_act
		action_right = right_act
		action_jump = jump_act
		action_crouch = crouch_act
		accent_color = color
		mouse_filter = Control.MOUSE_FILTER_IGNORE

	func get_center_local() -> Vector2:
		return size * 0.5

	func get_center_global() -> Vector2:
		return get_global_rect().get_center()

	func _draw() -> void:
		var c := get_center_local()
		var base_alpha := 0.34 if active else 0.22
		draw_circle(c, ring_radius, Color(0.06, 0.09, 0.14, base_alpha))
		draw_arc(c, ring_radius, 0.0, TAU, 48, Color(accent_color.r, accent_color.g, accent_color.b, 0.65 if active else 0.38), 2.5, true)
		draw_arc(c, ring_radius * 0.42, 0.0, TAU, 32, Color(1.0, 1.0, 1.0, 0.10), 1.5, true)

		var offset_lbl := ring_radius * 0.68
		var arrow_size := ring_radius * 0.13
		_draw_dir_arrow(c + Vector2.LEFT * offset_lbl, Vector2.LEFT, arrow_size, dir_flags.x == 1)
		_draw_dir_arrow(c + Vector2.RIGHT * offset_lbl, Vector2.RIGHT, arrow_size, dir_flags.y == 1)
		_draw_dir_arrow(c + Vector2.UP * offset_lbl, Vector2.UP, arrow_size, dir_flags.z == 1)
		_draw_dir_arrow(c + Vector2.DOWN * offset_lbl, Vector2.DOWN, arrow_size, dir_flags.w == 1)

		var knob_pos := c + knob_offset
		var knob_col := accent_color if active else Color(0.75, 0.85, 0.95)
		draw_circle(knob_pos, knob_radius, Color(knob_col.r, knob_col.g, knob_col.b, 0.55 if active else 0.32))
		draw_arc(knob_pos, knob_radius, 0.0, TAU, 32, Color(knob_col.r, knob_col.g, knob_col.b, 0.95 if active else 0.65), 2.5, true)

	func _draw_dir_arrow(center_pos: Vector2, dir: Vector2, arrow_size: float, highlighted: bool) -> void:
		var col := accent_color if highlighted else Color(0.85, 0.9, 0.98, 0.42)
		var perp := Vector2(-dir.y, dir.x)
		var tip := center_pos + dir * arrow_size
		var base_l := center_pos - dir * (arrow_size * 0.7) + perp * (arrow_size * 0.85)
		var base_r := center_pos - dir * (arrow_size * 0.7) - perp * (arrow_size * 0.85)
		draw_colored_polygon(PackedVector2Array([tip, base_l, base_r]), col)


class TouchActionButton extends Control:
	var action_name: StringName
	var label_text: String
	var base_color: Color
	var is_pressed := false
	var is_highlighted := false

	func _init(p_action: StringName, p_label: String, p_color: Color) -> void:
		action_name = p_action
		label_text = p_label
		base_color = p_color
		mouse_filter = Control.MOUSE_FILTER_IGNORE

	func contains_global_point(pt: Vector2) -> bool:
		var rect := get_global_rect()
		var radius := minf(rect.size.x, rect.size.y) * 0.5
		return pt.distance_to(rect.get_center()) <= radius * 1.15

	func _draw() -> void:
		var c := size * 0.5
		var r := minf(size.x, size.y) * 0.48
		var col := base_color
		if is_highlighted:
			col = col.lightened(0.25)

		var fill_alpha := 0.55 if is_pressed else (0.34 if is_highlighted else 0.24)
		var border_alpha := 0.98 if is_pressed else (0.90 if is_highlighted else 0.62)
		var draw_r := r * (0.94 if is_pressed else 1.0)

		draw_circle(c, draw_r, Color(col.r * 0.45, col.g * 0.45, col.b * 0.45, fill_alpha))
		draw_arc(c, draw_r, 0.0, TAU, 40, Color(col.r, col.g, col.b, border_alpha), 3.0 if is_highlighted else 2.5, true)
		if is_highlighted and not is_pressed:
			draw_arc(c, draw_r * 0.84, 0.0, TAU, 36, Color(1.0, 0.95, 0.55, 0.55), 1.5, true)

		var font := ThemeDB.fallback_font
		var fsize := maxi(12, int(draw_r * 0.34))
		var text_col := Color(1.0, 1.0, 1.0, 0.98 if is_pressed or is_highlighted else 0.88)
		draw_string(font, Vector2(0.0, c.y + fsize * 0.35), label_text, HORIZONTAL_ALIGNMENT_CENTER, size.x, fsize, text_col)


var _gameplay_active := false
var _two_player_mode := false
var _touch_detected := false

var _p1_widgets: Array[Control] = []
var _p2_widgets: Array[Control] = []
var _sticks: Array[VirtualStick] = []
var _buttons: Array[TouchActionButton] = []

var _p1_stick: VirtualStick
var _p1_light: TouchActionButton
var _p1_heavy: TouchActionButton
var _p1_special: TouchActionButton

var _p2_stick: VirtualStick
var _p2_light: TouchActionButton
var _p2_heavy: TouchActionButton
var _p2_special: TouchActionButton

var _menu_btn: Button

var _stick_fingers: Dictionary = {} # finger_idx -> VirtualStick
var _button_fingers: Dictionary = {} # finger_idx -> TouchActionButton
var _held_actions: Dictionary = {} # StringName -> bool

func _ready() -> void:
	_touch_detected = DisplayServer.is_touchscreen_available()
	_build_ui()
	_apply_layout()
	_update_visibility()

func _notification(what: int) -> void:
	if what == NOTIFICATION_WM_WINDOW_FOCUS_OUT:
		_release_all_touch()

func _exit_tree() -> void:
	_release_all_touch()

func set_gameplay_active(active: bool, two_player: bool) -> void:
	_gameplay_active = active
	_two_player_mode = two_player
	if not _gameplay_active:
		_release_all_touch()
	_apply_layout()
	_update_visibility()

func set_special_ready(is_p1: bool, ready: bool) -> void:
	var btn := _p1_special if is_p1 else _p2_special
	if btn and btn.is_highlighted != ready:
		btn.is_highlighted = ready
		btn.queue_redraw()

func _build_ui() -> void:
	_p1_stick = VirtualStick.new(&"p1_left", &"p1_right", &"p1_jump", &"p1_crouch", Color(0.2, 0.85, 1.0))
	_p1_light = TouchActionButton.new(&"p1_light", "PUNCH", Color(0.2, 0.82, 1.0))
	_p1_heavy = TouchActionButton.new(&"p1_heavy", "KICK", Color(0.95, 0.55, 0.15))
	_p1_special = TouchActionButton.new(&"p1_special", "SPEC", Color(0.98, 0.82, 0.2))

	_p2_stick = VirtualStick.new(&"p2_left", &"p2_right", &"p2_jump", &"p2_crouch", Color(1.0, 0.4, 0.25))
	_p2_light = TouchActionButton.new(&"p2_light", "PUNCH", Color(1.0, 0.45, 0.28))
	_p2_heavy = TouchActionButton.new(&"p2_heavy", "KICK", Color(0.95, 0.25, 0.3))
	_p2_special = TouchActionButton.new(&"p2_special", "SPEC", Color(0.98, 0.82, 0.2))

	_p1_widgets = [_p1_stick, _p1_light, _p1_heavy, _p1_special]
	_p2_widgets = [_p2_stick, _p2_light, _p2_heavy, _p2_special]
	_sticks = [_p1_stick, _p2_stick]
	_buttons = [_p1_light, _p1_heavy, _p1_special, _p2_light, _p2_heavy, _p2_special]

	for w in _p1_widgets:
		add_child(w)
	for w in _p2_widgets:
		add_child(w)

	_menu_btn = Button.new()
	_menu_btn.text = "MENU"
	_menu_btn.focus_mode = Control.FOCUS_NONE
	_menu_btn.add_theme_font_size_override("font_size", 14)
	_menu_btn.anchor_left = 0.5
	_menu_btn.anchor_right = 0.5
	_menu_btn.anchor_top = 0.0
	_menu_btn.anchor_bottom = 0.0
	_menu_btn.offset_left = -46.0
	_menu_btn.offset_top = 98.0
	_menu_btn.offset_right = 46.0
	_menu_btn.offset_bottom = 128.0
	_menu_btn.pressed.connect(func(): emit_signal("menu_pressed"))
	add_child(_menu_btn)

func _apply_layout() -> void:
	if not _two_player_mode:
		_set_bottom_left_rect(_p1_stick, 24.0, -284.0, 220.0, 220.0)
		_p1_stick.ring_radius = 74.0
		_p1_stick.knob_radius = 28.0

		_set_bottom_right_rect(_p1_light, -290.0, -181.0, 92.0, 92.0)
		_set_bottom_right_rect(_p1_heavy, -194.0, -254.0, 96.0, 96.0)
		_set_bottom_right_rect(_p1_special, -122.0, -166.0, 96.0, 96.0)

		for w in _p2_widgets:
			w.visible = false
	else:
		_set_bottom_left_rect(_p1_stick, 16.0, -234.0, 166.0, 166.0)
		_p1_stick.ring_radius = 56.0
		_p1_stick.knob_radius = 22.0

		_set_bottom_left_rect(_p1_light, 192.0, -156.0, 76.0, 76.0)
		_set_bottom_left_rect(_p1_heavy, 265.0, -221.0, 78.0, 78.0)
		_set_bottom_left_rect(_p1_special, 329.0, -147.0, 78.0, 78.0)

		_set_bottom_right_rect(_p2_stick, -404.0, -234.0, 166.0, 166.0)
		_p2_stick.ring_radius = 56.0
		_p2_stick.knob_radius = 22.0

		_set_bottom_right_rect(_p2_light, -228.0, -156.0, 76.0, 76.0)
		_set_bottom_right_rect(_p2_heavy, -155.0, -221.0, 78.0, 78.0)
		_set_bottom_right_rect(_p2_special, -91.0, -147.0, 78.0, 78.0)

		for w in _p2_widgets:
			w.visible = true

	for w in _p1_widgets:
		w.visible = true
		w.queue_redraw()
	for w in _p2_widgets:
		w.queue_redraw()

func _set_bottom_left_rect(ctrl: Control, x: float, y: float, w: float, h: float) -> void:
	ctrl.anchor_left = 0.0
	ctrl.anchor_right = 0.0
	ctrl.anchor_top = 1.0
	ctrl.anchor_bottom = 1.0
	ctrl.offset_left = x
	ctrl.offset_top = y
	ctrl.offset_right = x + w
	ctrl.offset_bottom = y + h

func _set_bottom_right_rect(ctrl: Control, x: float, y: float, w: float, h: float) -> void:
	ctrl.anchor_left = 1.0
	ctrl.anchor_right = 1.0
	ctrl.anchor_top = 1.0
	ctrl.anchor_bottom = 1.0
	ctrl.offset_left = x
	ctrl.offset_top = y
	ctrl.offset_right = x + w
	ctrl.offset_bottom = y + h

func _update_visibility() -> void:
	visible = _gameplay_active and _touch_detected

func _input(event: InputEvent) -> void:
	if event is InputEventScreenTouch:
		if not _touch_detected:
			_touch_detected = true
			_update_visibility()

		if not visible:
			return

		var touch := event as InputEventScreenTouch
		if touch.pressed:
			if _menu_btn.visible and _menu_btn.get_global_rect().has_point(touch.position):
				return
			_handle_touch_down(touch.index, touch.position)
		else:
			_handle_touch_up(touch.index)

	elif event is InputEventScreenDrag:
		if not visible:
			return
		var drag := event as InputEventScreenDrag
		if _stick_fingers.has(drag.index):
			var stick: VirtualStick = _stick_fingers[drag.index]
			_update_stick_position(stick, drag.position)

func _handle_touch_down(finger_idx: int, pos: Vector2) -> void:
	for btn in _buttons:
		if btn.visible and btn.contains_global_point(pos):
			_button_fingers[finger_idx] = btn
			btn.is_pressed = true
			btn.queue_redraw()
			_set_action(btn.action_name, true)
			return

	for stick in _sticks:
		if stick.visible and stick.get_global_rect().has_point(pos):
			_stick_fingers[finger_idx] = stick
			stick.active = true
			_update_stick_position(stick, pos)
			return

func _handle_touch_up(finger_idx: int) -> void:
	if _button_fingers.has(finger_idx):
		var btn: TouchActionButton = _button_fingers[finger_idx]
		_button_fingers.erase(finger_idx)
		var still_held := false
		for other_btn in _button_fingers.values():
			if other_btn == btn:
				still_held = true
				break
		if not still_held:
			btn.is_pressed = false
			btn.queue_redraw()
			_set_action(btn.action_name, false)

	if _stick_fingers.has(finger_idx):
		var stick: VirtualStick = _stick_fingers[finger_idx]
		_stick_fingers.erase(finger_idx)
		var still_held := false
		for other_stick in _stick_fingers.values():
			if other_stick == stick:
				still_held = true
				break
		if not still_held:
			_reset_stick(stick)

func _update_stick_position(stick: VirtualStick, pos: Vector2) -> void:
	var delta := pos - stick.get_center_global()
	if delta.length() > stick.ring_radius:
		delta = delta.normalized() * stick.ring_radius
	stick.knob_offset = delta

	var h_dead := stick.ring_radius * 0.20
	var v_thresh := stick.ring_radius * 0.42
	var want_left := delta.x < -h_dead
	var want_right := delta.x > h_dead
	var want_jump := delta.y < -v_thresh
	var want_crouch := delta.y > v_thresh

	stick.dir_flags = Vector4i(int(want_left), int(want_right), int(want_jump), int(want_crouch))
	stick.queue_redraw()

	_set_action(stick.action_left, want_left)
	_set_action(stick.action_right, want_right)
	_set_action(stick.action_jump, want_jump)
	_set_action(stick.action_crouch, want_crouch)

func _reset_stick(stick: VirtualStick) -> void:
	stick.active = false
	stick.knob_offset = Vector2.ZERO
	stick.dir_flags = Vector4i.ZERO
	stick.queue_redraw()

	_set_action(stick.action_left, false)
	_set_action(stick.action_right, false)
	_set_action(stick.action_jump, false)
	_set_action(stick.action_crouch, false)

func _set_action(action: StringName, pressed: bool) -> void:
	var was_pressed := bool(_held_actions.get(action, false))
	if was_pressed == pressed:
		return
	_held_actions[action] = pressed
	if pressed:
		Input.action_press(action)
	else:
		Input.action_release(action)

func _release_all_touch() -> void:
	_stick_fingers.clear()
	_button_fingers.clear()
	for stick in _sticks:
		if stick:
			_reset_stick(stick)
	for btn in _buttons:
		if btn:
			btn.is_pressed = false
			btn.queue_redraw()
			_set_action(btn.action_name, false)
	for action in _held_actions.keys():
		if bool(_held_actions[action]):
			Input.action_release(action)
			_held_actions[action] = false
```
