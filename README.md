# Smart_Purge
Simple Smart Purge for 3d printer ecosystems running Klipper


>[!NOTE]
>Macro [gocde_macro SMART_PURGE]
>
>intent:
>
>relative purge line positioning between edge of bed and part print area.

<br>

>[!TIP]
>BONUS: You can edit variables and edit them on the fly too!
>
>How to make this work:
>
>in your PRINT_START macro, before printing simply add:
>
>{SMART_PURGE]
>
>then copy / paste below in your macros.cfg

<br>

>[!IMPORTANT]
>QUICK THINGS:
>
>```SMART_PURGE LINE_LENGTH=100 PURGE_F=360```
>
>What this does for you
>
>Relative positioning (G91) for the purge strokes, so it’s robust no matter where you start the macro.
>
>Absolute anchor to front-left first, so the lane is positioned correctly vs. your parts.
>
>No changes to extrusion mode — works with either, by issuing cumulative E values when you’re in absolute mode and incremental when you’re in relative mode.
>
>Super slow prime (F360) to avoid back-pressure clogs.
>
>Short line (default 100 mm) and you can tweak with LINE_LENGTH=….

```
[gcode_macro SMART_PURGE]
# description: Front-left purge; geometry-based flow; short slow prime; 100mm line; relative positioning
# --- Tunables (safe defaults) ---
variable_margin: 5.0                # mm from bed edges & parts (front/left)
variable_line_z: 0.25              # purge Z height
variable_line_length: 100.0        # mm length of moving purge line
variable_backstep: 0.8             # mm Y offset between lanes
variable_passes: 2                 # number of lanes (2 is plenty for purge)
variable_travel_f: 6000            # mm/min travel speed
variable_line_f: 600               # mm/min drawing speed (10 mm/s)
# Prime (stationary) — keep gentle to avoid back-pressure
variable_prime_e: 15.0             # mm filament for the initial prime
variable_prime_f: 180              # mm/min (3 mm/s) for the prime

# Bead geometry for flow math (adjust to your first-layer settings)
variable_layer_h: 0.25             # mm
variable_line_w: 0.60              # mm
variable_fil_d: 1.75               # mm filament diameter
variable_flow: 1.00                # scalar (1.0 = 100%)

gcode:
  # ---- Bed coords: (0,0)=front-left; user Y max defaults to 300 (rear endstop) ----
  {% set BED_MAX_X = (params.BED_MAX_X|default(printer.toolhead.axis_maximum.x)|float) %}
  {% set BED_MAX_Y = (params.BED_MAX_Y|default(300)|float) %}
  {% set margin = margin|float %}
  {% set line_len_req = line_length|float %}
  {% set backstep = backstep|float %}
  {% set passes = passes|int %}

  # Home if needed (prevents "move out of range")
  {% set homed = printer.toolhead.homed_axes %}
  {% if 'x' not in homed or 'y' not in homed or 'z' not in homed %}
    G28
  {% endif %}

  # ---- Front gap placement from EXCLUDE_OBJECT (keeps purge ahead of parts) ----
  {% set have_objs = printer.exclude_object is defined and printer.exclude_object.objects|length > 0 %}
  {% if have_objs %}
    {% set obj_min_y = None %}
    {% for o in printer.exclude_object.objects %}
      {% if o.polygon is defined and o.polygon|length > 0 %}
        {% for pt in o.polygon %}
          {% set y = pt[1]|float %}
          {% if obj_min_y is none or y < obj_min_y %}{% set obj_min_y = y %}{% endif %}
        {% endfor %}
      {% endif %}
    {% endfor %}
  {% endif %}

  {% if have_objs and obj_min_y is not none %}
    {% set front_low  = 0.0 + margin %}
    {% set front_high = obj_min_y - margin %}
    {% if front_high < front_low %}{% set front_high = front_low %}{% endif %}
  {% else %}
    {% set front_low  = 0.0 + margin %}
    {% set front_high = front_low + 60.0 %}
    {% if front_high > (BED_MAX_Y - margin) %}{% set front_high = BED_MAX_Y - margin %}{% endif %}
  {% endif %}
  {% set target_y = front_low + (front_high - front_low) * 0.5 %}

  # ---- X span: start at front-left; cap line length inside bed ----
  {% set start_x_abs = 0.0 + margin %}
  {% set avail_w = BED_MAX_X - (2*margin) %}
  {% if avail_w < 0 %}{% set avail_w = 0 %}{% endif %}
  {% set line_len = line_len_req if line_len_req <= avail_w else avail_w %}

  # ---- Limit lanes so Y steps never cross into the part envelope ----
  {% set max_delta = front_high - target_y %}
  {% if backstep <= 0 %}{% set backstep = 0.8 %}{% endif %}
  {% set max_lanes = 1 + (max_delta // backstep)|int %}
  {% if passes > max_lanes %}{% set passes = max_lanes %}{% endif %}
  {% if passes < 1 %}{% set passes = 1 %}{% endif %}

  # ---- Geometry-based flow: E per mm of XY travel ----
  {% set lw = line_w|float %}
  {% set lh = layer_h|float %}
  {% set fd = fil_d|float %}
  {% set flow = flow|float %}
  {% set fil_area = 3.1415926535 * (fd/2.0) * (fd/2.0) %}
  {% if fil_area <= 0 %}{% set fil_area = 2.405 %}{% endif %}
  {% set e_per_mm = (lw * lh * flow) / fil_area %}
  {% set e_move = e_per_mm * line_len %}

  # ---- Respect current modes; we'll only switch positioning, not extrusion mode ----
  {% set was_abs_coord = printer.gcode_move.absolute_coordinates %}
  {% set is_abs_extrude = printer.gcode_move.absolute_extrude %}

  # ---- Anchor in ABSOLUTE, then draw in RELATIVE positioning ----
  G90
  G92 E0
  G1 Z{line_z} F{travel_f}
  G1 X{start_x_abs} Y{target_y} F{travel_f}

  ; Slow, gentle stationary prime
  {% if is_abs_extrude %}
    G1 E{prime_e} F{prime_f}
    {% set e_total = prime_e|float %}
  {% else %}
    G1 E{prime_e} F{prime_f}
  {% endif %}

  G91

  {% for i in range(passes) %}
    {% if (i % 2) == 0 %}
      ; forward line
      {% if is_abs_extrude %}
        {% set e_total = e_total + e_move %}
        G1 X{line_len} E{e_total} F{line_f}
      {% else %}
        G1 X{line_len} E{e_move} F{line_f}
      {% endif %}
    {% else %}
      ; back line
      {% if is_abs_extrude %}
        {% set e_total = e_total + e_move %}
        G1 X{-line_len} E{e_total} F{line_f}
      {% else %}
        G1 X{-line_len} E{e_move} F{line_f}
      {% endif %}
    {% endif %}

    {% if i < (passes - 1) %}
      {% set desired = backstep %}
      {% set remaining = max_delta - (backstep * (i + 1)) %}
      {% if remaining < 0 %}{% set desired = backstep + remaining %}{% endif %}
      G1 Y{desired} F{travel_f}
    {% endif %}
  {% endfor %}

  G92 E0
  {% if was_abs_coord %} G90 {% else %} G91 {% endif %}

```
