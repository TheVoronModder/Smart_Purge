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
# description: Front-left purge; keep ~FRONT_GAP from parts (or midpoint if tight); single straight line ≤100mm
# --- Tunables (safe defaults) ---
variable_margin: 5.0                # mm from bed edges
variable_front_gap: 30.0            # desired distance from front of part bbox to purge (mm)
variable_line_z: 0.25               # purge Z height
variable_line_length: 100.0         # requested line length (hard-capped to 100 mm)
variable_travel_f: 6000             # mm/min travel
variable_line_f: 600                # mm/min drawing speed (10 mm/s)
# Gentle stationary prime
variable_prime_e: 10.0              # mm filament for prime
variable_prime_f: 120               # mm/min (2 mm/s)
# Bead geometry for E-per-mm
variable_layer_h: 0.25              # mm
variable_line_w: 0.60               # mm
variable_fil_d: 1.75                # mm
variable_flow: 1.00                 # scalar

gcode:
  # --- Bed coords: (0,0)=front-left ; Y max default 300 unless overridden ---
  {% set BED_MAX_X = (params.BED_MAX_X|default(printer.toolhead.axis_maximum.x)|float) %}
  {% set BED_MAX_Y = (params.BED_MAX_Y|default(300)|float) %}

  {% set margin = margin|float %}
  {% set req_len = (line_length|float) %}
  {% if req_len > 100.0 %}{% set req_len = 100.0 %}{% endif %}  # hard cap 100mm
  {% set desired_gap = params.FRONT_GAP|default(front_gap)|float %}

  # Home if needed (prevents out-of-range)
  {% set homed = printer.toolhead.homed_axes %}
  {% if 'x' not in homed or 'y' not in homed or 'z' not in homed %}
    G28
  {% endif %}

  # --- Find front-most Y of all objects (EXCLUDE_OBJECT) ---
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

  # --- Choose purge Y: target ~30mm in front, else midpoint of available front gap ---
  {% set front_low  = 0.0 + margin %}
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

  # --- X span: start at front-left; clamp length to bed width and 100mm cap ---
  {% set start_x_abs = 0.0 + margin %}
  {% set avail_w = BED_MAX_X - (2*margin) %}
  {% if avail_w < 0 %}{% set avail_w = 0 %}{% endif %}
  {% set line_len = req_len if req_len <= avail_w else avail_w %}

  # --- Geometry-based flow (E per mm of XY travel) ---
  {% set lw = line_w|float %}
  {% set lh = layer_h|float %}
  {% set fd = fil_d|float %}
  {% set flow = flow|float %}
  {% set fil_area = 3.1415926535 * (fd/2.0) * (fd/2.0) %}
  {% if fil_area <= 0 %}{% set fil_area = 2.405 %}{% endif %}
  {% set e_per_mm = (lw * lh * flow) / fil_area %}
  {% set e_move = e_per_mm * line_len %}

  # Respect current modes (we only switch positioning, not extrusion)
  {% set was_abs_coord = printer.gcode_move.absolute_coordinates %}
  {% set is_abs_extrude = printer.gcode_move.absolute_extrude %}

  # --- Anchor & draw (absolute to spot, relative for stroke) ---
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
