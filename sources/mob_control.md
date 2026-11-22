### mob_control/project.godot
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

config/name="Mob Control"
run/main_scene="res://scenes/main_game.tscn"
config/features=PackedStringArray("4.2", "Forward Plus")

[display]

window/size/viewport_width=1280
window/size/viewport_height=720
window/stretch/mode="canvas_items"
window/stretch/aspect="expand"

[editor_plugins]

enabled=PackedStringArray("res://addons/scad_importer/plugin.cfg")

[input]

move_left={
"deadzone": 0.2,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":65,"physical_keycode":0,"key_label":0,"unicode":97,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":4194319,"physical_keycode":0,"key_label":0,"unicode":0,"echo":false,"script":null)
]
}
move_right={
"deadzone": 0.2,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":68,"physical_keycode":0,"key_label":0,"unicode":100,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":4194321,"physical_keycode":0,"key_label":0,"unicode":0,"echo":false,"script":null)
]
}
fire={
"deadzone": 0.2,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":32,"physical_keycode":0,"key_label":0,"unicode":32,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":87,"physical_keycode":0,"key_label":0,"unicode":119,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":4194320,"physical_keycode":0,"key_label":0,"unicode":0,"echo":false,"script":null)
]
}
```

### mob_control/addons/scad_importer/plugin.cfg
```text
[plugin]

name="OpenSCAD GLTF Importer"
description="Imports .scad files directly as 3D scenes using scad-gltf"
author="scad-gltf"
version="0.2"
script="scad_plugin.gd"

```

### mob_control/addons/scad_importer/scad_importer.gd
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

### mob_control/addons/scad_importer/scad_plugin.gd
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

### mob_control/models/mob_cannon.scad
```openscad
// Mobile Ally Cannon (~1.6m wide, ~1.4m tall, facing +Y Forward)
$fn = 24;
$asa = 35.0;

anim_data = [
  ["Idle", [
    ["BarrelBone", [
      [0.0, [0, 0, 0], [0, 0.15, 0.72]],
      [1.0, [0, 0, 0], [0, 0.15, 0.72]]
    ]],
    ["CoreRingBone", [
      [0.0, [0, 0, 0]],
      [0.5, [0, 90, 0]],
      [1.0, [0, 180, 0]],
      [1.5, [0, 270, 0]],
      [2.0, [0, 360, 0]]
    ]]
  ]],
  ["Fire", [
    ["BarrelBone", [
      [0.0, [0, 0, 0], [0, 0.15, 0.72]],
      [0.05, [0, 0, 0], [0, -0.14, 0.72]],
      [0.18, [0, 0, 0], [0, 0.15, 0.72]]
    ]],
    ["CoreRingBone", [
      [0.0, [0, 0, 0]],
      [0.09, [0, 90, 0]],
      [0.18, [0, 180, 0]]
    ]]
  ]]
];

module wheel(x_pos, y_pos) {
  translate([x_pos, y_pos, 0.24])
  rotate([0, 90, 0]) {
    color([0.12, 0.13, 0.16], roughness=0.85, metalness=0.1, $asa=30.0)
      cylinder(h=0.18, r=0.24, center=true);
    color([0.75, 0.80, 0.88], roughness=0.25, metalness=0.9, $asa=30.0)
      cylinder(h=0.20, r=0.12, center=true);
  }
}

armature(animations=anim_data) {
  bone(name="ChassisBone", t=[0, 0, 0], r=[0, 0, 0]) {
    // 4 Wheels (Left is -X, Right is +X)
    wheel(-0.68,  0.42);
    wheel( 0.68,  0.42);
    wheel(-0.68, -0.42);
    wheel( 0.68, -0.42);

    // Axles
    color([0.35, 0.38, 0.44], roughness=0.35, metalness=0.85) {
      translate([0,  0.42, 0.24]) rotate([0, 90, 0]) cylinder(h=1.28, r=0.06, center=true);
      translate([0, -0.42, 0.24]) rotate([0, 90, 0]) cylinder(h=1.28, r=0.06, center=true);
    }

    // Main armored carriage
    color([0.10, 0.36, 0.92], roughness=0.28, metalness=0.55) {
      translate([0, 0, 0.34]) cube([1.14, 1.15, 0.28], center=true);
    }

    // Front bumper & glowing guide rails
    color([0.92, 0.95, 1.0], roughness=0.2, metalness=0.7) {
      translate([-0.46, 0.18, 0.56]) cube([0.16, 0.95, 0.22], center=true);
      translate([ 0.46, 0.18, 0.56]) cube([0.16, 0.95, 0.22], center=true);
    }

    // Rear power pack
    color([0.18, 0.22, 0.28], roughness=0.45, metalness=0.75) {
      translate([0, -0.48, 0.62]) cube([0.78, 0.28, 0.44], center=true);
    }

    // Recoiling Cannon Barrel
    bone(name="BarrelBone", t=[0, 0.15, 0.72], r=[0, 0, 0]) {
      // Main launch tube facing +Y
      color([0.08, 0.48, 0.98], roughness=0.22, metalness=0.75, $asa=45.0)
        rotate([-90, 0, 0])
          cylinder(h=1.05, r1=0.32, r2=0.27, center=true);

      // Flared muzzle collar
      color([0.92, 0.95, 1.0], roughness=0.18, metalness=0.88, $asa=45.0)
        translate([0, 0.52, 0])
        rotate([-90, 0, 0])
          cylinder(h=0.16, r1=0.30, r2=0.35, center=true);

      // Glowing muzzle inner core
      color([0.2, 0.9, 1.0], roughness=0.2, emissive=[0.15, 0.85, 1.0], emissiveIntensity=2.4, unlit=true)
        translate([0, 0.58, 0])
        rotate([-90, 0, 0])
          cylinder(h=0.06, r=0.21, center=true);

      // Top sight fin
      color([1.0, 0.78, 0.12], roughness=0.3, metalness=0.4)
        translate([0, 0.05, 0.34]) cube([0.10, 0.62, 0.14], center=true);

      // Rotating Energy Ring around Barrel
      bone(name="CoreRingBone", t=[0, -0.05, 0], r=[0, 0, 0]) {
        for (a = [0 : 90 : 270]) {
          rotate([0, a, 0])
          translate([0.34, 0, 0])
          color([0.2, 0.95, 1.0], roughness=0.2, emissive=[0.2, 0.9, 1.0], emissiveIntensity=2.0)
            cube([0.08, 0.22, 0.14], center=true);
        }
      }
    }
  }
}
```

### mob_control/models/ally_mob.scad
```openscad
// Ally Mob Runner (~0.95m tall, facing +Y Forward)
$fn = 20;
$asa = 35.0;

anim_data = [
  ["Run", [
    ["BodyBone", [
      [0.00, [-6, 0, 0], [0, 0, 0.42]],
      [0.12, [-8, 0, 0], [0, 0, 0.47]],
      [0.24, [-6, 0, 0], [0, 0, 0.42]],
      [0.36, [-8, 0, 0], [0, 0, 0.47]],
      [0.48, [-6, 0, 0], [0, 0, 0.42]]
    ]],
    ["LeftArmBone", [
      [0.00, [ 40, 0, 0]],
      [0.24, [-40, 0, 0]],
      [0.48, [ 40, 0, 0]]
    ]],
    ["RightArmBone", [
      [0.00, [-40, 0, 0]],
      [0.24, [ 40, 0, 0]],
      [0.48, [-40, 0, 0]]
    ]],
    ["LeftLegBone", [
      [0.00, [-42, 0, 0]],
      [0.24, [ 42, 0, 0]],
      [0.48, [-42, 0, 0]]
    ]],
    ["RightLegBone", [
      [0.00, [ 42, 0, 0]],
      [0.24, [-42, 0, 0]],
      [0.48, [ 42, 0, 0]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="BodyBone", t=[0, 0, 0.42], r=[-6, 0, 0]) {
    // Torso
    color([0.08, 0.40, 0.96], roughness=0.30, metalness=0.25, $asa=45.0)
      scale([1.0, 0.8, 1.15]) sphere(r=0.20);

    // Chest Emblem
    color([0.85, 0.96, 1.0], roughness=0.2, emissive=[0.2, 0.8, 1.0], emissiveIntensity=1.2)
      translate([0, 0.14, 0.04])
        cube([0.14, 0.05, 0.14], center=true);

    // Head + Helmet
    bone(name="HeadBone", t=[0, 0, 0.31], r=[0, 0, 0]) {
      color([0.10, 0.46, 0.98], roughness=0.28, metalness=0.2, $asa=45.0)
        sphere(r=0.17);

      // Glowing Visor facing +Y
      color([0.3, 0.95, 1.0], roughness=0.15, emissive=[0.2, 0.9, 1.0], emissiveIntensity=2.2, unlit=true)
        translate([0, 0.13, 0.02])
          cube([0.20, 0.08, 0.07], center=true);
    }

    // Left Arm (-X)
    bone(name="LeftArmBone", t=[-0.23, 0, 0.12], r=[0, 0, 0]) {
      color([0.88, 0.93, 1.0], roughness=0.3, metalness=0.35, $asa=35.0)
        translate([0, 0, -0.11])
          cylinder(h=0.24, r1=0.055, r2=0.065, center=true);
    }

    // Right Arm (+X)
    bone(name="RightArmBone", t=[0.23, 0, 0.12], r=[0, 0, 0]) {
      color([0.88, 0.93, 1.0], roughness=0.3, metalness=0.35, $asa=35.0)
        translate([0, 0, -0.11])
          cylinder(h=0.24, r1=0.055, r2=0.065, center=true);
    }

    // Left Leg (-X)
    bone(name="LeftLegBone", t=[-0.10, 0, -0.15], r=[0, 0, 0]) {
      color([0.06, 0.24, 0.65], roughness=0.4, metalness=0.2, $asa=35.0)
        translate([0, 0, -0.12])
          cylinder(h=0.24, r1=0.065, r2=0.07, center=true);
      color([0.90, 0.94, 1.0], roughness=0.35, metalness=0.3)
        translate([0, 0.03, -0.24])
          cube([0.13, 0.17, 0.07], center=true);
    }

    // Right Leg (+X)
    bone(name="RightLegBone", t=[0.10, 0, -0.15], r=[0, 0, 0]) {
      color([0.06, 0.24, 0.65], roughness=0.4, metalness=0.2, $asa=35.0)
        translate([0, 0, -0.12])
          cylinder(h=0.24, r1=0.065, r2=0.07, center=true);
      color([0.90, 0.94, 1.0], roughness=0.35, metalness=0.3)
        translate([0, 0.03, -0.24])
          cube([0.13, 0.17, 0.07], center=true);
    }
  }
}
```

### mob_control/models/ally_giant.scad
```openscad
// Ally Giant Juggernaut (~1.85m tall, facing +Y Forward)
$fn = 24;
$asa = 35.0;

anim_data = [
  ["Run", [
    ["BodyBone", [
      [0.00, [-5, 0, 0], [0, 0, 0.85]],
      [0.20, [-7, 0, 0], [0, 0, 0.92]],
      [0.40, [-5, 0, 0], [0, 0, 0.85]],
      [0.60, [-7, 0, 0], [0, 0, 0.92]],
      [0.80, [-5, 0, 0], [0, 0, 0.85]]
    ]],
    ["LeftArmBone", [
      [0.00, [ 30, 0, 0]],
      [0.40, [-30, 0, 0]],
      [0.80, [ 30, 0, 0]]
    ]],
    ["RightArmBone", [
      [0.00, [-30, 0, 0]],
      [0.40, [ 30, 0, 0]],
      [0.80, [-30, 0, 0]]
    ]],
    ["LeftLegBone", [
      [0.00, [-32, 0, 0]],
      [0.40, [ 32, 0, 0]],
      [0.80, [-32, 0, 0]]
    ]],
    ["RightLegBone", [
      [0.00, [ 32, 0, 0]],
      [0.40, [-32, 0, 0]],
      [0.80, [ 32, 0, 0]]
    ]],
    ["CapeTipBone", [
      [0.00, [-18, 0, 0]],
      [0.20, [-28, 0, 0]],
      [0.40, [-18, 0, 0]],
      [0.60, [-28, 0, 0]],
      [0.80, [-18, 0, 0]]
    ]]
  ]]
];

armature(animations=anim_data) {
  // Skinned Heroic Energy Cape (outside bone hierarchy, cleared behind torso)
  skin(bones=["BodyBone", "CapeTipBone"], weights=function(x, y, z) [
    min(1.0, max(0.0, (z - 0.34) / 0.92)),
    1.0 - min(1.0, max(0.0, (z - 0.34) / 0.92))
  ]) {
    color([1.0, 0.76, 0.10], roughness=0.35, emissive=[0.45, 0.25, 0.0], emissiveIntensity=0.6)
      translate([0, -0.44, 0.34])
        linear_extrude(height=0.92, slices=10)
          square([0.66, 0.05], center=true);
  }

  bone(name="BodyBone", t=[0, 0, 0.85], r=[-5, 0, 0]) {
    // Heavy Armored Torso
    color([0.08, 0.38, 0.94], roughness=0.28, metalness=0.50, $asa=40.0)
      scale([1.22, 0.92, 1.22]) sphere(r=0.35);

    // Golden Chest Armor Plate
    color([1.0, 0.80, 0.16], roughness=0.22, metalness=0.85)
      translate([0, 0.24, 0.06]) cube([0.46, 0.22, 0.36], center=true);

    // Glowing Reactor Core
    color([0.25, 0.95, 1.0], roughness=0.15, emissive=[0.2, 0.9, 1.0], emissiveIntensity=2.8, unlit=true)
      translate([0, 0.36, 0.06])
      rotate([90, 0, 0])
        cylinder(h=0.06, r=0.11, center=true);

    // Helmeted Head
    bone(name="HeadBone", t=[0, 0, 0.56], r=[0, 0, 0]) {
      color([0.10, 0.44, 0.96], roughness=0.26, metalness=0.45, $asa=40.0)
        sphere(r=0.24);

      // Gold Crown Crest
      color([1.0, 0.82, 0.18], roughness=0.2, metalness=0.85)
        translate([0, 0.04, 0.20]) cube([0.10, 0.28, 0.16], center=true);

      // Visor facing +Y
      color([0.3, 0.95, 1.0], roughness=0.15, emissive=[0.2, 0.9, 1.0], emissiveIntensity=2.5, unlit=true)
        translate([0, 0.19, 0.03]) cube([0.28, 0.09, 0.09], center=true);
    }

    // Left Arm + Pauldron + Fist (-X)
    bone(name="LeftArmBone", t=[-0.48, 0, 0.22], r=[0, 0, 0]) {
      color([1.0, 0.80, 0.16], roughness=0.22, metalness=0.85, $asa=35.0)
        sphere(r=0.17);
      color([0.10, 0.44, 0.96], roughness=0.28, metalness=0.5, $asa=35.0)
        translate([0, 0, -0.25]) cylinder(h=0.44, r1=0.12, r2=0.13, center=true);
      color([0.92, 0.96, 1.0], roughness=0.25, metalness=0.7, $asa=30.0)
        translate([0, 0, -0.49]) sphere(r=0.14);
    }

    // Right Arm + Pauldron + Fist (+X)
    bone(name="RightArmBone", t=[0.48, 0, 0.22], r=[0, 0, 0]) {
      color([1.0, 0.80, 0.16], roughness=0.22, metalness=0.85, $asa=35.0)
        sphere(r=0.17);
      color([0.10, 0.44, 0.96], roughness=0.28, metalness=0.5, $asa=35.0)
        translate([0, 0, -0.25]) cylinder(h=0.44, r1=0.12, r2=0.13, center=true);
      color([0.92, 0.96, 1.0], roughness=0.25, metalness=0.7, $asa=30.0)
        translate([0, 0, -0.49]) sphere(r=0.14);
    }

    // Left Leg + Boot (-X)
    bone(name="LeftLegBone", t=[-0.21, 0, -0.30], r=[0, 0, 0]) {
      color([0.06, 0.22, 0.58], roughness=0.35, metalness=0.4, $asa=35.0)
        translate([0, 0, -0.25]) cylinder(h=0.50, r1=0.14, r2=0.13, center=true);
      color([1.0, 0.80, 0.16], roughness=0.25, metalness=0.8)
        translate([0, 0.04, -0.50]) cube([0.24, 0.30, 0.12], center=true);
    }

    // Right Leg + Boot (+X)
    bone(name="RightLegBone", t=[0.21, 0, -0.30], r=[0, 0, 0]) {
      color([0.06, 0.22, 0.58], roughness=0.35, metalness=0.4, $asa=35.0)
        translate([0, 0, -0.25]) cylinder(h=0.50, r1=0.14, r2=0.13, center=true);
      color([1.0, 0.80, 0.16], roughness=0.25, metalness=0.8)
        translate([0, 0.04, -0.50]) cube([0.24, 0.30, 0.12], center=true);
    }

    // Cape Anchor Bone
    bone(name="CapeTipBone", t=[0, -0.44, -0.51], r=[-18, 0, 0]) {}
  }
}
```

### mob_control/models/enemy_mob.scad
```openscad
// Enemy Crimson Raider (~0.98m tall, facing +Y Forward)
$fn = 20;
$asa = 35.0;

anim_data = [
  ["Run", [
    ["BodyBone", [
      [0.00, [-6, 0, 0], [0, 0, 0.43]],
      [0.12, [-8, 0, 0], [0, 0, 0.48]],
      [0.24, [-6, 0, 0], [0, 0, 0.43]],
      [0.36, [-8, 0, 0], [0, 0, 0.48]],
      [0.48, [-6, 0, 0], [0, 0, 0.43]]
    ]],
    ["LeftArmBone", [
      [0.00, [ 42, 0, 0]],
      [0.24, [-42, 0, 0]],
      [0.48, [ 42, 0, 0]]
    ]],
    ["RightArmBone", [
      [0.00, [-42, 0, 0]],
      [0.24, [ 42, 0, 0]],
      [0.48, [-42, 0, 0]]
    ]],
    ["LeftLegBone", [
      [0.00, [-44, 0, 0]],
      [0.24, [ 44, 0, 0]],
      [0.48, [-44, 0, 0]]
    ]],
    ["RightLegBone", [
      [0.00, [ 44, 0, 0]],
      [0.24, [-44, 0, 0]],
      [0.48, [ 44, 0, 0]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="BodyBone", t=[0, 0, 0.43], r=[-6, 0, 0]) {
    // Deep Crimson Torso
    color([0.76, 0.06, 0.10], roughness=0.36, metalness=0.25, $asa=40.0)
      scale([1.05, 0.85, 1.15]) sphere(r=0.20);

    // Dark Gunmetal Chestplate
    color([0.11, 0.10, 0.14], roughness=0.45, metalness=0.70)
      translate([0, 0.12, 0.04]) cube([0.22, 0.10, 0.20], center=true);

    // Horned Head
    bone(name="HeadBone", t=[0, 0, 0.31], r=[0, 0, 0]) {
      color([0.78, 0.08, 0.12], roughness=0.32, metalness=0.2, $asa=40.0)
        sphere(r=0.17);

      // Left & Right Horns
      color([0.12, 0.10, 0.12], roughness=0.3, metalness=0.65, $asa=30.0) {
        translate([-0.12, 0.02, 0.16]) rotate([15, -24, 0]) cylinder(h=0.18, r1=0.045, r2=0.005, center=true);
        translate([ 0.12, 0.02, 0.16]) rotate([15,  24, 0]) cylinder(h=0.18, r1=0.045, r2=0.005, center=true);
      }

      // Glowing Red-Orange Eyes facing +Y
      color([1.0, 0.35, 0.05], roughness=0.15, emissive=[1.0, 0.25, 0.05], emissiveIntensity=2.6, unlit=true) {
        translate([-0.06, 0.14, 0.03]) cube([0.07, 0.06, 0.05], center=true);
        translate([ 0.06, 0.14, 0.03]) cube([0.07, 0.06, 0.05], center=true);
      }
    }

    // Left Arm (-X)
    bone(name="LeftArmBone", t=[-0.24, 0, 0.12], r=[0, 0, 0]) {
      color([0.14, 0.12, 0.16], roughness=0.4, metalness=0.55, $asa=30.0)
        translate([0, 0, -0.11])
          cylinder(h=0.24, r1=0.055, r2=0.068, center=true);
    }

    // Right Arm (+X)
    bone(name="RightArmBone", t=[0.24, 0, 0.12], r=[0, 0, 0]) {
      color([0.14, 0.12, 0.16], roughness=0.4, metalness=0.55, $asa=30.0)
        translate([0, 0, -0.11])
          cylinder(h=0.24, r1=0.055, r2=0.068, center=true);
    }

    // Left Leg (-X)
    bone(name="LeftLegBone", t=[-0.10, 0, -0.15], r=[0, 0, 0]) {
      color([0.38, 0.04, 0.08], roughness=0.45, metalness=0.25, $asa=30.0)
        translate([0, 0, -0.12])
          cylinder(h=0.24, r1=0.065, r2=0.072, center=true);
      color([0.12, 0.10, 0.14], roughness=0.4, metalness=0.5)
        translate([0, 0.03, -0.24])
          cube([0.13, 0.17, 0.07], center=true);
    }

    // Right Leg (+X)
    bone(name="RightLegBone", t=[0.10, 0, -0.15], r=[0, 0, 0]) {
      color([0.38, 0.04, 0.08], roughness=0.45, metalness=0.25, $asa=30.0)
        translate([0, 0, -0.12])
          cylinder(h=0.24, r1=0.065, r2=0.072, center=true);
      color([0.12, 0.10, 0.14], roughness=0.4, metalness=0.5)
        translate([0, 0.03, -0.24])
          cube([0.13, 0.17, 0.07], center=true);
    }
  }
}
```

### mob_control/models/enemy_castle.scad
```openscad
// Enemy Fortress Gatehouse (~5.4m wide, ~3.8m tall, facing +Y Forward)
$fn = 24;
$asa = 25.0;

anim_data = [
  ["Idle", [
    ["CrystalBone", [
      [0.0, [0, 0,   0], [0, 0, 3.25]],
      [0.5, [0, 0,  90], [0, 0, 3.40]],
      [1.0, [0, 0, 180], [0, 0, 3.25]],
      [1.5, [0, 0, 270], [0, 0, 3.40]],
      [2.0, [0, 0, 360], [0, 0, 3.25]]
    ]]
  ]]
];

module tower(x_pos) {
  translate([x_pos, 0, 0]) {
    // Tower base
    color([0.22, 0.20, 0.26], roughness=0.75, metalness=0.25)
      translate([0, 0, 1.45]) cylinder(h=2.9, r1=0.88, r2=0.78, center=true);

    // Crimson parapet ring
    color([0.82, 0.14, 0.18], roughness=0.4, metalness=0.3)
      translate([0, 0, 2.95]) cylinder(h=0.34, r=0.92, center=true);

    // Battlement teeth
    color([0.25, 0.22, 0.28], roughness=0.7, metalness=0.3)
      for (a = [0 : 60 : 300]) {
        rotate([0, 0, a])
        translate([0.74, 0, 3.26])
          cube([0.24, 0.30, 0.30], center=true);
      }
  }
}

armature(animations=anim_data) {
  bone(name="FortressBone", t=[0, 0, 0], r=[0, 0, 0]) {
    // Left and Right Flanking Towers
    tower(-2.05);
    tower( 2.05);

    // Main Fortress Wall Left & Right of Gate
    color([0.26, 0.24, 0.30], roughness=0.78, metalness=0.2) {
      translate([-1.15, 0, 1.15]) cube([1.15, 1.35, 2.30], center=true);
      translate([ 1.15, 0, 1.15]) cube([1.15, 1.35, 2.30], center=true);
      // Gate Lintel / Bridge
      translate([0, 0, 2.00]) cube([1.40, 1.40, 0.72], center=true);
    }

    // Back wall sealing the fortress
    color([0.16, 0.14, 0.18], roughness=0.85, metalness=0.2)
      translate([0, -0.45, 1.00]) cube([2.80, 0.45, 2.00], center=true);

    // Dark Glowing Spawn Portal inside Gate (+Y front)
    color([0.95, 0.12, 0.18], roughness=0.2, emissive=[0.95, 0.10, 0.16], emissiveIntensity=2.2, unlit=true)
      translate([0, 0.10, 0.85]) cube([1.10, 0.12, 1.65], center=true);

    // Crimson Trim & Crenellations along top wall
    color([0.85, 0.14, 0.20], roughness=0.35, metalness=0.35) {
      translate([0, 0.72, 2.25]) cube([3.20, 0.18, 0.22], center=true);
      for (x = [-1.2, -0.6, 0.0, 0.6, 1.2]) {
        translate([x, 0.60, 2.52]) cube([0.32, 0.28, 0.34], center=true);
      }
    }

    // Central Pedestal Spire
    color([0.20, 0.18, 0.24], roughness=0.6, metalness=0.4)
      translate([0, 0, 2.55]) cylinder(h=0.45, r1=0.55, r2=0.35, center=true);

    // Floating Spinning Crimson Power Crystal
    bone(name="CrystalBone", t=[0, 0, 3.25], r=[0, 0, 0]) {
      color([1.0, 0.15, 0.22], roughness=0.15, metalness=0.2, emissive=[1.0, 0.12, 0.20], emissiveIntensity=2.8, $asa=0.0) {
        translate([0, 0,  0.22]) cylinder(h=0.44, r1=0.30, r2=0.01, center=true, $fn=6);
        translate([0, 0, -0.22]) cylinder(h=0.44, r1=0.01, r2=0.30, center=true, $fn=6);
      }
    }
  }
}
```

### mob_control/models/multiplier_gate.scad
```openscad
// Energy Multiplier Gate (~3.0m wide, ~2.2m tall, facing +Y Forward)
$fn = 20;
$asa = 30.0;

anim_data = [
  ["Pulse", [
    ["BeaconLeftBone", [
      [0.0, [0, 0,   0]],
      [0.4, [0, 0,  90]],
      [0.8, [0, 0, 180]],
      [1.2, [0, 0, 270]],
      [1.6, [0, 0, 360]]
    ]],
    ["BeaconRightBone", [
      [0.0, [0, 0, 360]],
      [0.4, [0, 0, 270]],
      [0.8, [0, 0, 180]],
      [1.2, [0, 0,  90]],
      [1.6, [0, 0,   0]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="GateFrameBone", t=[0, 0, 0], r=[0, 0, 0]) {
    // Left & Right Pylon Bases
    color([0.15, 0.20, 0.28], roughness=0.4, metalness=0.75) {
      translate([-1.42, 0, 0.14]) cube([0.38, 0.44, 0.28], center=true);
      translate([ 1.42, 0, 0.14]) cube([0.38, 0.44, 0.28], center=true);
    }

    // Left & Right Vertical Columns
    color([0.85, 0.92, 1.0], roughness=0.22, metalness=0.8) {
      translate([-1.42, 0, 1.08]) cylinder(h=1.72, r=0.12, center=true);
      translate([ 1.42, 0, 1.08]) cylinder(h=1.72, r=0.12, center=true);
    }

    // Top Crossbar
    color([0.15, 0.20, 0.28], roughness=0.35, metalness=0.75)
      translate([0, 0, 2.02]) cube([3.12, 0.26, 0.20], center=true);

    // Holographic Energy Field
    color([0.10, 0.78, 1.0], alpha=0.35, roughness=0.1, emissive=[0.1, 0.75, 1.0], emissiveIntensity=2.0, unlit=true)
      translate([0, 0, 1.02]) cube([2.60, 0.08, 1.78], center=true);

    // Spinning Pylon Beacons
    bone(name="BeaconLeftBone", t=[-1.42, 0, 2.26], r=[0, 0, 0]) {
      color([0.2, 0.95, 1.0], roughness=0.15, emissive=[0.2, 0.9, 1.0], emissiveIntensity=2.5, $asa=0.0)
        rotate([45, 45, 0]) cube([0.20, 0.20, 0.20], center=true);
    }

    bone(name="BeaconRightBone", t=[1.42, 0, 2.26], r=[0, 0, 0]) {
      color([0.2, 0.95, 1.0], roughness=0.15, emissive=[0.2, 0.9, 1.0], emissiveIntensity=2.5, $asa=0.0)
        rotate([45, 45, 0]) cube([0.20, 0.20, 0.20], center=true);
    }
  }
}
```

### mob_control/models/arena_block.scad
```openscad
// Heavy Arena Obstacle Barricade (~2.4m wide, ~1.05m tall, facing +Y Forward)
$fn = 16;
$asa = 0.0;

// Main concrete/steel barrier
color([0.28, 0.30, 0.36], roughness=0.65, metalness=0.4)
  translate([0, 0, 0.45]) cube([2.35, 0.85, 0.90], center=true);

// Top hazard warning cap
color([0.95, 0.74, 0.12], roughness=0.35, metalness=0.25)
  translate([0, 0, 0.96]) cube([2.40, 0.90, 0.14], center=true);

// Dark bumper stripes
color([0.12, 0.13, 0.16], roughness=0.5, metalness=0.5)
  for (x = [-0.8, 0.0, 0.8]) {
    translate([x, 0, 0.97]) cube([0.36, 0.92, 0.15], center=true);
  }
```

### mob_control/scenes/mob_cannon.tscn
```text
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/cannon_controller.gd" id="1_script"]
[ext_resource type="PackedScene" path="res://models/mob_cannon.scad" id="2_model"]

[node name="MobCannon" type="Node3D"]
script = ExtResource("1_script")

[node name="Model" parent="." instance=ExtResource("2_model")]

[node name="MuzzlePoint" type="Marker3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.72, -1.05)
```

### mob_control/scenes/ally_mob.tscn
```text
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/mob_unit.gd" id="1_script"]
[ext_resource type="PackedScene" path="res://models/ally_mob.scad" id="2_model"]

[node name="AllyMob" type="Node3D"]
script = ExtResource("1_script")
is_ally = true
max_hp = 1
move_speed = 6.2
radius = 0.32
damage_to_castle = 1

[node name="Model" parent="." instance=ExtResource("2_model")]
```

### mob_control/scenes/ally_giant.tscn
```text
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/mob_unit.gd" id="1_script"]
[ext_resource type="PackedScene" path="res://models/ally_giant.scad" id="2_model"]

[node name="AllyGiant" type="Node3D"]
script = ExtResource("1_script")
is_ally = true
is_giant = true
max_hp = 6
move_speed = 4.6
radius = 0.58
damage_to_castle = 8

[node name="Model" parent="." instance=ExtResource("2_model")]
```

### mob_control/scenes/enemy_mob.tscn
```text
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/mob_unit.gd" id="1_script"]
[ext_resource type="PackedScene" path="res://models/enemy_mob.scad" id="2_model"]

[node name="EnemyMob" type="Node3D"]
script = ExtResource("1_script")
is_ally = false
max_hp = 1
move_speed = 4.6
radius = 0.32
damage_to_castle = 1

[node name="Model" parent="." instance=ExtResource("2_model")]
```

### mob_control/scenes/enemy_castle.tscn
```text
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/enemy_castle.gd" id="1_script"]
[ext_resource type="PackedScene" path="res://models/enemy_castle.scad" id="2_model"]

[node name="EnemyCastle" type="Node3D"]
script = ExtResource("1_script")

[node name="Model" parent="." instance=ExtResource("2_model")]
transform = Transform3D(-1, 0, -8.74228e-08, 0, 1, 0, 8.74228e-08, 0, -1, 0, 0, 0)

[node name="HpLabel" type="Label3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 4.35, 0.35)
billboard = 1
text = "50"
font_size = 110
outline_size = 18
modulate = Color(1, 0.92, 0.92, 1)
outline_modulate = Color(0.25, 0.02, 0.05, 1)
```

### mob_control/scenes/multiplier_gate.tscn
```text
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/multiplier_gate.gd" id="1_script"]
[ext_resource type="PackedScene" path="res://models/multiplier_gate.scad" id="2_model"]

[node name="MultiplierGate" type="Node3D"]
script = ExtResource("1_script")

[node name="Model" parent="." instance=ExtResource("2_model")]

[node name="Label3D" type="Label3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1.35, 0.18)
billboard = 1
text = "x2"
font_size = 128
outline_size = 20
modulate = Color(0.9, 1, 1, 1)
outline_modulate = Color(0.03, 0.18, 0.36, 1)
```

### mob_control/scenes/arena_block.tscn
```text
[gd_scene load_steps=2 format=3]

[ext_resource type="PackedScene" path="res://models/arena_block.scad" id="1_model"]

[node name="ArenaBlock" type="Node3D"]

[node name="Model" parent="." instance=ExtResource("1_model")]
```

### mob_control/scripts/sfx_manager.gd
```gdscript
extends Node

var _players: Array[AudioStreamPlayer] = []
var _streams: Dictionary = {}
var _next_idx: int = 0
var _last_played_ms: Dictionary = {}
var _cooldowns_ms: Dictionary = {
	"shoot": 40,
	"multiply": 45,
	"clash": 45,
	"castle_hit": 65,
}

func _ready():
	for i in range(14):
		var p = AudioStreamPlayer.new()
		p.volume_db = -9.0
		add_child(p)
		_players.append(p)

	_streams["shoot"] = _make_tone(420.0, 220.0, 0.08, 0.25)
	_streams["giant"] = _make_tone(180.0, 90.0, 0.26, 0.45)
	_streams["multiply"] = _make_tone(540.0, 880.0, 0.11, 0.24)
	_streams["clash"] = _make_tone(260.0, 120.0, 0.07, 0.22)
	_streams["castle_hit"] = _make_impact(260.0, 95.0, 0.15, 0.42)
	_streams["win"] = _make_chord([523.25, 659.25, 783.99], 0.45, 0.32)
	_streams["lose"] = _make_tone(240.0, 95.0, 0.45, 0.35)

func play(sound_name: String, pitch_jitter: float = 0.06):
	if not _streams.has(sound_name):
		return

	var now_ms := Time.get_ticks_msec()
	var cd := int(_cooldowns_ms.get(sound_name, 0))
	if cd > 0 and _last_played_ms.has(sound_name):
		if now_ms - int(_last_played_ms[sound_name]) < cd:
			return
	_last_played_ms[sound_name] = now_ms

	var chosen: AudioStreamPlayer = null
	for offset in range(_players.size()):
		var idx := (_next_idx + offset) % _players.size()
		if not _players[idx].playing:
			chosen = _players[idx]
			_next_idx = (idx + 1) % _players.size()
			break
	if chosen == null:
		chosen = _players[_next_idx]
		_next_idx = (_next_idx + 1) % _players.size()

	chosen.stream = _streams[sound_name]
	chosen.volume_db = -7.5 if sound_name == "castle_hit" else -9.0
	chosen.pitch_scale = randf_range(1.0 - pitch_jitter, 1.0 + pitch_jitter)
	chosen.play()

func _make_tone(f_start: float, f_end: float, duration: float, gain: float) -> AudioStreamWAV:
	var mix_rate := 22050
	var count := int(mix_rate * duration)
	var data := PackedByteArray()
	data.resize(count * 2)
	var phase := 0.0
	for i in range(count):
		var t := float(i) / float(count)
		var freq := lerpf(f_start, f_end, t)
		phase += TAU * freq / float(mix_rate)
		var env := (1.0 - t) * minf(1.0, t * 18.0)
		var sample := sin(phase) * env * gain
		var val := clampi(int(sample * 32767.0), -32768, 32767)
		data[i * 2] = val & 0xFF
		data[i * 2 + 1] = (val >> 8) & 0xFF

	var wav := AudioStreamWAV.new()
	wav.format = AudioStreamWAV.FORMAT_16_BITS
	wav.mix_rate = mix_rate
	wav.stereo = false
	wav.data = data
	return wav

func _make_impact(f_start: float, f_end: float, duration: float, gain: float) -> AudioStreamWAV:
	var mix_rate := 22050
	var count := int(mix_rate * duration)
	var data := PackedByteArray()
	data.resize(count * 2)
	var phase1 := 0.0
	var phase2 := 0.0
	var phase3 := 0.0
	var rng := RandomNumberGenerator.new()
	rng.seed = 42
	for i in range(count):
		var t := float(i) / float(count)
		var freq := lerpf(f_start, f_end, sqrt(t))
		phase1 += TAU * freq / float(mix_rate)
		phase2 += TAU * (freq * 2.15) / float(mix_rate)
		phase3 += TAU * (freq * 3.4) / float(mix_rate)
		var attack := minf(1.0, t * 90.0)
		var env := pow(1.0 - t, 1.4) * attack
		var click_env := maxf(0.0, 1.0 - t * 6.0)
		var tone := sin(phase1) * 0.55 + sin(phase2) * 0.30 + sin(phase3) * 0.15
		var crunch := rng.randf_range(-1.0, 1.0) * 0.35 * click_env
		var sample := clampf((tone * env + crunch) * gain, -1.0, 1.0)
		var val := clampi(int(sample * 32767.0), -32768, 32767)
		data[i * 2] = val & 0xFF
		data[i * 2 + 1] = (val >> 8) & 0xFF

	var wav := AudioStreamWAV.new()
	wav.format = AudioStreamWAV.FORMAT_16_BITS
	wav.mix_rate = mix_rate
	wav.stereo = false
	wav.data = data
	return wav

func _make_chord(freqs: Array, duration: float, gain: float) -> AudioStreamWAV:
	var mix_rate := 22050
	var count := int(mix_rate * duration)
	var data := PackedByteArray()
	data.resize(count * 2)
	for i in range(count):
		var t := float(i) / float(count)
		var env := (1.0 - t) * minf(1.0, t * 14.0)
		var sum := 0.0
		for f in freqs:
			sum += sin(TAU * float(f) * (float(i) / float(mix_rate)))
		var sample := (sum / float(freqs.size())) * env * gain
		var val := clampi(int(sample * 32767.0), -32768, 32767)
		data[i * 2] = val & 0xFF
		data[i * 2 + 1] = (val >> 8) & 0xFF

	var wav := AudioStreamWAV.new()
	wav.format = AudioStreamWAV.FORMAT_16_BITS
	wav.mix_rate = mix_rate
	wav.stereo = false
	wav.data = data
	return wav
```

### mob_control/scripts/mob_unit.gd
```gdscript
extends Node3D

@export var is_ally: bool = true
@export var is_giant: bool = false
@export var max_hp: int = 1
@export var move_speed: float = 6.0
@export var radius: float = 0.32
@export var damage_to_castle: int = 1

var hp: int = 1
var passed_gate_ids: Dictionary = {}
var _anim_player: AnimationPlayer = null
var _spawn_pop: float = 0.0

func _ready():
	hp = max_hp
	if not is_ally:
		# Enemy models face +Y in SCAD (-> -Z forward in Godot).
		# Rotate 180 deg around Y so local forward (-transform.basis.z) faces +Z toward player.
		rotation.y = PI

	_anim_player = _find_animation_player(self)
	if _anim_player:
		var anim_name = _pick_animation(_anim_player, "Run")
		if anim_name != "":
			var anim = _anim_player.get_animation(anim_name)
			if anim:
				anim.loop_mode = Animation.LOOP_LINEAR
			_anim_player.play(anim_name)
			_anim_player.seek(randf() * 0.4, true)

	scale = Vector3.ONE * 0.35
	_spawn_pop = 1.0

func _process(delta: float):
	if _spawn_pop > 0.0:
		_spawn_pop = maxf(0.0, _spawn_pop - delta * 5.0)
		var s = lerpf(1.0, 1.25, sin(_spawn_pop * PI))
		scale = Vector3.ONE * s
	else:
		scale = Vector3.ONE

func step_forward(delta: float):
	# Move along local Forward (-transform.basis.z)
	var forward_dir: Vector3 = -transform.basis.z
	forward_dir.y = 0.0
	forward_dir = forward_dir.normalized()
	position += forward_dir * move_speed * delta

func has_passed_gate(gate_id: int) -> bool:
	return passed_gate_ids.has(gate_id)

func mark_gate_passed(gate_id: int):
	passed_gate_ids[gate_id] = true

func _find_animation_player(node: Node) -> AnimationPlayer:
	if node is AnimationPlayer:
		return node
	for child in node.get_children():
		var found = _find_animation_player(child)
		if found:
			return found
	return null

func _pick_animation(ap: AnimationPlayer, preferred: String) -> String:
	var list = ap.get_animation_list()
	if list.is_empty():
		return ""
	for a in list:
		if String(a).to_lower() == preferred.to_lower():
			return String(a)
	return String(list[0])
```

### mob_control/scripts/cannon_controller.gd
```gdscript
extends Node3D

signal fired_mob(spawn_pos: Vector3, is_giant: bool)
signal charge_changed(current: int, max_charge: int)

@export var move_speed: float = 11.5
@export var arena_half_width: float = 6.2
@export var fire_interval: float = 0.16
@export var shots_per_giant: int = 12

var is_active: bool = false
var _fire_cooldown: float = 0.0
var _charge: int = 0
var _dragging_mouse: bool = false
var _target_x: float = 0.0
var _using_pointer: bool = false
var _anim_player: AnimationPlayer = null

@onready var muzzle_point: Marker3D = $MuzzlePoint

func _ready():
	_target_x = position.x
	_anim_player = _find_animation_player(self)
	_play_anim("Idle", true)
	charge_changed.emit(_charge, shots_per_giant)

func reset_cannon():
	position.x = 0.0
	_target_x = 0.0
	_charge = 0
	_fire_cooldown = 0.0
	_dragging_mouse = false
	_using_pointer = false
	charge_changed.emit(_charge, shots_per_giant)

func _unhandled_input(event: InputEvent):
	if not is_active:
		return
	if event is InputEventMouseButton and event.button_index == MOUSE_BUTTON_LEFT:
		_dragging_mouse = event.pressed
		if event.pressed:
			_update_pointer_target(event.position)
	elif event is InputEventMouseMotion and _dragging_mouse:
		_update_pointer_target(event.position)

func _update_pointer_target(screen_pos: Vector2):
	var cam = get_viewport().get_camera_3d()
	if not cam:
		return
	var origin = cam.project_ray_origin(screen_pos)
	var dir = cam.project_ray_normal(screen_pos)
	if absf(dir.y) > 0.001:
		var t = -origin.y / dir.y
		if t > 0.0:
			var hit = origin + dir * t
			_target_x = clampf(hit.x, -arena_half_width, arena_half_width)
			_using_pointer = true

func _process(delta: float):
	if not is_active:
		_dragging_mouse = false
		return

	# Lateral movement: Character Right is +X, Left is -X
	var axis: float = Input.get_axis("move_left", "move_right")
	if absf(axis) > 0.01:
		_using_pointer = false
		position.x += axis * move_speed * delta
	elif _using_pointer:
		position.x = move_toward(position.x, _target_x, move_speed * 1.45 * delta)

	position.x = clampf(position.x, -arena_half_width, arena_half_width)

	_fire_cooldown = maxf(0.0, _fire_cooldown - delta)

	var wants_fire: bool = Input.is_action_pressed("fire") or _dragging_mouse
	if wants_fire and _fire_cooldown <= 0.0:
		_shoot()
		_fire_cooldown = fire_interval

func _shoot():
	_play_anim("Fire", false)
	var spawn_pos: Vector3 = muzzle_point.global_position if muzzle_point else (global_position + Vector3(0, 0, -1.1))
	spawn_pos.y = 0.0

	_charge += 1
	if _charge >= shots_per_giant:
		_charge = 0
		charge_changed.emit(_charge, shots_per_giant)
		fired_mob.emit(spawn_pos, true)
	else:
		charge_changed.emit(_charge, shots_per_giant)
		fired_mob.emit(spawn_pos, false)

func _play_anim(preferred: String, loop: bool):
	if not _anim_player:
		return
	var list = _anim_player.get_animation_list()
	for a in list:
		if String(a).to_lower() == preferred.to_lower():
			var anim = _anim_player.get_animation(a)
			if anim:
				anim.loop_mode = Animation.LOOP_LINEAR if loop else Animation.LOOP_NONE
			_anim_player.stop()
			_anim_player.play(a)
			return

func _find_animation_player(node: Node) -> AnimationPlayer:
	if node is AnimationPlayer:
		return node
	for child in node.get_children():
		var found = _find_animation_player(child)
		if found:
			return found
	return null
```

### mob_control/scripts/multiplier_gate.gd
```gdscript
extends Node3D

@export var multiplier: int = 2
@export var gate_width: float = 2.65
@export var move_range: float = 0.0
@export var move_speed: float = 1.6
@export var phase_offset: float = 0.0

var base_x: float = 0.0
var gate_id: int = 0
var _time: float = 0.0
var _pulse_scale: float = 0.0

@onready var label_3d: Label3D = $Label3D

func setup(p_id: int, p_multiplier: int, p_pos: Vector3, p_move_range: float, p_move_speed: float, p_phase: float = 0.0):
	gate_id = p_id
	multiplier = p_multiplier
	position = p_pos
	base_x = p_pos.x
	move_range = p_move_range
	move_speed = p_move_speed
	phase_offset = p_phase
	_update_label()

func _ready():
	_update_label()
	var ap = _find_animation_player(self)
	if ap:
		var list = ap.get_animation_list()
		if not list.is_empty():
			var anim = ap.get_animation(list[0])
			if anim:
				anim.loop_mode = Animation.LOOP_LINEAR
			ap.play(list[0])

func _process(delta: float):
	if move_range > 0.01:
		_time += delta * move_speed
		position.x = base_x + sin(_time + phase_offset) * move_range

	if _pulse_scale > 0.0:
		_pulse_scale = maxf(0.0, _pulse_scale - delta * 6.0)
		var s = 1.0 + sin(_pulse_scale * PI) * 0.08
		scale = Vector3(s, s, 1.0)
	else:
		scale = Vector3.ONE

func trigger_flash():
	_pulse_scale = 1.0

func _update_label():
	if label_3d:
		label_3d.text = "x%d" % multiplier
		if multiplier >= 4:
			label_3d.modulate = Color(1.0, 0.88, 0.35, 1.0)
		elif multiplier == 3:
			label_3d.modulate = Color(0.55, 1.0, 0.72, 1.0)
		else:
			label_3d.modulate = Color(0.88, 0.98, 1.0, 1.0)

func _find_animation_player(node: Node) -> AnimationPlayer:
	if node is AnimationPlayer:
		return node
	for child in node.get_children():
		var found = _find_animation_player(child)
		if found:
			return found
	return null
```

### mob_control/scripts/enemy_castle.gd
```gdscript
extends Node3D

signal hp_changed(current_hp: int, max_hp: int)
signal destroyed

@export var max_hp: int = 50
var current_hp: int = 50
var _hit_flash: float = 0.0

@onready var hp_label: Label3D = $HpLabel

func _ready():
	var ap = _find_animation_player(self)
	if ap:
		var list = ap.get_animation_list()
		if not list.is_empty():
			var anim = ap.get_animation(list[0])
			if anim:
				anim.loop_mode = Animation.LOOP_LINEAR
			ap.play(list[0])
	_refresh_label()

func setup(p_max_hp: int):
	max_hp = p_max_hp
	current_hp = p_max_hp
	_refresh_label()
	hp_changed.emit(current_hp, max_hp)

func take_damage(amount: int) -> bool:
	if current_hp <= 0:
		return false
	current_hp = maxi(0, current_hp - amount)
	_hit_flash = 1.0
	_refresh_label()
	hp_changed.emit(current_hp, max_hp)
	if current_hp <= 0:
		destroyed.emit()
		return true
	return false

func _process(delta: float):
	if _hit_flash > 0.0:
		_hit_flash = maxf(0.0, _hit_flash - delta * 7.0)
		var bounce = 1.0 + sin(_hit_flash * PI) * 0.06
		scale = Vector3(bounce, bounce, bounce)
	else:
		scale = Vector3.ONE

func _refresh_label():
	if hp_label:
		hp_label.text = str(current_hp)

func _find_animation_player(node: Node) -> AnimationPlayer:
	if node is AnimationPlayer:
		return node
	for child in node.get_children():
		var found = _find_animation_player(child)
		if found:
			return found
	return null
```

### mob_control/scripts/main_game.gd
```gdscript
extends Node3D

const ARENA_HALF_WIDTH := 6.8
const DEFENSE_LINE_Z := 8.1
const CASTLE_HIT_Z := -12.9
const ENEMY_SPAWN_Z := -13.4
const MAX_ALLIES := 140
const MAX_ENEMIES := 90

var _ally_mob_scene: PackedScene
var _ally_giant_scene: PackedScene
var _enemy_mob_scene: PackedScene
var _gate_scene: PackedScene
var _block_scene: PackedScene

var _allies: Array = []
var _enemies: Array = []
var _gates: Array = []
var _blocks: Array = [] # Array of Dictionaries {pos: Vector3, half_w: float, half_d: float}
var _fx_particles: Array = [] # Array of Dictionaries {mesh: MeshInstance3D, vel: Vector3, life: float}

var _current_level: int = 1
var _game_state: String = "menu" # "menu", "playing", "won", "lost"
var _enemy_wave_timer: float = 0.0
var _enemy_wave_interval: float = 2.4
var _enemy_wave_count: int = 3
var _next_gate_id: int = 1

var _burst_mesh_res: SphereMesh

@onready var camera_3d: Camera3D = $Camera3D
@onready var cannon = $MobCannon
@onready var castle = $EnemyCastle
@onready var entities_root: Node3D = $Entities
@onready var level_objects_root: Node3D = $LevelObjects
@onready var sfx = $SfxManager

# HUD & Overlays
@onready var level_label: Label = $CanvasLayer/Hud/TopBarBg/TopBar/LevelLabel
@onready var castle_hp_bar: ProgressBar = $CanvasLayer/Hud/TopBarBg/TopBar/CastleHpBox/CastleHpBar
@onready var castle_hp_text: Label = $CanvasLayer/Hud/TopBarBg/TopBar/CastleHpBox/CastleHpText
@onready var mob_count_label: Label = $CanvasLayer/Hud/TopBarBg/TopBar/MobCountLabel
@onready var charge_bar: ProgressBar = $CanvasLayer/Hud/BottomBar/ChargeBox/ChargeBar
@onready var charge_label: Label = $CanvasLayer/Hud/BottomBar/ChargeBox/ChargeLabel
@onready var overlay_panel: PanelContainer = $CanvasLayer/OverlayPanel
@onready var overlay_title: Label = $CanvasLayer/OverlayPanel/VBox/TitleLabel
@onready var overlay_subtitle: Label = $CanvasLayer/OverlayPanel/VBox/SubtitleLabel
@onready var overlay_button: Button = $CanvasLayer/OverlayPanel/VBox/ActionButton

func _ready():
	camera_3d.look_at(Vector3(0.0, 0.0, 0.0), Vector3.UP)

	# Dynamically load scenes containing .scad assets inside _ready()
	_ally_mob_scene = load("res://scenes/ally_mob.tscn")
	_ally_giant_scene = load("res://scenes/ally_giant.tscn")
	_enemy_mob_scene = load("res://scenes/enemy_mob.tscn")
	_gate_scene = load("res://scenes/multiplier_gate.tscn")
	_block_scene = load("res://scenes/arena_block.tscn")

	_burst_mesh_res = SphereMesh.new()
	_burst_mesh_res.radius = 0.16
	_burst_mesh_res.height = 0.32
	_burst_mesh_res.radial_segments = 8
	_burst_mesh_res.rings = 4

	_build_arena_markings()

	cannon.fired_mob.connect(_on_cannon_fired)
	cannon.charge_changed.connect(_on_charge_changed)
	castle.hp_changed.connect(_on_castle_hp_changed)
	castle.destroyed.connect(_on_castle_destroyed)
	overlay_button.pressed.connect(_on_overlay_button_pressed)

	_show_menu()

func _build_arena_markings():
	var markings = Node3D.new()
	markings.name = "ArenaMarkings"
	add_child(markings)

	# Defense Line strip at Z = DEFENSE_LINE_Z
	var def_mat := StandardMaterial3D.new()
	def_mat.albedo_color = Color(0.18, 0.72, 1.0, 0.85)
	def_mat.emission_enabled = true
	def_mat.emission = Color(0.12, 0.65, 1.0)
	def_mat.emission_energy_multiplier = 1.4

	var def_mesh := BoxMesh.new()
	def_mesh.size = Vector3(14.2, 0.04, 0.18)
	var def_strip := MeshInstance3D.new()
	def_strip.mesh = def_mesh
	def_strip.material_override = def_mat
	def_strip.position = Vector3(0.0, 0.02, DEFENSE_LINE_Z)
	markings.add_child(def_strip)

	# Subtle lane dashes (continuing briefly past the castle)
	var dash_mat := StandardMaterial3D.new()
	dash_mat.albedo_color = Color(0.45, 0.50, 0.62, 0.45)
	dash_mat.transparency = BaseMaterial3D.TRANSPARENCY_ALPHA
	var dash_mesh := BoxMesh.new()
	dash_mesh.size = Vector3(0.12, 0.02, 1.2)
	for x in [-2.3, 2.3]:
		for z_step in range(-26, 8, 3):
			if z_step >= -15 and z_step <= -12:
				continue
			var dash := MeshInstance3D.new()
			dash.mesh = dash_mesh
			dash.material_override = dash_mat
			dash.position = Vector3(x, 0.015, float(z_step))
			markings.add_child(dash)

func _show_menu():
	_game_state = "menu"
	cannon.is_active = false
	_load_level(1)
	overlay_title.text = "MOB CONTROL"
	overlay_subtitle.text = "Slide Cannon: A / D or Drag Mouse\nFire Mobs: Hold SPACE or Left-Click\nAim through Multiplier Gates to overwhelm the Enemy Fortress!"
	overlay_button.text = "PLAY LEVEL 1"
	overlay_panel.visible = true

func _start_level(lvl: int):
	_current_level = lvl
	_load_level(lvl)
	_game_state = "playing"
	cannon.is_active = true
	overlay_panel.visible = false
	_enemy_wave_timer = 1.0

func _load_level(lvl: int):
	_clear_entities_and_level()
	cannon.reset_cannon()

	level_label.text = "LEVEL %d" % lvl

	var castle_hp: int = 40 + (lvl - 1) * 28
	castle.setup(castle_hp)

	_enemy_wave_interval = maxf(1.35, 2.5 - float(lvl - 1) * 0.2)
	_enemy_wave_count = mini(7, 2 + int(lvl / 2))

	# Level layouts
	var layout_idx: int = ((lvl - 1) % 5) + 1
	match layout_idx:
		1:
			_spawn_gate(2, Vector3(-2.6, 0, 2.0), 0.0, 0.0)
			_spawn_gate(3, Vector3( 2.6, 0, -3.2), 1.8, 1.5)
		2:
			_spawn_gate(2, Vector3(-3.2, 0, 2.5), 1.2, 1.4, 0.0)
			_spawn_gate(3, Vector3( 3.2, 0, -2.0), 1.2, 1.7, 1.8)
			_spawn_block(Vector3(0.0, 0, -0.2))
		3:
			_spawn_gate(2, Vector3( 0.0, 0, 3.0), 3.4, 1.8)
			_spawn_gate(4, Vector3(-3.4, 0, -4.0), 0.0, 0.0)
			_spawn_gate(2, Vector3( 3.4, 0, -4.0), 0.0, 0.0)
			_spawn_block(Vector3(0.0, 0, -4.0))
		4:
			_spawn_gate(3, Vector3(-2.8, 0, 2.2), 1.6, 2.0, 0.0)
			_spawn_gate(3, Vector3( 2.8, 0, -3.8), 1.6, 2.1, 3.14)
			_spawn_block(Vector3(-3.0, 0, -0.9))
			_spawn_block(Vector3( 3.0, 0, -0.9))
		5:
			_spawn_gate(2, Vector3(0.0, 0, 3.4), 3.8, 2.2, 0.0)
			_spawn_gate(4, Vector3(0.0, 0, -4.2), 3.5, 2.4, 1.57)
			_spawn_block(Vector3(-3.4, 0, -0.6))
			_spawn_block(Vector3( 3.4, 0, -0.6))

	_update_hud_counts()

func _clear_entities_and_level():
	for a in _allies:
		if is_instance_valid(a):
			a.queue_free()
	_allies.clear()

	for e in _enemies:
		if is_instance_valid(e):
			e.queue_free()
	_enemies.clear()

	for g in _gates:
		if is_instance_valid(g):
			g.queue_free()
	_gates.clear()

	for fx in _fx_particles:
		if is_instance_valid(fx["mesh"]):
			fx["mesh"].queue_free()
	_fx_particles.clear()

	for c in level_objects_root.get_children():
		c.queue_free()
	_blocks.clear()
	_next_gate_id = 1

func _spawn_gate(mult: int, pos: Vector3, move_range: float, move_speed: float, phase: float = 0.0):
	var gate = _gate_scene.instantiate()
	level_objects_root.add_child(gate)
	gate.setup(_next_gate_id, mult, pos, move_range, move_speed, phase)
	_next_gate_id += 1
	_gates.append(gate)

func _spawn_block(pos: Vector3):
	var block = _block_scene.instantiate()
	level_objects_root.add_child(block)
	block.position = pos
	_blocks.append({"pos": pos, "half_w": 1.22, "half_d": 0.46})

func _on_cannon_fired(spawn_pos: Vector3, is_giant: bool):
	if _game_state != "playing":
		return
	if is_giant:
		_spawn_ally(spawn_pos, true)
		sfx.play("giant", 0.04)
	else:
		_spawn_ally(spawn_pos + Vector3(randf_range(-0.18, 0.18), 0.0, 0.0), false)
		sfx.play("shoot", 0.08)

func _spawn_ally(pos: Vector3, is_giant: bool, pre_passed_gate: int = -1):
	if _allies.size() >= MAX_ALLIES:
		return null
	var unit = (_ally_giant_scene if is_giant else _ally_mob_scene).instantiate()
	entities_root.add_child(unit)
	unit.position = Vector3(clampf(pos.x, -ARENA_HALF_WIDTH, ARENA_HALF_WIDTH), 0.0, pos.z)
	if pre_passed_gate != -1:
		unit.mark_gate_passed(pre_passed_gate)
	_allies.append(unit)
	return unit

func _spawn_enemy(pos: Vector3):
	if _enemies.size() >= MAX_ENEMIES:
		return
	var unit = _enemy_mob_scene.instantiate()
	entities_root.add_child(unit)
	unit.position = Vector3(clampf(pos.x, -ARENA_HALF_WIDTH, ARENA_HALF_WIDTH), 0.0, pos.z)
	_enemies.append(unit)

func _process(delta: float):
	_update_fx_particles(delta)

	if _game_state != "playing":
		return

	# Enemy Wave Spawner
	_enemy_wave_timer -= delta
	if _enemy_wave_timer <= 0.0:
		_enemy_wave_timer = _enemy_wave_interval
		for i in range(_enemy_wave_count):
			var rx: float = randf_range(-4.5, 4.5)
			var rz: float = ENEMY_SPAWN_Z + randf_range(-0.6, 0.6)
			_spawn_enemy(Vector3(rx, 0.0, rz))

	# Move Allies & Check Multiplier Gates
	var new_spawns: Array = []
	for a in _allies:
		if not is_instance_valid(a) or a.hp <= 0:
			continue
		a.step_forward(delta)
		_resolve_block_and_walls(a)

		for g in _gates:
			if not is_instance_valid(g):
				continue
			if a.has_passed_gate(g.gate_id):
				continue
			if absf(a.position.z - g.position.z) < 0.45 and absf(a.position.x - g.position.x) < (g.gate_width * 0.5):
				a.mark_gate_passed(g.gate_id)
				g.trigger_flash()
				sfx.play("multiply", 0.1)
				var extra: int = int(g.multiplier) - 1
				for k in range(extra):
					new_spawns.append({
						"pos": a.position + Vector3(randf_range(-0.55, 0.55), 0.0, randf_range(-0.25, 0.25)),
						"giant": a.is_giant,
						"gate_id": g.gate_id
					})

	for info in new_spawns:
		_spawn_ally(info["pos"], info["giant"], info["gate_id"])

	# Move Enemies
	for e in _enemies:
		if not is_instance_valid(e) or e.hp <= 0:
			continue
		e.step_forward(delta)
		_resolve_block_and_walls(e)

	# Soft crowd separation within same team
	_separate_crowd(_allies)
	_separate_crowd(_enemies)

	# Ally vs Enemy Clashes
	for a in _allies:
		if not is_instance_valid(a) or a.hp <= 0:
			continue
		for e in _enemies:
			if not is_instance_valid(e) or e.hp <= 0:
				continue
			var min_dist: float = float(a.radius) + float(e.radius) + 0.08
			if a.position.distance_squared_to(e.position) <= min_dist * min_dist:
				var dmg: int = mini(int(a.hp), int(e.hp))
				a.hp -= dmg
				e.hp -= dmg
				var mid: Vector3 = (a.position + e.position) * 0.5 + Vector3(0, 0.5, 0)
				_spawn_burst(mid, Color(1.0, 0.85, 0.35), 4)
				sfx.play("clash", 0.12)
				if a.hp <= 0:
					break

	# Check Castle Hits & Defense Line Breaches
	for i in range(_allies.size() - 1, -1, -1):
		var a = _allies[i]
		if not is_instance_valid(a) or a.hp <= 0:
			if is_instance_valid(a):
				a.queue_free()
			_allies.remove_at(i)
			continue
		if a.position.z <= CASTLE_HIT_Z:
			_spawn_burst(a.position + Vector3(0, 0.7, 0), Color(0.25, 0.85, 1.0), 6)
			sfx.play("castle_hit", 0.08)
			castle.take_damage(a.damage_to_castle)
			a.queue_free()
			_allies.remove_at(i)

	for i in range(_enemies.size() - 1, -1, -1):
		var e = _enemies[i]
		if not is_instance_valid(e) or e.hp <= 0:
			if is_instance_valid(e):
				e.queue_free()
			_enemies.remove_at(i)
			continue
		if e.position.z >= DEFENSE_LINE_Z:
			_trigger_defeat()
			break

	_update_hud_counts()

func _resolve_block_and_walls(unit: Node3D):
	var unit_radius: float = float(unit.get("radius"))
	unit.position.x = clampf(unit.position.x, -ARENA_HALF_WIDTH + unit_radius, ARENA_HALF_WIDTH - unit_radius)
	for b in _blocks:
		var bpos: Vector3 = b["pos"]
		var hw: float = float(b["half_w"]) + unit_radius
		var hd: float = float(b["half_d"]) + unit_radius
		var dx: float = unit.position.x - bpos.x
		var dz: float = unit.position.z - bpos.z
		if absf(dx) < hw and absf(dz) < hd:
			var push_dir: float = 1.0 if dx >= 0.0 else -1.0
			unit.position.x += push_dir * 0.14
			if bool(unit.get("is_ally")):
				unit.position.z = maxf(unit.position.z, bpos.z + hd * 0.9)
			else:
				unit.position.z = minf(unit.position.z, bpos.z - hd * 0.9)

func _separate_crowd(units: Array):
	var count: int = units.size()
	for i in range(count):
		var u1 = units[i]
		if not is_instance_valid(u1) or int(u1.hp) <= 0:
			continue
		for j in range(i + 1, mini(i + 10, count)):
			var u2 = units[j]
			if not is_instance_valid(u2) or int(u2.hp) <= 0:
				continue
			var min_r: float = (float(u1.radius) + float(u2.radius)) * 0.9
			var diff: Vector3 = u1.position - u2.position
			diff.y = 0.0
			var d_sq: float = diff.length_squared()
			if d_sq > 0.0001 and d_sq < min_r * min_r:
				var d: float = sqrt(d_sq)
				var push: Vector3 = diff / d * (min_r - d) * 0.35
				u1.position += push
				u2.position -= push

func _spawn_burst(pos: Vector3, col: Color, count: int):
	var mat := StandardMaterial3D.new()
	mat.albedo_color = col
	mat.emission_enabled = true
	mat.emission = col
	mat.emission_energy_multiplier = 2.0
	for i in range(count):
		var m := MeshInstance3D.new()
		m.mesh = _burst_mesh_res
		m.material_override = mat
		m.position = pos
		add_child(m)
		var vel := Vector3(randf_range(-3.5, 3.5), randf_range(2.2, 5.0), randf_range(-3.5, 3.5))
		_fx_particles.append({"mesh": m, "vel": vel, "life": 0.32})

func _update_fx_particles(delta: float):
	for i in range(_fx_particles.size() - 1, -1, -1):
		var p = _fx_particles[i]
		p["life"] -= delta
		if p["life"] <= 0.0:
			if is_instance_valid(p["mesh"]):
				p["mesh"].queue_free()
			_fx_particles.remove_at(i)
		else:
			p["vel"].y -= 14.0 * delta
			p["mesh"].position += p["vel"] * delta
			var s: float = clampf(float(p["life"]) / 0.32, 0.05, 1.0)
			p["mesh"].scale = Vector3.ONE * s

func _update_hud_counts():
	mob_count_label.text = "ALLIES: %d  |  ENEMIES: %d" % [_allies.size(), _enemies.size()]

func _on_charge_changed(current: int, max_charge: int):
	charge_bar.max_value = max_charge
	charge_bar.value = current
	charge_label.text = "GIANT CHARGE: %d / %d" % [current, max_charge]

func _on_castle_hp_changed(current_hp: int, max_hp: int):
	castle_hp_bar.max_value = max_hp
	castle_hp_bar.value = current_hp
	castle_hp_text.text = "FORTRESS HP: %d / %d" % [current_hp, max_hp]

func _on_castle_destroyed():
	if _game_state != "playing":
		return
	_game_state = "won"
	cannon.is_active = false
	sfx.play("win", 0.02)
	_spawn_burst(castle.global_position + Vector3(0, 2.0, 0.5), Color(1.0, 0.85, 0.25), 18)
	overlay_title.text = "VICTORY!"
	overlay_subtitle.text = "Enemy Fortress Destroyed on Level %d!" % _current_level
	overlay_button.text = "NEXT LEVEL (%d)" % (_current_level + 1)
	overlay_panel.visible = true

func _trigger_defeat():
	if _game_state != "playing":
		return
	_game_state = "lost"
	cannon.is_active = false
	sfx.play("lose", 0.02)
	overlay_title.text = "DEFENSE BREACHED!"
	overlay_subtitle.text = "Enemy raiders reached your cannon line on Level %d." % _current_level
	overlay_button.text = "RETRY LEVEL %d" % _current_level
	overlay_panel.visible = true

func _on_overlay_button_pressed():
	if _game_state == "won":
		_start_level(_current_level + 1)
	else:
		_start_level(_current_level)
```

### mob_control/scenes/main_game.tscn
```text
[gd_scene load_steps=17 format=3]

[ext_resource type="Script" path="res://scripts/main_game.gd" id="1_main"]
[ext_resource type="Script" path="res://scripts/sfx_manager.gd" id="2_sfx"]
[ext_resource type="PackedScene" path="res://scenes/mob_cannon.tscn" id="3_cannon"]
[ext_resource type="PackedScene" path="res://scenes/enemy_castle.tscn" id="4_castle"]

[sub_resource type="ProceduralSkyMaterial" id="ProceduralSkyMaterial_1"]
sky_top_color = Color(0.14, 0.28, 0.52, 1)
sky_horizon_color = Color(0.55, 0.72, 0.88, 1)
ground_bottom_color = Color(0.12, 0.15, 0.22, 1)
ground_horizon_color = Color(0.35, 0.45, 0.58, 1)

[sub_resource type="Sky" id="Sky_1"]
sky_material = SubResource("ProceduralSkyMaterial_1")

[sub_resource type="Environment" id="Environment_1"]
background_mode = 2
sky = SubResource("Sky_1")
ambient_light_source = 3
ambient_light_energy = 0.65
tonemap_mode = 2
glow_enabled = true
glow_intensity = 0.55
glow_bloom = 0.2

[sub_resource type="StandardMaterial3D" id="StandardMaterial3D_floor"]
albedo_color = Color(0.18, 0.22, 0.30, 1)
roughness = 0.78

[sub_resource type="BoxMesh" id="BoxMesh_floor"]
material = SubResource("StandardMaterial3D_floor")
size = Vector3(14.8, 0.4, 42.0)

[sub_resource type="StandardMaterial3D" id="StandardMaterial3D_wall"]
albedo_color = Color(0.12, 0.15, 0.22, 1)
metallic = 0.45
roughness = 0.45

[sub_resource type="BoxMesh" id="BoxMesh_wall"]
material = SubResource("StandardMaterial3D_wall")
size = Vector3(0.6, 0.8, 42.0)

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_topbar"]
bg_color = Color(0.05, 0.08, 0.14, 0.72)
content_margin_left = 20.0
content_margin_top = 10.0
content_margin_right = 20.0
content_margin_bottom = 10.0

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_hp_fill"]
bg_color = Color(0.92, 0.18, 0.24, 1)
corner_radius_top_left = 4
corner_radius_top_right = 4
corner_radius_bottom_right = 4
corner_radius_bottom_left = 4

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_charge_fill"]
bg_color = Color(0.16, 0.78, 1, 1)
corner_radius_top_left = 4
corner_radius_top_right = 4
corner_radius_bottom_right = 4
corner_radius_bottom_left = 4

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_panel"]
bg_color = Color(0.08, 0.11, 0.18, 0.92)
border_width_left = 2
border_width_top = 2
border_width_right = 2
border_width_bottom = 2
border_color = Color(0.24, 0.72, 1, 1)
corner_radius_top_left = 12
corner_radius_top_right = 12
corner_radius_bottom_right = 12
corner_radius_bottom_left = 12
content_margin_left = 28.0
content_margin_top = 24.0
content_margin_right = 28.0
content_margin_bottom = 24.0

[node name="MainGame" type="Node3D"]
script = ExtResource("1_main")

[node name="SfxManager" type="Node" parent="."]
script = ExtResource("2_sfx")

[node name="WorldEnvironment" type="WorldEnvironment" parent="."]
environment = SubResource("Environment_1")

[node name="DirectionalLight3D" type="DirectionalLight3D" parent="."]
transform = Transform3D(0.866025, -0.353553, 0.353553, 0, 0.707107, 0.707107, -0.5, -0.612372, 0.612372, 8, 16, 8)
light_energy = 1.25
shadow_enabled = true

[node name="Camera3D" type="Camera3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 0.7408, 0.6717, 0, -0.6717, 0.7408, 0, 16.5, 18.2)
fov = 58.0

[node name="Arena" type="Node3D" parent="."]

[node name="Floor" type="MeshInstance3D" parent="Arena"]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, -0.2, -8.5)
mesh = SubResource("BoxMesh_floor")

[node name="WallLeft" type="MeshInstance3D" parent="Arena"]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, -7.4, 0.3, -8.5)
mesh = SubResource("BoxMesh_wall")

[node name="WallRight" type="MeshInstance3D" parent="Arena"]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 7.4, 0.3, -8.5)
mesh = SubResource("BoxMesh_wall")

[node name="MobCannon" parent="." instance=ExtResource("3_cannon")]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 9.6)

[node name="EnemyCastle" parent="." instance=ExtResource("4_castle")]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, -14.2)

[node name="LevelObjects" type="Node3D" parent="."]

[node name="Entities" type="Node3D" parent="."]

[node name="CanvasLayer" type="CanvasLayer" parent="."]

[node name="Hud" type="Control" parent="CanvasLayer"]
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
mouse_filter = 2

[node name="TopBarBg" type="PanelContainer" parent="CanvasLayer/Hud"]
layout_mode = 1
anchors_preset = 10
anchor_right = 1.0
offset_bottom = 64.0
grow_horizontal = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_topbar")
mouse_filter = 2

[node name="TopBar" type="HBoxContainer" parent="CanvasLayer/Hud/TopBarBg"]
layout_mode = 2
theme_override_constants/separation = 24
mouse_filter = 2

[node name="LevelLabel" type="Label" parent="CanvasLayer/Hud/TopBarBg/TopBar"]
layout_mode = 2
theme_override_font_sizes/font_size = 24
text = "LEVEL 1"

[node name="CastleHpBox" type="VBoxContainer" parent="CanvasLayer/Hud/TopBarBg/TopBar"]
layout_mode = 2
size_flags_horizontal = 3
mouse_filter = 2

[node name="CastleHpText" type="Label" parent="CanvasLayer/Hud/TopBarBg/TopBar/CastleHpBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 15
text = "FORTRESS HP: 40 / 40"
horizontal_alignment = 1

[node name="CastleHpBar" type="ProgressBar" parent="CanvasLayer/Hud/TopBarBg/TopBar/CastleHpBox"]
custom_minimum_size = Vector2(0, 18)
layout_mode = 2
theme_override_styles/fill = SubResource("StyleBoxFlat_hp_fill")
max_value = 40.0
value = 40.0
show_percentage = false

[node name="MobCountLabel" type="Label" parent="CanvasLayer/Hud/TopBarBg/TopBar"]
layout_mode = 2
theme_override_font_sizes/font_size = 18
text = "ALLIES: 0  |  ENEMIES: 0"

[node name="BottomBar" type="HBoxContainer" parent="CanvasLayer/Hud"]
layout_mode = 1
anchors_preset = 12
anchor_top = 1.0
anchor_right = 1.0
anchor_bottom = 1.0
offset_left = 220.0
offset_top = -62.0
offset_right = -220.0
offset_bottom = -14.0
grow_horizontal = 2
grow_vertical = 0
mouse_filter = 2

[node name="ChargeBox" type="VBoxContainer" parent="CanvasLayer/Hud/BottomBar"]
layout_mode = 2
size_flags_horizontal = 3
mouse_filter = 2

[node name="ChargeLabel" type="Label" parent="CanvasLayer/Hud/BottomBar/ChargeBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 15
text = "GIANT CHARGE: 0 / 12"
horizontal_alignment = 1

[node name="ChargeBar" type="ProgressBar" parent="CanvasLayer/Hud/BottomBar/ChargeBox"]
custom_minimum_size = Vector2(0, 18)
layout_mode = 2
theme_override_styles/fill = SubResource("StyleBoxFlat_charge_fill")
max_value = 12.0
show_percentage = false

[node name="OverlayPanel" type="PanelContainer" parent="CanvasLayer"]
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -250.0
offset_top = -135.0
offset_right = 250.0
offset_bottom = 135.0
grow_horizontal = 2
grow_vertical = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_panel")

[node name="VBox" type="VBoxContainer" parent="CanvasLayer/OverlayPanel"]
layout_mode = 2
theme_override_constants/separation = 16
alignment = 1

[node name="TitleLabel" type="Label" parent="CanvasLayer/OverlayPanel/VBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 34
text = "MOB CONTROL"
horizontal_alignment = 1

[node name="SubtitleLabel" type="Label" parent="CanvasLayer/OverlayPanel/VBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 16
text = "Slide Cannon: A / D or Drag Mouse
Fire Mobs: Hold SPACE or Left-Click"
horizontal_alignment = 1

[node name="ActionButton" type="Button" parent="CanvasLayer/OverlayPanel/VBox"]
custom_minimum_size = Vector2(220, 48)
layout_mode = 2
size_flags_horizontal = 4
theme_override_font_sizes/font_size = 20
text = "PLAY LEVEL 1"
```