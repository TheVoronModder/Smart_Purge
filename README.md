# Smart_Purge
Simple Smart Purge for 3d printer ecosystems running Klipper


>[!NOTE]
>Macro [gocde_macro SMART_PURGE]
>
>intent:
>
>relative purge line positioning between edge of bed and part print area.



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



>[!IMPORTANT]
>QUICK THINGS:
>
>SMART_PURGE LINE_LENGTH=100 PURGE_F=360
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
# description: Front-left purge; front-gap midpoint; 50mm slow prime + 100mm line; relative positioning
# --- Tunables ---
variable_extrude_before: 50.0      # mm stationary prime (slow)
variable_move_extrude: 30.0        # mm per moving pass
variable_passes: 3                 # number of purge passes (zig-zag lanes)
variable_travel_f: 6000            # mm/min travel
variable_purge_f: 1200             # mm/min moving purge speed
variable_margin: 5.0               # mm from edges/parts
variable_backstep: 0.8             # mm Y offset between lanes
variable_line_z: 0.25              # Z height for purge
variable_line_length: 100.0        # mm line length (moving part)

gcode:
  # Bed coords: (0,0)=front-left ; user Y max = 300 (rear endstop)
  {% set BED_MAX_X = (params.BED_MAX_X|default(printer.toolhead.axis_maximum.x)|float) %}
  {% set BED_MAX_Y = (params.BED_MAX_Y|default(300)|float) %}
  {% set margin = margin|float %}
  {% set line_len = line_length|float %}
  {% set backstep = backstep|float %}
  {% set passes = passes|int %}

  # Figure front-most part Y from EXCLUDE_OBJECT polygons (if present)
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

  # Compute front gap and target_y (midpoint of gap)
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

  # X span: start at front-left margin, line length capped to available width
  {% set start_x_abs = 0.0 + margin %}
  {% set avail_w = BED_MAX_X - (2*margin) %}
  {% if line_len > avail_w %}{% set line_len = avail_w %}{% endif %}

  # Limit lane count so Y steps don't cross into parts
  {% set max_delta = front_high - target_y %}
  {% if backstep <= 0 %}{% set backstep = 0.8 %}{% endif %}
  {% set max_lanes = 1 + (max_delta // backstep)|int %}
  {% if passes > max_lanes %}{% set passes = max_lanes %}{% endif %}
  {% if passes < 1 %}{% set passes = 1 %}{% endif %}

  # Detect current modes so we can restore later and handle E correctly
  {% set was_abs_coord = printer.gcode_move.absolute_coordinates %}
  {% set is_abs_extrude = printer.gcode_move.absolute_extrude %}

  # --- Move to anchor in ABSOLUTE coords ---
  G90
  G92 E0
  G1 Z{line_z} F{travel_f}
  G1 X{start_x_abs} Y{target_y} F{travel_f}

  # --- Stationary prime (slow = 6 mm/s -> F360) ---
  {% if is_abs_extrude %}
    ; absolute E: grow the target
    G1 E{extrude_before} F360
    {% set e_total = extrude_before|float %}
  {% else %}
    ; relative E
    G1 E{extrude_before} F360
  {% endif %}

  # --- Purge passes in RELATIVE POSITIONING (G91) ---
  G91

  {% for i in range(passes) %}
    {% if (i % 2) == 0 %}
      ; forward X line
      {% if is_abs_extrude %}
        {% set e_total = e_total + (move_extrude|float) %}
        G1 X{line_len} E{e_total} F{purge_f}
      {% else %}
        G1 X{line_len} E{move_extrude} F{purge_f}
      {% endif %}
    {% else %}
      ; back X line
      {% if is_abs_extrude %}
        {% set e_total = e_total + (move_extrude|float) %}
        G1 X{-line_len} E{e_total} F{purge_f}
      {% else %}
        G1 X{-line_len} E{move_extrude} F{purge_f}
      {% endif %}
    {% endif %}

    {% if i < (passes - 1) %}
      ; step back in Y, but clamp final step if needed
      {% set desired = backstep %}
      {% set remaining = max_delta - (backstep * (i + 1)) %}
      {% if remaining < 0 %}{% set desired = backstep + remaining %}{% endif %}
      G1 Y{desired} F{travel_f}
    {% endif %}
  {% endfor %}

  ; tidy up
  G92 E0

  ; restore coord mode
  {% if was_abs_coord %}
    G90
  {% else %}
    G91
  {% endif %}
```
