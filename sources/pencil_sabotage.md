### pencil_sabotage/project.godot
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

config/name="Pencil Sabotage"
run/main_scene="res://scenes/main_scene.tscn"
config/features=PackedStringArray("4.3", "Forward Plus")

[editor_plugins]

enabled=PackedStringArray("res://addons/scad_importer/plugin.cfg")

[input]

lean_left={
"deadzone": 0.2,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":65,"key_label":0,"unicode":97,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194319,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
lean_right={
"deadzone": 0.2,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":68,"key_label":0,"unicode":100,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194321,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
restart_game={
"deadzone": 0.2,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":82,"key_label":0,"unicode":114,"location":0,"echo":false,"script":null)
]
}

[rendering]

lights_and_shadows/directional_shadow/soft_shadow_filter_quality=3
anti_aliasing/quality/msaa_3d=2
```

### pencil_sabotage/addons/scad_importer/plugin.cfg
```text
[plugin]

name="OpenSCAD GLTF Importer"
description="Imports .scad files directly as 3D scenes using scad-gltf"
author="scad-gltf"
version="0.2"
script="scad_plugin.gd"

```

### pencil_sabotage/addons/scad_importer/scad_importer.gd
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

### pencil_sabotage/addons/scad_importer/scad_plugin.gd
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

### pencil_sabotage/assets/models/pencil.scad
```openscad
// Standard wooden pencil oriented with sharp tip facing +Y (Forward)
// Total length ~0.24m, slightly chunky radius (0.009m) for clear visibility in flight

module pencil_model() {
    body_len = 0.175;
    wood_len = 0.032;
    lead_len = 0.011;
    ferrule_len = 0.018;
    eraser_len = 0.016;
    rad = 0.009;

    // Center roughly near balance point; tip extends toward +Y
    translate([0, -0.04, 0]) {
        // Painted hexagonal barrel
        color([0.96, 0.75, 0.10], name="pencil_barrel", roughness=0.42, metalness=0.0, $asa=0.0)
        rotate([-90, 0, 0])
            cylinder(h=body_len, r=rad, $fn=6);

        // Sharpened wood cone
        color([0.85, 0.68, 0.48], name="pencil_wood", roughness=0.78, metalness=0.0, $asa=35.0)
        translate([0, body_len, 0])
        rotate([-90, 0, 0])
            cylinder(h=wood_len, r1=rad * 0.94, r2=0.0028, $fn=24);

        // Graphite tip
        color([0.14, 0.14, 0.16], name="pencil_graphite", roughness=0.35, metalness=0.25, $asa=35.0)
        translate([0, body_len + wood_len - 0.002, 0])
        rotate([-90, 0, 0])
            cylinder(h=lead_len, r1=0.0030, r2=0.0004, $fn=16);

        // Metallic ferrule with crimp rings
        color([0.82, 0.84, 0.87], name="pencil_ferrule", roughness=0.25, metalness=0.95, $asa=45.0)
        translate([0, -ferrule_len, 0])
        rotate([-90, 0, 0]) {
            cylinder(h=ferrule_len, r=rad * 0.96, $fn=24);
            translate([0, 0, 0.004]) cylinder(h=0.002, r=rad * 1.03, $fn=24);
            translate([0, 0, 0.012]) cylinder(h=0.002, r=rad * 1.03, $fn=24);
        }

        // Pink rubber eraser
        color([0.92, 0.45, 0.52], name="pencil_eraser", roughness=0.88, metalness=0.0, $asa=45.0)
        translate([0, -ferrule_len - eraser_len, 0])
        rotate([-90, 0, 0])
            cylinder(h=eraser_len, r=rad * 0.90, $fn=24);
    }
}

pencil_model();
```

### pencil_sabotage/assets/models/sofa.scad
```openscad
// Cozy living room sofa (~2.2m wide, 0.95m deep, 0.88m tall)
// Faces +Y (Forward), sits on Z=0

module rounded_box(size, r) {
    sx = size[0];
    sy = size[1];
    sz = size[2];
    hull() {
        for (ix = [-1, 1], iy = [-1, 1], iz = [-1, 1]) {
            translate([
                ix * (sx * 0.5 - r),
                iy * (sy * 0.5 - r),
                iz * (sz * 0.5 - r)
            ])
            sphere(r=r, $fn=16);
        }
    }
}

module sofa_model() {
    // Wooden legs
    color([0.28, 0.17, 0.09], name="sofa_wood", roughness=0.55, metalness=0.0, $asa=35.0) {
        for (lx = [-0.95, 0.95], ly = [-0.36, 0.36]) {
            translate([lx, ly, 0])
                cylinder(h=0.10, r1=0.025, r2=0.036, $fn=16);
        }
    }

    // Main base frame
    color([0.20, 0.28, 0.38], name="sofa_fabric_dark", roughness=0.92, metalness=0.0, $asa=40.0) {
        translate([0, -0.02, 0.21])
            rounded_box([2.16, 0.88, 0.22], 0.04);

        // Backrest frame
        translate([0, -0.36, 0.54])
            rounded_box([2.16, 0.22, 0.68], 0.05);

        // Left (-X) and Right (+X) Armrests
        translate([-0.97, 0.02, 0.42])
            rounded_box([0.22, 0.92, 0.42], 0.05);
        translate([0.97, 0.02, 0.42])
            rounded_box([0.22, 0.92, 0.42], 0.05);
    }

    // Seat cushions (3 cushions)
    color([0.26, 0.36, 0.48], name="sofa_fabric_cushion", roughness=0.90, metalness=0.0, $asa=45.0) {
        for (cx = [-0.57, 0.0, 0.57]) {
            translate([cx, 0.06, 0.36])
                rounded_box([0.55, 0.74, 0.14], 0.045);
        }
        // Back cushions
        for (cx = [-0.57, 0.0, 0.57]) {
            translate([cx, -0.22, 0.62])
            rotate([10, 0, 0])
                rounded_box([0.55, 0.16, 0.42], 0.05);
        }
    }

    // Accent throw pillows
    color([0.85, 0.42, 0.22], name="sofa_pillow_orange", roughness=0.88, metalness=0.0, $asa=45.0) {
        translate([-0.74, -0.05, 0.54])
        rotate([12, 18, -22])
            rounded_box([0.32, 0.11, 0.32], 0.045);
    }
    color([0.88, 0.72, 0.28], name="sofa_pillow_mustard", roughness=0.88, metalness=0.0, $asa=45.0) {
        translate([0.74, -0.05, 0.54])
        rotate([12, -16, 20])
            rounded_box([0.32, 0.11, 0.32], 0.045);
    }
}

sofa_model();
```

### pencil_sabotage/assets/models/coffee_table.scad
```openscad
// Low wooden coffee table in front of the sofa (1.0m x 0.58m x 0.42m)

module coffee_table_model() {
    // Tabletop
    color([0.48, 0.30, 0.16], name="table_wood", roughness=0.45, metalness=0.0, $asa=20.0) {
        translate([0, 0, 0.40])
            cube([1.00, 0.58, 0.035], center=true);
        // Lower magazine shelf
        translate([0, 0, 0.14])
            cube([0.86, 0.44, 0.02], center=true);
    }

    // Black matte metal frame legs
    color([0.12, 0.12, 0.14], name="table_metal", roughness=0.5, metalness=0.85, $asa=20.0) {
        for (lx = [-0.45, 0.45], ly = [-0.24, 0.24]) {
            translate([lx, ly, 0.195])
                cube([0.03, 0.03, 0.39], center=true);
        }
    }


}

coffee_table_model();
```

### pencil_sabotage/assets/models/pencil_mug.scad
```openscad
// Ceramic mug filled with ready-to-throw pencils

module mini_pencil(ang_x, ang_y, rot_z) {
    rotate([ang_x, ang_y, rot_z]) {
        color([0.96, 0.75, 0.10], name="mug_pencil_body", roughness=0.42, metalness=0.0, $asa=0.0)
            cylinder(h=0.14, r=0.006, $fn=6);
        color([0.85, 0.68, 0.48], name="mug_pencil_wood", roughness=0.75, metalness=0.0, $asa=35.0)
        translate([0, 0, 0.14])
            cylinder(h=0.025, r1=0.0056, r2=0.0015, $fn=16);
        color([0.14, 0.14, 0.16], name="mug_pencil_tip", roughness=0.35, metalness=0.2, $asa=35.0)
        translate([0, 0, 0.163])
            cylinder(h=0.008, r1=0.0018, r2=0.0003, $fn=12);
    }
}

module pencil_mug_model() {
    // Coaster directly underneath the mug
    color([0.18, 0.45, 0.42], name="coaster", roughness=0.78, metalness=0.0, $asa=35.0)
        cylinder(h=0.008, r=0.064, $fn=32);

    // Ceramic mug resting flush on top of the coaster
    translate([0, 0, 0.008]) {
        color([0.92, 0.90, 0.86], name="ceramic_mug", roughness=0.22, metalness=0.0, $asa=45.0) {
            difference() {
                cylinder(h=0.115, r=0.048, $fn=28);
                translate([0, 0, 0.01])
                    cylinder(h=0.11, r=0.040, $fn=28);
            }
            // Handle on +X side
            translate([0.048, 0, 0.058])
            rotate([90, 0, 0])
                difference() {
                    cylinder(h=0.014, r=0.028, center=true, $fn=24);
                    cylinder(h=0.020, r=0.017, center=true, $fn=24);
                }
        }

        // Bunch of pencils sticking up out of the mug
        translate([0.0, 0.0, 0.015]) {
            translate([0.012, 0.008, 0]) mini_pencil(8, -10, 15);
            translate([-0.014, 0.010, 0]) mini_pencil(10, 12, 70);
            translate([0.004, -0.014, 0]) mini_pencil(-11, 6, 140);
            translate([-0.010, -0.012, 0]) mini_pencil(-9, -8, 220);
            translate([0.016, -0.006, 0]) mini_pencil(5, -14, 300);
            translate([0.0, 0.0, 0]) mini_pencil(3, 4, 45);
        }
    }
}

pencil_mug_model();
```

### pencil_sabotage/assets/models/computer_desk.scad
```openscad
// Wooden computer desk (1.6m wide, 0.78m deep, 0.76m tall)
// Built facing +Y (gamer sits at -Y looking toward +Y)

module computer_desk_model() {
    // Desk wooden top
    color([0.52, 0.35, 0.21], name="desk_wood", roughness=0.45, metalness=0.0, $asa=15.0)
        translate([0, 0, 0.745])
            cube([1.62, 0.78, 0.032], center=true);

    // Matte black steel legs and crossbars
    color([0.11, 0.11, 0.13], name="desk_metal", roughness=0.48, metalness=0.88, $asa=20.0) {
        for (lx = [-0.75, 0.75], ly = [-0.33, 0.33]) {
            translate([lx, ly, 0.365])
                cube([0.045, 0.045, 0.73], center=true);
        }
        // Bottom side runners
        for (lx = [-0.75, 0.75]) {
            translate([lx, 0, 0.05])
                cube([0.045, 0.68, 0.04], center=true);
        }
        // Rear stabilizer bar
        translate([0, 0.33, 0.24])
            cube([1.50, 0.03, 0.04], center=true);
    }

    // Large desk mat
    color([0.09, 0.10, 0.12], name="desk_mat", roughness=0.92, metalness=0.0, $asa=15.0)
        translate([-0.14, -0.08, 0.763])
            cube([0.78, 0.36, 0.005], center=true);

    // Mechanical keyboard with glowing keys
    color([0.14, 0.15, 0.18], name="keyboard_body", roughness=0.45, metalness=0.3, $asa=15.0)
        translate([-0.22, -0.10, 0.773])
            cube([0.36, 0.13, 0.018], center=true);

    color([0.2, 0.85, 1.0], name="keyboard_rgb", roughness=0.3, metalness=0.0, emissive=[0.15, 0.75, 1.0], emissiveIntensity=1.6, $asa=0.0)
        translate([-0.22, -0.10, 0.783])
            cube([0.33, 0.10, 0.005], center=true);

    // Gaming mouse
    color([0.13, 0.13, 0.15], name="mouse_body", roughness=0.38, metalness=0.1, $asa=45.0)
        translate([0.12, -0.10, 0.776])
        scale([1.0, 1.5, 0.7])
            sphere(r=0.032, $fn=24);

    color([1.0, 0.25, 0.55], name="mouse_rgb", roughness=0.3, emissive=[1.0, 0.2, 0.5], emissiveIntensity=1.5, $asa=35.0)
        translate([0.12, -0.11, 0.785])
            cube([0.012, 0.04, 0.01], center=true);

    // Energy drink can on desk corner
    color([0.10, 0.72, 0.35], name="soda_can", roughness=0.25, metalness=0.85, $asa=45.0)
        translate([-0.64, -0.16, 0.761])
            cylinder(h=0.12, r=0.032, $fn=24);
}

computer_desk_model();
```

### pencil_sabotage/assets/models/monitor.scad
```openscad
// Gaming monitor (0.64m wide, 0.38m tall panel)
// Screen faces +Y (Forward) so rotating 180 deg in Godot faces +Z

module monitor_model() {
    // Stand base
    color([0.12, 0.12, 0.14], name="monitor_stand", roughness=0.40, metalness=0.8, $asa=20.0) {
        translate([0, -0.02, 0.01])
            cube([0.28, 0.20, 0.02], center=true);
        // Vertical neck
        translate([0, -0.06, 0.17])
            cube([0.06, 0.04, 0.32], center=true);
    }

    // Monitor bezel & back shell
    color([0.10, 0.10, 0.12], name="monitor_bezel", roughness=0.42, metalness=0.2, $asa=15.0) {
        translate([0, -0.015, 0.32])
            cube([0.66, 0.03, 0.40], center=true);
        // Back bulge
        translate([0, -0.038, 0.32])
            cube([0.40, 0.03, 0.24], center=true);
    }

    // Active screen surface (named "monitor_screen" for runtime ShaderMaterial replacement)
    // Offset cleanly 8mm in front of bezel to prevent WebGL depth-buffer Z-fighting
    color([0.08, 0.16, 0.24], name="monitor_screen", roughness=0.18, metalness=0.05, emissive=[0.2, 0.55, 1.0], emissiveIntensity=1.4, $asa=0.0)
        translate([0, 0.008, 0.32])
            cube([0.62, 0.008, 0.36], center=true);

    // Status LED on bottom right of bezel
    color([0.2, 1.0, 0.4], name="monitor_led", roughness=0.3, emissive=[0.2, 1.0, 0.4], emissiveIntensity=2.0, $asa=0.0)
        translate([0.27, 0.005, 0.13])
            cube([0.018, 0.008, 0.006], center=true);
}

monitor_model();
```

### pencil_sabotage/assets/models/pc_tower.scad
```openscad
// Open-side ATX Gaming PC Tower (~0.22m W x 0.46m D x 0.48m H)
// Front faces +Y, Left side (-X) is OPEN exposing detailed internal hardware!
// Animated spinning fans via armature()

pc_anims = [
    ["running", [
        ["CPUFan", [
            [0.00, [0, 0, 0]],
            [0.20, [90, 0, 0]],
            [0.40, [180, 0, 0]],
            [0.60, [270, 0, 0]],
            [0.80, [360, 0, 0]]
        ]],
        ["FrontFan", [
            [0.00, [0, 0, 0]],
            [0.20, [0, 90, 0]],
            [0.40, [0, 180, 0]],
            [0.60, [0, 270, 0]],
            [0.80, [0, 360, 0]]
        ]],
        ["GPUFan1", [
            [0.00, [0, 0, 0]],
            [0.20, [0, 0, 90]],
            [0.40, [0, 0, 180]],
            [0.60, [0, 0, 270]],
            [0.80, [0, 0, 360]]
        ]],
        ["GPUFan2", [
            [0.00, [0, 0, 0]],
            [0.20, [0, 0, 90]],
            [0.40, [0, 0, 180]],
            [0.60, [0, 0, 270]],
            [0.80, [0, 0, 360]]
        ]]
    ]],
    ["broken", [
        ["CPUFan",  [[0.0, [18, 0, 0]]]],
        ["FrontFan",[[0.0, [0, 12, 0]]]],
        ["GPUFan1", [[0.0, [0, 0, 25]]]],
        ["GPUFan2", [[0.0, [0, 0, -15]]]]
    ]]
];

module fan_rotor_x(radius, blade_col) {
    color([0.14, 0.15, 0.17], name="fan_hub", roughness=0.35, metalness=0.3, $asa=45.0)
        rotate([0, 90, 0])
            cylinder(h=0.018, r=radius * 0.28, center=true, $fn=20);

    color(blade_col, name="fan_blades_glow", roughness=0.32, emissive=blade_col, emissiveIntensity=1.1, $asa=20.0)
    for (i = [0 : 6]) {
        rotate([i * (360 / 7), 0, 0])
        translate([0, radius * 0.55, 0])
        rotate([0, 28, 0])
            cube([0.004, radius * 0.62, radius * 0.23], center=true);
    }
}

module fan_rotor_y(radius, blade_col) {
    color([0.14, 0.15, 0.17], name="fan_hub", roughness=0.35, metalness=0.3, $asa=45.0)
        rotate([90, 0, 0])
            cylinder(h=0.018, r=radius * 0.28, center=true, $fn=20);

    color(blade_col, name="fan_blades_glow", roughness=0.32, emissive=blade_col, emissiveIntensity=1.1, $asa=20.0)
    for (i = [0 : 6]) {
        rotate([0, i * (360 / 7), 0])
        translate([radius * 0.55, 0, 0])
        rotate([28, 0, 0])
            cube([radius * 0.62, 0.004, radius * 0.23], center=true);
    }
}

module fan_rotor_z(radius, blade_col) {
    color([0.15, 0.16, 0.18], name="fan_hub", roughness=0.35, metalness=0.3, $asa=45.0)
        cylinder(h=0.012, r=radius * 0.28, center=true, $fn=18);

    color(blade_col, name="gpu_fan_blades", roughness=0.42, metalness=0.15, $asa=20.0)
    for (i = [0 : 6]) {
        rotate([0, 0, i * (360 / 7)])
        translate([radius * 0.54, 0, 0])
        rotate([24, 0, 0])
            cube([radius * 0.60, radius * 0.20, 0.003], center=true);
    }
}

module cable_segment(p1, p2, r) {
    hull() {
        translate(p1) sphere(r=r, $fn=12);
        translate(p2) sphere(r=r, $fn=12);
    }
}

armature(animations=pc_anims) {
    bone(name="CaseRoot", t=[0, 0, 0], r=[0, 0, 0]) {
        // Rubber feet with metallic trim rings
        color([0.07, 0.07, 0.08], name="pc_feet", roughness=0.85, metalness=0.0, $asa=35.0) {
            for (fx = [-0.08, 0.08], fy = [-0.19, 0.19]) {
                translate([fx, fy, 0.009])
                    cylinder(h=0.018, r1=0.017, r2=0.020, center=true, $fn=18);
            }
        }

        // Main outer case shell (Left side -X is completely open!)
        color([0.12, 0.13, 0.16], name="pc_case_metal", roughness=0.40, metalness=0.82, $asa=10.0) {
            // Bottom plate
            translate([0, 0, 0.025])
                cube([0.22, 0.46, 0.02], center=true);
            // Top plate
            translate([0, 0, 0.47])
                cube([0.22, 0.46, 0.02], center=true);
            // Solid right panel (+X)
            translate([0.10, 0, 0.248])
                cube([0.02, 0.46, 0.44], center=true);
            // Motherboard tray / inner backplate separator
            translate([0.09, 0.0, 0.248])
                cube([0.008, 0.42, 0.42], center=true);
            // Back panel (-Y)
            translate([0, -0.22, 0.248])
                cube([0.22, 0.02, 0.44], center=true);
            // Front bezel frame (+Y)
            translate([0, 0.215, 0.405])
                cube([0.22, 0.03, 0.13], center=true);
            translate([0, 0.215, 0.10])
                cube([0.22, 0.03, 0.16], center=true);
            translate([0.082, 0.215, 0.25])
                cube([0.056, 0.03, 0.20], center=true);
            translate([-0.082, 0.215, 0.25])
                cube([0.056, 0.03, 0.20], center=true);
        }

        // Front circular fan shroud + horizontal mesh bars
        color([0.09, 0.10, 0.12], name="front_shroud", roughness=0.5, metalness=0.6, $asa=35.0) {
            translate([0, 0.195, 0.25])
            rotate([90, 0, 0])
                difference() {
                    cube([0.13, 0.13, 0.024], center=true);
                    cylinder(h=0.032, r=0.058, center=true, $fn=28);
                }
            for (gz = [-0.045, -0.015, 0.015, 0.045]) {
                translate([0, 0.224, 0.25 + gz])
                    cube([0.115, 0.005, 0.006], center=true);
            }
        }

        // Rear PCIe expansion slot vents & I/O shield detail
        color([0.24, 0.26, 0.30], name="pc_rear_brackets", roughness=0.35, metalness=0.88, $asa=10.0) {
            // I/O shield box
            translate([0.05, -0.205, 0.35])
                cube([0.04, 0.015, 0.13], center=true);
            // PCIe slot covers below GPU
            for (sz = [0.12, 0.145, 0.17, 0.195]) {
                translate([0.01, -0.206, sz])
                    cube([0.11, 0.012, 0.014], center=true);
            }
        }

        // Rubber cable-routing grommets on motherboard tray
        color([0.06, 0.06, 0.07], name="cable_grommets", roughness=0.90, metalness=0.0, $asa=20.0) {
            translate([0.083, 0.145, 0.31])
                cube([0.008, 0.032, 0.075], center=true);
            translate([0.083, 0.145, 0.18])
                cube([0.008, 0.032, 0.065], center=true);
        }

        // Top-front power button & subtle RGB accent strip
        color([0.2, 0.85, 1.0], name="power_led", roughness=0.25, emissive=[0.2, 0.85, 1.0], emissiveIntensity=2.0, $asa=0.0)
            translate([-0.06, 0.19, 0.482])
                cube([0.024, 0.012, 0.006], center=true);

        // Motherboard PCB
        color([0.07, 0.10, 0.11], name="motherboard", roughness=0.52, metalness=0.25, $asa=0.0)
            translate([0.078, -0.04, 0.295])
                cube([0.012, 0.30, 0.30], center=true);

        // Motherboard VRM Heatsinks (top & rear of CPU socket)
        color([0.32, 0.35, 0.40], name="vrm_heatsink", roughness=0.28, metalness=0.88, $asa=10.0) {
            // Rear VRM block
            translate([0.060, -0.155, 0.35])
                cube([0.024, 0.038, 0.11], center=true);
            // Top VRM block
            translate([0.060, -0.06, 0.422])
                cube([0.024, 0.11, 0.028], center=true);
            // Chipset heatsink (lower-front of motherboard)
            translate([0.064, 0.055, 0.185])
                cube([0.016, 0.068, 0.062], center=true);
        }

        // M.2 NVMe SSD Shield between CPU and GPU
        color([0.22, 0.24, 0.28], name="m2_heatsink", roughness=0.30, metalness=0.85, $asa=0.0)
            translate([0.068, -0.05, 0.265])
                cube([0.010, 0.095, 0.024], center=true);
        color([0.25, 0.85, 1.0], name="m2_accent", roughness=0.3, emissive=[0.2, 0.75, 1.0], emissiveIntensity=1.2, $asa=0.0)
            translate([0.062, -0.05, 0.265])
                cube([0.003, 0.055, 0.005], center=true);

        // Reinforced steel PCIe x16 Slot
        color([0.74, 0.76, 0.80], name="pcie_slot", roughness=0.25, metalness=0.92, $asa=0.0)
            translate([0.066, -0.065, 0.222])
                cube([0.014, 0.10, 0.012], center=true);

        // 24-pin ATX Motherboard Power Header
        color([0.14, 0.14, 0.16], name="atx_header", roughness=0.5, metalness=0.2, $asa=0.0)
            translate([0.064, 0.10, 0.325])
                cube([0.018, 0.016, 0.055], center=true);

        // CPU Cooler: layered aluminum fins + copper heatpipes
        color([0.74, 0.76, 0.80], name="cpu_heatsink", roughness=0.26, metalness=0.92, $asa=0.0) {
            // Contact coldplate block
            translate([0.062, -0.06, 0.34])
                cube([0.020, 0.058, 0.058], center=true);
            // Stacked cooling fin plates
            for (fx = [0.030, 0.036, 0.042, 0.048, 0.054]) {
                translate([fx, -0.06, 0.34])
                    cube([0.003, 0.098, 0.098], center=true);
            }
        }
        // Copper heatpipes peeking through CPU cooler
        color([0.78, 0.44, 0.24], name="copper_pipes", roughness=0.25, metalness=0.92, $asa=45.0) {
            for (py = [-0.088, -0.032], pz = [0.305, 0.375]) {
                translate([0.044, py, pz])
                rotate([0, 90, 0])
                    cylinder(h=0.038, r=0.004, center=true, $fn=12);
            }
        }

        // CPU Fan Shroud
        color([0.14, 0.14, 0.16], name="cpu_fan_frame", roughness=0.42, metalness=0.25, $asa=30.0)
            translate([0.018, -0.06, 0.34])
            rotate([0, 90, 0])
                difference() {
                    cube([0.102, 0.102, 0.018], center=true);
                    cylinder(h=0.028, r=0.046, center=true, $fn=28);
                }

        // Animated CPU Fan
        bone(name="CPUFan", t=[0.016, -0.06, 0.34], r=[0, 0, 0]) {
            fan_rotor_x(0.044, [0.15, 0.82, 1.0]);
        }

        // RAM Sticks (2 sleek RGB sticks with metallic heatspreaders)
        color([0.16, 0.17, 0.20], name="ram_body", roughness=0.35, metalness=0.65, $asa=0.0) {
            translate([0.056, 0.048, 0.34]) cube([0.032, 0.008, 0.12], center=true);
            translate([0.056, 0.068, 0.34]) cube([0.032, 0.008, 0.12], center=true);
        }
        color([0.95, 0.25, 0.85], name="ram_rgb", roughness=0.25, emissive=[0.95, 0.25, 0.85], emissiveIntensity=1.9, $asa=0.0) {
            translate([0.039, 0.048, 0.34]) cube([0.005, 0.006, 0.114], center=true);
            translate([0.039, 0.068, 0.34]) cube([0.005, 0.006, 0.114], center=true);
        }

        // GPU (Graphics Card) with brushed backplate, fin stack, shroud & fan rings
        // Top brushed metal backplate
        color([0.26, 0.28, 0.32], name="gpu_backplate", roughness=0.28, metalness=0.85, $asa=10.0)
            translate([0.012, -0.04, 0.241])
                cube([0.126, 0.262, 0.006], center=true);

        // Exposed silver heatsink fin core inside GPU
        color([0.70, 0.72, 0.76], name="gpu_fins", roughness=0.30, metalness=0.90, $asa=0.0)
            translate([0.014, -0.04, 0.228])
                cube([0.118, 0.246, 0.018], center=true);

        // Copper heatpipes visible along the open (-X) side of GPU
        color([0.78, 0.44, 0.24], name="copper_pipes", roughness=0.25, metalness=0.92, $asa=45.0) {
            translate([-0.046, -0.04, 0.226])
            rotate([90, 0, 0])
                cylinder(h=0.22, r=0.0035, center=true, $fn=12);
            translate([-0.046, -0.04, 0.234])
            rotate([90, 0, 0])
                cylinder(h=0.18, r=0.0035, center=true, $fn=12);
        }

        // GPU outer dark shroud + bottom fan cowls
        color([0.15, 0.16, 0.19], name="gpu_body", roughness=0.38, metalness=0.65, $asa=10.0) {
            // Side fascia facing -X
            translate([-0.044, -0.04, 0.214])
                cube([0.014, 0.262, 0.024], center=true);
            // Front & rear end caps
            translate([0.012, 0.088, 0.220])
                cube([0.124, 0.012, 0.036], center=true);
            translate([0.012, -0.168, 0.220])
                cube([0.124, 0.012, 0.036], center=true);
            // Bottom fan plate with circular cutouts
            translate([0.010, -0.04, 0.203])
                difference() {
                    cube([0.122, 0.260, 0.012], center=true);
                    translate([-0.005, -0.06, 0]) cylinder(h=0.02, r=0.043, center=true, $fn=24);
                    translate([-0.005, 0.06, 0]) cylinder(h=0.02, r=0.043, center=true, $fn=24);
                }
        }

        // GPU 8-pin PCIe power socket on top-front edge
        color([0.10, 0.10, 0.12], name="gpu_pcie_plug", roughness=0.5, metalness=0.2, $asa=0.0)
            translate([-0.032, 0.055, 0.246])
                cube([0.022, 0.024, 0.012], center=true);

        // GPU RGB Side Strip (facing -X!)
        color([0.20, 1.0, 0.55], name="gpu_rgb", roughness=0.25, emissive=[0.20, 1.0, 0.55], emissiveIntensity=2.0, $asa=0.0)
            translate([-0.052, -0.03, 0.216])
                cube([0.004, 0.15, 0.009], center=true);

        // GPU Dual Fans seated cleanly inside bottom cowls
        bone(name="GPUFan1", t=[0.005, -0.10, 0.202], r=[0, 0, 0]) {
            fan_rotor_z(0.040, [0.22, 0.24, 0.28]);
        }
        bone(name="GPUFan2", t=[0.005, 0.02, 0.202], r=[0, 0, 0]) {
            fan_rotor_z(0.040, [0.22, 0.24, 0.28]);
        }

        // PSU (Power Supply Unit) at bottom-back with clean badge & modular sockets
        color([0.10, 0.10, 0.12], name="psu_body", roughness=0.45, metalness=0.65, $asa=10.0)
            translate([0.0, -0.11, 0.08])
                cube([0.15, 0.18, 0.09], center=true);

        // PSU top vent grill ribs
        color([0.06, 0.06, 0.07], name="psu_grill", roughness=0.6, metalness=0.4, $asa=0.0)
        for (gy = [-0.15, -0.13, -0.11, -0.09, -0.07]) {
            translate([0.0, gy, 0.126])
                cube([0.11, 0.008, 0.004], center=true);
        }

        // PSU sleek dark badge with crisp amber/gold accent bar
        color([0.16, 0.17, 0.20], name="psu_badge", roughness=0.4, metalness=0.4, $asa=0.0)
            translate([-0.076, -0.11, 0.082])
                cube([0.003, 0.115, 0.052], center=true);
        color([0.92, 0.68, 0.14], name="psu_label", roughness=0.35, metalness=0.2, $asa=0.0)
            translate([-0.078, -0.11, 0.095])
                cube([0.002, 0.095, 0.012], center=true);

        // PSU status LED
        color([1.0, 0.35, 0.10], name="psu_led", roughness=0.3, emissive=[1.0, 0.35, 0.10], emissiveIntensity=2.0, $asa=0.0)
            translate([-0.076, -0.036, 0.052])
                cube([0.005, 0.014, 0.010], center=true);

        // Cleanly routed modular PSU cables (never floating or clipping GPU fans!)
        color([0.18, 0.20, 0.24], name="psu_cables_sleeve", roughness=0.75, metalness=0.0, $asa=35.0) {
            // 24-pin ATX bundle: exits PSU front (+Y), tucks behind tray, curves cleanly into ATX header
            cable_segment([0.045, -0.02, 0.08], [0.068, 0.12, 0.10], 0.0075);
            cable_segment([0.068, 0.12, 0.10], [0.072, 0.135, 0.29], 0.0075);
            cable_segment([0.072, 0.135, 0.29], [0.058, 0.10, 0.325], 0.0075);
        }
        // Braided red-accent PCIe 8-pin GPU cable: routes from PSU front around the front of GPU into top plug
        color([0.78, 0.18, 0.18], name="psu_cables", roughness=0.70, metalness=0.0, $asa=35.0) {
            cable_segment([-0.025, -0.02, 0.095], [-0.032, 0.125, 0.125], 0.006);
            cable_segment([-0.032, 0.125, 0.125], [-0.036, 0.130, 0.235], 0.006);
            cable_segment([-0.036, 0.130, 0.235], [-0.032, 0.062, 0.248], 0.006);
        }

        // Front intake fan
        bone(name="FrontFan", t=[0.0, 0.195, 0.25], r=[0, 0, 0]) {
            fan_rotor_y(0.054, [0.35, 0.65, 1.0]);
        }
    }
}
```

### pencil_sabotage/assets/models/gaming_chair.scad
```openscad
// Articulated ergonomic gaming chair (~0.64m wide, 1.18m tall)
// Faces +Y (Forward)
// Animated ChairSwivel (around Z) and ChairBackrest (reclines backward with +X)

chair_anims = [
    ["playing", [
        ["ChairSwivel", [
            [0.0, [0, 0, -2]],
            [0.4, [0, 0, 3]],
            [0.8, [0, 0, -2]]
        ]],
        ["ChairBackrest", [
            [0.0, [8, 0, 0]],
            [0.4, [15, 0, 0]],
            [0.8, [8, 0, 0]]
        ]]
    ]],
    ["startled", [
        ["ChairSwivel", [
            [0.00, [0, 0, -2]],
            [0.22, [0, 0, -72]],
            [0.80, [0, 0, -75]]
        ]],
        ["ChairBackrest", [
            [0.00, [8, 0, 0]],
            [0.22, [20, 0, 0]],
            [0.80, [22, 0, 0]]
        ]]
    ]],
    ["defeat", [
        ["ChairSwivel", [
            [0.00, [0, 0, 0]],
            [0.35, [0, 0, -32]],
            [1.00, [0, 0, -35]]
        ]],
        ["ChairBackrest", [
            [0.00, [8, 0, 0]],
            [0.35, [24, 0, 0]],
            [1.00, [28, 0, 0]]
        ]]
    ]]
];

armature(animations=chair_anims) {
    bone(name="ChairBase", t=[0, 0, 0], r=[0, 0, 0]) {
        // 5-star caster base
        color([0.14, 0.14, 0.16], name="chair_metal", roughness=0.35, metalness=0.85, $asa=30.0) {
            for (i = [0 : 4]) {
                rotate([0, 0, i * 72]) {
                    translate([0, 0.15, 0.055])
                        cube([0.04, 0.30, 0.03], center=true);
                    translate([0, 0.28, 0.025])
                    rotate([0, 90, 0])
                        cylinder(h=0.03, r=0.025, center=true, $fn=16);
                }
            }
            // Gas lift cylinder
            translate([0, 0, 0.24])
                cylinder(h=0.38, r=0.026, center=true, $fn=18);
        }

        // Swiveling upper seat assembly
        bone(name="ChairSwivel", t=[0, 0, 0.41], r=[0, 0, 0]) {
            // Seat cushion
            color([0.12, 0.12, 0.15], name="chair_leather_black", roughness=0.55, metalness=0.05, $asa=35.0)
                translate([0, 0.02, 0.04])
                    cube([0.50, 0.48, 0.085], center=true);

            // Armrests
            color([0.16, 0.16, 0.18], name="chair_armrest", roughness=0.60, metalness=0.2, $asa=20.0) {
                for (ax = [-0.29, 0.29]) {
                    translate([ax, 0.0, 0.14])
                        cube([0.032, 0.04, 0.17], center=true);
                    translate([ax, 0.03, 0.23])
                        cube([0.07, 0.26, 0.028], center=true);
                }
            }

            // Reclining backrest hinged at rear of seat
            bone(name="ChairBackrest", t=[0, -0.20, 0.05], r=[8, 0, 0]) {
                // Steel spine bracket
                color([0.14, 0.14, 0.16], name="chair_metal", roughness=0.35, metalness=0.85, $asa=20.0)
                    translate([0, -0.03, 0.06])
                        cube([0.12, 0.04, 0.18], center=true);

                // Sleek contoured backrest (keeps gamer shoulders & head clearly visible)
                color([0.12, 0.12, 0.15], name="chair_leather_black", roughness=0.55, metalness=0.05, $asa=35.0) {
                    translate([0, 0, 0.34])
                        cube([0.40, 0.075, 0.62], center=true);
                    translate([-0.20, 0.03, 0.31])
                    rotate([0, 0, -16])
                        cube([0.065, 0.08, 0.52], center=true);
                    translate([0.20, 0.03, 0.31])
                    rotate([0, 0, 16])
                        cube([0.065, 0.08, 0.52], center=true);
                }

                // Red racing stripes & lumbar cushion
                color([0.85, 0.16, 0.18], name="chair_accent_red", roughness=0.52, metalness=0.05, $asa=35.0) {
                    translate([-0.12, 0.035, 0.34])
                        cube([0.05, 0.015, 0.56], center=true);
                    translate([0.12, 0.035, 0.34])
                        cube([0.05, 0.015, 0.56], center=true);
                    translate([0, 0.045, 0.60])
                        cube([0.22, 0.055, 0.10], center=true);
                }
            }
        }
    }
}
```

### pencil_sabotage/assets/models/gamer_character.scad
```openscad
// Stylized articulated gamer sitting in the gaming chair facing +Y (Forward)
// Uses armature() with "playing", "startled", and "defeat" animations.
// All keyframe transitions stay <= 75 deg per bone for clean quaternion interpolation.

gamer_anims = [
    ["playing", [
        ["Pelvis", [
            [0.0, [0, 0, -2]],
            [0.4, [0, 0, 3]],
            [0.8, [0, 0, -2]]
        ]],
        ["Torso", [
            [0.0, [6, 0, -2]],
            [0.4, [13, 0, 2]],
            [0.8, [6, 0, -2]]
        ]],
        ["Head", [
            [0.0, [2, 0, -3]],
            [0.4, [-2, 0, 4]],
            [0.8, [2, 0, -3]]
        ]],
        ["LeftArm", [
            [0.0, [-6, 0, 0]],
            [0.2, [-10, 0, 2]],
            [0.4, [-12, 0, -2]],
            [0.6, [-9, 0, 3]],
            [0.8, [-6, 0, 0]]
        ]],
        ["LeftForearm", [
            [0.0, [0, 0, 0]],
            [0.2, [-10, 0, 4]],
            [0.4, [4, 0, -4]],
            [0.6, [-8, 0, 0]],
            [0.8, [0, 0, 0]]
        ]],
        ["RightArm", [
            [0.0, [-6, 0, 0]],
            [0.4, [-12, 0, -5]],
            [0.8, [-6, 0, 0]]
        ]],
        ["RightForearm", [
            [0.0, [0, 0, -4]],
            [0.4, [0, 0, 6]],
            [0.8, [0, 0, -4]]
        ]]
    ]],
    ["startled", [
        ["Pelvis", [
            [0.00, [0, 0, -2]],
            [0.22, [0, 0, -72]],
            [0.80, [0, 0, -75]]
        ]],
        ["Torso", [
            [0.00, [6, 0, -2]],
            [0.22, [16, 0, -42]],
            [0.80, [18, 0, -44]]
        ]],
        ["Head", [
            [0.00, [2, 0, -3]],
            [0.22, [4, 0, -46]],
            [0.80, [6, 0, -48]]
        ]],
        ["LeftArm", [
            [0.00, [-6, 0, 0]],
            [0.22, [-24, 0, 25]],
            [0.80, [-20, 0, 22]]
        ]],
        ["LeftForearm", [
            [0.00, [0, 0, 0]],
            [0.22, [35, 0, 0]],
            [0.80, [30, 0, 0]]
        ]],
        ["RightArm", [
            [0.00, [-6, 0, 0]],
            [0.22, [-32, 0, -35]],
            [0.80, [-26, 0, -30]]
        ]],
        ["RightForearm", [
            [0.00, [0, 0, 0]],
            [0.22, [45, 0, -15]],
            [0.80, [40, 0, -10]]
        ]]
    ]],
    ["defeat", [
        ["Pelvis", [
            [0.00, [0, 0, 0]],
            [0.35, [0, 0, -32]],
            [1.00, [0, 0, -35]]
        ]],
        ["Torso", [
            [0.00, [6, 0, 0]],
            [0.35, [22, 0, 0]],
            [1.00, [26, 0, 0]]
        ]],
        ["Head", [
            [0.00, [2, 0, 0]],
            [0.35, [18, 0, 0]],
            [1.00, [22, 0, 0]]
        ]],
        ["LeftArm", [
            [0.00, [0, 0, 0]],
            [0.35, [-65, 0, 20]],
            [1.00, [-40, 0, 12]]
        ]],
        ["LeftForearm", [
            [0.00, [0, 0, 0]],
            [0.35, [40, 0, 0]],
            [1.00, [55, 0, 0]]
        ]],
        ["RightArm", [
            [0.00, [0, 0, 0]],
            [0.35, [-65, 0, -20]],
            [1.00, [-40, 0, -12]]
        ]],
        ["RightForearm", [
            [0.00, [0, 0, 0]],
            [0.35, [40, 0, 0]],
            [1.00, [55, 0, 0]]
        ]]
    ]]
];

module limb_capsule(p1, r1, p2, r2) {
    hull() {
        translate(p1) sphere(r=r1, $fn=18);
        translate(p2) sphere(r=r2, $fn=18);
    }
}

armature(animations=gamer_anims) {
    // Pelvis seated at Z = 0.53 (top of gaming chair seat)
    bone(name="Pelvis", t=[0, 0.02, 0.53], r=[0, 0, 0]) {
        color([0.11, 0.13, 0.18], name="gamer_jeans", roughness=0.88, metalness=0.0, $asa=45.0)
            scale([1.25, 1.0, 0.85])
                sphere(r=0.135, $fn=24);

        // Seated smooth legs (thighs forward along +Y, shins down along -Z)
        for (side = [-1, 1]) {
            translate([side * 0.10, 0.02, -0.01]) {
                color([0.11, 0.13, 0.18], name="gamer_jeans", roughness=0.88, metalness=0.0, $asa=45.0) {
                    // Smooth thigh capsule from hip to knee
                    limb_capsule([0, 0, 0], 0.075, [side * 0.04, 0.37, -0.015], 0.062);
                    // Smooth shin capsule from knee to ankle
                    limb_capsule([side * 0.04, 0.37, -0.015], 0.060, [side * 0.04, 0.37, -0.42], 0.048);
                }
                // Sculpted sneaker with white sole and crimson/navy upper
                translate([side * 0.04, 0.37, -0.45]) {
                    // White rubber sole
                    color([0.86, 0.87, 0.90], name="gamer_shoe_sole", roughness=0.60, metalness=0.0, $asa=35.0)
                    hull() {
                        translate([0, 0.0, -0.025]) cylinder(h=0.024, r=0.046, center=true, $fn=18);
                        translate([0, 0.11, -0.025]) cylinder(h=0.024, r=0.042, center=true, $fn=18);
                    }
                    // Sneaker upper
                    color([0.22, 0.24, 0.30], name="gamer_shoes", roughness=0.72, metalness=0.0, $asa=40.0)
                    hull() {
                        translate([0, 0.0, 0.005]) sphere(r=0.048, $fn=18);
                        translate([0, 0.05, 0.002]) sphere(r=0.045, $fn=18);
                        translate([0, 0.115, -0.008]) scale([1.0, 1.1, 0.65]) sphere(r=0.040, $fn=18);
                    }
                }
            }
        }

        // Torso
        bone(name="Torso", t=[0, -0.02, 0.06], r=[0, 0, 0]) {
            // Broad-shouldered sculpted hoodie torso
            color([0.20, 0.44, 0.72], name="gamer_hoodie", roughness=0.84, metalness=0.0, $asa=45.0) {
                hull() {
                    // Waist
                    translate([0, 0, 0.08]) scale([1.15, 0.85, 0.8]) sphere(r=0.135, $fn=24);
                    // Chest center
                    translate([0, 0.01, 0.27]) scale([1.22, 0.88, 0.95]) sphere(r=0.145, $fn=24);
                    // Left & Right shoulders blended seamlessly
                    translate([-0.152, 0.005, 0.35]) sphere(r=0.068, $fn=18);
                    translate([0.152, 0.005, 0.35]) sphere(r=0.068, $fn=18);
                }
                // Soft draped hood fold around back of neck (no stiff cylinder)
                hull() {
                    translate([0, -0.065, 0.39]) scale([1.1, 0.85, 0.65]) sphere(r=0.095, $fn=20);
                    translate([-0.07, 0.02, 0.38]) sphere(r=0.042, $fn=14);
                    translate([0.07, 0.02, 0.38]) sphere(r=0.042, $fn=14);
                }
            }

            // Flat contoured hoodie kangaroo pocket & flush drawstrings
            color([0.15, 0.35, 0.60], name="gamer_hoodie_pocket", roughness=0.88, metalness=0.0, $asa=30.0)
                hull() {
                    translate([-0.085, 0.105, 0.095]) sphere(r=0.014, $fn=12);
                    translate([0.085, 0.105, 0.095]) sphere(r=0.014, $fn=12);
                    translate([-0.060, 0.115, 0.165]) sphere(r=0.014, $fn=12);
                    translate([0.060, 0.115, 0.165]) sphere(r=0.014, $fn=12);
                }

            color([0.85, 0.86, 0.88], name="hoodie_strings", roughness=0.75, metalness=0.0, $asa=35.0) {
                limb_capsule([-0.026, 0.055, 0.382], 0.0042, [-0.032, 0.105, 0.295], 0.0042);
                limb_capsule([0.026, 0.055, 0.382], 0.0042, [0.032, 0.105, 0.295], 0.0042);
            }

            // Head & Gaming Headset
            bone(name="Head", t=[0, 0.01, 0.46], r=[0, 0, 0]) {
                // Warm natural skin tone
                color([0.78, 0.56, 0.42], name="gamer_skin", roughness=0.68, metalness=0.0, $asa=45.0) {
                    // Neck seated deeply into hoodie collar (no gap when tilting)
                    limb_capsule([0, -0.008, -0.065], 0.048, [0, -0.002, 0.065], 0.046);
                    // Head
                    translate([0, 0.01, 0.135])
                    scale([0.92, 0.98, 1.08])
                        sphere(r=0.112, $fn=28);
                    // Nose pointing +Y
                    translate([0, 0.118, 0.125])
                        sphere(r=0.018, $fn=14);
                }

                // Stylized sculpted hair (crown + back + side swept bangs)
                color([0.22, 0.14, 0.09], name="gamer_hair", roughness=0.85, metalness=0.0, $asa=45.0) {
                    translate([0, -0.012, 0.175])
                    scale([0.96, 1.00, 0.90])
                        sphere(r=0.114, $fn=24);
                    // Back hair taper
                    translate([0, -0.045, 0.125])
                    scale([0.92, 0.75, 0.95])
                        sphere(r=0.102, $fn=20);
                    // Front bangs sweep
                    translate([0.015, 0.045, 0.215])
                    rotate([18, 8, -10])
                    scale([1.05, 0.85, 0.55])
                        sphere(r=0.092, $fn=20);
                }

                // Expressive eyes (sclera + pupils + eyebrows)
                color([0.95, 0.95, 0.96], name="gamer_eye_white", roughness=0.25, metalness=0.0, $asa=35.0) {
                    translate([-0.038, 0.098, 0.148]) scale([1.0, 0.55, 1.15]) sphere(r=0.017, $fn=14);
                    translate([0.038, 0.098, 0.148]) scale([1.0, 0.55, 1.15]) sphere(r=0.017, $fn=14);
                }
                color([0.07, 0.08, 0.10], name="gamer_eyes", roughness=0.2, metalness=0.0, $asa=35.0) {
                    translate([-0.038, 0.106, 0.148]) sphere(r=0.010, $fn=12);
                    translate([0.038, 0.106, 0.148]) sphere(r=0.010, $fn=12);
                }
                color([0.18, 0.11, 0.07], name="gamer_eyebrows", roughness=0.85, metalness=0.0, $asa=15.0) {
                    translate([-0.040, 0.102, 0.173]) rotate([0, -8, 0]) cube([0.030, 0.012, 0.008], center=true);
                    translate([0.040, 0.102, 0.173]) rotate([0, 8, 0]) cube([0.030, 0.012, 0.008], center=true);
                }
                // Subtle mouth
                color([0.55, 0.32, 0.28], name="gamer_mouth", roughness=0.65, metalness=0.0, $asa=35.0)
                    translate([0, 0.104, 0.088])
                    scale([1.4, 0.6, 0.5])
                        sphere(r=0.014, $fn=12);

                // Gaming Headset + Boom Mic
                color([0.13, 0.14, 0.17], name="headset_body", roughness=0.38, metalness=0.45, $asa=45.0) {
                    // Ear cups on -X and +X
                    translate([-0.110, 0.005, 0.135])
                    rotate([0, 90, 0])
                        cylinder(h=0.036, r=0.046, center=true, $fn=20);
                    translate([0.110, 0.005, 0.135])
                    rotate([0, 90, 0])
                        cylinder(h=0.036, r=0.046, center=true, $fn=20);
                    // Headband arch clearing the hair cleanly
                    translate([0, 0.002, 0.162])
                    rotate([90, 0, 0])
                        difference() {
                            cylinder(h=0.026, r=0.132, center=true, $fn=28);
                            cylinder(h=0.034, r=0.116, center=true, $fn=28);
                            translate([0, -0.085, 0]) cube([0.32, 0.19, 0.04], center=true);
                        }
                    // Boom microphone arm from left earcup
                    limb_capsule([-0.112, 0.022, 0.115], 0.0048, [-0.058, 0.112, 0.088], 0.0048);
                }
                // Foam mic windscreen
                color([0.08, 0.08, 0.09], name="headset_mic_foam", roughness=0.95, metalness=0.0, $asa=35.0)
                    translate([-0.052, 0.118, 0.088])
                    scale([0.9, 1.3, 0.9])
                        sphere(r=0.011, $fn=12);

                // Neon headset ring glow
                color([0.2, 0.9, 1.0], name="headset_rgb", roughness=0.2, emissive=[0.2, 0.9, 1.0], emissiveIntensity=1.6, $asa=35.0) {
                    translate([-0.128, 0.005, 0.135])
                    rotate([0, 90, 0])
                        cylinder(h=0.006, r=0.030, center=true, $fn=18);
                    translate([0.128, 0.005, 0.135])
                    rotate([0, 90, 0])
                        cylinder(h=0.006, r=0.030, center=true, $fn=18);
                }
            }

            // Left Arm (Anatomical Left = -X), smooth capsules reaching toward keyboard
            bone(name="LeftArm", t=[-0.178, 0.008, 0.35], r=[0, 0, 0]) {
                color([0.20, 0.44, 0.72], name="gamer_hoodie", roughness=0.84, metalness=0.0, $asa=45.0)
                    limb_capsule([0, 0, 0], 0.058, [-0.035, 0.12, -0.185], 0.045);

                bone(name="LeftForearm", t=[-0.035, 0.12, -0.185], r=[0, 0, 0]) {
                    color([0.20, 0.44, 0.72], name="gamer_hoodie", roughness=0.84, metalness=0.0, $asa=45.0)
                        limb_capsule([0, 0, 0], 0.044, [0.03, 0.195, 0.026], 0.034);
                    // Hand on keyboard
                    color([0.78, 0.56, 0.42], name="gamer_skin", roughness=0.68, metalness=0.0, $asa=45.0)
                    translate([0.035, 0.225, 0.03])
                    scale([1.05, 1.3, 0.6])
                        sphere(r=0.034, $fn=16);
                }
            }

            // Right Arm (Anatomical Right = +X), smooth capsules reaching toward mouse
            bone(name="RightArm", t=[0.178, 0.008, 0.35], r=[0, 0, 0]) {
                color([0.20, 0.44, 0.72], name="gamer_hoodie", roughness=0.84, metalness=0.0, $asa=45.0)
                    limb_capsule([0, 0, 0], 0.058, [0.035, 0.12, -0.185], 0.045);

                bone(name="RightForearm", t=[0.035, 0.12, -0.185], r=[0, 0, 0]) {
                    color([0.20, 0.44, 0.72], name="gamer_hoodie", roughness=0.84, metalness=0.0, $asa=45.0)
                        limb_capsule([0, 0, 0], 0.044, [-0.035, 0.195, 0.026], 0.034);
                    // Hand on mouse
                    color([0.78, 0.56, 0.42], name="gamer_skin", roughness=0.68, metalness=0.0, $asa=45.0)
                    translate([-0.04, 0.225, 0.03])
                    scale([1.05, 1.3, 0.6])
                        sphere(r=0.034, $fn=16);
                }
            }
        }
    }
}
```

### pencil_sabotage/assets/models/room_environment.scad
```openscad
// Cozy room floor rug, baseboards, wall shelf, and potted plant
// Floor is at Z=0, room spans X=[-3.2..3.2], Y=[-3.2..3.2]

module shelf_book(w, d, h, cover_col, cover_name) {
    cov_t = 0.004;
    // Hardcover boards (left, right, and spine facing -Y toward room)
    color(cover_col, name=cover_name, roughness=0.68, metalness=0.0, $asa=15.0) {
        translate([-w * 0.5 + cov_t * 0.5, 0, h * 0.5])
            cube([cov_t, d, h], center=true);
        translate([w * 0.5 - cov_t * 0.5, 0, h * 0.5])
            cube([cov_t, d, h], center=true);
        translate([0, -d * 0.5 + cov_t * 0.5, h * 0.5])
            cube([w, cov_t, h], center=true);
    }
    // Recessed cream page block visible on top (+Z) and back (+Y)
    color([0.92, 0.89, 0.80], name="book_pages", roughness=0.88, metalness=0.0, $asa=15.0)
        translate([0, 0.002, h * 0.5])
            cube([w - cov_t * 2.0, d - cov_t * 2.2, h - 0.010], center=true);
    // Decorative spine bands on -Y spine
    color([0.90, 0.82, 0.52], name="book_spine_trim", roughness=0.38, metalness=0.55, $asa=15.0) {
        translate([0, -d * 0.5 - 0.0005, h * 0.82])
            cube([w * 0.72, 0.002, 0.008], center=true);
        translate([0, -d * 0.5 - 0.0005, h * 0.76])
            cube([w * 0.72, 0.002, 0.004], center=true);
        translate([0, -d * 0.5 - 0.0005, h * 0.16])
            cube([w * 0.72, 0.002, 0.006], center=true);
    }
}

module room_environment_model() {
    // Hardwood floor planks base
    color([0.55, 0.39, 0.25], name="room_floor", roughness=0.68, metalness=0.0, $asa=0.0)
        translate([0, 0, -0.02])
            cube([6.6, 6.8, 0.04], center=true);

    // Cozy living room area rug between sofa and desk (textured at runtime via rug_pattern.gdshader)
    color([0.26, 0.36, 0.46], name="room_rug_border", roughness=0.96, metalness=0.0, $asa=0.0)
        translate([0, -0.1, 0.005])
            cube([3.34, 3.74, 0.010], center=true);

    color([0.66, 0.61, 0.54], name="room_rug", roughness=0.95, metalness=0.0, $asa=0.0)
        translate([0, -0.1, 0.006])
            cube([3.18, 3.58, 0.012], center=true);

    // Fine woven rug fringes tucked flush against the front (+Y) and back (-Y) borders
    color([0.67, 0.62, 0.55], name="rug_fringe", roughness=0.98, metalness=0.0, $asa=0.0) {
        for (fx = [-1.62 : 0.05 : 1.62]) {
            translate([fx, 1.795, 0.0035]) cube([0.016, 0.065, 0.005], center=true);
            translate([fx, -1.995, 0.0035]) cube([0.016, 0.065, 0.005], center=true);
        }
    }

    // Walls (Back wall behind desk at +Y, side walls at -X and +X, wall behind sofa at -Y)
    color([0.58, 0.55, 0.50], name="room_walls", roughness=0.92, metalness=0.0, $asa=0.0) {
        // Front/desk wall (+Y)
        translate([0, 3.1, 1.6])
            cube([6.6, 0.15, 3.2], center=true);
        // Back wall behind sofa (-Y)
        translate([0, -3.2, 1.6])
            cube([6.6, 0.15, 3.2], center=true);
        // Left wall (-X)
        translate([-3.2, 0, 1.6])
            cube([0.15, 6.6, 3.2], center=true);
        // Right wall (+X)
        translate([3.2, 0, 1.6])
            cube([0.15, 6.6, 3.2], center=true);
    }

    // Crisp matte white ceiling
    color([0.90, 0.90, 0.92], name="room_ceiling", roughness=0.92, metalness=0.0, $asa=0.0)
        translate([0, 0, 3.25])
            cube([6.6, 6.6, 0.1], center=true);

    // White baseboards
    color([0.92, 0.91, 0.89], name="room_trim", roughness=0.55, metalness=0.0, $asa=0.0) {
        translate([0, 3.01, 0.06]) cube([6.4, 0.03, 0.12], center=true);
        translate([-3.11, 0, 0.06]) cube([0.03, 6.4, 0.12], center=true);
        translate([3.11, 0, 0.06]) cube([0.03, 6.4, 0.12], center=true);
    }

    // Poster frame positioned cleanly to the left of the desk & monitor
    color([0.16, 0.16, 0.18], name="poster_frame", roughness=0.45, metalness=0.2, $asa=0.0)
        translate([-1.35, 3.01, 1.78])
            cube([0.74, 0.03, 0.96], center=true);
    color([0.88, 0.38, 0.24], name="poster_art", roughness=0.75, metalness=0.0, $asa=0.0)
        translate([-1.35, 2.99, 1.78])
            cube([0.66, 0.02, 0.88], center=true);

    // Wall shelf flush against +Y wall (Y=3.025) with matte black metal brackets
    color([0.45, 0.29, 0.17], name="wall_shelf", roughness=0.48, metalness=0.0, $asa=15.0)
        translate([0.65, 2.885, 1.65])
            cube([0.95, 0.28, 0.035], center=true);

    color([0.13, 0.13, 0.15], name="shelf_brackets", roughness=0.42, metalness=0.82, $asa=20.0) {
        for (bx = [0.30, 1.00]) {
            // Vertical wall plate & horizontal shelf support
            translate([bx, 3.017, 1.572]) cube([0.024, 0.016, 0.12], center=true);
            translate([bx, 2.905, 1.625]) cube([0.024, 0.23, 0.014], center=true);
            // Diagonal support strut
            translate([bx, 2.935, 1.575])
            rotate([38, 0, 0])
                cube([0.016, 0.15, 0.012], center=true);
        }
    }

    // Hardcover books seated flush on shelf top (Z = 1.6675)
    translate([0.345, 2.885, 1.6675]) shelf_book(0.046, 0.185, 0.245, [0.22, 0.45, 0.72], "book_blue");
    translate([0.396, 2.885, 1.6675]) shelf_book(0.042, 0.175, 0.215, [0.80, 0.27, 0.24], "book_red");
    translate([0.448, 2.885, 1.6675]) shelf_book(0.050, 0.190, 0.260, [0.24, 0.60, 0.38], "book_green");
    translate([0.538, 2.885, 1.6720])
    rotate([0, -13, 0])
        shelf_book(0.040, 0.170, 0.205, [0.86, 0.64, 0.22], "book_amber");

    // Detailed trophy cup on walnut plinth at X = 0.75
    translate([0.75, 2.885, 1.6675]) {
        color([0.24, 0.14, 0.08], name="trophy_base", roughness=0.52, metalness=0.0, $asa=15.0) {
            translate([0, 0, 0.013]) cube([0.076, 0.076, 0.026], center=true);
            translate([0, 0, 0.031]) cube([0.060, 0.060, 0.010], center=true);
        }
        color([0.92, 0.76, 0.28], name="trophy_plaque", roughness=0.28, metalness=0.85, $asa=15.0)
            translate([0, -0.039, 0.013]) cube([0.046, 0.003, 0.014], center=true);

        color([0.94, 0.74, 0.22], name="shelf_trophy", roughness=0.22, metalness=0.92, $asa=35.0) {
            // Pedestal stem & decorative collar
            translate([0, 0, 0.036]) cylinder(h=0.012, r1=0.022, r2=0.011, $fn=24);
            translate([0, 0, 0.048]) cylinder(h=0.026, r1=0.008, r2=0.012, $fn=20);
            translate([0, 0, 0.060]) sphere(r=0.014, $fn=20);
            // Hollowed goblet chalice
            translate([0, 0, 0.074])
                difference() {
                    cylinder(h=0.064, r1=0.015, r2=0.038, $fn=28);
                    translate([0, 0, 0.010])
                        cylinder(h=0.058, r1=0.012, r2=0.033, $fn=28);
                }
            // Side trophy handles (-X and +X)
            for (sx = [-1, 1]) {
                translate([sx * 0.034, 0, 0.112])
                rotate([90, 0, 0])
                    difference() {
                        scale([0.75, 1.15, 1.0]) cylinder(h=0.007, r=0.020, center=true, $fn=20);
                        scale([0.75, 1.15, 1.0]) cylinder(h=0.012, r=0.013, center=true, $fn=20);
                    }
            }
        }
    }

    // Hollowed ceramic planter with potting soil & layered succulent rosette at X = 0.97
    translate([0.97, 2.885, 1.6675]) {
        color([0.88, 0.86, 0.82], name="mini_pot", roughness=0.45, metalness=0.0, $asa=35.0)
            difference() {
                cylinder(h=0.076, r1=0.034, r2=0.046, $fn=28);
                translate([0, 0, 0.012])
                    cylinder(h=0.068, r1=0.028, r2=0.040, $fn=28);
            }
        color([0.66, 0.36, 0.24], name="mini_pot_band", roughness=0.58, metalness=0.0, $asa=35.0)
            translate([0, 0, 0.056])
                difference() {
                    cylinder(h=0.012, r1=0.0440, r2=0.0459, $fn=28);
                    translate([0, 0, -0.002])
                        cylinder(h=0.016, r=0.039, $fn=28);
                }
        color([0.20, 0.14, 0.10], name="mini_soil", roughness=0.95, metalness=0.0, $asa=35.0)
            translate([0, 0, 0.058])
                cylinder(h=0.010, r=0.038, $fn=24);
        color([0.24, 0.62, 0.36], name="mini_plant", roughness=0.68, metalness=0.0, $asa=45.0) {
            // Outer rosette leaves curving upward and outward above the rim
            for (i = [0 : 6]) {
                rotate([0, 0, i * (360 / 7)])
                translate([0, 0, 0.071])
                rotate([0, -25, 0])
                translate([0.018, 0, 0])
                scale([1.30, 0.56, 0.24])
                    sphere(r=0.019, $fn=14);
            }
            // Middle rosette ring
            for (i = [0 : 5]) {
                rotate([0, 0, i * 60 + 30])
                translate([0, 0, 0.075])
                rotate([0, -48, 0])
                translate([0.015, 0, 0])
                scale([1.22, 0.54, 0.24])
                    sphere(r=0.016, $fn=14);
            }
            // Inner upright leaves
            for (i = [0 : 3]) {
                rotate([0, 0, i * 90 + 15])
                translate([0, 0, 0.080])
                rotate([0, -70, 0])
                translate([0.011, 0, 0])
                scale([1.15, 0.52, 0.24])
                    sphere(r=0.013, $fn=14);
            }
            // Center bud
            translate([0, 0, 0.088]) scale([0.72, 0.72, 1.25]) sphere(r=0.009, $fn=14);
        }
    }

    // Potted corner plant on right side of desk
    translate([1.45, 2.65, 0]) {
        color([0.74, 0.42, 0.26], name="plant_pot", roughness=0.80, metalness=0.0, $asa=45.0)
            cylinder(h=0.34, r1=0.14, r2=0.19, $fn=24);
        color([0.22, 0.56, 0.26], name="plant_leaves", roughness=0.70, metalness=0.0, $asa=45.0) {
            for (i = [0 : 5]) {
                rotate([0, 0, i * 60])
                translate([0.08, 0, 0.45])
                rotate([25, 0, 0])
                scale([0.6, 1.5, 0.18])
                    sphere(r=0.16, $fn=16);
            }
            translate([0, 0, 0.55])
            scale([0.7, 0.7, 1.2])
                sphere(r=0.12, $fn=16);
        }
    }

    // Modern pendant ceiling lamp aligned with CeilingLight at Godot (0, 2.65, -0.35)
    translate([0, 0.35, 3.20]) {
        // Matte charcoal canopy & suspension cord
        color([0.14, 0.14, 0.16], name="lamp_hardware", roughness=0.45, metalness=0.6, $asa=35.0) {
            translate([0, 0, -0.025])
                cylinder(h=0.026, r1=0.075, r2=0.085, $fn=28);
            translate([0, 0, -0.355])
                cylinder(h=0.315, r=0.007, $fn=16);
        }

        // Brushed brass collar & socket cap
        color([0.82, 0.64, 0.28], name="lamp_brass", roughness=0.28, metalness=0.88, $asa=35.0) {
            translate([0, 0, -0.045])
                cylinder(h=0.022, r=0.018, $fn=20);
            translate([0, 0, -0.395])
                cylinder(h=0.045, r1=0.036, r2=0.022, $fn=24);
        }

        // Outer tapered dome lampshade
        color([0.16, 0.17, 0.20], name="lamp_shade_outer", roughness=0.52, metalness=0.35, $asa=30.0)
        translate([0, 0, -0.54])
            difference() {
                cylinder(h=0.17, r1=0.26, r2=0.09, $fn=36);
                translate([0, 0, -0.004])
                    cylinder(h=0.166, r1=0.252, r2=0.082, $fn=36);
            }

        // Warm illuminated inner shade lining
        color([0.96, 0.88, 0.72], name="lamp_shade_inner", roughness=0.65, metalness=0.0, emissive=[0.90, 0.75, 0.50], emissiveIntensity=0.55, $asa=30.0)
        translate([0, 0, -0.538])
            difference() {
                cylinder(h=0.162, r1=0.250, r2=0.080, $fn=36);
                translate([0, 0, -0.004])
                    cylinder(h=0.158, r1=0.244, r2=0.074, $fn=36);
            }

        // Warm frosted globe bulb
        color([1.0, 0.95, 0.84], name="lamp_bulb", roughness=0.25, metalness=0.0, emissive=[1.0, 0.88, 0.64], emissiveIntensity=1.8, $asa=45.0)
            translate([0, 0, -0.49])
                sphere(r=0.055, $fn=24);
    }
}

room_environment_model();
```

### pencil_sabotage/shaders/monitor_screen.gdshader
```glsl
shader_type spatial;
render_mode unshaded;

uniform float damage_level : hint_range(0.0, 1.0) = 0.0;
uniform float is_bsod : hint_range(0.0, 1.0) = 0.0;
uniform float power_flicker : hint_range(0.0, 1.0) = 1.0;
uniform float hit_flash : hint_range(0.0, 1.0) = 0.0;

varying vec2 screen_uv;

void vertex() {
	vec3 world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
	// Monitor sits at (-0.14, 0.76, -2.28) rotated 180 deg Y; screen center in OpenSCAD is (0, 0.008, 0.32)
	// Screen dimensions: 0.62m wide x 0.36m tall -> world center (-0.14, 1.08, -2.272)
	screen_uv = clamp(vec2(
		(world_pos.x + 0.14) / 0.62 + 0.5,
		(world_pos.y - 1.08) / 0.36 + 0.5
	), 0.0, 1.0);
}

float hash(vec2 p) {
	return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453123);
}

float box_mask(vec2 p, vec2 center, vec2 half_size) {
	vec2 d = abs(p - center) - half_size;
	return 1.0 - smoothstep(0.0, 0.008, max(d.x, d.y));
}

float seg_dist(vec2 p, vec2 a, vec2 b) {
	vec2 pa = p - a;
	vec2 ba = b - a;
	float h = clamp(dot(pa, ba) / max(dot(ba, ba), 0.0001), 0.0, 1.0);
	return length(pa - ba * h);
}

float capsule_mask(vec2 p, vec2 a, vec2 b, float r) {
	return 1.0 - smoothstep(r - 0.006, r + 0.005, seg_dist(p, a, b));
}

vec3 draw_fighter(vec2 ap, vec2 base, float facing, float lunge, float punch, float kick, float jump_pose, float recoil, vec3 outfit_col, vec3 trim_col, vec3 skin_col, float t) {
	vec3 res = vec3(-1.0); // negative signals transparent
	vec2 hip = base + vec2(-recoil * 0.06 * facing, 0.125);
	vec2 back_foot = base + vec2(-0.065 * facing, 0.015 + jump_pose * 0.04);
	vec2 front_foot = base + vec2((0.065 + kick * 0.13) * facing, 0.015 + kick * 0.15 + jump_pose * 0.02);
	vec2 shoulder = hip + vec2((lunge * 0.045 - recoil * 0.04) * facing, 0.125);
	vec2 head = shoulder + vec2((0.01 - recoil * 0.03) * facing, 0.065);
	vec2 punch_hand = shoulder + vec2((0.08 + punch * 0.13) * facing, -0.02 + punch * 0.035);
	vec2 guard_hand = shoulder + vec2(0.045 * facing, 0.025);

	float alpha = 0.0;
	vec3 fcol = outfit_col;

	// Flowing headband / scarf tail
	vec2 hb_end = head + vec2(-0.085 * facing, 0.01 + 0.018 * sin(t * 10.0 + base.x));
	float hb = capsule_mask(ap, head + vec2(-0.015 * facing, 0.015), hb_end, 0.012);
	if (hb > 0.05) {
		fcol = mix(fcol, trim_col, hb);
		alpha = max(alpha, hb);
	}

	// Legs & torso
	float leg_b = capsule_mask(ap, hip, back_foot, 0.024);
	float leg_f = capsule_mask(ap, hip, front_foot, 0.025);
	float torso = capsule_mask(ap, hip, shoulder, 0.038);
	float body_m = max(torso, max(leg_b, leg_f));
	if (body_m > 0.05) {
		fcol = mix(fcol, outfit_col, body_m);
		alpha = max(alpha, body_m);
	}

	// Waist belt / sash trim
	float belt = capsule_mask(ap, hip + vec2(-0.03, 0.01), hip + vec2(0.03, 0.01), 0.014);
	if (belt > 0.05) {
		fcol = mix(fcol, trim_col, belt);
		alpha = max(alpha, belt);
	}

	// Arms & fists
	float arm_g = capsule_mask(ap, shoulder, guard_hand, 0.018);
	float arm_p = capsule_mask(ap, shoulder, punch_hand, 0.020);
	float arms = max(arm_g, arm_p);
	if (arms > 0.05) {
		fcol = mix(fcol, skin_col, arms);
		alpha = max(alpha, arms);
	}
	float glove = 1.0 - smoothstep(0.018, 0.026, length(ap - punch_hand));
	if (glove > 0.05) {
		fcol = mix(fcol, trim_col, glove);
		alpha = max(alpha, glove);
	}

	// Head + headband
	float head_m = 1.0 - smoothstep(0.032, 0.040, length(ap - head));
	if (head_m > 0.05) {
		fcol = mix(fcol, skin_col, head_m);
		alpha = max(alpha, head_m);
	}
	float band = capsule_mask(ap, head + vec2(-0.028, 0.014), head + vec2(0.028, 0.014), 0.009);
	if (band > 0.05) {
		fcol = mix(fcol, trim_col, band);
		alpha = max(alpha, band);
	}

	if (alpha > 0.05) {
		res = fcol;
	}
	return res;
}

vec3 game_scene(vec2 uv, float t, float dmg) {
	float aspect = 1.722;
	vec2 ap = vec2(uv.x * aspect, uv.y);
	float sim_t = (dmg > 0.55) ? floor(t * 7.0) / 7.0 : t;

	// 1. Golden sunset dojo / rooftop sky
	vec3 sky_top = vec3(0.10, 0.14, 0.36);
	vec3 sky_mid = vec3(0.55, 0.24, 0.42);
	vec3 sky_low = vec3(0.95, 0.52, 0.20);
	vec3 col = mix(sky_low, sky_mid, smoothstep(0.24, 0.58, uv.y));
	col = mix(col, sky_top, smoothstep(0.56, 0.96, uv.y));

	// Warm sunset sun in upper-center
	vec2 sun_p = ap - vec2(0.86, 0.46);
	float sun_d = length(sun_p);
	if (sun_d < 0.22) {
		float sun_halo = (1.0 - smoothstep(0.11, 0.22, sun_d)) * 0.40;
		col += vec3(1.0, 0.72, 0.30) * sun_halo;
	}
	if (sun_d < 0.12) {
		float sun_disc = 1.0 - smoothstep(0.112, 0.120, sun_d);
		col = mix(col, vec3(1.0, 0.90, 0.48), sun_disc);
	}

	// Distant temple / city skyline silhouette + glowing lanterns
	if (uv.y >= 0.24 && uv.y < 0.52) {
		float roof1 = step(abs(uv.x - 0.22), 0.12) * step(uv.y, 0.42 - abs(uv.x - 0.22) * 0.65);
		float roof2 = step(abs(uv.x - 0.78), 0.13) * step(uv.y, 0.44 - abs(uv.x - 0.78) * 0.60);
		float wall_bg = step(uv.y, 0.31);
		float sil = max(wall_bg, max(roof1, roof2));
		col = mix(col, vec3(0.13, 0.08, 0.18), sil * 0.92);

		// Warm paper lanterns swaying gently
		for (int l = 0; l < 3; l++) {
			vec2 lp = vec2(0.26 + float(l) * 0.60, 0.36 + 0.008 * sin(sim_t * 2.2 + float(l)));
			float ld = length(ap - lp);
			if (ld < 0.045) {
				float glow = (1.0 - smoothstep(0.012, 0.045, ld)) * 0.65;
				float bulb = 1.0 - smoothstep(0.012, 0.019, ld);
				col += vec3(1.0, 0.55, 0.18) * glow;
				col = mix(col, vec3(1.0, 0.88, 0.45), bulb);
			}
		}
	}

	// 2. Arena dojo wooden deck (uv.y < 0.25)
	if (uv.y < 0.25) {
		float floor_t = clamp((0.25 - uv.y) / 0.25, 0.0, 1.0);
		vec3 deck_col = mix(vec3(0.32, 0.19, 0.13), vec3(0.18, 0.10, 0.08), floor_t);
		// Perspective floor plank lines
		float p_x = (uv.x - 0.5) / max(0.08 + floor_t * 0.92, 0.05);
		float plank = 1.0 - smoothstep(0.015, 0.045, abs(fract(p_x * 6.0) - 0.5));
		deck_col = mix(deck_col, vec3(0.11, 0.06, 0.05), plank * 0.55);
		// Golden arena floor trim line
		float trim_line = 1.0 - smoothstep(0.003, 0.010, abs(uv.y - 0.242));
		col = mix(deck_col, vec3(0.95, 0.68, 0.28), trim_line * 0.85);
	}

	// 3. Choreographed fighting cycle (dash -> punch -> fireball -> impact!)
	float cyc = fract(sim_t * 0.38);
	float step_bounce = abs(sin(sim_t * 6.5)) * 0.012;

	// Melee exchange (0.10..0.34), fireball launch (0.42..0.72), jump kick (0.76..0.94)
	float melee_p = smoothstep(0.10, 0.18, cyc) * (1.0 - smoothstep(0.28, 0.36, cyc));
	float hadou_pose = smoothstep(0.38, 0.45, cyc) * (1.0 - smoothstep(0.58, 0.68, cyc));
	float jump_arc = sin(clamp((cyc - 0.76) / 0.18, 0.0, 1.0) * 3.14159) * 0.15;
	float kick_pose = smoothstep(0.77, 0.82, cyc) * (1.0 - smoothstep(0.90, 0.94, cyc));
	float hit_melee = smoothstep(0.18, 0.22, cyc) * (1.0 - smoothstep(0.28, 0.34, cyc));
	float hit_fire = smoothstep(0.68, 0.72, cyc) * (1.0 - smoothstep(0.80, 0.86, cyc));
	float hit_any = max(hit_melee, hit_fire);

	float f1_x = 0.58 + melee_p * 0.14 + kick_pose * 0.12 - hadou_pose * 0.05;
	float f1_y = 0.23 + step_bounce + jump_arc;
	vec2 f1_base = vec2(f1_x, f1_y);

	float f2_x = 1.14 - melee_p * 0.12 + hit_any * 0.07;
	float f2_y = 0.23 + step_bounce * 0.8 + abs(sin(sim_t * 5.0)) * 0.01;
	vec2 f2_base = vec2(f2_x, f2_y);

	// Ground contact shadows
	float sh1 = 1.0 - smoothstep(0.03, 0.09, length(vec2(ap.x - f1_x, (ap.y - 0.22) * 3.2)));
	float sh2 = 1.0 - smoothstep(0.03, 0.09, length(vec2(ap.x - f2_x, (ap.y - 0.22) * 3.2)));
	col *= 1.0 - max(sh1, sh2) * 0.42;

	// Draw Fighter 1 (Blue & White / Cyan gloves)
	vec3 f1_col = draw_fighter(
		ap, f1_base, 1.0,
		max(melee_p, hadou_pose),
		max(melee_p, hadou_pose),
		kick_pose,
		step(0.02, jump_arc),
		0.0,
		vec3(0.16, 0.44, 0.92),
		vec3(0.98, 0.82, 0.22),
		vec3(0.92, 0.72, 0.56),
		sim_t
	);
	if (f1_col.r >= 0.0) {
		col = f1_col;
	}

	// Draw Fighter 2 (Crimson & Gold)
	float f2_counter = smoothstep(0.84, 0.90, cyc) * (1.0 - smoothstep(0.94, 0.99, cyc));
	vec3 f2_col = draw_fighter(
		ap, f2_base, -1.0,
		f2_counter,
		f2_counter,
		0.0,
		0.0,
		hit_any,
		vec3(0.78, 0.14, 0.22),
		vec3(0.20, 0.92, 0.95),
		vec3(0.84, 0.64, 0.50),
		sim_t + 1.5
	);
	if (f2_col.r >= 0.0) {
		col = f2_col;
	}

	// 4. Glowing Hadouken Energy Blast
	if (cyc > 0.43 && cyc < 0.71) {
		float ft = clamp((cyc - 0.43) / 0.27, 0.0, 1.0);
		vec2 fb_pos = vec2(mix(f1_x + 0.16, f2_x - 0.04, ft), 0.37);
		vec2 fp = ap - fb_pos;
		float fd = length(fp);
		// Glowing comet tail behind blast
		float tail = (1.0 - smoothstep(0.015, 0.055, abs(fp.y))) * step(fp.x, 0.0) * smoothstep(-0.18, 0.0, fp.x);
		col += vec3(0.15, 0.75, 1.0) * tail * 0.75;
		if (fd < 0.095) {
			float outer = (1.0 - smoothstep(0.035, 0.095, fd)) * 0.85;
			float core = 1.0 - smoothstep(0.022, 0.042, fd);
			col += vec3(0.22, 0.85, 1.0) * outer;
			col = mix(col, vec3(0.96, 0.99, 1.0), core);
		}
	}

	// 5. Impact Starburst Flash on Hit
	if (hit_any > 0.05) {
		vec2 hit_pos = vec2(f2_x - 0.05, 0.37);
		vec2 hp_vec = ap - hit_pos;
		float hd = length(hp_vec);
		float ring_r = 0.04 + (1.0 - hit_any) * 0.08;
		float ring = (1.0 - smoothstep(0.004, 0.014, abs(hd - ring_r))) * hit_any;
		float star = (1.0 - smoothstep(0.02, 0.09, hd)) * hit_any;
		col += vec3(1.0, 0.85, 0.28) * (star * 0.9 + ring * 0.85);
	}

	// 6. Iconic Arcade Fighting Game HUD
	// Center round timer box
	float timer_bg = box_mask(uv, vec2(0.50, 0.88), vec2(0.050, 0.052));
	col = mix(col, vec3(0.08, 0.09, 0.14), timer_bg * 0.92);
	float timer_rim = box_mask(uv, vec2(0.50, 0.88), vec2(0.042, 0.040));
	col = mix(col, vec3(0.98, 0.84, 0.26), timer_rim * 0.28);
	// Two bold timer digits inside box
	float d_left = box_mask(uv, vec2(0.482, 0.88), vec2(0.011, 0.024));
	float d_right = box_mask(uv, vec2(0.518, 0.88), vec2(0.011, 0.024));
	col = mix(col, vec3(1.0, 0.94, 0.55), max(d_left, d_right));

	// Left P1 Health Bar (drains slightly over round)
	float p1_bg = box_mask(uv, vec2(0.245, 0.885), vec2(0.185, 0.032));
	col = mix(col, vec3(0.08, 0.08, 0.12), p1_bg);
	float p1_dmg_bg = box_mask(uv, vec2(0.245, 0.885), vec2(0.176, 0.022));
	col = mix(col, vec3(0.55, 0.10, 0.14), p1_dmg_bg);
	float p1_hp = clamp(0.84 - 0.10 * sin(sim_t * 0.5), 0.35, 0.95);
	float p1_w = 0.176 * p1_hp;
	float p1_fill = box_mask(uv, vec2(0.421 - p1_w, 0.885), vec2(p1_w, 0.022));
	col = mix(col, vec3(0.98, 0.84, 0.18), p1_fill);

	// Right P2 / CPU Health Bar (takes big chunks on hit!)
	float p2_bg = box_mask(uv, vec2(0.755, 0.885), vec2(0.185, 0.032));
	col = mix(col, vec3(0.08, 0.08, 0.12), p2_bg);
	float p2_dmg_bg = box_mask(uv, vec2(0.755, 0.885), vec2(0.176, 0.022));
	col = mix(col, vec3(0.55, 0.10, 0.14), p2_dmg_bg);
	float p2_hp = clamp(0.88 - cyc * 0.36 - dmg * 0.25, 0.18, 0.95);
	float p2_w = 0.176 * p2_hp;
	float p2_fill = box_mask(uv, vec2(0.579 + p2_w, 0.885), vec2(p2_w, 0.022));
	col = mix(col, vec3(0.22, 0.90, 0.45), p2_fill);

	// Round win indicator pips under health bars
	for (int p = 0; p < 2; p++) {
		float pip1 = 1.0 - smoothstep(0.010, 0.015, length(ap - vec2(0.68 - float(p) * 0.05, 0.815)));
		float pip2 = 1.0 - smoothstep(0.010, 0.015, length(ap - vec2(1.04 + float(p) * 0.05, 0.815)));
		col = mix(col, vec3(1.0, 0.86, 0.25), pip1);
		col = mix(col, (p == 0) ? vec3(1.0, 0.86, 0.25) : vec3(0.25, 0.28, 0.36), pip2);
	}

	// Bottom Super Combo Gauges
	float sup1_bg = box_mask(uv, vec2(0.20, 0.075), vec2(0.14, 0.020));
	col = mix(col, vec3(0.06, 0.08, 0.14), sup1_bg * 0.9);
	float sup1_v = fract(sim_t * 0.19);
	float sup1_fill = box_mask(uv, vec2(0.068 + sup1_v * 0.132, 0.075), vec2(sup1_v * 0.132, 0.011));
	col = mix(col, vec3(0.20, 0.88, 1.0), sup1_fill);

	float sup2_bg = box_mask(uv, vec2(0.80, 0.075), vec2(0.14, 0.020));
	col = mix(col, vec3(0.06, 0.08, 0.14), sup2_bg * 0.9);
	float sup2_v = 0.45 + 0.35 * sin(sim_t * 0.8);
	float sup2_fill = box_mask(uv, vec2(0.932 - sup2_v * 0.132, 0.075), vec2(sup2_v * 0.132, 0.011));
	col = mix(col, vec3(0.98, 0.32, 0.65), sup2_fill);

	// 7. Progressive system-damage alert overlay (when PC health drops below 55%)
	if (dmg > 0.45) {
		float warn_strength = smoothstep(0.45, 0.85, dmg);
		float pulse = 0.65 + 0.35 * sin(t * 6.5);
		float w_box = box_mask(uv, vec2(0.50, 0.52), vec2(0.17, 0.11));
		float w_inner = box_mask(uv, vec2(0.50, 0.52), vec2(0.155, 0.092));
		col = mix(col, vec3(0.95, 0.22, 0.20), w_box * warn_strength * pulse);
		col = mix(col, vec3(0.14, 0.03, 0.05), w_inner * warn_strength);
		float ex_bar = box_mask(uv, vec2(0.50, 0.54), vec2(0.014, 0.034));
		float ex_dot = box_mask(uv, vec2(0.50, 0.475), vec2(0.014, 0.012));
		col = mix(col, vec3(1.0, 0.86, 0.25), max(ex_bar, ex_dot) * warn_strength * pulse);
	}

	return col;
}

vec3 bsod_scene(vec2 uv) {
	vec3 col = vec3(0.04, 0.30, 0.74);

	// Crisp ":(" sad emoticon in upper-left
	float dot_top = 1.0 - smoothstep(0.020, 0.026, length(vec2((uv.x - 0.16) * 1.722, uv.y - 0.77)));
	float dot_bot = 1.0 - smoothstep(0.020, 0.026, length(vec2((uv.x - 0.16) * 1.722, uv.y - 0.63)));
	col = mix(col, vec3(0.96, 0.98, 1.0), max(dot_top, dot_bot));

	// Curved "(" arc to the right of the two dots
	vec2 arc_p = vec2((uv.x - 0.32) * 1.722, uv.y - 0.70);
	float arc_r = length(arc_p);
	float arc_ring = (1.0 - smoothstep(0.012, 0.018, abs(arc_r - 0.115))) * step(arc_p.x, -0.02);
	col = mix(col, vec3(0.96, 0.98, 1.0), arc_ring);

	// Stylized error headline & description bars
	float l1 = box_mask(uv, vec2(0.46, 0.49), vec2(0.32, 0.020));
	float l2 = box_mask(uv, vec2(0.42, 0.42), vec2(0.28, 0.016));
	float l3 = box_mask(uv, vec2(0.38, 0.35), vec2(0.24, 0.016));
	col = mix(col, vec3(0.92, 0.96, 1.0), max(l1, max(l2, l3)));

	// QR code square in bottom-left + stop code text bars beside it
	float qr_bg = box_mask(uv, vec2(0.19, 0.18), vec2(0.055, 0.080));
	col = mix(col, vec3(0.95, 0.97, 1.0), qr_bg);
	if (qr_bg > 0.5) {
		vec2 qg = floor(uv * vec2(55.0, 32.0));
		float qbit = step(0.48, hash(qg + vec2(4.2, 9.1)));
		col = mix(col, vec3(0.04, 0.30, 0.74), qbit);
	}
	float s1 = box_mask(uv, vec2(0.41, 0.21), vec2(0.14, 0.013));
	float s2 = box_mask(uv, vec2(0.45, 0.16), vec2(0.18, 0.013));
	col = mix(col, vec3(0.84, 0.92, 1.0), max(s1, s2));

	return col;
}

void fragment() {
	vec2 uv = screen_uv;
	float t = TIME;

	// Localized horizontal tear bands when damaged
	float slow_step = floor(t * 4.0);
	float band = floor(uv.y * 12.0);
	float tear_trig = step(1.0 - damage_level * 0.28, hash(vec2(band, slow_step)));
	uv.x = clamp(uv.x + tear_trig * (hash(vec2(slow_step, band)) - 0.5) * 0.11 * damage_level, 0.0, 1.0);

	vec3 col;
	if (is_bsod >= 0.5) {
		col = bsod_scene(uv);
	} else {
		float chroma = damage_level * 0.014;
		if (chroma > 0.002) {
			float r = game_scene(clamp(uv + vec2(chroma, 0.0), 0.0, 1.0), t, damage_level).r;
			float g = game_scene(uv, t, damage_level).g;
			float b = game_scene(clamp(uv - vec2(chroma, 0.0), 0.0, 1.0), t, damage_level).b;
			col = vec3(r, g, b);
		} else {
			col = game_scene(uv, t, damage_level);
		}

		// Corrupted artifact macroblocks when damaged
		if (damage_level > 0.18) {
			vec2 block_uv = floor(uv * vec2(18.0, 10.0));
			float block_noise = hash(block_uv + vec2(slow_step * 0.7, 3.1));
			float block_mask = step(1.0 - damage_level * 0.20, block_noise);
			vec3 glitch_col = (fract(block_noise * 13.7) > 0.5) ? vec3(0.20, 0.92, 0.98) : vec3(0.98, 0.25, 0.72);
			col = mix(col, glitch_col, block_mask * 0.55);
		}
	}

	// Impact crack & flash when a pencil hits the monitor directly
	if (hit_flash > 0.01) {
		vec2 cp = vec2((screen_uv.x - 0.5) * 1.722, screen_uv.y - 0.5);
		float crack_a = 1.0 - smoothstep(0.003, 0.010, seg_dist(cp, vec2(-0.18, 0.05), vec2(0.06, -0.02)));
		float crack_b = 1.0 - smoothstep(0.003, 0.010, seg_dist(cp, vec2(0.06, -0.02), vec2(0.24, 0.14)));
		float crack_c = 1.0 - smoothstep(0.003, 0.010, seg_dist(cp, vec2(0.06, -0.02), vec2(0.14, -0.22)));
		float cracks = max(crack_a, max(crack_b, crack_c));
		col = mix(col, vec3(0.95, 0.98, 1.0), cracks * hit_flash);
		col += vec3(0.35, 0.65, 1.0) * hit_flash * 0.35;
	}

	// Subtle glossy diagonal screen reflection + crisp edge darkening inside bezel
	float sheen = smoothstep(0.35, 0.75, screen_uv.x * 0.55 + screen_uv.y * 0.45) * 0.06;
	float edge_x = min(screen_uv.x, 1.0 - screen_uv.x);
	float edge_y = min(screen_uv.y, 1.0 - screen_uv.y);
	float bezel_ao = smoothstep(0.0, 0.035, min(edge_x, edge_y));

	col = (col + sheen) * mix(0.45, 1.0, bezel_ao) * power_flicker;
	ALBEDO = col;
}
```

### pencil_sabotage/scripts/sound_fx.gd
```gdscript
extends Node

var _players: Array[AudioStreamPlayer] = []
var _streams: Dictionary = {}

func _ready() -> void:
	for i in 8:
		var p := AudioStreamPlayer.new()
		p.bus = &"Master"
		add_child(p)
		_players.append(p)

	_streams["throw"] = _make_whoosh(0.14)
	_streams["hit_wood"] = _make_thud(0.11, 145.0, 60.0)
	_streams["hit_metal"] = _make_clink(0.22)
	_streams["hit_electric"] = _make_zap(0.28)
	_streams["gamer_hey"] = _make_grunt(0.26)
	_streams["glance_warn"] = _make_thud(0.08, 220.0, 140.0)
	_streams["victory"] = _make_chime(true)
	_streams["defeat"] = _make_chime(false)

func play(sfx_name: String, volume_db: float = -4.0, pitch_scale: float = 1.0) -> void:
	if not _streams.has(sfx_name):
		return
	for p in _players:
		if not p.playing:
			p.stream = _streams[sfx_name]
			p.volume_db = volume_db
			p.pitch_scale = pitch_scale * randf_range(0.94, 1.06)
			p.play()
			return
	_players[0].stream = _streams[sfx_name]
	_players[0].volume_db = volume_db
	_players[0].pitch_scale = pitch_scale
	_players[0].play()

func _make_wav(samples: PackedFloat32Array, sample_rate: int = 22050) -> AudioStreamWAV:
	var data := PackedByteArray()
	data.resize(samples.size() * 2)
	for i in samples.size():
		var val := int(clampf(samples[i], -1.0, 1.0) * 32767.0)
		data.encode_s16(i * 2, val)
	var wav := AudioStreamWAV.new()
	wav.format = AudioStreamWAV.FORMAT_16_BITS
	wav.mix_rate = sample_rate
	wav.stereo = false
	wav.data = data
	return wav

func _make_whoosh(dur: float) -> AudioStreamWAV:
	var sr := 22050
	var count := int(sr * dur)
	var buf := PackedFloat32Array()
	buf.resize(count)
	var lp := 0.0
	for i in count:
		var t := float(i) / float(count)
		var env := sin(t * PI) * (1.0 - t * 0.4)
		var white := randf_range(-1.0, 1.0)
		lp = lerp(lp, white, 0.15 + 0.35 * sin(t * PI))
		buf[i] = lp * env * 0.55
	return _make_wav(buf, sr)

func _make_thud(dur: float, f_start: float, f_end: float) -> AudioStreamWAV:
	var sr := 22050
	var count := int(sr * dur)
	var buf := PackedFloat32Array()
	buf.resize(count)
	var phase := 0.0
	for i in count:
		var t := float(i) / float(count)
		var freq := lerpf(f_start, f_end, t)
		phase += TAU * freq / float(sr)
		var env := exp(-t * 9.0)
		buf[i] = (sin(phase) * 0.8 + randf_range(-0.2, 0.2) * exp(-t * 25.0)) * env * 0.7
	return _make_wav(buf, sr)

func _make_clink(dur: float) -> AudioStreamWAV:
	var sr := 22050
	var count := int(sr * dur)
	var buf := PackedFloat32Array()
	buf.resize(count)
	for i in count:
		var t := float(i) / float(count)
		var sec := float(i) / float(sr)
		var env := exp(-t * 10.0)
		var s := sin(TAU * 1380.0 * sec) * 0.45 + sin(TAU * 2140.0 * sec) * 0.35 + sin(TAU * 3190.0 * sec) * 0.20
		buf[i] = s * env * 0.55
	return _make_wav(buf, sr)

func _make_zap(dur: float) -> AudioStreamWAV:
	var sr := 22050
	var count := int(sr * dur)
	var buf := PackedFloat32Array()
	buf.resize(count)
	var phase := 0.0
	for i in count:
		var t := float(i) / float(count)
		var freq := 240.0 + sin(t * 50.0) * 180.0
		phase += TAU * freq / float(sr)
		var env := exp(-t * 6.5)
		var saw := fmod(phase / TAU, 1.0) * 2.0 - 1.0
		var crackle: float = randf_range(-1.0, 1.0) * (1.0 if randf() > 0.45 else 0.0)
		buf[i] = (saw * 0.55 + crackle * 0.45) * env * 0.65
	return _make_wav(buf, sr)

func _make_grunt(dur: float) -> AudioStreamWAV:
	var sr := 22050
	var count := int(sr * dur)
	var buf := PackedFloat32Array()
	buf.resize(count)
	var phase := 0.0
	for i in count:
		var t := float(i) / float(count)
		var freq := lerpf(190.0, 295.0, sin(t * PI * 0.85))
		phase += TAU * freq / float(sr)
		var env := sin(t * PI)
		var pulse := sin(phase) + 0.5 * sin(phase * 2.0) + 0.25 * sin(phase * 3.0)
		buf[i] = clampf(pulse * 0.4, -0.8, 0.8) * env * 0.6
	return _make_wav(buf, sr)

func _make_chime(is_win: bool) -> AudioStreamWAV:
	var sr := 22050
	var dur := 0.65
	var count := int(sr * dur)
	var buf := PackedFloat32Array()
	buf.resize(count)
	var notes := [523.25, 659.25, 783.99, 1046.5] if is_win else [392.0, 349.23, 311.13, 261.63]
	for i in count:
		var t := float(i) / float(count)
		var idx := mini(int(t * notes.size()), notes.size() - 1)
		var local_t := fmod(t * notes.size(), 1.0)
		var freq: float = notes[idx]
		var sec := float(i) / float(sr)
		var env := exp(-local_t * 4.5) * (1.0 - t * 0.25)
		buf[i] = sin(TAU * freq * sec) * env * 0.55
	return _make_wav(buf, sr)
```

### pencil_sabotage/scripts/pencil_projectile.gd
```gdscript
extends Node3D

signal hit_target(hit_info: Dictionary)

@export var gravity: float = 9.8
@export var max_lifetime: float = 12.0

var velocity: Vector3 = Vector3.ZERO
var is_flying: bool = true
var _age: float = 0.0
var _roll_angle: float = 0.0
var _mesh_root: Node3D = null
var _stuck_quiver_time: float = 0.0
var _bounce_mode: bool = false
var _bounce_count: int = 0

func _ready() -> void:
	var scad_res := load("res://assets/models/pencil.scad") as PackedScene
	if scad_res:
		_mesh_root = scad_res.instantiate()
		add_child(_mesh_root)

func launch(start_pos: Vector3, initial_velocity: Vector3) -> void:
	global_position = start_pos
	velocity = initial_velocity
	is_flying = true
	if velocity.length_squared() > 0.001:
		look_at(global_position + velocity.normalized(), Vector3.UP)

func _physics_process(delta: float) -> void:
	_age += delta
	if _age > max_lifetime:
		queue_free()
		return

	if not is_flying:
		if _stuck_quiver_time > 0.0 and _mesh_root:
			_stuck_quiver_time -= delta
			var amp := _stuck_quiver_time * 0.14
			_mesh_root.rotation.x = sin(_age * 55.0) * amp
			_mesh_root.rotation.y = cos(_age * 47.0) * amp
		elif _mesh_root:
			_mesh_root.rotation.x = 0.0
			_mesh_root.rotation.y = 0.0
		return

	var prev_pos := global_position
	velocity.y -= gravity * delta
	var next_pos := prev_pos + velocity * delta

	var space_state := get_world_3d().direct_space_state
	var query := PhysicsRayQueryParameters3D.create(prev_pos, next_pos)
	query.collide_with_areas = true
	query.collide_with_bodies = true
	var result := space_state.intersect_ray(query)

	if not result.is_empty():
		_handle_collision(result)
		return

	global_position = next_pos
	if _bounce_mode:
		rotate_x(delta * 11.0)
		rotate_y(delta * 7.5)
	elif velocity.length_squared() > 0.01:
		look_at(global_position + velocity.normalized(), Vector3.UP)
		if _mesh_root:
			_roll_angle += delta * 18.0
			_mesh_root.rotation.z = _roll_angle

func _handle_collision(result: Dictionary) -> void:
	var collider: Object = result["collider"]
	var hit_pos: Vector3 = result["position"]
	var hit_normal: Vector3 = result["normal"]

	var hit_type := "room"
	var part_name := ""
	var should_stick := true

	if collider is Node:
		var node := collider as Node
		if node.is_in_group("pc_component"):
			hit_type = "pc_component"
			part_name = node.get_meta("part_name", "PC Part")
			should_stick = true
		elif node.is_in_group("pc_armor"):
			hit_type = "pc_armor"
			should_stick = false
		elif node.is_in_group("monitor"):
			hit_type = "monitor"
			should_stick = false
		elif node.is_in_group("gamer"):
			hit_type = "gamer"
			should_stick = false
		elif node.is_in_group("soft_prop"):
			hit_type = "soft_prop"
			should_stick = true

	var is_first_hit := not _bounce_mode

	if should_stick and not _bounce_mode:
		is_flying = false
		var incoming_dir := velocity.normalized()
		global_position = hit_pos - incoming_dir * 0.055
		look_at(global_position + incoming_dir, Vector3.UP)
		_stuck_quiver_time = 0.35
		if collider is Node3D:
			reparent(collider as Node3D, true)
	else:
		# Ricochet / bounce off metal, glass, gamer, or subsequent floor landings
		_bounce_count += 1
		var reflected := velocity.bounce(hit_normal)
		var normal_speed := reflected.dot(hit_normal)
		var v_normal := hit_normal * (normal_speed * (0.36 if is_first_hit else 0.28))
		var v_tangent := (reflected - hit_normal * normal_speed) * (0.42 if is_first_hit else 0.55)
		velocity = v_normal + v_tangent

		# Only add a small upward pop on the initial ricochet off vertical surfaces
		if is_first_hit and hit_normal.y < 0.5:
			velocity.y += 0.85

		var settle_on_floor := hit_normal.y > 0.6 and (velocity.y < 0.55 or _bounce_count >= 3)
		if settle_on_floor or velocity.length() < 0.9 or _bounce_count >= 5:
			is_flying = false
			var tangent_dir := velocity - hit_normal * velocity.dot(hit_normal)
			if tangent_dir.length_squared() < 0.0001:
				var fallback := Vector3.RIGHT if absf(hit_normal.dot(Vector3.RIGHT)) < 0.9 else Vector3.FORWARD
				tangent_dir = hit_normal.cross(fallback)
			tangent_dir = tangent_dir.normalized()
			global_position = hit_pos + hit_normal * 0.009
			look_at(global_position + tangent_dir, hit_normal)
			velocity = Vector3.ZERO
		else:
			global_position = hit_pos + hit_normal * 0.015
			_bounce_mode = true

	if is_first_hit:
		hit_target.emit({
			"type": hit_type,
			"part_name": part_name,
			"position": hit_pos,
			"normal": hit_normal,
			"collider": collider
		})
```

### pencil_sabotage/scripts/pc_target.gd
```gdscript
extends Node3D

signal component_damaged(part_id: String, part_label: String, damage: float, world_pos: Vector3, is_critical: bool)
signal armor_ricochet(world_pos: Vector3)

var _pc_model: Node3D = null
var _anim_player: AnimationPlayer = null
var _part_health: Dictionary = {
	"cpu_fan": 32.0,
	"gpu": 32.0,
	"ram": 24.0,
	"psu": 32.0,
	"front_fan": 24.0
}
var _destroyed_parts: Dictionary = {}
var _smoke_particles: Array[CPUParticles3D] = []
var _smoke_texture: ImageTexture = null
var _is_broken: bool = false

func _ready() -> void:
	_smoke_texture = _create_smoke_texture(64)
	var scad_res := load("res://assets/models/pc_tower.scad") as PackedScene
	if scad_res:
		_pc_model = scad_res.instantiate()
		add_child(_pc_model)
		_setup_animations(_pc_model)
	_build_hitboxes()

func _setup_animations(root: Node) -> void:
	_anim_player = _find_animation_player(root)
	if _anim_player:
		for anim_name in _anim_player.get_animation_list():
			var anim := _anim_player.get_animation(anim_name)
			if anim_name == "running":
				anim.loop_mode = Animation.LOOP_LINEAR
			else:
				anim.loop_mode = Animation.LOOP_NONE
		if _anim_player.has_animation("running"):
			_anim_player.play("running")

func _find_animation_player(node: Node) -> AnimationPlayer:
	if node is AnimationPlayer:
		return node as AnimationPlayer
	for child in node.get_children():
		var found := _find_animation_player(child)
		if found:
			return found
	return null

func _build_hitboxes() -> void:
	# Convert OpenSCAD coordinates (x, y, z) -> Godot local (x, z, -y)
	# Internal vulnerable targets (exposed on local -X open side)
	_add_hitbox("cpu_fan", "CPU Cooler", Vector3(0.02, 0.34, 0.06), Vector3(0.07, 0.11, 0.11), true)
	_add_hitbox("ram", "RAM Sticks", Vector3(0.05, 0.34, -0.06), Vector3(0.05, 0.13, 0.06), true)
	_add_hitbox("gpu", "GPU", Vector3(0.01, 0.22, 0.04), Vector3(0.14, 0.07, 0.27), true)
	_add_hitbox("psu", "PSU", Vector3(0.0, 0.08, 0.11), Vector3(0.16, 0.10, 0.19), true)
	_add_hitbox("front_fan", "Front Fan", Vector3(0.0, 0.25, -0.19), Vector3(0.12, 0.13, 0.06), true)

	# Metal armor plates (right side +X, back +Z, top, bottom) so misses clank realistically
	_add_hitbox("armor_right", "Case", Vector3(0.105, 0.25, 0.0), Vector3(0.025, 0.46, 0.46), false)
	_add_hitbox("armor_top", "Case", Vector3(0.0, 0.47, 0.0), Vector3(0.22, 0.03, 0.46), false)
	_add_hitbox("armor_back", "Case", Vector3(0.0, 0.25, 0.22), Vector3(0.22, 0.46, 0.03), false)

func _add_hitbox(part_id: String, part_label: String, local_pos: Vector3, box_size: Vector3, is_vulnerable: bool) -> void:
	var body := StaticBody3D.new()
	body.name = "Hitbox_" + part_id
	body.position = local_pos
	if is_vulnerable:
		body.add_to_group("pc_component")
		body.set_meta("part_id", part_id)
		body.set_meta("part_name", part_label)
	else:
		body.add_to_group("pc_armor")

	var col := CollisionShape3D.new()
	var shape := BoxShape3D.new()
	shape.size = box_size
	col.shape = shape
	body.add_child(col)
	add_child(body)

func apply_hit(collider: Object, hit_pos: Vector3) -> void:
	if _is_broken:
		return
	if collider is Node:
		var node := collider as Node
		if node.is_in_group("pc_armor"):
			armor_ricochet.emit(hit_pos)
			_spawn_sparks(hit_pos, Color(1.0, 0.85, 0.45), 10)
			return
		if node.is_in_group("pc_component"):
			var part_id: String = node.get_meta("part_id", "gpu")
			var part_label: String = node.get_meta("part_name", "PC Part")
			var is_crit := (part_id == "cpu_fan" or part_id == "gpu" or part_id == "psu")
			var dmg := 16.0 if is_crit else 12.0
			if _destroyed_parts.get(part_id, false):
				dmg *= 0.25

			_part_health[part_id] = maxf(0.0, _part_health.get(part_id, 30.0) - dmg)
			if _part_health[part_id] <= 0.0 and not _destroyed_parts.get(part_id, false):
				_destroyed_parts[part_id] = true
				_attach_persistent_smoke((node as Node3D).global_position)

			_spawn_sparks(hit_pos, Color(0.45, 0.85, 1.0), 22)
			component_damaged.emit(part_id, part_label, dmg, hit_pos, is_crit)

func set_broken_state() -> void:
	if _is_broken:
		return
	_is_broken = true
	if _anim_player and _anim_player.has_animation("broken"):
		_anim_player.play("broken")
	_dim_rgb_lights()
	_attach_persistent_smoke(global_position + Vector3(0.0, 0.28, 0.0), 1.5)

func reset_pc() -> void:
	_is_broken = false
	_part_health = {
		"cpu_fan": 32.0,
		"gpu": 32.0,
		"ram": 24.0,
		"psu": 32.0,
		"front_fan": 24.0
	}
	_destroyed_parts.clear()
	for s in _smoke_particles:
		if is_instance_valid(s):
			s.queue_free()
	_smoke_particles.clear()
	# Remove any stuck pencils from hitboxes
	for child in get_children():
		if child is StaticBody3D:
			for sub in child.get_children():
				if not (sub is CollisionShape3D):
					sub.queue_free()
	if _anim_player and _anim_player.has_animation("running"):
		_anim_player.play("running")
	_restore_rgb_lights()

func _dim_rgb_lights() -> void:
	if not _pc_model:
		return
	_set_emissive_multiplier(_pc_model, 0.0)

func _restore_rgb_lights() -> void:
	if not _pc_model:
		return
	_set_emissive_multiplier(_pc_model, 1.5)

func _set_emissive_multiplier(node: Node, energy: float) -> void:
	if node is MeshInstance3D:
		var mi := node as MeshInstance3D
		if mi.mesh:
			for i in mi.mesh.get_surface_count():
				var mat := mi.get_active_material(i)
				if mat is StandardMaterial3D:
					var sm := mat as StandardMaterial3D
					if sm.emission_enabled or sm.resource_name.ends_with("rgb") or sm.resource_name.ends_with("led") or sm.resource_name == "fan_blades_glow":
						var dup := sm.duplicate() as StandardMaterial3D
						dup.emission_enabled = energy > 0.0
						dup.emission_energy_multiplier = energy
						mi.set_surface_override_material(i, dup)
	for child in node.get_children():
		_set_emissive_multiplier(child, energy)

func _spawn_sparks(world_pos: Vector3, spark_color: Color, amount: int) -> void:
	var p := CPUParticles3D.new()
	get_tree().current_scene.add_child(p)
	p.global_position = world_pos
	p.one_shot = true
	p.explosiveness = 0.95
	p.amount = amount
	p.lifetime = 0.45
	p.direction = Vector3(-1.0, 0.8, 0.5)
	p.spread = 85.0
	p.initial_velocity_min = 2.0
	p.initial_velocity_max = 5.5
	p.gravity = Vector3(0.0, -9.8, 0.0)

	var mesh := SphereMesh.new()
	mesh.radius = 0.011
	mesh.height = 0.022
	var mat := StandardMaterial3D.new()
	mat.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	mat.albedo_color = spark_color
	mesh.material = mat
	p.mesh = mesh

	p.emitting = true
	get_tree().create_timer(0.8).timeout.connect(func():
		if is_instance_valid(p):
			p.queue_free()
	)

func _create_smoke_texture(size: int) -> ImageTexture:
	var img := Image.create(size, size, false, Image.FORMAT_RGBA8)
	var center := Vector2(size * 0.5, size * 0.5)
	var max_r := size * 0.48
	for y in size:
		for x in size:
			var pos := Vector2(x + 0.5, y + 0.5)
			var d := pos.distance_to(center) / max_r
			if d >= 1.0:
				img.set_pixel(x, y, Color(1.0, 1.0, 1.0, 0.0))
				continue
			# Smooth multi-frequency wispy cloud modulation
			var nx := float(x) / float(size)
			var ny := float(y) / float(size)
			var wisp := 0.55 \
				+ 0.25 * sin(nx * 13.0 + cos(ny * 9.0) * 1.4) \
				+ 0.20 * cos(ny * 17.0 - sin(nx * 11.0) * 1.3)
			var radial := pow(clampf(1.0 - d, 0.0, 1.0), 2.1)
			var alpha := clampf(radial * clampf(wisp, 0.25, 1.0), 0.0, 1.0)
			img.set_pixel(x, y, Color(1.0, 1.0, 1.0, alpha))
	return ImageTexture.create_from_image(img)

func _attach_persistent_smoke(world_pos: Vector3, scale_mul: float = 1.0) -> void:
	var p := CPUParticles3D.new()
	get_tree().current_scene.add_child(p)
	p.global_position = world_pos
	p.amount = int(26 * scale_mul)
	p.lifetime = 2.4
	p.preprocess = 0.4
	p.emission_shape = CPUParticles3D.EMISSION_SHAPE_SPHERE
	p.emission_sphere_radius = 0.035 * scale_mul
	p.direction = Vector3(-0.22, 1.0, 0.14)
	p.spread = 22.0
	p.initial_velocity_min = 0.28
	p.initial_velocity_max = 0.52
	p.gravity = Vector3(0.04, 0.18, 0.0)
	p.damping_min = 0.18
	p.damping_max = 0.32
	p.angle_min = -180.0
	p.angle_max = 180.0
	p.angular_velocity_min = -32.0
	p.angular_velocity_max = 32.0
	p.scale_amount_min = 0.16 * scale_mul
	p.scale_amount_max = 0.26 * scale_mul

	var scale_curve := Curve.new()
	scale_curve.add_point(Vector2(0.0, 0.35))
	scale_curve.add_point(Vector2(0.35, 1.15))
	scale_curve.add_point(Vector2(1.0, 2.20))
	p.scale_amount_curve = scale_curve

	var grad := Gradient.new()
	grad.set_color(0, Color(0.12, 0.12, 0.14, 0.0))
	grad.set_color(1, Color(0.45, 0.46, 0.48, 0.0))
	grad.add_point(0.14, Color(0.18, 0.18, 0.21, 0.52))
	grad.add_point(0.55, Color(0.30, 0.31, 0.34, 0.30))
	p.color_ramp = grad

	var mesh := QuadMesh.new()
	mesh.size = Vector2(1.0, 1.0)
	var mat := StandardMaterial3D.new()
	mat.transparency = BaseMaterial3D.TRANSPARENCY_ALPHA
	mat.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	mat.vertex_color_use_as_albedo = true
	mat.billboard_mode = BaseMaterial3D.BILLBOARD_PARTICLES
	mat.billboard_keep_scale = true
	mat.albedo_texture = _smoke_texture
	mat.disable_receive_shadows = true
	mesh.material = mat

	p.mesh = mesh
	p.emitting = true
	_smoke_particles.append(p)
```

### pencil_sabotage/scripts/gamer_npc.gd
```gdscript
extends Node3D

signal look_back_warning
signal look_back_started
signal look_back_ended

var _gamer_model: Node3D = null
var _chair_model: Node3D = null
var _anim_player: AnimationPlayer = null
var _chair_anim_player: AnimationPlayer = null
var _is_active: bool = false
var _is_warning: bool = false
var _is_looking_back: bool = false
var _look_back_timer: float = 0.0
var _warning_timer: float = 0.0
var _next_glance_timer: float = 5.5
var _alertness: float = 0.0
var _is_defeated: bool = false

func _ready() -> void:
	var chair_res := load("res://assets/models/gaming_chair.scad") as PackedScene
	if chair_res:
		_chair_model = chair_res.instantiate()
		add_child(_chair_model)
		_chair_anim_player = _setup_animations(_chair_model)

	var scad_res := load("res://assets/models/gamer_character.scad") as PackedScene
	if scad_res:
		_gamer_model = scad_res.instantiate()
		add_child(_gamer_model)
		_anim_player = _setup_animations(_gamer_model)

	_play_synced("playing", 0.0)
	_build_hitbox()

func _setup_animations(root: Node) -> AnimationPlayer:
	var player := _find_animation_player(root)
	if player:
		for anim_name in player.get_animation_list():
			var anim := player.get_animation(anim_name)
			if anim_name == "playing":
				anim.loop_mode = Animation.LOOP_LINEAR
			else:
				anim.loop_mode = Animation.LOOP_NONE
	return player

func _find_animation_player(node: Node) -> AnimationPlayer:
	if node is AnimationPlayer:
		return node as AnimationPlayer
	for child in node.get_children():
		var found := _find_animation_player(child)
		if found:
			return found
	return null

func _play_synced(anim_name: String, blend: float) -> void:
	if _anim_player and _anim_player.has_animation(anim_name):
		_anim_player.play(anim_name, blend)
	if _chair_anim_player and _chair_anim_player.has_animation(anim_name):
		_chair_anim_player.play(anim_name, blend)

func _build_hitbox() -> void:
	var body := StaticBody3D.new()
	body.name = "GamerHitbox"
	body.add_to_group("gamer")

	var torso_col := CollisionShape3D.new()
	var torso_shape := CapsuleShape3D.new()
	torso_shape.radius = 0.21
	torso_shape.height = 0.72
	torso_col.shape = torso_shape
	torso_col.position = Vector3(0.0, 0.88, 0.0)
	body.add_child(torso_col)

	var head_col := CollisionShape3D.new()
	var head_shape := SphereShape3D.new()
	head_shape.radius = 0.16
	head_col.shape = head_shape
	head_col.position = Vector3(0.0, 1.32, 0.0)
	body.add_child(head_col)

	add_child(body)

func _process(delta: float) -> void:
	if _is_defeated or not _is_active:
		return

	if _is_looking_back:
		_look_back_timer -= delta
		if _look_back_timer <= 0.0:
			_stop_looking_back()
	elif _is_warning:
		_warning_timer -= delta
		if _warning_timer <= 0.0:
			_is_warning = false
			trigger_look_back(randf_range(1.9, 2.7) + _alertness * 0.6)
	else:
		_next_glance_timer -= delta
		if _next_glance_timer <= 0.0:
			_is_warning = true
			_warning_timer = 0.65
			look_back_warning.emit()

func set_active(active: bool) -> void:
	_is_active = active

func set_alertness(value: float) -> void:
	_alertness = clampf(value, 0.0, 1.0)

func _roll_next_glance_time() -> float:
	var min_t := lerpf(4.6, 2.4, _alertness)
	var max_t := lerpf(7.6, 4.2, _alertness)
	return randf_range(min_t, max_t)

func trigger_look_back(duration: float = 2.1) -> void:
	if _is_defeated:
		return
	_is_warning = false
	_warning_timer = 0.0
	_look_back_timer = maxf(_look_back_timer, duration)
	if not _is_looking_back:
		_is_looking_back = true
		_play_synced("startled", 0.12)
		look_back_started.emit()

func _stop_looking_back() -> void:
	_is_looking_back = false
	_next_glance_timer = _roll_next_glance_time()
	_play_synced("playing", 0.25)
	look_back_ended.emit()

func is_looking_back() -> bool:
	return _is_looking_back

func is_warning() -> bool:
	return _is_warning

func set_defeat() -> void:
	_is_defeated = true
	_is_active = false
	_is_warning = false
	_is_looking_back = false
	_play_synced("defeat", 0.2)

func reset_gamer() -> void:
	_is_defeated = false
	_is_warning = false
	_is_looking_back = false
	_look_back_timer = 0.0
	_warning_timer = 0.0
	_alertness = 0.0
	_next_glance_timer = _roll_next_glance_time()
	_play_synced("playing", 0.1)
```

### pencil_sabotage/scripts/player_couch.gd
```gdscript
extends Node3D

signal pencil_thrown(projectile: Node3D)
signal touch_detected

@export var mouse_sensitivity: float = 0.0024
@export var touch_sensitivity: float = 0.0038
@export var min_throw_speed: float = 7.2
@export var max_throw_speed: float = 18.5
@export var charge_rate: float = 0.95
@export var lean_distance: float = 0.42

@onready var camera: Camera3D = $Camera3D
@onready var hand_pivot: Node3D = $Camera3D/HandPivot

var _yaw: float = 0.0
var _pitch: float = 0.02
var _lean_offset: float = 0.0
var _is_charging: bool = false
var _charge: float = 0.0
var _reload_timer: float = 0.0
var _can_play: bool = false
var _held_pencil: Node3D = null
var _pencil_scene: PackedScene = preload("res://scenes/pencil_projectile.tscn")
var _hand_rest_pos: Vector3 = Vector3(0.24, -0.20, -0.38)

var _touch_mode: bool = false
var _aim_touch_idx: int = -1
var _throw_touch_idx: int = -1
var _lean_left_idx: int = -1
var _lean_right_idx: int = -1

var _lean_left_ctrl: Control = null
var _lean_right_ctrl: Control = null
var _throw_ctrl: Control = null
var _pause_ctrl: Control = null

func _ready() -> void:
	_touch_mode = DisplayServer.is_touchscreen_available()
	var scad_res := load("res://assets/models/pencil.scad") as PackedScene
	if scad_res and hand_pivot:
		_held_pencil = scad_res.instantiate()
		hand_pivot.add_child(_held_pencil)
		hand_pivot.position = _hand_rest_pos
		hand_pivot.rotation_degrees = Vector3(8.0, 12.0, -10.0)

func setup_touch_controls(lean_left_ctrl: Control, lean_right_ctrl: Control, throw_ctrl: Control, pause_ctrl: Control) -> void:
	_lean_left_ctrl = lean_left_ctrl
	_lean_right_ctrl = lean_right_ctrl
	_throw_ctrl = throw_ctrl
	_pause_ctrl = pause_ctrl

func set_play_active(active: bool) -> void:
	_can_play = active
	_aim_touch_idx = -1
	_throw_touch_idx = -1
	_lean_left_idx = -1
	_lean_right_idx = -1
	if not active:
		_is_charging = false
		_charge = 0.0
		_reload_timer = 0.0
		Input.mouse_mode = Input.MOUSE_MODE_VISIBLE
	else:
		Input.mouse_mode = Input.MOUSE_MODE_VISIBLE if _touch_mode else Input.MOUSE_MODE_CAPTURED

func get_charge_ratio() -> float:
	return _charge if _is_charging else 0.0

func _enable_touch_mode() -> void:
	if not _touch_mode:
		_touch_mode = true
		Input.mouse_mode = Input.MOUSE_MODE_VISIBLE
		touch_detected.emit()

func _is_point_on_touch_ui(pos: Vector2) -> bool:
	if _pause_ctrl and _pause_ctrl.visible and _pause_ctrl.get_global_rect().has_point(pos):
		return true
	if _throw_ctrl and _throw_ctrl.visible and _throw_ctrl.get_global_rect().has_point(pos):
		return true
	if _lean_left_ctrl and _lean_left_ctrl.visible and _lean_left_ctrl.get_global_rect().has_point(pos):
		return true
	if _lean_right_ctrl and _lean_right_ctrl.visible and _lean_right_ctrl.get_global_rect().has_point(pos):
		return true
	return false

func _input(event: InputEvent) -> void:
	if event is InputEventScreenTouch or event is InputEventScreenDrag:
		_enable_touch_mode()

	if not _can_play:
		return

	if event is InputEventScreenTouch:
		var st := event as InputEventScreenTouch
		if st.pressed:
			if _pause_ctrl and _pause_ctrl.visible and _pause_ctrl.get_global_rect().has_point(st.position):
				return
			elif _throw_ctrl and _throw_ctrl.is_visible_in_tree() and _throw_ctrl.get_global_rect().has_point(st.position):
				if _throw_touch_idx == -1:
					_throw_touch_idx = st.index
					if _reload_timer <= 0.0:
						_is_charging = true
						_charge = 0.08
			elif _lean_left_ctrl and _lean_left_ctrl.is_visible_in_tree() and _lean_left_ctrl.get_global_rect().has_point(st.position):
				if _lean_left_idx == -1:
					_lean_left_idx = st.index
			elif _lean_right_ctrl and _lean_right_ctrl.is_visible_in_tree() and _lean_right_ctrl.get_global_rect().has_point(st.position):
				if _lean_right_idx == -1:
					_lean_right_idx = st.index
			else:
				if _aim_touch_idx == -1:
					_aim_touch_idx = st.index
		else:
			if st.index == _throw_touch_idx:
				_throw_touch_idx = -1
				if _is_charging:
					_release_throw()
			if st.index == _lean_left_idx:
				_lean_left_idx = -1
			if st.index == _lean_right_idx:
				_lean_right_idx = -1
			if st.index == _aim_touch_idx:
				_aim_touch_idx = -1
		return

	if event is InputEventScreenDrag:
		var sd := event as InputEventScreenDrag
		if sd.index == _aim_touch_idx:
			_yaw = clampf(_yaw - sd.relative.x * touch_sensitivity, -0.68, 0.68)
			_pitch = clampf(_pitch - sd.relative.y * touch_sensitivity, -0.35, 0.50)
		return

	# Allow 2-in-1 desktop users with a physical mouse to seamlessly switch back to mouse mode
	if _touch_mode and event is InputEventMouseButton and event.device >= 0:
		var mb_check := event as InputEventMouseButton
		if mb_check.pressed and not _is_point_on_touch_ui(mb_check.position):
			_touch_mode = false
			Input.mouse_mode = Input.MOUSE_MODE_CAPTURED

func _unhandled_input(event: InputEvent) -> void:
	if not _can_play or _touch_mode:
		return

	if event is InputEventMouseMotion and Input.mouse_mode == Input.MOUSE_MODE_CAPTURED:
		var mm := event as InputEventMouseMotion
		_yaw = clampf(_yaw - mm.relative.x * mouse_sensitivity, -0.68, 0.68)
		_pitch = clampf(_pitch - mm.relative.y * mouse_sensitivity, -0.35, 0.50)

	if event is InputEventMouseButton:
		var mb := event as InputEventMouseButton
		if mb.button_index == MOUSE_BUTTON_LEFT:
			if mb.pressed and _reload_timer <= 0.0:
				_is_charging = true
				_charge = 0.08
			elif not mb.pressed and _is_charging:
				_release_throw()
		elif mb.button_index == MOUSE_BUTTON_RIGHT and mb.pressed and _is_charging:
			_is_charging = false
			_charge = 0.0

func _process(delta: float) -> void:
	if not _can_play:
		return

	# Lean left / right on the sofa (A/D or touch buttons)
	var lean_input := Input.get_axis("lean_left", "lean_right")
	if _lean_left_idx != -1:
		lean_input -= 1.0
	if _lean_right_idx != -1:
		lean_input += 1.0
	lean_input = clampf(lean_input, -1.0, 1.0)
	var target_lean := lean_input * lean_distance
	_lean_offset = lerpf(_lean_offset, target_lean, delta * 8.5)
	position.x = _lean_offset

	# Apply gentle head tilt when leaning
	camera.rotation = Vector3(_pitch, _yaw, -_lean_offset * 0.14)

	if _reload_timer > 0.0:
		_reload_timer -= delta
		if hand_pivot:
			hand_pivot.visible = _reload_timer <= 0.06

	if _throw_touch_idx != -1 and not _is_charging and _reload_timer <= 0.0:
		_is_charging = true
		_charge = 0.08

	if _is_charging:
		_charge = minf(1.0, _charge + charge_rate * delta)

	if _lean_left_ctrl:
		_lean_left_ctrl.modulate = Color(0.55, 0.92, 1.0, 1.0) if _lean_left_idx != -1 else Color(1.0, 1.0, 1.0, 0.78)
	if _lean_right_ctrl:
		_lean_right_ctrl.modulate = Color(0.55, 0.92, 1.0, 1.0) if _lean_right_idx != -1 else Color(1.0, 1.0, 1.0, 0.78)
	if _throw_ctrl:
		if _is_charging:
			_throw_ctrl.modulate = Color(1.0, 0.85, 0.35, 1.0)
		elif _reload_timer > 0.0:
			_throw_ctrl.modulate = Color(0.65, 0.65, 0.70, 0.60)
		else:
			_throw_ctrl.modulate = Color(1.0, 1.0, 1.0, 0.85)

	# Animate held pencil pullback / sway
	if hand_pivot:
		var t := Time.get_ticks_msec() * 0.0025
		var sway := Vector3(sin(t) * 0.004, cos(t * 1.3) * 0.004, 0.0)
		var pullback := Vector3(0.03, -0.02, 0.16) * (_charge if _is_charging else 0.0)
		hand_pivot.position = hand_pivot.position.lerp(_hand_rest_pos + sway + pullback, delta * 14.0)
		var target_rot := Vector3(8.0 + _charge * 18.0, 12.0, -10.0 - _charge * 14.0)
		hand_pivot.rotation_degrees = hand_pivot.rotation_degrees.lerp(target_rot, delta * 14.0)

func _release_throw() -> void:
	_is_charging = false
	var charge_used := _charge
	var speed := lerpf(min_throw_speed, max_throw_speed, charge_used)
	_charge = 0.0
	_reload_timer = 0.25

	var proj := _pencil_scene.instantiate()
	get_tree().current_scene.add_child(proj)

	# Aim slightly upward for crisp ballistic feel & spawn from right hand offset
	var forward_dir := -camera.global_transform.basis.z.normalized()
	var right_dir := camera.global_transform.basis.x.normalized()
	var up_dir := camera.global_transform.basis.y.normalized()

	var spawn_pos := camera.global_position + forward_dir * 0.42 + right_dir * 0.10 - up_dir * 0.08

	# Converge throw direction onto the exact crosshair aim point
	var aim_point := camera.global_position + forward_dir * 4.5
	var space_state := get_world_3d().direct_space_state
	var query := PhysicsRayQueryParameters3D.create(
		camera.global_position + forward_dir * 0.3,
		camera.global_position + forward_dir * 15.0
	)
	query.collide_with_areas = true
	query.collide_with_bodies = true
	var hit := space_state.intersect_ray(query)
	if not hit.is_empty():
		var hit_dist := camera.global_position.distance_to(hit["position"])
		if hit_dist > 1.2:
			aim_point = hit["position"]

	var base_dir := (aim_point - spawn_pos).normalized()
	var spread := 0.008 + (0.006 if charge_used > 0.85 else 0.0)
	var jitter := right_dir * randf_range(-spread, spread) + up_dir * randf_range(-spread * 0.7, spread * 0.7)
	var launch_dir := (base_dir + up_dir * 0.032 + jitter).normalized()

	proj.launch(spawn_pos, launch_dir * speed)
	pencil_thrown.emit(proj)
```

### pencil_sabotage/scripts/game_manager.gd
```gdscript
extends Node3D

@onready var pc_target: Node3D = $PCTower
@onready var gamer_npc: Node3D = $GamerCharacter
@onready var player_couch: Node3D = $PlayerCouch
@onready var monitor_root: Node3D = $Monitor
@onready var sfx: Node = $SoundFX

# UI References
@onready var pc_bar: ProgressBar = $HUD/TopLeftPanel/VBox/PCBar
@onready var pc_pct_label: Label = $HUD/TopLeftPanel/VBox/PCLabel
@onready var parts_status_label: Label = $HUD/TopLeftPanel/VBox/PartsStatus
@onready var suspicion_bar: ProgressBar = $HUD/TopRightPanel/VBox/SuspicionBar
@onready var suspicion_alert: Label = $HUD/TopRightPanel/VBox/AlertLabel
@onready var charge_bar: ProgressBar = $HUD/CrosshairRoot/ChargeBar
@onready var crosshair_dot: ColorRect = $HUD/CrosshairRoot/Dot
@onready var feed_label: Label = $HUD/FeedLabel
@onready var bottom_left_panel: PanelContainer = $HUD/BottomLeftPanel
@onready var stats_label: Label = $HUD/BottomLeftPanel/StatsLabel
@onready var start_overlay: Control = $HUD/StartOverlay
@onready var start_desc: Label = $HUD/StartOverlay/Panel/VBox/Desc
@onready var end_overlay: Control = $HUD/EndOverlay
@onready var end_title: Label = $HUD/EndOverlay/Panel/VBox/EndTitle
@onready var end_desc: Label = $HUD/EndOverlay/Panel/VBox/EndDesc
@onready var restart_button: Button = $HUD/EndOverlay/Panel/VBox/RestartButton
@onready var pause_overlay: Control = $HUD/PauseOverlay
@onready var pause_button: Button = $HUD/PauseButton
@onready var touch_controls: Control = $HUD/TouchControls
@onready var lean_left_btn: Control = $HUD/TouchControls/LeanLeftBtn
@onready var lean_right_btn: Control = $HUD/TouchControls/LeanRightBtn
@onready var throw_btn: Control = $HUD/TouchControls/ThrowBtn

var pc_health: float = 100.0
var gamer_suspicion: float = 0.0
var throws_count: int = 0
var hits_count: int = 0
var damaged_parts: Dictionary = {}
var _game_state: String = "menu" # "menu", "playing", "paused", "won", "lost"
var _monitor_mat: ShaderMaterial = null
var _feed_timer: float = 0.0
var _is_touch_mode: bool = false
var _round_id: int = 0
var _end_overlay_tween: Tween = null

func _ready() -> void:
	_load_static_scad_props()
	_setup_monitor_shader()
	_setup_rug_shader()
	_setup_floor_shader()
	_setup_wall_shader()
	_setup_poster_shader()

	pc_target.component_damaged.connect(_on_pc_component_damaged)
	pc_target.armor_ricochet.connect(_on_pc_armor_ricochet)
	player_couch.pencil_thrown.connect(_on_pencil_thrown)
	player_couch.touch_detected.connect(_enable_touch_mode)
	player_couch.setup_touch_controls(lean_left_btn, lean_right_btn, throw_btn, pause_button)
	gamer_npc.look_back_warning.connect(_on_gamer_look_back_warning)
	gamer_npc.look_back_started.connect(_on_gamer_look_back)
	gamer_npc.look_back_ended.connect(_on_gamer_back_to_game)

	if DisplayServer.is_touchscreen_available():
		_enable_touch_mode()

	_show_menu()

func _load_static_scad_props() -> void:
	var props := [
		["RoomEnvironment", "res://assets/models/room_environment.scad", Vector3(0, 0, 0), Vector3(0, 0, 0)],
		["Sofa", "res://assets/models/sofa.scad", Vector3(0, 0, 2.25), Vector3(0, 0, 0)],
		["CoffeeTable", "res://assets/models/coffee_table.scad", Vector3(-0.75, 0, 1.42), Vector3(0, 12, 0)],
		["PencilMug", "res://assets/models/pencil_mug.scad", Vector3(-0.48, 0.418, 1.38), Vector3(0, -35, 0)],
		["ComputerDesk", "res://assets/models/computer_desk.scad", Vector3(0, 0, -2.05), Vector3(0, 180, 0)],
		["Monitor", "res://assets/models/monitor.scad", Vector3(-0.14, 0.76, -2.28), Vector3(0, 180, 0)]
	]

	for item in props:
		var node_name: String = item[0]
		var path: String = item[1]
		var target_node := get_node_or_null(node_name)
		if target_node == null:
			target_node = Node3D.new()
			target_node.name = node_name
			add_child(target_node)
		target_node.position = item[2]
		target_node.rotation_degrees = item[3]
		var scad_res := load(path) as PackedScene
		if scad_res:
			var inst := scad_res.instantiate()
			target_node.add_child(inst)

	_add_prop_colliders()

func _add_prop_colliders() -> void:
	# Monitor hitbox
	var mon_body := StaticBody3D.new()
	mon_body.name = "MonitorHitbox"
	mon_body.add_to_group("monitor")
	var mon_col := CollisionShape3D.new()
	var mon_box := BoxShape3D.new()
	mon_box.size = Vector3(0.65, 0.40, 0.08)
	mon_col.shape = mon_box
	mon_col.position = Vector3(0, 0.32, 0)
	mon_body.add_child(mon_col)
	monitor_root.add_child(mon_body)

	# Desk tabletop hitbox (pencils stick into wood)
	var desk_node := get_node_or_null("ComputerDesk")
	if desk_node:
		var desk_body := StaticBody3D.new()
		desk_body.add_to_group("soft_prop")
		var d_col := CollisionShape3D.new()
		var d_box := BoxShape3D.new()
		d_box.size = Vector3(1.62, 0.76, 0.78)
		d_col.shape = d_box
		d_col.position = Vector3(0, 0.38, 0)
		desk_body.add_child(d_col)
		desk_node.add_child(desk_body)

	# Floor & back wall colliders so missed pencils land naturally
	var room_body := StaticBody3D.new()
	room_body.name = "RoomBounds"
	var floor_col := CollisionShape3D.new()
	var floor_box := BoxShape3D.new()
	floor_box.size = Vector3(7.0, 0.1, 7.0)
	floor_col.shape = floor_box
	floor_col.position = Vector3(0, -0.05, 0)
	room_body.add_child(floor_col)

	var back_wall := CollisionShape3D.new()
	var bw_box := BoxShape3D.new()
	bw_box.size = Vector3(7.0, 3.5, 0.2)
	back_wall.shape = bw_box
	back_wall.position = Vector3(0, 1.6, -3.1)
	room_body.add_child(back_wall)
	add_child(room_body)

func _setup_monitor_shader() -> void:
	var shader := load("res://shaders/monitor_screen.gdshader") as Shader
	if not shader:
		return
	_monitor_mat = ShaderMaterial.new()
	_monitor_mat.shader = shader
	_monitor_mat.set_shader_parameter("damage_level", 0.0)
	_monitor_mat.set_shader_parameter("is_bsod", 0.0)
	_monitor_mat.set_shader_parameter("power_flicker", 1.0)
	_monitor_mat.set_shader_parameter("hit_flash", 0.0)
	_replace_material_by_name(monitor_root, "monitor_screen", _monitor_mat)

func _setup_rug_shader() -> void:
	var room_env := get_node_or_null("RoomEnvironment")
	if not room_env:
		return
	var shader := load("res://shaders/rug_pattern.gdshader") as Shader
	if not shader:
		return
	var rug_mat := ShaderMaterial.new()
	rug_mat.shader = shader
	_replace_material_by_name(room_env, "room_rug", rug_mat)

func _setup_floor_shader() -> void:
	var room_env := get_node_or_null("RoomEnvironment")
	if not room_env:
		return
	var shader := load("res://shaders/wood_floor.gdshader") as Shader
	if not shader:
		return
	var floor_mat := ShaderMaterial.new()
	floor_mat.shader = shader
	_replace_material_by_name(room_env, "room_floor", floor_mat)

func _setup_wall_shader() -> void:
	var room_env := get_node_or_null("RoomEnvironment")
	if not room_env:
		return
	var shader := load("res://shaders/wall_texture.gdshader") as Shader
	if not shader:
		return
	var wall_mat := ShaderMaterial.new()
	wall_mat.shader = shader
	_replace_material_by_name(room_env, "room_walls", wall_mat)

func _setup_poster_shader() -> void:
	var room_env := get_node_or_null("RoomEnvironment")
	if not room_env:
		return
	var shader := load("res://shaders/poster_art.gdshader") as Shader
	if not shader:
		return
	var poster_mat := ShaderMaterial.new()
	poster_mat.shader = shader
	_replace_material_by_name(room_env, "poster_art", poster_mat)

func _replace_material_by_name(node: Node, target_name: String, new_mat: Material) -> void:
	if node is MeshInstance3D:
		var mi := node as MeshInstance3D
		if mi.mesh:
			for i in mi.mesh.get_surface_count():
				var mat := mi.get_active_material(i)
				if mat == null:
					mat = mi.mesh.surface_get_material(i)
				if mat and (mat.resource_name == target_name or target_name in mat.resource_name):
					mi.set_surface_override_material(i, new_mat)
	for child in node.get_children():
		_replace_material_by_name(child, target_name, new_mat)

func _process(delta: float) -> void:
	if _feed_timer > 0.0:
		_feed_timer -= delta
		if _feed_timer <= 0.0:
			feed_label.text = ""

	if _game_state != "playing":
		return

	# Slowly cool down gamer suspicion when he's focused on his game
	if not gamer_npc.is_looking_back():
		gamer_suspicion = maxf(0.0, gamer_suspicion - delta * 1.4)
	_update_hud()

	var charge: float = player_couch.get_charge_ratio()
	charge_bar.visible = charge > 0.01
	charge_bar.value = charge * 100.0
	if gamer_npc.is_looking_back():
		crosshair_dot.color = Color(1.0, 0.3, 0.3)
	elif gamer_npc.is_warning():
		crosshair_dot.color = Color(1.0, 0.65, 0.2)
	else:
		crosshair_dot.color = Color(1.0, 0.95, 0.35)

func _input(event: InputEvent) -> void:
	if _game_state == "playing" and event is InputEventScreenTouch:
		var st := event as InputEventScreenTouch
		if st.pressed and pause_button.visible and pause_button.get_global_rect().has_point(st.position):
			_set_paused(true)
			get_viewport().set_input_as_handled()

func _unhandled_input(event: InputEvent) -> void:
	if event.is_action_pressed("ui_cancel"):
		if _game_state == "playing":
			_set_paused(true)
		elif _game_state == "paused":
			_set_paused(false)
	elif event.is_action_pressed("restart_game") and (_game_state == "won" or _game_state == "lost" or _game_state == "paused"):
		start_new_game()

func _enable_touch_mode() -> void:
	if _is_touch_mode:
		return
	_is_touch_mode = true
	bottom_left_panel.offset_top = -170.0
	bottom_left_panel.offset_bottom = -126.0
	bottom_left_panel.offset_right = 240.0
	start_desc.text = "You're sitting on the sofa. Across the room is an open,\nrunning PC with a gamer totally absorbed in their game.\n\nThrow pencils into the exposed PC parts (CPU Cooler, GPU,\nRAM Sticks, PSU) to short-circuit and crash the system!\nDon't hit the gamer, and hold fire when he looks back.\n\nControls:\n• Drag screen — Aim\n• Hold & release THROW — Charge & throw pencil\n• < / > — Lean left/right on the sofa for a better angle"
	restart_button.text = "PLAY AGAIN"
	touch_controls.visible = (_game_state == "playing")
	pause_button.visible = (_game_state == "playing")
	_update_hud()

func _show_menu() -> void:
	_round_id += 1
	if _end_overlay_tween and _end_overlay_tween.is_valid():
		_end_overlay_tween.kill()
	_game_state = "menu"
	gamer_npc.set_active(false)
	player_couch.set_play_active(false)
	suspicion_alert.visible = false
	start_overlay.visible = true
	end_overlay.visible = false
	end_overlay.modulate.a = 1.0
	pause_overlay.visible = false
	touch_controls.visible = false
	pause_button.visible = false
	charge_bar.visible = false
	crosshair_dot.visible = false
	_update_hud()

func start_new_game() -> void:
	_round_id += 1
	if _end_overlay_tween and _end_overlay_tween.is_valid():
		_end_overlay_tween.kill()
	pc_health = 100.0
	gamer_suspicion = 0.0
	throws_count = 0
	hits_count = 0
	damaged_parts.clear()

	# Clean up loose pencils in scene
	for child in get_children():
		if child.has_method("launch"):
			child.queue_free()

	pc_target.reset_pc()
	gamer_npc.reset_gamer()
	gamer_npc.set_active(true)
	suspicion_alert.visible = false
	if _monitor_mat:
		_monitor_mat.set_shader_parameter("damage_level", 0.0)
		_monitor_mat.set_shader_parameter("is_bsod", 0.0)
		_monitor_mat.set_shader_parameter("power_flicker", 1.0)
		_monitor_mat.set_shader_parameter("hit_flash", 0.0)

	start_overlay.visible = false
	end_overlay.visible = false
	end_overlay.modulate.a = 1.0
	pause_overlay.visible = false
	_game_state = "playing"
	touch_controls.visible = _is_touch_mode
	pause_button.visible = _is_touch_mode
	crosshair_dot.visible = true
	player_couch.set_play_active(true)
	_show_feed("Goal: Disable the open PC!", Color(0.5, 0.9, 1.0))
	_update_hud()

func _set_paused(paused: bool) -> void:
	if paused:
		_game_state = "paused"
		gamer_npc.set_active(false)
		pause_overlay.visible = true
		touch_controls.visible = false
		pause_button.visible = false
		charge_bar.visible = false
		crosshair_dot.visible = false
		player_couch.set_play_active(false)
	else:
		_game_state = "playing"
		gamer_npc.set_active(true)
		pause_overlay.visible = false
		touch_controls.visible = _is_touch_mode
		pause_button.visible = _is_touch_mode
		crosshair_dot.visible = true
		player_couch.set_play_active(true)

func _on_pencil_thrown(proj: Node3D) -> void:
	throws_count += 1
	sfx.play("throw", -5.0)
	proj.hit_target.connect(_on_projectile_hit)

	# Throwing while the gamer is actively watching you is super risky!
	if gamer_npc.is_looking_back():
		_add_suspicion(38.0, "He saw you throw!")

	_update_hud()

func _on_projectile_hit(info: Dictionary) -> void:
	if _game_state != "playing":
		return
	var hit_type: String = info["type"]
	var hit_pos: Vector3 = info["position"]
	var collider: Object = info["collider"]

	match hit_type:
		"pc_component":
			hits_count += 1
			sfx.play("hit_electric", -2.0)
			pc_target.apply_hit(collider, hit_pos)
		"pc_armor":
			sfx.play("hit_metal", -3.0)
			pc_target.apply_hit(collider, hit_pos)
		"monitor":
			sfx.play("hit_metal", -4.0, 0.75)
			_flash_monitor_glitch()
			_add_suspicion(24.0, "Monitor hit!")
			gamer_npc.trigger_look_back(2.3)
		"gamer":
			sfx.play("gamer_hey", -1.5)
			_add_suspicion(40.0, "Ouch! You hit the gamer!")
			gamer_npc.trigger_look_back(2.8)
		"soft_prop", "room":
			sfx.play("hit_wood", -7.0)
			if hit_pos.z < -0.5:
				_add_suspicion(10.0, "")
				if randf() < 0.45:
					gamer_npc.trigger_look_back(1.5)

func _on_pc_component_damaged(_part_id: String, part_label: String, damage: float, _world_pos: Vector3, is_critical: bool) -> void:
	pc_health = maxf(0.0, pc_health - damage)
	damaged_parts[part_label] = true

	var dmg_norm := 1.0 - (pc_health / 100.0)
	gamer_npc.set_alertness(dmg_norm)
	if _monitor_mat:
		_monitor_mat.set_shader_parameter("damage_level", dmg_norm)

	if damage <= 5.0:
		_show_feed("%s already fried! (-%d%%)" % [part_label, int(damage)], Color(0.9, 0.65, 0.35))
	else:
		var prefix := "CRIT! " if is_critical else ""
		_show_feed("%s%s (-%d%%)" % [prefix, part_label, int(damage)], Color(1.0, 0.85, 0.25))

	# Internal hits distract the gamer more often as the PC malfunctions
	if pc_health > 0.0 and randf() < 0.55:
		_add_suspicion(12.0, "")
		gamer_npc.trigger_look_back(1.9)

	_update_hud()
	if pc_health <= 0.0:
		_trigger_victory()

func _on_pc_armor_ricochet(_world_pos: Vector3) -> void:
	_show_feed("Case ricochet! (0 dmg)", Color(0.85, 0.85, 0.9))
	_add_suspicion(14.0, "")
	if randf() < 0.70:
		gamer_npc.trigger_look_back(1.8)
	_update_hud()

func _flash_monitor_glitch() -> void:
	if not _monitor_mat:
		return
	var prev_dmg := 1.0 - (pc_health / 100.0)
	_monitor_mat.set_shader_parameter("damage_level", maxf(prev_dmg, 0.75))
	_monitor_mat.set_shader_parameter("hit_flash", 1.0)
	var tw := create_tween()
	tw.tween_method(func(v: float):
		if _monitor_mat:
			_monitor_mat.set_shader_parameter("hit_flash", v)
	, 1.0, 0.0, 0.40)
	get_tree().create_timer(0.35).timeout.connect(func():
		if _monitor_mat and _game_state == "playing":
			_monitor_mat.set_shader_parameter("damage_level", prev_dmg)
	)

func _add_suspicion(amount: float, reason: String) -> void:
	gamer_suspicion = minf(100.0, gamer_suspicion + amount)
	if reason != "":
		_show_feed(reason, Color(1.0, 0.38, 0.32))
	_update_hud()
	if gamer_suspicion >= 100.0:
		_trigger_defeat()

func _on_gamer_look_back_warning() -> void:
	suspicion_alert.text = "HE'S TURNING! HOLD FIRE!"
	suspicion_alert.add_theme_color_override("font_color", Color(1.0, 0.75, 0.2, 1.0))
	suspicion_alert.visible = true
	sfx.play("glance_warn", -6.0)

func _on_gamer_look_back() -> void:
	suspicion_alert.text = "HE'S LOOKING! DON'T THROW!"
	suspicion_alert.add_theme_color_override("font_color", Color(1.0, 0.35, 0.3, 1.0))
	suspicion_alert.visible = true

func _on_gamer_back_to_game() -> void:
	suspicion_alert.visible = false

func _trigger_victory() -> void:
	_game_state = "won"
	touch_controls.visible = false
	pause_button.visible = false
	charge_bar.visible = false
	crosshair_dot.visible = false
	player_couch.set_play_active(false)
	pc_target.set_broken_state()
	gamer_npc.set_defeat()
	suspicion_alert.visible = false
	if _monitor_mat:
		_monitor_mat.set_shader_parameter("is_bsod", 1.0)
		_monitor_mat.set_shader_parameter("damage_level", 0.35)
	sfx.play("victory", -2.0)
	_show_feed("BLUE SCREEN OF DEATH!", Color(0.35, 1.0, 0.55))

	end_title.text = "COMPUTER DISABLED!"
	end_title.modulate = Color(0.35, 1.0, 0.55)
	var accuracy := int((float(hits_count) / maxf(1.0, float(throws_count))) * 100.0)
	end_desc.text = "Blue Screen of Death! The gamer is defeated.\nPencils thrown: %d  |  Part accuracy: %d%%" % [throws_count, accuracy]
	_schedule_end_overlay(1.8)

func _trigger_defeat() -> void:
	_game_state = "lost"
	touch_controls.visible = false
	pause_button.visible = false
	charge_bar.visible = false
	crosshair_dot.visible = false
	player_couch.set_play_active(false)
	gamer_npc.set_active(false)
	gamer_npc.trigger_look_back(10.0)
	suspicion_alert.visible = false
	sfx.play("defeat", -2.0)
	_show_feed("BUSTED! He caught you!", Color(1.0, 0.35, 0.35))

	end_title.text = "BUSTED!"
	end_title.modulate = Color(1.0, 0.35, 0.35)
	end_desc.text = "The gamer got mad and confiscated your pencils.\nRemaining PC health: %d%%" % int(ceilf(pc_health))
	_schedule_end_overlay(1.5)

func _schedule_end_overlay(delay: float) -> void:
	var seq := _round_id
	get_tree().create_timer(delay).timeout.connect(func():
		if seq != _round_id or (_game_state != "won" and _game_state != "lost"):
			return
		end_overlay.modulate.a = 0.0
		end_overlay.visible = true
		if _end_overlay_tween and _end_overlay_tween.is_valid():
			_end_overlay_tween.kill()
		_end_overlay_tween = create_tween()
		_end_overlay_tween.tween_property(end_overlay, "modulate:a", 1.0, 0.3)
	)

func _show_feed(msg: String, col: Color) -> void:
	feed_label.text = msg
	feed_label.modulate = col
	_feed_timer = 2.0

func _update_hud() -> void:
	pc_bar.value = pc_health
	pc_pct_label.text = "PC Health: %d%%" % int(ceilf(pc_health))
	suspicion_bar.value = gamer_suspicion

	var parts_list := ["CPU Cooler", "GPU", "RAM Sticks", "PSU"]
	var lines := PackedStringArray()
	for p in parts_list:
		var mark := "[X]" if damaged_parts.has(p) else "[ ]"
		lines.append("%s %s" % [mark, p])
	parts_status_label.text = "\n".join(lines)

	if _is_touch_mode:
		stats_label.text = "Throws: %d  |  PC Hits: %d" % [throws_count, hits_count]
	else:
		stats_label.text = "Throws: %d  |  PC Hits: %d\n[A / D] Lean on sofa   [LMB] Charge & throw" % [throws_count, hits_count]

func _on_start_button_pressed() -> void:
	start_new_game()

func _on_restart_button_pressed() -> void:
	start_new_game()

func _on_resume_button_pressed() -> void:
	_set_paused(false)

func _on_pause_button_pressed() -> void:
	if _game_state == "playing":
		_set_paused(true)
```

### pencil_sabotage/scenes/pencil_projectile.tscn
```text
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/pencil_projectile.gd" id="1_proj"]

[node name="PencilProjectile" type="Node3D"]
script = ExtResource("1_proj")
```

### pencil_sabotage/scenes/main_scene.tscn
```text
[gd_scene load_steps=13 format=3]

[ext_resource type="Script" path="res://scripts/game_manager.gd" id="1_gm"]
[ext_resource type="Script" path="res://scripts/pc_target.gd" id="2_pc"]
[ext_resource type="Script" path="res://scripts/gamer_npc.gd" id="3_gamer"]
[ext_resource type="Script" path="res://scripts/player_couch.gd" id="4_player"]
[ext_resource type="Script" path="res://scripts/sound_fx.gd" id="5_sfx"]

[sub_resource type="Environment" id="Environment_room"]
background_mode = 1
background_color = Color(0.1, 0.11, 0.14, 1)
ambient_light_source = 2
ambient_light_color = Color(0.68, 0.7, 0.76, 1)
ambient_light_energy = 0.22
tonemap_mode = 2
tonemap_exposure = 0.9
glow_enabled = false

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_panel"]
bg_color = Color(0.08, 0.1, 0.14, 0.78)
corner_radius_top_left = 10
corner_radius_top_right = 10
corner_radius_bottom_right = 10
corner_radius_bottom_left = 10
content_margin_left = 14.0
content_margin_top = 10.0
content_margin_right = 14.0
content_margin_bottom = 10.0

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_fill_green"]
bg_color = Color(0.22, 0.82, 0.45, 1)
corner_radius_top_left = 4
corner_radius_top_right = 4
corner_radius_bottom_right = 4
corner_radius_bottom_left = 4

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_fill_red"]
bg_color = Color(0.94, 0.28, 0.25, 1)
corner_radius_top_left = 4
corner_radius_top_right = 4
corner_radius_bottom_right = 4
corner_radius_bottom_left = 4

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_touch_btn"]
bg_color = Color(0.08, 0.1, 0.14, 0.72)
border_width_left = 2
border_width_top = 2
border_width_right = 2
border_width_bottom = 2
border_color = Color(1, 1, 1, 0.28)
corner_radius_top_left = 16
corner_radius_top_right = 16
corner_radius_bottom_right = 16
corner_radius_bottom_left = 16

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_touch_round"]
bg_color = Color(0.14, 0.16, 0.2, 0.78)
border_width_left = 2
border_width_top = 2
border_width_right = 2
border_width_bottom = 2
border_color = Color(1, 0.85, 0.35, 0.5)
corner_radius_top_left = 58
corner_radius_top_right = 58
corner_radius_bottom_right = 58
corner_radius_bottom_left = 58

[node name="MainScene" type="Node3D"]
script = ExtResource("1_gm")

[node name="WorldEnvironment" type="WorldEnvironment" parent="."]
environment = SubResource("Environment_room")

[node name="CeilingLight" type="OmniLight3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 2.65, -0.35)
light_color = Color(1, 0.94, 0.85, 1)
light_energy = 0.85
shadow_enabled = false
omni_range = 8.5
omni_attenuation = 1.25

[node name="DeskKeyLight" type="SpotLight3D" parent="."]
transform = Transform3D(0.965926, -0.183013, 0.183013, 0, 0.707107, 0.707107, -0.258819, -0.683013, 0.683013, 0.9, 2.35, -0.6)
light_color = Color(0.9, 0.95, 1, 1)
light_energy = 1.05
shadow_enabled = true
shadow_bias = 0.02
shadow_blur = 2.0
spot_range = 5.5
spot_angle = 48.0

[node name="PCInteriorLight" type="OmniLight3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0.44, 1.05, -1.85)
light_color = Color(0.25, 0.85, 1, 1)
light_energy = 0.45
omni_range = 1.3

[node name="SoundFX" type="Node" parent="."]
script = ExtResource("5_sfx")

[node name="Monitor" type="Node3D" parent="."]
transform = Transform3D(-1, 0, -8.74228e-08, 0, 1, 0, 8.74228e-08, 0, -1, -0.14, 0.76, -2.28)

[node name="PCTower" type="Node3D" parent="."]
transform = Transform3D(0.173648, 0, 0.984808, 0, 1, 0, -0.984808, 0, 0.173648, 0.52, 0.76, -2.02)
script = ExtResource("2_pc")

[node name="GamerCharacter" type="Node3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, -0.05, 0, -1.48)
script = ExtResource("3_gamer")

[node name="PlayerCouch" type="Node3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1.18, 2.3)
script = ExtResource("4_player")

[node name="Camera3D" type="Camera3D" parent="PlayerCouch"]
fov = 62.0

[node name="HandPivot" type="Node3D" parent="PlayerCouch/Camera3D"]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0.24, -0.2, -0.38)

[node name="HUD" type="CanvasLayer" parent="."]

[node name="TopLeftPanel" type="PanelContainer" parent="HUD"]
offset_left = 20.0
offset_top = 20.0
offset_right = 310.0
offset_bottom = 175.0
theme_override_styles/panel = SubResource("StyleBoxFlat_panel")

[node name="VBox" type="VBoxContainer" parent="HUD/TopLeftPanel"]
layout_mode = 2
theme_override_constants/separation = 6

[node name="PCLabel" type="Label" parent="HUD/TopLeftPanel/VBox"]
layout_mode = 2
text = "PC Health: 100%"

[node name="PCBar" type="ProgressBar" parent="HUD/TopLeftPanel/VBox"]
custom_minimum_size = Vector2(0, 18)
layout_mode = 2
theme_override_styles/fill = SubResource("StyleBoxFlat_fill_green")
value = 100.0
show_percentage = false

[node name="PartsStatus" type="Label" parent="HUD/TopLeftPanel/VBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 13
text = "[ ] CPU Cooler
[ ] GPU
[ ] RAM Sticks
[ ] PSU"

[node name="TopRightPanel" type="PanelContainer" parent="HUD"]
anchors_preset = 1
anchor_left = 1.0
anchor_right = 1.0
offset_left = -310.0
offset_top = 20.0
offset_right = -20.0
offset_bottom = 120.0
grow_horizontal = 0
theme_override_styles/panel = SubResource("StyleBoxFlat_panel")

[node name="VBox" type="VBoxContainer" parent="HUD/TopRightPanel"]
layout_mode = 2
theme_override_constants/separation = 6

[node name="SuspicionLabel" type="Label" parent="HUD/TopRightPanel/VBox"]
layout_mode = 2
text = "Gamer Suspicion"

[node name="SuspicionBar" type="ProgressBar" parent="HUD/TopRightPanel/VBox"]
custom_minimum_size = Vector2(0, 18)
layout_mode = 2
theme_override_styles/fill = SubResource("StyleBoxFlat_fill_red")
show_percentage = false

[node name="AlertLabel" type="Label" parent="HUD/TopRightPanel/VBox"]
visible = false
layout_mode = 2
theme_override_colors/font_color = Color(1, 0.35, 0.3, 1)
theme_override_font_sizes/font_size = 14
text = "HE'S LOOKING! DON'T THROW!"

[node name="BottomLeftPanel" type="PanelContainer" parent="HUD"]
anchors_preset = 2
anchor_top = 1.0
anchor_bottom = 1.0
offset_left = 20.0
offset_top = -76.0
offset_right = 460.0
offset_bottom = -20.0
grow_vertical = 0
theme_override_styles/panel = SubResource("StyleBoxFlat_panel")

[node name="StatsLabel" type="Label" parent="HUD/BottomLeftPanel"]
layout_mode = 2
theme_override_font_sizes/font_size = 14
text = "Throws: 0  |  PC Hits: 0
[A / D] Lean on sofa   [LMB] Charge & throw"

[node name="FeedLabel" type="Label" parent="HUD"]
anchors_preset = 5
anchor_left = 0.5
anchor_right = 0.5
offset_left = -240.0
offset_top = 28.0
offset_right = 240.0
offset_bottom = 60.0
grow_horizontal = 2
theme_override_font_sizes/font_size = 20
horizontal_alignment = 1

[node name="CrosshairRoot" type="Control" parent="HUD"]
layout_mode = 3
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
grow_horizontal = 2
grow_vertical = 2
mouse_filter = 2

[node name="Dot" type="ColorRect" parent="HUD/CrosshairRoot"]
layout_mode = 0
offset_left = -3.0
offset_top = -3.0
offset_right = 3.0
offset_bottom = 3.0
mouse_filter = 2
color = Color(1, 0.95, 0.35, 1)

[node name="ChargeBar" type="ProgressBar" parent="HUD/CrosshairRoot"]
visible = false
layout_mode = 0
offset_left = -36.0
offset_top = 16.0
offset_right = 36.0
offset_bottom = 26.0
mouse_filter = 2
show_percentage = false

[node name="TouchControls" type="Control" parent="HUD"]
visible = false
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
mouse_filter = 2

[node name="LeanLeftBtn" type="PanelContainer" parent="HUD/TouchControls"]
layout_mode = 1
anchors_preset = 2
anchor_top = 1.0
anchor_bottom = 1.0
offset_left = 24.0
offset_top = -114.0
offset_right = 108.0
offset_bottom = -30.0
grow_vertical = 0
mouse_filter = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_touch_btn")

[node name="Label" type="Label" parent="HUD/TouchControls/LeanLeftBtn"]
layout_mode = 2
mouse_filter = 2
theme_override_font_sizes/font_size = 32
text = "<"
horizontal_alignment = 1
vertical_alignment = 1

[node name="LeanRightBtn" type="PanelContainer" parent="HUD/TouchControls"]
layout_mode = 1
anchors_preset = 2
anchor_top = 1.0
anchor_bottom = 1.0
offset_left = 122.0
offset_top = -114.0
offset_right = 206.0
offset_bottom = -30.0
grow_vertical = 0
mouse_filter = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_touch_btn")

[node name="Label" type="Label" parent="HUD/TouchControls/LeanRightBtn"]
layout_mode = 2
mouse_filter = 2
theme_override_font_sizes/font_size = 32
text = ">"
horizontal_alignment = 1
vertical_alignment = 1

[node name="ThrowBtn" type="PanelContainer" parent="HUD/TouchControls"]
layout_mode = 1
anchors_preset = 3
anchor_left = 1.0
anchor_top = 1.0
anchor_right = 1.0
anchor_bottom = 1.0
offset_left = -142.0
offset_top = -142.0
offset_right = -26.0
offset_bottom = -26.0
grow_horizontal = 0
grow_vertical = 0
mouse_filter = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_touch_round")

[node name="Label" type="Label" parent="HUD/TouchControls/ThrowBtn"]
layout_mode = 2
mouse_filter = 2
theme_override_font_sizes/font_size = 18
text = "THROW"
horizontal_alignment = 1
vertical_alignment = 1

[node name="PauseButton" type="Button" parent="HUD"]
visible = false
anchors_preset = 1
anchor_left = 1.0
anchor_right = 1.0
offset_left = -76.0
offset_top = 132.0
offset_right = -20.0
offset_bottom = 176.0
grow_horizontal = 0
theme_override_font_sizes/font_size = 18
theme_override_styles/normal = SubResource("StyleBoxFlat_touch_btn")
text = "||"

[node name="StartOverlay" type="ColorRect" parent="HUD"]
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
color = Color(0.04, 0.06, 0.1, 0.72)

[node name="Panel" type="PanelContainer" parent="HUD/StartOverlay"]
layout_mode = 1
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -280.0
offset_top = -170.0
offset_right = 280.0
offset_bottom = 170.0
grow_horizontal = 2
grow_vertical = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_panel")

[node name="VBox" type="VBoxContainer" parent="HUD/StartOverlay/Panel"]
layout_mode = 2
theme_override_constants/separation = 14
alignment = 1

[node name="Title" type="Label" parent="HUD/StartOverlay/Panel/VBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 28
text = "PENCIL SABOTAGE"
horizontal_alignment = 1

[node name="Desc" type="Label" parent="HUD/StartOverlay/Panel/VBox"]
layout_mode = 2
text = "You're sitting on the sofa. Across the room is an open,
running PC with a gamer totally absorbed in their game.

Throw pencils into the exposed PC parts (CPU Cooler, GPU,
RAM Sticks, PSU) to short-circuit and crash the system!
Don't hit the gamer, and hold fire when he looks back.

Controls:
• Mouse — Aim
• Hold & release LMB — Charge & throw pencil
• A / D — Lean left/right on the sofa for a better angle"
horizontal_alignment = 1

[node name="StartButton" type="Button" parent="HUD/StartOverlay/Panel/VBox"]
custom_minimum_size = Vector2(200, 44)
layout_mode = 2
size_flags_horizontal = 4
text = "START THROWING"

[node name="EndOverlay" type="ColorRect" parent="HUD"]
visible = false
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
color = Color(0.04, 0.05, 0.08, 0.75)

[node name="Panel" type="PanelContainer" parent="HUD/EndOverlay"]
layout_mode = 1
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -260.0
offset_top = -125.0
offset_right = 260.0
offset_bottom = 125.0
grow_horizontal = 2
grow_vertical = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_panel")

[node name="VBox" type="VBoxContainer" parent="HUD/EndOverlay/Panel"]
layout_mode = 2
theme_override_constants/separation = 14
alignment = 1

[node name="EndTitle" type="Label" parent="HUD/EndOverlay/Panel/VBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 24
text = "RESULT"
horizontal_alignment = 1

[node name="EndDesc" type="Label" parent="HUD/EndOverlay/Panel/VBox"]
layout_mode = 2
text = "..."
horizontal_alignment = 1

[node name="RestartButton" type="Button" parent="HUD/EndOverlay/Panel/VBox"]
custom_minimum_size = Vector2(200, 44)
layout_mode = 2
size_flags_horizontal = 4
text = "PLAY AGAIN [R]"

[node name="PauseOverlay" type="ColorRect" parent="HUD"]
visible = false
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
color = Color(0.04, 0.05, 0.08, 0.65)

[node name="Panel" type="PanelContainer" parent="HUD/PauseOverlay"]
layout_mode = 1
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -180.0
offset_top = -100.0
offset_right = 180.0
offset_bottom = 100.0
grow_horizontal = 2
grow_vertical = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_panel")

[node name="VBox" type="VBoxContainer" parent="HUD/PauseOverlay/Panel"]
layout_mode = 2
theme_override_constants/separation = 12
alignment = 1

[node name="Label" type="Label" parent="HUD/PauseOverlay/Panel/VBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 22
text = "PAUSED"
horizontal_alignment = 1

[node name="ResumeButton" type="Button" parent="HUD/PauseOverlay/Panel/VBox"]
custom_minimum_size = Vector2(180, 40)
layout_mode = 2
size_flags_horizontal = 4
text = "RESUME"

[node name="RestartButton" type="Button" parent="HUD/PauseOverlay/Panel/VBox"]
custom_minimum_size = Vector2(180, 40)
layout_mode = 2
size_flags_horizontal = 4
text = "RESTART"

[connection signal="pressed" from="HUD/StartOverlay/Panel/VBox/StartButton" to="." method="_on_start_button_pressed"]
[connection signal="pressed" from="HUD/EndOverlay/Panel/VBox/RestartButton" to="." method="_on_restart_button_pressed"]
[connection signal="pressed" from="HUD/PauseOverlay/Panel/VBox/ResumeButton" to="." method="_on_resume_button_pressed"]
[connection signal="pressed" from="HUD/PauseOverlay/Panel/VBox/RestartButton" to="." method="_on_restart_button_pressed"]
[connection signal="pressed" from="HUD/PauseButton" to="." method="_on_pause_button_pressed"]
```

### pencil_sabotage/shaders/rug_pattern.gdshader
```gdshader
shader_type spatial;

varying vec2 rug_uv;
varying vec2 rug_meters;

void vertex() {
	vec3 world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
	// OpenSCAD rug centered at (0, -0.1) -> Godot world (0, +0.1), size 3.18m x 3.58m
	rug_meters = vec2(world_pos.x, world_pos.z - 0.10);
	rug_uv = vec2((rug_meters.x + 1.59) / 3.18, (rug_meters.y + 1.79) / 3.58);
}

void fragment() {
	vec3 cream = vec3(0.66, 0.61, 0.54);
	vec3 cream_dark = vec3(0.58, 0.53, 0.46);
	vec3 slate_blue = vec3(0.26, 0.36, 0.46);
	vec3 terracotta = vec3(0.74, 0.38, 0.24);
	vec3 mustard = vec3(0.80, 0.62, 0.28);

	// Subtle woven wool texture
	float weave_x = sin(rug_meters.x * 240.0);
	float weave_y = sin(rug_meters.y * 240.0);
	float weave = 0.5 + 0.5 * (weave_x * weave_y);
	vec3 base_col = mix(cream_dark, cream, 0.65 + 0.35 * weave);

	vec3 col = base_col;

	// Distance from outer edge in meters
	float edge_x = 1.59 - abs(rug_meters.x);
	float edge_y = 1.79 - abs(rug_meters.y);
	float edge_d = min(edge_x, edge_y);

	// Outer slate-blue frame band
	float outer_band = smoothstep(0.045, 0.050, edge_d) * (1.0 - smoothstep(0.105, 0.110, edge_d));
	col = mix(col, slate_blue, outer_band);

	// Inner terracotta pinstripe
	float inner_stripe = smoothstep(0.145, 0.150, edge_d) * (1.0 - smoothstep(0.175, 0.180, edge_d));
	col = mix(col, terracotta, inner_stripe);

	// Exact integer grid: 5 columns (2.60m) x 6 rows (3.12m) of 0.52m cells -> zero clipped diamonds!
	// Odd count on X (5) centers a column at X=0; even count on Y (6) shifts Y by half-cell
	if (abs(rug_meters.x) < 1.30 && abs(rug_meters.y) < 1.56) {
		float cell_len = 0.52;
		vec2 grid_pos = vec2(rug_meters.x + cell_len * 0.5, rug_meters.y);
		vec2 cell = fract(grid_pos / cell_len) - 0.5;
		float manhattan = abs(cell.x) + abs(cell.y);

		// Outer slate-blue diamond outline
		float d_outer = smoothstep(0.31, 0.32, manhattan) * (1.0 - smoothstep(0.40, 0.41, manhattan));
		col = mix(col, slate_blue, d_outer);

		// Middle terracotta diamond
		float d_mid = smoothstep(0.16, 0.17, manhattan) * (1.0 - smoothstep(0.24, 0.25, manhattan));
		col = mix(col, terracotta, d_mid);

		// Center mustard medallion dot
		float d_center = 1.0 - smoothstep(0.07, 0.085, manhattan);
		col = mix(col, mustard, d_center);

		// Corner accent diamonds only strictly between interior cells (never clipped at borders)
		if (abs(rug_meters.x) < 1.18 && abs(rug_meters.y) < 1.44) {
			float corner_m = abs(abs(cell.x) - 0.5) + abs(abs(cell.y) - 0.5);
			float d_corner = 1.0 - smoothstep(0.095, 0.11, corner_m);
			col = mix(col, terracotta, d_corner);
		}
	}

	// Micro fabric grain modulation across all pattern colors
	col *= 0.94 + 0.08 * weave;

	ALBEDO = col;
	ROUGHNESS = 0.94;
	METALLIC = 0.0;
}
```

### pencil_sabotage/shaders/wood_floor.gdshader
```gdshader
shader_type spatial;

varying vec2 floor_pos;

float hash21(vec2 p) {
	p = fract(p * vec2(123.34, 456.21));
	p += dot(p, p + 45.32);
	return fract(p.x * p.y);
}

float noise1d(float x) {
	float i = floor(x);
	float f = fract(x);
	f = f * f * (3.0 - 2.0 * f);
	return mix(hash21(vec2(i, 7.13)), hash21(vec2(i + 1.0, 7.13)), f);
}

float value_noise(vec2 p) {
	vec2 i = floor(p);
	vec2 f = fract(p);
	f = f * f * (3.0 - 2.0 * f);
	float a = hash21(i);
	float b = hash21(i + vec2(1.0, 0.0));
	float c = hash21(i + vec2(0.0, 1.0));
	float d = hash21(i + vec2(1.0, 1.0));
	return mix(mix(a, b, f.x), mix(c, d, f.x), f.y);
}

void vertex() {
	vec3 world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
	floor_pos = world_pos.xz;
}

void fragment() {
	// Plank dimensions in meters (0.18m wide x 1.15m long, running along Z)
	float plank_w = 0.18;
	float plank_l = 1.15;

	float col_idx = floor(floor_pos.x / plank_w);
	float row_shift = fract(hash21(vec2(col_idx, 3.7)) * 2.3) * plank_l;
	float z_shifted = floor_pos.y + row_shift;
	float row_idx = floor(z_shifted / plank_l);

	vec2 plank_id = vec2(col_idx, row_idx);
	float rnd = hash21(plank_id);
	float rnd2 = hash21(plank_id + vec2(19.3, 5.1));

	// Local coordinates within single plank [0..1]
	float lx = fract(floor_pos.x / plank_w);
	float lz = fract(z_shifted / plank_l);

	// Subtle organic wood grain stretched along plank length
	vec2 grain_uv = vec2(floor_pos.x * 38.0 + rnd * 17.0, z_shifted * 3.2 + rnd2 * 11.0);
	float warp = value_noise(grain_uv * vec2(0.35, 0.8)) * 2.4;
	float rings = sin((lx * 14.0 + warp + rnd * 6.28) * 3.14159);
	float fine_fiber = value_noise(vec2(floor_pos.x * 95.0, z_shifted * 14.0 + rnd * 9.0));
	float grain = 0.5 + 0.30 * rings + 0.20 * (fine_fiber - 0.5);

	// Warm oak palette with per-plank tone variation
	vec3 oak_light = vec3(0.58, 0.42, 0.26);
	vec3 oak_mid = vec3(0.50, 0.35, 0.21);
	vec3 oak_dark = vec3(0.41, 0.27, 0.16);
	vec3 plank_tint = mix(oak_dark, oak_light, 0.25 + 0.65 * rnd);
	vec3 wood_col = mix(plank_tint * 0.86, mix(plank_tint, oak_mid, 0.35), clamp(grain, 0.0, 1.0));

	// Plank bevel seams along X (width) and Z (length)
	float seam_x = min(lx, 1.0 - lx) * plank_w;
	float seam_z = min(lz, 1.0 - lz) * plank_l;
	float seam_mask_x = smoothstep(0.002, 0.0065, seam_x);
	float seam_mask_z = smoothstep(0.002, 0.0065, seam_z);
	float seam = seam_mask_x * seam_mask_z;

	vec3 groove_col = vec3(0.22, 0.14, 0.08);
	vec3 final_col = mix(groove_col, wood_col, seam);

	ALBEDO = final_col;
	ROUGHNESS = mix(0.92, 0.56 + 0.10 * (1.0 - grain), seam);
	METALLIC = 0.0;
}
```

### pencil_sabotage/shaders/wall_texture.gdshader
```gdshader
shader_type spatial;

varying vec3 world_pos;
varying vec3 world_nrm;

float hash21(vec2 p) {
	p = fract(p * vec2(234.34, 435.345));
	p += dot(p, p + 34.23);
	return fract(p.x * p.y);
}

float value_noise(vec2 p) {
	vec2 i = floor(p);
	vec2 f = fract(p);
	f = f * f * (3.0 - 2.0 * f);
	float a = hash21(i);
	float b = hash21(i + vec2(1.0, 0.0));
	float c = hash21(i + vec2(0.0, 1.0));
	float d = hash21(i + vec2(1.0, 1.0));
	return mix(mix(a, b, f.x), mix(c, d, f.x), f.y);
}

float fbm(vec2 p) {
	float v = 0.0;
	float amp = 0.5;
	for (int i = 0; i < 3; i++) {
		v += amp * value_noise(p);
		p *= 2.07;
		amp *= 0.5;
	}
	return v;
}

void vertex() {
	world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
	world_nrm = abs((MODEL_MATRIX * vec4(NORMAL, 0.0)).xyz);
}

void fragment() {
	float horiz = (world_nrm.x > world_nrm.z) ? world_pos.z : world_pos.x;
	float vert = world_pos.y;
	vec2 wall_uv = vec2(horiz, vert);

	// Soft matte plaster base
	float plaster = fbm(wall_uv * 2.5);
	float fine_grain = value_noise(wall_uv * 36.0);

	vec3 base_col = vec3(0.57, 0.56, 0.52);
	vec3 wave_fill = vec3(0.53, 0.53, 0.50);
	vec3 line_dark = vec3(0.43, 0.44, 0.42);
	vec3 line_light = vec3(0.69, 0.67, 0.62);

	// Organic flowing curved ribbon waves
	float phase_a = horiz * 2.4 + sin(vert * 2.2 + cos(horiz * 1.1) * 0.7) * 0.42;
	float band_a = smoothstep(-0.25, 0.25, sin(phase_a * 3.14159));
	vec3 col = mix(base_col, wave_fill, band_a * 0.75);

	// Crisp anti-aliased primary curved contour lines
	float d1 = abs(fract(phase_a * 0.85) - 0.5);
	float line_a = 1.0 - smoothstep(0.016, 0.042, d1);
	col = mix(col, line_dark, line_a * 0.65);

	// Secondary delicate counter-flowing curved cream lines
	float phase_b = horiz * 1.6 - cos(vert * 1.75 + sin(horiz * 0.9) * 0.6) * 0.36;
	float d2 = abs(fract(phase_b * 0.75 + 0.25) - 0.5);
	float line_b = 1.0 - smoothstep(0.012, 0.032, d2);
	col = mix(col, line_light, line_b * 0.55);

	// Subtle plaster mottling and fine tactile grain
	col *= 0.94 + 0.09 * plaster + 0.035 * (fine_grain - 0.5);

	// Gentle ambient darkening near ceiling edge
	float ceiling_ao = smoothstep(3.22, 2.95, vert);
	col *= mix(0.88, 1.0, ceiling_ao);

	ALBEDO = col;
	ROUGHNESS = mix(0.92, 0.70, max(line_a, line_b) * 0.6);
	METALLIC = 0.0;
}
```

### pencil_sabotage/shaders/poster_art.gdshader
```gdshader
shader_type spatial;

varying vec2 poster_uv;

float hash21(vec2 p) {
	p = fract(p * vec2(127.1, 311.7));
	p += dot(p, p + 19.19);
	return fract(p.x * p.y);
}

float value_noise(vec2 p) {
	vec2 i = floor(p);
	vec2 f = fract(p);
	f = f * f * (3.0 - 2.0 * f);
	float a = hash21(i);
	float b = hash21(i + vec2(1.0, 0.0));
	float c = hash21(i + vec2(0.0, 1.0));
	float d = hash21(i + vec2(1.0, 1.0));
	return mix(mix(a, b, f.x), mix(c, d, f.x), f.y);
}

float fbm(vec2 p) {
	float v = 0.0;
	float amp = 0.5;
	for (int i = 0; i < 4; i++) {
		v += amp * value_noise(p);
		p *= 2.05;
		amp *= 0.5;
	}
	return v;
}

float seg_dist(vec2 p, vec2 a, vec2 b) {
	vec2 pa = p - a;
	vec2 ba = b - a;
	float h = clamp(dot(pa, ba) / dot(ba, ba), 0.0, 1.0);
	return length(pa - ba * h);
}

void vertex() {
	vec3 world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
	// OpenSCAD poster_art centered at (-1.35, 2.99, 1.78), size 0.66m W x 0.88m H
	poster_uv = vec2(
		(world_pos.x + 1.35) / 0.66 + 0.5,
		(world_pos.y - 1.78) / 0.88 + 0.5
	);
}

void fragment() {
	vec2 uv = clamp(poster_uv, 0.0, 1.0);
	vec3 mat_col = vec3(0.91, 0.89, 0.84);

	float in_x = step(0.085, uv.x) * step(uv.x, 0.915);
	float in_y = step(0.070, uv.y) * step(uv.y, 0.930);
	float in_art = in_x * in_y;

	vec2 a = vec2(
		(uv.x - 0.085) / 0.830,
		(uv.y - 0.070) / 0.860
	);
	// Aspect-corrected space coordinates centered at (0, 0)
	vec2 p = vec2((a.x - 0.5) * 0.72, a.y - 0.5);

	// Deep cosmos background + soft colorful nebula clouds
	vec3 space_deep = vec3(0.03, 0.04, 0.11);
	vec3 space_mid = vec3(0.07, 0.11, 0.24);
	float neb1 = fbm(p * 4.5 + vec2(1.4, -0.8));
	float neb2 = fbm(p * 6.0 + vec2(-2.1, 2.5));

	vec3 art = mix(space_deep, space_mid, smoothstep(-0.45, 0.45, p.y * 0.7 - p.x * 0.4));
	art = mix(art, vec3(0.32, 0.14, 0.42), smoothstep(0.42, 0.82, neb1) * 0.55);
	art = mix(art, vec3(0.10, 0.42, 0.52), smoothstep(0.46, 0.84, neb2) * 0.45);

	// Crisp starfield
	vec2 star_grid = floor(a * vec2(26.0, 34.0));
	vec2 star_cell = fract(a * vec2(26.0, 34.0)) - 0.5;
	float sr = hash21(star_grid);
	if (sr > 0.88) {
		vec2 j = vec2(hash21(star_grid + 3.1), hash21(star_grid + 7.9)) - 0.5;
		float sd = length(star_cell - j * 0.45);
		float star = 1.0 - smoothstep(0.04, 0.13, sd);
		vec3 star_col = (sr > 0.96) ? vec3(1.0, 0.90, 0.68) : vec3(0.90, 0.95, 1.0);
		art = mix(art, star_col, star * 0.9);
	}

	// Prominent 4-point sparkle stars
	vec2 sp1 = p - vec2(-0.22, 0.31);
	float spark1 = max(
		(1.0 - smoothstep(0.0, 0.005, abs(sp1.x))) * (1.0 - smoothstep(0.0, 0.045, abs(sp1.y))),
		(1.0 - smoothstep(0.0, 0.005, abs(sp1.y))) * (1.0 - smoothstep(0.0, 0.045, abs(sp1.x)))
	);
	art = mix(art, vec3(1.0, 0.95, 0.80), spark1);

	vec2 sp2 = p - vec2(0.24, -0.26);
	float spark2 = max(
		(1.0 - smoothstep(0.0, 0.004, abs(sp2.x))) * (1.0 - smoothstep(0.0, 0.032, abs(sp2.y))),
		(1.0 - smoothstep(0.0, 0.004, abs(sp2.y))) * (1.0 - smoothstep(0.0, 0.032, abs(sp2.x)))
	);
	art = mix(art, vec3(0.75, 0.92, 1.0), spark2);

	// Small teal-ice moon in upper-left
	vec2 moon_c = vec2(-0.18, 0.17);
	float moon_d = length(p - moon_c);
	if (moon_d < 0.056) {
		float moon_edge = 1.0 - smoothstep(0.051, 0.056, moon_d);
		float moon_lit = smoothstep(-0.04, 0.04, (p.x - moon_c.x) * 0.7 + (p.y - moon_c.y) * 0.7);
		vec3 moon_col = mix(vec3(0.09, 0.14, 0.25), vec3(0.55, 0.84, 0.94), moon_lit);
		art = mix(art, moon_col, moon_edge);
	}

	// Shooting comet streaking across upper right
	vec2 comet_head = vec2(0.22, 0.31);
	vec2 comet_tail = vec2(0.04, 0.22);
	float cd = seg_dist(p, comet_head, comet_tail);
	float tail_t = clamp(dot(p - comet_tail, comet_head - comet_tail) / dot(comet_head - comet_tail, comet_head - comet_tail), 0.0, 1.0);
	float tail_glow = (1.0 - smoothstep(0.0, 0.012 * (0.2 + tail_t), cd)) * tail_t;
	art = mix(art, vec3(0.55, 0.88, 1.0), tail_glow * 0.75);
	art = mix(art, vec3(1.0, 0.96, 0.82), 1.0 - smoothstep(0.008, 0.014, length(p - comet_head)));

	// Large ringed gas giant planet (center-right)
	vec2 pl_c = vec2(0.05, 0.02);
	vec2 pp = p - pl_c;
	float pl_r = 0.145;
	float pl_d = length(pp);

	// Tilted planetary ring coordinates
	float rang = -0.38;
	vec2 rp = vec2(pp.x * cos(rang) - pp.y * sin(rang), pp.x * sin(rang) + pp.y * cos(rang));
	float ring_rad = length(vec2(rp.x * 0.44, rp.y * 1.95));
	float ring_band = smoothstep(0.095, 0.102, ring_rad) * (1.0 - smoothstep(0.142, 0.149, ring_rad));
	float ring_gap = 1.0 - smoothstep(0.003, 0.007, abs(ring_rad - 0.122));
	float ring_mask = ring_band * (1.0 - ring_gap * 0.75);
	vec3 ring_col = mix(vec3(0.86, 0.74, 0.56), vec3(0.62, 0.48, 0.42), smoothstep(0.098, 0.145, ring_rad));

	// Back half of ring (behind planet)
	if (rp.y > 0.0) {
		art = mix(art, ring_col, ring_mask * 0.82);
	}

	// Planet sphere with atmospheric bands & sunlit crescent
	if (pl_d < pl_r) {
		float pl_edge = 1.0 - smoothstep(pl_r - 0.005, pl_r, pl_d);
		float bands = 0.5 + 0.5 * sin(rp.y * 55.0 + fbm(pp * 18.0) * 2.5);
		vec3 pl_warm = mix(vec3(0.86, 0.42, 0.25), vec3(0.95, 0.72, 0.36), bands);
		float light = smoothstep(-0.10, 0.10, pp.x * 0.65 + pp.y * 0.75);
		vec3 pl_shaded = mix(vec3(0.11, 0.10, 0.20), pl_warm, 0.15 + 0.85 * light);
		// Rim atmospheric glow on sunlit side
		float rim = smoothstep(pl_r * 0.78, pl_r, pl_d) * light;
		pl_shaded = mix(pl_shaded, vec3(1.0, 0.85, 0.55), rim * 0.45);
		art = mix(art, pl_shaded, pl_edge);
	}

	// Front half of ring (in front of planet)
	if (rp.y <= 0.0) {
		art = mix(art, ring_col, ring_mask * 0.90);
	}

	// Minimalist constellation lines in lower left
	vec2 c1 = vec2(-0.24, -0.18);
	vec2 c2 = vec2(-0.14, -0.25);
	vec2 c3 = vec2(-0.20, -0.34);
	vec2 c4 = vec2(-0.07, -0.33);
	float cline = min(min(seg_dist(p, c1, c2), seg_dist(p, c2, c3)), seg_dist(p, c2, c4));
	art = mix(art, vec3(0.45, 0.72, 0.92), (1.0 - smoothstep(0.0015, 0.004, cline)) * 0.55);
	float cnode = min(min(length(p - c1), length(p - c2)), min(length(p - c3), length(p - c4)));
	art = mix(art, vec3(0.95, 0.94, 0.85), 1.0 - smoothstep(0.006, 0.010, cnode));

	// Retro rocket ascending in lower right
	vec2 rk = p - vec2(0.14, -0.23);
	float rk_ang = -0.45;
	vec2 rkp = vec2(rk.x * cos(rk_ang) - rk.y * sin(rk_ang), rk.x * sin(rk_ang) + rk.y * cos(rk_ang));
	// Rocket exhaust flame
	if (rkp.y < -0.04 && rkp.y > -0.14 && abs(rkp.x) < 0.018 * (1.0 + (rkp.y + 0.04) * 8.0)) {
		art = mix(art, vec3(0.98, 0.65, 0.22), 0.9);
	}
	// Rocket body capsule
	float rk_body = step(abs(rkp.x), 0.018 * (1.0 - max(0.0, rkp.y) * 9.0)) * step(-0.042, rkp.y) * step(rkp.y, 0.065);
	art = mix(art, vec3(0.92, 0.91, 0.88), rk_body);
	// Rocket red nose & fins
	if (rkp.y > 0.032 && rk_body > 0.5) {
		art = vec3(0.86, 0.28, 0.24);
	}
	// Porthole window
	if (length(rkp - vec2(0.0, 0.008)) < 0.008) {
		art = vec3(0.28, 0.78, 0.95);
	}

	// Subtle inner mat bevel shadow
	float inner_edge = min(min(a.x, 1.0 - a.x), min(a.y, 1.0 - a.y));
	float bevel = 1.0 - smoothstep(0.0, 0.012, inner_edge);
	art = mix(art, vec3(0.12, 0.12, 0.14), bevel * 0.4);

	// Fine matte art-paper grain
	float paper = 0.97 + 0.05 * hash21(floor(uv * 240.0));
	vec3 final_col = mix(mat_col, art, in_art) * paper;

	ALBEDO = final_col;
	ROUGHNESS = 0.84;
	METALLIC = 0.0;
}
```
