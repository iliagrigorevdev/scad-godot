### space_invaders/project.godot

```ini
config_version=5

[application]
config/name="Space Invaders 3D"
config/description="Classic Space Invaders reimagined with procedural 3D OpenSCAD assets and Godot 4"
run/main_scene="res://scenes/main_game.tscn"
config/features=PackedStringArray("4.7", "Forward Plus")

[display]
window/size/viewport_width=1280
window/size/viewport_height=720
window/stretch/mode="canvas_items"
window/stretch/aspect="keep"
window/handheld/orientation="landscape"

[input_devices]
pointing/emulate_touch_from_mouse=true
pointing/emulate_mouse_from_touch=true

[editor_plugins]
enabled=PackedStringArray("res://addons/scad_importer/plugin.cfg")
```

### space_invaders/addons/scad_importer/plugin.cfg

```text
[plugin]

name="OpenSCAD GLTF Importer"
description="Imports .scad files directly as 3D scenes using scad-gltf"
author="scad-gltf"
version="0.2"
script="scad_plugin.gd"

```

### space_invaders/addons/scad_importer/scad_importer.gd

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

### space_invaders/addons/scad_importer/scad_plugin.gd

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

### space_invaders/assets/models/player_ship.scad

```openscad
$fn = 24;

// Player Ship: Sleek retro-futuristic arcade cannon
union() {
    // Main Armored Hull
    color([0.22, 0.28, 0.38], metalness=0.8, roughness=0.25) {
        translate([0, 0, 0.2])
            cube([1.4, 1.3, 0.45], center=true);
        // Sloped Nose
        translate([0, 0.75, 0.15])
            cylinder(h=0.35, r1=0.65, r2=0.25, center=true);
        // Center Cannon Housing
        translate([0, 0.3, 0.45])
            cube([0.5, 0.8, 0.3], center=true);
    }

    // Side Wings & Shield Flanks
    color([0.18, 0.24, 0.32], metalness=0.85, roughness=0.2) {
        translate([-1.05, -0.15, 0.18])
            cube([0.8, 1.1, 0.26], center=true);
        translate([1.05, -0.15, 0.18])
            cube([0.8, 1.1, 0.26], center=true);
        // Wing Stabilizers
        translate([-1.4, -0.4, 0.4])
            cube([0.15, 0.5, 0.4], center=true);
        translate([1.4, -0.4, 0.4])
            cube([0.15, 0.5, 0.4], center=true);
    }

    // Laser Cannon Barrels
    color([0.7, 0.75, 0.85], metalness=0.9, roughness=0.15) {
        translate([0, 1.05, 0.35])
            rotate([-90, 0, 0])
                cylinder(h=0.9, r=0.1, center=true);
        translate([-0.65, 0.85, 0.25])
            rotate([-90, 0, 0])
                cylinder(h=0.75, r=0.07, center=true);
        translate([0.65, 0.85, 0.25])
            rotate([-90, 0, 0])
                cylinder(h=0.75, r=0.07, center=true);
    }

    // Muzzle Plasma Emitters (Glows Cyan)
    color([0.1, 0.9, 1.0], alpha=0.95, metalness=0.1, roughness=0.1, emissive=[0.1, 0.9, 1.0], emissiveIntensity=3.0) {
        translate([0, 1.5, 0.35]) sphere(r=0.11);
        translate([-0.65, 1.25, 0.25]) sphere(r=0.08);
        translate([0.65, 1.25, 0.25]) sphere(r=0.08);
    }

    // Cockpit Canopy (Glows Cyan)
    color([0.05, 0.8, 0.95], alpha=0.9, metalness=0.2, roughness=0.1, emissive=[0.05, 0.8, 0.95], emissiveIntensity=2.2) {
        translate([0, 0.1, 0.52])
            scale([1, 1.5, 0.75])
                sphere(r=0.36);
    }

    // Dual Rear Thruster Exhausts (Glows Plasma Amber)
    color([1.0, 0.55, 0.1], metalness=0.1, roughness=0.4, emissive=[1.0, 0.55, 0.1], emissiveIntensity=3.5) {
        translate([-0.5, -0.72, 0.2])
            rotate([90, 0, 0])
                cylinder(h=0.22, r=0.18, center=true);
        translate([0.5, -0.72, 0.2])
            rotate([90, 0, 0])
                cylinder(h=0.22, r=0.18, center=true);
    }
}
```

### space_invaders/assets/models/invader_squid.scad

```openscad
$fn = 20;

// Invader Squid (Top Row - Red/Magenta)
anim_data = [
  ["march", [
    ["TentacleLeft", [
      [0.0, [0, 0, -15], [-0.35, 0, -0.35]],
      [0.5, [0, 0, 20], [-0.35, 0, -0.35]],
      [1.0, [0, 0, -15], [-0.35, 0, -0.35]]
    ]],
    ["TentacleRight", [
      [0.0, [0, 0, 15], [0.35, 0, -0.35]],
      [0.5, [0, 0, -20], [0.35, 0, -0.35]],
      [1.0, [0, 0, 15], [0.35, 0, -0.35]]
    ]],
    ["AntennaLeft", [
      [0.0, [0, -10, 0], [-0.25, 0, 0.45]],
      [0.5, [0, 15, 0], [-0.25, 0, 0.45]],
      [1.0, [0, -10, 0], [-0.25, 0, 0.45]]
    ]],
    ["AntennaRight", [
      [0.0, [0, 10, 0], [0.25, 0, 0.45]],
      [0.5, [0, -15, 0], [0.25, 0, 0.45]],
      [1.0, [0, 10, 0], [0.25, 0, 0.45]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="Body", t=[0, 0, 0], r=[0, 0, 0]) {
    // Main Head - glowing retro magenta
    color([0.95, 0.15, 0.45], metalness=0.3, roughness=0.3, emissive=[0.7, 0.1, 0.3], emissiveIntensity=1.5) {
      cube([0.75, 0.55, 0.65], center=true);
    }
    // Distinct Glowing Eyes
    color([1.0, 1.0, 0.2], metalness=0.1, roughness=0.1, emissive=[1.0, 1.0, 0.2], emissiveIntensity=2.6) {
      translate([-0.22, 0.29, 0.1]) cube([0.16, 0.05, 0.16], center=true);
      translate([0.22, 0.29, 0.1]) cube([0.16, 0.05, 0.16], center=true);
    }

    // Left Tentacle
    bone(name="TentacleLeft", t=[-0.35, 0, -0.35], r=[0, 0, 0]) {
      color([0.9, 0.1, 0.4], metalness=0.3, roughness=0.3, emissive=[0.5, 0.05, 0.2], emissiveIntensity=1.2) {
        translate([0, 0, -0.18]) cube([0.18, 0.35, 0.45], center=true);
      }
    }

    // Right Tentacle
    bone(name="TentacleRight", t=[0.35, 0, -0.35], r=[0, 0, 0]) {
      color([0.9, 0.1, 0.4], metalness=0.3, roughness=0.3, emissive=[0.5, 0.05, 0.2], emissiveIntensity=1.2) {
        translate([0, 0, -0.18]) cube([0.18, 0.35, 0.45], center=true);
      }
    }

    // Left Antenna
    bone(name="AntennaLeft", t=[-0.25, 0, 0.45], r=[0, 0, 0]) {
      color([1.0, 0.2, 0.5], metalness=0.2, roughness=0.4, emissive=[0.6, 0.1, 0.3], emissiveIntensity=1.2) {
        translate([0, 0, 0.18]) cube([0.1, 0.15, 0.35], center=true);
      }
    }

    // Right Antenna
    bone(name="AntennaRight", t=[0.25, 0, 0.45], r=[0, 0, 0]) {
      color([1.0, 0.2, 0.5], metalness=0.2, roughness=0.4, emissive=[0.6, 0.1, 0.3], emissiveIntensity=1.2) {
        translate([0, 0, 0.18]) cube([0.1, 0.15, 0.35], center=true);
      }
    }
  }
}
```

### space_invaders/assets/models/invader_crab.scad

```openscad
$fn = 20;

// Invader Crab (Middle Row - Amber/Yellow)
anim_data = [
  ["march", [
    ["ClawLeft", [
      [0.0, [0, 0, 0], [-0.62, 0, 0.1]],
      [0.5, [0, 0, -25], [-0.62, 0, 0.25]],
      [1.0, [0, 0, 0], [-0.62, 0, 0.1]]
    ]],
    ["ClawRight", [
      [0.0, [0, 0, 0], [0.62, 0, 0.1]],
      [0.5, [0, 0, 25], [0.62, 0, 0.25]],
      [1.0, [0, 0, 0], [0.62, 0, 0.1]]
    ]],
    ["LegLeft", [
      [0.0, [0, 0, 0], [-0.35, 0, -0.35]],
      [0.5, [0, 0, 18], [-0.42, 0, -0.28]],
      [1.0, [0, 0, 0], [-0.35, 0, -0.35]]
    ]],
    ["LegRight", [
      [0.0, [0, 0, 0], [0.35, 0, -0.35]],
      [0.5, [0, 0, -18], [0.42, 0, -0.28]],
      [1.0, [0, 0, 0], [0.35, 0, -0.35]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="Body", t=[0, 0, 0], r=[0, 0, 0]) {
    // Carapace - glowing warm yellow amber
    color([1.0, 0.75, 0.1], metalness=0.3, roughness=0.3, emissive=[0.6, 0.4, 0.05], emissiveIntensity=1.4) {
      cube([0.9, 0.55, 0.55], center=true);
    }
    // Visor / Eyes - crisp glowing cyan
    color([0.2, 1.0, 1.0], metalness=0.1, roughness=0.1, emissive=[0.2, 1.0, 1.0], emissiveIntensity=2.6) {
      translate([0, 0.29, 0.05]) cube([0.65, 0.05, 0.18], center=true);
    }

    // Left Claw
    bone(name="ClawLeft", t=[-0.62, 0, 0.1], r=[0, 0, 0]) {
      color([1.0, 0.6, 0.0], metalness=0.3, roughness=0.3, emissive=[0.5, 0.3, 0.0], emissiveIntensity=1.2) {
        cube([0.3, 0.35, 0.45], center=true);
        translate([0, 0.18, 0.16]) cube([0.12, 0.2, 0.18], center=true);
      }
    }

    // Right Claw
    bone(name="ClawRight", t=[0.62, 0, 0.1], r=[0, 0, 0]) {
      color([1.0, 0.6, 0.0], metalness=0.3, roughness=0.3, emissive=[0.5, 0.3, 0.0], emissiveIntensity=1.2) {
        cube([0.3, 0.35, 0.45], center=true);
        translate([0, 0.18, 0.16]) cube([0.12, 0.2, 0.18], center=true);
      }
    }

    // Left Leg
    bone(name="LegLeft", t=[-0.35, 0, -0.35], r=[0, 0, 0]) {
      color([0.9, 0.5, 0.0], metalness=0.3, roughness=0.4) {
        cube([0.22, 0.28, 0.35], center=true);
      }
    }

    // Right Leg
    bone(name="LegRight", t=[0.35, 0, -0.35], r=[0, 0, 0]) {
      color([0.9, 0.5, 0.0], metalness=0.3, roughness=0.4) {
        cube([0.22, 0.28, 0.35], center=true);
      }
    }
  }
}
```

### space_invaders/assets/models/invader_octopus.scad

```openscad
$fn = 20;

// Invader Octopus (Bottom Row - Neon Green)
anim_data = [
  ["march", [
    ["OuterLeftTentacle", [
      [0.0, [0, 0, 0], [-0.52, 0, -0.35]],
      [0.5, [0, 0, 22], [-0.58, 0, -0.28]],
      [1.0, [0, 0, 0], [-0.52, 0, -0.35]]
    ]],
    ["OuterRightTentacle", [
      [0.0, [0, 0, 0], [0.52, 0, -0.35]],
      [0.5, [0, 0, -22], [0.58, 0, -0.28]],
      [1.0, [0, 0, 0], [0.52, 0, -0.35]]
    ]],
    ["InnerLeftTentacle", [
      [0.0, [0, 0, 0], [-0.18, 0, -0.35]],
      [0.5, [0, 0, -14], [-0.14, 0, -0.4]],
      [1.0, [0, 0, 0], [-0.18, 0, -0.35]]
    ]],
    ["InnerRightTentacle", [
      [0.0, [0, 0, 0], [0.18, 0, -0.35]],
      [0.5, [0, 0, 14], [0.14, 0, -0.4]],
      [1.0, [0, 0, 0], [0.18, 0, -0.35]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="Body", t=[0, 0, 0], r=[0, 0, 0]) {
    // Dome Head - glowing emerald green
    color([0.15, 0.95, 0.45], metalness=0.3, roughness=0.3, emissive=[0.1, 0.6, 0.25], emissiveIntensity=1.4) {
      translate([0, 0, 0.08]) cube([1.05, 0.55, 0.55], center=true);
    }
    // Glowing Red Eyes
    color([1.0, 0.2, 0.2], metalness=0.1, roughness=0.1, emissive=[1.0, 0.2, 0.2], emissiveIntensity=2.6) {
      translate([-0.26, 0.29, 0.08]) cube([0.16, 0.05, 0.16], center=true);
      translate([0.26, 0.29, 0.08]) cube([0.16, 0.05, 0.16], center=true);
    }

    // Outer Left Tentacle
    bone(name="OuterLeftTentacle", t=[-0.52, 0, -0.35], r=[0, 0, 0]) {
      color([0.1, 0.85, 0.35], metalness=0.3, roughness=0.4, emissive=[0.05, 0.4, 0.15], emissiveIntensity=1.2) {
        cube([0.2, 0.3, 0.4], center=true);
      }
    }

    // Outer Right Tentacle
    bone(name="OuterRightTentacle", t=[0.52, 0, -0.35], r=[0, 0, 0]) {
      color([0.1, 0.85, 0.35], metalness=0.3, roughness=0.4, emissive=[0.05, 0.4, 0.15], emissiveIntensity=1.2) {
        cube([0.2, 0.3, 0.4], center=true);
      }
    }

    // Inner Left Tentacle
    bone(name="InnerLeftTentacle", t=[-0.18, 0, -0.35], r=[0, 0, 0]) {
      color([0.1, 0.85, 0.35], metalness=0.3, roughness=0.4, emissive=[0.05, 0.4, 0.15], emissiveIntensity=1.2) {
        cube([0.16, 0.3, 0.4], center=true);
      }
    }

    // Inner Right Tentacle
    bone(name="InnerRightTentacle", t=[0.18, 0, -0.35], r=[0, 0, 0]) {
      color([0.1, 0.85, 0.35], metalness=0.3, roughness=0.4, emissive=[0.05, 0.4, 0.15], emissiveIntensity=1.2) {
        cube([0.16, 0.3, 0.4], center=true);
      }
    }
  }
}
```

### space_invaders/assets/models/mystery_ship.scad

```openscad
$fn = 24;

// Mystery Ship / UFO (Flying Saucer - Red / Violet / Chrome)
anim_data = [
  ["spin", [
    ["Rotator", [
      [0.0, [0, 0, 0]],
      [0.25, [0, 0, 90]],
      [0.5, [0, 0, 180]],
      [0.75, [0, 0, 270]],
      [1.0, [0, 0, 360]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="Body", t=[0, 0, 0], r=[0, 0, 0]) {
    // Central Saucer Disc with glowing underbelly
    color([0.88, 0.12, 0.28], metalness=0.8, roughness=0.25, emissive=[0.4, 0.05, 0.1], emissiveIntensity=1.3) {
      scale([1.4, 1.0, 0.32])
        sphere(r=1.0);
    }

    // Upper Cabin Dome
    color([0.2, 0.85, 1.0], alpha=0.9, metalness=0.1, roughness=0.1, emissive=[0.2, 0.85, 1.0], emissiveIntensity=2.4) {
      translate([0, 0, 0.2])
        scale([0.8, 0.6, 0.4])
          sphere(r=0.6);
    }

    // Spinning Propulsion Beacon Ring
    bone(name="Rotator", t=[0, 0, 0], r=[0, 0, 0]) {
      color([1.0, 0.9, 0.2], metalness=0.1, roughness=0.1, emissive=[1.0, 0.9, 0.2], emissiveIntensity=3.5) {
        translate([1.25, 0, 0]) sphere(r=0.16);
        translate([-1.25, 0, 0]) sphere(r=0.16);
        translate([0, 0.95, 0]) sphere(r=0.16);
        translate([0, -0.95, 0]) sphere(r=0.16);
      }
    }
  }
}
```

### space_invaders/assets/models/bunker_block.scad

```openscad
$fn = 16;

// Bunker Block: Destructible energy fortification module
union() {
    color([0.2, 0.85, 0.3], metalness=0.5, roughness=0.3, emissive=[0.05, 0.4, 0.1], emissiveIntensity=1.0) {
        cube([0.52, 0.52, 0.46], center=true);
    }
    // Energy core lattice
    color([0.4, 1.0, 0.5], metalness=0.1, roughness=0.2, emissive=[0.4, 1.0, 0.5], emissiveIntensity=2.0) {
        cube([0.56, 0.14, 0.14], center=true);
        cube([0.14, 0.56, 0.14], center=true);
    }
}
```

### space_invaders/assets/models/player_laser.scad

```openscad
$fn = 16;

// Player Laser: High-energy plasma beam aligned along fire line (OpenSCAD Y -> Godot Z)
union() {
    // Core beam elongated along fire line
    color([0.25, 0.95, 1.0], alpha=0.95, metalness=0.1, roughness=0.1, emissive=[0.35, 0.95, 1.0], emissiveIntensity=4.5) {
        rotate([90, 0, 0])
            cylinder(h=1.4, r=0.08, center=true);
        sphere(r=0.12);
    }
    // Leading plasma tip
    color([0.8, 1.0, 1.0], emissive=[0.9, 1.0, 1.0], emissiveIntensity=5.5) {
        translate([0, 0.7, 0]) sphere(r=0.09);
    }
    // Trailing plasma pulse
    color([0.15, 0.75, 1.0], emissive=[0.2, 0.8, 1.0], emissiveIntensity=3.5) {
        translate([0, -0.6, 0]) sphere(r=0.07);
    }
}
```

### space_invaders/assets/models/enemy_laser.scad

```openscad
$fn = 16;

// Enemy Laser: Jagged plasma zig-zag bomb aligned along fire line (OpenSCAD Y -> Godot Z)
union() {
    // Zig-zag teeth undulating along fire line
    color([1.0, 0.22, 0.1], metalness=0.1, roughness=0.1, emissive=[1.0, 0.25, 0.1], emissiveIntensity=4.5) {
        translate([0, 0.45, 0]) cube([0.16, 0.28, 0.16], center=true);
        translate([0.12, 0.15, 0]) cube([0.16, 0.28, 0.16], center=true);
        translate([-0.12, -0.15, 0]) cube([0.16, 0.28, 0.16], center=true);
        translate([0, -0.45, 0]) cube([0.16, 0.28, 0.16], center=true);
    }
    // Blazing energetic center core
    color([1.0, 0.8, 0.2], emissive=[1.0, 0.8, 0.2], emissiveIntensity=5.5) {
        sphere(r=0.14);
    }
}
```

### space_invaders/scripts/sound_synth.gd

```gdscript
extends Node

var shoot_stream: AudioStreamWAV
var step_streams: Array[AudioStreamWAV] = []
var alien_hit_stream: AudioStreamWAV
var player_hit_stream: AudioStreamWAV
var ufo_stream: AudioStreamWAV
var ufo_hit_stream: AudioStreamWAV

var audio_players: Array[AudioStreamPlayer] = []
var ufo_player: AudioStreamPlayer
const POOL_SIZE = 8

func _ready():
	_generate_audio_streams()
	for i in range(POOL_SIZE):
		var p = AudioStreamPlayer.new()
		add_child(p)
		audio_players.append(p)

	ufo_player = AudioStreamPlayer.new()
	add_child(ufo_player)

func _play_stream(stream: AudioStreamWAV, volume_db: float = 0.0):
	if not stream:
		return
	for p in audio_players:
		if not p.playing:
			p.stream = stream
			p.volume_db = volume_db
			p.play()
			return
	audio_players[0].stream = stream
	audio_players[0].volume_db = volume_db
	audio_players[0].play()

func play_shoot():
	_play_stream(shoot_stream, -2.0)

func play_invader_step(index: int):
	var s = step_streams[index % step_streams.size()]
	_play_stream(s, 0.0)

func play_alien_hit():
	_play_stream(alien_hit_stream, 1.0)

func play_player_hit():
	_play_stream(player_hit_stream, 2.0)

func play_ufo_hit():
	_play_stream(ufo_hit_stream, 2.0)

func start_ufo_sound():
	if ufo_player and not ufo_player.playing:
		ufo_player.stream = ufo_stream
		ufo_player.volume_db = -4.0
		ufo_player.play()

func stop_ufo_sound():
	if ufo_player and ufo_player.playing:
		ufo_player.stop()

func _generate_audio_streams():
	var rate = 22050

	# 1. Player Shoot
	var shoot_samples = int(rate * 0.11)
	var shoot_bytes = PackedByteArray()
	shoot_bytes.resize(shoot_samples)
	for i in range(shoot_samples):
		var t = float(i) / rate
		var f = lerp(900.0, 220.0, float(i) / shoot_samples)
		var env = 1.0 - (float(i) / shoot_samples)
		var val = 128 + int(80.0 * env * (1.0 if fmod(t * f, 1.0) < 0.5 else -1.0))
		shoot_bytes[i] = clamp(val, 0, 255)
	shoot_stream = _create_wav(shoot_bytes, rate)

	# 2. Four Invader March Steps
	var freqs = [175.0, 155.0, 138.0, 123.0]
	for base_f in freqs:
		var step_samples = int(rate * 0.075)
		var step_bytes = PackedByteArray()
		step_bytes.resize(step_samples)
		for i in range(step_samples):
			var t = float(i) / rate
			var env = 1.0 - pow(float(i) / step_samples, 2.0)
			var val = 128 + int(90.0 * env * (1.0 if fmod(t * base_f, 1.0) < 0.5 else -1.0))
			step_bytes[i] = clamp(val, 0, 255)
		step_streams.append(_create_wav(step_bytes, rate))

	# 3. Alien Hit
	var hit_samples = int(rate * 0.16)
	var hit_bytes = PackedByteArray()
	hit_bytes.resize(hit_samples)
	for i in range(hit_samples):
		var env = exp(-6.0 * (float(i) / hit_samples))
		var noise = randf_range(-1.0, 1.0)
		var val = 128 + int(90.0 * env * noise)
		hit_bytes[i] = clamp(val, 0, 255)
	alien_hit_stream = _create_wav(hit_bytes, rate)

	# 4. Player Death Hit
	var p_samples = int(rate * 0.45)
	var p_bytes = PackedByteArray()
	p_bytes.resize(p_samples)
	for i in range(p_samples):
		var t = float(i) / rate
		var env = 1.0 - (float(i) / p_samples)
		var noise = randf_range(-1.0, 1.0)
		var rumble = sin(2.0 * PI * 60.0 * t)
		var val = 128 + int((70.0 * noise + 40.0 * rumble) * env)
		p_bytes[i] = clamp(val, 0, 255)
	player_hit_stream = _create_wav(p_bytes, rate)

	# 5. UFO Warble
	var ufo_samples = int(rate * 0.28)
	var ufo_bytes = PackedByteArray()
	ufo_bytes.resize(ufo_samples)
	for i in range(ufo_samples):
		var t = float(i) / rate
		var mod_freq = 510.0 + 70.0 * sin(2.0 * PI * 8.0 * t)
		var val = 128 + int(70.0 * (1.0 if fmod(t * mod_freq, 1.0) < 0.5 else -1.0))
		ufo_bytes[i] = clamp(val, 0, 255)
	ufo_stream = _create_wav(ufo_bytes, rate, true)

	# 6. UFO Hit Chime
	var uh_samples = int(rate * 0.35)
	var uh_bytes = PackedByteArray()
	uh_bytes.resize(uh_samples)
	for i in range(uh_samples):
		var t = float(i) / rate
		var note_idx = int(float(i) / uh_samples * 4.0)
		var note_freqs = [600.0, 800.0, 1000.0, 1200.0]
		var f = note_freqs[mini(note_idx, 3)]
		var env = 1.0 - (float(i) / uh_samples)
		var val = 128 + int(85.0 * env * (1.0 if fmod(t * f, 1.0) < 0.5 else -1.0))
		uh_bytes[i] = clamp(val, 0, 255)
	ufo_hit_stream = _create_wav(uh_bytes, rate)

func _create_wav(data: PackedByteArray, sample_rate: int, loop: bool = false) -> AudioStreamWAV:
	var s = AudioStreamWAV.new()
	s.format = AudioStreamWAV.FORMAT_8_BITS
	s.mix_rate = sample_rate
	s.data = data
	if loop:
		s.loop_mode = AudioStreamWAV.LOOP_FORWARD
		s.loop_end = data.size()
	return s
```

### space_invaders/scripts/explosion.gd

```gdscript
extends Node3D

func _ready():
	var particles = $CPUParticles3D
	if particles:
		particles.emitting = true
	var timer = get_tree().create_timer(0.6)
	timer.timeout.connect(queue_free)
```

### space_invaders/scenes/explosion.tscn

```tscn
[gd_scene load_steps=4 format=3]

[ext_resource type="Script" path="res://scripts/explosion.gd" id="1_exp"]

[sub_resource type="StandardMaterial3D" id="StandardMaterial3D_exp"]
shading_mode = 0
albedo_color = Color(1, 0.7, 0.2, 1)

[sub_resource type="BoxMesh" id="BoxMesh_exp"]
material = SubResource("StandardMaterial3D_exp")
size = Vector3(0.12, 0.12, 0.12)

[node name="Explosion" type="Node3D"]
script = ExtResource("1_exp")

[node name="CPUParticles3D" type="CPUParticles3D" parent="."]
emitting = false
amount = 24
lifetime = 0.5
one_shot = true
explosiveness = 0.95
mesh = SubResource("BoxMesh_exp")
direction = Vector3(0, 1, 0)
spread = 180.0
gravity = Vector3(0, -5, 0)
initial_velocity_min = 3.0
initial_velocity_max = 7.0
```

### space_invaders/scripts/laser.gd

```gdscript
extends Area3D

enum LaserType { PLAYER, ENEMY }

@export var laser_type: LaserType = LaserType.PLAYER
@export var speed: float = 24.0

func _ready():
	area_entered.connect(_on_area_entered)
	if laser_type == LaserType.PLAYER:
		collision_layer = 4
		collision_mask = 2 | 16 | 32
		speed = 28.0
	else:
		collision_layer = 8
		collision_mask = 1 | 16
		speed = 13.0

func _physics_process(delta: float):
	if laser_type == LaserType.PLAYER:
		global_position.z -= speed * delta
		if global_position.z < -19.0:
			queue_free()
	else:
		global_position.z += speed * delta
		if global_position.z > 14.0:
			queue_free()

func _on_area_entered(area: Area3D):
	if area.has_method("take_hit"):
		area.take_hit(1, self)
		queue_free()
```

### space_invaders/scenes/laser.tscn

```tscn
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/laser.gd" id="1_las"]

[sub_resource type="BoxShape3D" id="BoxShape3D_las"]
size = Vector3(0.35, 0.35, 1.3)

[node name="Laser" type="Area3D"]
collision_layer = 4
collision_mask = 50
script = ExtResource("1_las")

[node name="CollisionShape3D" type="CollisionShape3D" parent="."]
shape = SubResource("BoxShape3D_las")
```

### space_invaders/scripts/bunker_block.gd

```gdscript
extends Area3D

signal block_destroyed

func _ready():
	collision_layer = 16
	collision_mask = 0

func take_hit(_damage: int, _bullet):
	block_destroyed.emit()
	queue_free()
```

### space_invaders/scenes/bunker_block.tscn

```tscn
[gd_scene load_steps=4 format=3]

[ext_resource type="Script" path="res://scripts/bunker_block.gd" id="1_bb"]
[ext_resource type="PackedScene" path="res://assets/models/bunker_block.scad" id="2_mod"]

[sub_resource type="BoxShape3D" id="BoxShape3D_bb"]
size = Vector3(0.55, 0.55, 0.55)

[node name="BunkerBlock" type="Area3D"]
collision_layer = 16
collision_mask = 0
script = ExtResource("1_bb")

[node name="Model" parent="." instance=ExtResource("2_mod")]

[node name="CollisionShape3D" type="CollisionShape3D" parent="."]
shape = SubResource("BoxShape3D_bb")
```

### space_invaders/scripts/bunker.gd

```gdscript
extends Node3D

const BunkerBlockScene = preload("res://scenes/bunker_block.tscn")

func _ready():
	_build_bunker()

func _build_bunker():
	var grid = [
		[0, 1, 1, 1, 0],
		[1, 1, 1, 1, 1],
		[1, 1, 1, 1, 1],
		[1, 0, 0, 0, 1]
	]
	var block_spacing = 0.55
	var rows = grid.size()
	var cols = grid[0].size()

	for r in range(rows):
		for c in range(cols):
			if grid[r][c] == 1:
				var block = BunkerBlockScene.instantiate()
				var x_pos = (float(c) - float(cols - 1) / 2.0) * block_spacing
				var z_pos = (float(r) - float(rows - 1) / 2.0) * block_spacing
				block.position = Vector3(x_pos, 0.25, z_pos)
				add_child(block)
```

### space_invaders/scenes/bunker.tscn

```tscn
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/bunker.gd" id="1_bunk"]

[node name="Bunker" type="Node3D"]
script = ExtResource("1_bunk")
```

### space_invaders/scripts/invader.gd

```gdscript
extends Area3D

enum InvaderType { SQUID, CRAB, OCTOPUS }

@export var type: InvaderType = InvaderType.SQUID
@export var points: int = 30
@export var row: int = 0
@export var col: int = 0

var anim_player: AnimationPlayer = null
var step_state: bool = false

const SquidScene = preload("res://assets/models/invader_squid.scad")
const CrabScene = preload("res://assets/models/invader_crab.scad")
const OctopusScene = preload("res://assets/models/invader_octopus.scad")

signal invader_killed(invader)

func _ready():
	collision_layer = 2
	collision_mask = 0
	# Enemies face forward towards the player (+Z)
	rotation.y = PI
	_setup_model()

func _setup_model():
	var model_node = null
	match type:
		InvaderType.SQUID:
			points = 30
			model_node = SquidScene.instantiate()
		InvaderType.CRAB:
			points = 20
			model_node = CrabScene.instantiate()
		InvaderType.OCTOPUS:
			points = 10
			model_node = OctopusScene.instantiate()

	if model_node:
		add_child(model_node)
		_find_animation_player(model_node)

func _find_animation_player(node: Node):
	if node is AnimationPlayer:
		anim_player = node
		return
	for child in node.get_children():
		_find_animation_player(child)
		if anim_player:
			return

func step_animation():
	if anim_player and anim_player.has_animation("march"):
		step_state = not step_state
		anim_player.play("march")
		anim_player.seek(0.5 if step_state else 0.0, true)
		anim_player.pause()

func take_hit(_damage: int, _bullet):
	invader_killed.emit(self)
	queue_free()
```

### space_invaders/scenes/invader.tscn

```tscn
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/invader.gd" id="1_inv"]

[sub_resource type="BoxShape3D" id="BoxShape3D_inv"]
size = Vector3(1.4, 0.8, 1.2)

[node name="Invader" type="Area3D"]
collision_layer = 2
collision_mask = 0
script = ExtResource("1_inv")

[node name="CollisionShape3D" type="CollisionShape3D" parent="."]
shape = SubResource("BoxShape3D_inv")
```

### space_invaders/scripts/invader_fleet.gd

```gdscript
extends Node3D

signal fleet_stepped(step_index)
signal fleet_cleared
signal invasion_reached
signal invader_destroyed(points, global_pos)

const InvaderScene = preload("res://scenes/invader.tscn")
const LaserScene = preload("res://scenes/laser.tscn")
const EnemyLaserModel = preload("res://assets/models/enemy_laser.scad")

var invaders: Array[Area3D] = []
var move_direction: float = 1.0
var step_size_x: float = 0.42
var drop_size_z: float = 0.75
var base_step_interval: float = 0.65
var current_step_timer: float = 0.0
var step_index: int = 0
var total_invaders: int = 0

var bomb_cooldown: float = 1.8
var bomb_timer: float = 0.0

const COLS = 10
const ROWS = 5
const SPACING_X = 2.15
const SPACING_Z = 2.4
const BOUND_X = 12.0
const INVASION_Z = 4.6

var is_active: bool = false

func start_fleet(wave: int = 1):
	_clear_fleet()
	base_step_interval = maxf(0.25, 0.65 - float(wave - 1) * 0.05)
	current_step_timer = base_step_interval
	move_direction = 1.0
	step_index = 0

	for r in range(ROWS):
		for c in range(COLS):
			var invader = InvaderScene.instantiate()
			invader.row = r
			invader.col = c

			if r == 0:
				invader.type = invader.InvaderType.SQUID
			elif r == 1 or r == 2:
				invader.type = invader.InvaderType.CRAB
			else:
				invader.type = invader.InvaderType.OCTOPUS

			var x = (float(c) - float(COLS - 1) / 2.0) * SPACING_X
			# Balanced starting depth framing the upper screen below top HUD
			var z = -13.1 + float(r) * SPACING_Z
			invader.position = Vector3(x, 0.0, z)
			invader.invader_killed.connect(_on_invader_killed)
			add_child(invader)
			invaders.append(invader)

	total_invaders = invaders.size()
	is_active = true

func _clear_fleet():
	for inv in invaders:
		if is_instance_valid(inv):
			inv.queue_free()
	invaders.clear()

func _physics_process(delta: float):
	if not is_active or invaders.is_empty():
		return

	current_step_timer -= delta
	if current_step_timer <= 0.0:
		_perform_fleet_step()
		var alive_count = invaders.size()
		var speed_ratio = float(alive_count) / float(total_invaders)
		var next_interval = lerpf(0.06, base_step_interval, speed_ratio)
		current_step_timer = next_interval

	bomb_timer -= delta
	if bomb_timer <= 0.0:
		_attempt_drop_bomb()
		bomb_timer = randf_range(1.0, 2.2)

func _perform_fleet_step():
	var min_x = 999.0
	var max_x = -999.0
	var max_z = -999.0

	for inv in invaders:
		if is_instance_valid(inv):
			min_x = minf(min_x, inv.global_position.x)
			max_x = maxf(max_x, inv.global_position.x)
			max_z = maxf(max_z, inv.global_position.z)

	var should_drop = false
	if move_direction > 0.0 and max_x + step_size_x >= BOUND_X:
		should_drop = true
	elif move_direction < 0.0 and min_x - step_size_x <= -BOUND_X:
		should_drop = true

	if should_drop:
		move_direction *= -1.0
		for inv in invaders:
			if is_instance_valid(inv):
				inv.global_position.z += drop_size_z
				inv.step_animation()
		if max_z + drop_size_z >= INVASION_Z:
			invasion_reached.emit()
			is_active = false
			return
	else:
		for inv in invaders:
			if is_instance_valid(inv):
				inv.global_position.x += move_direction * step_size_x
				inv.step_animation()

	fleet_stepped.emit(step_index % 4)
	step_index += 1

func _attempt_drop_bomb():
	if invaders.is_empty():
		return

	var column_lowest = {}
	for inv in invaders:
		if is_instance_valid(inv):
			var c = inv.col
			if not column_lowest.has(c) or inv.global_position.z > column_lowest[c].global_position.z:
				column_lowest[c] = inv

	var available_shooters = column_lowest.values()
	if available_shooters.is_empty():
		return

	var shooter = available_shooters.pick_random()
	var bomb = LaserScene.instantiate()
	bomb.laser_type = bomb.LaserType.ENEMY

	var model = EnemyLaserModel.instantiate()
	bomb.add_child(model)

	var parent_tree = get_tree().current_scene
	if parent_tree:
		bomb.position = shooter.global_position + Vector3(0.0, 0.35, 0.9)
		parent_tree.add_child(bomb)

func _on_invader_killed(invader: Area3D):
	var pos = invader.global_position
	var pts = invader.points
	invaders.erase(invader)
	invader_destroyed.emit(pts, pos)

	if invaders.is_empty():
		is_active = false
		fleet_cleared.emit()
```

### space_invaders/scripts/mystery_ship.gd

```gdscript
extends Area3D

signal mystery_ship_destroyed(points, global_pos)
signal mystery_ship_escaped

const MysteryModelScene = preload("res://assets/models/mystery_ship.scad")

var speed: float = 6.8
var direction: float = 1.0
var score_values = [50, 100, 150, 300]
var anim_player: AnimationPlayer = null

func _ready():
	collision_layer = 32
	collision_mask = 0
	var model = MysteryModelScene.instantiate()
	add_child(model)
	_find_anim_player(model)
	if anim_player and anim_player.has_animation("spin"):
		var anim = anim_player.get_animation("spin")
		if anim:
			anim.loop_mode = Animation.LOOP_LINEAR
		anim_player.play("spin")

func _find_anim_player(node: Node):
	if node is AnimationPlayer:
		anim_player = node
		return
	for c in node.get_children():
		_find_anim_player(c)
		if anim_player:
			return

func setup_spawn(from_left: bool):
	if from_left:
		direction = 1.0
		global_position = Vector3(-16.0, 0.5, -15.5)
	else:
		direction = -1.0
		global_position = Vector3(16.0, 0.5, -15.5)

func _physics_process(delta: float):
	global_position.x += direction * speed * delta
	if (direction > 0.0 and global_position.x > 17.0) or (direction < 0.0 and global_position.x < -17.0):
		mystery_ship_escaped.emit()
		queue_free()

func take_hit(_damage: int, _bullet):
	var pts = score_values.pick_random()
	mystery_ship_destroyed.emit(pts, global_position)
	queue_free()
```

### space_invaders/scenes/mystery_ship.tscn

```tscn
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/mystery_ship.gd" id="1_ms"]

[sub_resource type="BoxShape3D" id="BoxShape3D_ms"]
size = Vector3(2.6, 0.8, 1.8)

[node name="MysteryShip" type="Area3D"]
collision_layer = 32
collision_mask = 0
script = ExtResource("1_ms")

[node name="CollisionShape3D" type="CollisionShape3D" parent="."]
shape = SubResource("BoxShape3D_ms")
```

### space_invaders/scripts/touch_controls.gd

```gdscript
extends Control

signal move_input(dir: float)
signal fire_pressed
signal start_pressed

var active_touches: Dictionary = {}
var current_dir: float = 0.0
var is_firing: bool = false

func _ready():
	mouse_filter = MOUSE_FILTER_PASS

func _gui_input(event: InputEvent):
	if event is InputEventScreenTouch:
		if event.pressed:
			active_touches[event.index] = event.position
			start_pressed.emit()
			_check_fire(event.position)
		else:
			active_touches.erase(event.index)
		_recalculate_movement()
	elif event is InputEventScreenDrag:
		active_touches[event.index] = event.position
		_recalculate_movement()

func _check_fire(pos: Vector2):
	var vp = get_viewport_rect().size
	var norm_x = pos.x / vp.x
	var norm_y = pos.y / vp.y
	if (norm_x >= 0.38 and norm_x <= 0.62) or norm_y < 0.60:
		fire_pressed.emit()

func _recalculate_movement():
	var dir = 0.0
	var vp = get_viewport_rect().size

	for idx in active_touches:
		var pos = active_touches[idx]
		var norm_x = pos.x / vp.x
		if norm_x < 0.40:
			dir -= 1.0
		elif norm_x > 0.60:
			dir += 1.0

	current_dir = clampf(dir, -1.0, 1.0)
	move_input.emit(current_dir)
```

### space_invaders/scripts/player.gd

```gdscript
extends Area3D

signal player_hit
signal player_fired(laser_pos)

const LaserScene = preload("res://scenes/laser.tscn")
const PlayerLaserModel = preload("res://assets/models/player_laser.scad")
const PlayerShipModel = preload("res://assets/models/player_ship.scad")

@export var move_speed: float = 13.0
@export var bound_x: float = 11.2

var can_fire: bool = true
var fire_cooldown: float = 0.32
var current_fire_timer: float = 0.0
var is_invulnerable: bool = false
var model_node: Node3D = null

var touch_dir: float = 0.0

func _ready():
	collision_layer = 1
	collision_mask = 0
	model_node = PlayerShipModel.instantiate()
	add_child(model_node)

func set_touch_direction(dir: float):
	touch_dir = dir

func touch_fire():
	if can_fire and visible:
		_fire_laser()

func _physics_process(delta: float):
	if current_fire_timer > 0.0:
		current_fire_timer -= delta
		if current_fire_timer <= 0.0:
			can_fire = true

	var input_x = touch_dir
	if Input.is_action_pressed("ui_left") or Input.is_key_pressed(KEY_A) or Input.is_key_pressed(KEY_LEFT):
		input_x -= 1.0
	if Input.is_action_pressed("ui_right") or Input.is_key_pressed(KEY_D) or Input.is_key_pressed(KEY_RIGHT):
		input_x += 1.0
	input_x = clampf(input_x, -1.0, 1.0)

	global_position.x += input_x * move_speed * delta
	global_position.x = clampf(global_position.x, -bound_x, bound_x)

	if model_node:
		var target_tilt = -input_x * 0.18
		model_node.rotation.z = lerpf(model_node.rotation.z, target_tilt, 15.0 * delta)

	if can_fire and (Input.is_action_pressed("ui_accept") or Input.is_key_pressed(KEY_SPACE) or Input.is_mouse_button_pressed(MOUSE_BUTTON_LEFT)):
		_fire_laser()

func _fire_laser():
	can_fire = false
	current_fire_timer = fire_cooldown

	var laser = LaserScene.instantiate()
	laser.laser_type = laser.LaserType.PLAYER

	var laser_model = PlayerLaserModel.instantiate()
	laser.add_child(laser_model)

	var spawn_pos = global_position + Vector3(0.0, 0.35, -1.6)
	var parent_tree = get_tree().current_scene
	if parent_tree:
		laser.position = spawn_pos
		parent_tree.add_child(laser)

	player_fired.emit(spawn_pos)

func take_hit(_damage: int, _bullet):
	if is_invulnerable:
		return
	player_hit.emit()

func trigger_respawn():
	is_invulnerable = true
	visible = false
	await get_tree().create_timer(0.6).timeout
	global_position.x = 0.0
	visible = true
	for i in range(5):
		visible = not visible
		await get_tree().create_timer(0.12).timeout
	visible = true
	is_invulnerable = false
```

### space_invaders/scenes/player.tscn

```tscn
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/player.gd" id="1_ply"]

[sub_resource type="BoxShape3D" id="BoxShape3D_ply"]
size = Vector3(2.2, 0.8, 1.8)

[node name="Player" type="Area3D"]
collision_layer = 1
collision_mask = 0
script = ExtResource("1_ply")

[node name="CollisionShape3D" type="CollisionShape3D" parent="."]
shape = SubResource("BoxShape3D_ply")
```

### space_invaders/scripts/main_game.gd

```gdscript
extends Node3D

enum GameState { TITLE, PLAYING, PLAYER_DYING, WAVE_CLEARED, GAME_OVER }

var current_state: GameState = GameState.PLAYING
var score: int = 0
var hi_score: int = 5000
var lives: int = 3
var wave: int = 1

var ufo_spawn_timer: float = 12.0
var active_ufo: Area3D = null

@onready var sound_synth = $SoundSynth
@onready var fleet = $InvaderFleet
@onready var player = $Player
@onready var bunkers_container = $Bunkers
@onready var touch_controls = $CanvasLayer/TouchControls

# UI References
@onready var score_label = $CanvasLayer/HUD/TopBar/ScoreLabel
@onready var hi_score_label = $CanvasLayer/HUD/TopBar/HiScoreLabel
@onready var wave_label = $CanvasLayer/HUD/TopBar/WaveLabel
@onready var lives_label = $CanvasLayer/HUD/BottomBar/LivesLabel
@onready var message_overlay = $CanvasLayer/CenterMessage
@onready var title_label = $CanvasLayer/CenterMessage/TitleLabel
@onready var prompt_label = $CanvasLayer/CenterMessage/PromptLabel

const ExplosionScene = preload("res://scenes/explosion.tscn")
const MysteryShipScene = preload("res://scenes/mystery_ship.tscn")
const BunkerScene = preload("res://scenes/bunker.tscn")

func _ready():
	_setup_signals()
	_reset_bunkers()
	_update_ui()
	_start_new_game()

func _setup_signals():
	fleet.fleet_stepped.connect(_on_fleet_stepped)
	fleet.fleet_cleared.connect(_on_fleet_cleared)
	fleet.invasion_reached.connect(_on_invasion_reached)
	fleet.invader_destroyed.connect(_on_invader_destroyed)

	player.player_fired.connect(_on_player_fired)
	player.player_hit.connect(_on_player_hit)

	if touch_controls:
		touch_controls.move_input.connect(_on_touch_move)
		touch_controls.fire_pressed.connect(_on_touch_fire)
		touch_controls.start_pressed.connect(_on_touch_start)

func _on_touch_move(dir: float):
	if player:
		player.set_touch_direction(dir)

func _on_touch_fire():
	if current_state == GameState.PLAYING and player:
		player.touch_fire()

func _on_touch_start():
	if current_state == GameState.TITLE or current_state == GameState.GAME_OVER:
		_start_new_game()

func _process(delta: float):
	match current_state:
		GameState.TITLE, GameState.GAME_OVER:
			if Input.is_key_pressed(KEY_SPACE) or Input.is_action_just_pressed("ui_accept"):
				_start_new_game()
		GameState.PLAYING:
			ufo_spawn_timer -= delta
			if ufo_spawn_timer <= 0.0:
				_spawn_ufo()
				ufo_spawn_timer = randf_range(16.0, 26.0)

func _start_new_game():
	score = 0
	lives = 3
	wave = 1
	current_state = GameState.PLAYING
	message_overlay.visible = false
	player.visible = true
	player.global_position = Vector3(0, 0, 7.8)

	_reset_bunkers()
	fleet.start_fleet(wave)
	_update_ui()

func _start_next_wave():
	wave += 1
	current_state = GameState.PLAYING
	message_overlay.visible = false
	fleet.start_fleet(wave)
	_update_ui()

func _reset_bunkers():
	for c in bunkers_container.get_children():
		c.queue_free()

	var bunker_x_positions = [-7.2, -2.4, 2.4, 7.2]
	for x_pos in bunker_x_positions:
		var b = BunkerScene.instantiate()
		b.position = Vector3(x_pos, 0.0, 4.8)
		bunkers_container.add_child(b)

func _spawn_ufo():
	if active_ufo != null and is_instance_valid(active_ufo):
		return

	var ufo = MysteryShipScene.instantiate()
	# Add to tree first so setup_spawn can modify global_position without throwing errors
	add_child(ufo)

	var from_left = randf() > 0.5
	ufo.setup_spawn(from_left)
	ufo.mystery_ship_destroyed.connect(_on_ufo_destroyed)
	ufo.mystery_ship_escaped.connect(_on_ufo_escaped)
	active_ufo = ufo
	sound_synth.start_ufo_sound()

func _on_ufo_destroyed(points_awarded: int, pos: Vector3):
	sound_synth.stop_ufo_sound()
	sound_synth.play_ufo_hit()
	_spawn_explosion(pos)
	_add_score(points_awarded)
	active_ufo = null

func _on_ufo_escaped():
	sound_synth.stop_ufo_sound()
	active_ufo = null

func _on_fleet_stepped(step_idx: int):
	sound_synth.play_invader_step(step_idx)

func _on_invader_destroyed(points_awarded: int, pos: Vector3):
	sound_synth.play_alien_hit()
	_spawn_explosion(pos)
	_add_score(points_awarded)

func _on_player_fired(_pos: Vector3):
	sound_synth.play_shoot()

func _on_player_hit():
	if current_state != GameState.PLAYING:
		return

	current_state = GameState.PLAYER_DYING
	sound_synth.play_player_hit()
	_spawn_explosion(player.global_position)
	lives -= 1
	_update_ui()

	if lives <= 0:
		_game_over()
	else:
		player.trigger_respawn()
		await get_tree().create_timer(1.2).timeout
		if current_state == GameState.PLAYER_DYING:
			current_state = GameState.PLAYING

func _on_fleet_cleared():
	current_state = GameState.WAVE_CLEARED
	message_overlay.visible = true
	title_label.text = "WAVE %d CLEARED!" % wave
	prompt_label.text = "PREPARING NEXT WAVE..."
	await get_tree().create_timer(2.0).timeout
	_start_next_wave()

func _on_invasion_reached():
	_game_over()

func _game_over():
	current_state = GameState.GAME_OVER
	if sound_synth:
		sound_synth.stop_ufo_sound()
	message_overlay.visible = true
	title_label.text = "GAME OVER"
	prompt_label.text = "TAP SCREEN OR PRESS [SPACE] TO RETRY"

func _add_score(pts: int):
	score += pts
	if score > hi_score:
		hi_score = score
	_update_ui()

func _spawn_explosion(pos: Vector3):
	var exp_node = ExplosionScene.instantiate()
	exp_node.position = pos
	add_child(exp_node)

func _update_ui():
	if score_label:
		score_label.text = "SCORE: %05d" % score
	if hi_score_label:
		hi_score_label.text = "HI-SCORE: %05d" % hi_score
	if wave_label:
		wave_label.text = "WAVE: %02d" % wave
	if lives_label:
		lives_label.text = "LIVES: %d" % lives
```

### space_invaders/scenes/main_game.tscn

```tscn
[gd_scene load_steps=10 format=3]

[ext_resource type="Script" path="res://scripts/main_game.gd" id="1_main"]
[ext_resource type="Script" path="res://scripts/sound_synth.gd" id="2_snd"]
[ext_resource type="Script" path="res://scripts/invader_fleet.gd" id="3_flt"]
[ext_resource type="PackedScene" path="res://scenes/player.tscn" id="4_ply"]
[ext_resource type="Script" path="res://scripts/touch_controls.gd" id="5_tch"]

[sub_resource type="Environment" id="Environment_main"]
background_mode = 1
background_color = Color(0.015, 0.015, 0.04, 1)
ambient_light_source = 2
ambient_light_color = Color(0.25, 0.3, 0.45, 1)
ambient_light_energy = 1.1
glow_enabled = true
glow_intensity = 0.5
glow_bloom = 0.35

[sub_resource type="StandardMaterial3D" id="StandardMaterial3D_star"]
shading_mode = 0
albedo_color = Color(0.85, 0.92, 1, 0.85)

[sub_resource type="BoxMesh" id="BoxMesh_star"]
material = SubResource("StandardMaterial3D_star")
size = Vector3(0.08, 0.08, 0.08)

[sub_resource type="LabelSettings" id="LabelSettings_arcade"]
font_size = 22
font_color = Color(0.2, 1, 0.4, 1)
outline_size = 2
outline_color = Color(0, 0.4, 0.1, 1)

[node name="MainGame" type="Node3D"]
script = ExtResource("1_main")

[node name="WorldEnvironment" type="WorldEnvironment" parent="."]
environment = SubResource("Environment_main")

[node name="DirectionalLight3D" type="DirectionalLight3D" parent="."]
transform = Transform3D(0.866, -0.354, 0.354, 0, 0.707, 0.707, -0.5, -0.612, 0.612, 5, 20, 12)
light_color = Color(0.95, 0.97, 1, 1)
light_energy = 1.3
shadow_enabled = true

[node name="Camera3D" type="Camera3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 0.707, 0.707, 0, -0.707, 0.707, 0, 15.5, 15.5)
fov = 50.0

[node name="StarfieldParticles" type="CPUParticles3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, -3, 0)
amount = 180
lifetime = 10.0
preprocess = 5.0
emission_shape = 3
emission_box_extents = Vector3(26, 1, 26)
mesh = SubResource("BoxMesh_star")
gravity = Vector3(0, 0, 0)

[node name="SoundSynth" type="Node" parent="."]
script = ExtResource("2_snd")

[node name="Bunkers" type="Node3D" parent="."]

[node name="InvaderFleet" type="Node3D" parent="."]
script = ExtResource("3_flt")

[node name="Player" parent="." instance=ExtResource("4_ply")]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 7.8)

[node name="CanvasLayer" type="CanvasLayer" parent="."]

[node name="TouchControls" type="Control" parent="CanvasLayer"]
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
script = ExtResource("5_tch")

[node name="HUD" type="Control" parent="CanvasLayer"]
mouse_filter = 2
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2

[node name="TopBar" type="HBoxContainer" parent="CanvasLayer/HUD"]
layout_mode = 1
anchors_preset = 10
anchor_right = 1.0
offset_left = 40.0
offset_top = 20.0
offset_right = -40.0
offset_bottom = 60.0
grow_horizontal = 2
theme_override_constants/separation = 80

[node name="ScoreLabel" type="Label" parent="CanvasLayer/HUD/TopBar"]
layout_mode = 2
text = "SCORE: 00000"
label_settings = SubResource("LabelSettings_arcade")

[node name="HiScoreLabel" type="Label" parent="CanvasLayer/HUD/TopBar"]
layout_mode = 2
size_flags_horizontal = 3
text = "HI-SCORE: 05000"
label_settings = SubResource("LabelSettings_arcade")
horizontal_alignment = 1

[node name="WaveLabel" type="Label" parent="CanvasLayer/HUD/TopBar"]
layout_mode = 2
text = "WAVE: 01"
label_settings = SubResource("LabelSettings_arcade")
horizontal_alignment = 2

[node name="BottomBar" type="HBoxContainer" parent="CanvasLayer/HUD"]
layout_mode = 1
anchors_preset = 12
anchor_top = 1.0
anchor_right = 1.0
anchor_bottom = 1.0
offset_left = 40.0
offset_top = -50.0
offset_right = -40.0
offset_bottom = -20.0
grow_horizontal = 2
grow_vertical = 0

[node name="LivesLabel" type="Label" parent="CanvasLayer/HUD/BottomBar"]
layout_mode = 2
text = "LIVES: 3"
label_settings = SubResource("LabelSettings_arcade")

[node name="CenterMessage" type="VBoxContainer" parent="CanvasLayer"]
visible = false
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -300.0
offset_top = -100.0
offset_right = 300.0
offset_bottom = 100.0
grow_horizontal = 2
grow_vertical = 2
theme_override_constants/separation = 20
alignment = 1

[node name="TitleLabel" type="Label" parent="CanvasLayer/CenterMessage"]
layout_mode = 2
theme_override_colors/font_color = Color(1, 0.2, 0.4, 1)
theme_override_font_sizes/font_size = 38
text = "SPACE INVADERS 3D"
horizontal_alignment = 1

[node name="PromptLabel" type="Label" parent="CanvasLayer/CenterMessage"]
layout_mode = 2
theme_override_colors/font_color = Color(0.9, 0.9, 0.9, 1)
theme_override_font_sizes/font_size = 20
text = "TAP SCREEN OR PRESS [SPACE] TO START"
horizontal_alignment = 1
```
