# Smart_Purge
Simple Smart Purge for 3d printer ecosystems running Klipper


Macro [gocde_macro SMART_PURGE]

intent:

relative purge line positioning between edge of bed and part print area.


BONUS: You can edit variables and edit them on the fly too!


```
[gcode_macro SMART_PURGE]
description: Purge line placed ~50% away from parts; slow & heavy purge
# Tunables
variable_extrude_before: 80.0
variable_move_extrude: 30.0
variable_passes: 3
variable_travel_f: 6000
variable_purge_f: 1200
variable_margin: 5.0
variable_backstep: 0.8
variable_line_z: 0.25

gcode:
  {% set tool = printer.toolhead %}
  {% set xmin = tool.axis_minimum.x|float %}
  {% set xmax = tool.axis_maximum.x|float %}
  {% set ymin = tool.axis_minimum.y|float %}
  {% set ymax = tool.axis_maximum.y|float %}

  {% set have_objs = printer.exclude_object is defined and printer.exclude_object.objects|length > 0 %}
  {% if have_objs %}
    {% set obj_min_x = None %}
    {% set obj_max_x = None %}
    {% set obj_min_y = None %}
    {% set obj_max_y = None %}

    {% for o in printer.exclude_object.objects %}
      {% if o.polygon is defined and o.polygon|length > 0 %}
        {% for pt in o.polygon %}
          {% set x = pt[0]|float %}
          {% set y = pt[1]|float %}
          {% set obj_min_x = x if obj_min_x is none or x < obj_min_x else obj_min_x %}
          {% set obj_max_x = x if obj_max_x is none or x > obj_max_x else obj_max_x %}
          {% set obj_min_y = y if obj_min_y is none or y < obj_min_y else obj_min_y %}
          {% set obj_max_y = y if obj_max_y is none or y > obj_max_y else obj_max_y %}
        {% endfor %}
      {% endif %}
    {% endfor %}

    {% if obj_min_y is not none and obj_max_y is not none %}
      {% set free_front = (obj_min_y - ymin)|float %}
      {% set free_back  = (ymax - obj_max_y)|float %}

      {% if free_front >= free_back %}
        {% set target_y = ymin + free_front * 0.5 %}
        {% set low_y = ymin + margin %}
        {% set high_y = obj_min_y - margin %}
      {% else %}
        {% set target_y = ymax - free_back * 0.5 %}
        {% set low_y = obj_max_y + margin %}
        {% set high_y = ymax - margin %}
      {% endif %}

      {% if target_y < low_y %}{% set target_y = low_y %}{% endif %}
      {% if target_y > high_y %}{% set target_y = high_y %}{% endif %}

      {% set start_x = xmin + margin %}
      {% set end_x   = xmax - margin %}
    {% else %}
      {% set target_y = ymin + 60.0 %}
      {% set start_x  = xmin + 50.0 %}
      {% set end_x    = start_x + 150.0 %}
    {% endif %}
  {% else %}
    {% set target_y = ymin + 60.0 %}
    {% set start_x  = xmin + 50.0 %}
    {% set end_x    = start_x + 150.0 %}
  {% endif %}

  # Purge sequence
  G92 E0
  G1 Z{line_z} F{travel_f}
  G1 X{start_x} Y{target_y} F{travel_f}

  # Stationary prime
  G1 E{extrude_before} F{purge_f}

  # Multi-pass zig-zag
  {% for i in range(passes|int) %}
    {% if (i % 2) == 0 %}
      G1 X{end_x} E{move_extrude} F{purge_f}
    {% else %}
      G1 X{start_x} E{move_extrude} F{purge_f}
    {% endif %}
    {% if i < (passes|int - 1) %}
      G1 Y{target_y + backstep * (i + 1)} F{travel_f}
    {% endif %}
  {% endfor %}

  G92 E0
```
