### sokoban_3d/project.godot
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

config/name="Sokoban 3D"
run/main_scene="res://scenes/main.tscn"
config/features=PackedStringArray("4.2", "Forward Plus")

[display]

window/size/viewport_width=1280
window/size/viewport_height=720
window/stretch/mode="canvas_items"
window/stretch/aspect="keep"
window/handheld/orientation="landscape"

[editor_plugins]

enabled=PackedStringArray("res://addons/scad_importer/plugin.cfg")

[input]

move_up={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":87,"key_label":0,"unicode":119,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194320,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
move_down={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":83,"key_label":0,"unicode":115,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194322,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
move_left={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":65,"key_label":0,"unicode":97,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194319,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
move_right={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":68,"key_label":0,"unicode":100,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194321,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
undo={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":90,"key_label":0,"unicode":122,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194308,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
restart={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":82,"key_label":0,"unicode":114,"location":0,"echo":false,"script":null)
]
}
camera_toggle={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":67,"key_label":0,"unicode":99,"location":0,"echo":false,"script":null)
]
}
next_level={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":78,"key_label":0,"unicode":110,"location":0,"echo":false,"script":null)
]
}
prev_level={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":80,"key_label":0,"unicode":112,"location":0,"echo":false,"script":null)
]
}

[rendering]

anti_aliasing/quality/msaa_3d=2
```

### sokoban_3d/addons/scad_importer/plugin.cfg
```text
[plugin]

name="OpenSCAD GLTF Importer"
description="Imports .scad files directly as 3D scenes using scad-gltf"
author="scad-gltf"
version="0.2"
script="scad_plugin.gd"

```

### sokoban_3d/addons/scad_importer/scad_importer.gd
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

### sokoban_3d/addons/scad_importer/scad_plugin.gd
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

### sokoban_3d/assets/models/floor_tile.scad
```openscad
// Industrial warehouse floor tile: 2.0m x 2.0m, top surface at Z = 0.0
$fn = 16;

module floor_tile() {
    // Main concrete slab
    color([0.19, 0.21, 0.24], metalness=0.1, roughness=0.85)
        translate([0, 0, -0.09])
            cube([1.92, 1.92, 0.16], center=true);

    // Recessed metallic border frame
    color([0.12, 0.14, 0.17], metalness=0.65, roughness=0.4)
        translate([0, 0, -0.105])
            cube([2.0, 2.0, 0.15], center=true);

    // Subtle center expansion seam cross
    color([0.14, 0.16, 0.19], metalness=0.4, roughness=0.6) {
        translate([0, 0, -0.008])
            cube([1.84, 0.03, 0.01], center=true);
        translate([0, 0, -0.008])
            cube([0.03, 1.84, 0.01], center=true);
    }

    // Four corner anchor rivets
    color([0.36, 0.39, 0.44], metalness=0.9, roughness=0.3, $asa=35.0)
        for (sx = [-0.82, 0.82], sy = [-0.82, 0.82])
            translate([sx, sy, -0.015])
                cylinder(h=0.02, r=0.04, center=false);
}

floor_tile();
```

### sokoban_3d/assets/models/warehouse_wall.scad
```openscad
// Industrial warehouse wall block: 2.0m x 2.0m x 2.2m, sitting on Z = 0.0
$fn = 20;

module warehouse_wall() {
    // Heavy reinforced concrete base
    color([0.24, 0.26, 0.29], metalness=0.1, roughness=0.9)
        translate([0, 0, 0.18])
            cube([1.98, 1.98, 0.36], center=true);

    // Hazard safety bumper trim around base
    color([0.92, 0.68, 0.08], metalness=0.2, roughness=0.45)
        translate([0, 0, 0.24])
            cube([2.01, 2.01, 0.10], center=true);

    // Dark hazard stripe bands
    color([0.12, 0.12, 0.14], metalness=0.3, roughness=0.5)
        for (i = [-0.6, 0.0, 0.6]) {
            translate([i, 0, 0.24])
                cube([0.24, 2.02, 0.105], center=true);
            translate([0, i, 0.24])
                cube([2.02, 0.24, 0.105], center=true);
        }

    // Corrugated steel upper core
    color([0.31, 0.35, 0.40], metalness=0.75, roughness=0.42)
        translate([0, 0, 1.20])
            cube([1.88, 1.88, 1.70], center=true);

    // Vertical structural ribs on all four sides
    color([0.22, 0.25, 0.29], metalness=0.82, roughness=0.35)
        for (pos = [-0.60, 0.0, 0.60]) {
            translate([pos, 0, 1.20])
                cube([0.14, 1.96, 1.66], center=true);
            translate([0, pos, 1.20])
                cube([1.96, 0.14, 1.66], center=true);
        }

    // Corner steel pillars
    color([0.18, 0.20, 0.24], metalness=0.88, roughness=0.3)
        for (sx = [-0.88, 0.88], sy = [-0.88, 0.88])
            translate([sx, sy, 1.10])
                cube([0.22, 0.22, 2.20], center=true);

    // Top steel cap
    color([0.26, 0.29, 0.33], metalness=0.78, roughness=0.36)
        translate([0, 0, 2.13])
            cube([1.98, 1.98, 0.14], center=true);
}

warehouse_wall();
```

### sokoban_3d/assets/models/goal_pad.scad
```openscad
// Animated cargo docking goal pad: 1.8m footprint sitting flush on Z = 0.0
$fn = 32;

goal_animations = [
    ["pulse", [
        ["PadRing", [
            [0.0, [0, 0, 0]],
            [1.5, [0, 0, 90]],
            [3.0, [0, 0, 180]],
            [4.5, [0, 0, 270]],
            [6.0, [0, 0, 360]]
        ]],
        ["CenterBeacon", [
            [0.0, [0, 0, 0], [0, 0, 0.04]],
            [1.0, [0, 0, 0], [0, 0, 0.075]],
            [2.0, [0, 0, 0], [0, 0, 0.04]]
        ]]
    ]]
];

armature(animations=goal_animations) {
    // Base docking plate
    bone(name="PadBase", t=[0, 0, 0], r=[0, 0, 0]) {
        color([0.14, 0.17, 0.21], metalness=0.8, roughness=0.35)
            translate([0, 0, 0.015])
                cube([1.72, 1.72, 0.03], center=true);

        // Four glowing corner brackets
        color([0.05, 0.78, 0.95], metalness=0.1, roughness=0.2, emissive=[0.05, 0.80, 1.0], emissiveIntensity=2.2)
            for (sx = [-0.70, 0.70], sy = [-0.70, 0.70]) {
                translate([sx, sy - sign(sy) * 0.10, 0.032])
                    cube([0.10, 0.30, 0.016], center=true);
                translate([sx - sign(sx) * 0.10, sy, 0.032])
                    cube([0.30, 0.10, 0.016], center=true);
            }
    }

    // Rotating luminous ring
    bone(name="PadRing", t=[0, 0, 0.03], r=[0, 0, 0]) {
        color([0.10, 0.85, 1.0], metalness=0.2, roughness=0.2, emissive=[0.10, 0.85, 1.0], emissiveIntensity=2.4, $asa=45.0)
            difference() {
                cylinder(h=0.02, r=0.62, center=false);
                translate([0, 0, -0.01])
                    cylinder(h=0.04, r=0.50, center=false);
            }

        // Four directional tabs around the rotating ring
        color([0.95, 0.75, 0.15], metalness=0.3, roughness=0.3, emissive=[0.95, 0.72, 0.1], emissiveIntensity=1.6)
            for (ang = [0, 90, 180, 270])
                rotate([0, 0, ang])
                    translate([0, 0.56, 0.015])
                        cube([0.12, 0.16, 0.025], center=true);
    }

    // Floating center diamond beacon
    bone(name="CenterBeacon", t=[0, 0, 0.04], r=[0, 0, 0]) {
        color([0.15, 0.95, 1.0], metalness=0.1, roughness=0.15, emissive=[0.15, 0.95, 1.0], emissiveIntensity=2.8, $asa=45.0)
            rotate([0, 0, 45])
                cube([0.30, 0.30, 0.02], center=true);
    }
}
```

### sokoban_3d/assets/models/cargo_crate.scad
```openscad
// Heavy industrial Sokoban cargo crate: 1.54m x 1.54m x 1.48m, base at Z = 0.0
$fn = 24;

crate_animations = [
    ["idle", [
        ["StatusCore", [
            [0.0, [0, 0, 0], [0, 0, 1.49]],
            [1.0, [0, 0, 0], [0, 0, 1.49]]
        ]]
    ]],
    ["on_goal", [
        ["StatusCore", [
            [0.0, [0, 0, 0], [0, 0, 1.53]],
            [0.6, [0, 0, 90], [0, 0, 1.58]],
            [1.2, [0, 0, 180], [0, 0, 1.53]],
            [1.8, [0, 0, 270], [0, 0, 1.58]],
            [2.4, [0, 0, 360], [0, 0, 1.53]]
        ]]
    ]]
];

armature(animations=crate_animations) {
    bone(name="CrateBody", t=[0, 0, 0], r=[0, 0, 0]) {
        // Inner timber core
        color([0.62, 0.41, 0.21], metalness=0.05, roughness=0.78)
            translate([0, 0, 0.74])
                cube([1.40, 1.40, 1.34], center=true);

        // Horizontal wooden plank grooves (visual relief slats)
        color([0.52, 0.33, 0.16], metalness=0.05, roughness=0.82)
            for (z_lvl = [0.34, 0.74, 1.14]) {
                translate([0, 0, z_lvl])
                    cube([1.44, 1.44, 0.08], center=true);
            }

        // Diagonal timber cross-braces on X and Y faces
        color([0.68, 0.46, 0.24], metalness=0.05, roughness=0.74) {
            for (sy = [-0.69, 0.69])
                translate([0, sy, 0.74])
                    rotate([0, 45, 0])
                        cube([0.14, 0.08, 1.45], center=true);
            for (sx = [-0.69, 0.69])
                translate([sx, 0, 0.74])
                    rotate([45, 0, 0])
                        cube([0.08, 0.14, 1.45], center=true);
        }

        // Heavy gunmetal steel corner frame
        color([0.19, 0.22, 0.26], metalness=0.86, roughness=0.32) {
            // 4 vertical corner posts
            for (sx = [-0.68, 0.68], sy = [-0.68, 0.68])
                translate([sx, sy, 0.74])
                    cube([0.18, 0.18, 1.48], center=true);

            // Bottom rim bars
            for (sy = [-0.68, 0.68])
                translate([0, sy, 0.09])
                    cube([1.54, 0.18, 0.18], center=true);
            for (sx = [-0.68, 0.68])
                translate([sx, 0, 0.09])
                    cube([0.18, 1.54, 0.18], center=true);

            // Top rim bars
            for (sy = [-0.68, 0.68])
                translate([0, sy, 1.39])
                    cube([1.54, 0.18, 0.18], center=true);
            for (sx = [-0.68, 0.68])
                translate([sx, 0, 1.39])
                    cube([0.18, 1.54, 0.18], center=true);
        }

        // Top recessed metal plate
        color([0.25, 0.28, 0.32], metalness=0.8, roughness=0.38)
            translate([0, 0, 1.44])
                cube([1.10, 1.10, 0.08], center=true);
    }

    // Animated top status beacon core
    bone(name="StatusCore", t=[0, 0, 1.49], r=[0, 0, 0]) {
        color([0.95, 0.65, 0.15], metalness=0.3, roughness=0.25, emissive=[0.95, 0.55, 0.10], emissiveIntensity=1.6, $asa=45.0)
            cylinder(h=0.07, r=0.24, center=true);
        color([0.22, 0.25, 0.29], metalness=0.9, roughness=0.3)
            cube([0.56, 0.10, 0.05], center=true);
    }
}
```

### sokoban_3d/assets/models/sokoban_worker.scad
```openscad
// Articulated warehouse worker bot (~1.72m tall), facing +Y (Forward), standing on Z = 0.0
// Left side is -X, Right side is +X
$fn = 24;

worker_animations = [
    ["idle", [
        ["Pelvis", [
            [0.0, [0, 0, 0], [0, 0, 0.72]],
            [0.8, [0, 0, 0], [0, 0, 0.745]],
            [1.6, [0, 0, 0], [0, 0, 0.72]]
        ]],
        ["Torso", [
            [0.0, [0, 0, 0]],
            [0.8, [-2, 0, 0]],
            [1.6, [0, 0, 0]]
        ]],
        ["Head", [
            [0.0, [0, 0, 0]],
            [0.8, [0, 0, 6]],
            [1.6, [0, 0, -6]],
            [2.4, [0, 0, 0]]
        ]],
        ["LeftArm", [
            [0.0, [5, -6, 0]],
            [0.8, [8, -8, 0]],
            [1.6, [5, -6, 0]]
        ]],
        ["RightArm", [
            [0.0, [5, 6, 0]],
            [0.8, [8, 8, 0]],
            [1.6, [5, 6, 0]]
        ]],
        ["LeftLeg", [
            [0.0, [0, 0, 0]],
            [1.6, [0, 0, 0]]
        ]],
        ["RightLeg", [
            [0.0, [0, 0, 0]],
            [1.6, [0, 0, 0]]
        ]]
    ]],
    ["walk", [
        ["Pelvis", [
            [0.0, [0, 0, 0], [0, 0, 0.72]],
            [0.125, [0, 0, 0], [0, 0, 0.77]],
            [0.25, [0, 0, 0], [0, 0, 0.72]],
            [0.375, [0, 0, 0], [0, 0, 0.77]],
            [0.50, [0, 0, 0], [0, 0, 0.72]]
        ]],
        ["Torso", [
            [0.0, [-6, 0, -4]],
            [0.25, [-6, 0, 4]],
            [0.50, [-6, 0, -4]]
        ]],
        ["Head", [
            [0.0, [4, 0, 3]],
            [0.25, [4, 0, -3]],
            [0.50, [4, 0, 3]]
        ]],
        ["LeftLeg", [
            [0.0, [32, 0, 0]],
            [0.25, [-32, 0, 0]],
            [0.50, [32, 0, 0]]
        ]],
        ["RightLeg", [
            [0.0, [-32, 0, 0]],
            [0.25, [32, 0, 0]],
            [0.50, [-32, 0, 0]]
        ]],
        ["LeftArm", [
            [0.0, [-26, -5, 0]],
            [0.25, [28, -5, 0]],
            [0.50, [-26, -5, 0]]
        ]],
        ["RightArm", [
            [0.0, [28, 5, 0]],
            [0.25, [-26, 5, 0]],
            [0.50, [28, 5, 0]]
        ]]
    ]],
    ["push", [
        ["Pelvis", [
            [0.0, [0, 0, 0], [0, 0, 0.68]],
            [0.2, [0, 0, 0], [0, 0, 0.71]],
            [0.4, [0, 0, 0], [0, 0, 0.68]]
        ]],
        ["Torso", [
            [0.0, [-18, 0, 0]],
            [0.2, [-20, 0, 0]],
            [0.4, [-18, 0, 0]]
        ]],
        ["Head", [
            [0.0, [14, 0, 0]],
            [0.2, [16, 0, 0]],
            [0.4, [14, 0, 0]]
        ]],
        ["LeftArm", [
            [0.0, [85, -4, 0]],
            [0.2, [88, -4, 0]],
            [0.4, [85, -4, 0]]
        ]],
        ["RightArm", [
            [0.0, [85, 4, 0]],
            [0.2, [88, 4, 0]],
            [0.4, [85, 4, 0]]
        ]],
        ["LeftLeg", [
            [0.0, [26, 0, 0]],
            [0.2, [-18, 0, 0]],
            [0.4, [26, 0, 0]]
        ]],
        ["RightLeg", [
            [0.0, [-26, 0, 0]],
            [0.2, [18, 0, 0]],
            [0.4, [-26, 0, 0]]
        ]]
    ]]
];

module leg_mesh() {
    // Hip joint sphere
    color([0.18, 0.20, 0.24], metalness=0.85, roughness=0.35, $asa=45.0)
        sphere(r=0.09);

    // Upper thigh
    color([0.24, 0.27, 0.32], metalness=0.7, roughness=0.4, $asa=40.0)
        translate([0, 0, -0.17])
            cylinder(h=0.28, r1=0.085, r2=0.10, center=true);

    // Knee bumper
    color([0.95, 0.62, 0.10], metalness=0.2, roughness=0.4)
        translate([0, 0.03, -0.32])
            cube([0.16, 0.14, 0.10], center=true);

    // Lower shin
    color([0.22, 0.25, 0.29], metalness=0.75, roughness=0.38)
        translate([0, 0, -0.47])
            cube([0.14, 0.15, 0.26], center=true);

    // Heavy industrial boot (sole reaches local z = -0.64 -> world z = 0.0)
    color([0.14, 0.15, 0.18], metalness=0.6, roughness=0.55)
        translate([0, 0.04, -0.59])
            cube([0.17, 0.27, 0.10], center=true);
}

module arm_mesh() {
    // Shoulder cap
    color([0.95, 0.62, 0.10], metalness=0.25, roughness=0.35, $asa=45.0)
        sphere(r=0.095);

    // Upper arm
    color([0.22, 0.25, 0.29], metalness=0.8, roughness=0.35, $asa=40.0)
        translate([0, 0, -0.16])
            cylinder(h=0.24, r=0.07, center=true);

    // Forearm gauntlet
    color([0.95, 0.62, 0.10], metalness=0.25, roughness=0.35)
        translate([0, 0, -0.36])
            cube([0.14, 0.14, 0.22], center=true);

    // Pusher glove / gripper pad
    color([0.15, 0.17, 0.20], metalness=0.7, roughness=0.45)
        translate([0, 0.02, -0.50])
            cube([0.15, 0.18, 0.10], center=true);
}

armature(animations=worker_animations) {
    bone(name="Pelvis", t=[0, 0, 0.72], r=[0, 0, 0]) {
        // Waist / pelvis block
        color([0.20, 0.23, 0.27], metalness=0.8, roughness=0.35)
            cube([0.42, 0.28, 0.16], center=true);

        // Belt utility buckle on +Y front
        color([0.95, 0.70, 0.12], metalness=0.4, roughness=0.3)
            translate([0, 0.15, 0])
                cube([0.16, 0.04, 0.08], center=true);

        // Left Leg (-X)
        bone(name="LeftLeg", t=[-0.15, 0, -0.08], r=[0, 0, 0]) {
            leg_mesh();
        }

        // Right Leg (+X)
        bone(name="RightLeg", t=[0.15, 0, -0.08], r=[0, 0, 0]) {
            leg_mesh();
        }

        // Upper Torso
        bone(name="Torso", t=[0, 0, 0.10], r=[0, 0, 0]) {
            // Chest armor shell
            color([0.95, 0.62, 0.10], metalness=0.22, roughness=0.35)
                translate([0, 0, 0.22])
                    cube([0.48, 0.34, 0.42], center=true);

            // Dark chest grill on +Y front
            color([0.17, 0.19, 0.23], metalness=0.8, roughness=0.38)
                translate([0, 0.17, 0.24])
                    cube([0.28, 0.04, 0.22], center=true);

            // Glowing chest core emblem on +Y
            color([0.10, 0.90, 1.0], metalness=0.1, roughness=0.2, emissive=[0.10, 0.90, 1.0], emissiveIntensity=2.2, $asa=45.0)
                translate([0, 0.195, 0.26])
                    rotate([90, 0, 0])
                        cylinder(h=0.02, r=0.05, center=true);

            // Power backpack on -Y back
            color([0.24, 0.27, 0.31], metalness=0.75, roughness=0.4)
                translate([0, -0.22, 0.22])
                    cube([0.34, 0.14, 0.34], center=true);
            color([0.15, 0.95, 0.45], metalness=0.1, roughness=0.2, emissive=[0.15, 0.95, 0.45], emissiveIntensity=1.8)
                translate([0, -0.295, 0.26])
                    cube([0.18, 0.02, 0.06], center=true);

            // Left Arm (-X)
            bone(name="LeftArm", t=[-0.32, 0, 0.36], r=[5, -6, 0]) {
                arm_mesh();
            }

            // Right Arm (+X)
            bone(name="RightArm", t=[0.32, 0, 0.36], r=[5, 6, 0]) {
                arm_mesh();
            }

            // Head & Safety Helmet
            bone(name="Head", t=[0, 0, 0.48], r=[0, 0, 0]) {
                // Neck cylinder
                color([0.18, 0.20, 0.24], metalness=0.85, roughness=0.35, $asa=40.0)
                    translate([0, 0, 0.03])
                        cylinder(h=0.08, r=0.08, center=true);

                // Head block
                color([0.24, 0.27, 0.32], metalness=0.7, roughness=0.35)
                    translate([0, 0, 0.18])
                        cube([0.30, 0.28, 0.24], center=true);

                // Glowing cyan visor facing +Y (Forward)
                color([0.10, 0.92, 1.0], metalness=0.1, roughness=0.15, emissive=[0.10, 0.92, 1.0], emissiveIntensity=2.6)
                    translate([0, 0.145, 0.18])
                        cube([0.24, 0.03, 0.09], center=true);

                // Yellow hard hat dome
                color([0.98, 0.74, 0.08], metalness=0.15, roughness=0.30, $asa=45.0)
                    translate([0, 0, 0.30])
                        scale([1.05, 1.05, 0.55])
                            sphere(r=0.19);

                // Hard hat brim facing +Y
                color([0.98, 0.74, 0.08], metalness=0.15, roughness=0.30)
                    translate([0, 0.18, 0.29])
                        cube([0.28, 0.12, 0.03], center=true);

                // Top beacon antenna
                color([0.95, 0.30, 0.10], metalness=0.2, roughness=0.2, emissive=[0.95, 0.30, 0.10], emissiveIntensity=1.5, $asa=45.0)
                    translate([0.12, -0.06, 0.42])
                        cylinder(h=0.08, r=0.025, center=true);
            }
        }
    }
}
```

### sokoban_3d/scripts/level_data.gd
```gdscript
class_name LevelData
extends RefCounted

# Grid legend:
# '#' = Wall
# ' ' = Floor
# '.' = Goal pad
# '$' = Cargo crate
# '*' = Cargo crate starting on goal
# '@' = Player on floor
# '+' = Player starting on goal

const LEVELS: Array[Dictionary] = [
	{
		"name": "Warehouse Intro",
		"par_moves": 9,
		"map": [
			"#######",
			"#  .  #",
			"#  $  #",
			"#.$@$.#",
			"#     #",
			"#######"
		]
	},
	{
		"name": "Corner Dock",
		"par_moves": 14,
		"map": [
			"#######",
			"#..   #",
			"# $$  #",
			"#  @  #",
			"#     #",
			"#######"
		]
	},
	{
		"name": "Three Crates",
		"par_moves": 18,
		"map": [
			"########",
			"#  ..  #",
			"# .$$  #",
			"#  $@  #",
			"#      #",
			"########"
		]
	},
	{
		"name": "Corridor Swap",
		"par_moves": 24,
		"map": [
			"#########",
			"#       #",
			"# $ . $ #",
			"#  .@.  #",
			"#   $   #",
			"#       #",
			"#########"
		]
	},
	{
		"name": "Forklift Cross",
		"par_moves": 26,
		"map": [
			"#########",
			"#   .   #",
			"#  $$$  #",
			"# . @ . #",
			"#   $   #",
			"#   .   #",
			"#########"
		]
	},
	{
		"name": "Double Lock",
		"par_moves": 29,
		"map": [
			"#########",
			"#  #    #",
			"#  $  . #",
			"# #$##. #",
			"# @ $ . #",
			"#       #",
			"#########"
		]
	},
	{
		"name": "Depot Master",
		"par_moves": 38,
		"map": [
			"##########",
			"#  #     #",
			"#  $ . . #",
			"#  $#$ . #",
			"# @ $  . #",
			"#        #",
			"##########"
		]
	}
]

static func get_level_count() -> int:
	return LEVELS.size()

static func parse_level(index: int) -> Dictionary:
	var idx: int = clampi(index, 0, LEVELS.size() - 1)
	var raw: Dictionary = LEVELS[idx]
	var lines: Array = raw["map"]

	var height: int = lines.size()
	var width: int = 0
	for line in lines:
		width = maxi(width, line.length())

	var walls: Dictionary = {}
	var floors: Dictionary = {}
	var goals: Array[Vector2i] = []
	var crates: Array[Vector2i] = []
	var player_start: Vector2i = Vector2i.ZERO

	for r in range(height):
		var line: String = lines[r]
		for c in range(line.length()):
			var ch: String = line[c]
			var pos := Vector2i(c, r)
			match ch:
				"#":
					walls[pos] = true
				" ":
					floors[pos] = true
				".":
					floors[pos] = true
					goals.append(pos)
				"$":
					floors[pos] = true
					crates.append(pos)
				"*":
					floors[pos] = true
					goals.append(pos)
					crates.append(pos)
				"@":
					floors[pos] = true
					player_start = pos
				"+":
					floors[pos] = true
					goals.append(pos)
					player_start = pos

	return {
		"index": idx,
		"name": raw["name"],
		"par_moves": raw["par_moves"],
		"width": width,
		"height": height,
		"walls": walls,
		"floors": floors,
		"goals": goals,
		"crates": crates,
		"player_start": player_start
	}
```

### sokoban_3d/scripts/sound_synth.gd
```gdscript
class_name SoundSynth
extends Node

const SAMPLE_RATE: int = 22050

var muted: bool = false

var _sfx_step: AudioStreamWAV
var _sfx_push: AudioStreamWAV
var _sfx_goal_on: AudioStreamWAV
var _sfx_undo: AudioStreamWAV
var _sfx_bump: AudioStreamWAV
var _sfx_win: AudioStreamWAV

func _ready() -> void:
	_sfx_step = _make_tone(190.0, 120.0, 0.06, 0.22)
	_sfx_push = _make_noise_thump(140.0, 85.0, 0.11, 0.35)
	_sfx_goal_on = _make_chord([523.25, 659.25, 783.99], 0.20, 0.30)
	_sfx_undo = _make_tone(420.0, 260.0, 0.08, 0.22)
	_sfx_bump = _make_tone(110.0, 70.0, 0.08, 0.28)
	_sfx_win = _make_arpeggio([523.25, 659.25, 783.99, 1046.5], 0.10, 0.32)

func play_step() -> void:
	_play(_sfx_step)

func play_push() -> void:
	_play(_sfx_push)

func play_goal_on() -> void:
	_play(_sfx_goal_on)

func play_undo() -> void:
	_play(_sfx_undo)

func play_bump() -> void:
	_play(_sfx_bump)

func play_win() -> void:
	_play(_sfx_win)

func _play(stream: AudioStreamWAV) -> void:
	if muted or stream == null:
		return
	var p := AudioStreamPlayer.new()
	p.stream = stream
	p.volume_db = -6.0
	add_child(p)
	p.finished.connect(p.queue_free)
	p.play()

func _make_tone(freq_start: float, freq_end: float, duration: float, amp: float) -> AudioStreamWAV:
	var count: int = int(SAMPLE_RATE * duration)
	var data := PackedByteArray()
	data.resize(count * 2)
	var phase: float = 0.0
	for i in range(count):
		var t: float = float(i) / float(count)
		var freq: float = lerpf(freq_start, freq_end, t)
		phase += TAU * freq / float(SAMPLE_RATE)
		var env: float = (1.0 - t) * (1.0 - t)
		var sample: float = sin(phase) * env * amp
		var val: int = clampi(int(sample * 32767.0), -32768, 32767)
		data.encode_s16(i * 2, val)
	return _build_wav(data)

func _make_noise_thump(freq_start: float, freq_end: float, duration: float, amp: float) -> AudioStreamWAV:
	var count: int = int(SAMPLE_RATE * duration)
	var data := PackedByteArray()
	data.resize(count * 2)
	var phase: float = 0.0
	var rng := RandomNumberGenerator.new()
	rng.seed = 42
	for i in range(count):
		var t: float = float(i) / float(count)
		var freq: float = lerpf(freq_start, freq_end, t)
		phase += TAU * freq / float(SAMPLE_RATE)
		var env: float = (1.0 - t)
		var tone: float = sin(phase) * 0.7 + rng.randf_range(-0.3, 0.3)
		var sample: float = tone * env * amp
		var val: int = clampi(int(sample * 32767.0), -32768, 32767)
		data.encode_s16(i * 2, val)
	return _build_wav(data)

func _make_chord(freqs: Array, duration: float, amp: float) -> AudioStreamWAV:
	var count: int = int(SAMPLE_RATE * duration)
	var data := PackedByteArray()
	data.resize(count * 2)
	for i in range(count):
		var t: float = float(i) / float(count)
		var time_sec: float = float(i) / float(SAMPLE_RATE)
		var env: float = sin(minf(t * 10.0, 1.0) * PI * 0.5) * (1.0 - t)
		var sum: float = 0.0
		for f in freqs:
			sum += sin(TAU * float(f) * time_sec)
		var sample: float = (sum / float(freqs.size())) * env * amp
		var val: int = clampi(int(sample * 32767.0), -32768, 32767)
		data.encode_s16(i * 2, val)
	return _build_wav(data)

func _make_arpeggio(freqs: Array, note_len: float, amp: float) -> AudioStreamWAV:
	var total_dur: float = note_len * float(freqs.size()) + 0.25
	var count: int = int(SAMPLE_RATE * total_dur)
	var data := PackedByteArray()
	data.resize(count * 2)
	for i in range(count):
		var time_sec: float = float(i) / float(SAMPLE_RATE)
		var sum: float = 0.0
		for idx in range(freqs.size()):
			var start_t: float = float(idx) * note_len
			if time_sec >= start_t:
				var dt: float = time_sec - start_t
				var env: float = exp(-dt * 6.5)
				sum += sin(TAU * float(freqs[idx]) * dt) * env
		var sample: float = (sum * 0.45) * amp
		var val: int = clampi(int(sample * 32767.0), -32768, 32767)
		data.encode_s16(i * 2, val)
	return _build_wav(data)

func _build_wav(data: PackedByteArray) -> AudioStreamWAV:
	var wav := AudioStreamWAV.new()
	wav.data = data
	wav.format = AudioStreamWAV.FORMAT_16_BITS
	wav.mix_rate = SAMPLE_RATE
	wav.stereo = false
	return wav
```

### sokoban_3d/scripts/camera_rig.gd
```gdscript
class_name CameraRig
extends Node3D

@onready var camera: Camera3D = $Camera3D

enum ViewMode { ISOMETRIC, TOP_DOWN }
var mode: ViewMode = ViewMode.ISOMETRIC

var _target_focus: Vector3 = Vector3.ZERO
var _zoom_scale: float = 1.0
var _base_distance: float = 16.0

const ISO_PITCH_DEG: float = -48.0
const TOP_PITCH_DEG: float = -84.0
const MIN_ZOOM: float = 0.65
const MAX_ZOOM: float = 1.55

func _ready() -> void:
	_apply_camera_immediately()

func configure_for_board(center: Vector3, board_span: float) -> void:
	_target_focus = center
	_base_distance = clampf(board_span * 1.15, 11.0, 26.0)

func toggle_mode() -> ViewMode:
	if mode == ViewMode.ISOMETRIC:
		mode = ViewMode.TOP_DOWN
	else:
		mode = ViewMode.ISOMETRIC
	return mode

func get_mode_label() -> String:
	return "Iso" if mode == ViewMode.ISOMETRIC else "Top"

func _unhandled_input(event: InputEvent) -> void:
	if event is InputEventMouseButton and event.pressed:
		if event.button_index == MOUSE_BUTTON_WHEEL_UP:
			_zoom_scale = clampf(_zoom_scale - 0.08, MIN_ZOOM, MAX_ZOOM)
		elif event.button_index == MOUSE_BUTTON_WHEEL_DOWN:
			_zoom_scale = clampf(_zoom_scale + 0.08, MIN_ZOOM, MAX_ZOOM)

func _process(delta: float) -> void:
	var desired_pitch: float = ISO_PITCH_DEG if mode == ViewMode.ISOMETRIC else TOP_PITCH_DEG
	var desired_yaw: float = 0.0
	var desired_dist: float = _base_distance * _zoom_scale

	global_position = global_position.lerp(_target_focus, 1.0 - exp(-8.0 * delta))
	rotation_degrees.x = lerpf(rotation_degrees.x, desired_pitch, 1.0 - exp(-8.0 * delta))
	rotation_degrees.y = lerpf(rotation_degrees.y, desired_yaw, 1.0 - exp(-8.0 * delta))
	if camera:
		camera.position.z = lerpf(camera.position.z, desired_dist, 1.0 - exp(-8.0 * delta))

func _apply_camera_immediately() -> void:
	global_position = _target_focus
	rotation_degrees = Vector3(ISO_PITCH_DEG, 0.0, 0.0)
	if camera:
		camera.position = Vector3(0.0, 0.0, _base_distance * _zoom_scale)
```

### sokoban_3d/scripts/player_controller.gd
```gdscript
class_name PlayerController
extends Node3D

var _visual_root: Node3D
var _anim_player: AnimationPlayer
var _target_yaw: float = 0.0
var _current_anim: String = ""

func setup_model(model_scene: PackedScene) -> void:
	if _visual_root:
		_visual_root.queue_free()

	_visual_root = model_scene.instantiate() as Node3D
	add_child(_visual_root)

	_anim_player = _find_animation_player(_visual_root)
	if _anim_player:
		for anim_name in _anim_player.get_animation_list():
			var anim := _anim_player.get_animation(anim_name)
			if anim:
				anim.loop_mode = Animation.LOOP_LINEAR
		play_anim("idle", true)

func face_direction(world_dir: Vector3, immediate: bool = false) -> void:
	if world_dir.length_squared() < 0.001:
		return
	# OpenSCAD models face +Y which imports to Godot Forward (-Z).
	# Local -Z points toward world_dir when yaw = atan2(-world_dir.x, -world_dir.z).
	_target_yaw = atan2(-world_dir.x, -world_dir.z)
	if immediate:
		rotation.y = _target_yaw

func play_anim(anim_name: String, force: bool = false) -> void:
	if _anim_player == null:
		return
	var resolved: String = _resolve_anim_name(anim_name)
	if resolved.is_empty():
		return
	if not force and _current_anim == resolved and _anim_player.is_playing():
		return
	_current_anim = resolved
	_anim_player.play(resolved, 0.08)

func _process(delta: float) -> void:
	rotation.y = lerp_angle(rotation.y, _target_yaw, 1.0 - exp(-18.0 * delta))

func _resolve_anim_name(requested: String) -> String:
	if _anim_player.has_animation(requested):
		return requested
	for candidate in _anim_player.get_animation_list():
		if candidate.to_lower().contains(requested.to_lower()):
			return candidate
	return ""

func _find_animation_player(root: Node) -> AnimationPlayer:
	if root is AnimationPlayer:
		return root as AnimationPlayer
	for child in root.get_children():
		var found := _find_animation_player(child)
		if found:
			return found
	return null
```

### sokoban_3d/scripts/crate_entity.gd
```gdscript
class_name CrateEntity
extends Node3D

var grid_pos: Vector2i = Vector2i.ZERO
var on_goal: bool = false

var _visual_root: Node3D
var _anim_player: AnimationPlayer
var _beacon_light: OmniLight3D
var _goal_ring: MeshInstance3D

func setup_model(model_scene: PackedScene) -> void:
	if _visual_root:
		_visual_root.queue_free()

	_visual_root = model_scene.instantiate() as Node3D
	add_child(_visual_root)
	_anim_player = _find_animation_player(_visual_root)

	if _anim_player:
		for anim_name in _anim_player.get_animation_list():
			var anim := _anim_player.get_animation(anim_name)
			if anim:
				anim.loop_mode = Animation.LOOP_LINEAR

	# Create a crisp Godot-side goal indicator light & top ring overlay
	_beacon_light = OmniLight3D.new()
	_beacon_light.position = Vector3(0.0, 1.75, 0.0)
	_beacon_light.light_color = Color(0.2, 1.0, 0.55)
	_beacon_light.light_energy = 0.0
	_beacon_light.omni_range = 3.2
	add_child(_beacon_light)

	var torus := TorusMesh.new()
	torus.inner_radius = 0.24
	torus.outer_radius = 0.34
	var mat := StandardMaterial3D.new()
	mat.albedo_color = Color(0.15, 0.95, 0.55)
	mat.emission_enabled = true
	mat.emission = Color(0.15, 1.0, 0.55)
	mat.emission_energy_multiplier = 2.6
	torus.material = mat

	_goal_ring = MeshInstance3D.new()
	_goal_ring.mesh = torus
	_goal_ring.position = Vector3(0.0, 1.54, 0.0)
	_goal_ring.visible = false
	add_child(_goal_ring)

	set_on_goal(false)

func set_on_goal(is_on_goal: bool) -> void:
	on_goal = is_on_goal
	if _beacon_light:
		_beacon_light.light_energy = 1.6 if on_goal else 0.0
	if _goal_ring:
		_goal_ring.visible = on_goal
	if _anim_player:
		var target_anim := _resolve_anim_name("on_goal" if on_goal else "idle")
		if not target_anim.is_empty():
			_anim_player.play(target_anim, 0.15)

func play_goal_pop() -> void:
	var tween := create_tween()
	tween.tween_property(_visual_root, "scale", Vector3(1.06, 0.94, 1.06), 0.07)
	tween.tween_property(_visual_root, "scale", Vector3(0.98, 1.05, 0.98), 0.08)
	tween.tween_property(_visual_root, "scale", Vector3.ONE, 0.08)

func _resolve_anim_name(requested: String) -> String:
	if _anim_player.has_animation(requested):
		return requested
	for candidate in _anim_player.get_animation_list():
		if candidate.to_lower().contains(requested.to_lower()):
			return candidate
	return ""

func _find_animation_player(root: Node) -> AnimationPlayer:
	if root is AnimationPlayer:
		return root as AnimationPlayer
	for child in root.get_children():
		var found := _find_animation_player(child)
		if found:
			return found
	return null
```

### sokoban_3d/scripts/main.gd
```gdscript
extends Node3D

const TILE_SIZE: float = 2.0
const MOVE_DURATION: float = 0.14
const HOLD_INITIAL_DELAY: float = 0.28
const HOLD_REPEAT_INTERVAL: float = 0.16
const SWIPE_THRESHOLD_PX: float = 44.0

var _floor_scad: PackedScene
var _wall_scad: PackedScene
var _goal_scad: PackedScene
var _crate_scad: PackedScene
var _worker_scad: PackedScene

@onready var board_root: Node3D = $BoardRoot
@onready var camera_rig: CameraRig = $CameraRig
@onready var sound_synth: SoundSynth = $SoundSynth

@onready var level_label: Label = %LevelLabel
@onready var moves_label: Label = %MovesLabel
@onready var pushes_label: Label = %PushesLabel
@onready var goals_label: Label = %GoalsLabel
@onready var cam_button: Button = %CameraButton
@onready var mute_button: Button = %MuteButton
@onready var win_overlay: Control = %WinOverlay
@onready var win_title: Label = %WinTitle
@onready var win_stats: Label = %WinStats
@onready var next_win_button: Button = %NextWinButton

var _current_level_index: int = 0
var _walls: Dictionary = {}
var _goals: Dictionary = {}
var _crates_by_pos: Dictionary = {}
var _crate_entities: Array[CrateEntity] = []
var _player_pos: Vector2i = Vector2i.ZERO
var _player: PlayerController

var _moves: int = 0
var _pushes: int = 0
var _undo_stack: Array[Dictionary] = []

var _is_moving: bool = false
var _level_won: bool = false
var _buffered_dir: Vector2i = Vector2i.ZERO
var _move_tween: Tween

# Delayed Auto Shift (DAS) for comfortable single-tap precision + smooth hold repeat
var _last_held_dir: Vector2i = Vector2i.ZERO
var _hold_timer: float = 0.0
var _hold_repeating: bool = false

# Touch D-Pad & Swipe tracking
var _touch_dpad_dir: Vector2i = Vector2i.ZERO
var _active_touch_id: int = -1
var _touch_start_pos: Vector2 = Vector2.ZERO
var _touch_swipe_fired: bool = false

func _ready() -> void:
	_floor_scad = load("res://assets/models/floor_tile.scad") as PackedScene
	_wall_scad = load("res://assets/models/warehouse_wall.scad") as PackedScene
	_goal_scad = load("res://assets/models/goal_pad.scad") as PackedScene
	_crate_scad = load("res://assets/models/cargo_crate.scad") as PackedScene
	_worker_scad = load("res://assets/models/sokoban_worker.scad") as PackedScene

	_connect_ui()
	load_level(0)

func _connect_ui() -> void:
	%PrevLevelButton.pressed.connect(func(): load_level(_current_level_index - 1))
	%NextLevelButton.pressed.connect(func(): load_level(_current_level_index + 1))
	%UndoButton.pressed.connect(undo_move)
	%RestartButton.pressed.connect(restart_level)
	cam_button.pressed.connect(_on_toggle_camera)
	mute_button.pressed.connect(_on_toggle_mute)
	%ReplayWinButton.pressed.connect(restart_level)
	next_win_button.pressed.connect(_on_next_win)

	_bind_dpad_button(%TouchUpButton, Vector2i(0, -1))
	_bind_dpad_button(%TouchDownButton, Vector2i(0, 1))
	_bind_dpad_button(%TouchLeftButton, Vector2i(-1, 0))
	_bind_dpad_button(%TouchRightButton, Vector2i(1, 0))

func _bind_dpad_button(btn: Button, dir: Vector2i) -> void:
	btn.button_down.connect(func(): _on_dpad_down(dir))
	btn.button_up.connect(func(): _on_dpad_up(dir))

func _on_dpad_down(dir: Vector2i) -> void:
	if _level_won:
		return
	_touch_dpad_dir = dir
	_reset_hold_timer(dir)
	_trigger_discrete_move(dir)

func _on_dpad_up(dir: Vector2i) -> void:
	if _touch_dpad_dir == dir:
		_touch_dpad_dir = Vector2i.ZERO

func load_level(index: int) -> void:
	var total: int = LevelData.get_level_count()
	_current_level_index = wrapi(index, 0, total)

	if _move_tween and _move_tween.is_valid():
		_move_tween.kill()

	_is_moving = false
	_level_won = false
	_buffered_dir = Vector2i.ZERO
	_touch_dpad_dir = Vector2i.ZERO
	_last_held_dir = Vector2i.ZERO
	_hold_timer = 0.0
	_hold_repeating = false
	_active_touch_id = -1
	_touch_swipe_fired = false

	_moves = 0
	_pushes = 0
	_undo_stack.clear()
	_walls.clear()
	_goals.clear()
	_crates_by_pos.clear()
	_crate_entities.clear()
	win_overlay.visible = false

	for child in board_root.get_children():
		child.queue_free()

	var data: Dictionary = LevelData.parse_level(_current_level_index)
	_walls = data["walls"]
	var floors: Dictionary = data["floors"]
	var goals_list: Array[Vector2i] = data["goals"]
	var crates_list: Array[Vector2i] = data["crates"]
	_player_pos = data["player_start"]

	# Spawn floor tiles under floors and walls
	for pos in floors.keys():
		_spawn_static_tile(_floor_scad, pos)
	for pos in _walls.keys():
		_spawn_static_tile(_floor_scad, pos)
		_spawn_static_tile(_wall_scad, pos)

	# Spawn goal pads and start their pulse animation
	for gpos in goals_list:
		_goals[gpos] = true
		var goal_node := _spawn_static_tile(_goal_scad, gpos)
		if goal_node:
			_start_looping_animations(goal_node)

	# Spawn crates
	for cpos in crates_list:
		var crate := CrateEntity.new()
		crate.setup_model(_crate_scad)
		crate.grid_pos = cpos
		crate.position = _grid_to_world(cpos)
		board_root.add_child(crate)
		_crates_by_pos[cpos] = crate
		_crate_entities.append(crate)
		crate.set_on_goal(_goals.has(cpos))

	# Spawn player
	_player = PlayerController.new()
	_player.setup_model(_worker_scad)
	_player.position = _grid_to_world(_player_pos)
	_player.face_direction(Vector3.FORWARD, true)
	board_root.add_child(_player)

	# Configure camera rig to frame the board
	var width: int = data["width"]
	var height: int = data["height"]
	var center := Vector3(
		float(width - 1) * TILE_SIZE * 0.5,
		0.4,
		float(height - 1) * TILE_SIZE * 0.5
	)
	var span: float = maxf(float(width), float(height)) * TILE_SIZE
	camera_rig.configure_for_board(center, span)

	_update_hud(data["name"], data["par_moves"])

func restart_level() -> void:
	load_level(_current_level_index)

func undo_move() -> void:
	if _is_moving or _undo_stack.is_empty():
		return

	if _level_won:
		_level_won = false
		win_overlay.visible = false

	_buffered_dir = Vector2i.ZERO
	var snap: Dictionary = _undo_stack.pop_back()
	_player_pos = snap["player"]
	_moves = snap["moves"]
	_pushes = snap["pushes"]

	var crate_positions: Array = snap["crates"]
	_crates_by_pos.clear()
	for i in range(_crate_entities.size()):
		var crate := _crate_entities[i]
		var pos: Vector2i = crate_positions[i]
		crate.grid_pos = pos
		crate.position = _grid_to_world(pos)
		_crates_by_pos[pos] = crate
		crate.set_on_goal(_goals.has(pos))

	_player.position = _grid_to_world(_player_pos)
	_player.face_direction(snap["facing"], true)
	_player.play_anim("idle", true)
	sound_synth.play_undo()

	var data: Dictionary = LevelData.parse_level(_current_level_index)
	_update_hud(data["name"], data["par_moves"])

func _unhandled_input(event: InputEvent) -> void:
	if event.is_action_pressed("restart"):
		restart_level()
		return
	if event.is_action_pressed("undo"):
		undo_move()
		return
	if event.is_action_pressed("camera_toggle"):
		_on_toggle_camera()
		return
	if event.is_action_pressed("next_level"):
		load_level(_current_level_index + 1)
		return
	if event.is_action_pressed("prev_level"):
		load_level(_current_level_index - 1)
		return

	if _level_won:
		return

	# Touch screen swipe support over the 3D playfield
	if event is InputEventScreenTouch:
		var touch := event as InputEventScreenTouch
		if touch.pressed and _active_touch_id == -1:
			_active_touch_id = touch.index
			_touch_start_pos = touch.position
			_touch_swipe_fired = false
		elif not touch.pressed and touch.index == _active_touch_id:
			_active_touch_id = -1
			_touch_swipe_fired = false
		return

	if event is InputEventScreenDrag:
		var drag := event as InputEventScreenDrag
		if drag.index == _active_touch_id and not _touch_swipe_fired:
			var delta_pos: Vector2 = drag.position - _touch_start_pos
			if delta_pos.length() >= SWIPE_THRESHOLD_PX:
				_touch_swipe_fired = true
				var swipe_dir: Vector2i
				if absf(delta_pos.x) > absf(delta_pos.y):
					swipe_dir = Vector2i(1, 0) if delta_pos.x > 0.0 else Vector2i(-1, 0)
				else:
					swipe_dir = Vector2i(0, 1) if delta_pos.y > 0.0 else Vector2i(0, -1)
				_trigger_discrete_move(swipe_dir)
		return

	# Ignore OS key-repeat echoes so our DAS timer governs hold speed cleanly
	if event.is_echo():
		return

	var dir := Vector2i.ZERO
	if event.is_action_pressed("move_up"):
		dir = Vector2i(0, -1)
	elif event.is_action_pressed("move_down"):
		dir = Vector2i(0, 1)
	elif event.is_action_pressed("move_left"):
		dir = Vector2i(-1, 0)
	elif event.is_action_pressed("move_right"):
		dir = Vector2i(1, 0)

	if dir != Vector2i.ZERO:
		_reset_hold_timer(dir)
		_trigger_discrete_move(dir)

func _trigger_discrete_move(dir: Vector2i) -> void:
	if _is_moving:
		_buffered_dir = dir
	else:
		_try_move(dir)

func _reset_hold_timer(dir: Vector2i) -> void:
	_last_held_dir = dir
	_hold_timer = 0.0
	_hold_repeating = false

func _process(delta: float) -> void:
	var held_dir := _read_held_direction()
	if held_dir == Vector2i.ZERO:
		_last_held_dir = Vector2i.ZERO
		_hold_timer = 0.0
		_hold_repeating = false
		if not _is_moving and _player:
			_player.play_anim("idle")
		return

	if held_dir != _last_held_dir:
		_reset_hold_timer(held_dir)
	else:
		_hold_timer += delta

	if _level_won or _is_moving:
		return

	if not _hold_repeating:
		if _hold_timer >= HOLD_INITIAL_DELAY:
			_hold_repeating = true
			_hold_timer = 0.0
			_try_move(held_dir)
	else:
		if _hold_timer >= HOLD_REPEAT_INTERVAL:
			_hold_timer = 0.0
			_try_move(held_dir)

func _read_held_direction() -> Vector2i:
	if _touch_dpad_dir != Vector2i.ZERO:
		return _touch_dpad_dir
	if Input.is_action_pressed("move_up"):
		return Vector2i(0, -1)
	if Input.is_action_pressed("move_down"):
		return Vector2i(0, 1)
	if Input.is_action_pressed("move_left"):
		return Vector2i(-1, 0)
	if Input.is_action_pressed("move_right"):
		return Vector2i(1, 0)
	return Vector2i.ZERO

func _try_move(dir: Vector2i) -> void:
	var next_pos: Vector2i = _player_pos + dir
	var world_dir := Vector3(float(dir.x), 0.0, float(dir.y))
	_player.face_direction(world_dir)

	if _walls.has(next_pos):
		_player.play_anim("idle")
		sound_synth.play_bump()
		return

	var pushed_crate: CrateEntity = _crates_by_pos.get(next_pos, null)
	if pushed_crate != null:
		var crate_dest: Vector2i = next_pos + dir
		if _walls.has(crate_dest) or _crates_by_pos.has(crate_dest):
			_player.play_anim("idle")
			sound_synth.play_bump()
			return

		_push_undo_snapshot(world_dir)
		_crates_by_pos.erase(next_pos)
		_crates_by_pos[crate_dest] = pushed_crate
		pushed_crate.grid_pos = crate_dest
		_pushes += 1
	else:
		_push_undo_snapshot(world_dir)

	_player_pos = next_pos
	_moves += 1
	_is_moving = true

	if pushed_crate != null:
		_player.play_anim("push")
		sound_synth.play_push()
	else:
		_player.play_anim("walk")
		sound_synth.play_step()

	_move_tween = create_tween()
	_move_tween.set_parallel(true)
	_move_tween.tween_property(_player, "position", _grid_to_world(_player_pos), MOVE_DURATION).set_trans(Tween.TRANS_SINE).set_ease(Tween.EASE_OUT)
	if pushed_crate != null:
		_move_tween.tween_property(pushed_crate, "position", _grid_to_world(pushed_crate.grid_pos), MOVE_DURATION).set_trans(Tween.TRANS_SINE).set_ease(Tween.EASE_OUT)
	_move_tween.set_parallel(false)
	_move_tween.tween_callback(_on_move_finished.bind(pushed_crate))

func _on_move_finished(pushed_crate: CrateEntity) -> void:
	_is_moving = false

	if pushed_crate != null:
		var was_on_goal: bool = pushed_crate.on_goal
		var now_on_goal: bool = _goals.has(pushed_crate.grid_pos)
		pushed_crate.set_on_goal(now_on_goal)
		if now_on_goal and not was_on_goal:
			pushed_crate.play_goal_pop()
			sound_synth.play_goal_on()

	var data: Dictionary = LevelData.parse_level(_current_level_index)
	_update_hud(data["name"], data["par_moves"])

	if _check_win():
		_on_level_won(data["name"], data["par_moves"])
		return

	if _buffered_dir != Vector2i.ZERO:
		var queued := _buffered_dir
		_buffered_dir = Vector2i.ZERO
		_try_move(queued)
		return

	_player.play_anim("idle")

func _check_win() -> bool:
	if _goals.is_empty():
		return false
	for gpos in _goals.keys():
		if not _crates_by_pos.has(gpos):
			return false
	return true

func _on_level_won(level_name: String, par_moves: int) -> void:
	_level_won = true
	_buffered_dir = Vector2i.ZERO
	_touch_dpad_dir = Vector2i.ZERO
	_player.play_anim("idle", true)
	sound_synth.play_win()

	var is_last: bool = _current_level_index >= LevelData.get_level_count() - 1
	win_title.text = "ALL LEVELS CLEARED!" if is_last else "SHIFT COMPLETE!"
	win_stats.text = "%s\nMoves: %d (Par %d)   |   Pushes: %d" % [level_name, _moves, par_moves, _pushes]
	next_win_button.text = "Play Again from Level 1" if is_last else "Next Level (N)"
	win_overlay.visible = true

func _push_undo_snapshot(facing_dir: Vector3) -> void:
	var crate_positions: Array[Vector2i] = []
	for crate in _crate_entities:
		crate_positions.append(crate.grid_pos)
	_undo_stack.append({
		"player": _player_pos,
		"crates": crate_positions,
		"moves": _moves,
		"pushes": _pushes,
		"facing": facing_dir
	})

func _update_hud(level_name: String, par_moves: int) -> void:
	var total: int = LevelData.get_level_count()
	level_label.text = "Level %d/%d: %s" % [_current_level_index + 1, total, level_name]
	moves_label.text = "Moves: %d (Par %d)" % [_moves, par_moves]
	pushes_label.text = "Pushes: %d" % _pushes

	var covered: int = 0
	for gpos in _goals.keys():
		if _crates_by_pos.has(gpos):
			covered += 1
	goals_label.text = "Docked: %d / %d" % [covered, _goals.size()]

func _on_toggle_camera() -> void:
	camera_rig.toggle_mode()
	cam_button.text = "Cam: %s (C)" % camera_rig.get_mode_label()

func _on_toggle_mute() -> void:
	sound_synth.muted = not sound_synth.muted
	mute_button.text = "Sound: Off" if sound_synth.muted else "Sound: On"

func _on_next_win() -> void:
	load_level(_current_level_index + 1)

func _grid_to_world(gpos: Vector2i) -> Vector3:
	return Vector3(float(gpos.x) * TILE_SIZE, 0.0, float(gpos.y) * TILE_SIZE)

func _spawn_static_tile(scene: PackedScene, gpos: Vector2i) -> Node3D:
	if scene == null:
		return null
	var inst := scene.instantiate() as Node3D
	inst.position = _grid_to_world(gpos)
	board_root.add_child(inst)
	return inst

func _start_looping_animations(root: Node) -> void:
	if root == null:
		return
	if root is AnimationPlayer:
		var ap := root as AnimationPlayer
		for anim_name in ap.get_animation_list():
			var anim := ap.get_animation(anim_name)
			if anim:
				anim.loop_mode = Animation.LOOP_LINEAR
		if ap.get_animation_list().size() > 0:
			ap.play(ap.get_animation_list()[0])
		return
	for child in root.get_children():
		_start_looping_animations(child)
```

### sokoban_3d/scenes/main.tscn
```text
[gd_scene load_steps=7 format=3]

[ext_resource type="Script" path="res://scripts/main.gd" id="1_main"]
[ext_resource type="Script" path="res://scripts/camera_rig.gd" id="2_cam"]
[ext_resource type="Script" path="res://scripts/sound_synth.gd" id="3_sfx"]

[sub_resource type="ProceduralSkyMaterial" id="ProceduralSkyMaterial_1"]
sky_top_color = Color(0.11, 0.16, 0.24, 1)
sky_horizon_color = Color(0.22, 0.28, 0.36, 1)
ground_bottom_color = Color(0.08, 0.10, 0.14, 1)
ground_horizon_color = Color(0.16, 0.20, 0.26, 1)

[sub_resource type="Sky" id="Sky_1"]
sky_material = SubResource("ProceduralSkyMaterial_1")

[sub_resource type="Environment" id="Environment_1"]
background_mode = 2
sky = SubResource("Sky_1")
ambient_light_source = 3
ambient_light_color = Color(0.42, 0.48, 0.58, 1)
ambient_light_energy = 0.65
tonemap_mode = 2
glow_enabled = true
glow_intensity = 0.45
glow_bloom = 0.15

[node name="Main" type="Node3D"]
script = ExtResource("1_main")

[node name="WorldEnvironment" type="WorldEnvironment" parent="."]
environment = SubResource("Environment_1")

[node name="DirectionalLight3D" type="DirectionalLight3D" parent="."]
transform = Transform3D(0.766044, -0.454519, 0.454519, 0, 0.707107, 0.707107, -0.642788, -0.541675, 0.541675, 10, 18, 10)
light_color = Color(1, 0.96, 0.9, 1)
light_energy = 1.35
shadow_enabled = true
directional_shadow_max_distance = 60.0

[node name="FillLight3D" type="DirectionalLight3D" parent="."]
transform = Transform3D(-0.766044, 0.321394, -0.55667, 0, 0.866025, 0.5, 0.642788, 0.383022, -0.663414, -8, 10, -8)
light_color = Color(0.55, 0.75, 1, 1)
light_energy = 0.45

[node name="BoardRoot" type="Node3D" parent="."]

[node name="CameraRig" type="Node3D" parent="."]
script = ExtResource("2_cam")

[node name="Camera3D" type="Camera3D" parent="CameraRig"]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 16)
fov = 45.0

[node name="SoundSynth" type="Node" parent="."]
script = ExtResource("3_sfx")

[node name="HUD" type="CanvasLayer" parent="."]

[node name="TopBar" type="PanelContainer" parent="HUD"]
anchors_preset = 10
anchor_right = 1.0
offset_left = 16.0
offset_top = 14.0
offset_right = -16.0
offset_bottom = 66.0
grow_horizontal = 2

[node name="Margin" type="MarginContainer" parent="HUD/TopBar"]
layout_mode = 2
theme_override_constants/margin_left = 14
theme_override_constants/margin_top = 8
theme_override_constants/margin_right = 14
theme_override_constants/margin_bottom = 8

[node name="HBox" type="HBoxContainer" parent="HUD/TopBar/Margin"]
layout_mode = 2
theme_override_constants/separation = 16

[node name="LevelLabel" type="Label" parent="HUD/TopBar/Margin/HBox"]
unique_name_in_owner = true
layout_mode = 2
theme_override_font_sizes/font_size = 18
text = "Level 1/7: Warehouse Intro"

[node name="Spacer1" type="Control" parent="HUD/TopBar/Margin/HBox"]
layout_mode = 2
size_flags_horizontal = 3

[node name="MovesLabel" type="Label" parent="HUD/TopBar/Margin/HBox"]
unique_name_in_owner = true
layout_mode = 2
theme_override_font_sizes/font_size = 16
text = "Moves: 0 (Par 9)"

[node name="PushesLabel" type="Label" parent="HUD/TopBar/Margin/HBox"]
unique_name_in_owner = true
layout_mode = 2
theme_override_font_sizes/font_size = 16
text = "Pushes: 0"

[node name="GoalsLabel" type="Label" parent="HUD/TopBar/Margin/HBox"]
unique_name_in_owner = true
layout_mode = 2
theme_override_colors/font_color = Color(0.3, 0.95, 1, 1)
theme_override_font_sizes/font_size = 16
text = "Docked: 0 / 3"

[node name="Spacer2" type="Control" parent="HUD/TopBar/Margin/HBox"]
layout_mode = 2
size_flags_horizontal = 3

[node name="PrevLevelButton" type="Button" parent="HUD/TopBar/Margin/HBox"]
unique_name_in_owner = true
layout_mode = 2
focus_mode = 0
text = "< Prev (P)"

[node name="NextLevelButton" type="Button" parent="HUD/TopBar/Margin/HBox"]
unique_name_in_owner = true
layout_mode = 2
focus_mode = 0
text = "Next (N) >"

[node name="UndoButton" type="Button" parent="HUD/TopBar/Margin/HBox"]
unique_name_in_owner = true
layout_mode = 2
focus_mode = 0
text = "Undo (Z)"

[node name="RestartButton" type="Button" parent="HUD/TopBar/Margin/HBox"]
unique_name_in_owner = true
layout_mode = 2
focus_mode = 0
text = "Restart (R)"

[node name="CameraButton" type="Button" parent="HUD/TopBar/Margin/HBox"]
unique_name_in_owner = true
layout_mode = 2
focus_mode = 0
text = "Cam: Iso (C)"

[node name="MuteButton" type="Button" parent="HUD/TopBar/Margin/HBox"]
unique_name_in_owner = true
layout_mode = 2
focus_mode = 0
text = "Sound: On"

[node name="BottomHint" type="PanelContainer" parent="HUD"]
anchors_preset = 7
anchor_left = 0.5
anchor_top = 1.0
anchor_right = 0.5
anchor_bottom = 1.0
offset_left = -360.0
offset_top = -48.0
offset_right = 360.0
offset_bottom = -12.0
grow_horizontal = 2
grow_vertical = 0

[node name="HintLabel" type="Label" parent="HUD/BottomHint"]
layout_mode = 2
theme_override_colors/font_color = Color(0.82, 0.87, 0.94, 1)
theme_override_font_sizes/font_size = 14
text = "Move: WASD / Arrows / Swipe / D-Pad   |   Undo: Z   |   Restart: R   |   Cam: C"
horizontal_alignment = 1
vertical_alignment = 1

[node name="TouchDPad" type="GridContainer" parent="HUD"]
modulate = Color(1, 1, 1, 0.85)
anchors_preset = 3
anchor_left = 1.0
anchor_top = 1.0
anchor_right = 1.0
anchor_bottom = 1.0
offset_left = -196.0
offset_top = -196.0
offset_right = -20.0
offset_bottom = -20.0
grow_horizontal = 0
grow_vertical = 0
theme_override_constants/h_separation = 6
theme_override_constants/v_separation = 6
columns = 3

[node name="EmptyTL" type="Control" parent="HUD/TouchDPad"]
custom_minimum_size = Vector2(54, 54)
layout_mode = 2

[node name="TouchUpButton" type="Button" parent="HUD/TouchDPad"]
unique_name_in_owner = true
custom_minimum_size = Vector2(54, 54)
layout_mode = 2
focus_mode = 0
theme_override_font_sizes/font_size = 20
text = "^"

[node name="EmptyTR" type="Control" parent="HUD/TouchDPad"]
custom_minimum_size = Vector2(54, 54)
layout_mode = 2

[node name="TouchLeftButton" type="Button" parent="HUD/TouchDPad"]
unique_name_in_owner = true
custom_minimum_size = Vector2(54, 54)
layout_mode = 2
focus_mode = 0
theme_override_font_sizes/font_size = 20
text = "<"

[node name="EmptyCenter" type="Control" parent="HUD/TouchDPad"]
custom_minimum_size = Vector2(54, 54)
layout_mode = 2

[node name="TouchRightButton" type="Button" parent="HUD/TouchDPad"]
unique_name_in_owner = true
custom_minimum_size = Vector2(54, 54)
layout_mode = 2
focus_mode = 0
theme_override_font_sizes/font_size = 20
text = ">"

[node name="EmptyBL" type="Control" parent="HUD/TouchDPad"]
custom_minimum_size = Vector2(54, 54)
layout_mode = 2

[node name="TouchDownButton" type="Button" parent="HUD/TouchDPad"]
unique_name_in_owner = true
custom_minimum_size = Vector2(54, 54)
layout_mode = 2
focus_mode = 0
theme_override_font_sizes/font_size = 20
text = "v"

[node name="EmptyBR" type="Control" parent="HUD/TouchDPad"]
custom_minimum_size = Vector2(54, 54)
layout_mode = 2

[node name="WinOverlay" type="CenterContainer" parent="HUD"]
unique_name_in_owner = true
visible = false
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2

[node name="WinPanel" type="PanelContainer" parent="HUD/WinOverlay"]
custom_minimum_size = Vector2(440, 220)
layout_mode = 2

[node name="Margin" type="MarginContainer" parent="HUD/WinOverlay/WinPanel"]
layout_mode = 2
theme_override_constants/margin_left = 28
theme_override_constants/margin_top = 24
theme_override_constants/margin_right = 28
theme_override_constants/margin_bottom = 24

[node name="VBox" type="VBoxContainer" parent="HUD/WinOverlay/WinPanel/Margin"]
layout_mode = 2
theme_override_constants/separation = 16
alignment = 1

[node name="WinTitle" type="Label" parent="HUD/WinOverlay/WinPanel/Margin/VBox"]
unique_name_in_owner = true
layout_mode = 2
theme_override_colors/font_color = Color(0.25, 0.95, 0.6, 1)
theme_override_font_sizes/font_size = 28
text = "SHIFT COMPLETE!"
horizontal_alignment = 1

[node name="WinStats" type="Label" parent="HUD/WinOverlay/WinPanel/Margin/VBox"]
unique_name_in_owner = true
layout_mode = 2
theme_override_font_sizes/font_size = 18
text = "Moves: 9 | Pushes: 3"
horizontal_alignment = 1

[node name="Buttons" type="HBoxContainer" parent="HUD/WinOverlay/WinPanel/Margin/VBox"]
layout_mode = 2
theme_override_constants/separation = 16
alignment = 1

[node name="ReplayWinButton" type="Button" parent="HUD/WinOverlay/WinPanel/Margin/VBox/Buttons"]
unique_name_in_owner = true
custom_minimum_size = Vector2(140, 40)
layout_mode = 2
focus_mode = 0
text = "Replay (R)"

[node name="NextWinButton" type="Button" parent="HUD/WinOverlay/WinPanel/Margin/VBox/Buttons"]
unique_name_in_owner = true
custom_minimum_size = Vector2(160, 40)
layout_mode = 2
focus_mode = 0
text = "Next Level (N)"
```
