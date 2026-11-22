### candy_crush_3d/addons/scad_importer/plugin.cfg
```ini
[plugin]

name="OpenSCAD GLTF Importer"
description="Imports .scad files directly as 3D scenes using scad-gltf"
author="scad-gltf"
version="0.2"
script="scad_plugin.gd"

```

### candy_crush_3d/addons/scad_importer/scad_importer.gd
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

### candy_crush_3d/addons/scad_importer/scad_plugin.gd
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

### candy_crush_3d/assets/board/board_frame.scad
```openscad
$fn = 28;
$asa = 45;

difference() {
    color([0.18, 0.12, 0.25], roughness=0.5, metalness=0.1)
    hull() {
        for (x = [-4.35, 4.35]) {
            for (z = [-4.35, 4.35]) {
                translate([x, 0, z])
                    rotate([90, 0, 0])
                    cylinder(h=0.12, r=0.35, center=true);
            }
        }
    }
    cube([8.08, 0.5, 8.08], center=true);
}

// Gold candy trim
color([0.96, 0.76, 0.2], roughness=0.25, metalness=0.7, specularIntensity=1.0) {
    for (x = [-4.12, 4.12]) {
        translate([x, 0.05, 0]) cube([0.08, 0.06, 8.3], center=true);
    }
    for (z = [-4.12, 4.12]) {
        translate([0, 0.05, z]) cube([8.3, 0.06, 0.08], center=true);
    }
}

```

### candy_crush_3d/assets/board/board_tile.scad
```openscad
$fn = 24;
$asa = 45;

color([0.26, 0.19, 0.34], roughness=0.6, metalness=0.0) {
    hull() {
        for(x=[-0.43, 0.43]) {
            for(z=[-0.43, 0.43]) {
                translate([x, 0, z])
                    rotate([90, 0, 0])
                    cylinder(h=0.06, r=0.04, center=true);
            }
        }
    }
}
color([0.36, 0.27, 0.46], roughness=0.5, metalness=0.0) {
    hull() {
        for(x=[-0.39, 0.39]) {
            for(z=[-0.39, 0.39]) {
                translate([x, 0.02, z])
                    rotate([90, 0, 0])
                    cylinder(h=0.06, r=0.03, center=true);
            }
        }
    }
}

```

### candy_crush_3d/assets/board/selection_cursor.scad
```openscad
$fn = 28;
$asa = 45;

color([1.0, 0.95, 0.3], alpha=0.9, roughness=0.15, metalness=0.2, emissive=[0.8, 0.7, 0.1], emissiveIntensity=2.0) {
    rotate([90, 0, 0])
        rotate_extrude()
        translate([0.48, 0, 0])
        circle(r=0.035);
}

```

### candy_crush_3d/assets/candies/candy_blue.scad
```openscad
$fn = 28;
$asa = 45;

color([0.1, 0.5, 0.98], alpha=1.0, roughness=0.12, metalness=0.05, specularIntensity=1.0) {
    sphere(r=0.33);
}
color([0.25, 0.9, 1.0], alpha=1.0, roughness=0.2) {
    rotate([25, 0, 30])
        rotate_extrude()
        translate([0.33, 0, 0])
        circle(r=0.05);
}

```

### candy_crush_3d/assets/candies/candy_color_bomb.scad
```openscad
$fn = 28;
$asa = 45;

anim_data = [
  ["IdleSpin", [
    ["Root", [
      [0.0, [0, 0, 0]],
      [1.0, [0, 90, 0]],
      [2.0, [0, 180, 0]],
      [3.0, [0, 270, 0]],
      [4.0, [0, 360, 0]]
    ]]
  ]]
];

armature(animations=anim_data) {
  bone(name="Root", t=[0, 0, 0], r=[0, 0, 0]) {
    color([0.22, 0.13, 0.08], roughness=0.25, specularIntensity=0.9)
      sphere(r=0.34);

    color([1.0, 0.15, 0.2], roughness=0.2, emissive=[0.5, 0.05, 0.05], emissiveIntensity=1.8)
      translate([0.23, 0.1, 0.22]) sphere(r=0.075);
    color([0.15, 0.9, 0.3], roughness=0.2, emissive=[0.05, 0.5, 0.05], emissiveIntensity=1.8)
      translate([-0.23, -0.12, 0.18]) sphere(r=0.075);
    color([0.15, 0.55, 1.0], roughness=0.2, emissive=[0.05, 0.2, 0.5], emissiveIntensity=1.8)
      translate([0.1, 0.14, -0.26]) sphere(r=0.075);
    color([1.0, 0.85, 0.1], roughness=0.2, emissive=[0.5, 0.4, 0.05], emissiveIntensity=1.8)
      translate([-0.18, -0.09, -0.22]) sphere(r=0.075);
    color([0.9, 0.25, 1.0], roughness=0.2, emissive=[0.4, 0.05, 0.5], emissiveIntensity=1.8)
      translate([0, -0.14, 0.28]) sphere(r=0.075);
    color([1.0, 0.5, 0.05], roughness=0.2, emissive=[0.5, 0.25, 0.05], emissiveIntensity=1.8)
      translate([0.24, -0.14, -0.15]) sphere(r=0.075);
  }
}

```

### candy_crush_3d/assets/candies/candy_green.scad
```openscad
$fn = 24;
$asa = 45;

color([0.08, 0.82, 0.28], alpha=1.0, roughness=0.15, metalness=0.05, specularIntensity=1.0) {
    hull() {
        for (x = [-0.18, 0.18]) {
            for (z = [-0.18, 0.18]) {
                for (y = [-0.1, 0.1]) {
                    translate([x, y, z]) sphere(r=0.13);
                }
            }
        }
    }
}
color([0.92, 1.0, 0.92], alpha=1.0, roughness=0.2) {
    translate([0, 0.15, 0])
        rotate([90, 0, 0])
        cylinder(r=0.12, h=0.12, center=true);
}

```

### candy_crush_3d/assets/candies/candy_orange.scad
```openscad
$fn = 28;
$asa = 45;

color([1.0, 0.45, 0.05], alpha=1.0, roughness=0.15, metalness=0.05, specularIntensity=1.0) {
    scale([1.25, 0.7, 0.8])
        sphere(r=0.38);
}
color([1.0, 0.96, 0.9], alpha=1.0, roughness=0.2, metalness=0.0) {
    cylinder(r=0.27, h=0.12, center=true);
}

```

### candy_crush_3d/assets/candies/candy_purple.scad
```openscad
$fn = 6;
$asa = 0;

color([0.72, 0.14, 0.92], alpha=1.0, roughness=0.1, metalness=0.1, specularIntensity=1.0) {
    rotate([90, 0, 0])
    hull() {
        cylinder(r=0.36, h=0.14, center=true);
        translate([0, 0, 0.18]) cylinder(r=0.22, h=0.04, center=true);
        translate([0, 0, -0.18]) cylinder(r=0.22, h=0.04, center=true);
    }
}
color([0.95, 0.7, 1.0], alpha=1.0, roughness=0.2) {
    translate([0, 0.16, 0])
        rotate([90, 0, 0])
        cylinder(r=0.11, h=0.02, center=true);
}

```

### candy_crush_3d/assets/candies/candy_red.scad
```openscad
$fn = 28;
$asa = 45;

color([0.95, 0.12, 0.2], alpha=1.0, roughness=0.15, metalness=0.05, specularIntensity=1.0) {
    hull() {
        translate([-0.18, 0, 0]) sphere(r=0.32);
        translate([0.18, 0, 0.08]) sphere(r=0.28);
        translate([0, 0, -0.12]) sphere(r=0.24);
    }
}
color([1.0, 0.85, 0.9], alpha=0.9, roughness=0.1) {
    translate([-0.12, 0.22, -0.05]) sphere(r=0.06);
}

```

### candy_crush_3d/assets/candies/candy_yellow.scad
```openscad
$fn = 28;
$asa = 45;

color([1.0, 0.86, 0.04], alpha=1.0, roughness=0.15, metalness=0.05, specularIntensity=1.0) {
    hull() {
        translate([0, 0, 0.22]) sphere(r=0.14);
        translate([0, 0, -0.05])
            rotate([90, 0, 0])
            cylinder(r1=0.32, r2=0.24, h=0.22, center=true);
        translate([0, 0, -0.22]) sphere(r=0.22);
    }
}

```

### candy_crush_3d/project.godot
```ini
config_version=5

[application]
config/name="Candy Crush 3D"
config/description="A procedural 3D match-3 candy crush game built with Godot 4 and OpenSCAD"
run/main_scene="res://scenes/main.tscn"
config/features=PackedStringArray("4.3", "Forward Plus")

[display]
window/size/viewport_width=1280
window/size/viewport_height=720
window/stretch/mode="canvas_items"
window/stretch/aspect="keep"
window/handheld/orientation="landscape"

[editor_plugins]
enabled=PackedStringArray("res://addons/scad_importer/plugin.cfg")

[rendering]
environment/defaults/default_clear_color=Color(0.12, 0.07, 0.18, 1)
anti_aliasing/quality/msaa_3d=2

```

### candy_crush_3d/README.md
```markdown
# Candy Crush 3D (OpenSCAD & Godot 4)

A vibrant, fully procedural 3D match-3 candy crush game built with Godot 4 and procedural OpenSCAD 3D models imported via the `scad-gltf` importer.

## Features

- **Procedural 3D Candies (.scad)**:
  - 🍒 **Red Jelly Bean** (`candy_red.scad`): Glossy kidney bean shape with specular highlights.
  - 🍊 **Orange Lozenge** (`candy_orange.scad`): Smooth orange pebble with white sugar stripe.
  - 🍋 **Yellow Lemon Drop** (`candy_yellow.scad`): Conical drop candy.
  - 🍏 **Green Mint Pillow** (`candy_green.scad`): Rounded square mint pillow with central emblem.
  - 🫐 **Blue Lollipop Orb** (`candy_blue.scad`): Spherical candy with cyan Saturn ring.
  - 🍇 **Purple Gem** (`candy_purple.scad`): Faceted hexagonal hard candy jewel.
  - 🍫 **Color Bomb** (`candy_color_bomb.scad`): Special chocolate sphere studded with spinning rainbow sprinkles!
  - 🧇 **Board Tile & Frame** (`board_tile.scad`, `board_frame.scad`): Frosted biscuit cell tiles and gold-trimmed candy frame.
- **Match-3 Mechanics**:
  - Click & swap or drag & swap adjacent candies.
  - Automatic move validation with bouncy recoil animation on invalid swap.
  - Match-4 and Match-5 detection.
  - Color Bomb mechanics: swap a Color Bomb with any candy to clear all candies of that color!
  - Gravity cascade with bouncing drops.
  - Combo streaks with dynamic feedback ("SWEET!", "TASTY!", "DELICIOUS!", "DIVINE!", "SUGAR CRUSH!").
  - Deadlock detection & automatic board reshuffle.
- **Procedural Audio**:
  - In-engine synthesized sound effects for selections, swaps, combos, and color explosions.
- **Game Modes**:
  - **Challenge Mode**: 25 moves to reach 5,000 points.
  - **Zen Mode**: Relax with unlimited moves!

## How to Run

1. Open this folder in Godot 4.3+.
2. The `scad_importer` addon will automatically convert all `.scad` files to glTF 3D scenes.
3. Press **F5** (Play) to start crushing candies!

```

### candy_crush_3d/scenes/main.tscn
```ini
[gd_scene load_steps=7 format=3 uid="uid://mainscene101"]

[ext_resource type="Script" path="res://scripts/main.gd" id="1_main"]
[ext_resource type="Script" path="res://scripts/board.gd" id="2_board"]
[ext_resource type="Script" path="res://scripts/audio_manager.gd" id="3_audio"]
[ext_resource type="Script" path="res://scripts/hud.gd" id="4_hud"]
[ext_resource type="PackedScene" path="res://assets/board/board_frame.scad" id="5_frame"]

[sub_resource type="Environment" id="Environment_1"]
background_mode = 1
background_color = Color(0.12, 0.08, 0.18, 1)
ambient_light_source = 2
ambient_light_color = Color(0.62, 0.54, 0.72, 1)
ambient_light_energy = 0.6
tonemap_mode = 2
tonemap_exposure = 1.0
glow_enabled = true
glow_intensity = 0.25
glow_bloom = 0.02

[node name="Main" type="Node3D"]
script = ExtResource("1_main")

[node name="WorldEnvironment" type="WorldEnvironment" parent="."]
environment = SubResource("Environment_1")

[node name="DirectionalLight3D" type="DirectionalLight3D" parent="."]
transform = Transform3D(0.866025, -0.25, 0.433013, 0, 0.866025, 0.5, -0.5, -0.433013, 0.75, 3, 5, 5)
light_color = Color(1, 0.97, 0.92, 1)
light_energy = 0.8
shadow_enabled = true
shadow_opacity = 0.45
shadow_blur = 1.8

[node name="FillLight" type="DirectionalLight3D" parent="."]
transform = Transform3D(-0.866025, 0.25, -0.433013, 0, 0.866025, 0.5, 0.5, 0.433013, -0.75, -3, 3, 4)
light_color = Color(0.7, 0.78, 0.95, 1)
light_energy = 0.45

[node name="Camera3D" type="Camera3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, -0.3, 10.6)
current = true
fov = 52.0

[node name="BoardFrame" parent="." instance=ExtResource("5_frame")]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, -0.3, -0.06)

[node name="Board" type="Node3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, -0.3, 0)
script = ExtResource("2_board")

[node name="AudioManager" type="Node" parent="."]
script = ExtResource("3_audio")

[node name="HUD" type="CanvasLayer" parent="."]
script = ExtResource("4_hud")

[node name="Margin" type="MarginContainer" parent="HUD"]
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
theme_override_constants/margin_left = 24
theme_override_constants/margin_top = 16
theme_override_constants/margin_right = 24
theme_override_constants/margin_bottom = 16

[node name="VBoxTop" type="VBoxContainer" parent="HUD/Margin"]
layout_mode = 2
size_flags_vertical = 0
theme_override_constants/separation = 8

[node name="HBoxHeader" type="HBoxContainer" parent="HUD/Margin/VBoxTop"]
layout_mode = 2
alignment = 1
theme_override_constants/separation = 30

[node name="PanelScore" type="PanelContainer" parent="HUD/Margin/VBoxTop/HBoxHeader"]
custom_minimum_size = Vector2(150, 56)
layout_mode = 2

[node name="VBox" type="VBoxContainer" parent="HUD/Margin/VBoxTop/HBoxHeader/PanelScore"]
layout_mode = 2
alignment = 1

[node name="Title" type="Label" parent="HUD/Margin/VBoxTop/HBoxHeader/PanelScore/VBox"]
layout_mode = 2
text = "SCORE"
horizontal_alignment = 1

[node name="ScoreVal" type="Label" parent="HUD/Margin/VBoxTop/HBoxHeader/PanelScore/VBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 22
text = "0"
horizontal_alignment = 1

[node name="PanelMoves" type="PanelContainer" parent="HUD/Margin/VBoxTop/HBoxHeader"]
custom_minimum_size = Vector2(150, 56)
layout_mode = 2

[node name="VBox" type="VBoxContainer" parent="HUD/Margin/VBoxTop/HBoxHeader/PanelMoves"]
layout_mode = 2
alignment = 1

[node name="Title" type="Label" parent="HUD/Margin/VBoxTop/HBoxHeader/PanelMoves/VBox"]
layout_mode = 2
text = "MOVES"
horizontal_alignment = 1

[node name="MovesVal" type="Label" parent="HUD/Margin/VBoxTop/HBoxHeader/PanelMoves/VBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 22
text = "25"
horizontal_alignment = 1

[node name="PanelTarget" type="PanelContainer" parent="HUD/Margin/VBoxTop/HBoxHeader"]
custom_minimum_size = Vector2(150, 56)
layout_mode = 2

[node name="VBox" type="VBoxContainer" parent="HUD/Margin/VBoxTop/HBoxHeader/PanelTarget"]
layout_mode = 2
alignment = 1

[node name="Title" type="Label" parent="HUD/Margin/VBoxTop/HBoxHeader/PanelTarget/VBox"]
layout_mode = 2
text = "TARGET"
horizontal_alignment = 1

[node name="TargetVal" type="Label" parent="HUD/Margin/VBoxTop/HBoxHeader/PanelTarget/VBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 22
text = "5,000"
horizontal_alignment = 1

[node name="ProgressBar" type="ProgressBar" parent="HUD/Margin/VBoxTop"]
custom_minimum_size = Vector2(380, 14)
layout_mode = 2
size_flags_horizontal = 4
max_value = 5000.0
show_percentage = false

[node name="HBoxBottom" type="HBoxContainer" parent="HUD/Margin"]
layout_mode = 2
size_flags_vertical = 8
alignment = 1
theme_override_constants/separation = 20

[node name="BtnRestart" type="Button" parent="HUD/Margin/HBoxBottom"]
custom_minimum_size = Vector2(130, 40)
layout_mode = 2
text = "Restart"

[node name="BtnZen" type="Button" parent="HUD/Margin/HBoxBottom"]
custom_minimum_size = Vector2(130, 40)
layout_mode = 2
text = "Zen Mode"

[node name="CenterBanner" type="CenterContainer" parent="HUD"]
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
grow_horizontal = 2
grow_vertical = 2
mouse_filter = 2

[node name="ComboLabel" type="Label" parent="HUD/CenterBanner"]
layout_mode = 2
theme_override_colors/font_color = Color(1, 0.9, 0.2, 1)
theme_override_colors/font_shadow_color = Color(0, 0, 0, 0.8)
theme_override_constants/shadow_offset_x = 3
theme_override_constants/shadow_offset_y = 3
theme_override_font_sizes/font_size = 54
text = "SWEET!"
horizontal_alignment = 1

[node name="GameOverPanel" type="PanelContainer" parent="HUD"]
visible = false
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -180.0
offset_top = -120.0
offset_right = 180.0
offset_bottom = 120.0
grow_horizontal = 2
grow_vertical = 2

[node name="VBox" type="VBoxContainer" parent="HUD/GameOverPanel"]
layout_mode = 2
alignment = 1
theme_override_constants/separation = 16

[node name="Title" type="Label" parent="HUD/GameOverPanel/VBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 28
text = "LEVEL COMPLETE!"
horizontal_alignment = 1

[node name="FinalScore" type="Label" parent="HUD/GameOverPanel/VBox"]
layout_mode = 2
theme_override_font_sizes/font_size = 20
text = "Final Score: 6400"
horizontal_alignment = 1

[node name="BtnPlayAgain" type="Button" parent="HUD/GameOverPanel/VBox"]
custom_minimum_size = Vector2(140, 44)
layout_mode = 2
size_flags_horizontal = 4
text = "Play Again"

```

### candy_crush_3d/scripts/audio_manager.gd
```gdscript
extends Node
class_name AudioManager

var sample_rate: int = 22050
var streams: Dictionary = {}

func _ready():
	_generate_audio_streams()

func _generate_audio_streams():
	streams["select"] = _synth_tone(800.0, 1200.0, 0.06, 0.3, "sine")
	streams["swap"] = _synth_tone(400.0, 600.0, 0.12, 0.4, "sine")
	streams["invalid"] = _synth_tone(220.0, 150.0, 0.18, 0.4, "saw")
	streams["bomb"] = _synth_noise_pop(0.35, 0.5)

func _synth_tone(freq_start: float, freq_end: float, duration: float, volume: float, waveform: String) -> AudioStreamWAV:
	var total_samples = int(sample_rate * duration)
	var bytes = PackedByteArray()
	bytes.resize(total_samples)
	var phase = 0.0

	for i in range(total_samples):
		var frac = float(i) / float(total_samples)
		var freq = lerp(freq_start, freq_end, frac)
		phase += 2.0 * PI * freq / float(sample_rate)
		var env = sin(frac * PI)
		var val = 0.0
		if waveform == "sine":
			val = sin(phase)
		elif waveform == "saw":
			val = fmod(phase / PI, 2.0) - 1.0
		val *= env * volume
		bytes[i] = int(clamp((val * 0.5 + 0.5) * 255.0, 0, 255))

	var wav = AudioStreamWAV.new()
	wav.format = AudioStreamWAV.FORMAT_8_BITS
	wav.mix_rate = sample_rate
	wav.data = bytes
	return wav

func _synth_noise_pop(duration: float, volume: float) -> AudioStreamWAV:
	var total_samples = int(sample_rate * duration)
	var bytes = PackedByteArray()
	bytes.resize(total_samples)
	for i in range(total_samples):
		var frac = float(i) / float(total_samples)
		var env = exp(-frac * 7.0)
		var val = (randf() * 2.0 - 1.0) * env * volume
		bytes[i] = int(clamp((val * 0.5 + 0.5) * 255.0, 0, 255))

	var wav = AudioStreamWAV.new()
	wav.format = AudioStreamWAV.FORMAT_8_BITS
	wav.mix_rate = sample_rate
	wav.data = bytes
	return wav

func play_sound(sname: String):
	if streams.has(sname):
		_play_stream(streams[sname], 1.0)

func play_match(combo: int):
	var base_freq = 440.0 * pow(1.12, min(combo, 10))
	var wav = _synth_tone(base_freq, base_freq * 1.5, 0.16, 0.4, "sine")
	_play_stream(wav, 1.0)

func _play_stream(stream: AudioStream, pitch_scale: float = 1.0):
	var player = AudioStreamPlayer.new()
	add_child(player)
	player.stream = stream
	player.pitch_scale = pitch_scale
	player.play()
	player.finished.connect(player.queue_free)

```

### candy_crush_3d/scripts/board.gd
```gdscript
extends Node3D
class_name Match3Board

signal score_updated(score, added)
signal moves_updated(moves_left)
signal combo_triggered(combo_count, message)
signal board_ready()
signal game_over(won, final_score)

const GRID_COLS = 8
const GRID_ROWS = 8
const CELL_SIZE = 1.0

var candy_scenes: Array[PackedScene] = []
var color_bomb_scene: PackedScene = null
var tile_scene: PackedScene = null
var cursor_scene: PackedScene = null

var grid: Array = []
var selected_candy: Candy = null
var is_animating: bool = false
var audio_mgr: AudioManager = null
var cursor_node: Node3D = null

var score: int = 0
var moves_left: int = 25
var zen_mode: bool = false
var combo_streak: int = 0
var target_score: int = 5000

var drag_start_cell: Vector2i = Vector2i(-1, -1)
var is_dragging: bool = false

func _ready():
	_load_assets()
	_create_tiles()
	_init_grid()
	_spawn_initial_board()

func set_audio_manager(mgr: AudioManager):
	audio_mgr = mgr

func _load_assets():
	candy_scenes.append(load("res://assets/candies/candy_red.scad"))
	candy_scenes.append(load("res://assets/candies/candy_orange.scad"))
	candy_scenes.append(load("res://assets/candies/candy_yellow.scad"))
	candy_scenes.append(load("res://assets/candies/candy_green.scad"))
	candy_scenes.append(load("res://assets/candies/candy_blue.scad"))
	candy_scenes.append(load("res://assets/candies/candy_purple.scad"))
	color_bomb_scene = load("res://assets/candies/candy_color_bomb.scad")
	tile_scene = load("res://assets/board/board_tile.scad")
	cursor_scene = load("res://assets/board/selection_cursor.scad")

func _create_tiles():
	var tiles_parent = Node3D.new()
	tiles_parent.name = "Tiles"
	add_child(tiles_parent)

	for col in range(GRID_COLS):
		for row in range(GRID_ROWS):
			if tile_scene:
				var t = tile_scene.instantiate()
				t.position = grid_to_world(col, row) + Vector3(0, 0, -0.06)
				tiles_parent.add_child(t)

	if cursor_scene:
		cursor_node = cursor_scene.instantiate()
		cursor_node.visible = false
		add_child(cursor_node)

func _init_grid():
	grid.clear()
	for col in range(GRID_COLS):
		var col_arr = []
		for row in range(GRID_ROWS):
			col_arr.append(null)
		grid.append(col_arr)

func grid_to_world(col: int, row: int) -> Vector3:
	var x = (col - (GRID_COLS - 1) / 2.0) * CELL_SIZE
	var y = ((GRID_ROWS - 1) / 2.0 - row) * CELL_SIZE
	return Vector3(x, y, 0)

func world_to_grid(world_pos: Vector3) -> Vector2i:
	var local = to_local(world_pos)
	var col = round(local.x / CELL_SIZE + (GRID_COLS - 1) / 2.0)
	var row = round((GRID_ROWS - 1) / 2.0 - local.y / CELL_SIZE)
	return Vector2i(int(col), int(row))

func is_valid_cell(col: int, row: int) -> bool:
	return col >= 0 and col < GRID_COLS and row >= 0 and row < GRID_ROWS

func _spawn_initial_board():
	for col in range(GRID_COLS):
		for row in range(GRID_ROWS):
			var valid_types = []
			for t in range(candy_scenes.size()):
				if col >= 2 and grid[col-1][row] and grid[col-2][row] and grid[col-1][row].candy_type == t and grid[col-2][row].candy_type == t:
					continue
				if row >= 2 and grid[col][row-1] and grid[col][row-2] and grid[col][row-1].candy_type == t and grid[col][row-2].candy_type == t:
					continue
				valid_types.append(t)

			var chosen_type = valid_types[randi() % valid_types.size()] if valid_types.size() > 0 else 0
			_spawn_candy(col, row, chosen_type, true)

func _spawn_candy(col: int, row: int, type: int, instant: bool = false) -> Candy:
	var candy = Candy.new()
	var res = color_bomb_scene if type == 6 else candy_scenes[type]
	candy.setup(type, col, row, res)
	add_child(candy)
	grid[col][row] = candy

	var target_pos = grid_to_world(col, row)
	if instant:
		candy.position = target_pos
	else:
		candy.position = target_pos + Vector3(0, 5.0 + (GRID_ROWS - row) * 0.4, 0)
		candy.drop_to_pos(target_pos, 0.28 + (GRID_ROWS - row) * 0.035)
	return candy

func handle_input_click(world_pos: Vector3, is_press: bool):
	if is_animating:
		return

	var cell = world_to_grid(world_pos)
	if not is_valid_cell(cell.x, cell.y):
		if not is_press and is_dragging:
			is_dragging = false
		return

	if is_press:
		drag_start_cell = cell
		is_dragging = true
		_on_cell_clicked(cell.x, cell.y)
	else:
		if is_dragging and drag_start_cell != cell:
			var diff = cell - drag_start_cell
			if abs(diff.x) + abs(diff.y) == 1:
				if is_valid_cell(drag_start_cell.x, drag_start_cell.y):
					var c1 = grid[drag_start_cell.x][drag_start_cell.y]
					var c2 = grid[cell.x][cell.y]
					if c1 and c2:
						try_swap(c1, c2)
		is_dragging = false

func _on_cell_clicked(col: int, row: int):
	var clicked = grid[col][row]
	if not clicked:
		return

	if selected_candy == null:
		_select_candy(clicked)
	elif selected_candy == clicked:
		_deselect_candy()
	else:
		var dist = abs(selected_candy.grid_col - clicked.grid_col) + abs(selected_candy.grid_row - clicked.grid_row)
		if dist == 1:
			var prev = selected_candy
			_deselect_candy()
			try_swap(prev, clicked)
		else:
			_deselect_candy()
			_select_candy(clicked)

func _select_candy(candy: Candy):
	selected_candy = candy
	candy.set_selected(true)
	if cursor_node:
		cursor_node.visible = true
		cursor_node.position = candy.position + Vector3(0, 0, 0.1)
	if audio_mgr:
		audio_mgr.play_sound("select")

func _deselect_candy():
	if selected_candy:
		selected_candy.set_selected(false)
		selected_candy = null
	if cursor_node:
		cursor_node.visible = false

func try_swap(c1: Candy, c2: Candy):
	is_animating = true
	_deselect_candy()

	if audio_mgr:
		audio_mgr.play_sound("swap")

	var col1 = c1.grid_col
	var row1 = c1.grid_row
	var col2 = c2.grid_col
	var row2 = c2.grid_row

	grid[col1][row1] = c2
	grid[col2][row2] = c1
	c1.grid_col = col2
	c1.grid_row = row2
	c2.grid_col = col1
	c2.grid_row = row1

	var pos1 = grid_to_world(col1, row1)
	var pos2 = grid_to_world(col2, row2)

	var tw1 = c1.move_to_pos(pos2, 0.2)
	var tw2 = c2.move_to_pos(pos1, 0.2)

	tw1.finished.connect(func():
		if c1.candy_type == 6 or c2.candy_type == 6:
			_handle_color_bomb_swap(c1, c2)
			return

		var matches = _find_all_matches()
		if matches.size() > 0:
			if not zen_mode:
				moves_left -= 1
				moves_updated.emit(moves_left)
			combo_streak = 0
			_process_matches(matches)
		else:
			if audio_mgr:
				audio_mgr.play_sound("invalid")
			grid[col1][row1] = c1
			grid[col2][row2] = c2
			c1.grid_col = col1
			c1.grid_row = row1
			c2.grid_col = col2
			c2.grid_row = row2
			var rtw1 = c1.move_to_pos(pos1, 0.18, Tween.TRANS_BACK, Tween.EASE_OUT)
			c2.move_to_pos(pos2, 0.18, Tween.TRANS_BACK, Tween.EASE_OUT)
			rtw1.finished.connect(func():
				is_animating = false
			)
	)

func _handle_color_bomb_swap(c1: Candy, c2: Candy):
	if not zen_mode:
		moves_left -= 1
		moves_updated.emit(moves_left)
	if audio_mgr:
		audio_mgr.play_sound("bomb")

	var bomb = c1 if c1.candy_type == 6 else c2
	var other = c2 if c1.candy_type == 6 else c1

	var targets = []
	if other.candy_type == 6:
		for c in range(GRID_COLS):
			for r in range(GRID_ROWS):
				if grid[c][r]:
					targets.append(grid[c][r])
	else:
		targets.append(bomb)
		var target_type = other.candy_type
		for c in range(GRID_COLS):
			for r in range(GRID_ROWS):
				if grid[c][r] and grid[c][r].candy_type == target_type and grid[c][r] != bomb:
					targets.append(grid[c][r])

	combo_streak = 1
	combo_triggered.emit(combo_streak, "COLOR CRUSH!")
	_destroy_candies(targets)

func _find_all_matches() -> Array:
	var matched_set = {}

	# Horizontal sliding window
	for r in range(GRID_ROWS):
		for c in range(GRID_COLS - 2):
			var c0 = grid[c][r]
			var c1 = grid[c+1][r]
			var c2 = grid[c+2][r]
			if c0 and c1 and c2:
				var t = c0.candy_type
				if t < 6 and c1.candy_type == t and c2.candy_type == t:
					matched_set[Vector2i(c, r)] = true
					matched_set[Vector2i(c+1, r)] = true
					matched_set[Vector2i(c+2, r)] = true

	# Vertical sliding window
	for c in range(GRID_COLS):
		for r in range(GRID_ROWS - 2):
			var c0 = grid[c][r]
			var c1 = grid[c][r+1]
			var c2 = grid[c][r+2]
			if c0 and c1 and c2:
				var t = c0.candy_type
				if t < 6 and c1.candy_type == t and c2.candy_type == t:
					matched_set[Vector2i(c, r)] = true
					matched_set[Vector2i(c, r+1)] = true
					matched_set[Vector2i(c, r+2)] = true

	var result = []
	for coord in matched_set.keys():
		var candy = grid[coord.x][coord.y]
		if candy and is_instance_valid(candy):
			result.append(candy)
	return result

func _process_matches(matches: Array):
	combo_streak += 1

	var base_points = matches.size() * 60
	var combo_bonus = (combo_streak - 1) * 100
	var added_score = base_points + combo_bonus
	score += added_score
	score_updated.emit(score, added_score)

	if audio_mgr:
		audio_mgr.play_match(combo_streak)

	var combo_msg = ""
	if combo_streak == 1 and matches.size() >= 5:
		combo_msg = "SUGAR CRUSH!"
	elif combo_streak == 2:
		combo_msg = "SWEET!"
	elif combo_streak == 3:
		combo_msg = "TASTY!"
	elif combo_streak == 4:
		combo_msg = "DELICIOUS!"
	elif combo_streak >= 5:
		combo_msg = "DIVINE!"

	if combo_msg != "":
		combo_triggered.emit(combo_streak, combo_msg)

	_destroy_candies(matches)

func _destroy_candies(candies: Array):
	# Deduplicate and validate candies
	var unique_candies = []
	for c in candies:
		if c and is_instance_valid(c) and not unique_candies.has(c):
			unique_candies.append(c)

	if unique_candies.is_empty():
		_drop_and_refill()
		return

	# Dictionary is captured by reference in closures!
	var tracker = {"remaining": unique_candies.size()}

	for candy in unique_candies:
		var c = candy.grid_col
		var r = candy.grid_row
		if grid[c][r] == candy:
			grid[c][r] = null

		_spawn_particle_burst(candy.position, candy.candy_type)
		candy.play_match_effect(func():
			tracker["remaining"] -= 1
			if tracker["remaining"] <= 0:
				_drop_and_refill()
		)

	# Fallback safety timer: ensures game never hangs if an effect is skipped
	get_tree().create_timer(0.45).timeout.connect(func():
		if tracker["remaining"] > 0:
			tracker["remaining"] = 0
			_drop_and_refill()
	)

func _spawn_particle_burst(pos: Vector3, type: int):
	var p = CPUParticles3D.new()
	p.position = pos
	p.emitting = true
	p.one_shot = true
	p.explosiveness = 0.95
	p.amount = 16
	p.lifetime = 0.6
	p.direction = Vector3(0, 1, 0)
	p.spread = 180.0
	p.initial_velocity_min = 2.0
	p.initial_velocity_max = 4.5
	p.scale_amount_min = 0.04
	p.scale_amount_max = 0.09

	var colors = [
		Color(0.95, 0.2, 0.3),
		Color(1.0, 0.55, 0.1),
		Color(1.0, 0.9, 0.15),
		Color(0.2, 0.9, 0.4),
		Color(0.2, 0.6, 1.0),
		Color(0.8, 0.3, 1.0),
		Color(1.0, 0.8, 0.2)
	]
	p.color = colors[type % colors.size()]
	add_child(p)

	get_tree().create_timer(1.0).timeout.connect(p.queue_free)

func _drop_and_refill():
	var max_anim_time = 0.3

	for col in range(GRID_COLS):
		var empty_row = GRID_ROWS - 1
		for row in range(GRID_ROWS - 1, -1, -1):
			if grid[col][row] != null:
				if empty_row != row:
					var candy = grid[col][row]
					grid[col][empty_row] = candy
					grid[col][row] = null
					candy.grid_row = empty_row
					var target_pos = grid_to_world(col, empty_row)
					var fall_dist = empty_row - row
					var dur = 0.22 + fall_dist * 0.04
					max_anim_time = max(max_anim_time, dur)
					candy.drop_to_pos(target_pos, dur)
				empty_row -= 1

		var spawn_offset = 0
		for r in range(empty_row, -1, -1):
			spawn_offset += 1
			var new_type = 6 if randf() < 0.03 else randi() % candy_scenes.size()
			var new_candy = _spawn_candy(col, r, new_type, false)
			var dur = 0.28 + (GRID_ROWS - r) * 0.035
			max_anim_time = max(max_anim_time, dur)

	var wait_time = max_anim_time + 0.08
	get_tree().create_timer(wait_time).timeout.connect(func():
		var cascade_matches = _find_all_matches()
		if cascade_matches.size() > 0:
			_process_matches(cascade_matches)
		else:
			combo_streak = 0
			is_animating = false
			_check_game_status()
	)

func _check_game_status():
	if not zen_mode and moves_left <= 0:
		var won = score >= target_score
		game_over.emit(won, score)
		return

	if not _has_possible_moves():
		combo_triggered.emit(0, "SHUFFLING CANDIES!")
		_shuffle_board()

func _has_possible_moves() -> bool:
	for c in range(GRID_COLS):
		for r in range(GRID_ROWS):
			var cur = grid[c][r]
			if not cur:
				continue
			if cur.candy_type == 6:
				return true

			if c < GRID_COLS - 1:
				if _test_swap_makes_match(c, r, c + 1, r):
					return true
			if r < GRID_ROWS - 1:
				if _test_swap_makes_match(c, r, c, r + 1):
					return true
	return false

func _test_swap_makes_match(c1: int, r1: int, c2: int, r2: int) -> bool:
	var temp = grid[c1][r1]
	grid[c1][r1] = grid[c2][r2]
	grid[c2][r2] = temp

	var matches = _find_all_matches()

	temp = grid[c1][r1]
	grid[c1][r1] = grid[c2][r2]
	grid[c2][r2] = temp

	return matches.size() > 0

func _shuffle_board():
	is_animating = true
	var all_candies = []
	for c in range(GRID_COLS):
		for r in range(GRID_ROWS):
			if grid[c][r]:
				all_candies.append(grid[c][r])

	all_candies.shuffle()
	var idx = 0
	for c in range(GRID_COLS):
		for r in range(GRID_ROWS):
			if idx < all_candies.size():
				var candy = all_candies[idx]
				idx += 1
				grid[c][r] = candy
				candy.grid_col = c
				candy.grid_row = r
				candy.move_to_pos(grid_to_world(c, r), 0.4, Tween.TRANS_BACK, Tween.EASE_OUT)

	get_tree().create_timer(0.45).timeout.connect(func():
		var matches = _find_all_matches()
		if matches.size() > 0:
			_process_matches(matches)
		else:
			is_animating = false
	)

func restart_game(zen: bool = false):
	zen_mode = zen
	score = 0
	moves_left = 25
	is_animating = false
	_deselect_candy()
	score_updated.emit(score, 0)
	moves_updated.emit(moves_left)

	for c in range(GRID_COLS):
		for r in range(GRID_ROWS):
			if grid[c][r]:
				grid[c][r].queue_free()
				grid[c][r] = null

	_spawn_initial_board()

```

### candy_crush_3d/scripts/candy.gd
```gdscript
extends Node3D
class_name Candy

signal clicked(candy)

var grid_col: int = 0
var grid_row: int = 0
var candy_type: int = 0
var is_special: bool = false
var model_node: Node3D = null
var is_selected: bool = false
var tween_active: Tween = null

func setup(type: int, col: int, row: int, scene_res: PackedScene):
	candy_type = type
	grid_col = col
	grid_row = row
	is_special = (type == 6)

	if scene_res:
		model_node = scene_res.instantiate()
		add_child(model_node)
		var anim_player = _find_animation_player(model_node)
		if anim_player and anim_player.has_animation("IdleSpin"):
			anim_player.play("IdleSpin")

func _find_animation_player(node: Node) -> AnimationPlayer:
	if node is AnimationPlayer:
		return node
	for child in node.get_children():
		var found = _find_animation_player(child)
		if found:
			return found
	return null

func move_to_pos(target_pos: Vector3, duration: float = 0.22, trans_type = Tween.TRANS_SINE, ease_type = Tween.EASE_OUT) -> Tween:
	if tween_active and tween_active.is_valid():
		tween_active.kill()
	tween_active = create_tween()
	tween_active.set_trans(trans_type).set_ease(ease_type)
	tween_active.tween_property(self, "position", target_pos, duration)
	return tween_active

func drop_to_pos(target_pos: Vector3, duration: float = 0.35, delay: float = 0.0) -> Tween:
	if tween_active and tween_active.is_valid():
		tween_active.kill()
	tween_active = create_tween()
	if delay > 0.0:
		tween_active.tween_interval(delay)
	tween_active.set_trans(Tween.TRANS_BOUNCE).set_ease(Tween.EASE_OUT)
	tween_active.tween_property(self, "position", target_pos, duration)
	return tween_active

func set_selected(selected: bool):
	is_selected = selected
	var target_z = 0.35 if selected else 0.0
	var target_scale = Vector3(1.2, 1.2, 1.2) if selected else Vector3.ONE

	var tw = create_tween().set_parallel(true)
	tw.set_trans(Tween.TRANS_BACK).set_ease(Tween.EASE_OUT)
	tw.tween_property(self, "position:z", target_z, 0.2)
	tw.tween_property(self, "scale", target_scale, 0.2)

func play_match_effect(on_complete: Callable):
	if tween_active and tween_active.is_valid():
		tween_active.kill()

	var tw = create_tween()
	tw.set_trans(Tween.TRANS_BACK).set_ease(Tween.EASE_OUT)
	tw.tween_property(self, "scale", Vector3(1.35, 1.35, 1.35), 0.12)
	tw.parallel().tween_property(self, "position:z", 0.5, 0.12)
	tw.set_trans(Tween.TRANS_QUAD).set_ease(Tween.EASE_IN)
	tw.tween_property(self, "scale", Vector3.ZERO, 0.15)
	tw.tween_callback(func():
		if on_complete.is_valid():
			on_complete.call()
		queue_free()
	)

```

### candy_crush_3d/scripts/hud.gd
```gdscript
extends CanvasLayer
class_name HUD

signal restart_pressed(zen_mode)
signal toggle_zen_pressed(zen_active)

@onready var score_label = $Margin/VBoxTop/HBoxHeader/PanelScore/VBox/ScoreVal
@onready var moves_label = $Margin/VBoxTop/HBoxHeader/PanelMoves/VBox/MovesVal
@onready var target_label = $Margin/VBoxTop/HBoxHeader/PanelTarget/VBox/TargetVal
@onready var progress_bar = $Margin/VBoxTop/ProgressBar
@onready var combo_banner = $CenterBanner/ComboLabel
@onready var game_over_panel = $GameOverPanel
@onready var final_score_label = $GameOverPanel/VBox/FinalScore
@onready var title_label = $GameOverPanel/VBox/Title
@onready var btn_zen = $Margin/HBoxBottom/BtnZen

var current_displayed_score: int = 0
var target_score_val: int = 5000
var is_zen: bool = false

func _ready():
	combo_banner.visible = false
	game_over_panel.visible = false
	$Margin/HBoxBottom/BtnRestart.pressed.connect(_on_restart_clicked)
	btn_zen.pressed.connect(_on_toggle_zen_clicked)
	$GameOverPanel/VBox/BtnPlayAgain.pressed.connect(_on_restart_clicked)
	progress_bar.max_value = target_score_val

func update_score(new_score: int, added: int):
	var tw = create_tween()
	tw.tween_method(func(val):
		score_label.text = str(val)
	, current_displayed_score, new_score, 0.4)
	current_displayed_score = new_score
	progress_bar.value = new_score

func update_moves(moves: int):
	if is_zen:
		moves_label.text = "∞"
	else:
		moves_label.text = str(moves)

func show_combo(combo: int, message: String):
	combo_banner.text = message
	combo_banner.visible = true
	combo_banner.scale = Vector2(0.5, 0.5)

	var tw = create_tween()
	tw.set_trans(Tween.TRANS_BACK).set_ease(Tween.EASE_OUT)
	tw.tween_property(combo_banner, "scale", Vector2(1.3, 1.3), 0.25)
	tw.tween_property(combo_banner, "scale", Vector2.ONE, 0.15)
	tw.tween_interval(0.6)
	tw.tween_property(combo_banner, "modulate:a", 0.0, 0.2)
	tw.tween_callback(func():
		combo_banner.visible = false
		combo_banner.modulate.a = 1.0
	)

func show_game_over(won: bool, final_score: int):
	game_over_panel.visible = true
	title_label.text = "LEVEL COMPLETE! 🌟" if won else "OUT OF MOVES!"
	final_score_label.text = "Final Score: " + str(final_score)

func _on_restart_clicked():
	game_over_panel.visible = false
	current_displayed_score = 0
	score_label.text = "0"
	progress_bar.value = 0
	restart_pressed.emit(is_zen)

func _on_toggle_zen_clicked():
	is_zen = not is_zen
	btn_zen.text = "Challenge Mode" if is_zen else "Zen Mode"
	toggle_zen_pressed.emit(is_zen)

```

### candy_crush_3d/scripts/main.gd
```gdscript
extends Node3D

@onready var camera: Camera3D = $Camera3D
@onready var board: Match3Board = $Board
@onready var hud: HUD = $HUD
@onready var audio_mgr: AudioManager = $AudioManager

func _ready():
	board.set_audio_manager(audio_mgr)

	board.score_updated.connect(hud.update_score)
	board.moves_updated.connect(hud.update_moves)
	board.combo_triggered.connect(hud.show_combo)
	board.game_over.connect(hud.show_game_over)

	hud.restart_pressed.connect(board.restart_game)
	hud.toggle_zen_pressed.connect(func(zen):
		board.restart_game(zen)
	)

func _unhandled_input(event: InputEvent):
	if event is InputEventMouseButton and event.button_index == MOUSE_BUTTON_LEFT:
		var mouse_pos = event.position
		var ray_origin = camera.project_ray_origin(mouse_pos)
		var ray_dir = camera.project_ray_normal(mouse_pos)

		var board_plane = Plane(Vector3(0, 0, 1), board.global_position.z)
		var hit = board_plane.intersects_ray(ray_origin, ray_dir)
		if hit != null:
			board.handle_input_click(hit, event.pressed)

```
