### packman_3d/addons/scad_importer/plugin.cfg
```ini
[plugin]

name="OpenSCAD GLTF Importer"
description="Imports .scad files directly as 3D scenes using scad-gltf"
author="Ilia Grigorev"
version="0.1"
script="scad_plugin.gd"
```

### packman_3d/addons/scad_importer/scad_importer.gd
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

### packman_3d/addons/scad_importer/scad_plugin.gd
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

### packman_3d/project.godot
```ini
; Engine configuration file.
; It's best edited using the editor UI and not directly,
; since the parameters that go here are not all obvious.

config_version=5

[application]

config/name="PAC-MAN 3D // Cyber Maze Edition"
run/main_scene="res://scenes/main.tscn"
config/features=PackedStringArray("4.3", "Forward Plus")
config/icon="res://icon.svg"

[display]

window/size/viewport_width=1280
window/size/viewport_height=720
window/stretch/mode="canvas_items"
window/stretch/aspect="keep"
window/handheld/orientation="landscape"

[editor_plugins]

enabled=PackedStringArray("res://addons/scad_importer/plugin.cfg")

[input_devices]

pointing/emulate_touch_from_mouse=true
pointing/emulate_mouse_from_touch=true

[rendering]

anti_aliasing/quality/msaa_3d=2
anti_aliasing/quality/screen_space_aa=1
lights_and_shadows/directional_shadow/soft_shadow_filter_quality=2
```

### packman_3d/icon.svg
```xml
<svg xmlns="http://www.w3.org/2000/svg" width="128" height="128" viewBox="0 0 128 128">
  <rect width="128" height="128" rx="24" fill="#080b18"/>
  <path d="M64 20 A44 44 0 1 1 32.8 95.2 L64 64 L95.2 95.2 A44 44 0 0 1 64 20 Z" fill="#ffd70e"/>
  <circle cx="64" cy="40" r="6" fill="#111"/>
  <circle cx="108" cy="64" r="7" fill="#00d9ff"/>
  <circle cx="122" cy="64" r="5" fill="#ffb833"/>
</svg>
```

### packman_3d/assets/models/packman.scad
```openscad
$fn = 40;
$asa = 45;

// Pacman faces +Y (Forward), Left is -X, Right is +X, Up is +Z
anim_data = [
  ["Chomp", [
    ["MouthTop", [
      [0.00, [0, 0, 0], [0, 0, 0]],
      [0.18, [28, 0, 0], [0, 0, 0]],
      [0.36, [0, 0, 0], [0, 0, 0]]
    ]],
    ["MouthBottom", [
      [0.00, [0, 0, 0], [0, 0, 0]],
      [0.18, [-28, 0, 0], [0, 0, 0]],
      [0.36, [0, 0, 0], [0, 0, 0]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="BodyRoot", t=[0, 0, 0], r=[0, 0, 0]) {

    // Solid rear skull (seamless and always closed at the back -Y)
    color([1.0, 0.86, 0.05], roughness=0.25, metalness=0.05, specularIntensity=0.8, emissive=[0.08, 0.06, 0.0], emissiveIntensity=0.4, $asa=45) {
      intersection() {
        sphere(r=0.58);
        translate([-0.8, -0.8, -0.8])
          cube([1.6, 0.82, 1.6]);
      }
    }

    // Upper front jaw (Forward is +Y, Z >= 0)
    bone(name="MouthTop", t=[0, 0, 0], r=[0, 0, 0]) {
      color([1.0, 0.86, 0.05], roughness=0.25, metalness=0.05, specularIntensity=0.8, emissive=[0.08, 0.06, 0.0], emissiveIntensity=0.4, $asa=45) {
        intersection() {
          sphere(r=0.58);
          translate([-0.8, 0.0, 0.0])
            cube([1.6, 0.8, 0.8]);
        }
      }

      // Big expressive white eye sclera
      color([0.96, 0.97, 1.0], roughness=0.15, metalness=0.0, specularIntensity=1.0, $asa=35) {
        translate([-0.22, 0.32, 0.32])
          scale([0.16, 0.16, 0.22])
            sphere(r=1.0, $fn=28);
        translate([0.22, 0.32, 0.32])
          scale([0.16, 0.16, 0.22])
            sphere(r=1.0, $fn=28);
      }

      // Large dark arcade pupils facing Forward (+Y)
      color([0.04, 0.08, 0.20], roughness=0.1, metalness=0.1, specularIntensity=1.0, emissive=[0.01, 0.02, 0.05], emissiveIntensity=0.5) {
        translate([-0.22, 0.45, 0.33])
          scale([0.10, 0.08, 0.15])
            sphere(r=1.0, $fn=24);
        translate([0.22, 0.45, 0.33])
          scale([0.10, 0.08, 0.15])
            sphere(r=1.0, $fn=24);
      }

      // Glossy bright highlights (upper glint + lower sparkle)
      color([1.0, 1.0, 1.0], roughness=0.05, metalness=0.0, emissive=[1.0, 1.0, 1.0], emissiveIntensity=1.5) {
        // Main glint
        translate([-0.20, 0.50, 0.38])
          sphere(r=0.042, $fn=16);
        translate([0.24, 0.50, 0.38])
          sphere(r=0.042, $fn=16);

        // Secondary sparkle
        translate([-0.25, 0.49, 0.29])
          sphere(r=0.022, $fn=14);
        translate([0.19, 0.49, 0.29])
          sphere(r=0.022, $fn=14);
      }
    }

    // Lower front jaw (Forward is +Y, Z <= 0)
    bone(name="MouthBottom", t=[0, 0, 0], r=[0, 0, 0]) {
      color([1.0, 0.86, 0.05], roughness=0.25, metalness=0.05, specularIntensity=0.8, emissive=[0.08, 0.06, 0.0], emissiveIntensity=0.4, $asa=45) {
        intersection() {
          sphere(r=0.58);
          translate([-0.8, 0.0, -0.8])
            cube([1.6, 0.8, 0.8]);
        }
      }

      // Interior mouth lining at the front
      color([0.75, 0.1, 0.1], roughness=0.5, metalness=0.0) {
        translate([0.0, 0.20, -0.02])
          cylinder(r=0.36, h=0.03, center=true, $fn=24);
      }
    }
  }
}
```

### packman_3d/assets/models/ghost_blinky.scad
```openscad
$fn = 36;
$asa = 45;

anim_data = [
  ["Float", [
    ["GhostBody", [
      [0.00, [0, 0, 0], [0, 0, 0.00]],
      [0.35, [0, 0, 0], [0, 0, 0.06]],
      [0.70, [0, 0, 0], [0, 0, 0.00]]
    ]],
    ["GhostEyes", [
      [0.00, [0, 0, 0]],
      [0.35, [0, 0, 5]],
      [0.70, [0, 0, 0]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="GhostRoot", t=[0, 0, 0], r=[0, 0, 0]) {
    bone(name="GhostBody", t=[0, 0, 0], r=[0, 0, 0]) {
      // Red Chaser ghost body
      color([0.95, 0.12, 0.12], roughness=0.3, metalness=0.08, emissive=[0.25, 0.03, 0.03], emissiveIntensity=0.5, $asa=45) {
        translate([0, 0, 0.22])
          sphere(r=0.56);
        translate([0, 0, -0.16])
          cylinder(r=0.56, h=0.38, center=false);
        for (i = [0:3]) {
          rotate([0, 0, i * 90 + 45])
            translate([0.0, 0.35, -0.24])
              cylinder(r1=0.16, r2=0.20, h=0.22, center=true, $fn=18);
        }
      }

      // Eyes facing Forward (+Y)
      bone(name="GhostEyes", t=[0, 0, 0], r=[0, 0, 0]) {
        color([0.96, 0.96, 1.0], roughness=0.2, metalness=0.0, $asa=35) {
          translate([-0.20, 0.38, 0.26])
            scale([0.18, 0.22, 0.24])
              sphere(r=1.0);
          translate([0.20, 0.38, 0.26])
            scale([0.18, 0.22, 0.24])
              sphere(r=1.0);
        }
        color([0.1, 0.25, 0.9], roughness=0.15, metalness=0.1, emissive=[0.05, 0.12, 0.4], emissiveIntensity=0.6) {
          translate([-0.20, 0.54, 0.26])
            scale([0.11, 0.11, 0.14])
              sphere(r=1.0);
          translate([0.20, 0.54, 0.26])
            scale([0.11, 0.11, 0.14])
              sphere(r=1.0);
        }
      }
    }
  }
}
```

### packman_3d/assets/models/ghost_pinky.scad
```openscad
$fn = 36;
$asa = 45;

anim_data = [
  ["Float", [
    ["GhostBody", [
      [0.00, [0, 0, 0], [0, 0, 0.00]],
      [0.35, [0, 0, 0], [0, 0, 0.06]],
      [0.70, [0, 0, 0], [0, 0, 0.00]]
    ]],
    ["GhostEyes", [
      [0.00, [0, 0, 0]],
      [0.35, [0, 0, 5]],
      [0.70, [0, 0, 0]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="GhostRoot", t=[0, 0, 0], r=[0, 0, 0]) {
    bone(name="GhostBody", t=[0, 0, 0], r=[0, 0, 0]) {
      // Pink Ambusher ghost body
      color([1.0, 0.55, 0.78], roughness=0.3, metalness=0.08, emissive=[0.25, 0.08, 0.16], emissiveIntensity=0.5, $asa=45) {
        translate([0, 0, 0.22])
          sphere(r=0.56);
        translate([0, 0, -0.16])
          cylinder(r=0.56, h=0.38, center=false);
        for (i = [0:3]) {
          rotate([0, 0, i * 90 + 45])
            translate([0.0, 0.35, -0.24])
              cylinder(r1=0.16, r2=0.20, h=0.22, center=true, $fn=18);
        }
      }

      // Eyes facing Forward (+Y)
      bone(name="GhostEyes", t=[0, 0, 0], r=[0, 0, 0]) {
        color([0.96, 0.96, 1.0], roughness=0.2, metalness=0.0, $asa=35) {
          translate([-0.20, 0.38, 0.26])
            scale([0.18, 0.22, 0.24])
              sphere(r=1.0);
          translate([0.20, 0.38, 0.26])
            scale([0.18, 0.22, 0.24])
              sphere(r=1.0);
        }
        color([0.1, 0.25, 0.9], roughness=0.15, metalness=0.1, emissive=[0.05, 0.12, 0.4], emissiveIntensity=0.6) {
          translate([-0.20, 0.54, 0.26])
            scale([0.11, 0.11, 0.14])
              sphere(r=1.0);
          translate([0.20, 0.54, 0.26])
            scale([0.11, 0.11, 0.14])
              sphere(r=1.0);
        }
      }
    }
  }
}
```

### packman_3d/assets/models/ghost_inky.scad
```openscad
$fn = 36;
$asa = 45;

anim_data = [
  ["Float", [
    ["GhostBody", [
      [0.00, [0, 0, 0], [0, 0, 0.00]],
      [0.35, [0, 0, 0], [0, 0, 0.06]],
      [0.70, [0, 0, 0], [0, 0, 0.00]]
    ]],
    ["GhostEyes", [
      [0.00, [0, 0, 0]],
      [0.35, [0, 0, 5]],
      [0.70, [0, 0, 0]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="GhostRoot", t=[0, 0, 0], r=[0, 0, 0]) {
    bone(name="GhostBody", t=[0, 0, 0], r=[0, 0, 0]) {
      // Cyan Flanker ghost body
      color([0.05, 0.88, 0.98], roughness=0.3, metalness=0.08, emissive=[0.02, 0.22, 0.28], emissiveIntensity=0.5, $asa=45) {
        translate([0, 0, 0.22])
          sphere(r=0.56);
        translate([0, 0, -0.16])
          cylinder(r=0.56, h=0.38, center=false);
        for (i = [0:3]) {
          rotate([0, 0, i * 90 + 45])
            translate([0.0, 0.35, -0.24])
              cylinder(r1=0.16, r2=0.20, h=0.22, center=true, $fn=18);
        }
      }

      // Eyes facing Forward (+Y)
      bone(name="GhostEyes", t=[0, 0, 0], r=[0, 0, 0]) {
        color([0.96, 0.96, 1.0], roughness=0.2, metalness=0.0, $asa=35) {
          translate([-0.20, 0.38, 0.26])
            scale([0.18, 0.22, 0.24])
              sphere(r=1.0);
          translate([0.20, 0.38, 0.26])
            scale([0.18, 0.22, 0.24])
              sphere(r=1.0);
        }
        color([0.1, 0.25, 0.9], roughness=0.15, metalness=0.1, emissive=[0.05, 0.12, 0.4], emissiveIntensity=0.6) {
          translate([-0.20, 0.54, 0.26])
            scale([0.11, 0.11, 0.14])
              sphere(r=1.0);
          translate([0.20, 0.54, 0.26])
            scale([0.11, 0.11, 0.14])
              sphere(r=1.0);
        }
      }
    }
  }
}
```

### packman_3d/assets/models/ghost_clyde.scad
```openscad
$fn = 36;
$asa = 45;

anim_data = [
  ["Float", [
    ["GhostBody", [
      [0.00, [0, 0, 0], [0, 0, 0.00]],
      [0.35, [0, 0, 0], [0, 0, 0.06]],
      [0.70, [0, 0, 0], [0, 0, 0.00]]
    ]],
    ["GhostEyes", [
      [0.00, [0, 0, 0]],
      [0.35, [0, 0, 5]],
      [0.70, [0, 0, 0]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="GhostRoot", t=[0, 0, 0], r=[0, 0, 0]) {
    bone(name="GhostBody", t=[0, 0, 0], r=[0, 0, 0]) {
      // Orange Patrol ghost body
      color([1.0, 0.62, 0.05], roughness=0.3, metalness=0.08, emissive=[0.26, 0.14, 0.02], emissiveIntensity=0.5, $asa=45) {
        translate([0, 0, 0.22])
          sphere(r=0.56);
        translate([0, 0, -0.16])
          cylinder(r=0.56, h=0.38, center=false);
        for (i = [0:3]) {
          rotate([0, 0, i * 90 + 45])
            translate([0.0, 0.35, -0.24])
              cylinder(r1=0.16, r2=0.20, h=0.22, center=true, $fn=18);
        }
      }

      // Eyes facing Forward (+Y)
      bone(name="GhostEyes", t=[0, 0, 0], r=[0, 0, 0]) {
        color([0.96, 0.96, 1.0], roughness=0.2, metalness=0.0, $asa=35) {
          translate([-0.20, 0.38, 0.26])
            scale([0.18, 0.22, 0.24])
              sphere(r=1.0);
          translate([0.20, 0.38, 0.26])
            scale([0.18, 0.22, 0.24])
              sphere(r=1.0);
        }
        color([0.1, 0.25, 0.9], roughness=0.15, metalness=0.1, emissive=[0.05, 0.12, 0.4], emissiveIntensity=0.6) {
          translate([-0.20, 0.54, 0.26])
            scale([0.11, 0.11, 0.14])
              sphere(r=1.0);
          translate([0.20, 0.54, 0.26])
            scale([0.11, 0.11, 0.14])
              sphere(r=1.0);
        }
      }
    }
  }
}
```

### packman_3d/assets/models/ghost_frightened.scad
```openscad
$fn = 36;
$asa = 45;

anim_data = [
  ["Panic", [
    ["PanicBody", [
      [0.00, [0, 0, -5], [0, 0, 0.00]],
      [0.15, [0, 0, 5], [0, 0, 0.04]],
      [0.30, [0, 0, -5], [0, 0, 0.00]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="PanicRoot", t=[0, 0, 0], r=[0, 0, 0]) {
    bone(name="PanicBody", t=[0, 0, 0], r=[0, 0, 0]) {
      // Vulnerable Electric Blue Body
      color([0.12, 0.22, 0.85], roughness=0.3, metalness=0.08, emissive=[0.08, 0.15, 0.50], emissiveIntensity=0.8, $asa=45) {
        translate([0, 0, 0.22])
          sphere(r=0.56);
        translate([0, 0, -0.16])
          cylinder(r=0.56, h=0.38, center=false);
        for (i = [0:3]) {
          rotate([0, 0, i * 90 + 45])
            translate([0.0, 0.35, -0.24])
              cylinder(r1=0.16, r2=0.20, h=0.22, center=true, $fn=18);
        }
      }

      // Yellow frightened dots
      color([1.0, 0.92, 0.2], roughness=0.15, metalness=0.0, emissive=[0.5, 0.45, 0.1], emissiveIntensity=0.8) {
        translate([-0.16, 0.50, 0.26])
          sphere(r=0.085);
        translate([0.16, 0.50, 0.26])
          sphere(r=0.085);
      }

      // Wavy zigzag frightened mouth facing Forward (+Y)
      color([0.85, 0.95, 1.0], roughness=0.15, metalness=0.0, emissive=[0.4, 0.45, 0.5], emissiveIntensity=0.8) {
        translate([0.0, 0.52, 0.08])
          cube([0.36, 0.06, 0.05], center=true);
        translate([-0.12, 0.52, 0.04])
          cube([0.08, 0.06, 0.08], center=true);
        translate([0.12, 0.52, 0.04])
          cube([0.08, 0.06, 0.08], center=true);
      }
    }
  }
}
```

### packman_3d/assets/models/ghost_flash.scad
```openscad
$fn = 36;
$asa = 45;

anim_data = [
  ["Panic", [
    ["FlashBody", [
      [0.00, [0, 0, -5], [0, 0, 0.00]],
      [0.15, [0, 0, 5], [0, 0, 0.04]],
      [0.30, [0, 0, -5], [0, 0, 0.00]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="FlashRoot", t=[0, 0, 0], r=[0, 0, 0]) {
    bone(name="FlashBody", t=[0, 0, 0], r=[0, 0, 0]) {
      // Flashing White Warning Ghost
      color([0.95, 0.98, 1.0], roughness=0.25, metalness=0.08, emissive=[0.5, 0.55, 0.6], emissiveIntensity=0.9, $asa=45) {
        translate([0, 0, 0.22])
          sphere(r=0.56);
        translate([0, 0, -0.16])
          cylinder(r=0.56, h=0.38, center=false);
        for (i = [0:3]) {
          rotate([0, 0, i * 90 + 45])
            translate([0.0, 0.35, -0.24])
              cylinder(r1=0.16, r2=0.20, h=0.22, center=true, $fn=18);
        }
      }

      // Red panic eyes
      color([1.0, 0.15, 0.15], roughness=0.15, metalness=0.0, emissive=[0.6, 0.1, 0.1], emissiveIntensity=0.9) {
        translate([-0.16, 0.50, 0.26])
          sphere(r=0.085);
        translate([0.16, 0.50, 0.26])
          sphere(r=0.085);
      }

      // Red panic mouth facing Forward (+Y)
      color([1.0, 0.15, 0.15], roughness=0.15, metalness=0.0, emissive=[0.6, 0.1, 0.1], emissiveIntensity=0.9) {
        translate([0.0, 0.52, 0.08])
          cube([0.36, 0.06, 0.05], center=true);
        translate([-0.12, 0.52, 0.04])
          cube([0.08, 0.06, 0.08], center=true);
        translate([0.12, 0.52, 0.04])
          cube([0.08, 0.06, 0.08], center=true);
      }
    }
  }
}
```

### packman_3d/assets/models/ghost_eyes.scad
```openscad
$fn = 28;
$asa = 35;

anim_data = [
  ["Look", [
    ["EyesRoot", [
      [0.00, [0, 0, 0], [0, 0, 0.00]],
      [0.25, [0, 0, 0], [0, 0, 0.05]],
      [0.50, [0, 0, 0], [0, 0, 0.00]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="EyesRoot", t=[0, 0, 0], r=[0, 0, 0]) {
    // Eaten Ghost Floating Eyes facing Forward (+Y)
    color([0.96, 0.96, 1.0], roughness=0.15, metalness=0.0, $asa=35) {
      translate([-0.20, 0.22, 0.26])
        scale([0.18, 0.22, 0.24])
          sphere(r=1.0);
      translate([0.20, 0.22, 0.26])
        scale([0.18, 0.22, 0.24])
          sphere(r=1.0);
    }
    color([0.1, 0.25, 0.95], roughness=0.1, metalness=0.2, emissive=[0.08, 0.2, 0.6], emissiveIntensity=0.8) {
      translate([-0.20, 0.40, 0.26])
        scale([0.11, 0.11, 0.14])
          sphere(r=1.0);
      translate([0.20, 0.40, 0.26])
        scale([0.11, 0.11, 0.14])
          sphere(r=1.0);
    }
  }
}
```

### packman_3d/assets/models/gate.scad
```openscad
$fn = 24;
$asa = 45;

// Cyber Energy Gate: Obsidian brackets and glowing cyan barrier beam
color([0.08, 0.11, 0.18], roughness=0.3, metalness=0.85, $asa=35) {
  translate([-0.54, 0, 0.15])
    cube([0.12, 0.22, 0.30], center=true);
  translate([0.54, 0, 0.15])
    cube([0.12, 0.22, 0.30], center=true);
}

// Glowing cyan barrier beam (clean balanced glow)
color([0.0, 0.75, 0.9], roughness=0.2, metalness=0.1, emissive=[0.0, 0.5, 0.7], emissiveIntensity=0.8) {
  translate([0, 0, 0.18])
    cube([1.04, 0.06, 0.10], center=true);
  translate([0, 0, 0.10])
    cube([0.96, 0.04, 0.04], center=true);
}
```

### packman_3d/assets/models/pellet.scad
```openscad
$fn = 20;
$asa = 45;

anim_data = [
  ["Spin", [
    ["Gem", [
      [0.0, [0, 0, 0], [0, 0, 0]],
      [0.5, [0, 0, 90], [0, 0, 0]],
      [1.0, [0, 0, 180], [0, 0, 0]],
      [1.5, [0, 0, 270], [0, 0, 0]],
      [2.0, [0, 0, 360], [0, 0, 0]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="Gem", t=[0, 0, 0], r=[0, 0, 0]) {
    color([1.0, 0.88, 0.3], roughness=0.2, metalness=0.1, emissive=[0.5, 0.4, 0.08], emissiveIntensity=0.6, $asa=45) {
      sphere(r=0.12);
      cylinder(r=0.14, h=0.05, center=true);
    }
  }
}
```

### packman_3d/assets/models/power_pellet.scad
```openscad
$fn = 28;
$asa = 45;

anim_data = [
  ["PowerSpin", [
    ["Core", [
      [0.0, [0, 0, 0], [0, 0, 0.00]],
      [0.5, [0, 0, 0], [0, 0, 0.04]],
      [1.0, [0, 0, 0], [0, 0, 0.00]]
    ]],
    ["OuterRing", [
      [0.0, [0, 0, 0], [0, 0, 0]],
      [0.5, [90, 0, 0], [0, 0, 0]],
      [1.0, [180, 0, 0], [0, 0, 0]],
      [1.5, [270, 0, 0], [0, 0, 0]],
      [2.0, [360, 0, 0], [0, 0, 0]]
    ]],
    ["InnerRing", [
      [0.0, [0, 0, 0], [0, 0, 0]],
      [0.5, [0, 90, 0], [0, 0, 0]],
      [1.0, [0, 180, 0], [0, 0, 0]],
      [1.5, [0, 270, 0], [0, 0, 0]],
      [2.0, [0, 360, 0], [0, 0, 0]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="PowerRoot", t=[0, 0, 0], r=[0, 0, 0]) {
    bone(name="Core", t=[0, 0, 0], r=[0, 0, 0]) {
      color([1.0, 0.95, 0.65], roughness=0.15, metalness=0.05, emissive=[0.6, 0.55, 0.2], emissiveIntensity=0.9, $asa=45) {
        sphere(r=0.25);
      }
    }

    bone(name="OuterRing", t=[0, 0, 0], r=[0, 0, 0]) {
      color([0.2, 0.8, 1.0], roughness=0.2, metalness=0.8, emissive=[0.0, 0.3, 0.6], emissiveIntensity=0.6) {
        difference() {
          cylinder(r=0.38, h=0.04, center=true);
          cylinder(r=0.32, h=0.06, center=true);
        }
      }
    }

    bone(name="InnerRing", t=[0, 0, 0], r=[0, 0, 0]) {
      color([1.0, 0.45, 0.15], roughness=0.2, metalness=0.8, emissive=[0.5, 0.15, 0.0], emissiveIntensity=0.6) {
        difference() {
          cylinder(r=0.30, h=0.03, center=true);
          cylinder(r=0.25, h=0.05, center=true);
        }
      }
    }
  }
}
```

### packman_3d/assets/models/wall.scad
```openscad
$fn = 24;
$asa = 35;

// Cyber Maze Wall block (1.16 x 1.16 x 0.88)
color([0.08, 0.11, 0.18], roughness=0.3, metalness=0.85, $asa=35) {
  translate([0, 0, 0.44])
    cube([1.16, 1.16, 0.88], center=true);
}

// Crisp subtle glowing cyan trim (balanced accent without wash-out)
color([0.0, 0.8, 0.95], roughness=0.2, metalness=0.1, emissive=[0.0, 0.4, 0.55], emissiveIntensity=0.5) {
  translate([0, 0, 0.89])
    difference() {
      cube([1.18, 1.18, 0.05], center=true);
      cube([0.98, 0.98, 0.08], center=true);
    }
  translate([0, 0, 0.88])
    cylinder(r=0.16, h=0.06, center=true);
}

// Corner chrome conduits
color([0.85, 0.90, 0.95], roughness=0.1, metalness=0.95) {
  for (dx = [-0.52, 0.52]) {
    for (dy = [-0.52, 0.52]) {
      translate([dx, dy, 0.44])
        cylinder(r=0.045, h=0.90, center=true);
    }
  }
}
```

### packman_3d/assets/models/cherry.scad
```openscad
$fn = 28;
$asa = 45;

anim_data = [
  ["Float", [
    ["CherryRoot", [
      [0.0, [0, 0, 0], [0, 0, 0.0]],
      [0.4, [0, 0, 15], [0, 0, 0.06]],
      [0.8, [0, 0, 0], [0, 0, 0.0]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="CherryRoot", t=[0, 0, 0], r=[0, 0, 0]) {
    // Twin glossy red cherries
    color([0.95, 0.05, 0.15], roughness=0.15, metalness=0.05, specularIntensity=1.2, emissive=[0.2, 0.0, 0.03], emissiveIntensity=0.5, $asa=45) {
      translate([-0.16, 0.0, 0.18])
        sphere(r=0.18);
      translate([0.16, 0.06, 0.18])
        sphere(r=0.18);
    }
    // Wooden stems
    color([0.45, 0.28, 0.12], roughness=0.6, metalness=0.0) {
      translate([-0.08, 0.01, 0.36])
        rotate([0, 20, 0])
          cylinder(r=0.022, h=0.28, center=true);
      translate([0.08, 0.04, 0.36])
        rotate([0, -20, 0])
          cylinder(r=0.022, h=0.28, center=true);
    }
    // Green leaf
    color([0.1, 0.85, 0.2], roughness=0.3, metalness=0.0, emissive=[0.02, 0.2, 0.04], emissiveIntensity=0.5, $asa=35) {
      translate([0.06, 0.04, 0.48])
        rotate([15, 30, 25])
          scale([0.14, 0.07, 0.03])
            sphere(r=1.0);
    }
  }
}
```

### packman_3d/scripts/audio_synth.gd
```gdscript
class_name AudioSynth
extends Node

const SAMPLE_RATE: int = 44100

static func sine(phase: float) -> float:
	return sin(phase)

static func square(phase: float, duty: float = 0.5) -> float:
	var t = fposmod(phase / (2.0 * PI), 1.0)
	return 1.0 if t < duty else -1.0

static func triangle(phase: float) -> float:
	var t = fposmod(phase / (2.0 * PI), 1.0)
	return 2.0 * abs(2.0 * (t - floor(t + 0.5))) - 1.0

static func create_stream_from_samples(samples: PackedByteArray) -> AudioStreamWAV:
	var wav = AudioStreamWAV.new()
	wav.format = AudioStreamWAV.FORMAT_16_BITS
	wav.mix_rate = SAMPLE_RATE
	wav.stereo = false
	wav.data = samples
	return wav

static func generate_waka_0() -> AudioStreamWAV:
	var dur = 0.085
	var total = int(dur * SAMPLE_RATE)
	var bytes = PackedByteArray()
	bytes.resize(total * 2)
	var phase = 0.0
	for i in range(total):
		var frac = float(i) / float(total)
		var freq = 220.0 + (480.0 - 220.0) * frac
		phase += 2.0 * PI * freq / SAMPLE_RATE
		var env = minf(frac / 0.15, 1.0) * maxf(1.0 - frac, 0.0)
		var val = 0.5 * square(phase, 0.45) + 0.5 * triangle(phase)
		var sample = clampi(int(val * env * 14000.0), -32768, 32767)
		bytes.encode_s16(i * 2, sample)
	return create_stream_from_samples(bytes)

static func generate_waka_1() -> AudioStreamWAV:
	var dur = 0.085
	var total = int(dur * SAMPLE_RATE)
	var bytes = PackedByteArray()
	bytes.resize(total * 2)
	var phase = 0.0
	for i in range(total):
		var frac = float(i) / float(total)
		var freq = 480.0 - (480.0 - 190.0) * frac
		phase += 2.0 * PI * freq / SAMPLE_RATE
		var env = minf(frac / 0.15, 1.0) * maxf(1.0 - frac, 0.0)
		var val = 0.5 * square(phase, 0.45) + 0.5 * triangle(phase)
		var sample = clampi(int(val * env * 14000.0), -32768, 32767)
		bytes.encode_s16(i * 2, sample)
	return create_stream_from_samples(bytes)

static func generate_power_pellet() -> AudioStreamWAV:
	var dur = 0.25
	var total = int(dur * SAMPLE_RATE)
	var bytes = PackedByteArray()
	bytes.resize(total * 2)
	var phase = 0.0
	for i in range(total):
		var t = float(i) / SAMPLE_RATE
		var freq = 80.0 + 40.0 * sin(2.0 * PI * 12.0 * t)
		phase += 2.0 * PI * freq / SAMPLE_RATE
		var val = triangle(phase)
		var sample = clampi(int(val * 16000.0), -32768, 32767)
		bytes.encode_s16(i * 2, sample)
	return create_stream_from_samples(bytes)

static func generate_eat_ghost() -> AudioStreamWAV:
	var notes = [523.25, 659.25, 783.99, 1046.50, 1318.51, 1567.98]
	var note_dur = 0.07
	var note_samples = int(note_dur * SAMPLE_RATE)
	var total = note_samples * notes.size()
	var bytes = PackedByteArray()
	bytes.resize(total * 2)
	var byte_idx = 0
	for freq in notes:
		var phase = 0.0
		for i in range(note_samples):
			var frac = float(i) / float(note_samples)
			phase += 2.0 * PI * freq / SAMPLE_RATE
			var env = pow(1.0 - frac, 1.4)
			var val = 0.6 * square(phase, 0.35) + 0.4 * sine(phase)
			var sample = clampi(int(val * env * 18000.0), -32768, 32767)
			bytes.encode_s16(byte_idx, sample)
			byte_idx += 2
	return create_stream_from_samples(bytes)

static func generate_death() -> AudioStreamWAV:
	var steps = 14
	var step_dur = 0.08
	var count = int(step_dur * SAMPLE_RATE)
	var total = count * steps
	var bytes = PackedByteArray()
	bytes.resize(total * 2)
	var byte_idx = 0
	for s in range(steps):
		var freq = 820.0 * pow(0.86, s)
		var phase = 0.0
		for i in range(count):
			var t = float(i) / float(count)
			var pitch = freq * (1.0 - t * 0.15)
			phase += 2.0 * PI * pitch / SAMPLE_RATE
			var env = 1.0 - t * 0.5
			var val = square(phase, 0.5)
			var sample = clampi(int(val * env * 15000.0), -32768, 32767)
			bytes.encode_s16(byte_idx, sample)
			byte_idx += 2
	return create_stream_from_samples(bytes)

static func generate_fruit() -> AudioStreamWAV:
	var freqs = [880.0, 1320.0]
	var durs = [0.12, 0.20]
	var total = int((durs[0] + durs[1]) * SAMPLE_RATE)
	var bytes = PackedByteArray()
	bytes.resize(total * 2)
	var byte_idx = 0
	for idx in range(freqs.size()):
		var freq = freqs[idx]
		var dur = durs[idx]
		var count = int(dur * SAMPLE_RATE)
		var phase = 0.0
		for i in range(count):
			var t = float(i) / float(count)
			phase += 2.0 * PI * freq / SAMPLE_RATE
			var env = pow(1.0 - t, 1.2)
			var val = 0.7 * sine(phase) + 0.3 * sine(phase * 2.0)
			var sample = clampi(int(val * env * 19000.0), -32768, 32767)
			bytes.encode_s16(byte_idx, sample)
			byte_idx += 2
	return create_stream_from_samples(bytes)

static func generate_game_start() -> AudioStreamWAV:
	var melody = [
		Vector2(493.88, 0.12), Vector2(987.77, 0.12), Vector2(739.99, 0.12), Vector2(622.25, 0.12),
		Vector2(987.77, 0.12), Vector2(739.99, 0.16), Vector2(622.25, 0.22),
		Vector2(523.25, 0.12), Vector2(1046.50, 0.12), Vector2(783.99, 0.12), Vector2(659.25, 0.12),
		Vector2(1046.50, 0.12), Vector2(783.99, 0.16), Vector2(659.25, 0.22),
		Vector2(493.88, 0.12), Vector2(987.77, 0.12), Vector2(739.99, 0.12), Vector2(622.25, 0.12),
		Vector2(987.77, 0.12), Vector2(739.99, 0.16), Vector2(622.25, 0.22),
		Vector2(622.25, 0.08), Vector2(659.25, 0.08), Vector2(698.46, 0.08), Vector2(739.99, 0.08),
		Vector2(783.99, 0.08), Vector2(830.61, 0.08), Vector2(880.00, 0.08), Vector2(987.77, 0.35)
	]
	var total_dur = 0.0
	for item in melody:
		total_dur += item.y
	var total_samples = int(total_dur * SAMPLE_RATE)
	var bytes = PackedByteArray()
	bytes.resize(total_samples * 2)
	var byte_idx = 0
	for item in melody:
		var freq = item.x
		var dur = item.y
		var count = int(dur * SAMPLE_RATE)
		var phase = 0.0
		for i in range(count):
			var frac = float(i) / float(count)
			phase += 2.0 * PI * freq / SAMPLE_RATE
			var env = minf(1.0 - frac * 0.25, 1.0)
			var val = square(phase, 0.25)
			var sample = clampi(int(val * env * 14000.0), -32768, 32767)
			bytes.encode_s16(byte_idx, sample)
			byte_idx += 2
	return create_stream_from_samples(bytes)
```

### packman_3d/scripts/maze_data.gd
```gdscript
class_name MazeData
extends RefCounted

const MAP_WIDTH: int = 21
const MAP_HEIGHT: int = 21
const CELL_SIZE: float = 1.2

const TILES: Array[String] = [
	"WWWWWWWWWWWWWWWWWWWWW", # 0
	"WO........W........OW", # 1
	"W.WWW.WWW.W.WWW.WWW.W", # 2
	"W.WWW.WWW.W.WWW.WWW.W", # 3
	"W...................W", # 4
	"W.WWW.W.WWWWW.W.WWW.W", # 5
	"W.....W...W...W.....W", # 6
	"WWWWW.WWW   WWW.WWWWW", # 7: Open corridor 9,10,11 (Blinky starts at 7, 10!)
	"    W.W WWDWW W.W    ", # 8: Top of Ghost House; Cyber Gate 'D' at index 10
	"WWWWW.W GGGGG W.WWWWW", # 9: Inside Ghost House
	"     .  GGGGG  .     ", # 10: Tunnel row & Ghost House center
	"WWWWW.W GGGGG W.WWWWW", # 11: Inside Ghost House
	"    W.W WWWWW W.W    ", # 12: Bottom wall of Ghost House
	"WWWWW.W       W.WWWWW", # 13: Corridor beneath Ghost House (Cherry spawns at 13, 10!)
	"W.........W.........W", # 14
	"W.WWW.WWW.W.WWW.WWW.W", # 15
	"WO..W.....P.....W..OW", # 16: Pacman start at (16, 10)
	"WWW.W.W.WWWWW.W.W.WWW", # 17
	"W.....W...W...W.....W", # 18
	"W.WWWWWWW.W.WWWWWWW.W", # 19
	"WWWWWWWWWWWWWWWWWWWWW"  # 20
]

enum Dir { NONE, UP, DOWN, LEFT, RIGHT }

static func dir_to_offset(dir: Dir) -> Vector2i:
	match dir:
		Dir.UP: return Vector2i(-1, 0)
		Dir.DOWN: return Vector2i(1, 0)
		Dir.LEFT: return Vector2i(0, -1)
		Dir.RIGHT: return Vector2i(0, 1)
		_: return Vector2i.ZERO

static func opposite_dir(dir: Dir) -> Dir:
	match dir:
		Dir.UP: return Dir.DOWN
		Dir.DOWN: return Dir.UP
		Dir.LEFT: return Dir.RIGHT
		Dir.RIGHT: return Dir.LEFT
		_: return Dir.NONE

static func dir_to_vector3(dir: Dir) -> Vector3:
	match dir:
		Dir.UP: return Vector3.FORWARD
		Dir.DOWN: return Vector3.BACK
		Dir.LEFT: return Vector3.LEFT
		Dir.RIGHT: return Vector3.RIGHT
		_: return Vector3.ZERO

static func grid_to_world(row: int, col: int) -> Vector3:
	var half_w = float(MAP_WIDTH - 1) * 0.5
	var half_h = float(MAP_HEIGHT - 1) * 0.5
	return Vector3(
		(float(col) - half_w) * CELL_SIZE,
		0.0,
		(float(row) - half_h) * CELL_SIZE
	)

static func is_wall(row: int, col: int) -> bool:
	if row < 0 or row >= MAP_HEIGHT or col < 0 or col >= MAP_WIDTH:
		return true
	return TILES[row][col] == "W"
```

### packman_3d/scripts/floating_text.gd
```gdscript
class_name FloatingText
extends Label3D

var lifetime: float = 1.2

func _ready() -> void:
	billboard = BaseMaterial3D.BILLBOARD_ENABLED
	modulate = Color(0.2, 1.0, 0.4)
	font_size = 40
	outline_size = 10
	outline_modulate = Color.BLACK

func _process(delta: float) -> void:
	lifetime -= delta
	position.y += delta * 1.2
	modulate.a = clampf(lifetime / 0.5, 0.0, 1.0)
	if lifetime <= 0.0:
		queue_free()
```

### packman_3d/scripts/pellet.gd
```gdscript
class_name Pellet
extends Node3D

@export var is_power: bool = false
var seed_val: float = 0.0
var base_y: float = 0.25

var outer_ring: Node3D = null
var inner_ring: Node3D = null

func setup(power: bool, row: int, col: int) -> void:
	is_power = power
	seed_val = float(row * 31 + col)
	base_y = 0.35 if is_power else 0.25

func _ready() -> void:
	start_looping_animations(self)
	find_rings(self)

func find_rings(node: Node) -> void:
	if node.name.containsn("OuterRing"):
		outer_ring = node as Node3D
	elif node.name.containsn("InnerRing"):
		inner_ring = node as Node3D
	for child in node.get_children():
		find_rings(child)

func start_looping_animations(node: Node) -> void:
	if node is AnimationPlayer:
		for anim_name in node.get_animation_list():
			var anim = node.get_animation(anim_name)
			if anim:
				anim.loop_mode = Animation.LOOP_LINEAR
		var list = node.get_animation_list()
		if list.size() > 0:
			node.play(list[0])
	for child in node.get_children():
		start_looping_animations(child)

func _process(delta: float) -> void:
	var t = Time.get_ticks_msec() * 0.001
	if is_power:
		var pulse = 1.0 + sin(t * 5.0 + seed_val) * 0.12
		scale = Vector3.ONE * pulse
		position.y = base_y + sin(t * 3.0 + seed_val) * 0.04
		rotate_y(delta * 1.5)
		if is_instance_valid(outer_ring):
			outer_ring.rotate_x(delta * 2.5)
		if is_instance_valid(inner_ring):
			inner_ring.rotate_z(delta * 3.5)
	else:
		rotate_y(delta * 2.5)
		position.y = base_y + sin(t * 3.5 + seed_val) * 0.02
```

### packman_3d/scripts/cherry.gd
```gdscript
class_name CherryFruit
extends Node3D

var lifetime: float = 20.0
var base_y: float = 0.38

func _ready() -> void:
	start_looping_animations(self)

func start_looping_animations(node: Node) -> void:
	if node is AnimationPlayer:
		for anim_name in node.get_animation_list():
			var anim = node.get_animation(anim_name)
			if anim:
				anim.loop_mode = Animation.LOOP_LINEAR
		var list = node.get_animation_list()
		if list.size() > 0:
			node.play(list[0])
	for child in node.get_children():
		start_looping_animations(child)

func _process(delta: float) -> void:
	var t = Time.get_ticks_msec() * 0.001
	rotate_y(delta * 1.8)
	position.y = base_y + sin(t * 3.0) * 0.05
```

### packman_3d/scripts/pacman.gd
```gdscript
class_name PacmanCharacter
extends Node3D

var grid_pos: Vector2i = Vector2i(16, 10)
var next_tile: Vector2i = Vector2i(16, 10)
var progress: float = 1.0
var speed: float = 3.2
var current_dir: MazeData.Dir = MazeData.Dir.NONE
var queued_dir: MazeData.Dir = MazeData.Dir.NONE
var chomp_flip: bool = false

@onready var aura_light: OmniLight3D = $AuraLight
var anim_player: AnimationPlayer = null
var mouth_top: Node3D = null
var mouth_bottom: Node3D = null

func _ready() -> void:
	find_animation_and_mouth(self)
	if anim_player:
		for anim_name in anim_player.get_animation_list():
			var anim = anim_player.get_animation(anim_name)
			if anim:
				anim.loop_mode = Animation.LOOP_LINEAR
		var list = anim_player.get_animation_list()
		if list.size() > 0:
			anim_player.play(list[0])

func find_animation_and_mouth(node: Node) -> void:
	if node is AnimationPlayer and anim_player == null:
		anim_player = node
	if node.name.containsn("MouthTop"):
		mouth_top = node as Node3D
	elif node.name.containsn("MouthBottom"):
		mouth_bottom = node as Node3D
	for child in node.get_children():
		find_animation_and_mouth(child)

func reset_to_start() -> void:
	grid_pos = Vector2i(16, 10)
	next_tile = Vector2i(16, 10)
	progress = 1.0
	current_dir = MazeData.Dir.NONE
	queued_dir = MazeData.Dir.NONE
	chomp_flip = false
	var world = MazeData.grid_to_world(16, 10)
	position = Vector3(world.x, 0.50, world.z)
	rotation = Vector3.ZERO

func queue_direction(new_dir: MazeData.Dir) -> void:
	queued_dir = new_dir
	if new_dir == MazeData.opposite_dir(current_dir):
		current_dir = new_dir
		var tmp = grid_pos
		grid_pos = next_tile
		next_tile = tmp
		progress = clampf(1.0 - progress, 0.0, 1.0)

func _process(_delta: float) -> void:
	# Continuous procedural chomp: in Godot (where Forward is -Z),
	# a positive X rotation opens the upper jaw UP, and negative opens the lower jaw DOWN
	var t = Time.get_ticks_msec() * 0.001
	var chomp_angle = (sin(t * 16.0) * 0.5 + 0.5) * deg_to_rad(30.0)
	if is_instance_valid(mouth_top):
		mouth_top.rotation.x = chomp_angle
	if is_instance_valid(mouth_bottom):
		mouth_bottom.rotation.x = -chomp_angle

func update_movement(delta: float) -> void:
	if progress >= 1.0:
		grid_pos = next_tile

		# Tunnel Wrap-Around on row 10
		if grid_pos.x == 10:
			if grid_pos.y == 0 and current_dir == MazeData.Dir.LEFT:
				grid_pos = Vector2i(10, MazeData.MAP_WIDTH - 1)
				next_tile = Vector2i(10, MazeData.MAP_WIDTH - 2)
				progress = 0.0
				return
			elif grid_pos.y == MazeData.MAP_WIDTH - 1 and current_dir == MazeData.Dir.RIGHT:
				grid_pos = Vector2i(10, 0)
				next_tile = Vector2i(10, 1)
				progress = 0.0
				return

		if queued_dir != MazeData.Dir.NONE:
			var off = MazeData.dir_to_offset(queued_dir)
			var tr = grid_pos.x + off.x
			var tc = grid_pos.y + off.y
			if not MazeData.is_wall(tr, tc):
				current_dir = queued_dir

		if current_dir != MazeData.Dir.NONE:
			var off = MazeData.dir_to_offset(current_dir)
			var tr = grid_pos.x + off.x
			var tc = grid_pos.y + off.y
			if not MazeData.is_wall(tr, tc):
				next_tile = Vector2i(tr, tc)
				progress = 0.0
			else:
				current_dir = MazeData.Dir.NONE

	if progress < 1.0:
		progress = minf(progress + delta * speed, 1.0)
		var from_w = MazeData.grid_to_world(grid_pos.x, grid_pos.y)
		var to_w = MazeData.grid_to_world(next_tile.x, next_tile.y)
		var curr = from_w.lerp(to_w, progress)
		position.x = curr.x
		position.z = curr.z

		if current_dir != MazeData.Dir.NONE:
			var move_vec = MazeData.dir_to_vector3(current_dir)
			if move_vec != Vector3.ZERO:
				look_at(position + move_vec, Vector3.UP)
```

### packman_3d/scripts/ghost.gd
```gdscript
class_name GhostCharacter
extends Node3D

enum Identity { BLINKY, PINKY, INKY, CLYDE }
enum State { IN_HOUSE, CHASE, SCATTER, FRIGHTENED, EATEN }

@export var identity: Identity = Identity.BLINKY
var state: State = State.SCATTER
var grid_pos: Vector2i = Vector2i(7, 10)
var next_tile: Vector2i = Vector2i(7, 10)
var progress: float = 1.0
var current_dir: MazeData.Dir = MazeData.Dir.NONE
var home_corner: Vector2i = Vector2i(0, MazeData.MAP_WIDTH - 1)
var house_timer: float = 0.0
var base_color: Color = Color.RED

@onready var visual_normal: Node3D = $VisualNormal
@onready var visual_frightened: Node3D = $VisualFrightened
@onready var visual_flash: Node3D = $VisualFlash
@onready var visual_eyes: Node3D = $VisualEyes
@onready var aura_light: OmniLight3D = $AuraLight

func setup(p_identity: Identity, start_pos: Vector2i, p_state: State, exit_delay: float) -> void:
	identity = p_identity
	grid_pos = start_pos
	next_tile = start_pos
	progress = 1.0
	state = p_state
	house_timer = exit_delay

	match identity:
		Identity.BLINKY:
			home_corner = Vector2i(0, MazeData.MAP_WIDTH - 1)
			base_color = Color(1.0, 0.2, 0.2)
		Identity.PINKY:
			home_corner = Vector2i(0, 0)
			base_color = Color(1.0, 0.45, 0.75)
		Identity.INKY:
			home_corner = Vector2i(MazeData.MAP_HEIGHT - 1, MazeData.MAP_WIDTH - 1)
			base_color = Color(0.1, 0.85, 1.0)
		Identity.CLYDE:
			home_corner = Vector2i(MazeData.MAP_HEIGHT - 1, 0)
			base_color = Color(1.0, 0.62, 0.1)

	var world = MazeData.grid_to_world(grid_pos.x, grid_pos.y)
	position = Vector3(world.x, 0.50, world.z)

func _ready() -> void:
	start_nested_animations(self)

func start_nested_animations(node: Node) -> void:
	if node is AnimationPlayer:
		for anim_name in node.get_animation_list():
			var anim = node.get_animation(anim_name)
			if anim:
				anim.loop_mode = Animation.LOOP_LINEAR
		var list = node.get_animation_list()
		if list.size() > 0:
			node.play(list[0])
	for child in node.get_children():
		start_nested_animations(child)

func reset_to_initial() -> void:
	match identity:
		Identity.BLINKY:
			setup(Identity.BLINKY, Vector2i(7, 10), State.SCATTER, 0.0)
		Identity.PINKY:
			setup(Identity.PINKY, Vector2i(10, 10), State.IN_HOUSE, 2.0)
		Identity.INKY:
			setup(Identity.INKY, Vector2i(10, 9), State.IN_HOUSE, 4.0)
		Identity.CLYDE:
			setup(Identity.CLYDE, Vector2i(10, 11), State.IN_HOUSE, 6.0)

	rotation = Vector3.ZERO
	current_dir = MazeData.Dir.NONE

func _process(_delta: float) -> void:
	# Continuous procedural floating hover and body wobble animation
	var t = Time.get_ticks_msec() * 0.001
	var offset = float(identity) * 1.5
	var bob = sin(t * 5.0 + offset) * 0.06
	var wobble = sin(t * 8.0 + offset) * 0.04

	for vis in [visual_normal, visual_frightened, visual_flash, visual_eyes]:
		if is_instance_valid(vis) and vis.visible:
			vis.position.y = bob
			vis.rotation.z = wobble

func update_ghost_ai(delta: float, pacman_pos: Vector2i, pacman_dir: MazeData.Dir, level: int) -> void:
	if state == State.IN_HOUSE:
		house_timer -= delta
		if house_timer <= 0.0:
			grid_pos = Vector2i(7, 10)
			next_tile = Vector2i(7, 10)
			progress = 1.0
			state = State.CHASE
		else:
			return

	var speed = 2.8 + float(level) * 0.15
	if state == State.EATEN:
		speed = 5.4
	elif state == State.FRIGHTENED:
		speed = 1.8

	if progress >= 1.0:
		grid_pos = next_tile

		# Tunnel Wrap-Around on row 10
		if grid_pos.x == 10:
			if grid_pos.y == 0 and current_dir == MazeData.Dir.LEFT:
				grid_pos = Vector2i(10, MazeData.MAP_WIDTH - 1)
				next_tile = Vector2i(10, MazeData.MAP_WIDTH - 2)
				progress = 0.0
			elif grid_pos.y == MazeData.MAP_WIDTH - 1 and current_dir == MazeData.Dir.RIGHT:
				grid_pos = Vector2i(10, 0)
				next_tile = Vector2i(10, 1)
				progress = 0.0

		if state == State.EATEN and grid_pos == Vector2i(7, 10):
			state = State.CHASE

		# Target determination
		var target = Vector2i.ZERO
		match state:
			State.EATEN:
				target = Vector2i(7, 10)
			State.SCATTER:
				target = home_corner
			State.FRIGHTENED:
				target = Vector2i(randi_range(0, MazeData.MAP_HEIGHT - 1), randi_range(0, MazeData.MAP_WIDTH - 1))
			State.CHASE:
				match identity:
					Identity.BLINKY:
						target = pacman_pos
					Identity.PINKY:
						var off = MazeData.dir_to_offset(pacman_dir)
						target = Vector2i(
							clampi(pacman_pos.x + off.x * 4, 0, MazeData.MAP_HEIGHT - 1),
							clampi(pacman_pos.y + off.y * 4, 0, MazeData.MAP_WIDTH - 1)
						)
					Identity.INKY:
						var off = MazeData.dir_to_offset(pacman_dir)
						target = Vector2i(
							clampi(pacman_pos.x + off.x * 2, 0, MazeData.MAP_HEIGHT - 1),
							clampi(pacman_pos.y + off.y * 2, 0, MazeData.MAP_WIDTH - 1)
						)
					Identity.CLYDE:
						var dist = Vector2(grid_pos).distance_to(Vector2(pacman_pos))
						target = pacman_pos if dist > 6.0 else home_corner
			_:
				target = Vector2i(7, 10)

		var dirs = [MazeData.Dir.UP, MazeData.Dir.DOWN, MazeData.Dir.LEFT, MazeData.Dir.RIGHT]
		var best_dir = MazeData.Dir.NONE
		var best_dist = 1e9

		for d in dirs:
			if d == MazeData.opposite_dir(current_dir) and state != State.FRIGHTENED:
				continue
			var off = MazeData.dir_to_offset(d)
			var nr = grid_pos.x + off.x
			var nc = grid_pos.y + off.y
			if not MazeData.is_wall(nr, nc):
				var dist = Vector2(nr, nc).distance_squared_to(Vector2(target))
				if dist < best_dist:
					best_dist = dist
					best_dir = d

		if best_dir != MazeData.Dir.NONE:
			current_dir = best_dir
			var off = MazeData.dir_to_offset(best_dir)
			next_tile = Vector2i(grid_pos.x + off.x, grid_pos.y + off.y)
			progress = 0.0

	if progress < 1.0:
		progress = minf(progress + delta * speed, 1.0)
		var from_w = MazeData.grid_to_world(grid_pos.x, grid_pos.y)
		var to_w = MazeData.grid_to_world(next_tile.x, next_tile.y)
		var curr = from_w.lerp(to_w, progress)
		position.x = curr.x
		position.z = curr.z

		if current_dir != MazeData.Dir.NONE:
			var move_vec = MazeData.dir_to_vector3(current_dir)
			if move_vec != Vector3.ZERO:
				look_at(position + move_vec, Vector3.UP)

func update_visuals(frightened_timer: float) -> void:
	var flash_on = fmod(Time.get_ticks_msec() * 0.008, 2.0) > 1.0
	var is_warning = state == State.FRIGHTENED and frightened_timer < 2.5

	visual_normal.visible = (state != State.FRIGHTENED and state != State.EATEN)
	visual_frightened.visible = (state == State.FRIGHTENED and (!is_warning or !flash_on))
	visual_flash.visible = (state == State.FRIGHTENED and is_warning and flash_on)
	visual_eyes.visible = (state == State.EATEN)

	if state == State.EATEN:
		aura_light.light_color = Color(0.2, 0.5, 1.0)
		aura_light.light_energy = 0.4
	elif state == State.FRIGHTENED:
		if is_warning and flash_on:
			aura_light.light_color = Color.WHITE
			aura_light.light_energy = 0.8
		else:
			aura_light.light_color = Color(0.15, 0.35, 1.0)
			aura_light.light_energy = 0.6
	else:
		aura_light.light_color = base_color
		aura_light.light_energy = 0.5
```

### packman_3d/scripts/camera_controller.gd
```gdscript
class_name CameraController
extends Camera3D

var target_position: Vector3 = Vector3.ZERO
var target_look: Vector3 = Vector3.ZERO

func _ready() -> void:
	var start_pos = MazeData.grid_to_world(16, 10) + Vector3(0.0, 0.5, 0.0)
	position = start_pos + Vector3(0.0, 8.5, 7.0)
	look_at(start_pos + Vector3(0.0, 0.5, 0.0), Vector3.UP)

func update_camera(delta: float, pacman_pos: Vector3) -> void:
	target_position = pacman_pos + Vector3(0.0, 8.5, 7.0)
	target_look = pacman_pos + Vector3(0.0, 0.5, 0.0)
	position = position.lerp(target_position, delta * 8.0)
	look_at(target_look, Vector3.UP)
```

### packman_3d/scripts/touch_controls.gd
```gdscript
class_name TouchControls
extends Control

signal direction_pressed(dir: MazeData.Dir)
signal screen_tapped()

const SWIPE_THRESHOLD: float = 24.0

var active_touch_index: int = -1
var touch_start_pos: Vector2 = Vector2.ZERO
var touch_initial_pos: Vector2 = Vector2.ZERO
var has_dragged: bool = false

func _ready() -> void:
	_connect_dpad("DPad")
	_connect_dpad("DPadRight")

func _connect_dpad(dpad_name: String) -> void:
	if has_node(dpad_name):
		var dpad = get_node(dpad_name)
		if dpad.has_node("BtnUp"):
			dpad.get_node("BtnUp").button_down.connect(func(): direction_pressed.emit(MazeData.Dir.UP))
		if dpad.has_node("BtnDown"):
			dpad.get_node("BtnDown").button_down.connect(func(): direction_pressed.emit(MazeData.Dir.DOWN))
		if dpad.has_node("BtnLeft"):
			dpad.get_node("BtnLeft").button_down.connect(func(): direction_pressed.emit(MazeData.Dir.LEFT))
		if dpad.has_node("BtnRight"):
			dpad.get_node("BtnRight").button_down.connect(func(): direction_pressed.emit(MazeData.Dir.RIGHT))

func _input(event: InputEvent) -> void:
	if event is InputEventScreenTouch:
		if event.pressed:
			if active_touch_index == -1:
				active_touch_index = event.index
				touch_start_pos = event.position
				touch_initial_pos = event.position
				has_dragged = false
		else:
			if event.index == active_touch_index:
				active_touch_index = -1
				if not has_dragged:
					screen_tapped.emit()
					_handle_tap(event.position)

	elif event is InputEventScreenDrag:
		if event.index == active_touch_index:
			_process_drag(event.position)

	elif event is InputEventMouseButton:
		if event.button_index == MOUSE_BUTTON_LEFT:
			if event.pressed:
				if active_touch_index == -1:
					active_touch_index = 0
					touch_start_pos = event.position
					touch_initial_pos = event.position
					has_dragged = false
			else:
				if active_touch_index == 0:
					active_touch_index = -1
					if not has_dragged:
						screen_tapped.emit()
						_handle_tap(event.position)

	elif event is InputEventMouseMotion:
		if active_touch_index == 0 and (event.button_mask & MOUSE_BUTTON_MASK_LEFT):
			_process_drag(event.position)

func _process_drag(current_pos: Vector2) -> void:
	var diff = current_pos - touch_start_pos
	if diff.length() >= SWIPE_THRESHOLD:
		var dir = MazeData.Dir.NONE
		if abs(diff.x) > abs(diff.y):
			dir = MazeData.Dir.RIGHT if diff.x > 0 else MazeData.Dir.LEFT
		else:
			dir = MazeData.Dir.DOWN if diff.y > 0 else MazeData.Dir.UP
		direction_pressed.emit(dir)
		touch_start_pos = current_pos
		has_dragged = true

func _handle_tap(pos: Vector2) -> void:
	var screen_center = get_viewport_rect().size * 0.5
	var offset = pos - screen_center
	if offset.length() > 30.0:
		var dir = MazeData.Dir.NONE
		if abs(offset.x) > abs(offset.y):
			dir = MazeData.Dir.RIGHT if offset.x > 0 else MazeData.Dir.LEFT
		else:
			dir = MazeData.Dir.DOWN if offset.y > 0 else MazeData.Dir.UP
		direction_pressed.emit(dir)
```

### packman_3d/scripts/main.gd
```gdscript
class_name MainGameController
extends Node3D

enum Phase { READY, PLAYING, DYING, GAME_OVER, VICTORY }

const WALL_SCAD = preload("res://assets/models/wall.scad")
const GATE_SCAD = preload("res://assets/models/gate.scad")
const PELLET_SCAD = preload("res://assets/models/pellet.scad")
const POWER_PELLET_SCAD = preload("res://assets/models/power_pellet.scad")
const CHERRY_SCAD = preload("res://assets/models/cherry.scad")

const PACKMAN_SCAD = preload("res://assets/models/packman.scad")
const BLINKY_SCAD = preload("res://assets/models/ghost_blinky.scad")
const PINKY_SCAD = preload("res://assets/models/ghost_pinky.scad")
const INKY_SCAD = preload("res://assets/models/ghost_inky.scad")
const CLYDE_SCAD = preload("res://assets/models/ghost_clyde.scad")
const FRIGHT_SCAD = preload("res://assets/models/ghost_frightened.scad")
const FLASH_SCAD = preload("res://assets/models/ghost_flash.scad")
const EYES_SCAD = preload("res://assets/models/ghost_eyes.scad")

@onready var maze_container: Node3D = $MazeContainer
@onready var pellets_container: Node3D = $PelletsContainer
@onready var characters_container: Node3D = $CharactersContainer
@onready var camera_node: CameraController = $MainCamera

@onready var score_label: Label = $HUD/ScoreLabel
@onready var info_label: Label = $HUD/InfoLabel
@onready var banner_label: Label = $HUD/BannerLabel
@onready var touch_controls: TouchControls = $HUD/TouchControls

@onready var audio_player_fx: AudioStreamPlayer = $AudioFX
@onready var audio_player_music: AudioStreamPlayer = $AudioMusic

var snd_waka_0: AudioStreamWAV
var snd_waka_1: AudioStreamWAV
var snd_power_pellet: AudioStreamWAV
var snd_eat_ghost: AudioStreamWAV
var snd_death: AudioStreamWAV
var snd_fruit: AudioStreamWAV
var snd_game_start: AudioStreamWAV

var score: int = 0
var high_score: int = 10000
var lives: int = 3
var level: int = 1
var pellets_remaining: int = 0
var phase: Phase = Phase.READY
var phase_timer: float = 3.5
var frightened_timer: float = 0.0
var ghost_kill_streak: int = 200
var scatter_timer: float = 7.0
var is_scatter: bool = true
var fruit_respawn_timer: float = 0.0

var pacman: PacmanCharacter = null
var ghosts: Array[GhostCharacter] = []
var active_cherry: CherryFruit = null

func _ready() -> void:
	init_procedural_audio()
	build_maze_architecture()
	spawn_characters()
	init_touch_controls()
	start_level_round()

func init_touch_controls() -> void:
	if touch_controls:
		touch_controls.direction_pressed.connect(_on_touch_direction)
		touch_controls.screen_tapped.connect(_on_touch_tapped)

func _on_touch_direction(dir: MazeData.Dir) -> void:
	if is_instance_valid(pacman):
		pacman.queue_direction(dir)

func _on_touch_tapped() -> void:
	if phase == Phase.GAME_OVER:
		score = 0
		lives = 3
		level = 1
		start_level_round()

func init_procedural_audio() -> void:
	snd_waka_0 = AudioSynth.generate_waka_0()
	snd_waka_1 = AudioSynth.generate_waka_1()
	snd_power_pellet = AudioSynth.generate_power_pellet()
	snd_eat_ghost = AudioSynth.generate_eat_ghost()
	snd_death = AudioSynth.generate_death()
	snd_fruit = AudioSynth.generate_fruit()
	snd_game_start = AudioSynth.generate_game_start()

	audio_player_music.stream = snd_game_start
	audio_player_music.play()

func play_fx(stream: AudioStreamWAV, volume_db: float = 0.0) -> void:
	audio_player_fx.stream = stream
	audio_player_fx.volume_db = volume_db
	audio_player_fx.play()

func build_maze_architecture() -> void:
	for r in range(MazeData.MAP_HEIGHT):
		for c in range(MazeData.MAP_WIDTH):
			var tile = MazeData.TILES[r][c]
			var world = MazeData.grid_to_world(r, c)
			if tile == "W":
				var wall = WALL_SCAD.instantiate()
				wall.position = Vector3(world.x, 0.0, world.z)
				maze_container.add_child(wall)
			elif tile == "D":
				var gate = GATE_SCAD.instantiate()
				gate.position = Vector3(world.x, 0.0, world.z)
				maze_container.add_child(gate)

func spawn_characters() -> void:
	# 1. Spawn Pacman with gentle ambient aura
	pacman = PacmanCharacter.new()
	pacman.name = "Pacman"
	var pac_visual = PACKMAN_SCAD.instantiate()
	pacman.add_child(pac_visual)

	var pac_light = OmniLight3D.new()
	pac_light.name = "AuraLight"
	pac_light.light_color = Color(1.0, 0.9, 0.2)
	pac_light.light_energy = 0.6
	pac_light.omni_range = 3.5
	pac_light.position = Vector3(0.0, 0.45, 0.0)
	pacman.add_child(pac_light)
	characters_container.add_child(pacman)
	pacman.reset_to_start()

	# 2. Spawn Ghosts with subtle colored aura
	var configs = [
		{"type": GhostCharacter.Identity.BLINKY, "pos": Vector2i(7, 10), "state": GhostCharacter.State.SCATTER, "delay": 0.0, "scad": BLINKY_SCAD},
		{"type": GhostCharacter.Identity.PINKY, "pos": Vector2i(10, 10), "state": GhostCharacter.State.IN_HOUSE, "delay": 2.0, "scad": PINKY_SCAD},
		{"type": GhostCharacter.Identity.INKY, "pos": Vector2i(10, 9), "state": GhostCharacter.State.IN_HOUSE, "delay": 4.0, "scad": INKY_SCAD},
		{"type": GhostCharacter.Identity.CLYDE, "pos": Vector2i(10, 11), "state": GhostCharacter.State.IN_HOUSE, "delay": 6.0, "scad": CLYDE_SCAD},
	]

	for cfg in configs:
		var ghost = GhostCharacter.new()
		ghost.name = "Ghost_" + str(cfg.type)

		var v_norm = cfg.scad.instantiate()
		v_norm.name = "VisualNormal"
		ghost.add_child(v_norm)

		var v_fright = FRIGHT_SCAD.instantiate()
		v_fright.name = "VisualFrightened"
		v_fright.visible = false
		ghost.add_child(v_fright)

		var v_flash = FLASH_SCAD.instantiate()
		v_flash.name = "VisualFlash"
		v_flash.visible = false
		ghost.add_child(v_flash)

		var v_eyes = EYES_SCAD.instantiate()
		v_eyes.name = "VisualEyes"
		v_eyes.visible = false
		ghost.add_child(v_eyes)

		var g_light = OmniLight3D.new()
		g_light.name = "AuraLight"
		g_light.omni_range = 3.0
		g_light.light_energy = 0.5
		g_light.position = Vector3(0.0, 0.40, 0.0)
		ghost.add_child(g_light)

		characters_container.add_child(ghost)
		ghost.setup(cfg.type, cfg.pos, cfg.state, cfg.delay)
		ghosts.append(ghost)

func start_level_round() -> void:
	for child in pellets_container.get_children():
		child.queue_free()
	if active_cherry:
		active_cherry.queue_free()
		active_cherry = null

	pellets_remaining = 0
	for r in range(MazeData.MAP_HEIGHT):
		for c in range(MazeData.MAP_WIDTH):
			var tile = MazeData.TILES[r][c]
			var world = MazeData.grid_to_world(r, c)
			if tile == ".":
				var pellet = Pellet.new()
				var p_scad = PELLET_SCAD.instantiate()
				pellet.add_child(p_scad)
				pellet.position = Vector3(world.x, 0.25, world.z)
				pellet.setup(false, r, c)
				pellets_container.add_child(pellet)
				pellets_remaining += 1
			elif tile == "O":
				var power = Pellet.new()
				var p_scad = POWER_PELLET_SCAD.instantiate()
				power.add_child(p_scad)
				power.position = Vector3(world.x, 0.35, world.z)
				power.setup(true, r, c)
				pellets_container.add_child(power)
				pellets_remaining += 1

	spawn_bonus_cherry()

	pacman.reset_to_start()
	for ghost in ghosts:
		ghost.reset_to_initial()

	phase = Phase.READY
	phase_timer = 2.8
	banner_label.text = "READY!"
	banner_label.visible = true

func spawn_bonus_cherry() -> void:
	if active_cherry:
		active_cherry.queue_free()
	active_cherry = CherryFruit.new()
	var cherry_mesh = CHERRY_SCAD.instantiate()
	active_cherry.add_child(cherry_mesh)
	var world = MazeData.grid_to_world(13, 10)
	active_cherry.position = Vector3(world.x, 0.38, world.z)
	add_child(active_cherry)

func _process(delta: float) -> void:
	handle_player_input()

	if score > high_score:
		high_score = score

	match phase:
		Phase.READY:
			phase_timer -= delta
			banner_label.visible = true
			banner_label.text = "READY!"
			if phase_timer <= 0.0:
				phase = Phase.PLAYING
				banner_label.visible = false

		Phase.PLAYING:
			update_gameplay(delta)

		Phase.DYING:
			phase_timer -= delta
			banner_label.visible = true
			banner_label.text = "OUCH!"
			if phase_timer <= 0.0:
				lives -= 1
				if lives > 0:
					pacman.reset_to_start()
					for ghost in ghosts:
						ghost.reset_to_initial()
					phase = Phase.READY
					phase_timer = 2.0
				else:
					phase = Phase.GAME_OVER
					banner_label.text = "GAME OVER - TAP TO RETRY"

		Phase.GAME_OVER:
			banner_label.visible = true
			if Input.is_action_just_pressed("ui_accept"):
				score = 0
				lives = 3
				level = 1
				start_level_round()

		Phase.VICTORY:
			banner_label.visible = true
			banner_label.text = "STAGE CLEARED!"
			phase_timer -= delta
			if phase_timer <= 0.0:
				level += 1
				start_level_round()

	camera_node.update_camera(delta, pacman.position)
	update_hud()

func handle_player_input() -> void:
	if Input.is_action_pressed("ui_up") or Input.is_key_pressed(KEY_W):
		pacman.queue_direction(MazeData.Dir.UP)
	elif Input.is_action_pressed("ui_down") or Input.is_key_pressed(KEY_S):
		pacman.queue_direction(MazeData.Dir.DOWN)
	elif Input.is_action_pressed("ui_left") or Input.is_key_pressed(KEY_A):
		pacman.queue_direction(MazeData.Dir.LEFT)
	elif Input.is_action_pressed("ui_right") or Input.is_key_pressed(KEY_D):
		pacman.queue_direction(MazeData.Dir.RIGHT)

func update_gameplay(delta: float) -> void:
	if frightened_timer > 0.0:
		frightened_timer = maxf(frightened_timer - delta, 0.0)
		if frightened_timer == 0.0:
			for ghost in ghosts:
				if ghost.state == GhostCharacter.State.FRIGHTENED:
					ghost.state = GhostCharacter.State.SCATTER if is_scatter else GhostCharacter.State.CHASE
	else:
		scatter_timer -= delta
		if scatter_timer <= 0.0:
			is_scatter = !is_scatter
			scatter_timer = 7.0 if is_scatter else 20.0
			for ghost in ghosts:
				if ghost.state == GhostCharacter.State.CHASE or ghost.state == GhostCharacter.State.SCATTER:
					ghost.state = GhostCharacter.State.SCATTER if is_scatter else GhostCharacter.State.CHASE

	pacman.update_movement(delta)
	for ghost in ghosts:
		ghost.update_ghost_ai(delta, pacman.grid_pos, pacman.current_dir, level)
		ghost.update_visuals(frightened_timer)

	# Collisions: Pellets
	var pac_pos = pacman.position
	for child in pellets_container.get_children():
		var pellet = child as Pellet
		if not is_instance_valid(pellet):
			continue
		if pac_pos.distance_to(pellet.position) < 0.65:
			pellets_remaining -= 1
			if pellet.is_power:
				score += 50
				frightened_timer = 7.0
				ghost_kill_streak = 200
				for ghost in ghosts:
					if ghost.state != GhostCharacter.State.IN_HOUSE and ghost.state != GhostCharacter.State.EATEN:
						ghost.state = GhostCharacter.State.FRIGHTENED
				play_fx(snd_power_pellet, -2.0)
			else:
				score += 10
				pacman.chomp_flip = !pacman.chomp_flip
				play_fx(snd_waka_0 if pacman.chomp_flip else snd_waka_1, -4.0)

			pellet.queue_free()

			if pellets_remaining <= 0:
				phase = Phase.VICTORY
				phase_timer = 3.0
				return

	# Collisions: Bonus Cherry Fruit
	if is_instance_valid(active_cherry):
		active_cherry.lifetime -= delta
		if active_cherry.lifetime <= 0.0:
			active_cherry.queue_free()
			active_cherry = null
			fruit_respawn_timer = 15.0
		elif pac_pos.distance_to(active_cherry.position) < 0.85:
			score += 100
			play_fx(snd_fruit, -2.0)
			spawn_floating_text("+100", active_cherry.position, Color(1.0, 0.4, 0.7))
			active_cherry.queue_free()
			active_cherry = null
			fruit_respawn_timer = 20.0
	else:
		fruit_respawn_timer -= delta
		if fruit_respawn_timer <= 0.0:
			spawn_bonus_cherry()
			fruit_respawn_timer = 25.0

	# Collisions: Ghosts
	for ghost in ghosts:
		if pac_pos.distance_to(ghost.position) < 0.75:
			if ghost.state == GhostCharacter.State.FRIGHTENED:
				ghost.state = GhostCharacter.State.EATEN
				score += ghost_kill_streak
				play_fx(snd_eat_ghost, -1.0)
				spawn_floating_text("+" + str(ghost_kill_streak), ghost.position, Color(0.2, 1.0, 0.4))
				ghost_kill_streak *= 2
			elif ghost.state == GhostCharacter.State.CHASE or ghost.state == GhostCharacter.State.SCATTER:
				phase = Phase.DYING
				phase_timer = 2.0
				play_fx(snd_death, 0.0)
				break

func spawn_floating_text(text: String, world_pos: Vector3, color: Color) -> void:
	var float_label = FloatingText.new()
	float_label.text = text
	float_label.position = world_pos + Vector3(0.0, 0.8, 0.0)
	float_label.modulate = color
	add_child(float_label)

func update_hud() -> void:
	score_label.text = "SCORE: %06d   HIGH: %06d" % [score, high_score]
	info_label.text = "LIVES: %d  |  LEVEL: %d" % [lives, level]
```

### packman_3d/scenes/main.tscn
```ini
[gd_scene load_steps=8 format=3 uid="uid://c7m5f1a9px3yq"]

[ext_resource type="Script" path="res://scripts/main.gd" id="1_main"]
[ext_resource type="Script" path="res://scripts/camera_controller.gd" id="2_cam"]
[ext_resource type="Script" path="res://scripts/touch_controls.gd" id="3_touch"]

[sub_resource type="ProceduralSkyMaterial" id="ProceduralSkyMaterial_1"]
sky_top_color = Color(0.02, 0.03, 0.06, 1)
sky_horizon_color = Color(0.04, 0.06, 0.12, 1)
ground_bottom_color = Color(0.01, 0.02, 0.04, 1)
ground_horizon_color = Color(0.04, 0.06, 0.12, 1)

[sub_resource type="Sky" id="Sky_1"]
sky_material = SubResource("ProceduralSkyMaterial_1")

[sub_resource type="Environment" id="Environment_1"]
background_mode = 2
sky = SubResource("Sky_1")
ambient_light_source = 3
ambient_light_color = Color(0.08, 0.1, 0.22, 1)
ambient_light_energy = 0.4
reflected_light_source = 2
tonemap_mode = 2
glow_enabled = true
glow_levels/1 = 0.0
glow_levels/2 = 0.0
glow_levels/3 = 1.0
glow_levels/4 = 0.0
glow_levels/5 = 0.2
glow_intensity = 0.18
glow_strength = 0.75
glow_bloom = 0.0
glow_blend_mode = 0

[sub_resource type="StandardMaterial3D" id="StandardMaterial3D_floor"]
albedo_color = Color(0.02, 0.03, 0.06, 1)
metallic = 0.8
roughness = 0.25

[sub_resource type="PlaneMesh" id="PlaneMesh_floor"]
material = SubResource("StandardMaterial3D_floor")
size = Vector2(32, 32)

[node name="Main" type="Node3D"]
script = ExtResource("1_main")

[node name="WorldEnvironment" type="WorldEnvironment" parent="."]
environment = SubResource("Environment_1")

[node name="KeyLight" type="DirectionalLight3D" parent="."]
transform = Transform3D(0.921, -0.321, 0.221, 0, 0.567, 0.824, -0.389, -0.759, 0.522, 0, 15, 0)
light_color = Color(0.9, 0.95, 1, 1)
light_energy = 0.9
shadow_enabled = true

[node name="Floor" type="MeshInstance3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, -0.05, 0)
mesh = SubResource("PlaneMesh_floor")

[node name="MainCamera" type="Camera3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 0.658, 0.753, 0, -0.753, 0.658, 0, 9, 14.2)
current = true
fov = 48.0
script = ExtResource("2_cam")

[node name="MazeContainer" type="Node3D" parent="."]

[node name="PelletsContainer" type="Node3D" parent="."]

[node name="CharactersContainer" type="Node3D" parent="."]

[node name="AudioFX" type="AudioStreamPlayer" parent="."]
bus = &"Master"

[node name="AudioMusic" type="AudioStreamPlayer" parent="."]
bus = &"Master"

[node name="HUD" type="CanvasLayer" parent="."]

[node name="ScoreLabel" type="Label" parent="HUD"]
offset_left = 24.0
offset_top = 16.0
offset_right = 400.0
offset_bottom = 52.0
theme_override_colors/font_color = Color(1, 0.9, 0.2, 1)
theme_override_font_sizes/font_size = 24
text = "SCORE: 000000   HIGH: 010000"

[node name="InfoLabel" type="Label" parent="HUD"]
anchors_preset = 1
anchor_left = 1.0
anchor_right = 1.0
offset_left = -460.0
offset_top = 16.0
offset_right = -24.0
offset_bottom = 52.0
grow_horizontal = 0
theme_override_colors/font_color = Color(0.2, 0.9, 1, 1)
theme_override_font_sizes/font_size = 24
text = "LIVES: 3  |  LEVEL: 1"
horizontal_alignment = 2

[node name="BannerLabel" type="Label" parent="HUD"]
anchors_preset = 7
anchor_left = 0.5
anchor_top = 1.0
anchor_right = 0.5
anchor_bottom = 1.0
offset_left = -300.0
offset_top = -80.0
offset_right = 300.0
offset_bottom = -24.0
grow_horizontal = 2
grow_vertical = 0
theme_override_colors/font_color = Color(1, 0.85, 0.1, 1)
theme_override_colors/font_outline_color = Color(0, 0, 0, 1)
theme_override_constants/outline_size = 8
theme_override_font_sizes/font_size = 38
text = "READY!"
horizontal_alignment = 1
vertical_alignment = 1

[node name="TouchControls" type="Control" parent="HUD"]
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
mouse_filter = 1
script = ExtResource("3_touch")

[node name="DPad" type="Control" parent="HUD/TouchControls"]
layout_mode = 1
anchors_preset = 2
anchor_top = 1.0
anchor_bottom = 1.0
offset_left = 30.0
offset_top = -250.0
offset_right = 250.0
offset_bottom = -30.0
grow_vertical = 0
mouse_filter = 1

[node name="BtnUp" type="Button" parent="HUD/TouchControls/DPad"]
modulate = Color(1, 1, 1, 0)
layout_mode = 1
anchors_preset = 5
anchor_left = 0.5
anchor_right = 0.5
offset_left = -45.0
offset_top = 0.0
offset_right = 45.0
offset_bottom = 80.0
grow_horizontal = 2
focus_mode = 0
text = "UP"

[node name="BtnDown" type="Button" parent="HUD/TouchControls/DPad"]
modulate = Color(1, 1, 1, 0)
layout_mode = 1
anchors_preset = 7
anchor_left = 0.5
anchor_top = 1.0
anchor_right = 0.5
anchor_bottom = 1.0
offset_left = -45.0
offset_top = -80.0
offset_right = 45.0
offset_bottom = 0.0
grow_horizontal = 2
grow_vertical = 0
focus_mode = 0
text = "DOWN"

[node name="BtnLeft" type="Button" parent="HUD/TouchControls/DPad"]
modulate = Color(1, 1, 1, 0)
layout_mode = 1
anchors_preset = 4
anchor_top = 0.5
anchor_bottom = 0.5
offset_left = 0.0
offset_top = -45.0
offset_right = 80.0
offset_bottom = 45.0
grow_vertical = 2
focus_mode = 0
text = "LEFT"

[node name="BtnRight" type="Button" parent="HUD/TouchControls/DPad"]
modulate = Color(1, 1, 1, 0)
layout_mode = 1
anchors_preset = 6
anchor_left = 1.0
anchor_top = 0.5
anchor_right = 1.0
anchor_bottom = 0.5
offset_left = -80.0
offset_top = -45.0
offset_right = 0.0
offset_bottom = 45.0
grow_horizontal = 0
grow_vertical = 2
focus_mode = 0
text = "RIGHT"

[node name="DPadRight" type="Control" parent="HUD/TouchControls"]
layout_mode = 1
anchors_preset = 3
anchor_left = 1.0
anchor_top = 1.0
anchor_right = 1.0
anchor_bottom = 1.0
offset_left = -250.0
offset_top = -250.0
offset_right = -30.0
offset_bottom = -30.0
grow_horizontal = 0
grow_vertical = 0
mouse_filter = 1

[node name="BtnUp" type="Button" parent="HUD/TouchControls/DPadRight"]
modulate = Color(1, 1, 1, 0)
layout_mode = 1
anchors_preset = 5
anchor_left = 0.5
anchor_right = 0.5
offset_left = -45.0
offset_top = 0.0
offset_right = 45.0
offset_bottom = 80.0
grow_horizontal = 2
focus_mode = 0
text = "UP"

[node name="BtnDown" type="Button" parent="HUD/TouchControls/DPadRight"]
modulate = Color(1, 1, 1, 0)
layout_mode = 1
anchors_preset = 7
anchor_left = 0.5
anchor_top = 1.0
anchor_right = 0.5
anchor_bottom = 1.0
offset_left = -45.0
offset_top = -80.0
offset_right = 45.0
offset_bottom = 0.0
grow_horizontal = 2
grow_vertical = 0
focus_mode = 0
text = "DOWN"

[node name="BtnLeft" type="Button" parent="HUD/TouchControls/DPadRight"]
modulate = Color(1, 1, 1, 0)
layout_mode = 1
anchors_preset = 4
anchor_top = 0.5
anchor_bottom = 0.5
offset_left = 0.0
offset_top = -45.0
offset_right = 80.0
offset_bottom = 45.0
grow_vertical = 2
focus_mode = 0
text = "LEFT"

[node name="BtnRight" type="Button" parent="HUD/TouchControls/DPadRight"]
modulate = Color(1, 1, 1, 0)
layout_mode = 1
anchors_preset = 6
anchor_left = 1.0
anchor_top = 0.5
anchor_right = 1.0
anchor_bottom = 0.5
offset_left = -80.0
offset_top = -45.0
offset_right = 0.0
offset_bottom = 45.0
grow_horizontal = 0
grow_vertical = 2
focus_mode = 0
text = "RIGHT"
```
