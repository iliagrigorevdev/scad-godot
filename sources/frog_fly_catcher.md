### frog_fly_catcher/project.godot
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

config/name="Frog Fly Catcher"
run/main_scene="res://scenes/main_scene.tscn"
config/features=PackedStringArray("4.3", "Forward Plus")

[display]

window/size/viewport_width=1280
window/size/viewport_height=720
window/stretch/mode="canvas_items"

[editor_plugins]

enabled=PackedStringArray("res://addons/scad_importer/plugin.cfg")

[input]

jump={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":32,"physical_keycode":0,"key_label":0,"unicode":32,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":4194320,"physical_keycode":0,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":87,"physical_keycode":0,"key_label":0,"unicode":119,"location":0,"echo":false,"script":null)
]
}

[rendering]

anti_aliasing/quality/msaa_3d=2
```

### frog_fly_catcher/addons/scad_importer/plugin.cfg
```text
[plugin]

name="OpenSCAD GLTF Importer"
description="Imports .scad files directly as 3D scenes using scad-gltf"
author="scad-gltf"
version="0.1"
script="scad_plugin.gd"

```

### frog_fly_catcher/addons/scad_importer/scad_importer.gd
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

### frog_fly_catcher/addons/scad_importer/scad_plugin.gd
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

### frog_fly_catcher/assets/frog.scad
```openscad
// Stylized Frog Hero (1 unit = 1 meter, +Z Up, +Y Forward, -X Left, +X Right)
$fn = 24;
$asa = 40;

anim_data = [
  ["idle", [
    ["Body", [
      [0.0, [0, 0, 0], [0, 0, 0.28]],
      [0.7, [2, 0, 0], [0, 0, 0.30]],
      [1.4, [0, 0, 0], [0, 0, 0.28]]
    ]],
    ["Jaw", [
      [0.0, [0, 0, 0]],
      [0.7, [-4, 0, 0]],
      [1.4, [0, 0, 0]]
    ]],
    ["TongueTip", [
      [0.0, [0, 0, 0], [0, 0.16, 0.02]],
      [1.4, [0, 0, 0], [0, 0.16, 0.02]]
    ]],
    ["LeftArm", [
      [0.0, [0, 0, 0]],
      [1.4, [0, 0, 0]]
    ]],
    ["RightArm", [
      [0.0, [0, 0, 0]],
      [1.4, [0, 0, 0]]
    ]],
    ["LeftLeg", [
      [0.0, [0, 0, 0]],
      [1.4, [0, 0, 0]]
    ]],
    ["RightLeg", [
      [0.0, [0, 0, 0]],
      [1.4, [0, 0, 0]]
    ]]
  ]],
  ["jump", [
    ["Body", [
      [0.0, [10, 0, 0], [0, 0, 0.25]],
      [0.15, [-18, 0, 0], [0, 0, 0.36]],
      [0.45, [8, 0, 0], [0, 0, 0.32]],
      [0.6, [0, 0, 0], [0, 0, 0.28]]
    ]],
    ["LeftLeg", [
      [0.0, [0, 0, 0]],
      [0.15, [-55, 0, -10]],
      [0.45, [-35, 0, -6]],
      [0.6, [0, 0, 0]]
    ]],
    ["RightLeg", [
      [0.0, [0, 0, 0]],
      [0.15, [-55, 0, 10]],
      [0.45, [-35, 0, 6]],
      [0.6, [0, 0, 0]]
    ]],
    ["LeftArm", [
      [0.0, [0, 0, 0]],
      [0.15, [65, 0, -15]],
      [0.6, [0, 0, 0]]
    ]],
    ["RightArm", [
      [0.0, [0, 0, 0]],
      [0.15, [65, 0, 15]],
      [0.6, [0, 0, 0]]
    ]]
  ]],
  ["eat", [
    ["Body", [
      [0.0, [-10, 0, 0], [0, 0, 0.31]],
      [0.15, [-14, 0, 0], [0, 0, 0.32]],
      [0.35, [0, 0, 0], [0, 0, 0.28]]
    ]],
    ["Jaw", [
      [0.0, [0, 0, 0]],
      [0.08, [-42, 0, 0]],
      [0.22, [-38, 0, 0]],
      [0.35, [0, 0, 0]]
    ]],
    ["TongueTip", [
      [0.0, [0, 0, 0], [0, 0.16, 0.02]],
      [0.10, [0, 0, 0], [0, 0.48, 0.04]],
      [0.24, [0, 0, 0], [0, 0.18, 0.02]],
      [0.35, [0, 0, 0], [0, 0.16, 0.02]]
    ]],
    ["LeftArm", [
      [0.0, [20, 0, -5]],
      [0.35, [0, 0, 0]]
    ]],
    ["RightArm", [
      [0.0, [20, 0, 5]],
      [0.35, [0, 0, 0]]
    ]],
    ["LeftLeg", [
      [0.0, [-15, 0, -4]],
      [0.35, [0, 0, 0]]
    ]],
    ["RightLeg", [
      [0.0, [-15, 0, 4]],
      [0.35, [0, 0, 0]]
    ]]
  ]]
];

module frog_skin() {
  color([0.12, 0.56, 0.18], name="frog_skin", roughness=0.38, metalness=0.0, specularIntensity=0.5, $asa=42)
    children();
}

module frog_belly() {
  color([0.84, 0.88, 0.45], name="frog_belly", roughness=0.45, metalness=0.0, $asa=42)
    children();
}

module frog_spots() {
  color([0.07, 0.40, 0.12], name="frog_spots", roughness=0.40, metalness=0.0, $asa=35)
    children();
}

module eye_white() {
  color([0.96, 0.96, 0.90], name="eye_white", roughness=0.12, specularIntensity=1.0, $asa=45)
    children();
}

module eye_pupil() {
  color([0.03, 0.03, 0.05], name="eye_pupil", roughness=0.05, specularIntensity=1.0, $asa=45)
    children();
}

module mouth_inside() {
  color([0.78, 0.22, 0.30], name="mouth_inside", roughness=0.45, $asa=35)
    children();
}

module tongue_mat() {
  color([0.96, 0.30, 0.45], name="tongue_mat", roughness=0.25, specularIntensity=0.8, $asa=40)
    children();
}

module leg_assembly(side = 1) {
  // side = -1 for Left (-X), +1 for Right (+X)
  // Upper thigh
  frog_skin() {
    hull() {
      translate([0, 0, 0]) sphere(r=0.10, $fn=20);
      translate([side * 0.14, 0.06, -0.04]) sphere(r=0.08, $fn=20);
    }
    // Lower shin
    hull() {
      translate([side * 0.14, 0.06, -0.04]) sphere(r=0.075, $fn=20);
      translate([side * 0.06, -0.06, -0.12]) sphere(r=0.055, $fn=18);
    }
    // Webbed foot resting near ground Z = -0.14 relative to leg pivot (Z = 0.14)
    translate([side * 0.09, 0.08, -0.12]) {
      for (ang = [-25, 0, 25]) {
        rotate([0, 0, ang * side])
          hull() {
            sphere(r=0.032, $fn=14);
            translate([side * 0.03, 0.16, -0.005]) sphere(r=0.022, $fn=12);
          }
      }
      // Foot webbing
      translate([side * 0.01, 0.08, -0.01])
        scale([1.1, 1.3, 0.25])
          cylinder(h=0.06, r=0.08, center=true, $fn=18);
    }
  }
}

module arm_assembly(side = 1) {
  frog_skin() {
    hull() {
      sphere(r=0.065, $fn=18);
      translate([side * 0.05, 0.08, -0.11]) sphere(r=0.05, $fn=18);
    }
    hull() {
      translate([side * 0.05, 0.08, -0.11]) sphere(r=0.048, $fn=18);
      translate([side * 0.02, 0.14, -0.22]) sphere(r=0.04, $fn=16);
    }
    // Little hand toes
    translate([side * 0.02, 0.16, -0.23]) {
      for (ang = [-22, 0, 22]) {
        rotate([0, 0, ang])
          hull() {
            sphere(r=0.022, $fn=12);
            translate([0, 0.07, 0]) sphere(r=0.016, $fn=12);
          }
      }
    }
  }
}

armature(animations=anim_data) {
  bone(name="Root", t=[0, 0, 0], r=[0, 0, 0]) {

    bone(name="Body", t=[0, 0, 0.28], r=[0, 0, 0]) {
      // Main plump torso
      frog_skin() {
        rotate([-18, 0, 0])
          scale([1.05, 1.15, 0.88])
            sphere(r=0.28, $fn=32);
      }

      // Cream belly patch
      frog_belly() {
        translate([0, 0.11, -0.08])
          rotate([-22, 0, 0])
            scale([0.85, 0.92, 0.72])
              sphere(r=0.24, $fn=28);
      }

      // Cute dorsal spots
      frog_spots() {
        translate([-0.11, -0.12, 0.20]) scale([1.0, 1.2, 0.45]) sphere(r=0.065, $fn=18);
        translate([0.10, -0.14, 0.19]) scale([1.0, 1.1, 0.45]) sphere(r=0.055, $fn=18);
        translate([0.0, -0.22, 0.12]) scale([1.1, 1.0, 0.45]) sphere(r=0.06, $fn=18);
      }

      // Upper Head & Wide Snout
      bone(name="Head", t=[0, 0.14, 0.16], r=[0, 0, 0]) {
        frog_skin() {
          hull() {
            translate([0, -0.02, 0.02]) scale([1.22, 1.0, 0.68]) sphere(r=0.22, $fn=28);
            translate([0, 0.18, -0.02]) scale([1.32, 0.75, 0.42]) sphere(r=0.19, $fn=28);
          }
        }

        // Upper mouth roof
        mouth_inside() {
          translate([0, 0.12, -0.055])
            scale([1.18, 0.95, 0.22])
              sphere(r=0.17, $fn=24);
        }

        // Cute nose bumps
        frog_spots() {
          translate([-0.06, 0.28, 0.03]) sphere(r=0.018, $fn=12);
          translate([0.06, 0.28, 0.03]) sphere(r=0.018, $fn=12);
        }

        // Left Eye (-X)
        bone(name="LeftEye", t=[-0.14, 0.07, 0.13], r=[0, 0, 0]) {
          frog_skin() {
            translate([0, 0, -0.02]) scale([1.0, 1.0, 1.05]) sphere(r=0.105, $fn=24);
          }
          eye_white() {
            translate([-0.01, 0.03, 0.01]) sphere(r=0.088, $fn=24);
          }
          eye_pupil() {
            translate([-0.01, 0.092, 0.02]) scale([0.85, 0.6, 1.1]) sphere(r=0.044, $fn=20);
          }
        }

        // Right Eye (+X)
        bone(name="RightEye", t=[0.14, 0.07, 0.13], r=[0, 0, 0]) {
          frog_skin() {
            translate([0, 0, -0.02]) scale([1.0, 1.0, 1.05]) sphere(r=0.105, $fn=24);
          }
          eye_white() {
            translate([0.01, 0.03, 0.01]) sphere(r=0.088, $fn=24);
          }
          eye_pupil() {
            translate([0.01, 0.092, 0.02]) scale([0.85, 0.6, 1.1]) sphere(r=0.044, $fn=20);
          }
        }
      }

      // Lower Jaw & Throat Pouch
      bone(name="Jaw", t=[0, 0.08, 0.04], r=[0, 0, 0]) {
        frog_belly() {
          hull() {
            translate([0, 0.04, -0.02]) scale([1.15, 0.85, 0.55]) sphere(r=0.18, $fn=24);
            translate([0, 0.22, 0.02]) scale([1.22, 0.68, 0.28]) sphere(r=0.17, $fn=24);
          }
        }
        mouth_inside() {
          translate([0, 0.14, 0.042])
            scale([1.10, 0.88, 0.18])
              sphere(r=0.16, $fn=22);
        }
      }

      // Built-in Tongue Tip (used in SCAD eat clip)
      bone(name="TongueTip", t=[0, 0.16, 0.02], r=[0, 0, 0]) {
        tongue_mat() {
          hull() {
            translate([0, -0.08, 0]) sphere(r=0.035, $fn=16);
            translate([0, 0.08, 0]) sphere(r=0.048, $fn=18);
          }
        }
      }

      // Front Arms
      bone(name="LeftArm", t=[-0.21, 0.14, -0.03], r=[0, 0, 0]) {
        arm_assembly(-1);
      }
      bone(name="RightArm", t=[0.21, 0.14, -0.03], r=[0, 0, 0]) {
        arm_assembly(1);
      }
    }

    // Hind Legs
    bone(name="LeftLeg", t=[-0.24, -0.10, 0.14], r=[0, 0, 0]) {
      leg_assembly(-1);
    }
    bone(name="RightLeg", t=[0.24, -0.10, 0.14], r=[0, 0, 0]) {
      leg_assembly(1);
    }
  }
}
```

### frog_fly_catcher/assets/fly.scad
```openscad
// Stylized Buzzing Fly (1 unit = 1 meter, +Z Up, +Y Forward, -X Left, +X Right)
$fn = 22;
$asa = 42;

anim_data = [
  ["buzz", [
    ["LeftWing", [
      [0.00, [0, 38, 12]],
      [0.04, [0, -35, 12]],
      [0.08, [0, 38, 12]]
    ]],
    ["RightWing", [
      [0.00, [0, -38, -12]],
      [0.04, [0, 35, -12]],
      [0.08, [0, -38, -12]]
    ]],
    ["FlyBody", [
      [0.00, [0, 0, 0], [0, 0, 0.00]],
      [0.04, [2, 0, 0], [0, 0, 0.02]],
      [0.08, [0, 0, 0], [0, 0, 0.00]]
    ]]
  ]]
];

module fly_body_mat() {
  color([0.11, 0.24, 0.30], name="fly_body", metalness=0.72, roughness=0.25, specularIntensity=0.9, $asa=45)
    children();
}

module fly_abdomen_mat() {
  color([0.14, 0.34, 0.24], name="fly_abdomen", metalness=0.65, roughness=0.30, $asa=45)
    children();
}

module fly_eye_mat() {
  color([0.88, 0.12, 0.16], name="fly_eye", metalness=0.1, roughness=0.12, specularIntensity=1.0, $asa=45)
    children();
}

module fly_wing_mat() {
  color([0.84, 0.95, 1.0], alpha=0.55, name="fly_wing", roughness=0.10, metalness=0.05, specularIntensity=1.0, $asa=35)
    children();
}

module fly_leg_mat() {
  color([0.07, 0.08, 0.10], name="fly_leg", roughness=0.45, $asa=30)
    children();
}

armature(animations=anim_data) {
  bone(name="FlyBody", t=[0, 0, 0], r=[0, 0, 0]) {
    // Thorax
    fly_body_mat() {
      scale([0.92, 1.15, 0.95]) sphere(r=0.105, $fn=24);
    }

    // Plump Abdomen (-Y)
    fly_abdomen_mat() {
      translate([0, -0.18, -0.015])
        rotate([-8, 0, 0])
          scale([0.95, 1.35, 0.92])
            sphere(r=0.115, $fn=26);
    }

    // Head (+Y)
    fly_body_mat() {
      translate([0, 0.13, 0.01])
        scale([1.05, 0.85, 0.95])
          sphere(r=0.082, $fn=22);
    }

    // Large Compound Eyes
    fly_eye_mat() {
      translate([-0.052, 0.155, 0.025]) scale([0.85, 1.0, 1.1]) sphere(r=0.055, $fn=20);
      translate([0.052, 0.155, 0.025]) scale([0.85, 1.0, 1.1]) sphere(r=0.055, $fn=20);
    }

    // Tiny Proboscis
    fly_leg_mat() {
      translate([0, 0.21, -0.03])
        rotate([65, 0, 0])
          cylinder(h=0.07, r1=0.018, r2=0.008, $fn=10);
    }

    // 6 Dangling Legs
    fly_leg_mat() {
      for (side = [-1, 1]) {
        for (y_off = [-0.04, 0.01, 0.06]) {
          translate([side * 0.055, y_off, -0.06])
            rotate([25, side * -28, 0])
              cylinder(h=0.11, r=0.011, center=true, $fn=10);
        }
      }
    }

    // Left Wing (-X)
    bone(name="LeftWing", t=[-0.045, 0.02, 0.085], r=[0, 20, 12]) {
      fly_wing_mat() {
        translate([-0.16, -0.07, 0])
          rotate([0, 0, 18])
            scale([1.85, 0.78, 0.08])
              sphere(r=0.11, $fn=22);
      }
    }

    // Right Wing (+X)
    bone(name="RightWing", t=[0.045, 0.02, 0.085], r=[0, -20, -12]) {
      fly_wing_mat() {
        translate([0.16, -0.07, 0])
          rotate([0, 0, -18])
            scale([1.85, 0.78, 0.08])
              sphere(r=0.11, $fn=22);
      }
    }
  }
}
```

### frog_fly_catcher/assets/wasp.scad
```openscad
// Stylized Hazard Wasp (1 unit = 1 meter, +Z Up, +Y Forward, -X Left, +X Right)
$fn = 22;
$asa = 42;

anim_data = [
  ["buzz", [
    ["LeftWing", [
      [0.00, [0, 36, 12]],
      [0.04, [0, -34, 12]],
      [0.08, [0, 36, 12]]
    ]],
    ["RightWing", [
      [0.00, [0, -36, -12]],
      [0.04, [0, 34, -12]],
      [0.08, [0, -36, -12]]
    ]],
    ["WaspBody", [
      [0.00, [0, 0, 0], [0, 0, 0.00]],
      [0.04, [-2, 0, 0], [0, 0, 0.02]],
      [0.08, [0, 0, 0], [0, 0, 0.00]]
    ]]
  ]]
];

module wasp_yellow() {
  color([1.0, 0.76, 0.04], name="wasp_yellow", roughness=0.28, metalness=0.1, emissive=[0.25, 0.14, 0.0], emissiveIntensity=0.8, $asa=42)
    children();
}

module wasp_black() {
  color([0.07, 0.06, 0.08], name="wasp_black", roughness=0.25, metalness=0.2, $asa=42)
    children();
}

module wasp_eye() {
  color([0.95, 0.25, 0.05], name="wasp_eye", roughness=0.15, emissive=[0.4, 0.05, 0.0], emissiveIntensity=1.2, $asa=45)
    children();
}

module wasp_wing() {
  color([1.0, 0.96, 0.85], alpha=0.58, name="wasp_wing", roughness=0.1, specularIntensity=1.0, $asa=35)
    children();
}

armature(animations=anim_data) {
  bone(name="WaspBody", t=[0, 0, 0], r=[0, 0, 0]) {
    // Thorax
    wasp_black() {
      scale([0.9, 1.15, 0.92]) sphere(r=0.105, $fn=22);
    }

    // Head (+Y)
    wasp_yellow() {
      translate([0, 0.14, 0.01])
        scale([1.05, 0.85, 0.95])
          sphere(r=0.078, $fn=20);
    }

    // Angry Eyes
    wasp_eye() {
      translate([-0.05, 0.165, 0.025]) scale([0.8, 1.0, 1.1]) sphere(r=0.045, $fn=18);
      translate([0.05, 0.165, 0.025]) scale([0.8, 1.0, 1.1]) sphere(r=0.045, $fn=18);
    }

    // Antennae (V-shaped, curving forward and outward)
    wasp_black() {
      for (side = [-1, 1]) {
        translate([side * 0.028, 0.19, 0.055])
          rotate([-42, side * 24, 0]) {
            cylinder(h=0.11, r1=0.010, r2=0.007, $fn=10);
            translate([0, 0, 0.11]) sphere(r=0.013, $fn=10);
          }
      }
    }

    // Striped Abdomen (-Y)
    translate([0, -0.20, -0.02]) rotate([-10, 0, 0]) {
      wasp_yellow() {
        scale([0.95, 1.45, 0.92]) sphere(r=0.115, $fn=28);
      }
      wasp_black() {
        translate([0, 0.05, 0]) scale([0.98, 0.34, 0.95]) sphere(r=0.117, $fn=26);
        translate([0, -0.035, 0]) scale([0.93, 0.34, 0.90]) sphere(r=0.115, $fn=26);
        translate([0, -0.115, 0]) scale([0.74, 0.34, 0.71]) sphere(r=0.104, $fn=24);
      }
      // Sharp Stinger (rooted inside abdomen tip)
      wasp_black() {
        translate([0, -0.135, 0])
          rotate([90, 0, 0])
            cylinder(h=0.12, r1=0.030, r2=0.002, $fn=14);
      }
    }

    // Wings
    bone(name="LeftWing", t=[-0.045, 0.02, 0.085], r=[0, 20, 12]) {
      wasp_wing() {
        translate([-0.17, -0.07, 0])
          rotate([0, 0, 16])
            scale([1.95, 0.72, 0.08])
              sphere(r=0.11, $fn=20);
      }
    }

    bone(name="RightWing", t=[0.045, 0.02, 0.085], r=[0, -20, -12]) {
      wasp_wing() {
        translate([0.17, -0.07, 0])
          rotate([0, 0, -16])
            scale([1.95, 0.72, 0.08])
              sphere(r=0.11, $fn=20);
      }
    }
  }
}
```

### frog_fly_catcher/assets/lily_pad_pond.scad
```openscad
// Lily Pad Pond Environment (1 unit = 1 meter, +Z Up, +Y Forward)
$fn = 36;
$asa = 32;

module water_mat() {
  color([0.08, 0.34, 0.46], name="pond_water", roughness=0.08, metalness=0.15, specularIntensity=1.0, $asa=45)
    children();
}

module pad_mat() {
  color([0.10, 0.42, 0.14], name="lily_pad", roughness=0.50, $asa=35)
    children();
}

module pad_vein_mat() {
  color([0.18, 0.56, 0.20], name="lily_pad_vein", roughness=0.52, $asa=30)
    children();
}

module lotus_petal_mat() {
  color([0.98, 0.46, 0.70], name="lotus_petal", roughness=0.32, $asa=40)
    children();
}

module lotus_center_mat() {
  color([0.98, 0.84, 0.22], name="lotus_center", roughness=0.40, $asa=35)
    children();
}

module stone_mat() {
  color([0.44, 0.46, 0.48], name="pond_stone", roughness=0.88, $asa=25)
    children();
}

module bank_grass_mat() {
  color([0.14, 0.40, 0.12], name="bank_grass", roughness=0.82, $asa=32)
    children();
}

module distant_hill_mat() {
  color([0.16, 0.44, 0.22], name="distant_hill", roughness=0.88, $asa=35)
    children();
}

module trunk_mat() {
  color([0.36, 0.22, 0.12], name="tree_trunk", roughness=0.88, $asa=25)
    children();
}

module foliage_dark_mat() {
  color([0.12, 0.42, 0.14], name="foliage_dark", roughness=0.72, $asa=35)
    children();
}

module foliage_light_mat() {
  color([0.22, 0.58, 0.20], name="foliage_light", roughness=0.70, $asa=35)
    children();
}

module cloud_mat() {
  color([0.97, 0.98, 1.0], name="cloud_white", roughness=0.95, emissive=[0.18, 0.20, 0.24], emissiveIntensity=0.45, $asa=42)
    children();
}

module lily_pad(rad = 1.35, notch_deg = 225) {
  pad_mat() {
    difference() {
      union() {
        cylinder(h=0.05, r=rad, center=false, $fn=44);
        // Subtle raised rim
        translate([0, 0, 0.045])
          rotate_extrude(angle=360, $fn=44)
            translate([rad - 0.028, 0, 0])
              circle(r=0.028, $fn=10);
      }
      // Classic V-notch cleanly cutting both pad and rim
      rotate([0, 0, notch_deg])
        translate([0, 0, -0.02])
          linear_extrude(height=0.14)
            polygon([[0, 0], [rad * 1.18, -rad * 0.24], [rad * 1.18, rad * 0.24]]);
    }
  }
  // Leaf veins radiating from center (staying clear of the V-notch)
  pad_vein_mat() {
    for (a = [0 : 40 : 280]) {
      rotate([0, 0, notch_deg + 40 + a])
        translate([rad * 0.45, 0, 0.052])
          cube([rad * 0.76, 0.024, 0.008], center=true);
    }
  }
}

module lotus_flower() {
  // Golden center pad
  lotus_center_mat() {
    translate([0, 0, 0.125])
      scale([1.0, 1.0, 0.65])
        sphere(r=0.095, $fn=18);
  }
  // Blooming upward & outward petals (negative Y rotation tilts +X tip upward!)
  lotus_petal_mat() {
    for (ring = [0, 1]) {
      count = (ring == 0) ? 8 : 6;
      tilt = (ring == 0) ? -28 : -56;
      rad_off = (ring == 0) ? 0.14 : 0.105;
      z_off = (ring == 0) ? 0.085 : 0.115;
      for (i = [0 : count - 1]) {
        rotate([0, 0, i * (360 / count) + ring * 25])
          translate([rad_off, 0, z_off])
            rotate([0, tilt, 0])
              scale([1.35, 0.72, 0.30])
                sphere(r=0.105, $fn=16);
      }
    }
  }
}

// Pond Water Surface (top at Z = -0.01)
water_mat() {
  translate([0, 0, -0.05])
    cylinder(h=0.04, r=24.0, center=false, $fn=64);
}

// Main Hero Lily Pad at origin (top at Z = 0.05)
translate([0, 0, 0])
  lily_pad(rad=1.42, notch_deg=35);

// Surrounding Decorative Lily Pads
translate([-2.8, 1.8, -0.005]) rotate([0, 0, 40]) lily_pad(rad=0.95, notch_deg=110);
translate([3.1, 2.2, -0.005]) rotate([0, 0, -25]) {
  lily_pad(rad=1.10, notch_deg=200);
  translate([0.1, 0.1, 0.02]) lotus_flower();
}
translate([-3.4, -1.6, -0.005]) rotate([0, 0, 120]) {
  lily_pad(rad=0.85, notch_deg=60);
  translate([-0.05, 0.05, 0.02]) lotus_flower();
}
translate([2.7, -1.8, -0.005]) rotate([0, 0, 270]) lily_pad(rad=0.78, notch_deg=15);
translate([0.6, 4.2, -0.005]) rotate([0, 0, 80]) lily_pad(rad=1.20, notch_deg=310);

module pond_tree(h = 3.4, crown_r = 1.45) {
  trunk_mat() {
    cylinder(h=h * 0.55, r1=0.22, r2=0.13, $fn=12);
    translate([0.12, 0.04, h * 0.36])
      rotate([15, 22, 0])
        cylinder(h=h * 0.25, r1=0.10, r2=0.05, $fn=10);
  }
  foliage_dark_mat() {
    translate([0, 0, h * 0.62]) scale([1.15, 1.1, 0.92]) sphere(r=crown_r, $fn=18);
    translate([-crown_r * 0.55, 0.2, h * 0.55]) scale([1.0, 0.95, 0.88]) sphere(r=crown_r * 0.72, $fn=16);
  }
  foliage_light_mat() {
    translate([crown_r * 0.48, -0.15, h * 0.68]) scale([1.0, 0.95, 0.85]) sphere(r=crown_r * 0.68, $fn=16);
    translate([0, 0, h * 0.82]) scale([1.0, 1.0, 0.82]) sphere(r=crown_r * 0.62, $fn=16);
  }
}

module pond_bush(r = 0.85) {
  foliage_dark_mat() {
    translate([0, 0, r * 0.45]) scale([1.25, 1.05, 0.78]) sphere(r=r, $fn=16);
  }
  foliage_light_mat() {
    translate([r * 0.45, -r * 0.15, r * 0.55]) scale([1.0, 0.95, 0.8]) sphere(r=r * 0.72, $fn=14);
    translate([-r * 0.42, r * 0.12, r * 0.52]) scale([1.0, 0.95, 0.78]) sphere(r=r * 0.66, $fn=14);
  }
}

module fluffy_cloud(s = 1.0) {
  cloud_mat() {
    scale([s, s, s]) {
      scale([1.4, 0.85, 0.75]) sphere(r=1.1, $fn=18);
      translate([-1.05, 0.1, -0.15]) scale([1.05, 0.8, 0.7]) sphere(r=0.85, $fn=16);
      translate([1.10, -0.1, -0.12]) scale([1.1, 0.82, 0.72]) sphere(r=0.90, $fn=16);
      translate([0.25, 0.15, 0.45]) scale([0.95, 0.78, 0.75]) sphere(r=0.80, $fn=16);
    }
  }
}

// Far Shoreline Grassy Bank (-Y in SCAD -> +Z in Godot, extending past screen edges)
bank_grass_mat() {
  for (i = [0 : 24]) {
    ang = 165 + i * 8.0;
    dist = 10.6 + (i % 3) * 1.1;
    translate([cos(ang) * dist, sin(ang) * dist, -0.15])
      rotate([0, 0, ang])
        scale([2.8, 2.3, 0.58 + (i % 2) * 0.2])
          sphere(r=1.60, $fn=18);
  }
}

// Distant Rolling Hills behind the pond (covering full horizontal FOV)
distant_hill_mat() {
  translate([-20.0, -16.0, -0.4]) scale([5.4, 3.0, 2.0]) sphere(r=2.3, $fn=20);
  translate([-11.0, -19.5, -0.4]) scale([4.8, 2.8, 1.9]) sphere(r=2.2, $fn=20);
  translate([-1.5, -21.5, -0.5]) scale([5.6, 3.0, 2.2]) sphere(r=2.3, $fn=20);
  translate([9.8, -19.0, -0.4]) scale([4.8, 2.8, 1.9]) sphere(r=2.2, $fn=20);
  translate([19.5, -15.5, -0.4]) scale([5.2, 3.0, 2.0]) sphere(r=2.3, $fn=20);
}

// Pond Shoreline Mossy Stones (-Y in SCAD -> +Z in Godot)
stone_mat() {
  for (i = [0 : 15]) {
    ang = 202 + i * 9.0;
    dist = 8.4 + (i % 3) * 0.75;
    translate([cos(ang) * dist, sin(ang) * dist, 0.04])
      rotate([0, 0, i * 37])
        scale([1.35 + (i % 2) * 0.45, 0.95, 0.52 + (i % 3) * 0.16])
          sphere(r=0.68, $fn=16);
  }
}

// Shoreline Trees & Bushes framing the view
translate([-8.2, -10.8, 0.25]) pond_tree(h=3.9, crown_r=1.55);
translate([-4.5, -12.6, 0.25]) pond_bush(r=0.95);
translate([3.2, -13.0, 0.25]) pond_bush(r=1.05);
translate([7.8, -11.2, 0.25]) pond_tree(h=3.6, crown_r=1.45);
translate([11.4, -9.2, 0.20]) pond_bush(r=1.15);
translate([-11.5, -8.5, 0.20]) pond_bush(r=1.10);
translate([-12.8, -4.0, 0.20]) pond_bush(r=1.20);
translate([12.6, -4.2, 0.20]) pond_bush(r=1.15);

// Sky Clouds
translate([-7.5, -21.0, 7.0]) rotate([0, 0, 12]) fluffy_cloud(s=1.15);
translate([6.2, -22.5, 7.6]) rotate([0, 0, -10]) fluffy_cloud(s=1.35);
translate([0.5, -24.0, 8.6]) fluffy_cloud(s=0.95);
```

### frog_fly_catcher/assets/cattail_reed.scad
```openscad
// Swaying Cattail Reed Clump (1 unit = 1 meter, +Z Up, +Y Forward)
$fn = 16;
$asa = 32;

anim_data = [
  ["sway", [
    ["ReedRoot", [
      [0.0, [0, 0, 0]],
      [1.2, [3, 4, 0]],
      [2.4, [0, 0, 0]],
      [3.6, [-3, -4, 0]],
      [4.8, [0, 0, 0]]
    ]]
  ]]
];

module stalk_mat() {
  color([0.18, 0.52, 0.18], name="reed_stalk", roughness=0.55, $asa=30)
    children();
}

module cattail_mat() {
  color([0.42, 0.22, 0.10], name="cattail_head", roughness=0.92, $asa=35)
    children();
}

armature(animations=anim_data) {
  bone(name="ReedRoot", t=[0, 0, 0], r=[0, 0, 0]) {
    // Tall stalk 1 with cattail
    translate([0, 0, 0]) rotate([4, -3, 0]) {
      stalk_mat() cylinder(h=2.1, r1=0.032, r2=0.016, $fn=12);
      cattail_mat() translate([0, 0, 1.65]) cylinder(h=0.34, r=0.058, $fn=16);
    }

    // Tall stalk 2 with cattail
    translate([0.16, 0.08, 0]) rotate([-5, 6, 0]) {
      stalk_mat() cylinder(h=1.75, r1=0.028, r2=0.014, $fn=12);
      cattail_mat() translate([0, 0, 1.38]) cylinder(h=0.28, r=0.052, $fn=16);
    }

    // Leaf blades
    stalk_mat() {
      for (i = [0 : 4]) {
        rotate([0, 0, i * 72])
          translate([0.08, 0, 0])
            rotate([16 + (i % 2) * 8, 0, 0])
              scale([0.8, 0.18, 1.0])
                cylinder(h=1.2 + (i % 3) * 0.25, r1=0.05, r2=0.005, $fn=8);
      }
    }
  }
}
```

### frog_fly_catcher/scripts/procedural_audio.gd
```gdscript
class_name ProceduralAudio
extends Node

const SAMPLE_RATE := 22050

var _players: Array[AudioStreamPlayer] = []
var _next_idx: int = 0

var _wav_jump: AudioStreamWAV
var _wav_tongue: AudioStreamWAV
var _wav_catch: AudioStreamWAV
var _wav_catch_gold: AudioStreamWAV
var _wav_sting: AudioStreamWAV
var _wav_miss: AudioStreamWAV


func _ready() -> void:
	for i in range(6):
		var player := AudioStreamPlayer.new()
		player.volume_db = -8.0
		add_child(player)
		_players.append(player)

	_wav_jump = _build_jump_wav()
	_wav_tongue = _build_tongue_wav()
	_wav_catch = _build_catch_wav(false)
	_wav_catch_gold = _build_catch_wav(true)
	_wav_sting = _build_sting_wav()
	_wav_miss = _build_miss_wav()


func _make_wav(samples: PackedFloat32Array) -> AudioStreamWAV:
	var wav := AudioStreamWAV.new()
	wav.format = AudioStreamWAV.FORMAT_16_BITS
	wav.mix_rate = SAMPLE_RATE
	wav.stereo = false
	var bytes := PackedByteArray()
	bytes.resize(samples.size() * 2)
	for i in range(samples.size()):
		var s := clampi(int(samples[i] * 32767.0), -32768, 32767)
		bytes.encode_s16(i * 2, s)
	wav.data = bytes
	return wav


func _play_wav(wav: AudioStreamWAV) -> void:
	if _players.is_empty() or wav == null:
		return
	var player := _players[_next_idx]
	_next_idx = (_next_idx + 1) % _players.size()
	player.stop()
	player.stream = wav
	player.play()


func _build_jump_wav() -> AudioStreamWAV:
	var duration := 0.16
	var total := int(SAMPLE_RATE * duration)
	var samples := PackedFloat32Array()
	samples.resize(total)
	var phase := 0.0
	for i in range(total):
		var t := float(i) / float(total)
		var freq := lerpf(190.0, 480.0, pow(t, 0.7))
		var env := sin(t * PI) * (1.0 - t * 0.35)
		phase += TAU * freq / SAMPLE_RATE
		samples[i] = sin(phase) * 0.45 * env
	return _make_wav(samples)


func _build_tongue_wav() -> AudioStreamWAV:
	var duration := 0.11
	var total := int(SAMPLE_RATE * duration)
	var samples := PackedFloat32Array()
	samples.resize(total)
	var phase := 0.0
	for i in range(total):
		var t := float(i) / float(total)
		var freq := lerpf(640.0, 260.0, t)
		var env := (1.0 - t) * sin(minf(1.0, t * 8.0) * PI * 0.5)
		phase += TAU * freq / SAMPLE_RATE
		samples[i] = (sin(phase) * 0.35 + sin(phase * 2.0) * 0.15) * env
	return _make_wav(samples)


func _build_catch_wav(is_golden: bool) -> AudioStreamWAV:
	var duration := 0.28 if is_golden else 0.20
	var total := int(SAMPLE_RATE * duration)
	var samples := PackedFloat32Array()
	samples.resize(total)
	var phase := 0.0
	for i in range(total):
		var t := float(i) / float(total)
		var freq := 523.25
		if is_golden:
			if t > 0.6:
				freq = 1046.5
			elif t > 0.3:
				freq = 783.99
			else:
				freq = 659.25
		else:
			freq = 783.99 if t > 0.35 else 523.25
		var env := exp(-t * 5.0)
		phase += TAU * freq / SAMPLE_RATE
		samples[i] = (sin(phase) * 0.42 + sin(phase * 2.0) * 0.12) * env
	return _make_wav(samples)


func _build_sting_wav() -> AudioStreamWAV:
	var duration := 0.32
	var total := int(SAMPLE_RATE * duration)
	var samples := PackedFloat32Array()
	samples.resize(total)
	var phase := 0.0
	for i in range(total):
		var t := float(i) / float(total)
		var freq := lerpf(185.0, 82.0, t)
		var env := (1.0 - t)
		phase += TAU * freq / SAMPLE_RATE
		var saw := fmod(phase / TAU, 1.0) * 2.0 - 1.0
		samples[i] = saw * 0.35 * env
	return _make_wav(samples)


func _build_miss_wav() -> AudioStreamWAV:
	var duration := 0.18
	var total := int(SAMPLE_RATE * duration)
	var samples := PackedFloat32Array()
	samples.resize(total)
	var phase := 0.0
	for i in range(total):
		var t := float(i) / float(total)
		var freq := lerpf(280.0, 165.0, t)
		var env := (1.0 - t) * 0.65
		phase += TAU * freq / SAMPLE_RATE
		samples[i] = sin(phase) * 0.3 * env
	return _make_wav(samples)


func play_jump() -> void:
	_play_wav(_wav_jump)


func play_tongue() -> void:
	_play_wav(_wav_tongue)


func play_catch(is_golden: bool) -> void:
	_play_wav(_wav_catch_gold if is_golden else _wav_catch)


func play_sting() -> void:
	_play_wav(_wav_sting)


func play_miss() -> void:
	_play_wav(_wav_miss)
```

### frog_fly_catcher/scripts/frog_player.gd
```gdscript
class_name FrogPlayer
extends CharacterBody3D

signal jumped
signal tongue_snapped(origin: Vector3, target: Vector3)

const GRAVITY := 23.0
const JUMP_IMPULSE := 9.2
const BASE_Y := 0.05

var is_active := false
var is_eating := false
var can_air_snap := false

var _anim_player: AnimationPlayer
var _tongue_pivot: Node3D
var _tongue_beam: MeshInstance3D
var _tongue_tip: MeshInstance3D
var _tongue_timer := 0.0
var _tongue_duration := 0.16
var _tongue_target_world := Vector3.ZERO
var _hurt_flash_timer := 0.0
var _hurt_overlay_meshes: Array[MeshInstance3D] = []


func _ready() -> void:
	_anim_player = _find_animation_player($FrogModel)
	_collect_meshes($FrogModel)
	_apply_realistic_skin_shaders()
	_setup_tongue_visual()
	_play_anim("idle")


func _apply_realistic_skin_shaders() -> void:
	var shader := Shader.new()
	shader.code = """
shader_type spatial;
render_mode cull_back, diffuse_burley, specular_schlick_ggx;

uniform vec3 u_base_color : source_color = vec3(0.11, 0.50, 0.16);
uniform vec3 u_mottle_color : source_color = vec3(0.05, 0.30, 0.09);
uniform vec3 u_highlight_color : source_color = vec3(0.28, 0.68, 0.22);
uniform float u_bump_strength = 0.12;
uniform float u_pore_scale = 18.0;
uniform float u_roughness_min = 0.08;
uniform float u_roughness_max = 0.24;

varying vec3 v_local_pos;
varying float v_dorsal;
varying vec3 v_nx;
varying vec3 v_ny;
varying vec3 v_nz;

float hash3(vec3 p) {
	p = fract(p * vec3(0.1031, 0.1030, 0.0973));
	p += dot(p, p.yxz + 33.33);
	return fract((p.x + p.y) * p.z);
}

float noise3(vec3 p) {
	vec3 i = floor(p);
	vec3 f = fract(p);
	f = f * f * (3.0 - 2.0 * f);

	float n000 = hash3(i + vec3(0.0, 0.0, 0.0));
	float n100 = hash3(i + vec3(1.0, 0.0, 0.0));
	float n010 = hash3(i + vec3(0.0, 1.0, 0.0));
	float n110 = hash3(i + vec3(1.0, 1.0, 0.0));
	float n001 = hash3(i + vec3(0.0, 0.0, 1.0));
	float n101 = hash3(i + vec3(1.0, 0.0, 1.0));
	float n011 = hash3(i + vec3(0.0, 1.0, 1.0));
	float n111 = hash3(i + vec3(1.0, 1.0, 1.0));

	float nx00 = mix(n000, n100, f.x);
	float nx10 = mix(n010, n110, f.x);
	float nx01 = mix(n001, n101, f.x);
	float nx11 = mix(n011, n111, f.x);

	float nxy0 = mix(nx00, nx10, f.y);
	float nxy1 = mix(nx01, nx11, f.y);

	return mix(nxy0, nxy1, f.z);
}

float fbm3(vec3 p) {
	float v = 0.0;
	float amp = 0.5;
	for (int i = 0; i < 4; i++) {
		v += amp * noise3(p);
		p *= 2.05;
		amp *= 0.5;
	}
	return v;
}

float skin_bumps(vec3 p) {
	float b1 = noise3(p);
	float b2 = noise3(p * 1.9 + vec3(5.7, 2.3, 9.1));
	return b1 * 0.68 + b2 * 0.32;
}

void vertex() {
	// Rest-pose 3D coordinates baked into UV/UV2 so pigments never swim during animations
	v_local_pos = vec3(UV.x, UV.y, UV2.x);
	v_dorsal = UV2.y;
	v_nx = normalize(MODELVIEW_MATRIX[0].xyz);
	v_ny = normalize(MODELVIEW_MATRIX[1].xyz);
	v_nz = normalize(MODELVIEW_MATRIX[2].xyz);
}

void fragment() {
	// Smooth organic mottling + dorsal/side gradient
	float macro = fbm3(v_local_pos * 5.2);
	float mid_pattern = fbm3(v_local_pos * 11.5 + vec3(4.2, 1.7, 8.1));
	float dorsal_mask = smoothstep(-0.35, 0.65, v_dorsal);

	vec3 skin_col = mix(u_highlight_color, u_base_color, smoothstep(0.28, 0.72, macro));
	skin_col = mix(skin_col, u_mottle_color, smoothstep(0.54, 0.78, mid_pattern) * (0.40 + 0.60 * dorsal_mask));

	// Subtle soft amphibian spots
	float spots = smoothstep(0.70, 0.82, noise3(v_local_pos * 14.0 + vec3(2.5)));
	skin_col = mix(skin_col, u_mottle_color * 0.78, spots * 0.35 * dorsal_mask);

	// Smooth 3D analytical-style bump gradient in local space -> view space (zero aliasing)
	vec3 bp = v_local_pos * u_pore_scale;
	float eps = 0.06;
	float h0 = skin_bumps(bp);
	vec3 grad_local = vec3(
		skin_bumps(bp + vec3(eps, 0.0, 0.0)) - h0,
		skin_bumps(bp + vec3(0.0, eps, 0.0)) - h0,
		skin_bumps(bp + vec3(0.0, 0.0, eps)) - h0
	) / eps;
	vec3 grad_view = v_nx * grad_local.x + v_ny * grad_local.y + v_nz * grad_local.z;
	vec3 surf_grad = grad_view - NORMAL * dot(NORMAL, grad_view);
	NORMAL = normalize(NORMAL - surf_grad * u_bump_strength);

	// Wet mucus Fresnel sheen & soft moisture variation
	float ndotv = clamp(dot(NORMAL, VIEW), 0.0, 1.0);
	float fresnel = pow(1.0 - ndotv, 3.0);
	float wetness = smoothstep(0.32, 0.74, h0);

	ALBEDO = skin_col + u_highlight_color * fresnel * 0.18;
	ROUGHNESS = mix(u_roughness_max, u_roughness_min, wetness * 0.65 + fresnel * 0.35);
	METALLIC = 0.0;
	SPECULAR = 0.90;
	EMISSION = skin_col * fresnel * 0.12;
}
"""

	var skin_mat := ShaderMaterial.new()
	skin_mat.shader = shader
	skin_mat.set_shader_parameter("u_base_color", Color(0.12, 0.52, 0.17))
	skin_mat.set_shader_parameter("u_mottle_color", Color(0.05, 0.28, 0.09))
	skin_mat.set_shader_parameter("u_highlight_color", Color(0.32, 0.68, 0.22))
	skin_mat.set_shader_parameter("u_bump_strength", 0.10)
	skin_mat.set_shader_parameter("u_pore_scale", 18.0)
	skin_mat.set_shader_parameter("u_roughness_min", 0.07)
	skin_mat.set_shader_parameter("u_roughness_max", 0.22)

	var spots_mat := ShaderMaterial.new()
	spots_mat.shader = shader
	spots_mat.set_shader_parameter("u_base_color", Color(0.05, 0.28, 0.09))
	spots_mat.set_shader_parameter("u_mottle_color", Color(0.02, 0.15, 0.04))
	spots_mat.set_shader_parameter("u_highlight_color", Color(0.11, 0.40, 0.13))
	spots_mat.set_shader_parameter("u_bump_strength", 0.11)
	spots_mat.set_shader_parameter("u_pore_scale", 20.0)
	spots_mat.set_shader_parameter("u_roughness_min", 0.09)
	spots_mat.set_shader_parameter("u_roughness_max", 0.24)

	var belly_mat := ShaderMaterial.new()
	belly_mat.shader = shader
	belly_mat.set_shader_parameter("u_base_color", Color(0.84, 0.86, 0.50))
	belly_mat.set_shader_parameter("u_mottle_color", Color(0.70, 0.74, 0.36))
	belly_mat.set_shader_parameter("u_highlight_color", Color(0.95, 0.95, 0.70))
	belly_mat.set_shader_parameter("u_bump_strength", 0.05)
	belly_mat.set_shader_parameter("u_pore_scale", 22.0)
	belly_mat.set_shader_parameter("u_roughness_min", 0.12)
	belly_mat.set_shader_parameter("u_roughness_max", 0.28)

	for m in _hurt_overlay_meshes:
		var orig_mesh := m.mesh as ArrayMesh
		if orig_mesh == null:
			continue

		var has_skin_surface := false
		for i in range(orig_mesh.get_surface_count()):
			var check_mat := m.get_active_material(i)
			if check_mat != null and check_mat.resource_name in ["frog_skin", "frog_spots", "frog_belly"]:
				has_skin_surface = true
				break

		if has_skin_surface:
			var new_mesh := ArrayMesh.new()
			for i in range(orig_mesh.get_surface_count()):
				var arrays := orig_mesh.surface_get_arrays(i)
				var verts: PackedVector3Array = arrays[Mesh.ARRAY_VERTEX]
				var norms: PackedVector3Array = arrays[Mesh.ARRAY_NORMAL]
				var uvs := PackedVector2Array()
				var uv2s := PackedVector2Array()
				uvs.resize(verts.size())
				uv2s.resize(verts.size())
				for v_idx in range(verts.size()):
					var v := verts[v_idx]
					var ny := norms[v_idx].y if v_idx < norms.size() else 0.0
					uvs[v_idx] = Vector2(v.x, v.y)
					uv2s[v_idx] = Vector2(v.z, ny)
				arrays[Mesh.ARRAY_TEX_UV] = uvs
				arrays[Mesh.ARRAY_TEX_UV2] = uv2s
				var flags := orig_mesh.surface_get_format(i) & Mesh.ARRAY_FLAG_USE_8_BONE_WEIGHTS
				new_mesh.add_surface_from_arrays(Mesh.PRIMITIVE_TRIANGLES, arrays, [], {}, flags)
				new_mesh.surface_set_material(i, orig_mesh.surface_get_material(i))
			m.mesh = new_mesh

		for i in range(m.mesh.get_surface_count()):
			var mat := m.get_active_material(i)
			if mat == null:
				continue
			match mat.resource_name:
				"frog_skin":
					m.set_surface_override_material(i, skin_mat)
				"frog_spots":
					m.set_surface_override_material(i, spots_mat)
				"frog_belly":
					m.set_surface_override_material(i, belly_mat)


func reset_to_pad() -> void:
	global_position = Vector3(0.0, BASE_Y, 0.0)
	velocity = Vector3.ZERO
	is_eating = false
	can_air_snap = false
	_tongue_timer = 0.0
	_tongue_pivot.visible = false
	_play_anim("idle")


func _setup_tongue_visual() -> void:
	_tongue_pivot = Node3D.new()
	_tongue_pivot.name = "DynamicTongue"
	_tongue_pivot.position = Vector3(0.0, 0.42, -0.24)
	_tongue_pivot.visible = false
	add_child(_tongue_pivot)

	var mat := StandardMaterial3D.new()
	mat.albedo_color = Color(0.96, 0.28, 0.45)
	mat.roughness = 0.25

	var cyl := CylinderMesh.new()
	cyl.top_radius = 0.025
	cyl.bottom_radius = 0.032
	cyl.height = 1.0
	cyl.radial_segments = 12

	_tongue_beam = MeshInstance3D.new()
	_tongue_beam.mesh = cyl
	_tongue_beam.material_override = mat
	# Rotate cylinder so its length aligns along -Z (Godot forward)
	_tongue_beam.rotation_degrees.x = 90.0
	_tongue_pivot.add_child(_tongue_beam)

	var tip_sphere := SphereMesh.new()
	tip_sphere.radius = 0.055
	tip_sphere.height = 0.11
	_tongue_tip = MeshInstance3D.new()
	_tongue_tip.mesh = tip_sphere
	_tongue_tip.material_override = mat
	_tongue_pivot.add_child(_tongue_tip)


func _collect_meshes(node: Node) -> void:
	if node is MeshInstance3D:
		_hurt_overlay_meshes.append(node)
	for child in node.get_children():
		_collect_meshes(child)


func _find_animation_player(node: Node) -> AnimationPlayer:
	if node is AnimationPlayer:
		return node
	for child in node.get_children():
		var found := _find_animation_player(child)
		if found != null:
			return found
	return null


func _play_anim(anim_name: String) -> void:
	if _anim_player == null:
		return
	if _anim_player.has_animation(anim_name):
		if anim_name == "idle":
			var anim := _anim_player.get_animation("idle")
			if anim != null:
				anim.loop_mode = Animation.LOOP_LINEAR
		_anim_player.play(anim_name, 0.08)


func _physics_process(delta: float) -> void:
	if _hurt_flash_timer > 0.0:
		_hurt_flash_timer -= delta
		var flash := _hurt_flash_timer > 0.0 and int(_hurt_flash_timer * 20.0) % 2 == 0
		for m in _hurt_overlay_meshes:
			m.visible = not flash
	else:
		for m in _hurt_overlay_meshes:
			m.visible = true

	if _tongue_timer > 0.0:
		_tongue_timer -= delta
		_update_tongue_transform()
		if _tongue_timer <= 0.0:
			_tongue_pivot.visible = false
			is_eating = false
			if is_on_floor_pad():
				_play_anim("idle")

	if not is_active:
		return

	velocity.x = 0.0
	velocity.z = 0.0

	# Gravity
	if not is_on_floor_pad() or velocity.y > 0.0:
		velocity.y -= GRAVITY * delta

	global_position += velocity * delta

	# Land on lily pad
	if global_position.y <= BASE_Y:
		global_position.y = BASE_Y
		if velocity.y < 0.0:
			velocity.y = 0.0
			can_air_snap = false
			if not is_eating:
				_play_anim("idle")

	# Variable jump cut for quick short hops
	if velocity.y > 1.5 and not Input.is_action_pressed("jump") and not Input.is_mouse_button_pressed(MOUSE_BUTTON_LEFT):
		velocity.y *= 0.86


func _unhandled_input(event: InputEvent) -> void:
	if not is_active:
		return
	var triggered := event.is_action_pressed("jump")
	if event is InputEventMouseButton and event.button_index == MOUSE_BUTTON_LEFT and event.pressed:
		triggered = true

	if not triggered:
		return

	if is_on_floor_pad():
		velocity.y = JUMP_IMPULSE
		can_air_snap = true
		_play_anim("jump")
		jumped.emit()
	elif can_air_snap:
		# Manual mid-air tongue strike!
		can_air_snap = false
		var default_target := get_mouth_world_pos() + Vector3(0.0, 0.9, -0.55)
		trigger_tongue_snap(default_target)
		tongue_snapped.emit(get_mouth_world_pos(), default_target)


func is_on_floor_pad() -> bool:
	return global_position.y <= BASE_Y + 0.02 and velocity.y <= 0.05


func get_mouth_world_pos() -> Vector3:
	return global_position + Vector3(0.0, 0.42, -0.24)


func trigger_tongue_snap(target_world: Vector3) -> void:
	is_eating = true
	_tongue_target_world = target_world
	_tongue_timer = _tongue_duration
	_tongue_pivot.visible = true
	_play_anim("eat")
	_update_tongue_transform()


func trigger_hurt_flash() -> void:
	_hurt_flash_timer = 0.65


func _update_tongue_transform() -> void:
	var progress := 1.0 - clampf(_tongue_timer / _tongue_duration, 0.0, 1.0)
	# Shoot out fast in first 35%, retract over remaining 65%
	var ext := 0.0
	if progress < 0.35:
		ext = progress / 0.35
	else:
		ext = 1.0 - (progress - 0.35) / 0.65

	var mouth := get_mouth_world_pos()
	var diff := _tongue_target_world - mouth
	var dist := maxf(0.15, diff.length()) * ext
	if diff.length_squared() > 0.0001:
		_tongue_pivot.look_at(_tongue_target_world, Vector3.UP)

	_tongue_beam.scale = Vector3(1.0, dist, 1.0)
	_tongue_beam.position = Vector3(0.0, 0.0, -dist * 0.5)
	_tongue_tip.position = Vector3(0.0, 0.0, -dist)
```

### frog_fly_catcher/scripts/fly_entity.gd
```gdscript
class_name FlyEntity
extends Node3D

signal escaped(fly: FlyEntity)

enum FlyType { NORMAL, GOLDEN }

var fly_type: FlyType = FlyType.NORMAL
var direction: float = 1.0
var speed: float = 3.2
var base_y: float = 1.6
var lane_z: float = -0.35
var wave_amp: float = 0.25
var wave_freq: float = 4.0
var _age: float = 0.0
var _caught: bool = false
var _anim_player: AnimationPlayer
var _shadow_ring: MeshInstance3D


func _ready() -> void:
	_anim_player = _find_animation_player($FlyModel)
	if _anim_player != null and _anim_player.has_animation("buzz"):
		var anim := _anim_player.get_animation("buzz")
		if anim != null:
			anim.loop_mode = Animation.LOOP_LINEAR
		_anim_player.play("buzz")

	if fly_type == FlyType.GOLDEN:
		_apply_golden_material($FlyModel)

	_create_height_shadow()
	_orient_along_flight(Vector3(direction, 0.0, 0.0))


func _create_height_shadow() -> void:
	_shadow_ring = MeshInstance3D.new()
	var torus := TorusMesh.new()
	torus.inner_radius = 0.14
	torus.outer_radius = 0.19
	_shadow_ring.mesh = torus
	var mat := StandardMaterial3D.new()
	mat.albedo_color = Color(1.0, 0.85, 0.2, 0.65) if fly_type == FlyType.GOLDEN else Color(0.15, 0.95, 0.55, 0.45)
	mat.transparency = BaseMaterial3D.TRANSPARENCY_ALPHA
	mat.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	_shadow_ring.material_override = mat
	add_child(_shadow_ring)


func _apply_golden_material(node: Node) -> void:
	if node is MeshInstance3D and node.mesh != null:
		var m := node as MeshInstance3D
		for i in range(m.mesh.get_surface_count()):
			var surf_mat := m.get_active_material(i)
			if surf_mat != null and (surf_mat.resource_name == "fly_body" or surf_mat.resource_name == "fly_abdomen"):
				var gold := StandardMaterial3D.new()
				gold.albedo_color = Color(1.0, 0.82, 0.18)
				gold.metallic = 0.9
				gold.roughness = 0.18
				gold.emission_enabled = true
				gold.emission = Color(1.0, 0.72, 0.1)
				gold.emission_energy_multiplier = 1.6
				m.set_surface_override_material(i, gold)
	for child in node.get_children():
		_apply_golden_material(child)


func _find_animation_player(node: Node) -> AnimationPlayer:
	if node is AnimationPlayer:
		return node
	for child in node.get_children():
		var found := _find_animation_player(child)
		if found != null:
			return found
	return null


func _orient_along_flight(vel: Vector3) -> void:
	if vel.length_squared() > 0.001:
		# OpenSCAD +Y faces Godot -Z, so look_at faces flight direction naturally
		look_at(global_position + vel.normalized(), Vector3.UP)


func _process(delta: float) -> void:
	if _caught:
		return
	_age += delta
	var vx := direction * speed
	var vy := cos(_age * wave_freq) * wave_amp * wave_freq
	global_position.x += vx * delta
	global_position.y = base_y + sin(_age * wave_freq) * wave_amp
	global_position.z = lane_z
	_orient_along_flight(Vector3(vx, vy * 0.4, 0.0))

	if _shadow_ring != null:
		_shadow_ring.global_position = Vector3(global_position.x, 0.06, global_position.z)
		_shadow_ring.global_rotation = Vector3.ZERO
		var in_zone := absf(global_position.x) < 1.25
		var s := 1.25 if in_zone else 0.85
		_shadow_ring.scale = Vector3(s, 1.0, s)

	if (direction > 0.0 and global_position.x > 6.8) or (direction < 0.0 and global_position.x < -6.8):
		escaped.emit(self)
		queue_free()


func catch_and_consume() -> void:
	if _caught:
		return
	_caught = true
	queue_free()


func is_caught() -> bool:
	return _caught
```

### frog_fly_catcher/scripts/wasp_entity.gd
```gdscript
class_name WaspEntity
extends Node3D

signal passed(wasp: WaspEntity)

var direction: float = 1.0
var speed: float = 3.5
var base_y: float = 1.65
var lane_z: float = -0.35
var wave_amp: float = 0.30
var wave_freq: float = 5.0
var _age: float = 0.0
var _consumed: bool = false
var _anim_player: AnimationPlayer
var _warn_ring: MeshInstance3D


func _ready() -> void:
	_anim_player = _find_animation_player($WaspModel)
	if _anim_player != null and _anim_player.has_animation("buzz"):
		var anim := _anim_player.get_animation("buzz")
		if anim != null:
			anim.loop_mode = Animation.LOOP_LINEAR
		_anim_player.play("buzz")
	_create_warning_ring()
	_orient_along_flight(Vector3(direction, 0.0, 0.0))


func _create_warning_ring() -> void:
	_warn_ring = MeshInstance3D.new()
	var torus := TorusMesh.new()
	torus.inner_radius = 0.16
	torus.outer_radius = 0.22
	_warn_ring.mesh = torus
	var mat := StandardMaterial3D.new()
	mat.albedo_color = Color(1.0, 0.2, 0.12, 0.72)
	mat.transparency = BaseMaterial3D.TRANSPARENCY_ALPHA
	mat.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	_warn_ring.material_override = mat
	add_child(_warn_ring)


func _find_animation_player(node: Node) -> AnimationPlayer:
	if node is AnimationPlayer:
		return node
	for child in node.get_children():
		var found := _find_animation_player(child)
		if found != null:
			return found
	return null


func _orient_along_flight(vel: Vector3) -> void:
	if vel.length_squared() > 0.001:
		look_at(global_position + vel.normalized(), Vector3.UP)


func _process(delta: float) -> void:
	if _consumed:
		return
	_age += delta
	var vx := direction * speed
	var vy := cos(_age * wave_freq) * wave_amp * wave_freq
	global_position.x += vx * delta
	global_position.y = base_y + sin(_age * wave_freq) * wave_amp
	global_position.z = lane_z
	_orient_along_flight(Vector3(vx, vy * 0.4, 0.0))

	if _warn_ring != null:
		_warn_ring.global_position = Vector3(global_position.x, 0.06, global_position.z)
		_warn_ring.global_rotation = Vector3.ZERO

	if (direction > 0.0 and global_position.x > 6.8) or (direction < 0.0 and global_position.x < -6.8):
		passed.emit(self)
		queue_free()


func consume_sting() -> void:
	if _consumed:
		return
	_consumed = true
	queue_free()


func is_consumed() -> bool:
	return _consumed
```

### frog_fly_catcher/scripts/main_game.gd
```gdscript
extends Node3D

enum GameState { MENU, PLAYING, GAME_OVER }

const MAX_HEARTS := 3
const MISSES_PER_HEART := 3

@onready var _camera: Camera3D = $Camera3D
@onready var _sun: DirectionalLight3D = $DirectionalLight3D
@onready var _frog: FrogPlayer = $FrogPlayer
@onready var _entities_container: Node3D = $EntitiesContainer
@onready var _particles_container: Node3D = $ParticlesContainer
@onready var _pond_model: Node3D = $LilyPadPond
@onready var _reeds_container: Node3D = $Reeds

@onready var _score_label: Label = $HUD/TopBar/ScoreLabel
@onready var _combo_label: Label = $HUD/TopBar/ComboLabel
@onready var _hearts_icons_box: HBoxContainer = $HUD/TopBar/HeartsBox/HeartsIcons
@onready var _misses_label: Label = $HUD/TopBar/MissesLabel
@onready var _best_label: Label = $HUD/TopBar/BestLabel
@onready var _popup_label: Label = $HUD/PopupLabel

@onready var _menu_panel: PanelContainer = $HUD/MenuPanel
@onready var _start_button: Button = $HUD/MenuPanel/VBox/StartButton
@onready var _gameover_panel: PanelContainer = $HUD/GameOverPanel
@onready var _final_score_label: Label = $HUD/GameOverPanel/VBox/FinalScoreLabel
@onready var _retry_button: Button = $HUD/GameOverPanel/VBox/RetryButton

var _fly_scene: PackedScene
var _wasp_scene: PackedScene
var _audio: ProceduralAudio

var _state: GameState = GameState.MENU
var _score := 0
var _best_score := 0
var _combo := 1
var _caught_count := 0
var _hearts := MAX_HEARTS
var _misses := 0
var _spawn_timer := 0.8
var _elapsed_play := 0.0
var _popup_timer := 0.0
var _cam_base_pos := Vector3(1.5, 2.35, -6.4)
var _cam_shake := 0.0
var _water_shader_mat: ShaderMaterial


func _ready() -> void:
	_fly_scene = load("res://scenes/fly_entity.tscn")
	_wasp_scene = load("res://scenes/wasp_entity.tscn")

	_audio = ProceduralAudio.new()
	add_child(_audio)

	_camera.global_position = _cam_base_pos
	_camera.look_at(Vector3(0.0, 1.35, 0.0), Vector3.UP)
	_sun.look_at(Vector3.ZERO, Vector3.UP)

	_frog.jumped.connect(_on_frog_jumped)
	_frog.tongue_snapped.connect(_on_frog_manual_snap)

	_start_button.pressed.connect(_start_game)
	_retry_button.pressed.connect(_start_game)

	_apply_pond_water_shader()
	_start_reed_animations()
	_build_heart_icons()
	_update_hud()
	_show_menu()


func _apply_pond_water_shader() -> void:
	var shader := Shader.new()
	shader.code = """
shader_type spatial;
render_mode specular_schlick_ggx;

uniform float u_time = 0.0;

void fragment() {
	vec3 world_pos = (INV_VIEW_MATRIX * vec4(VERTEX, 1.0)).xyz;
	float dist = length(world_pos.xz);
	float ripple = sin(dist * 7.5 - u_time * 3.2) * 0.5 + 0.5;
	float caustic = sin(world_pos.x * 2.4 + u_time * 1.4) * cos(world_pos.z * 2.4 - u_time * 1.1);
	vec3 deep = vec3(0.06, 0.28, 0.42);
	vec3 shallow = vec3(0.14, 0.52, 0.62);
	ALBEDO = mix(deep, shallow, ripple * 0.22 + caustic * 0.12 + 0.2);
	ROUGHNESS = 0.09;
	METALLIC = 0.12;
	SPECULAR = 0.85;
}
"""
	_water_shader_mat = ShaderMaterial.new()
	_water_shader_mat.shader = shader
	_replace_material_by_name(_pond_model, "pond_water", _water_shader_mat)


func _replace_material_by_name(node: Node, target_name: String, replacement: Material) -> void:
	if node is MeshInstance3D and node.mesh != null:
		var m := node as MeshInstance3D
		for i in range(m.mesh.get_surface_count()):
			var surf_mat := m.get_active_material(i)
			if surf_mat != null and surf_mat.resource_name == target_name:
				m.set_surface_override_material(i, replacement)
	for child in node.get_children():
		_replace_material_by_name(child, target_name, replacement)


func _start_reed_animations() -> void:
	for reed in _reeds_container.get_children():
		var anim_player := _find_animation_player(reed)
		if anim_player != null and anim_player.has_animation("sway"):
			var anim := anim_player.get_animation("sway")
			if anim != null:
				anim.loop_mode = Animation.LOOP_LINEAR
			anim_player.speed_scale = randf_range(0.8, 1.2)
			anim_player.play("sway")


func _find_animation_player(node: Node) -> AnimationPlayer:
	if node is AnimationPlayer:
		return node
	for child in node.get_children():
		var found := _find_animation_player(child)
		if found != null:
			return found
	return null


func _show_menu() -> void:
	_state = GameState.MENU
	_frog.is_active = false
	_frog.reset_to_pad()
	_menu_panel.visible = true
	_gameover_panel.visible = false
	_popup_label.visible = false


func _start_game() -> void:
	_clear_entities()
	_score = 0
	_combo = 1
	_caught_count = 0
	_hearts = MAX_HEARTS
	_misses = 0
	_elapsed_play = 0.0
	_spawn_timer = 0.6

	_frog.reset_to_pad()
	_frog.is_active = true
	_state = GameState.PLAYING
	_menu_panel.visible = false
	_gameover_panel.visible = false
	_show_popup("CATCH THE FLIES!", Color(0.55, 1.0, 0.55))
	_update_hud()


func _clear_entities() -> void:
	for child in _entities_container.get_children():
		child.queue_free()


func _process(delta: float) -> void:
	if _water_shader_mat != null:
		_water_shader_mat.set_shader_parameter("u_time", Time.get_ticks_msec() * 0.001)

	if _cam_shake > 0.0:
		_cam_shake = maxf(0.0, _cam_shake - delta * 2.5)
		var offset := Vector3(
			randf_range(-0.12, 0.12) * _cam_shake,
			randf_range(-0.10, 0.10) * _cam_shake,
			0.0
		)
		_camera.global_position = _cam_base_pos + offset
	else:
		_camera.global_position = _cam_base_pos

	if _popup_timer > 0.0:
		_popup_timer -= delta
		if _popup_timer <= 0.0:
			_popup_label.visible = false

	if _state != GameState.PLAYING:
		return

	_elapsed_play += delta
	_spawn_timer -= delta
	if _spawn_timer <= 0.0:
		_spawn_next_bug()
		var min_gap := maxf(0.65, 1.45 - _elapsed_play * 0.012)
		var max_gap := maxf(1.05, 2.25 - _elapsed_play * 0.014)
		_spawn_timer = randf_range(min_gap, max_gap)

	_check_airborne_auto_catch()


func _spawn_next_bug() -> void:
	var dir := 1.0 if randf() < 0.5 else -1.0
	var speed_boost := minf(2.6, _elapsed_play * 0.032)

	# Occasional Wasp once player has warmed up
	if _caught_count >= 2 and randf() < 0.22:
		var wasp := _wasp_scene.instantiate() as WaspEntity
		wasp.direction = dir
		wasp.speed = randf_range(2.9, 4.1) + speed_boost * 0.75
		wasp.base_y = randf_range(1.05, 2.35)
		wasp.wave_amp = randf_range(0.18, 0.36)
		wasp.wave_freq = randf_range(3.8, 5.5)
		wasp.position = Vector3(-dir * 6.4, wasp.base_y, wasp.lane_z)
		_entities_container.add_child(wasp)
		return

	var fly := _fly_scene.instantiate() as FlyEntity
	var is_gold := _caught_count >= 3 and randf() < 0.18
	fly.fly_type = FlyEntity.FlyType.GOLDEN if is_gold else FlyEntity.FlyType.NORMAL
	fly.direction = dir
	fly.speed = (randf_range(3.8, 4.9) if is_gold else randf_range(2.6, 3.8)) + speed_boost
	fly.base_y = randf_range(1.05, 2.55)
	fly.wave_amp = randf_range(0.12, 0.38)
	fly.wave_freq = randf_range(3.0, 5.2)
	fly.position = Vector3(-dir * 6.4, fly.base_y, fly.lane_z)
	fly.escaped.connect(_on_fly_escaped)
	_entities_container.add_child(fly)


func _check_airborne_auto_catch() -> void:
	if _frog.is_on_floor_pad():
		return

	var mouth := _frog.get_mouth_world_pos()
	for child in _entities_container.get_children():
		if child is FlyEntity:
			var fly := child as FlyEntity
			if fly.is_caught():
				continue
			var dist := mouth.distance_to(fly.global_position)
			if dist <= 1.15:
				_catch_fly(fly)
				return
		elif child is WaspEntity:
			var wasp := child as WaspEntity
			if wasp.is_consumed():
				continue
			var dist := mouth.distance_to(wasp.global_position)
			if dist <= 0.78:
				_hit_wasp(wasp)
				return


func _on_frog_jumped() -> void:
	_audio.play_jump()


func _on_frog_manual_snap(origin: Vector3, _default_target: Vector3) -> void:
	_audio.play_tongue()
	if _state != GameState.PLAYING:
		return

	# Check extended tongue strike range against flies & wasps
	var best_fly: FlyEntity = null
	var best_fly_dist := 2.15
	for child in _entities_container.get_children():
		if child is FlyEntity:
			var fly := child as FlyEntity
			if fly.is_caught():
				continue
			var d := origin.distance_to(fly.global_position)
			if d < best_fly_dist:
				best_fly_dist = d
				best_fly = fly

	if best_fly != null:
		_catch_fly(best_fly)
		return

	for child in _entities_container.get_children():
		if child is WaspEntity:
			var wasp := child as WaspEntity
			if not wasp.is_consumed() and origin.distance_to(wasp.global_position) < 1.65:
				_hit_wasp(wasp)
				return


func _catch_fly(fly: FlyEntity) -> void:
	var target_pos := fly.global_position
	var is_gold := fly.fly_type == FlyEntity.FlyType.GOLDEN
	fly.catch_and_consume()

	_frog.trigger_tongue_snap(target_pos)
	_audio.play_catch(is_gold)
	_spawn_burst(target_pos, Color(1.0, 0.85, 0.2) if is_gold else Color(0.55, 1.0, 0.45))

	var base_pts := 300 if is_gold else 100
	var pts := base_pts * _combo
	_score += pts
	_caught_count += 1
	_combo = mini(10, _combo + 1)
	_misses = 0
	if _score > _best_score:
		_best_score = _score

	_cam_shake = 0.28 if is_gold else 0.14
	var label_text := "+%d GOLDEN!" % pts if is_gold else "+%d" % pts
	_show_popup(label_text, Color(1.0, 0.88, 0.25) if is_gold else Color(0.6, 1.0, 0.55))
	_update_hud()


func _hit_wasp(wasp: WaspEntity) -> void:
	var target_pos := wasp.global_position
	wasp.consume_sting()

	_frog.trigger_tongue_snap(target_pos)
	_frog.trigger_hurt_flash()
	_audio.play_sting()
	_spawn_burst(target_pos, Color(1.0, 0.25, 0.15))

	_combo = 1
	_hearts -= 1
	_cam_shake = 0.75
	_show_popup("OUCH! WASP!", Color(1.0, 0.3, 0.25))
	_update_hud()

	if _hearts <= 0:
		_trigger_game_over()


func _on_fly_escaped(_fly: FlyEntity) -> void:
	if _state != GameState.PLAYING:
		return
	_combo = 1
	_misses += 1
	_audio.play_miss()
	if _misses >= MISSES_PER_HEART:
		_misses = 0
		_hearts -= 1
		_show_popup("ESCAPED! -1 LIFE", Color(1.0, 0.55, 0.3))
		if _hearts <= 0:
			_update_hud()
			_trigger_game_over()
			return
	_update_hud()


func _trigger_game_over() -> void:
	_state = GameState.GAME_OVER
	_frog.is_active = false
	_final_score_label.text = "Final Score: %d\nFlies Caught: %d\nBest Score: %d" % [_score, _caught_count, _best_score]
	_gameover_panel.visible = true


func _show_popup(text: String, col: Color) -> void:
	_popup_label.text = text
	_popup_label.add_theme_color_override("font_color", col)
	_popup_label.visible = true
	_popup_timer = 0.75


func _spawn_burst(pos: Vector3, col: Color) -> void:
	var particles := CPUParticles3D.new()
	particles.emitting = true
	particles.one_shot = true
	particles.explosiveness = 0.95
	particles.amount = 16
	particles.lifetime = 0.45
	particles.initial_velocity_min = 1.6
	particles.initial_velocity_max = 3.5
	particles.spread = 180.0
	particles.gravity = Vector3(0.0, -5.0, 0.0)
	particles.position = pos

	var sphere := SphereMesh.new()
	sphere.radius = 0.045
	sphere.height = 0.09
	var mat := StandardMaterial3D.new()
	mat.albedo_color = col
	mat.emission_enabled = true
	mat.emission = col
	mat.emission_energy_multiplier = 1.8
	sphere.material = mat
	particles.mesh = sphere

	_particles_container.add_child(particles)
	get_tree().create_timer(0.6).timeout.connect(particles.queue_free)


func _build_heart_icons() -> void:
	for child in _hearts_icons_box.get_children():
		child.queue_free()
	for i in range(MAX_HEARTS):
		var heart := _HeartWidget.new()
		heart.custom_minimum_size = Vector2(24, 24)
		heart.size_flags_vertical = Control.SIZE_SHRINK_CENTER
		_hearts_icons_box.add_child(heart)


func _update_hud() -> void:
	_score_label.text = "Score: %d" % _score
	_combo_label.text = "Combo: x%d" % _combo
	var icons := _hearts_icons_box.get_children()
	for i in range(icons.size()):
		if icons[i] is _HeartWidget:
			(icons[i] as _HeartWidget).set_filled(i < _hearts)
	_misses_label.text = "Misses: %d/%d" % [_misses, MISSES_PER_HEART]
	_best_label.text = "Best: %d" % _best_score


class _HeartWidget extends Control:
	var _filled := true

	func set_filled(val: bool) -> void:
		if _filled != val:
			_filled = val
			queue_redraw()

	func _draw() -> void:
		var pts := PackedVector2Array([
			Vector2(12, 22),
			Vector2(2, 12),
			Vector2(2, 7),
			Vector2(6, 4),
			Vector2(10, 5),
			Vector2(12, 8),
			Vector2(14, 5),
			Vector2(18, 4),
			Vector2(22, 7),
			Vector2(22, 12),
		])
		var fill_col := Color(1.0, 0.28, 0.36) if _filled else Color(0.18, 0.22, 0.26, 0.65)
		draw_colored_polygon(pts, fill_col)
		var outline := pts.duplicate()
		outline.append(pts[0])
		draw_polyline(outline, Color(0.08, 0.12, 0.16), 2.0, true)
```

### frog_fly_catcher/scenes/frog_player.tscn
```text
[gd_scene load_steps=4 format=3]

[ext_resource type="Script" path="res://scripts/frog_player.gd" id="1_script"]
[ext_resource type="PackedScene" path="res://assets/frog.scad" id="2_frog"]

[sub_resource type="SphereShape3D" id="SphereShape3D_body"]
radius = 0.34

[node name="FrogPlayer" type="CharacterBody3D"]
script = ExtResource("1_script")

[node name="FrogModel" parent="." instance=ExtResource("2_frog")]

[node name="CollisionShape3D" type="CollisionShape3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.32, 0)
shape = SubResource("SphereShape3D_body")
```

### frog_fly_catcher/scenes/fly_entity.tscn
```text
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/fly_entity.gd" id="1_script"]
[ext_resource type="PackedScene" path="res://assets/fly.scad" id="2_fly"]

[node name="FlyEntity" type="Node3D"]
script = ExtResource("1_script")

[node name="FlyModel" parent="." instance=ExtResource("2_fly")]
```

### frog_fly_catcher/scenes/wasp_entity.tscn
```text
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/wasp_entity.gd" id="1_script"]
[ext_resource type="PackedScene" path="res://assets/wasp.scad" id="2_wasp"]

[node name="WaspEntity" type="Node3D"]
script = ExtResource("1_script")

[node name="WaspModel" parent="." instance=ExtResource("2_wasp")]
```

### frog_fly_catcher/scenes/main_scene.tscn
```text
[gd_scene load_steps=8 format=3]

[ext_resource type="Script" path="res://scripts/main_game.gd" id="1_main"]
[ext_resource type="PackedScene" path="res://assets/lily_pad_pond.scad" id="2_pond"]
[ext_resource type="PackedScene" path="res://assets/cattail_reed.scad" id="3_reed"]
[ext_resource type="PackedScene" path="res://scenes/frog_player.tscn" id="4_frog"]

[sub_resource type="ProceduralSkyMaterial" id="ProceduralSkyMaterial_sky"]
sky_top_color = Color(0.24, 0.56, 0.88, 1)
sky_horizon_color = Color(0.72, 0.88, 0.96, 1)
ground_bottom_color = Color(0.14, 0.28, 0.34, 1)
ground_horizon_color = Color(0.60, 0.80, 0.88, 1)

[sub_resource type="Sky" id="Sky_pond"]
sky_material = SubResource("ProceduralSkyMaterial_sky")

[sub_resource type="Environment" id="Environment_pond"]
background_mode = 2
sky = SubResource("Sky_pond")
ambient_light_source = 2
ambient_light_color = Color(0.75, 0.86, 0.92, 1)
ambient_light_energy = 0.55
glow_enabled = true
glow_intensity = 0.45
glow_bloom = 0.15

[node name="MainGame" type="Node3D"]
script = ExtResource("1_main")

[node name="WorldEnvironment" type="WorldEnvironment" parent="."]
environment = SubResource("Environment_pond")

[node name="DirectionalLight3D" type="DirectionalLight3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 0.707107, 0.707107, 0, -0.707107, 0.707107, 4.5, 8.5, -5.5)
light_color = Color(1, 0.96, 0.86, 1)
light_energy = 1.15
shadow_enabled = true

[node name="Camera3D" type="Camera3D" parent="."]
transform = Transform3D(-1, 0, 0, 0, 0.984808, 0.173648, 0, 0.173648, -0.984808, 1.5, 2.35, -6.4)
fov = 48.0

[node name="LilyPadPond" parent="." instance=ExtResource("2_pond")]

[node name="Reeds" type="Node3D" parent="."]

[node name="Reed1" parent="Reeds" instance=ExtResource("3_reed")]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, -4.6, -0.02, 2.8)

[node name="Reed2" parent="Reeds" instance=ExtResource("3_reed")]
transform = Transform3D(0.866025, 0, 0.5, 0, 1, 0, -0.5, 0, 0.866025, 4.4, -0.02, 3.2)

[node name="Reed3" parent="Reeds" instance=ExtResource("3_reed")]
transform = Transform3D(0.939693, 0, -0.34202, 0, 1, 0, 0.34202, 0, 0.939693, -2.5, -0.02, 5.1)

[node name="Reed4" parent="Reeds" instance=ExtResource("3_reed")]
transform = Transform3D(0.766044, 0, -0.642788, 0, 1, 0, 0.642788, 0, 0.766044, 2.8, -0.02, 4.9)

[node name="FrogPlayer" parent="." instance=ExtResource("4_frog")]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0.05, 0)

[node name="EntitiesContainer" type="Node3D" parent="."]

[node name="ParticlesContainer" type="Node3D" parent="."]

[node name="HUD" type="CanvasLayer" parent="."]

[node name="TopBar" type="HBoxContainer" parent="HUD"]
anchors_preset = 10
anchor_right = 1.0
offset_left = 24.0
offset_top = 16.0
offset_right = -24.0
offset_bottom = 60.0
theme_override_constants/separation = 28

[node name="ScoreLabel" type="Label" parent="HUD/TopBar"]
layout_mode = 2
vertical_alignment = 1
theme_override_colors/font_color = Color(1, 1, 1, 1)
theme_override_colors/font_outline_color = Color(0.06, 0.14, 0.18, 1)
theme_override_constants/outline_size = 5
theme_override_font_sizes/font_size = 24
text = "Score: 0"

[node name="ComboLabel" type="Label" parent="HUD/TopBar"]
layout_mode = 2
vertical_alignment = 1
theme_override_colors/font_color = Color(1, 0.88, 0.35, 1)
theme_override_colors/font_outline_color = Color(0.06, 0.14, 0.18, 1)
theme_override_constants/outline_size = 5
theme_override_font_sizes/font_size = 24
text = "Combo: x1"

[node name="HeartsBox" type="HBoxContainer" parent="HUD/TopBar"]
layout_mode = 2
size_flags_vertical = 4
theme_override_constants/separation = 8
alignment = 1

[node name="HeartsText" type="Label" parent="HUD/TopBar/HeartsBox"]
layout_mode = 2
vertical_alignment = 1
theme_override_colors/font_color = Color(1, 0.48, 0.52, 1)
theme_override_colors/font_outline_color = Color(0.06, 0.14, 0.18, 1)
theme_override_constants/outline_size = 5
theme_override_font_sizes/font_size = 24
text = "Lives:"

[node name="HeartsIcons" type="HBoxContainer" parent="HUD/TopBar/HeartsBox"]
layout_mode = 2
size_flags_vertical = 4
theme_override_constants/separation = 5
alignment = 1

[node name="MissesLabel" type="Label" parent="HUD/TopBar"]
layout_mode = 2
vertical_alignment = 1
theme_override_colors/font_color = Color(0.85, 0.94, 1, 1)
theme_override_colors/font_outline_color = Color(0.06, 0.14, 0.18, 1)
theme_override_constants/outline_size = 5
theme_override_font_sizes/font_size = 22
text = "Misses: 0/3"

[node name="BestLabel" type="Label" parent="HUD/TopBar"]
layout_mode = 2
size_flags_horizontal = 3
horizontal_alignment = 2
vertical_alignment = 1
theme_override_colors/font_color = Color(0.75, 1, 0.78, 1)
theme_override_colors/font_outline_color = Color(0.06, 0.14, 0.18, 1)
theme_override_constants/outline_size = 5
theme_override_font_sizes/font_size = 22
text = "Best: 0"

[node name="PopupLabel" type="Label" parent="HUD"]
visible = false
anchors_preset = 5
anchor_left = 0.5
anchor_right = 0.5
offset_left = -180.0
offset_top = 84.0
offset_right = 180.0
offset_bottom = 132.0
horizontal_alignment = 1
theme_override_colors/font_outline_color = Color(0.05, 0.1, 0.12, 1)
theme_override_constants/outline_size = 6
theme_override_font_sizes/font_size = 32
text = "+100"

[node name="MenuPanel" type="PanelContainer" parent="HUD"]
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.27
anchor_right = 0.5
anchor_bottom = 0.27
offset_left = -255.0
offset_top = -125.0
offset_right = 255.0
offset_bottom = 135.0

[node name="VBox" type="VBoxContainer" parent="HUD/MenuPanel"]
layout_mode = 2
theme_override_constants/separation = 14
alignment = 1

[node name="TitleLabel" type="Label" parent="HUD/MenuPanel/VBox"]
layout_mode = 2
horizontal_alignment = 1
theme_override_colors/font_color = Color(0.58, 1, 0.5, 1)
theme_override_font_sizes/font_size = 34
text = "FROG & FLIES"

[node name="DescLabel" type="Label" parent="HUD/MenuPanel/VBox"]
layout_mode = 2
horizontal_alignment = 1
theme_override_font_sizes/font_size = 17
text = "Jump at the right moment to catch flies!
Watch out for wasps — they sting!

Jump & Tongue: Space / W / Up / LMB"

[node name="StartButton" type="Button" parent="HUD/MenuPanel/VBox"]
layout_mode = 2
custom_minimum_size = Vector2(200, 48)
size_flags_horizontal = 4
theme_override_font_sizes/font_size = 22
text = "Play"

[node name="GameOverPanel" type="PanelContainer" parent="HUD"]
visible = false
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -210.0
offset_top = -130.0
offset_right = 210.0
offset_bottom = 130.0

[node name="VBox" type="VBoxContainer" parent="HUD/GameOverPanel"]
layout_mode = 2
theme_override_constants/separation = 16
alignment = 1

[node name="GameOverTitle" type="Label" parent="HUD/GameOverPanel/VBox"]
layout_mode = 2
horizontal_alignment = 1
theme_override_colors/font_color = Color(1, 0.45, 0.38, 1)
theme_override_font_sizes/font_size = 32
text = "Game Over!"

[node name="FinalScoreLabel" type="Label" parent="HUD/GameOverPanel/VBox"]
layout_mode = 2
horizontal_alignment = 1
theme_override_font_sizes/font_size = 20
text = "Final Score: 0"

[node name="RetryButton" type="Button" parent="HUD/GameOverPanel/VBox"]
layout_mode = 2
custom_minimum_size = Vector2(200, 48)
size_flags_horizontal = 4
theme_override_font_sizes/font_size = 22
text = "Play Again"
```