# Smart_Purge
Simple Smart Purge for 3d printer ecosystems running Klipper


Macro [gocde_macro SMART_PURGE]

intent:

relative purge line positioning between edge of bed and part print area.


BONUS: You can edit variables and edit them on the fly too!

How to make this work:

in your PRINT_START macro, before printing simply add:

{SMART_PURGE]

then copy / paste below in your macros.cfg


```
[gcode_macro SMART_PURGE]
# description: Front-left purge; front-gap midpoint; relative extrusion inside macro
# --- Tunables ---
variable_extrude_before: 80.0      # mm extruded while stationary
variable_move_extrude: 30.0        # mm extruded per pass while moving
variable_passes: 3                 # number of purge passes
variable_travel_f: 6000            # mm/min travel speed
variable_purge_f: 1200             # mm/min purge line speed
variable_margin: 5.0               # keep this far from edges & parts
variable_backstep: 0.8             # Y offset between passes (mm)
variable_line_z: 0.25              # purge height

gcode:
  {% set BED_MAX_X = (params.BED_MAX_X|default(printer.toolhead.axis_maximum.x)|float) %}
  {% set BED_MAX_Y = (params.BED_MAX_Y|default(300)|float) %}
  {% set margin = margin|float %}

  {% set was_abs = printer.gcode_move.absolute_extrude %}
  M83 ; use relative extrusion

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
    {% set target_y = front_low + (front_high - front_low) * 0.5 %}
  {% else %}
    {% set front_low  = 0.0 + margin %}
    {% set front_high = front_low + 60.0 %}
    {% if front_high > (BED_MAX_Y - margin) %}{% set front_high = BED_MAX_Y - margin %}{% endif %}
    {% set target_y = front_low + (front_high - front_low) * 0.5 %}
  {% endif %}

  {% set start_x = 0.0 + margin %}
  {% set end_x   = BED_MAX_X - margin %}

  ; ---- Purge sequence ----
  G92 E0
  G1 Z{line_z} F{travel_f}
  G1 X{start_x} Y{target_y} F{travel_f}

  ; Stationary prime at 15 mm/s (900 mm/min)
  G1 E{extrude_before} F900

  {% for i in range(passes|int) %}
    {% if (i % 2) == 0 %}
      G1 X{end_x} E{move_extrude} F{purge_f}
    {% else %}
      G1 X{start_x} E{move_extrude} F{purge_f}
    {% endif %}
    {% if i < (passes|int - 1) %}
      {% set next_y = target_y + backstep * (i + 1) %}
      {% if have_objs and obj_min_y is not none and next_y > (obj_min_y - margin) %}
        {% set next_y = obj_min_y - margin %}
      {% endif %}
      G1 Y{next_y} F{travel_f}
    {% endif %}
  {% endfor %}

  G92 E0
  {% if was_abs %}M82{% endif %}
```
