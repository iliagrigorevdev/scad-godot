### asteroids_3d/addons/scad_importer/plugin.cfg

```text
[plugin]

name="OpenSCAD GLTF Importer"
description="Imports .scad files directly as 3D scenes using scad-gltf"
author="scad-gltf"
version="0.2"
script="scad_plugin.gd"

```

### asteroids_3d/addons/scad_importer/scad_importer.gd

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

### asteroids_3d/addons/scad_importer/scad_plugin.gd

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

### asteroids_3d/project.godot

```ini
config_version=5

[application]

config/name="Asteroids 3D"
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

[input_devices]

pointing/emulate_touch_from_mouse=true
```

### asteroids_3d/models/player_ship.scad

```openscad
$fn = 28;
$asa = 35;

module player_ship() {
    // Fuselage
    color([0.22, 0.26, 0.32], metalness=0.85, roughness=0.3) {
        hull() {
            translate([0, 1.8, 0.05]) sphere(r=0.15);
            translate([0, 0.3, 0.2]) sphere(r=0.45);
            translate([0, -1.1, 0.0]) cube([0.9, 0.6, 0.35], center=true);
        }
    }

    // Cockpit canopy (Glass glow)
    color([0.1, 0.8, 1.0], alpha=0.9, metalness=0.1, roughness=0.1, emissive=[0.15, 0.7, 1.0], emissiveIntensity=2.2) {
        hull() {
            translate([0, 0.8, 0.35]) sphere(r=0.22);
            translate([0, -0.1, 0.42]) sphere(r=0.28);
        }
    }

    // Wings
    color([0.32, 0.36, 0.44], metalness=0.85, roughness=0.25) {
        hull() {
            translate([-0.35, 0.3, 0.05]) cube([0.1, 0.8, 0.1], center=true);
            translate([-1.65, -0.85, 0.05]) cube([0.1, 0.4, 0.08], center=true);
            translate([-0.45, -1.15, 0.05]) cube([0.1, 0.5, 0.1], center=true);
        }
        hull() {
            translate([0.35, 0.3, 0.05]) cube([0.1, 0.8, 0.1], center=true);
            translate([1.65, -0.85, 0.05]) cube([0.1, 0.4, 0.08], center=true);
            translate([0.45, -1.15, 0.05]) cube([0.1, 0.5, 0.1], center=true);
        }
        translate([-1.6, -0.8, 0.25]) rotate([0, -15, 0]) cube([0.08, 0.4, 0.45], center=true);
        translate([1.6, -0.8, 0.25]) rotate([0, 15, 0]) cube([0.08, 0.4, 0.45], center=true);
    }

    // Wing stripe accents
    color([1.0, 0.45, 0.05], metalness=0.2, roughness=0.4) {
        translate([-0.75, -0.3, 0.1]) cube([0.12, 0.7, 0.04], center=true);
        translate([0.75, -0.3, 0.1]) cube([0.12, 0.7, 0.04], center=true);
    }

    // Wing Cannons
    color([0.15, 0.15, 0.18], metalness=0.9, roughness=0.2) {
        translate([-1.45, 0.15, 0.05]) rotate([90, 0, 0]) cylinder(r=0.06, h=1.0, center=true);
        translate([1.45, 0.15, 0.05]) rotate([90, 0, 0]) cylinder(r=0.06, h=1.0, center=true);
    }

    // Thruster Nozzles & Glow (-Y)
    color([0.12, 0.12, 0.15], metalness=0.95, roughness=0.4) {
        translate([-0.3, -1.2, 0.05]) rotate([90, 0, 0]) cylinder(r=0.17, h=0.25, center=true);
        translate([0.3, -1.2, 0.05]) rotate([90, 0, 0]) cylinder(r=0.17, h=0.25, center=true);
    }
    color([0.2, 0.75, 1.0], emissive=[0.3, 0.85, 1.0], emissiveIntensity=3.5) {
        translate([-0.3, -1.32, 0.05]) rotate([90, 0, 0]) cylinder(r1=0.12, r2=0.03, h=0.35, center=true);
        translate([0.3, -1.32, 0.05]) rotate([90, 0, 0]) cylinder(r1=0.12, r2=0.03, h=0.35, center=true);
    }
}

player_ship();
```

### asteroids_3d/models/asteroid_large.scad

```openscad
$fn = 24;
$asa = 35;

module asteroid_large() {
    color([0.48, 0.44, 0.40], metalness=0.15, roughness=0.85) {
        difference() {
            hull() {
                sphere(r=1.8);
                translate([1.2, 0.8, 0.5]) sphere(r=1.0);
                translate([-1.1, 0.9, -0.7]) sphere(r=1.1);
                translate([0.6, -1.3, 0.8]) sphere(r=0.9);
                translate([-1.3, -0.8, -0.5]) sphere(r=1.0);
                translate([0, 1.5, -0.6]) sphere(r=0.8);
                translate([0, -1.5, 0.2]) sphere(r=0.95);
                translate([1.0, 0, -1.3]) sphere(r=0.9);
                translate([-0.8, 0, 1.3]) sphere(r=0.9);
            }
            translate([1.7, 0.6, 0.7]) sphere(r=0.8);
            translate([-1.3, -1.1, 0.9]) sphere(r=0.7);
            translate([0, -1.8, -0.6]) sphere(r=0.85);
            translate([-1.5, 1.0, -0.8]) sphere(r=0.65);
        }
    }
}

asteroid_large();
```

### asteroids_3d/models/asteroid_medium.scad

```openscad
$fn = 20;
$asa = 35;

module asteroid_medium() {
    color([0.52, 0.46, 0.42], metalness=0.15, roughness=0.85) {
        difference() {
            hull() {
                sphere(r=1.1);
                translate([0.8, -0.6, 0.4]) sphere(r=0.6);
                translate([-0.7, 0.7, -0.5]) sphere(r=0.7);
                translate([-0.8, -0.6, 0.3]) sphere(r=0.65);
                translate([0.5, 0.9, 0.3]) sphere(r=0.55);
                translate([0, 0, -1.0]) sphere(r=0.6);
            }
            translate([0.9, 0.4, 0.5]) sphere(r=0.5);
            translate([-0.7, -0.7, -0.6]) sphere(r=0.45);
        }
    }
}

asteroid_medium();
```

### asteroids_3d/models/asteroid_small.scad

```openscad
$fn = 16;
$asa = 35;

module asteroid_small() {
    color([0.55, 0.50, 0.48], metalness=0.1, roughness=0.85) {
        difference() {
            hull() {
                sphere(r=0.65);
                translate([0.5, 0.4, -0.2]) sphere(r=0.35);
                translate([-0.4, -0.5, 0.3]) sphere(r=0.4);
                translate([-0.4, 0.5, 0.1]) sphere(r=0.35);
                translate([0.3, -0.4, -0.3]) sphere(r=0.3);
            }
            translate([0.5, -0.3, 0.4]) sphere(r=0.3);
        }
    }
}

asteroid_small();
```

### asteroids_3d/models/ufo_ship.scad

```openscad
$fn = 32;
$asa = 35;

module ufo_ship() {
    color([0.35, 0.40, 0.48], metalness=0.9, roughness=0.2) {
        hull() {
            cylinder(r=2.0, h=0.15, center=true);
            translate([0, 0, 0.22]) cylinder(r=1.1, h=0.18, center=true);
        }
        hull() {
            cylinder(r=2.0, h=0.15, center=true);
            translate([0, 0, -0.22]) cylinder(r=1.2, h=0.18, center=true);
        }
    }

    color([0.15, 0.18, 0.22], metalness=0.95, roughness=0.4) {
        cylinder(r=2.08, h=0.08, center=true);
    }

    for (a = [0:45:315]) {
        rotate([0, 0, a])
        translate([1.95, 0, 0])
        color([1.0, 0.1, 0.3], emissive=[1.0, 0.15, 0.35], emissiveIntensity=3.0)
            sphere(r=0.11);
    }

    color([0.2, 0.9, 0.6], alpha=0.85, metalness=0.1, roughness=0.05, emissive=[0.1, 0.8, 0.5], emissiveIntensity=1.6) {
        translate([0, 0, 0.26])
            scale([1, 1, 0.8])
            sphere(r=0.8);
    }

    color([0.1, 0.7, 1.0], emissive=[0.2, 0.8, 1.0], emissiveIntensity=2.5) {
        translate([0, 0, -0.3])
            cylinder(r1=0.55, r2=0.18, h=0.18, center=true);
    }
}

ufo_ship();
```

### asteroids_3d/models/laser_player.scad

```openscad
$fn = 16;
$asa = 30;

module laser_player() {
    color([0.2, 0.8, 1.0], emissive=[0.3, 0.9, 1.0], emissiveIntensity=4.0) {
        hull() {
            translate([0, 0.8, 0]) sphere(r=0.1);
            translate([0, -0.8, 0]) sphere(r=0.1);
        }
    }
}

laser_player();
```

### asteroids_3d/models/laser_ufo.scad

```openscad
$fn = 16;
$asa = 30;

module laser_ufo() {
    color([1.0, 0.2, 0.1], emissive=[1.0, 0.3, 0.1], emissiveIntensity=4.0) {
        hull() {
            translate([0, 0.7, 0]) sphere(r=0.12);
            translate([0, -0.7, 0]) sphere(r=0.12);
        }
    }
}

laser_ufo();
```

### asteroids_3d/models/debris.scad

```openscad
$fn = 12;
$asa = 35;

module debris() {
    color([0.6, 0.55, 0.5], metalness=0.4, roughness=0.7, emissive=[0.9, 0.4, 0.1], emissiveIntensity=1.5) {
        hull() {
            translate([0.3, 0.2, 0]) sphere(r=0.2);
            translate([-0.3, -0.1, 0.1]) sphere(r=0.15);
            translate([0, -0.3, -0.2]) sphere(r=0.18);
            translate([0.1, 0.4, 0.3]) sphere(r=0.1);
        }
    }
}

debris();
```

### asteroids_3d/scripts/sound_manager.gd

```gdscript
extends Node

var laser_stream: AudioStreamWAV
var ufo_laser_stream: AudioStreamWAV
var explosion_large_stream: AudioStreamWAV
var explosion_medium_stream: AudioStreamWAV
var explosion_small_stream: AudioStreamWAV
var hyperspace_stream: AudioStreamWAV
var game_over_stream: AudioStreamWAV

var audio_players: Array[AudioStreamPlayer] = []
var max_players: int = 10
var current_player: int = 0

func _ready() -> void:
	for i in range(max_players):
		var p = AudioStreamPlayer.new()
		add_child(p)
		audio_players.append(p)

	_generate_sounds()

func _generate_sounds() -> void:
	laser_stream = _create_sweep(0.12, 920.0, 220.0, 0.45)
	ufo_laser_stream = _create_sweep(0.16, 1250.0, 480.0, 0.4)
	hyperspace_stream = _create_sweep(0.28, 160.0, 1200.0, 0.5)
	game_over_stream = _create_game_over_tune()

	# High-impact multi-layer explosions
	# (shockwave crack transient + deep sub-bass drop + dynamic low-pass debris roar + soft analog saturation)
	explosion_large_stream = _create_layered_explosion(0.68, 0.95, 150.0, 32.0, 85.0)
	explosion_medium_stream = _create_layered_explosion(0.48, 0.85, 200.0, 48.0, 120.0)
	explosion_small_stream = _create_layered_explosion(0.30, 0.75, 260.0, 70.0, 160.0)

func play_laser() -> void:
	_play(laser_stream, 1.0, 0.94, 1.06)

func play_ufo_laser() -> void:
	_play(ufo_laser_stream, 1.0, 0.94, 1.06)

func play_explosion(size_type: int = 0) -> void:
	match size_type:
		0: # Large asteroid, UFO, player destruction
			_play(explosion_large_stream, 1.0, 0.90, 1.10)
		1: # Medium asteroid
			_play(explosion_medium_stream, 1.0, 0.92, 1.08)
		2: # Small asteroid
			_play(explosion_small_stream, 1.0, 0.95, 1.15)
		_:
			_play(explosion_large_stream, 1.0, 0.90, 1.10)

func play_hyperspace() -> void:
	_play(hyperspace_stream, 1.0, 0.98, 1.04)

func play_game_over() -> void:
	_play(game_over_stream, 1.0, 1.0, 1.0)

func _play(stream: AudioStreamWAV, vol_mult: float, min_pitch: float, max_pitch: float) -> void:
	if not stream:
		return
	var player = audio_players[current_player]
	current_player = (current_player + 1) % max_players
	player.stream = stream
	player.volume_db = linear_to_db(vol_mult)
	player.pitch_scale = randf_range(min_pitch, max_pitch)
	player.play()

func _create_layered_explosion(dur: float, vol: float, bass_start: float, bass_end: float, crack_speed: float) -> AudioStreamWAV:
	var stream = AudioStreamWAV.new()
	stream.format = AudioStreamWAV.FORMAT_8_BITS
	stream.mix_rate = 22050
	stream.stereo = false

	var total_samples = int(dur * 22050)
	var data = PackedByteArray()
	data.resize(total_samples)

	var lp_val1: float = 0.0
	var lp_val2: float = 0.0
	var bass_phase: float = 0.0

	for i in range(total_samples):
		var t = float(i) / 22050.0
		var progress = t / dur

		# 1. Initial Shockwave Transient / Crack
		var crack_env = exp(-t * crack_speed)
		var crack_noise = randf_range(-1.0, 1.0) * crack_env * 0.8

		# 2. Sub-bass boom pitch drop
		var bass_freq = lerp(bass_start, bass_end, pow(progress, 0.55))
		bass_phase += (bass_freq / 22050.0) * TAU
		var bass_env = exp(-t * 4.8) * 0.75
		var bass_sine = sin(bass_phase)
		var bass_tone = clampf(bass_sine * 1.6, -1.0, 1.0) * bass_env

		# 3. Dynamic Low-Pass Filtered Noise (Debris roar & thunder)
		var cutoff = lerp(0.45, 0.035, pow(progress, 0.5))
		var raw_noise = randf_range(-1.0, 1.0)
		lp_val1 = lerp(lp_val1, raw_noise, cutoff)
		lp_val2 = lerp(lp_val2, lp_val1, cutoff)
		var rumble_env = pow(1.0 - progress, 1.7) * 0.7
		var rumble = lp_val2 * rumble_env

		# 4. Sum + Soft Saturation / Distortion
		var mix = (crack_noise + bass_tone + rumble) * 1.35
		var saturated = mix / (1.0 + absf(mix))

		var final_val = saturated * vol
		var sample_byte = int(clampf((final_val * 0.5 + 0.5) * 255.0, 0, 255))
		data[i] = sample_byte

	stream.data = data
	return stream

func _create_sweep(dur: float, start_freq: float, end_freq: float, vol: float) -> AudioStreamWAV:
	var stream = AudioStreamWAV.new()
	stream.format = AudioStreamWAV.FORMAT_8_BITS
	stream.mix_rate = 22050
	stream.stereo = false

	var total_samples = int(dur * 22050)
	var data = PackedByteArray()
	data.resize(total_samples)

	for i in range(total_samples):
		var t = float(i) / 22050.0
		var progress = t / dur
		var freq = lerp(start_freq, end_freq, progress)
		var phase = fmod(t * freq, 1.0)
		var wave = (phase * 2.0 - 1.0)
		var env = (1.0 - progress) * vol
		var sample = int(clampf((wave * env * 0.5 + 0.5) * 255.0, 0, 255))
		data[i] = sample

	stream.data = data
	return stream

func _create_game_over_tune() -> AudioStreamWAV:
	var dur = 0.8
	var stream = AudioStreamWAV.new()
	stream.format = AudioStreamWAV.FORMAT_8_BITS
	stream.mix_rate = 22050
	stream.stereo = false

	var total_samples = int(dur * 22050)
	var data = PackedByteArray()
	data.resize(total_samples)

	var notes = [440.0, 392.0, 349.0, 261.0]
	for i in range(total_samples):
		var t = float(i) / 22050.0
		var note_idx = int(t / (dur / float(notes.size())))
		note_idx = clampi(note_idx, 0, notes.size() - 1)
		var freq = notes[note_idx]
		var phase = fmod(t * freq, 1.0)
		var wave = 1.0 if phase > 0.5 else -1.0
		var note_t = fmod(t, dur / float(notes.size()))
		var env = (1.0 - note_t / (dur / float(notes.size()))) * 0.4
		var sample = int(clampf((wave * env * 0.5 + 0.5) * 255.0, 0, 255))
		data[i] = sample

	stream.data = data
	return stream
```

### asteroids_3d/scripts/player.gd

```gdscript
extends Area3D

signal player_destroyed
signal laser_fired(pos: Vector3, forward_dir: Vector3)
signal hyperspace_used

const ROTATION_SPEED: float = 3.6
const ACCELERATION: float = 24.0
const MAX_SPEED: float = 19.0
const DRAG: float = 2.0
const FIRE_COOLDOWN: float = 0.18

var velocity: Vector3 = Vector3.ZERO
var fire_timer: float = 0.0
var invulnerable_timer: float = 3.0
var hyperspace_cooldown: float = 0.0
var is_alive: bool = true

var bounds_x: float = 28.0
var bounds_z: float = 17.0

var joystick_touch_index: int = -1
var joystick_start_pos: Vector2 = Vector2.ZERO
var joystick_current_pos: Vector2 = Vector2.ZERO

var fire_touch_index: int = -1
var touch_is_firing: bool = false

@onready var model_container: Node3D = $ModelContainer
@onready var thruster_light: OmniLight3D = $ThrusterLight

func _ready() -> void:
	add_to_group("player")
	var ship_scene = load("res://models/player_ship.scad")
	if ship_scene:
		var model = ship_scene.instantiate()
		model_container.add_child(model)
	thruster_light.visible = false

func _input(event: InputEvent) -> void:
	if not is_alive:
		joystick_touch_index = -1
		fire_touch_index = -1
		touch_is_firing = false
		return

	if event is InputEventScreenTouch:
		var half_width = get_viewport().get_visible_rect().size.x / 2.0
		if event.pressed:
			if event.position.x < half_width:
				if joystick_touch_index == -1:
					joystick_touch_index = event.index
					joystick_start_pos = event.position
					joystick_current_pos = event.position
			else:
				if fire_touch_index == -1:
					fire_touch_index = event.index
					touch_is_firing = true
		else:
			if event.index == joystick_touch_index:
				joystick_touch_index = -1
			if event.index == fire_touch_index:
				fire_touch_index = -1
				touch_is_firing = false

	elif event is InputEventScreenDrag:
		if event.index == joystick_touch_index:
			joystick_current_pos = event.position
			# Dynamic range clamp prevents the drag origin from drifting too far away
			var delta_pos = joystick_current_pos - joystick_start_pos
			var max_range = 80.0
			if delta_pos.length() > max_range:
				joystick_start_pos = joystick_current_pos - delta_pos.normalized() * max_range

func _physics_process(delta: float) -> void:
	if not is_alive:
		return

	if fire_timer > 0.0:
		fire_timer -= delta
	if hyperspace_cooldown > 0.0:
		hyperspace_cooldown -= delta

	if invulnerable_timer > 0.0:
		invulnerable_timer -= delta
		model_container.visible = fmod(invulnerable_timer * 10.0, 2.0) > 1.0
	else:
		model_container.visible = true

	# Turning & Joystick calculations
	var turn_dir: float = 0.0
	var touch_thrust: bool = false
	var touch_brake: bool = false

	if joystick_touch_index != -1:
		var delta_pos = joystick_current_pos - joystick_start_pos
		var dx = delta_pos.x
		var dy = delta_pos.y
		var max_range = 80.0

		# Move forward: dragging up activates thrust smoothly
		if dy < -18.0:
			touch_thrust = true
		elif dy > 24.0:
			touch_brake = true

		# Rotation: non-linear analog response with forward corridor.
		# When thrusting forward, a wider deadband prevents unintended thumb wobbles from spinning the ship.
		var deadzone_x = 32.0 if touch_thrust else 14.0
		if absf(dx) > deadzone_x:
			var norm = clampf((absf(dx) - deadzone_x) / (max_range - deadzone_x), 0.0, 1.0)
			var curved_norm = pow(norm, 1.5)
			turn_dir = -signf(dx) * curved_norm

	if Input.is_key_pressed(KEY_A) or Input.is_key_pressed(KEY_LEFT):
		turn_dir += 1.0
	if Input.is_key_pressed(KEY_D) or Input.is_key_pressed(KEY_RIGHT):
		turn_dir -= 1.0

	turn_dir = clampf(turn_dir, -1.0, 1.0)
	rotation.y += turn_dir * ROTATION_SPEED * delta

	# Forward Thrust
	var is_thrusting: bool = touch_thrust or Input.is_key_pressed(KEY_W) or Input.is_key_pressed(KEY_UP)
	# Reverse Brake
	var is_braking: bool = touch_brake or Input.is_key_pressed(KEY_S) or Input.is_key_pressed(KEY_DOWN)

	thruster_light.visible = is_thrusting

	if is_thrusting:
		var forward_vec: Vector3 = -transform.basis.z
		velocity += forward_vec * ACCELERATION * delta
		if velocity.length() > MAX_SPEED:
			velocity = velocity.normalized() * MAX_SPEED
	elif is_braking:
		velocity = velocity.move_toward(Vector3.ZERO, DRAG * 3.5 * delta)
	else:
		velocity = velocity.move_toward(Vector3.ZERO, DRAG * delta)

	# Movement
	global_position += velocity * delta
	global_position.y = 0.0

	# Screen wrap-around
	if global_position.x > bounds_x:
		global_position.x = -bounds_x
	elif global_position.x < -bounds_x:
		global_position.x = bounds_x

	if global_position.z > bounds_z:
		global_position.z = -bounds_z
	elif global_position.z < -bounds_z:
		global_position.z = bounds_z

	# Shooting
	if (touch_is_firing or Input.is_key_pressed(KEY_SPACE) or Input.is_key_pressed(KEY_J)) and fire_timer <= 0.0:
		shoot()

	# Hyperspace emergency jump
	if (Input.is_key_pressed(KEY_SHIFT) or Input.is_key_pressed(KEY_H)) and hyperspace_cooldown <= 0.0:
		hyperspace_jump()

func shoot() -> void:
	fire_timer = FIRE_COOLDOWN
	var spawn_pos: Vector3 = global_position - transform.basis.z * 1.6
	laser_fired.emit(spawn_pos, -transform.basis.z)

func hyperspace_jump() -> void:
	hyperspace_cooldown = 4.0
	global_position = Vector3(randf_range(-22.0, 22.0), 0.0, randf_range(-14.0, 14.0))
	velocity = Vector3.ZERO
	invulnerable_timer = 1.8
	hyperspace_used.emit()

func hit() -> void:
	if invulnerable_timer > 0.0 or not is_alive:
		return
	is_alive = false
	visible = false
	joystick_touch_index = -1
	fire_touch_index = -1
	touch_is_firing = false
	player_destroyed.emit()

func respawn(pos: Vector3) -> void:
	global_position = pos
	velocity = Vector3.ZERO
	rotation.y = 0.0
	is_alive = true
	visible = true
	invulnerable_timer = 3.0
	joystick_touch_index = -1
	fire_touch_index = -1
	touch_is_firing = false
```

### asteroids_3d/scenes/player.tscn

```tscn
[gd_scene load_steps=3 format=3 uid="uid://b1player"]

[ext_resource type="Script" path="res://scripts/player.gd" id="1_player"]

[sub_resource type="SphereShape3D" id="SphereShape3D_hitbox"]
radius = 1.2

[node name="Player" type="Area3D"]
script = ExtResource("1_player")

[node name="CollisionShape3D" type="CollisionShape3D" parent="."]
shape = SubResource("SphereShape3D_hitbox")

[node name="ModelContainer" type="Node3D" parent="."]

[node name="ThrusterLight" type="OmniLight3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 1.4)
light_color = Color(0.2, 0.8, 1, 1)
light_energy = 3.5
omni_range = 3.0
```

### asteroids_3d/scripts/asteroid.gd

```gdscript
extends Area3D

enum Size { LARGE, MEDIUM, SMALL }

signal destroyed(size: Size, pos: Vector3)

@export var asteroid_size: Size = Size.LARGE

var velocity: Vector3 = Vector3.ZERO
var rotation_axis: Vector3 = Vector3.UP
var rotation_speed: float = 1.0

var bounds_x: float = 29.0
var bounds_z: float = 18.0

@onready var model_container: Node3D = $ModelContainer
@onready var collision_shape: CollisionShape3D = $CollisionShape3D

func _ready() -> void:
	add_to_group("asteroids")
	body_entered.connect(_on_body_entered)
	area_entered.connect(_on_area_entered)

	var model_path: String = ""
	var col_radius: float = 1.8

	match asteroid_size:
		Size.LARGE:
			model_path = "res://models/asteroid_large.scad"
			col_radius = 2.1
		Size.MEDIUM:
			model_path = "res://models/asteroid_medium.scad"
			col_radius = 1.3
		Size.SMALL:
			model_path = "res://models/asteroid_small.scad"
			col_radius = 0.75

	var shape = SphereShape3D.new()
	shape.radius = col_radius
	collision_shape.shape = shape

	var scad_scene = load(model_path)
	if scad_scene:
		var model = scad_scene.instantiate()
		var rand_scale = randf_range(0.9, 1.15)
		model.scale = Vector3.ONE * rand_scale
		model_container.add_child(model)

	rotation_axis = Vector3(
		randf_range(-1.0, 1.0),
		randf_range(-1.0, 1.0),
		randf_range(-1.0, 1.0)
	).normalized()
	rotation_speed = randf_range(0.8, 2.2)

func _physics_process(delta: float) -> void:
	global_position += velocity * delta
	global_position.y = 0.0

	model_container.rotate(rotation_axis, rotation_speed * delta)

	# Wrap bounds
	if global_position.x > bounds_x:
		global_position.x = -bounds_x
	elif global_position.x < -bounds_x:
		global_position.x = bounds_x

	if global_position.z > bounds_z:
		global_position.z = -bounds_z
	elif global_position.z < -bounds_z:
		global_position.z = bounds_z

func hit() -> void:
	destroyed.emit(asteroid_size, global_position)
	queue_free()

func _on_area_entered(area: Area3D) -> void:
	if area.is_in_group("player"):
		area.hit()

func _on_body_entered(_body: Node) -> void:
	pass
```

### asteroids_3d/scenes/asteroid.tscn

```tscn
[gd_scene load_steps=2 format=3 uid="uid://b1asteroid"]

[ext_resource type="Script" path="res://scripts/asteroid.gd" id="1_asteroid"]

[node name="Asteroid" type="Area3D"]
script = ExtResource("1_asteroid")

[node name="CollisionShape3D" type="CollisionShape3D" parent="."]

[node name="ModelContainer" type="Node3D" parent="."]
```

### asteroids_3d/scripts/laser.gd

```gdscript
extends Area3D

@export var is_enemy: bool = false
@export var speed: float = 44.0
@export var lifetime: float = 1.1

var direction: Vector3 = Vector3.FORWARD
var timer: float = 0.0

var bounds_x: float = 29.0
var bounds_z: float = 18.0

@onready var model_container: Node3D = $ModelContainer

func _ready() -> void:
	add_to_group("lasers")
	area_entered.connect(_on_area_entered)

	var model_path = "res://models/laser_ufo.scad" if is_enemy else "res://models/laser_player.scad"
	var scad_scene = load(model_path)
	if scad_scene:
		var model = scad_scene.instantiate()
		model_container.add_child(model)

func _physics_process(delta: float) -> void:
	timer += delta
	if timer >= lifetime:
		queue_free()
		return

	global_position += direction * speed * delta
	global_position.y = 0.0

	# Wrap bounds
	if global_position.x > bounds_x:
		global_position.x = -bounds_x
	elif global_position.x < -bounds_x:
		global_position.x = bounds_x

	if global_position.z > bounds_z:
		global_position.z = -bounds_z
	elif global_position.z < -bounds_z:
		global_position.z = bounds_z

func _on_area_entered(area: Area3D) -> void:
	if is_enemy:
		if area.is_in_group("player"):
			area.hit()
			queue_free()
	else:
		if area.is_in_group("asteroids") or area.is_in_group("ufo"):
			area.hit()
			queue_free()
```

### asteroids_3d/scenes/laser.tscn

```tscn
[gd_scene load_steps=3 format=3 uid="uid://b1laser"]

[ext_resource type="Script" path="res://scripts/laser.gd" id="1_laser"]

[sub_resource type="SphereShape3D" id="SphereShape3D_laser"]
radius = 0.45

[node name="Laser" type="Area3D"]
script = ExtResource("1_laser")

[node name="CollisionShape3D" type="CollisionShape3D" parent="."]
shape = SubResource("SphereShape3D_laser")

[node name="ModelContainer" type="Node3D" parent="."]
```

### asteroids_3d/scripts/ufo.gd

```gdscript
extends Area3D

signal ufo_destroyed(pos: Vector3)
signal ufo_fired(pos: Vector3, dir: Vector3)

const SPEED: float = 9.5
const FIRE_INTERVAL: float = 1.8

var direction_x: float = 1.0
var base_z: float = 0.0
var time_alive: float = 0.0
var fire_timer: float = 1.0

var bounds_x: float = 30.0

@onready var model_container: Node3D = $ModelContainer

func _ready() -> void:
	add_to_group("ufo")
	area_entered.connect(_on_area_entered)

	var scad_scene = load("res://models/ufo_ship.scad")
	if scad_scene:
		var model = scad_scene.instantiate()
		model_container.add_child(model)

func _physics_process(delta: float) -> void:
	time_alive += delta
	fire_timer += delta

	global_position.x += direction_x * SPEED * delta
	global_position.z = base_z + sin(time_alive * 2.2) * 3.5
	global_position.y = 0.0

	model_container.rotate_y(1.5 * delta)

	if fire_timer >= FIRE_INTERVAL:
		fire_timer = 0.0
		fire_at_player()

	if (direction_x > 0 and global_position.x > bounds_x + 3.0) or (direction_x < 0 and global_position.x < -bounds_x - 3.0):
		queue_free()

func fire_at_player() -> void:
	var players = get_tree().get_nodes_in_group("player")
	var target_dir: Vector3 = Vector3(randf_range(-1.0, 1.0), 0.0, randf_range(-1.0, 1.0)).normalized()
	if players.size() > 0 and players[0].is_alive:
		var diff = players[0].global_position - global_position
		diff.y = 0.0
		if diff.length() > 0.1:
			target_dir = diff.normalized()

	ufo_fired.emit(global_position, target_dir)

func hit() -> void:
	ufo_destroyed.emit(global_position)
	queue_free()

func _on_area_entered(area: Area3D) -> void:
	if area.is_in_group("player"):
		area.hit()
		hit()
```

### asteroids_3d/scenes/ufo.tscn

```tscn
[gd_scene load_steps=3 format=3 uid="uid://b1ufo"]

[ext_resource type="Script" path="res://scripts/ufo.gd" id="1_ufo"]

[sub_resource type="SphereShape3D" id="SphereShape3D_ufo"]
radius = 2.0

[node name="UFO" type="Area3D"]
script = ExtResource("1_ufo")

[node name="CollisionShape3D" type="CollisionShape3D" parent="."]
shape = SubResource("SphereShape3D_ufo")

[node name="ModelContainer" type="Node3D" parent="."]
```

### asteroids_3d/scripts/explosion.gd

```gdscript
extends Node3D

const DEBRIS_COUNT: int = 11
const DURATION: float = 0.85

var timer: float = 0.0
var chunks: Array = []

func _ready() -> void:
	var debris_scene = load("res://models/debris.scad")
	for i in range(DEBRIS_COUNT):
		var chunk_pivot = Node3D.new()
		add_child(chunk_pivot)

		if debris_scene:
			var model = debris_scene.instantiate()
			chunk_pivot.add_child(model)

		var speed = randf_range(8.0, 22.0)
		var angle = randf() * TAU
		var vel = Vector3(cos(angle), randf_range(-0.5, 0.5), sin(angle)) * speed
		var rot_axis = Vector3(randf_range(-1,1), randf_range(-1,1), randf_range(-1,1)).normalized()
		var rot_spd = randf_range(6.0, 18.0)

		chunks.append({
			"node": chunk_pivot,
			"vel": vel,
			"rot_axis": rot_axis,
			"rot_spd": rot_spd
		})

func _process(delta: float) -> void:
	timer += delta
	var progress = timer / DURATION
	if progress >= 1.0:
		queue_free()
		return

	var scale_factor = 1.0 - progress
	for item in chunks:
		item["node"].global_position += item["vel"] * delta
		item["node"].rotate(item["rot_axis"], item["rot_spd"] * delta)
		item["node"].scale = Vector3.ONE * scale_factor
```

### asteroids_3d/scenes/explosion.tscn

```tscn
[gd_scene load_steps=2 format=3 uid="uid://b1explosion"]

[ext_resource type="Script" path="res://scripts/explosion.gd" id="1_explosion"]

[node name="Explosion" type="Node3D"]
script = ExtResource("1_explosion")
```

### asteroids_3d/scripts/hud.gd

```gdscript
extends CanvasLayer

@onready var score_label: Label = $Margin/TopRow/ScoreLabel
@onready var high_score_label: Label = $Margin/TopRow/HighScoreLabel
@onready var wave_label: Label = $Margin/TopRow/WaveLabel
@onready var lives_label: Label = $Margin/TopRow/LivesLabel
@onready var game_over_panel: PanelContainer = $GameOverCenter/GameOverPanel
@onready var final_score_label: Label = $GameOverCenter/GameOverPanel/VBox/FinalScoreLabel

func _ready() -> void:
	game_over_panel.visible = false

func update_hud(score: int, high_score: int, wave: int, lives: int) -> void:
	score_label.text = "SCORE %05d" % score
	high_score_label.text = "HIGH %05d" % high_score
	wave_label.text = "WAVE %d" % wave

	# Use ASCII chevrons ('^') representing ship hulls for guaranteed web font compatibility
	var ships_str = ""
	for i in range(max(0, lives)):
		ships_str += "^ "
	lives_label.text = "LIVES %s" % ships_str.strip_edges()

func show_game_over(final_score: int) -> void:
	final_score_label.text = "FINAL SCORE: %05d" % final_score
	game_over_panel.visible = true

func hide_game_over() -> void:
	game_over_panel.visible = false
```

### asteroids_3d/scenes/hud.tscn

```tscn
[gd_scene load_steps=4 format=3 uid="uid://b1hud"]

[ext_resource type="Script" path="res://scripts/hud.gd" id="1_hud"]

[sub_resource type="StyleBoxFlat" id="StyleBoxFlat_panel"]
bg_color = Color(0.06, 0.08, 0.14, 0.94)
border_width_left = 3
border_width_top = 3
border_width_right = 3
border_width_bottom = 3
border_color = Color(0.2, 0.8, 1, 1)
corner_radius_top_left = 8
corner_radius_top_right = 8
corner_radius_bottom_right = 8
corner_radius_bottom_left = 8

[sub_resource type="LabelSettings" id="LabelSettings_large"]
font_size = 32
font_color = Color(1, 0.35, 0.35, 1)
outline_size = 4
outline_color = Color(0, 0, 0, 1)

[node name="HUD" type="CanvasLayer"]
script = ExtResource("1_hud")

[node name="Margin" type="MarginContainer" parent="."]
anchors_preset = 10
anchor_right = 1.0
offset_bottom = 48.0
grow_horizontal = 2
theme_override_constants/margin_left = 18
theme_override_constants/margin_top = 12
theme_override_constants/margin_right = 18

[node name="TopRow" type="HBoxContainer" parent="Margin"]
layout_mode = 2

[node name="ScoreLabel" type="Label" parent="Margin/TopRow"]
layout_mode = 2
size_flags_horizontal = 3
text = "SCORE 00000"

[node name="HighScoreLabel" type="Label" parent="Margin/TopRow"]
layout_mode = 2
size_flags_horizontal = 3
text = "HIGH 00000"
horizontal_alignment = 1

[node name="WaveLabel" type="Label" parent="Margin/TopRow"]
layout_mode = 2
size_flags_horizontal = 3
text = "WAVE 1"
horizontal_alignment = 1

[node name="LivesLabel" type="Label" parent="Margin/TopRow"]
layout_mode = 2
size_flags_horizontal = 3
text = "LIVES ^ ^ ^"
horizontal_alignment = 2

[node name="GameOverCenter" type="CenterContainer" parent="."]
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2

[node name="GameOverPanel" type="PanelContainer" parent="GameOverCenter"]
custom_minimum_size = Vector2(380, 220)
layout_mode = 2
theme_override_styles/panel = SubResource("StyleBoxFlat_panel")

[node name="VBox" type="VBoxContainer" parent="GameOverCenter/GameOverPanel"]
layout_mode = 2
alignment = 1

[node name="Title" type="Label" parent="GameOverCenter/GameOverPanel/VBox"]
layout_mode = 2
text = "GAME OVER"
label_settings = SubResource("LabelSettings_large")
horizontal_alignment = 1

[node name="FinalScoreLabel" type="Label" parent="GameOverCenter/GameOverPanel/VBox"]
layout_mode = 2
text = "FINAL SCORE: 00000"
horizontal_alignment = 1

[node name="RestartPrompt" type="Label" parent="GameOverCenter/GameOverPanel/VBox"]
layout_mode = 2
text = "PRESS SPACE, ENTER OR TOUCH TO RESTART"
horizontal_alignment = 1
```

### asteroids_3d/scripts/main.gd

```gdscript
extends Node3D

const ASTEROID_SCENE = preload("res://scenes/asteroid.tscn")
const LASER_SCENE = preload("res://scenes/laser.tscn")
const UFO_SCENE = preload("res://scenes/ufo.tscn")
const EXPLOSION_SCENE = preload("res://scenes/explosion.tscn")

var score: int = 0
var high_score: int = 0
var wave: int = 1
var lives: int = 3
var is_game_over: bool = false

var ufo_spawn_timer: float = 18.0
var camera_trauma: float = 0.0

@onready var camera: Camera3D = $Camera3D
@onready var player: Area3D = $Player
@onready var hud: CanvasLayer = $HUD
@onready var sound_manager: Node = $SoundManager
@onready var laser_container: Node3D = $LaserContainer
@onready var asteroid_container: Node3D = $AsteroidContainer
@onready var ufo_container: Node3D = $UFOContainer
@onready var explosion_container: Node3D = $ExplosionContainer

func _ready() -> void:
	print("Asteroids 3D Initialized with Punchy Multi-Layer Audio and Smooth Shading")
	create_starfield()
	player.laser_fired.connect(_on_player_laser_fired)
	player.player_destroyed.connect(_on_player_destroyed)
	player.hyperspace_used.connect(_on_player_hyperspace)
	start_game()

func create_starfield() -> void:
	var star_mat = StandardMaterial3D.new()
	star_mat.shading_mode = BaseMaterial3D.SHADING_MODE_UNSHADED
	star_mat.albedo_color = Color(1.3, 1.3, 1.6, 1.0)

	var star_mesh = BoxMesh.new()
	star_mesh.size = Vector3(0.18, 0.18, 0.18)
	star_mesh.material = star_mat

	var multimesh = MultiMesh.new()
	multimesh.transform_format = MultiMesh.TRANSFORM_3D
	multimesh.mesh = star_mesh
	multimesh.instance_count = 140

	for i in range(140):
		var t = Transform3D()
		var pos = Vector3(
			randf_range(-35.0, 35.0),
			randf_range(-14.0, -8.0),
			randf_range(-22.0, 22.0)
		)
		t.origin = pos
		var s = randf_range(0.6, 1.8)
		t = t.scaled(Vector3(s, s, s))
		multimesh.set_instance_transform(i, t)

	var mmi = MultiMeshInstance3D.new()
	mmi.multimesh = multimesh
	add_child(mmi)

func _input(event: InputEvent) -> void:
	if is_game_over and event is InputEventScreenTouch and event.pressed:
		get_viewport().set_input_as_handled()
		start_game()

func start_game() -> void:
	score = 0
	wave = 1
	lives = 3
	is_game_over = false
	hud.hide_game_over()
	hud.update_hud(score, high_score, wave, lives)

	clear_all_entities()
	player.respawn(Vector3.ZERO)
	spawn_wave(wave)

func clear_all_entities() -> void:
	for c in asteroid_container.get_children():
		c.queue_free()
	for c in laser_container.get_children():
		c.queue_free()
	for c in ufo_container.get_children():
		c.queue_free()
	for c in explosion_container.get_children():
		c.queue_free()

func spawn_wave(current_wave: int) -> void:
	var asteroid_count: int = 3 + current_wave
	for i in range(asteroid_count):
		spawn_asteroid_at_edge(0)

func spawn_asteroid_at_edge(size_val: int) -> void:
	var asteroid = ASTEROID_SCENE.instantiate()
	asteroid.asteroid_size = size_val
	asteroid.destroyed.connect(_on_asteroid_destroyed)

	var spawn_pos = Vector3.ZERO
	if randf() > 0.5:
		spawn_pos.x = -28.0 if randf() > 0.5 else 28.0
		spawn_pos.z = randf_range(-16.0, 16.0)
	else:
		spawn_pos.x = randf_range(-26.0, 26.0)
		spawn_pos.z = -17.0 if randf() > 0.5 else 17.0

	var target_point = Vector3(randf_range(-10.0, 10.0), 0.0, randf_range(-8.0, 8.0))
	var move_speed = randf_range(4.5, 7.5) + (wave * 0.4)
	asteroid.velocity = (target_point - spawn_pos).normalized() * move_speed

	asteroid_container.add_child(asteroid)
	asteroid.global_position = spawn_pos

func _process(delta: float) -> void:
	# Camera shake decay
	if camera_trauma > 0.0:
		camera_trauma = max(0.0, camera_trauma - delta * 1.8)
		var shake = camera_trauma * camera_trauma
		camera.h_offset = randf_range(-1.0, 1.0) * shake * 0.6
		camera.v_offset = randf_range(-1.0, 1.0) * shake * 0.6
	else:
		camera.h_offset = 0.0
		camera.v_offset = 0.0

	if is_game_over:
		if Input.is_key_pressed(KEY_SPACE) or Input.is_key_pressed(KEY_ENTER):
			start_game()
		return

	# UFO spawning
	ufo_spawn_timer -= delta
	if ufo_spawn_timer <= 0.0 and ufo_container.get_child_count() == 0:
		ufo_spawn_timer = randf_range(20.0, 32.0)
		spawn_ufo()

	# Check wave completion
	if asteroid_container.get_child_count() == 0:
		wave += 1
		score += 500
		update_high_score()
		hud.update_hud(score, high_score, wave, lives)
		spawn_wave(wave)

func add_camera_shake(amount: float) -> void:
	camera_trauma = clampf(camera_trauma + amount, 0.0, 1.0)

func spawn_ufo() -> void:
	var ufo = UFO_SCENE.instantiate()
	var from_left: bool = randf() > 0.5
	ufo.direction_x = 1.0 if from_left else -1.0
	var spawn_pos = Vector3(-30.0 if from_left else 30.0, 0.0, randf_range(-10.0, 10.0))
	ufo_container.add_child(ufo)
	ufo.global_position = spawn_pos
	ufo.base_z = spawn_pos.z
	ufo.ufo_destroyed.connect(_on_ufo_destroyed)
	ufo.ufo_fired.connect(_on_ufo_fired)

func _on_player_laser_fired(pos: Vector3, forward_dir: Vector3) -> void:
	var laser = LASER_SCENE.instantiate()
	laser.is_enemy = false
	laser.direction = forward_dir
	laser.rotation.y = atan2(-forward_dir.x, -forward_dir.z)
	laser_container.add_child(laser)
	laser.global_position = pos
	sound_manager.play_laser()

func _on_ufo_fired(pos: Vector3, dir: Vector3) -> void:
	var laser = LASER_SCENE.instantiate()
	laser.is_enemy = true
	laser.direction = dir
	laser.rotation.y = atan2(-dir.x, -dir.z)
	laser_container.add_child(laser)
	laser.global_position = pos
	sound_manager.play_ufo_laser()

func _on_player_hyperspace() -> void:
	sound_manager.play_hyperspace()
	add_camera_shake(0.2)

func _on_asteroid_destroyed(size_val: int, pos: Vector3) -> void:
	spawn_explosion(pos)
	sound_manager.play_explosion(size_val)

	match size_val:
		0: # Large
			add_camera_shake(0.4)
			score += 20
			split_asteroid(1, pos)
		1: # Medium
			add_camera_shake(0.25)
			score += 50
			split_asteroid(2, pos)
		2: # Small
			add_camera_shake(0.15)
			score += 100

	update_high_score()
	hud.update_hud(score, high_score, wave, lives)

func split_asteroid(new_size_val: int, pos: Vector3) -> void:
	for i in range(2):
		var asteroid = ASTEROID_SCENE.instantiate()
		asteroid.asteroid_size = new_size_val
		asteroid.destroyed.connect(_on_asteroid_destroyed)

		var angle = randf() * TAU
		var spd = randf_range(8.0, 14.0) if new_size_val == 1 else randf_range(13.0, 19.0)
		asteroid.velocity = Vector3(cos(angle), 0.0, sin(angle)) * spd

		asteroid_container.add_child(asteroid)
		asteroid.global_position = pos

func _on_ufo_destroyed(pos: Vector3) -> void:
	score += 250
	update_high_score()
	hud.update_hud(score, high_score, wave, lives)
	spawn_explosion(pos)
	add_camera_shake(0.55)
	sound_manager.play_explosion(0)

func _on_player_destroyed() -> void:
	spawn_explosion(player.global_position)
	add_camera_shake(0.85)
	sound_manager.play_explosion(0)
	lives -= 1
	hud.update_hud(score, high_score, wave, lives)

	if lives > 0:
		get_tree().create_timer(1.2).timeout.connect(func():
			player.respawn(Vector3.ZERO)
		)
	else:
		is_game_over = true
		sound_manager.play_game_over()
		hud.show_game_over(score)

func spawn_explosion(pos: Vector3) -> void:
	var exp_node = EXPLOSION_SCENE.instantiate()
	explosion_container.add_child(exp_node)
	exp_node.global_position = pos

func update_high_score() -> void:
	if score > high_score:
		high_score = score
```

### asteroids_3d/scenes/main.tscn

```tscn
[gd_scene load_steps=7 format=3 uid="uid://b1mainscene"]

[ext_resource type="Script" path="res://scripts/main.gd" id="1_main"]
[ext_resource type="PackedScene" path="res://scenes/player.tscn" id="2_player"]
[ext_resource type="PackedScene" path="res://scenes/hud.tscn" id="3_hud"]
[ext_resource type="Script" path="res://scripts/sound_manager.gd" id="4_sound"]

[sub_resource type="Environment" id="Environment_space"]
background_mode = 1
background_color = Color(0.015, 0.02, 0.04, 1)
ambient_light_source = 2
ambient_light_color = Color(0.12, 0.15, 0.22, 1)
ambient_light_energy = 0.9
glow_enabled = true
glow_bloom = 0.25
glow_hdr_threshold = 0.7

[sub_resource type="CameraAttributesPractical" id="CameraAttributesPractical_cam"]

[node name="Main" type="Node3D"]
script = ExtResource("1_main")

[node name="SoundManager" type="Node" parent="."]
script = ExtResource("4_sound")

[node name="WorldEnvironment" type="WorldEnvironment" parent="."]
environment = SubResource("Environment_space")
camera_attributes = SubResource("CameraAttributesPractical_cam")

[node name="DirectionalLight3D" type="DirectionalLight3D" parent="."]
transform = Transform3D(0.866025, -0.353553, 0.353553, 0, 0.707107, 0.707107, -0.5, -0.612372, 0.612372, 10, 20, 10)
light_color = Color(1, 0.96, 0.9, 1)
light_energy = 1.7
shadow_enabled = true

[node name="Camera3D" type="Camera3D" parent="."]
transform = Transform3D(1, 0, 0, 0, 0.0871557, 0.996195, 0, -0.996195, 0.0871557, 0, 36, 3.2)
current = true
fov = 52.0

[node name="Player" parent="." instance=ExtResource("2_player")]

[node name="HUD" parent="." instance=ExtResource("3_hud")]

[node name="AsteroidContainer" type="Node3D" parent="."]

[node name="LaserContainer" type="Node3D" parent="."]

[node name="UFOContainer" type="Node3D" parent="."]

[node name="ExplosionContainer" type="Node3D" parent="."]
```
