# Air Hockey — Godot 4 Project

### air_hockey/project.godot
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

config/name="Air Hockey"
run/main_scene="res://scenes/main.tscn"
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
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":65,"key_label":0,"unicode":97,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194319,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
move_right={
"deadzone": 0.2,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":68,"key_label":0,"unicode":100,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194321,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
move_forward={
"deadzone": 0.2,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":87,"key_label":0,"unicode":119,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194320,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
move_back={
"deadzone": 0.2,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":83,"key_label":0,"unicode":115,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194322,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
toggle_camera={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":67,"key_label":0,"unicode":99,"location":0,"echo":false,"script":null)
]
}
reset_match={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":82,"key_label":0,"unicode":114,"location":0,"echo":false,"script":null)
]
}
toggle_mute={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":77,"key_label":0,"unicode":109,"location":0,"echo":false,"script":null)
]
}

[rendering]

anti_aliasing/quality/msaa_3d=2
```

### air_hockey/addons/scad_importer/plugin.cfg
```text
[plugin]

name="OpenSCAD GLTF Importer"
description="Imports .scad files directly as 3D scenes using scad-gltf"
author="scad-gltf"
version="0.1"
script="scad_plugin.gd"

```

### air_hockey/addons/scad_importer/scad_importer.gd
```gdscript
@tool
extends EditorSceneFormatImporter

func _get_extensions():
	return PackedStringArray(["scad"])

func _get_import_flags():
	return EditorSceneFormatImporter.IMPORT_SCENE

func _import_scene(path: String, flags: int, options: Dictionary) -> Object:
	var global_source = ProjectSettings.globalize_path(path)
	var unique_id = str(hash(path))
	var temp_glb_path = ProjectSettings.globalize_path("user://scad_cache_" + unique_id + ".glb")

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

	if exit_code != 0:
		push_error("Failed to compile SCAD file: %s. Ensure scad-gltf is installed." % path.get_file())
		push_error("scad-gltf convert output: ", "\n".join(output))
		return null

	var gltf_doc = GLTFDocument.new()
	var gltf_state = GLTFState.new()
	var err = gltf_doc.append_from_file(temp_glb_path, gltf_state)

	if FileAccess.file_exists(temp_glb_path):
		DirAccess.remove_absolute(temp_glb_path)

	if err != OK:
		push_error("Failed to parse the generated GLB for %s." % path.get_file())
		return null

	var generated_scene = gltf_doc.generate_scene(gltf_state)
	if generated_scene:
		generated_scene.name = path.get_file().get_basename()

	return generated_scene

```

### air_hockey/addons/scad_importer/scad_plugin.gd
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

### air_hockey/assets/models/hockey_table.scad
```openscad
// Arcade Air Hockey Table (1 unit = 1 meter, Z-up, +Y = Forward / Opponent End)
$asa = 30.0;
$fn = 24;

// Dimensions matching gameplay constants:
// Inner playfield: X in [-0.55, +0.55], Y in [-1.10, +1.10], surface at Z = 0.80
// Goal mouth: X in [-0.18, +0.18] at Y = +/- 1.10
// Corner chamfer: 45 deg passing through (+/-0.55, +/-0.98) and (+/-0.43, +/-1.10)

table_anim = [
  ["idle", [
    ["ArchEmblem", [
      [0.0, [0, 0, 0]],
      [1.0, [0, 0, 90]],
      [2.0, [0, 0, 180]],
      [3.0, [0, 0, 270]],
      [4.0, [0, 0, 360]]
    ]],
    ["PlayerBeacon", [
      [0.0, [0, 0, 0], [-0.42, 0.0, 1.580]],
      [2.0, [0, 0, 0], [-0.42, 0.0, 1.580]]
    ]],
    ["OpponentBeacon", [
      [0.0, [0, 0, 0], [0.42, 0.0, 1.580]],
      [2.0, [0, 0, 0], [0.42, 0.0, 1.580]]
    ]]
  ]],
  ["goal_flash", [
    ["PlayerBeacon", [
      [0.0, [0, 0, 0], [-0.42, 0.0, 1.580]],
      [0.15, [0, 0, 90], [-0.42, 0.0, 1.598]],
      [0.30, [0, 0, 180], [-0.42, 0.0, 1.580]],
      [0.45, [0, 0, 270], [-0.42, 0.0, 1.598]],
      [0.60, [0, 0, 360], [-0.42, 0.0, 1.580]]
    ]],
    ["OpponentBeacon", [
      [0.0, [0, 0, 0], [0.42, 0.0, 1.580]],
      [0.15, [0, 0, 90], [0.42, 0.0, 1.598]],
      [0.30, [0, 0, 180], [0.42, 0.0, 1.580]],
      [0.45, [0, 0, 270], [0.42, 0.0, 1.598]],
      [0.60, [0, 0, 360], [0.42, 0.0, 1.580]]
    ]]
  ]]
];

module leg_assembly(sx, sy) {
  // Main support pillar
  color([0.09, 0.10, 0.14], metalness=0.80, roughness=0.30, $asa=20.0)
    translate([sx * 0.45, sy * 0.88, 0.04])
      cylinder(h=0.62, r1=0.048, r2=0.068);

  // Brushed chrome leveling foot
  color([0.78, 0.82, 0.88], metalness=0.95, roughness=0.18, $asa=35.0)
    translate([sx * 0.45, sy * 0.88, 0.0])
      cylinder(h=0.04, r1=0.065, r2=0.052);
}

// Exact 45-degree triangular corner bumper fitting flush inside the side/end rail corner
module corner_bumper(cx, cy, rot_z, is_opponent) {
  color([0.10, 0.12, 0.17], metalness=0.68, roughness=0.30, $asa=15.0)
    translate([cx, cy, 0.0])
      difference() {
        translate([-0.06, -0.06, 0.76]) cube([0.12, 0.12, 0.094]);
        rotate([0, 0, rot_z])
          translate([-0.14, -0.14, 0.75])
            cube([0.28, 0.14, 0.12]);
      }

  // Matching corner bevel neon accent linking side & end cushions
  if (is_opponent) {
    color([1.0, 0.20, 0.50], emissive=[1.0, 0.20, 0.50], emissiveIntensity=1.5, unlit=true)
      translate([cx, cy, 0.842])
        rotate([0, 0, rot_z])
          translate([-0.085, 0.001, 0.0])
            cube([0.17, 0.004, 0.008]);
  } else {
    color([0.10, 0.88, 1.0], emissive=[0.10, 0.88, 1.0], emissiveIntensity=1.5, unlit=true)
      translate([cx, cy, 0.842])
        rotate([0, 0, rot_z])
          translate([-0.085, 0.001, 0.0])
            cube([0.17, 0.004, 0.008]);
  }
}

module goal_pocket(sy) {
  // Recessed goal chute strictly outside playfield (|Y| in [1.10, 1.28])
  color([0.04, 0.05, 0.08], metalness=0.45, roughness=0.50) {
    translate([-0.195, sy * 1.18, 0.7525])
      cube([0.03, 0.16, 0.205], center=true);
    translate([0.195, sy * 1.18, 0.7525])
      cube([0.03, 0.16, 0.205], center=true);
  }
  color([0.03, 0.04, 0.06], metalness=0.35, roughness=0.55) {
    translate([0.0, sy * 1.26, 0.7525])
      cube([0.42, 0.04, 0.205], center=true);
    translate([0.0, sy * 1.19, 0.65])
      cube([0.42, 0.18, 0.02], center=true);
  }
  // Matte dark-anodized goal hood bridge (low glare)
  color([0.06, 0.07, 0.10], metalness=0.42, roughness=0.55, specularIntensity=0.15)
    translate([0.0, sy * 1.14, 0.8525])
      cube([0.40, 0.08, 0.035], center=true);

  // Brushed chrome goal post caps at (+/-0.184, sy * 1.104)
  color([0.80, 0.84, 0.90], metalness=0.92, roughness=0.18) {
    translate([-0.184, sy * 1.104, 0.80]) cylinder(h=0.056, r=0.011, $fn=16);
    translate([0.184, sy * 1.104, 0.80]) cylinder(h=0.056, r=0.011, $fn=16);
  }

  // Inner backplate glow bar + top hood accent inside chute
  if (sy < 0) {
    color([0.08, 0.88, 1.0], emissive=[0.08, 0.88, 1.0], emissiveIntensity=1.6, unlit=true) {
      translate([0.0, sy * 1.238, 0.78]) cube([0.34, 0.008, 0.022], center=true);
      translate([0.0, sy * 1.104, 0.869]) cube([0.36, 0.008, 0.005], center=true);
    }
  } else {
    color([1.0, 0.20, 0.50], emissive=[1.0, 0.20, 0.50], emissiveIntensity=1.6, unlit=true) {
      translate([0.0, sy * 1.238, 0.78]) cube([0.34, 0.008, 0.022], center=true);
      translate([0.0, sy * 1.104, 0.869]) cube([0.36, 0.008, 0.005], center=true);
    }
  }
}

module faceoff_marker(fx, fy) {
  color([0.10, 0.72, 0.94], emissive=[0.08, 0.60, 0.86], emissiveIntensity=0.55, roughness=0.30, $asa=32.0) {
    difference() {
      translate([fx, fy, 0.80]) cylinder(h=0.0008, r=0.082, $fn=32);
      translate([fx, fy, 0.7995]) cylinder(h=0.002, r=0.075, $fn=32);
    }
    translate([fx, fy, 0.80]) cylinder(h=0.0008, r=0.014, $fn=20);
  }
}

armature(animations=table_anim) {
  bone(name="TableRoot", t=[0, 0, 0], r=[0, 0, 0]) {
    // 1. Deep Obsidian-Navy Satin Playfield Surface (Z = 0.76 to 0.80) for maximum puck & neon contrast
    color([0.028, 0.038, 0.062], metalness=0.14, roughness=0.34, specularIntensity=0.28, $asa=15.0)
      translate([-0.55, -1.10, 0.76])
        cube([1.10, 2.20, 0.04]);

    // 2. Crisp Playfield Markings (Z = 0.800 to 0.8008)
    color([0.07, 0.45, 0.65], emissive=[0.05, 0.35, 0.55], emissiveIntensity=0.35, roughness=0.35) {
      translate([-0.535, -0.96, 0.80]) cube([0.006, 1.92, 0.0006]);
      translate([0.529, -0.96, 0.80]) cube([0.006, 1.92, 0.0006]);
    }

    // Center line
    color([0.12, 0.78, 0.98], emissive=[0.08, 0.68, 0.92], emissiveIntensity=0.72, roughness=0.30)
      translate([-0.54, -0.008, 0.80])
        cube([1.08, 0.016, 0.0008]);

    // Center faceoff circle
    color([0.12, 0.78, 0.98], emissive=[0.08, 0.68, 0.92], emissiveIntensity=0.72, roughness=0.30, $asa=40.0) {
      difference() {
        translate([0, 0, 0.80]) cylinder(h=0.0008, r=0.175, $fn=48);
        translate([0, 0, 0.7995]) cylinder(h=0.002, r=0.160, $fn=48);
      }
      difference() {
        translate([0, 0, 0.80]) cylinder(h=0.0008, r=0.105, $fn=48);
        translate([0, 0, 0.7995]) cylinder(h=0.002, r=0.098, $fn=48);
      }
      translate([0, 0, 0.80]) cylinder(h=0.0008, r=0.022, $fn=24);
    }

    // Four quadrant faceoff dots
    faceoff_marker(-0.27, -0.40);
    faceoff_marker(0.27, -0.40);
    faceoff_marker(-0.27, 0.40);
    faceoff_marker(0.27, 0.40);

    // Player end goal crease (Y = -1.10, cyan) + zone line
    color([0.10, 0.82, 1.0], emissive=[0.08, 0.72, 0.96], emissiveIntensity=0.75, roughness=0.30, $asa=40.0) {
      difference() {
        translate([0, -1.10, 0.80]) cylinder(h=0.0008, r=0.25, $fn=40);
        translate([0, -1.10, 0.7995]) cylinder(h=0.002, r=0.236, $fn=40);
        translate([-0.30, -1.40, 0.799]) cube([0.60, 0.30, 0.004]);
      }
      translate([-0.54, -0.58, 0.80]) cube([1.08, 0.011, 0.0008]);
    }

    // Opponent end goal crease (Y = +1.10, magenta) + zone line
    color([1.0, 0.22, 0.52], emissive=[0.92, 0.18, 0.46], emissiveIntensity=0.75, roughness=0.30, $asa=40.0) {
      difference() {
        translate([0, 1.10, 0.80]) cylinder(h=0.0008, r=0.25, $fn=40);
        translate([0, 1.10, 0.7995]) cylinder(h=0.002, r=0.236, $fn=40);
        translate([-0.30, 1.10, 0.799]) cube([0.60, 0.30, 0.004]);
      }
      translate([-0.54, 0.57, 0.80]) cube([1.08, 0.011, 0.0008]);
    }

    // 3. Under-Cabinet Chassis
    color([0.06, 0.07, 0.10], metalness=0.55, roughness=0.45)
      translate([-0.58, -1.10, 0.64])
        cube([1.16, 2.20, 0.12]);

    // Lower neon underglow trim strips along sides
    color([0.08, 0.85, 1.0], emissive=[0.08, 0.85, 1.0], emissiveIntensity=1.6, unlit=true) {
      translate([-0.588, -1.06, 0.67]) cube([0.01, 2.12, 0.018]);
      translate([0.578, -1.06, 0.67]) cube([0.01, 2.12, 0.018]);
    }

    // 4. Side & End Rails (Z = 0.76 to 0.855)
    color([0.10, 0.12, 0.17], metalness=0.72, roughness=0.28) {
      translate([-0.63, -1.18, 0.76]) cube([0.08, 2.36, 0.095]);
      translate([0.55, -1.18, 0.76]) cube([0.08, 2.36, 0.095]);

      translate([-0.63, -1.18, 0.76]) cube([0.45, 0.08, 0.095]);
      translate([0.18, -1.18, 0.76]) cube([0.45, 0.08, 0.095]);

      translate([-0.63, 1.10, 0.76]) cube([0.45, 0.08, 0.095]);
      translate([0.18, 1.10, 0.76]) cube([0.45, 0.08, 0.095]);
    }

    // Top brushed-metal rail trim inlay
    color([0.62, 0.68, 0.76], metalness=0.88, roughness=0.22) {
      translate([-0.605, -1.15, 0.855]) cube([0.022, 2.30, 0.005]);
      translate([0.583, -1.15, 0.855]) cube([0.022, 2.30, 0.005]);
      translate([-0.583, -1.15, 0.855]) cube([0.39, 0.022, 0.005]);
      translate([0.193, -1.15, 0.855]) cube([0.39, 0.022, 0.005]);
      translate([-0.583, 1.128, 0.855]) cube([0.39, 0.022, 0.005]);
      translate([0.193, 1.128, 0.855]) cube([0.39, 0.022, 0.005]);
    }

    // Inner glowing rim accent along side cushions (Cyan on player half, Magenta on AI half)
    color([0.10, 0.88, 1.0], emissive=[0.10, 0.88, 1.0], emissiveIntensity=1.5, unlit=true) {
      translate([-0.552, -0.98, 0.842]) cube([0.004, 0.98, 0.008]);
      translate([0.548, -0.98, 0.842]) cube([0.004, 0.98, 0.008]);
    }
    color([1.0, 0.20, 0.50], emissive=[1.0, 0.20, 0.50], emissiveIntensity=1.5, unlit=true) {
      translate([-0.552, 0.0, 0.842]) cube([0.004, 0.98, 0.008]);
      translate([0.548, 0.0, 0.842]) cube([0.004, 0.98, 0.008]);
    }

    // 5. Exact 45-degree Corner Bumpers matching |x| + |y| = 1.53 (all 4 corners symmetric & flush)
    corner_bumper(0.49, 1.04, -45, true);
    corner_bumper(-0.49, 1.04, 45, true);
    corner_bumper(0.49, -1.04, -135, false);
    corner_bumper(-0.49, -1.04, 135, false);

    // 6. Goal Pockets & Goal Mouth Neon Markers
    goal_pocket(-1);
    goal_pocket(1);

    // Player goal neon rim (Cyan)
    color([0.08, 0.90, 1.0], emissive=[0.08, 0.90, 1.0], emissiveIntensity=1.9, unlit=true)
      translate([-0.18, -1.106, 0.832])
        cube([0.36, 0.012, 0.008]);

    // Opponent goal neon rim (Magenta)
    color([1.0, 0.20, 0.52], emissive=[1.0, 0.20, 0.52], emissiveIntensity=1.9, unlit=true)
      translate([-0.18, 1.094, 0.832])
        cube([0.36, 0.012, 0.008]);

    // 7. Four Legs + Cross Stretcher Bars
    leg_assembly(-1, -1);
    leg_assembly(1, -1);
    leg_assembly(-1, 1);
    leg_assembly(1, 1);

    color([0.08, 0.09, 0.13], metalness=0.65, roughness=0.35) {
      translate([-0.47, -0.88, 0.24]) cube([0.94, 0.035, 0.045]);
      translate([-0.47, 0.845, 0.24]) cube([0.94, 0.035, 0.045]);
      translate([-0.025, -0.88, 0.24]) cube([0.05, 1.76, 0.04]);
    }

    // 8. Center Overhead Scoreboard Gantry Arch at Y = 0 (hidden cleanly in Top-Down View)
    bone(name="ScoreboardArch", t=[0, 0, 0], r=[0, 0, 0]) {
      color([0.08, 0.10, 0.15], metalness=0.82, roughness=0.26) {
        // Side pillars rising from side rails
        translate([-0.625, -0.032, 0.855]) cube([0.050, 0.064, 0.64]);
        translate([0.575, -0.032, 0.855]) cube([0.050, 0.064, 0.64]);
        // Main crossbar (Z = 1.485 to 1.565)
        translate([-0.635, -0.044, 1.485]) cube([1.27, 0.088, 0.080]);
        // Stepped central jumbotron crown housing (Z = 1.565 to 1.610)
        translate([-0.25, -0.036, 1.565]) cube([0.50, 0.072, 0.045]);
      }

      // Brushed chrome pedestal mounts on bridge top (zero floating gaps)
      color([0.72, 0.78, 0.86], metalness=0.92, roughness=0.20) {
        translate([-0.42, 0.0, 1.565]) cylinder(h=0.015, r1=0.030, r2=0.026, $fn=24);
        translate([0.42, 0.0, 1.565]) cylinder(h=0.015, r1=0.030, r2=0.026, $fn=24);
        translate([0.0, 0.0, 1.610]) cylinder(h=0.026, r1=0.016, r2=0.011, $fn=20);
      }

      // Vertical inner pillar neon slits
      color([0.12, 0.88, 1.0], emissive=[0.12, 0.88, 1.0], emissiveIntensity=1.5, unlit=true) {
        translate([-0.576, -0.005, 0.89]) cube([0.005, 0.010, 0.56]);
        translate([0.571, -0.005, 0.89]) cube([0.005, 0.010, 0.56]);
      }

      // Arch front & back neon light bars (cyan left, magenta right) + central amber crown accent
      color([0.12, 0.88, 1.0], emissive=[0.12, 0.88, 1.0], emissiveIntensity=1.7, unlit=true) {
        translate([-0.55, -0.048, 1.517]) cube([0.52, 0.008, 0.014]);
        translate([-0.55, 0.040, 1.517]) cube([0.52, 0.008, 0.014]);
      }
      color([1.0, 0.22, 0.52], emissive=[1.0, 0.22, 0.52], emissiveIntensity=1.7, unlit=true) {
        translate([0.03, -0.048, 1.517]) cube([0.52, 0.008, 0.014]);
        translate([0.03, 0.040, 1.517]) cube([0.52, 0.008, 0.014]);
      }
      color([1.0, 0.72, 0.18], emissive=[1.0, 0.65, 0.10], emissiveIntensity=1.6, unlit=true) {
        translate([-0.14, -0.040, 1.582]) cube([0.28, 0.006, 0.012]);
        translate([-0.14, 0.034, 1.582]) cube([0.28, 0.006, 0.012]);
      }

      // Animated Championship Trophy Gem (spins continuously in idle)
      bone(name="ArchEmblem", t=[0, 0, 1.648], r=[0, 0, 0]) {
        color([1.0, 0.72, 0.16], metalness=0.85, roughness=0.18, emissive=[1.0, 0.58, 0.08], emissiveIntensity=1.6, $asa=30.0) {
          translate([0, 0, 0.008]) cylinder(h=0.022, r1=0.028, r2=0.006, $fn=6);
          translate([0, 0, -0.014]) cylinder(h=0.022, r1=0.008, r2=0.028, $fn=6);
        }
      }

      bone(name="PlayerBeacon", t=[-0.42, 0.0, 1.580], r=[0, 0, 0]) {
        color([0.10, 0.88, 1.0], emissive=[0.10, 0.88, 1.0], emissiveIntensity=1.9, unlit=true, $asa=45.0)
          cylinder(h=0.028, r1=0.024, r2=0.015, $fn=24);
      }

      bone(name="OpponentBeacon", t=[0.42, 0.0, 1.580], r=[0, 0, 0]) {
        color([1.0, 0.20, 0.52], emissive=[1.0, 0.20, 0.52], emissiveIntensity=1.9, unlit=true, $asa=45.0)
          cylinder(h=0.028, r1=0.024, r2=0.015, $fn=24);
      }
    }
  }
}
```

### air_hockey/assets/models/player_mallet.scad
```openscad
// Player Air Hockey Mallet (Cyan Neon Theme, Radius = 0.055m, Base at Z = 0)
$asa = 45.0;
$fn = 36;

mallet_anim = [
  ["idle", [
    ["CoreRing", [
      [0.0, [0, 0, 0], [0, 0, 0.022]],
      [1.0, [0, 0, 90], [0, 0, 0.022]],
      [2.0, [0, 0, 180], [0, 0, 0.022]],
      [3.0, [0, 0, 270], [0, 0, 0.022]],
      [4.0, [0, 0, 360], [0, 0, 0.022]]
    ]]
  ]]
];

armature(animations=mallet_anim) {
  bone(name="MalletBody", t=[0, 0, 0], r=[0, 0, 0]) {
    // Low-friction dark base skirt
    color([0.07, 0.09, 0.13], metalness=0.6, roughness=0.25)
      cylinder(h=0.010, r=0.055);

    // Subtle under-rim neon halo ring
    color([0.12, 0.92, 1.0], emissive=[0.12, 0.92, 1.0], emissiveIntensity=2.0, unlit=true)
      translate([0, 0, 0.010])
        cylinder(h=0.003, r=0.0555);

    // Flared metallic striker body
    color([0.06, 0.52, 0.92], metalness=0.45, roughness=0.20)
      translate([0, 0, 0.013])
        cylinder(h=0.013, r1=0.055, r2=0.042);

    // Ergonomic handle stem
    color([0.12, 0.14, 0.20], metalness=0.82, roughness=0.24)
      translate([0, 0, 0.026])
        cylinder(h=0.034, r1=0.015, r2=0.019);

    // Rounded handle grip knob
    color([0.08, 0.68, 1.0], metalness=0.35, roughness=0.18)
      translate([0, 0, 0.062])
        scale([1.0, 1.0, 0.72])
          sphere(r=0.024);

    // Top chrome cap
    color([0.88, 0.92, 0.98], metalness=0.95, roughness=0.14)
      translate([0, 0, 0.077])
        cylinder(h=0.003, r=0.011);

    // Animated glowing ring inset
    bone(name="CoreRing", t=[0, 0, 0.022], r=[0, 0, 0]) {
      color([0.22, 0.95, 1.0], emissive=[0.22, 0.95, 1.0], emissiveIntensity=2.5, unlit=true)
        difference() {
          cylinder(h=0.006, r=0.046, $fn=8);
          translate([0, 0, -0.001]) cylinder(h=0.008, r=0.038, $fn=8);
        }
    }
  }
}
```

### air_hockey/assets/models/opponent_mallet.scad
```openscad
// Opponent AI Air Hockey Mallet (Crimson/Magenta Neon Theme, Radius = 0.055m, Base at Z = 0)
$asa = 45.0;
$fn = 36;

mallet_anim = [
  ["idle", [
    ["CoreRing", [
      [0.0, [0, 0, 0], [0, 0, 0.022]],
      [1.0, [0, 0, -90], [0, 0, 0.022]],
      [2.0, [0, 0, -180], [0, 0, 0.022]],
      [3.0, [0, 0, -270], [0, 0, 0.022]],
      [4.0, [0, 0, -360], [0, 0, 0.022]]
    ]]
  ]]
];

armature(animations=mallet_anim) {
  bone(name="MalletBody", t=[0, 0, 0], r=[0, 0, 0]) {
    // Low-friction dark base skirt
    color([0.07, 0.09, 0.13], metalness=0.6, roughness=0.25)
      cylinder(h=0.010, r=0.055);

    // Subtle under-rim neon halo ring
    color([1.0, 0.22, 0.54], emissive=[1.0, 0.22, 0.54], emissiveIntensity=2.0, unlit=true)
      translate([0, 0, 0.010])
        cylinder(h=0.003, r=0.0555);

    // Flared metallic striker body
    color([0.88, 0.14, 0.44], metalness=0.45, roughness=0.20)
      translate([0, 0, 0.013])
        cylinder(h=0.013, r1=0.055, r2=0.042);

    // Ergonomic handle stem
    color([0.12, 0.14, 0.20], metalness=0.82, roughness=0.24)
      translate([0, 0, 0.026])
        cylinder(h=0.034, r1=0.015, r2=0.019);

    // Rounded handle grip knob
    color([0.98, 0.20, 0.50], metalness=0.35, roughness=0.18)
      translate([0, 0, 0.062])
        scale([1.0, 1.0, 0.72])
          sphere(r=0.024);

    // Top chrome cap
    color([0.88, 0.92, 0.98], metalness=0.95, roughness=0.14)
      translate([0, 0, 0.077])
        cylinder(h=0.003, r=0.011);

    // Animated glowing ring inset
    bone(name="CoreRing", t=[0, 0, 0.022], r=[0, 0, 0]) {
      color([1.0, 0.30, 0.60], emissive=[1.0, 0.30, 0.60], emissiveIntensity=2.5, unlit=true)
        difference() {
          cylinder(h=0.006, r=0.046, $fn=8);
          translate([0, 0, -0.001]) cylinder(h=0.008, r=0.038, $fn=8);
        }
    }
  }
}
```

### air_hockey/assets/models/hockey_puck.scad
```openscad
// High-Visibility Neon Amber-Orange Air Hockey Puck (Radius = 0.042m, Height = 0.016m, Base at Z = 0)
$asa = 40.0;
$fn = 36;

puck_anim = [
  ["spin", [
    ["PuckRoot", [
      [0.0, [0, 0, 0]],
      [0.5, [0, 0, 90]],
      [1.0, [0, 0, 180]],
      [1.5, [0, 0, 270]],
      [2.0, [0, 0, 360]]
    ]]
  ]]
];

armature(animations=puck_anim) {
  bone(name="PuckRoot", t=[0, 0, 0], r=[0, 0, 0]) {
    // Low-friction dark anodized outer skirt at base
    color([0.10, 0.10, 0.14], metalness=0.6, roughness=0.25)
      cylinder(h=0.004, r=0.042);

    // Vivid luminous neon orange-amber body for high contrast against dark slate + cyan/magenta
    color([1.0, 0.44, 0.06], metalness=0.1, roughness=0.2, emissive=[1.0, 0.40, 0.04], emissiveIntensity=1.75, unlit=true)
      translate([0, 0, 0.004])
        cylinder(h=0.010, r=0.042);

    // Hot gold-white top rim bevel
    color([1.0, 0.90, 0.42], metalness=0.1, roughness=0.15, emissive=[1.0, 0.84, 0.30], emissiveIntensity=2.0, unlit=true)
      translate([0, 0, 0.014])
        cylinder(h=0.0022, r1=0.042, r2=0.038);

    // High-contrast dark metallic core disk & spinning crosshair
    color([0.07, 0.08, 0.11], metalness=0.75, roughness=0.25) {
      translate([0, 0, 0.0158]) cylinder(h=0.001, r=0.021);
      translate([-0.031, -0.0035, 0.0158]) cube([0.062, 0.007, 0.001]);
      translate([-0.0035, -0.031, 0.0158]) cube([0.007, 0.062, 0.001]);
    }

    // Bright glowing center pip
    color([1.0, 0.94, 0.50], emissive=[1.0, 0.90, 0.35], emissiveIntensity=2.1, unlit=true)
      translate([0, 0, 0.0165])
        cylinder(h=0.0008, r=0.0075);
  }
}
```

### air_hockey/assets/models/overhead_lamp.scad
```openscad
// Sleek Overhead Arcade Pendant Light (mounted above table)
$asa = 30.0;
$fn = 24;

lamp_anim = [
  ["idle", [
    ["FixtureBody", [
      [0.0, [0, 0, 0]],
      [2.0, [0, 0, 0]]
    ]]
  ]]
];

armature(animations=lamp_anim) {
  bone(name="FixtureBody", t=[0, 0, 0], r=[0, 0, 0]) {
    // Hood housing
    color([0.10, 0.11, 0.15], metalness=0.8, roughness=0.3)
      translate([-0.22, -0.78, 0.0])
        cube([0.44, 1.56, 0.055]);

    // Glowing LED diffuser panel facing downward
    color([0.92, 0.97, 1.0], emissive=[0.85, 0.94, 1.0], emissiveIntensity=2.4, unlit=true)
      translate([-0.18, -0.72, -0.004])
        cube([0.36, 1.44, 0.006]);

    // Neon rim accent around hood
    color([0.15, 0.85, 1.0], emissive=[0.15, 0.85, 1.0], emissiveIntensity=1.6, unlit=true)
      translate([-0.225, -0.785, 0.022])
        difference() {
          cube([0.45, 1.57, 0.01]);
          translate([0.008, 0.008, -0.002]) cube([0.434, 1.554, 0.014]);
        }

    // Suspension rods up to ceiling
    color([0.65, 0.70, 0.78], metalness=0.95, roughness=0.2) {
      translate([0, -0.58, 0.055]) cylinder(h=2.5, r=0.012);
      translate([0, 0.58, 0.055]) cylinder(h=2.5, r=0.012);
    }
  }
}
```

### air_hockey/scripts/sound_synth.gd
```gdscript
extends Node

# Procedural WAV synthesizer for zero-dependency arcade sound effects.

const MIX_RATE := 22050

var _players: Array[AudioStreamPlayer] = []
var _next_player := 0
var _muted := false

var _wav_mallet_hit: AudioStreamWAV
var _wav_wall_hit: AudioStreamWAV
var _wav_goal_player: AudioStreamWAV
var _wav_goal_opponent: AudioStreamWAV
var _wav_faceoff: AudioStreamWAV
var _wav_victory: AudioStreamWAV

func _ready() -> void:
	for i in range(6):
		var p := AudioStreamPlayer.new()
		p.bus = &"Master"
		add_child(p)
		_players.append(p)

	_wav_mallet_hit = _make_sweep_click(0.065, 760.0, 210.0, 0.65, 0.25)
	_wav_wall_hit = _make_sweep_click(0.048, 430.0, 140.0, 0.48, 0.12)
	_wav_faceoff = _make_tone_sequence([880.0], [0.09], 0.35)
	_wav_goal_player = _make_tone_sequence([523.25, 659.25, 783.99, 1046.5], [0.09, 0.09, 0.09, 0.22], 0.45)
	_wav_goal_opponent = _make_tone_sequence([311.13, 233.08, 174.61], [0.12, 0.12, 0.24], 0.45)
	_wav_victory = _make_tone_sequence([523.25, 659.25, 783.99, 1046.5, 1318.5], [0.1, 0.1, 0.1, 0.14, 0.32], 0.5)

func set_muted(muted: bool) -> void:
	_muted = muted

func is_muted() -> bool:
	return _muted

func play_mallet_hit(strength: float = 1.0) -> void:
	_play_wav(_wav_mallet_hit, clampf(strength, 0.25, 1.15), randf_range(0.94, 1.08))

func play_wall_hit(strength: float = 1.0) -> void:
	_play_wav(_wav_wall_hit, clampf(strength, 0.2, 1.0), randf_range(0.92, 1.08))

func play_faceoff() -> void:
	_play_wav(_wav_faceoff, 0.75, 1.0)

func play_goal(player_scored: bool) -> void:
	if player_scored:
		_play_wav(_wav_goal_player, 0.9, 1.0)
	else:
		_play_wav(_wav_goal_opponent, 0.9, 1.0)

func play_victory() -> void:
	_play_wav(_wav_victory, 0.95, 1.0)

func _play_wav(wav: AudioStreamWAV, gain: float, pitch: float) -> void:
	if _muted or wav == null or _players.is_empty():
		return
	var p := _players[_next_player]
	_next_player = (_next_player + 1) % _players.size()
	p.stream = wav
	p.volume_db = linear_to_db(clampf(gain, 0.01, 1.2))
	p.pitch_scale = pitch
	p.play()

func _make_sweep_click(duration: float, f_start: float, f_end: float, amp: float, noise_mix: float) -> AudioStreamWAV:
	var count := int(MIX_RATE * duration)
	var data := PackedByteArray()
	data.resize(count * 2)
	var phase := 0.0
	for i in range(count):
		var t := float(i) / float(count)
		var freq := lerpf(f_start, f_end, pow(t, 0.55))
		phase += TAU * freq / float(MIX_RATE)
		var env := exp(-t * 6.5) * (1.0 - t)
		var tone := sin(phase)
		var noise := randf_range(-1.0, 1.0)
		var sample := (tone * (1.0 - noise_mix) + noise * noise_mix) * env * amp
		data.encode_s16(i * 2, int(clampf(sample, -1.0, 1.0) * 32767.0))
	return _build_wav(data)

func _make_tone_sequence(freqs: Array, durations: Array, amp: float) -> AudioStreamWAV:
	var total_samples := 0
	for d in durations:
		total_samples += int(MIX_RATE * float(d))
	var data := PackedByteArray()
	data.resize(total_samples * 2)
	var idx := 0
	for n in range(freqs.size()):
		var freq := float(freqs[n])
		var dur := float(durations[n])
		var seg_count := int(MIX_RATE * dur)
		var phase := 0.0
		for i in range(seg_count):
			var t := float(i) / float(seg_count)
			phase += TAU * freq / float(MIX_RATE)
			var attack := minf(t * 25.0, 1.0)
			var release := exp(-t * 3.5)
			var wave := (sin(phase) * 0.75 + sin(phase * 2.0) * 0.25)
			var sample := wave * attack * release * amp
			data.encode_s16(idx * 2, int(clampf(sample, -1.0, 1.0) * 32767.0))
			idx += 1
	return _build_wav(data)

func _build_wav(data: PackedByteArray) -> AudioStreamWAV:
	var wav := AudioStreamWAV.new()
	wav.format = AudioStreamWAV.FORMAT_16_BITS
	wav.mix_rate = MIX_RATE
	wav.stereo = false
	wav.data = data
	return wav
```

### air_hockey/scripts/arena.gd
```gdscript
extends Node3D

signal score_changed(player_score: int, opponent_score: int)
signal goal_scored(by_player: bool, player_score: int, opponent_score: int)
signal match_ended(player_won: bool, player_score: int, opponent_score: int, max_puck_speed: float)
signal camera_mode_changed(mode_name: String)

enum Difficulty { CASUAL, PRO, ACE }
enum CameraMode { PLAYER_VIEW, BROADCAST_VIEW, TOP_DOWN }

# Exact playfield geometry matching hockey_table.scad (meters)
const PLAYFIELD_Y := 0.80
const HALF_WIDTH := 0.55
const HALF_LENGTH := 1.10
const CORNER_CHAMFER_SUM := 1.53 # |x| + |z| <= 1.53
const GOAL_HALF_WIDTH := 0.18
const GOAL_POST_RADIUS := 0.010

const PUCK_RADIUS := 0.042
const MALLET_RADIUS := 0.055

const SUBSTEPS := 8
const WALL_RESTITUTION := 0.82
const MALLET_RESTITUTION := 0.72
const AIR_FRICTION_PER_SEC := 0.84 # smooth air-cushion drag
const MAX_PUCK_SPEED := 5.6
const PLAYER_MAX_SPEED := 4.6

@onready var _table_node: Node3D = $HockeyTable
@onready var _overhead_lamp: Node3D = $OverheadLamp
@onready var _player_mallet: Node3D = $PlayerMallet
@onready var _opponent_mallet: Node3D = $OpponentMallet
@onready var _puck_node: Node3D = $HockeyPuck
@onready var _puck_light: OmniLight3D = $HockeyPuck/PuckLight
@onready var _camera: Camera3D = $MainCamera
@onready var _goal_light_player: OmniLight3D = $GoalLightPlayer
@onready var _goal_light_opponent: OmniLight3D = $GoalLightOpponent
@onready var _impact_sparks: GPUParticles3D = $ImpactSparks
@onready var _goal_burst: GPUParticles3D = $GoalBurst
@onready var _sound_synth: Node = $SoundSynth

var _difficulty: Difficulty = Difficulty.PRO
var _target_score := 7
var _player_score := 0
var _opponent_score := 0
var _match_active := false
var _round_live := false
var _goal_pause_timer := 0.0
var _puck_drop_offset := 0.0
var _max_puck_speed_recorded := 0.0

# 2D XZ simulation state
var _puck_pos := Vector2.ZERO
var _puck_vel := Vector2.ZERO

var _player_pos := Vector2(0.0, 0.72)
var _player_vel := Vector2.ZERO
var _player_target := Vector2(0.0, 0.72)

var _ai_pos := Vector2(0.0, -0.72)
var _ai_vel := Vector2.ZERO
var _ai_strafe_bias := 0.0

var _using_mouse := true
var _last_mouse_pos := Vector2.ZERO

var _camera_mode: CameraMode = CameraMode.PLAYER_VIEW
var _camera_shake := 0.0
var _cam_base_pos := Vector3(0.0, 2.14, 2.04)
var _cam_look_target := Vector3(0.0, 0.74, -0.02)
var _cam_target_fov := 50.0

var _table_anim_player: AnimationPlayer = null
var _table_skeleton: Skeleton3D = null
var _arch_bone_idx := -1
var _arch_node: Node3D = null

func _ready() -> void:
	_disable_shadows(_overhead_lamp)
	_find_table_animation_player()
	_find_arch_nodes()
	_play_idle_model_animations()
	_apply_camera_mode(CameraMode.PLAYER_VIEW, false)
	_sync_visuals(0.016)

func _disable_shadows(root: Node) -> void:
	if not root:
		return
	if root is GeometryInstance3D:
		(root as GeometryInstance3D).cast_shadow = GeometryInstance3D.SHADOW_CASTING_SETTING_OFF
	for child in root.get_children():
		_disable_shadows(child)

func _find_table_animation_player() -> void:
	if _table_node:
		var anims: Array[Node] = _table_node.find_children("", "AnimationPlayer", true, false)
		if not anims.is_empty():
			_table_anim_player = anims[0] as AnimationPlayer
			if _table_anim_player.has_animation("idle"):
				_table_anim_player.get_animation("idle").loop_mode = Animation.LOOP_LINEAR
				_table_anim_player.play("idle")

func _find_arch_nodes() -> void:
	if not _table_node:
		return
	var skels: Array[Node] = _table_node.find_children("", "Skeleton3D", true, false)
	if not skels.is_empty():
		_table_skeleton = skels[0] as Skeleton3D
		if _table_skeleton:
			_arch_bone_idx = _table_skeleton.find_bone("ScoreboardArch")
	var found := _table_node.find_child("ScoreboardArch", true, false)
	if found is Node3D:
		_arch_node = found as Node3D

func _set_scoreboard_arch_visible(is_visible: bool) -> void:
	if _arch_node:
		_arch_node.visible = is_visible
	if _table_skeleton and _arch_bone_idx >= 0:
		var s := Vector3.ONE if is_visible else Vector3(0.0001, 0.0001, 0.0001)
		_table_skeleton.set_bone_pose_scale(_arch_bone_idx, s)

func _play_idle_model_animations() -> void:
	for n: Node3D in [_player_mallet, _opponent_mallet]:
		var anims: Array[Node] = n.find_children("", "AnimationPlayer", true, false)
		for ap: Node in anims:
			if ap is AnimationPlayer and ap.has_animation("idle"):
				ap.get_animation("idle").loop_mode = Animation.LOOP_LINEAR
				ap.play("idle")
	var puck_anims: Array[Node] = _puck_node.find_children("", "AnimationPlayer", true, false)
	for ap: Node in puck_anims:
		if ap is AnimationPlayer and ap.has_animation("spin"):
			ap.get_animation("spin").loop_mode = Animation.LOOP_LINEAR
			ap.play("spin")

func start_match(difficulty: int, target_score: int) -> void:
	_difficulty = clampi(difficulty, 0, 2) as Difficulty
	_target_score = maxi(1, target_score)
	_player_score = 0
	_opponent_score = 0
	_max_puck_speed_recorded = 0.0
	_match_active = true
	score_changed.emit(_player_score, _opponent_score)
	_reset_round(0)

func stop_match() -> void:
	_match_active = false
	_round_live = false

func set_muted(muted: bool) -> void:
	_sound_synth.set_muted(muted)

func is_muted() -> bool:
	return bool(_sound_synth.is_muted())

func cycle_camera_mode() -> void:
	var next_mode := (int(_camera_mode) + 1) % 3
	_apply_camera_mode(next_mode as CameraMode, true)

func _apply_camera_mode(mode: CameraMode, notify: bool) -> void:
	_camera_mode = mode
	var show_overhead := (_camera_mode != CameraMode.TOP_DOWN)
	_overhead_lamp.visible = show_overhead
	_set_scoreboard_arch_visible(show_overhead)
	match _camera_mode:
		CameraMode.PLAYER_VIEW:
			_cam_base_pos = Vector3(0.0, 2.14, 2.04)
			_cam_look_target = Vector3(0.0, 0.74, -0.02)
			_cam_target_fov = 50.0
			if notify:
				camera_mode_changed.emit("Player View")
		CameraMode.BROADCAST_VIEW:
			_cam_base_pos = Vector3(1.88, 1.98, 1.18)
			_cam_look_target = Vector3(0.0, 0.76, 0.0)
			_cam_target_fov = 50.0
			if notify:
				camera_mode_changed.emit("Broadcast View")
		CameraMode.TOP_DOWN:
			_cam_base_pos = Vector3(0.0, 3.12, 0.08)
			_cam_look_target = Vector3(0.0, 0.80, 0.0)
			_cam_target_fov = 58.0
			if notify:
				camera_mode_changed.emit("Top-Down View")

func _reset_round(serve_to: int) -> void:
	# serve_to: 0 = center, -1 = toward opponent (-Z), 1 = toward player (+Z)
	_round_live = true
	_goal_pause_timer = 0.0
	_puck_drop_offset = 0.0
	_ai_strafe_bias = randf_range(-0.14, 0.14)

	if serve_to == 1:
		_puck_pos = Vector2(0.0, 0.28)
	elif serve_to == -1:
		_puck_pos = Vector2(0.0, -0.28)
	else:
		_puck_pos = Vector2.ZERO
	_puck_vel = Vector2(randf_range(-0.12, 0.12), 0.0)

	_player_pos = Vector2(0.0, 0.74)
	_player_target = _player_pos
	_player_vel = Vector2.ZERO
	_ai_pos = Vector2(0.0, -0.74)
	_ai_vel = Vector2.ZERO

	_sound_synth.play_faceoff()
	_sync_visuals(0.016)

func _input(event: InputEvent) -> void:
	if event is InputEventMouseMotion:
		if event.relative.length_squared() > 1.0:
			_using_mouse = true
			_last_mouse_pos = event.position

func _physics_process(delta: float) -> void:
	# Decay lighting & camera shake
	_goal_light_player.light_energy = lerpf(_goal_light_player.light_energy, 0.0, minf(1.0, delta * 5.0))
	_goal_light_opponent.light_energy = lerpf(_goal_light_opponent.light_energy, 0.0, minf(1.0, delta * 5.0))
	_camera_shake = maxf(0.0, _camera_shake - delta * 2.8)
	_update_camera(delta)

	if not _match_active:
		_sync_visuals(delta)
		return

	if not _round_live:
		_puck_drop_offset = lerpf(_puck_drop_offset, -0.10, minf(1.0, delta * 9.0))
		_goal_pause_timer -= delta
		_sync_visuals(delta)
		if _goal_pause_timer <= 0.0:
			if _player_score >= _target_score or _opponent_score >= _target_score:
				_match_active = false
				var player_won := _player_score >= _target_score
				if player_won:
					_sound_synth.play_victory()
				match_ended.emit(player_won, _player_score, _opponent_score, _max_puck_speed_recorded)
			else:
				# Loser of last point receives serve on their side
				var serve_side := 1 if _last_scorer_was_opponent else -1
				_reset_round(serve_side)
		return

	_update_player_target(delta)

	# Compute smooth linear player step across the frame so substepping doesn't spike velocity
	var frame_to_target := _player_target - _player_pos
	var max_frame_step := PLAYER_MAX_SPEED * delta
	var next_player_pos := _player_target
	if frame_to_target.length() > max_frame_step:
		next_player_pos = _player_pos + frame_to_target.normalized() * max_frame_step
	next_player_pos = _clamp_mallet_to_half(next_player_pos, true)
	var player_step_delta := (next_player_pos - _player_pos) / float(SUBSTEPS)
	_player_vel = (next_player_pos - _player_pos) / maxf(delta, 0.0001)

	var sub_dt := delta / float(SUBSTEPS)
	var hit_mallet_this_frame := false
	var hit_mallet_strength := 0.0
	var hit_wall_this_frame := false
	var hit_wall_strength := 0.0

	for _step in range(SUBSTEPS):
		# 1. Advance player mallet smoothly across substeps
		_player_pos = _clamp_mallet_to_half(_player_pos + player_step_delta, true)

		# 2. Advance AI mallet
		var prev_ai := _ai_pos
		_ai_pos = _step_ai(sub_dt)
		_ai_pos = _clamp_mallet_to_half(_ai_pos, false)
		_ai_vel = (_ai_pos - prev_ai) / maxf(sub_dt, 0.0001)

		# 3. Integrate puck
		_puck_pos += _puck_vel * sub_dt
		_puck_vel *= pow(AIR_FRICTION_PER_SEC, sub_dt)

		# 4. Mallet collisions
		var m_hit_p := _resolve_mallet_puck(_player_pos, _player_vel)
		if m_hit_p > 0.0:
			hit_mallet_this_frame = true
			hit_mallet_strength = maxf(hit_mallet_strength, m_hit_p)

		var m_hit_ai := _resolve_mallet_puck(_ai_pos, _ai_vel)
		if m_hit_ai > 0.0:
			hit_mallet_this_frame = true
			hit_mallet_strength = maxf(hit_mallet_strength, m_hit_ai)

		# 5. Table walls, corners, and goal posts
		var w_hit := _resolve_table_walls()
		if w_hit > 0.0:
			hit_wall_this_frame = true
			hit_wall_strength = maxf(hit_wall_strength, w_hit)

		# 6. Clamp max puck speed
		var spd := _puck_vel.length()
		if spd > MAX_PUCK_SPEED:
			_puck_vel = (_puck_vel / spd) * MAX_PUCK_SPEED
			spd = MAX_PUCK_SPEED
		if spd > _max_puck_speed_recorded:
			_max_puck_speed_recorded = spd

		# 7. Check goals
		if _check_goals():
			break

	if hit_mallet_this_frame:
		_sound_synth.play_mallet_hit(clampf(hit_mallet_strength / 4.5, 0.35, 1.1))
		if hit_mallet_strength > 2.2:
			_trigger_impact_sparks(_puck_pos)
	elif hit_wall_this_frame:
		_sound_synth.play_wall_hit(clampf(hit_wall_strength / 4.5, 0.25, 0.95))

	_sync_visuals(delta)

func _update_player_target(delta: float) -> void:
	var kb_input := Vector2(
		Input.get_action_strength("move_right") - Input.get_action_strength("move_left"),
		Input.get_action_strength("move_back") - Input.get_action_strength("move_forward")
	)

	if kb_input.length_squared() > 0.04:
		_using_mouse = false
		# Map keyboard input relative to current camera XZ forward/right
		var cam_basis := _camera.global_transform.basis
		var cam_right := Vector2(cam_basis.x.x, cam_basis.x.z).normalized()
		var cam_back := Vector2(cam_basis.z.x, cam_basis.z.z).normalized()
		var move_xz := (cam_right * kb_input.x + cam_back * kb_input.y).normalized()
		_player_target = _clamp_mallet_to_half(_player_target + move_xz * 1.65 * delta, true)
	elif _using_mouse:
		var viewport_mouse := get_viewport().get_mouse_position()
		var ray_origin := _camera.project_ray_origin(viewport_mouse)
		var ray_dir := _camera.project_ray_normal(viewport_mouse)
		var play_plane := Plane(Vector3.UP, PLAYFIELD_Y)
		var hit_3d = play_plane.intersects_ray(ray_origin, ray_dir)
		if hit_3d != null:
			_player_target = _clamp_mallet_to_half(Vector2(hit_3d.x, hit_3d.z), true)

func _clamp_mallet_to_half(pos: Vector2, is_player: bool) -> Vector2:
	var max_x := HALF_WIDTH - MALLET_RADIUS
	pos.x = clampf(pos.x, -max_x, max_x)

	if is_player:
		var min_z := MALLET_RADIUS + 0.015 # Stay on +Z half
		var max_z := HALF_LENGTH - MALLET_RADIUS
		pos.y = clampf(pos.y, min_z, max_z)
	else:
		var min_z := -(HALF_LENGTH - MALLET_RADIUS)
		var max_z := -(MALLET_RADIUS + 0.015) # Stay on -Z half
		pos.y = clampf(pos.y, min_z, max_z)

	# Respect 45-degree corner chamfers
	var max_sum := CORNER_CHAMFER_SUM - MALLET_RADIUS * 1.4142135
	var abs_sum := absf(pos.x) + absf(pos.y)
	if abs_sum > max_sum:
		var excess := (abs_sum - max_sum) * 0.5
		pos.x -= signf(pos.x) * excess
		pos.y -= signf(pos.y) * excess
	return pos

func _step_ai(sub_dt: float) -> Vector2:
	var ai_speed := 2.35
	var attack_aggression := 0.55
	var reaction_error := 0.04

	match _difficulty:
		Difficulty.CASUAL:
			ai_speed = 1.45
			attack_aggression = 0.35
			reaction_error = 0.07
		Difficulty.PRO:
			ai_speed = 2.35
			attack_aggression = 0.70
			reaction_error = 0.028
		Difficulty.ACE:
			ai_speed = 3.35
			attack_aggression = 0.92
			reaction_error = 0.010

	var desired := Vector2(clampf(_puck_pos.x, -0.24, 0.24), -0.84)

	if _puck_pos.y < -0.02:
		# Puck is in AI's half
		if _puck_pos.y < _ai_pos.y + 0.02:
			# Puck is behind AI mallet! Sidestep to avoid own-goal and circle behind
			var side_dir := 1.0 if _puck_pos.x < 0.0 else -1.0
			desired = Vector2(
				clampf(_puck_pos.x + side_dir * 0.16, -0.44, 0.44),
				maxf(_puck_pos.y - 0.14, -(HALF_LENGTH - MALLET_RADIUS - 0.02))
			)
		else:
			# Aim behind puck to strike toward player goal (+Z)
			var target_goal := Vector2(_ai_strafe_bias, HALF_LENGTH)
			var shot_dir := (target_goal - _puck_pos).normalized()
			var approach_dist := 0.08
			if _puck_pos.distance_to(_ai_pos) < 0.22 and _puck_vel.y > -2.0:
				# Drive straight through puck!
				desired = _puck_pos + shot_dir * 0.12
			else:
				desired = _puck_pos - shot_dir * approach_dist
	else:
		# Puck is in player's half: track predicted X at defensive line
		var predict_x := _puck_pos.x
		if _puck_vel.y < -0.35:
			var time_to_goal := (-0.82 - _puck_pos.y) / _puck_vel.y
			time_to_goal = clampf(time_to_goal, 0.0, 0.8)
			predict_x = _puck_pos.x + _puck_vel.x * time_to_goal
			# Simple reflection prediction on side walls
			var bound := HALF_WIDTH - PUCK_RADIUS
			if predict_x > bound:
				predict_x = bound - (predict_x - bound)
			elif predict_x < -bound:
				predict_x = -bound - (predict_x + bound)

		var def_z := lerpf(-0.90, -0.68, attack_aggression)
		desired = Vector2(clampf(predict_x + reaction_error * sin(Time.get_ticks_msec() * 0.004), -0.26, 0.26), def_z)

	var delta_vec := desired - _ai_pos
	var max_step := ai_speed * sub_dt
	if delta_vec.length() > max_step:
		return _ai_pos + delta_vec.normalized() * max_step
	return desired

func _resolve_mallet_puck(mallet_pos: Vector2, mallet_vel: Vector2) -> float:
	var diff := _puck_pos - mallet_pos
	var dist := diff.length()
	var min_dist := PUCK_RADIUS + MALLET_RADIUS
	if dist >= min_dist:
		return 0.0

	var n := Vector2(0.0, -1.0) if dist < 0.0001 else (diff / dist)
	_puck_pos = mallet_pos + n * (min_dist + 0.0005)

	var rel_vel := _puck_vel - mallet_vel
	var vn := rel_vel.dot(n)
	if vn < 0.0:
		var impulse := -(1.0 + MALLET_RESTITUTION) * vn
		_puck_vel += n * impulse
		# Subtle tangential English/deflection from mallet sweep
		var tangent := Vector2(-n.y, n.x)
		_puck_vel += tangent * (mallet_vel.dot(tangent) * 0.12)
		return absf(vn)
	return 0.0

func _resolve_table_walls() -> float:
	var max_impact := 0.0
	var max_x := HALF_WIDTH - PUCK_RADIUS
	var max_z := HALF_LENGTH - PUCK_RADIUS

	# 1. Side walls (X)
	if absf(_puck_pos.y) <= HALF_LENGTH - PUCK_RADIUS:
		if _puck_pos.x > max_x:
			_puck_pos.x = max_x
			if _puck_vel.x > 0.0:
				max_impact = maxf(max_impact, _puck_vel.x)
				_puck_vel.x = -_puck_vel.x * WALL_RESTITUTION
		elif _puck_pos.x < -max_x:
			_puck_pos.x = -max_x
			if _puck_vel.x < 0.0:
				max_impact = maxf(max_impact, -_puck_vel.x)
				_puck_vel.x = -_puck_vel.x * WALL_RESTITUTION

	# 2. Corner 45-degree chamfers (|x| + |z| <= 1.53 - PUCK_RADIUS * sqrt(2))
	var chamfer_limit := CORNER_CHAMFER_SUM - PUCK_RADIUS * 1.4142135
	var abs_sum := absf(_puck_pos.x) + absf(_puck_pos.y)
	if abs_sum > chamfer_limit:
		var sx := 1.0 if _puck_pos.x >= 0.0 else -1.0
		var sy := 1.0 if _puck_pos.y >= 0.0 else -1.0
		var n := Vector2(sx, sy).normalized()
		var pen := (abs_sum - chamfer_limit) / 1.4142135
		_puck_pos -= n * pen
		var vn := _puck_vel.dot(n)
		if vn > 0.0:
			max_impact = maxf(max_impact, vn)
			_puck_vel -= (1.0 + WALL_RESTITUTION) * vn * n

	# 3. Goal posts at (+/-GOAL_HALF_WIDTH, +/-HALF_LENGTH)
	for sx in [-1.0, 1.0]:
		for sy in [-1.0, 1.0]:
			var post := Vector2(sx * GOAL_HALF_WIDTH, sy * HALF_LENGTH)
			var d := _puck_pos - post
			var dist := d.length()
			var min_d := PUCK_RADIUS + GOAL_POST_RADIUS
			if dist < min_d and dist > 0.0001:
				var pn := d / dist
				_puck_pos = post + pn * min_d
				var vn := _puck_vel.dot(pn)
				if vn < 0.0:
					max_impact = maxf(max_impact, -vn)
					_puck_vel -= (1.0 + WALL_RESTITUTION) * vn * pn

	# 4. End walls outside the goal opening
	if absf(_puck_pos.x) >= GOAL_HALF_WIDTH:
		if _puck_pos.y > max_z:
			_puck_pos.y = max_z
			if _puck_vel.y > 0.0:
				max_impact = maxf(max_impact, _puck_vel.y)
				_puck_vel.y = -_puck_vel.y * WALL_RESTITUTION
		elif _puck_pos.y < -max_z:
			_puck_pos.y = -max_z
			if _puck_vel.y < 0.0:
				max_impact = maxf(max_impact, -_puck_vel.y)
				_puck_vel.y = -_puck_vel.y * WALL_RESTITUTION
	elif absf(_puck_pos.y) > max_z:
		# Inside goal throat: constrain X inside pocket walls
		var throat_x := GOAL_HALF_WIDTH - PUCK_RADIUS
		_puck_pos.x = clampf(_puck_pos.x, -throat_x, throat_x)

	return max_impact

var _last_scorer_was_opponent := false

func _check_goals() -> bool:
	if absf(_puck_pos.x) < GOAL_HALF_WIDTH:
		if _puck_pos.y > HALF_LENGTH + PUCK_RADIUS * 0.45:
			_on_goal(false)
			return true
		elif _puck_pos.y < -(HALF_LENGTH + PUCK_RADIUS * 0.45):
			_on_goal(true)
			return true
	return false

func _on_goal(by_player: bool) -> void:
	_round_live = false
	_goal_pause_timer = 1.6
	_puck_vel = Vector2.ZERO
	_last_scorer_was_opponent = not by_player

	if by_player:
		_player_score += 1
		_goal_light_opponent.light_color = Color(0.1, 0.9, 1.0)
		_goal_light_opponent.light_energy = 7.0
		_goal_burst.position = Vector3(0.0, PLAYFIELD_Y + 0.08, -HALF_LENGTH)
	else:
		_opponent_score += 1
		_goal_light_player.light_color = Color(1.0, 0.2, 0.5)
		_goal_light_player.light_energy = 7.0
		_goal_burst.position = Vector3(0.0, PLAYFIELD_Y + 0.08, HALF_LENGTH)

	_goal_burst.restart()
	_goal_burst.emitting = true
	_camera_shake = 0.55
	_sound_synth.play_goal(by_player)

	if _table_anim_player and _table_anim_player.has_animation("goal_flash"):
		_table_anim_player.play("goal_flash")
		if _table_anim_player.has_animation("idle"):
			_table_anim_player.queue("idle")

	score_changed.emit(_player_score, _opponent_score)
	goal_scored.emit(by_player, _player_score, _opponent_score)

func _trigger_impact_sparks(pos_xz: Vector2) -> void:
	_impact_sparks.position = Vector3(pos_xz.x, PLAYFIELD_Y + 0.015, pos_xz.y)
	_impact_sparks.restart()
	_impact_sparks.emitting = true

func _update_camera(delta: float) -> void:
	var shake_offset := Vector3.ZERO
	if _camera_shake > 0.001:
		shake_offset = Vector3(
			randf_range(-1.0, 1.0),
			randf_range(-1.0, 1.0),
			randf_range(-1.0, 1.0)
		) * (_camera_shake * 0.03)

	var target_pos := _cam_base_pos + shake_offset
	var cam_lerp := minf(1.0, delta * 7.5)
	_camera.position = _camera.position.lerp(target_pos, cam_lerp)
	_camera.fov = lerpf(_camera.fov, _cam_target_fov, cam_lerp)
	_camera.look_at(_cam_look_target, Vector3.UP)

func _sync_visuals(delta: float) -> void:
	_player_mallet.position = Vector3(_player_pos.x, PLAYFIELD_Y, _player_pos.y)
	_opponent_mallet.position = Vector3(_ai_pos.x, PLAYFIELD_Y, _ai_pos.y)
	_puck_node.position = Vector3(_puck_pos.x, PLAYFIELD_Y + _puck_drop_offset, _puck_pos.y)

	# Subtle tactile tilt on mallets during fast strikes
	var tilt_lerp := minf(1.0, delta * 14.0)
	_player_mallet.rotation.z = lerpf(_player_mallet.rotation.z, clampf(-_player_vel.x * 0.028, -0.09, 0.09), tilt_lerp)
	_player_mallet.rotation.x = lerpf(_player_mallet.rotation.x, clampf(_player_vel.y * 0.028, -0.09, 0.09), tilt_lerp)
	_opponent_mallet.rotation.z = lerpf(_opponent_mallet.rotation.z, clampf(-_ai_vel.x * 0.028, -0.09, 0.09), tilt_lerp)
	_opponent_mallet.rotation.x = lerpf(_opponent_mallet.rotation.x, clampf(_ai_vel.y * 0.028, -0.09, 0.09), tilt_lerp)

	# Scale puck neon glow gently with speed
	var speed_norm := clampf(_puck_vel.length() / MAX_PUCK_SPEED, 0.0, 1.0)
	_puck_light.light_energy = 0.65 + speed_norm * 0.95
```

### air_hockey/scripts/hud.gd
```gdscript
extends CanvasLayer

signal start_match_requested(difficulty: int, target_score: int)
signal restart_requested()
signal menu_requested()
signal camera_toggle_requested()
signal mute_toggle_requested()

@onready var _player_score_label: Label = %PlayerScoreLabel
@onready var _opponent_score_label: Label = %OpponentScoreLabel
@onready var _target_label: Label = %TargetLabel
@onready var _banner_label: Label = %BannerLabel
@onready var _camera_btn: Button = %CameraButton
@onready var _mute_btn: Button = %MuteButton

@onready var _menu_panel: PanelContainer = %MenuPanel
@onready var _difficulty_option: OptionButton = %DifficultyOption
@onready var _score_option: OptionButton = %ScoreOption
@onready var _play_button: Button = %PlayButton

@onready var _game_over_panel: PanelContainer = %GameOverPanel
@onready var _result_title: Label = %ResultTitle
@onready var _result_detail: Label = %ResultDetail
@onready var _play_again_btn: Button = %PlayAgainButton
@onready var _back_to_menu_btn: Button = %BackToMenuButton

var _banner_timer := 0.0

func _ready() -> void:
	_difficulty_option.clear()
	_difficulty_option.add_item("Casual", 0)
	_difficulty_option.add_item("Arcade Pro", 1)
	_difficulty_option.add_item("Cyber Ace", 2)
	_difficulty_option.select(1)

	_score_option.clear()
	_score_option.add_item("First to 5", 5)
	_score_option.add_item("First to 7", 7)
	_score_option.add_item("First to 10", 10)
	_score_option.select(1)

	_play_button.pressed.connect(_on_play_pressed)
	_play_again_btn.pressed.connect(func(): restart_requested.emit())
	_back_to_menu_btn.pressed.connect(func(): menu_requested.emit())
	_camera_btn.pressed.connect(func(): camera_toggle_requested.emit())
	_mute_btn.pressed.connect(func(): mute_toggle_requested.emit())
	%MenuButton.pressed.connect(func(): menu_requested.emit())
	%RestartButton.pressed.connect(func(): restart_requested.emit())

	_banner_label.visible = false
	_game_over_panel.visible = false
	_menu_panel.visible = true

func _process(delta: float) -> void:
	if _banner_timer > 0.0:
		_banner_timer -= delta
		if _banner_timer <= 0.0:
			_banner_label.visible = false

func _on_play_pressed() -> void:
	var diff := _difficulty_option.get_selected_id()
	var target := _score_option.get_selected_id()
	start_match_requested.emit(diff, target)

func show_menu() -> void:
	_menu_panel.visible = true
	_game_over_panel.visible = false
	_banner_label.visible = false

func show_gameplay(target_score: int) -> void:
	_menu_panel.visible = false
	_game_over_panel.visible = false
	_target_label.text = "FIRST TO %d" % target_score
	show_banner("FACE OFF!", Color(0.25, 0.9, 1.0), 1.3)

func update_scores(player_score: int, opponent_score: int) -> void:
	_player_score_label.text = str(player_score)
	_opponent_score_label.text = str(opponent_score)

func show_banner(msg: String, tint: Color, duration: float = 1.4) -> void:
	_banner_label.text = msg
	_banner_label.add_theme_color_override("font_color", tint)
	_banner_label.visible = true
	_banner_timer = duration

func show_game_over(player_won: bool, player_score: int, opponent_score: int, max_speed: float) -> void:
	_game_over_panel.visible = true
	if player_won:
		_result_title.text = "VICTORY!"
		_result_title.add_theme_color_override("font_color", Color(0.2, 0.92, 1.0))
	else:
		_result_title.text = "DEFEAT"
		_result_title.add_theme_color_override("font_color", Color(1.0, 0.25, 0.52))
	_result_detail.text = "Final Score: %d - %d\nTop Puck Speed: %.1f m/s" % [player_score, opponent_score, max_speed]

func update_mute_label(muted: bool) -> void:
	_mute_btn.text = "Sound: OFF [M]" if muted else "Sound: ON [M]"

func update_camera_label(mode_name: String) -> void:
	_camera_btn.text = "%s [C]" % mode_name
```

### air_hockey/scripts/main.gd
```gdscript
extends Node

@onready var _arena: Node3D = $Arena
@onready var _hud: CanvasLayer = $Hud

var _last_difficulty := 1
var _last_target_score := 7
var _in_match := false

func _ready() -> void:
	_hud.start_match_requested.connect(_on_start_match_requested)
	_hud.restart_requested.connect(_on_restart_requested)
	_hud.menu_requested.connect(_on_menu_requested)
	_hud.camera_toggle_requested.connect(_on_camera_toggle)
	_hud.mute_toggle_requested.connect(_on_mute_toggle)

	_arena.score_changed.connect(_on_score_changed)
	_arena.goal_scored.connect(_on_goal_scored)
	_arena.match_ended.connect(_on_match_ended)
	_arena.camera_mode_changed.connect(func(mode_name: String): _hud.update_camera_label(mode_name))

	_hud.update_scores(0, 0)
	_hud.show_menu()

func _unhandled_input(event: InputEvent) -> void:
	if event.is_action_pressed("toggle_camera"):
		_on_camera_toggle()
	elif event.is_action_pressed("toggle_mute"):
		_on_mute_toggle()
	elif event.is_action_pressed("reset_match") and _in_match:
		_on_restart_requested()
	elif event.is_action_pressed("ui_cancel"):
		_on_menu_requested()

func _on_start_match_requested(difficulty: int, target_score: int) -> void:
	_last_difficulty = difficulty
	_last_target_score = target_score
	_in_match = true
	_hud.show_gameplay(target_score)
	_arena.start_match(difficulty, target_score)

func _on_restart_requested() -> void:
	_in_match = true
	_hud.show_gameplay(_last_target_score)
	_arena.start_match(_last_difficulty, _last_target_score)

func _on_menu_requested() -> void:
	_in_match = false
	_arena.stop_match()
	_hud.show_menu()

func _on_camera_toggle() -> void:
	_arena.cycle_camera_mode()

func _on_mute_toggle() -> void:
	var next_muted: bool = not bool(_arena.is_muted())
	_arena.set_muted(next_muted)
	_hud.update_mute_label(next_muted)

func _on_score_changed(player_score: int, opponent_score: int) -> void:
	_hud.update_scores(player_score, opponent_score)

func _on_goal_scored(by_player: bool, player_score: int, opponent_score: int) -> void:
	if player_score >= _last_target_score or opponent_score >= _last_target_score:
		return
	if by_player:
		_hud.show_banner("PLAYER SCORES!", Color(0.18, 0.92, 1.0), 1.4)
	else:
		_hud.show_banner("OPPONENT SCORES!", Color(1.0, 0.25, 0.52), 1.4)

func _on_match_ended(player_won: bool, player_score: int, opponent_score: int, max_speed: float) -> void:
	_in_match = false
	_hud.show_game_over(player_won, player_score, opponent_score, max_speed)
```

### air_hockey/scenes/arena.tscn
```tscn
[gd_scene load_steps=22 format=3]

[ext_resource type="Script" path="res://scripts/arena.gd" id="1_arena_gd"]
[ext_resource type="Script" path="res://scripts/sound_synth.gd" id="2_sound_gd"]
[ext_resource type="PackedScene" path="res://assets/models/hockey_table.scad" id="3_table_scad"]
[ext_resource type="PackedScene" path="res://assets/models/player_mallet.scad" id="4_player_scad"]
[ext_resource type="PackedScene" path="res://assets/models/opponent_mallet.scad" id="5_opponent_scad"]
[ext_resource type="PackedScene" path="res://assets/models/hockey_puck.scad" id="6_puck_scad"]
[ext_resource type="PackedScene" path="res://assets/models/overhead_lamp.scad" id="7_lamp_scad"]

[sub_resource type="Environment" id="Environment_arcade"]
background_mode = 1
background_color = Color(0.075, 0.105, 0.195, 1)
ambient_light_source = 2
ambient_light_color = Color(0.24, 0.32, 0.52, 1)
ambient_light_energy = 0.36
tonemap_mode = 2
glow_enabled = true
glow_intensity = 0.55
glow_strength = 0.85
glow_bloom = 0.14
glow_hdr_threshold = 0.95
fog_enabled = true
fog_light_color = Color(0.075, 0.105, 0.195, 1)
fog_density = 0.042

[sub_resource type="StandardMaterial3D" id="StandardMaterial3D_floor"]
albedo_color = Color(0.085, 0.108, 0.175, 1)
metallic = 0.25
roughness = 0.52

[sub_resource type="PlaneMesh" id="PlaneMesh_floor"]
material = SubResource("StandardMaterial3D_floor")
size = Vector2(40, 40)

[sub_resource type="StandardMaterial3D" id="StandardMaterial3D_mat"]
albedo_color = Color(0.055, 0.072, 0.125, 1)
metallic = 0.2
roughness = 0.62

[sub_resource type="BoxMesh" id="BoxMesh_mat"]
material = SubResource("StandardMaterial3D_mat")
size = Vector3(2.2, 0.012, 3.4)

[sub_resource type="StandardMaterial3D" id="StandardMaterial3D_mat_trim"]
shading_mode = 0
albedo_color = Color(0.12, 0.78, 1.05, 1)

[sub_resource type="BoxMesh" id="BoxMesh_mat_trim_x"]
material = SubResource("StandardMaterial3D_mat_trim")
size = Vector3(2.24, 0.016, 0.022)

[sub_resource type="BoxMesh" id="BoxMesh_mat_trim_z"]
material = SubResource("StandardMaterial3D_mat_trim")
size = Vector3(0.022, 0.016, 3.44)

[sub_resource type="ParticleProcessMaterial" id="ParticleProcessMaterial_sparks"]
direction = Vector3(0, 1, 0)
spread = 85.0
initial_velocity_min = 2.0
initial_velocity_max = 4.8
gravity = Vector3(0, -6.5, 0)
scale_min = 0.7
scale_max = 1.6

[sub_resource type="StandardMaterial3D" id="StandardMaterial3D_spark_mesh"]
shading_mode = 0
albedo_color = Color(1.5, 0.92, 0.28, 1)

[sub_resource type="BoxMesh" id="BoxMesh_spark"]
material = SubResource("StandardMaterial3D_spark_mesh")
size = Vector3(0.013, 0.013, 0.013)

[sub_resource type="ParticleProcessMaterial" id="ParticleProcessMaterial_goal"]
direction = Vector3(0, 1, 0)
spread = 70.0
initial_velocity_min = 2.8
initial_velocity_max = 6.0
gravity = Vector3(0, -5.0, 0)
scale_min = 0.9
scale_max = 2.0

[sub_resource type="StandardMaterial3D" id="StandardMaterial3D_goal_mesh"]
shading_mode = 0
albedo_color = Color(0.3, 1.25, 1.45, 1)

[sub_resource type="BoxMesh" id="BoxMesh_goal"]
material = SubResource("StandardMaterial3D_goal_mesh")
size = Vector3(0.02, 0.02, 0.02)

[node name="Arena" type="Node3D"]
script = ExtResource("1_arena_gd")

[node name="WorldEnvironment" type="WorldEnvironment" parent="."]
environment = SubResource("Environment_arcade")

[node name="DirectionalLight3D" type="DirectionalLight3D" parent="."]
transform = Transform3D(0.766044, -0.454519, 0.454519, 0, 0.707107, 0.707107, -0.642788, -0.541675, 0.541675, 2.4, 4.2, 2.8)
light_color = Color(0.86, 0.93, 1, 1)
light_energy = 0.42
shadow_enabled = true
shadow_opacity = 0.28
shadow_blur = 2.4

[node name="TableSpotLight" type="SpotLight3D" parent="."]
transform = Transform3D(1, 0, 0, 0, -4.37114e-08, 1, 0, -1, -4.37114e-08, 0, 2.45, 0)
light_color = Color(0.92, 0.97, 1, 1)
light_energy = 0.78
spot_range = 5.0
spot_attenuation = 1.15
spot_angle = 58.0

[node name="UnderTableGlow" type="OmniLight3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.36, 0)
light_color = Color(0.14, 0.74, 1, 1)
light_energy = 1.8
omni_range = 4.2

[node name="RoomFillLeft" type="OmniLight3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, -2.8, 2.2, -2.8)
light_color = Color(0.22, 0.52, 1, 1)
light_energy = 1.4
omni_range = 8.5

[node name="RoomFillRight" type="OmniLight3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 2.8, 2.2, -2.8)
light_color = Color(0.85, 0.28, 0.68, 1)
light_energy = 1.2
omni_range = 8.5

[node name="FloorMesh" type="MeshInstance3D" parent="."]
mesh = SubResource("PlaneMesh_floor")

[node name="ArenaFloorMat" type="MeshInstance3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.006, 0)
mesh = SubResource("BoxMesh_mat")

[node name="MatTrimFront" type="MeshInstance3D" parent="ArenaFloorMat"]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.002, 1.71)
mesh = SubResource("BoxMesh_mat_trim_x")

[node name="MatTrimBack" type="MeshInstance3D" parent="ArenaFloorMat"]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.002, -1.71)
mesh = SubResource("BoxMesh_mat_trim_x")

[node name="MatTrimLeft" type="MeshInstance3D" parent="ArenaFloorMat"]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, -1.11, 0.002, 0)
mesh = SubResource("BoxMesh_mat_trim_z")

[node name="MatTrimRight" type="MeshInstance3D" parent="ArenaFloorMat"]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 1.11, 0.002, 0)
mesh = SubResource("BoxMesh_mat_trim_z")

[node name="OverheadLamp" parent="." instance=ExtResource("7_lamp_scad")]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 3.4, 0)

[node name="HockeyTable" parent="." instance=ExtResource("3_table_scad")]

[node name="PlayerMallet" type="Node3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.8, 0.74)

[node name="Model" parent="PlayerMallet" instance=ExtResource("4_player_scad")]

[node name="MalletLight" type="OmniLight3D" parent="PlayerMallet"]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.04, 0)
light_color = Color(0.12, 0.9, 1, 1)
light_energy = 0.55
omni_range = 0.45

[node name="OpponentMallet" type="Node3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.8, -0.74)

[node name="Model" parent="OpponentMallet" instance=ExtResource("5_opponent_scad")]

[node name="MalletLight" type="OmniLight3D" parent="OpponentMallet"]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.04, 0)
light_color = Color(1, 0.22, 0.52, 1)
light_energy = 0.55
omni_range = 0.45

[node name="HockeyPuck" type="Node3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.8, 0)

[node name="Model" parent="HockeyPuck" instance=ExtResource("6_puck_scad")]

[node name="PuckLight" type="OmniLight3D" parent="HockeyPuck"]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.035, 0)
light_color = Color(1, 0.52, 0.1, 1)
light_energy = 0.75
omni_range = 0.58

[node name="GoalLightPlayer" type="OmniLight3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.95, 1.16)
light_color = Color(1, 0.2, 0.5, 1)
light_energy = 0.0
omni_range = 1.6

[node name="GoalLightOpponent" type="OmniLight3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.95, -1.16)
light_color = Color(0.1, 0.9, 1, 1)
light_energy = 0.0
omni_range = 1.6

[node name="ImpactSparks" type="GPUParticles3D" parent="."]
emitting = false
amount = 18
lifetime = 0.3
one_shot = true
explosiveness = 0.95
process_material = SubResource("ParticleProcessMaterial_sparks")
draw_pass_1 = SubResource("BoxMesh_spark")

[node name="GoalBurst" type="GPUParticles3D" parent="."]
emitting = false
amount = 44
lifetime = 0.7
one_shot = true
explosiveness = 0.95
process_material = SubResource("ParticleProcessMaterial_goal")
draw_pass_1 = SubResource("BoxMesh_goal")

[node name="MainCamera" type="Camera3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 0.819152, 0.573576, 0, -0.573576, 0.819152, 0, 2.14, 2.04)
fov = 50.0

[node name="SoundSynth" type="Node" parent="."]
script = ExtResource("2_sound_gd")
```

### air_hockey/scenes/main.tscn
```tscn
[gd_scene load_steps=10 format=3]

[ext_resource type="Script" path="res://scripts/main.gd" id="1_main_gd"]
[ext_resource type="PackedScene" path="res://scenes/arena.tscn" id="2_arena_tscn"]
[ext_resource type="Script" path="res://scripts/hud.gd" id="3_hud_gd"]

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_topbar"]
bg_color = Color(0.035, 0.05, 0.095, 0.9)
border_width_left = 1
border_width_right = 1
border_width_bottom = 2
border_color = Color(0.12, 0.82, 1, 0.65)
corner_radius_bottom_left = 14
corner_radius_bottom_right = 14
content_margin_left = 20.0
content_margin_top = 8.0
content_margin_right = 20.0
content_margin_bottom = 8.0

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_panel"]
bg_color = Color(0.045, 0.062, 0.115, 0.94)
border_width_left = 2
border_width_top = 2
border_width_right = 2
border_width_bottom = 2
border_color = Color(0.15, 0.85, 1, 0.8)
corner_radius_top_left = 16
corner_radius_top_right = 16
corner_radius_bottom_right = 16
corner_radius_bottom_left = 16
content_margin_left = 28.0
content_margin_top = 24.0
content_margin_right = 28.0
content_margin_bottom = 24.0

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_btn_normal"]
bg_color = Color(0.08, 0.12, 0.21, 0.92)
border_width_left = 1
border_width_top = 1
border_width_right = 1
border_width_bottom = 1
border_color = Color(0.18, 0.78, 0.98, 0.55)
corner_radius_top_left = 8
corner_radius_top_right = 8
corner_radius_bottom_right = 8
corner_radius_bottom_left = 8
content_margin_left = 12.0
content_margin_top = 6.0
content_margin_right = 12.0
content_margin_bottom = 6.0

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_btn_hover"]
bg_color = Color(0.12, 0.22, 0.36, 0.96)
border_width_left = 1
border_width_top = 1
border_width_right = 1
border_width_bottom = 1
border_color = Color(0.28, 0.92, 1, 0.92)
corner_radius_top_left = 8
corner_radius_top_right = 8
corner_radius_bottom_right = 8
corner_radius_bottom_left = 8
content_margin_left = 12.0
content_margin_top = 6.0
content_margin_right = 12.0
content_margin_bottom = 6.0

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_btn_pressed"]
bg_color = Color(0.16, 0.34, 0.52, 1)
border_width_left = 1
border_width_top = 1
border_width_right = 1
border_width_bottom = 1
border_color = Color(0.45, 0.96, 1, 1)
corner_radius_top_left = 8
corner_radius_top_right = 8
corner_radius_bottom_right = 8
corner_radius_bottom_left = 8
content_margin_left = 12.0
content_margin_top = 6.0
content_margin_right = 12.0
content_margin_bottom = 6.0

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_hint"]
bg_color = Color(0.035, 0.05, 0.095, 0.78)
border_width_left = 1
border_width_top = 1
border_width_right = 1
border_width_bottom = 1
border_color = Color(0.16, 0.72, 0.94, 0.35)
corner_radius_top_left = 8
corner_radius_top_right = 8
corner_radius_bottom_right = 8
corner_radius_bottom_left = 8
content_margin_left = 12.0
content_margin_top = 5.0
content_margin_right = 12.0
content_margin_bottom = 5.0

[sub_resource type="Theme" id="Theme_arcade"]
Button/colors/font_color = Color(0.88, 0.95, 1, 1)
Button/colors/font_hover_color = Color(1, 1, 1, 1)
Button/colors/font_pressed_color = Color(0.65, 0.98, 1, 1)
Button/styles/hover = SubResource("StyleBoxFlat_btn_hover")
Button/styles/normal = SubResource("StyleBoxFlat_btn_normal")
Button/styles/pressed = SubResource("StyleBoxFlat_btn_pressed")

[node name="Main" type="Node"]
script = ExtResource("1_main_gd")

[node name="Arena" parent="." instance=ExtResource("2_arena_tscn")]

[node name="Hud" type="CanvasLayer" parent="."]
script = ExtResource("3_hud_gd")

[node name="RootControl" type="Control" parent="Hud"]
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
mouse_filter = 2
theme = SubResource("Theme_arcade")

[node name="TopBar" type="PanelContainer" parent="Hud/RootControl"]
layout_mode = 1
anchors_preset = 5
anchor_left = 0.5
anchor_right = 0.5
offset_left = -390.0
offset_right = 390.0
offset_bottom = 66.0
grow_horizontal = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_topbar")

[node name="HBox" type="HBoxContainer" parent="Hud/RootControl/TopBar"]
layout_mode = 2
theme_override_constants/separation = 14
alignment = 1

[node name="PlayerTag" type="Label" parent="Hud/RootControl/TopBar/HBox"]
layout_mode = 2
theme_override_colors/font_color = Color(0.18, 0.92, 1, 1)
theme_override_font_sizes/font_size = 18
text = "YOU"

[node name="PlayerScoreLabel" type="Label" parent="Hud/RootControl/TopBar/HBox"]
unique_name_in_owner = true
layout_mode = 2
theme_override_colors/font_color = Color(0.28, 0.95, 1, 1)
theme_override_font_sizes/font_size = 34
text = "0"

[node name="CenterInfo" type="VBoxContainer" parent="Hud/RootControl/TopBar/HBox"]
layout_mode = 2
alignment = 1

[node name="Dash" type="Label" parent="Hud/RootControl/TopBar/HBox/CenterInfo"]
layout_mode = 2
theme_override_font_sizes/font_size = 18
text = "VS"
horizontal_alignment = 1

[node name="TargetLabel" type="Label" parent="Hud/RootControl/TopBar/HBox/CenterInfo"]
unique_name_in_owner = true
layout_mode = 2
theme_override_colors/font_color = Color(0.68, 0.78, 0.92, 1)
theme_override_font_sizes/font_size = 11
text = "FIRST TO 7"
horizontal_alignment = 1

[node name="OpponentScoreLabel" type="Label" parent="Hud/RootControl/TopBar/HBox"]
unique_name_in_owner = true
layout_mode = 2
theme_override_colors/font_color = Color(1, 0.25, 0.54, 1)
theme_override_font_sizes/font_size = 34
text = "0"

[node name="OpponentTag" type="Label" parent="Hud/RootControl/TopBar/HBox"]
layout_mode = 2
theme_override_colors/font_color = Color(1, 0.25, 0.54, 1)
theme_override_font_sizes/font_size = 18
text = "AI"

[node name="Spacer" type="Control" parent="Hud/RootControl/TopBar/HBox"]
custom_minimum_size = Vector2(14, 0)
layout_mode = 2

[node name="CameraButton" type="Button" parent="Hud/RootControl/TopBar/HBox"]
unique_name_in_owner = true
layout_mode = 2
text = "Player View [C]"

[node name="MuteButton" type="Button" parent="Hud/RootControl/TopBar/HBox"]
unique_name_in_owner = true
layout_mode = 2
text = "Sound: ON [M]"

[node name="RestartButton" type="Button" parent="Hud/RootControl/TopBar/HBox"]
unique_name_in_owner = true
layout_mode = 2
text = "Restart [R]"

[node name="MenuButton" type="Button" parent="Hud/RootControl/TopBar/HBox"]
unique_name_in_owner = true
layout_mode = 2
text = "Menu [Esc]"

[node name="BannerLabel" type="Label" parent="Hud/RootControl"]
unique_name_in_owner = true
visible = false
layout_mode = 1
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -260.0
offset_top = -140.0
offset_right = 260.0
offset_bottom = -70.0
grow_horizontal = 2
grow_vertical = 2
theme_override_colors/font_shadow_color = Color(0, 0, 0, 0.85)
theme_override_constants/shadow_offset_x = 3
theme_override_constants/shadow_offset_y = 3
theme_override_font_sizes/font_size = 46
text = "FACE OFF!"
horizontal_alignment = 1
vertical_alignment = 1

[node name="ControlsBar" type="PanelContainer" parent="Hud/RootControl"]
layout_mode = 1
anchors_preset = 2
anchor_top = 1.0
anchor_bottom = 1.0
offset_left = 14.0
offset_top = -42.0
offset_right = 566.0
offset_bottom = -10.0
grow_vertical = 0
mouse_filter = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_hint")

[node name="ControlsHint" type="Label" parent="Hud/RootControl/ControlsBar"]
layout_mode = 2
theme_override_colors/font_color = Color(0.8, 0.88, 0.98, 0.92)
theme_override_font_sizes/font_size = 13
text = "Mouse or WASD / Arrows: Move Mallet   |   C: Camera   |   R: Restart   |   M: Mute"

[node name="MenuPanel" type="PanelContainer" parent="Hud/RootControl"]
unique_name_in_owner = true
layout_mode = 1
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -195.0
offset_top = -155.0
offset_right = 195.0
offset_bottom = 155.0
grow_horizontal = 2
grow_vertical = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_panel")

[node name="VBox" type="VBoxContainer" parent="Hud/RootControl/MenuPanel"]
layout_mode = 2
theme_override_constants/separation = 14

[node name="TitleLabel" type="Label" parent="Hud/RootControl/MenuPanel/VBox"]
layout_mode = 2
theme_override_colors/font_color = Color(0.2, 0.9, 1, 1)
theme_override_font_sizes/font_size = 32
text = "NEON AIR HOCKEY"
horizontal_alignment = 1

[node name="SubtitleLabel" type="Label" parent="Hud/RootControl/MenuPanel/VBox"]
layout_mode = 2
theme_override_colors/font_color = Color(0.75, 0.82, 0.95, 1)
theme_override_font_sizes/font_size = 14
text = "Fast-paced Arcade Table Hockey"
horizontal_alignment = 1

[node name="DiffRow" type="HBoxContainer" parent="Hud/RootControl/MenuPanel/VBox"]
layout_mode = 2

[node name="Label" type="Label" parent="Hud/RootControl/MenuPanel/VBox/DiffRow"]
layout_mode = 2
size_flags_horizontal = 3
text = "AI Difficulty"

[node name="DifficultyOption" type="OptionButton" parent="Hud/RootControl/MenuPanel/VBox/DiffRow"]
unique_name_in_owner = true
custom_minimum_size = Vector2(150, 0)
layout_mode = 2

[node name="ScoreRow" type="HBoxContainer" parent="Hud/RootControl/MenuPanel/VBox"]
layout_mode = 2

[node name="Label" type="Label" parent="Hud/RootControl/MenuPanel/VBox/ScoreRow"]
layout_mode = 2
size_flags_horizontal = 3
text = "Match Length"

[node name="ScoreOption" type="OptionButton" parent="Hud/RootControl/MenuPanel/VBox/ScoreRow"]
unique_name_in_owner = true
custom_minimum_size = Vector2(150, 0)
layout_mode = 2

[node name="PlayButton" type="Button" parent="Hud/RootControl/MenuPanel/VBox"]
unique_name_in_owner = true
custom_minimum_size = Vector2(0, 46)
layout_mode = 2
theme_override_font_sizes/font_size = 20
text = "PLAY MATCH"

[node name="GameOverPanel" type="PanelContainer" parent="Hud/RootControl"]
unique_name_in_owner = true
visible = false
layout_mode = 1
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -185.0
offset_top = -130.0
offset_right = 185.0
offset_bottom = 130.0
grow_horizontal = 2
grow_vertical = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_panel")

[node name="VBox" type="VBoxContainer" parent="Hud/RootControl/GameOverPanel"]
layout_mode = 2
theme_override_constants/separation = 14

[node name="ResultTitle" type="Label" parent="Hud/RootControl/GameOverPanel/VBox"]
unique_name_in_owner = true
layout_mode = 2
theme_override_font_sizes/font_size = 36
text = "VICTORY!"
horizontal_alignment = 1

[node name="ResultDetail" type="Label" parent="Hud/RootControl/GameOverPanel/VBox"]
unique_name_in_owner = true
layout_mode = 2
theme_override_colors/font_color = Color(0.85, 0.9, 1, 1)
theme_override_font_sizes/font_size = 16
text = "Final Score: 7 - 4"
horizontal_alignment = 1

[node name="PlayAgainButton" type="Button" parent="Hud/RootControl/GameOverPanel/VBox"]
unique_name_in_owner = true
custom_minimum_size = Vector2(0, 42)
layout_mode = 2
theme_override_font_sizes/font_size = 18
text = "PLAY AGAIN"

[node name="BackToMenuButton" type="Button" parent="Hud/RootControl/GameOverPanel/VBox"]
unique_name_in_owner = true
custom_minimum_size = Vector2(0, 38)
layout_mode = 2
text = "MAIN MENU"
```