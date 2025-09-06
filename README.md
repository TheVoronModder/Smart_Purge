# Smart_Purge_Check
Simple Smart Purge for 3d printer ecosystems running Klipper


>[!NOTE]
>Macro [gocde_macro SMART_PURGE]
>
>intent:
>
>relative purge line positioning between edge of bed and part print area.
>
>You may switch between a Check Mark or a DOT (blob) before the line purge
>
>SMART_PURGE STYLE=DOT
>
>SMART_PURGE STYLE=CHECK FRONT_GAP=40
>

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
# description: Front-left purge; STYLE=CHECK or DOT; draws check ✓ or blob, shifts +10mm, then purge line (≤100mm)
# --- Tunables ---
variable_front_gap: 30.0
variable_line_z: 0.25
variable_line_length: 100.0
variable_travel_f: 6000
variable_line_f: 600
variable_prime_e: 10.0
variable_prime_f: 120
variable_layer_h: 0.25
variable_line_w: 0.60
variable_fil_d: 1.75
variable_flow: 1.00
# Per-bed-size safety margins
variable_margin_250: 5.0
variable_margin_300: 5.0
variable_margin_350: 5.0

gcode:
  # --- Bed limits from config ---
  {% set BED_MIN_X = printer.toolhead.axis_minimum.x|float %}
  {% set BED_MAX_X = printer.toolhead.axis_maximum.x|float %}
  {% set BED_MIN_Y = printer.toolhead.axis_minimum.y|float %}
  {% set BED_MAX_Y = printer.toolhead.axis_maximum.y|float %}
  {% set BED_X = BED_MAX_X - BED_MIN_X %}
  {% set BED_Y = BED_MAX_Y - BED_MIN_Y %}

  # --- Pick size bucket & margin (avoid lambda/key) ---
  {% if BED_X < 275 %}
    {% set size = 250.0 %}
    {% set default_margin = margin_250|float %}
  {% elif BED_X < 325 %}
    {% set size = 300.0 %}
    {% set default_margin = margin_300|float %}
  {% else %}
    {% set size = 350.0 %}
    {% set default_margin = margin_350|float %}
  {% endif %}
  {% set margin = (params.MARGIN|float) if params.MARGIN is defined else default_margin %}

  {% set req_len = (line_length|float) %}
  {% if req_len > 100.0 %}{% set req_len = 100.0 %}{% endif %}
  {% set desired_gap = params.FRONT_GAP|default(front_gap)|float %}
  {% set style = params.STYLE|default("CHECK")|upper %}

  # Home if needed
  {% set homed = printer.toolhead.homed_axes %}
  {% if 'x' not in homed or 'y' not in homed or 'z' not in homed %}
    G28
  {% endif %}

  # --- Object-aware Y ---
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

  {% set front_low  = BED_MIN_Y + margin %}
  {% if have_objs and obj_min_y is not none %}
    {% set front_high = obj_min_y - margin %}
    {% if front_high < front_low %}{% set front_high = front_low %}{% endif %}
    {% set gap = front_high - front_low %}
    {% if gap >= desired_gap %}
      {% set target_y = obj_min_y - desired_gap %}
    {% else %}
      {% set target_y = front_low + gap * 0.5 %}
    {% endif %}
  {% else %}
    {% set target_y = front_low + desired_gap %}
    {% if target_y > (BED_MAX_Y - margin) %}{% set target_y = BED_MAX_Y - margin %}{% endif %}
  {% endif %}

  # --- Start X ---
  {% set start_x_abs = BED_MIN_X + margin %}
  {% set avail_w = (BED_MAX_X - margin) - (BED_MIN_X + margin) %}
  {% if avail_w < 0 %}{% set avail_w = 0 %}{% endif %}

  # --- Flow calc ---
  {% set lw = line_w|float %}
  {% set lh = layer_h|float %}
  {% set fd = fil_d|float %}
  {% set flow = flow|float %}
  {% set fil_area = 3.1415926535 * (fd/2.0) * (fd/2.0) %}
  {% if fil_area <= 0 %}{% set fil_area = 2.405 %}{% endif %}
  {% set e_per_mm = (lw * lh * flow) / fil_area %}

  {% set was_abs_coord = printer.gcode_move.absolute_coordinates %}
  {% set is_abs_extrude = printer.gcode_move.absolute_extrude %}

  # --- Anchor ---
  G90
  G92 E0
  G1 Z{line_z} F{travel_f}
  G1 X{start_x_abs} Y{target_y} F{travel_f}

  ; Gentle stationary prime
  {% if is_abs_extrude %}
    G1 E{prime_e} F{prime_f}
    {% set e_total = prime_e|float %}
  {% else %}
    G1 E{prime_e} F{prime_f}
  {% endif %}

  # --- Style switch ---
  {% if style == "DOT" %}
    ; just leave a blob (already primed)
    G4 P500
    {% set dx_check = 0.0 %}
    {% set dy_check = 0.0 %}
  {% else %}
    ; draw check mark
    {% set dx1 = 3.0 %}{% set dy1 = 1.0 %}
    {% set dx2 = 6.0 %}{% set dy2 = 4.0 %}
    {% set len1 = (dx1*dx1 + dy1*dy1) ** 0.5 %}
    {% set len2 = (dx2*dx2 + dy2*dy2) ** 0.5 %}
    {% set e1 = e_per_mm * len1 %}
    {% set e2 = e_per_mm * len2 %}
    G91
    {% if is_abs_extrude %}
      {% set e_total = e_total + e1 %}
      G1 X{dx1} Y{dy1} E{e_total} F{line_f}
      {% set e_total = e_total + e2 %}
      G1 X{dx2} Y{dy2} E{e_total} F{line_f}
    {% else %}
      G1 X{dx1} Y{dy1} E{e1} F{line_f}
      G1 X{dx2} Y{dy2} E{e2} F{line_f}
    {% endif %}
    G1 Y{- (dy1 + dy2)} F{travel_f}
    {% set dx_check = dx1 + dx2 %}
    {% set dy_check = 0.0 %}
  {% endif %}

  # --- Purge line start ---
  {% set line_start_x = start_x_abs + dx_check + 10.0 %}
  {% set max_right = BED_MAX_X - margin %}
  {% if line_start_x < (BED_MIN_X + margin) %}{% set line_start_x = BED_MIN_X + margin %}{% endif %}
  {% if line_start_x > max_right %}{% set line_start_x = max_right %}{% endif %}

  {% set cap_len = req_len %}
  {% set room = max_right - line_start_x %}
  {% if room < 0 %}{% set room = 0 %}{% endif %}
  {% set line_len = cap_len if cap_len <= room else room %}
  {% set e_move = e_per_mm * line_len %}

  G90
  G1 X{line_start_x} Y{target_y} F{travel_f}
  G91
  {% if is_abs_extrude %}
    {% set e_total = e_total + e_move %}
    G1 X{line_len} E{e_total} F{line_f}
  {% else %}
    G1 X{line_len} E{e_move} F{line_f}
  {% endif %}

  G92 E0
  {% if was_abs_coord %} G90 {% else %} G91 {% endif %}

```

>[!NOTE]
>How it works:
><br>
> If there’s ≥ 30 mm between the part’s front edge and the bed front (after margin), it puts the purge exactly 30 mm in front of the part.
>
>If there’s < 30 mm available, it uses the midpoint of the remaining gap so you still stay in front without crowding.
