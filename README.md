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
# description: Front-left purge; STYLE=CHECK|DOT; object-aware front gap; bed-size margins; ≤100 mm line
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
  ##### --- Gather bed limits & size bucket ---
  {% set BED_MIN_X = printer.toolhead.axis_minimum.x|float %}
  {% set BED_MAX_X = printer.toolhead.axis_maximum.x|float %}
  {% set BED_MIN_Y = printer.toolhead.axis_minimum.y|float %}
  {% set BED_MAX_Y = printer.toolhead.axis_maximum.y|float %}
  {% set BED_X = BED_MAX_X - BED_MIN_X %}
  {% set BED_Y = BED_MAX_Y - BED_MIN_Y %}

  {% if BED_X < 275 %}
    {% set default_margin = margin_250|float %}
  {% elif BED_X < 325 %}
    {% set default_margin = margin_300|float %}
  {% else %}
    {% set default_margin = margin_350|float %}
  {% endif %}
  {% set margin = (params.MARGIN|float) if params.MARGIN is defined else default_margin %}

  ##### --- Inputs / caps ---
  {% set req_len = (params.LINE_LEN|default(line_length))|float %}
  {% if req_len > 100.0 %}{% set req_len = 100.0 %}{% endif %}
  {% if req_len < 0.1 %}{% set req_len = 0.1 %}{% endif %}
  {% set desired_gap = params.FRONT_GAP|default(front_gap)|float %}
  {% set style = params.STYLE|default("CHECK")|upper %}

  ##### --- Home if needed ---
  {% set homed = printer.toolhead.homed_axes|string %}
  {% if 'x' not in homed or 'y' not in homed or 'z' not in homed %}
    G28
  {% endif %}

  SAVE_GCODE_STATE NAME=SMART_PURGE_STATE

  ##### --- Compute object-aware Y target ---
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

  ##### --- Start X (front-left) & available width ---
  {% set start_x_abs = BED_MIN_X + margin %}
  {% set max_right = BED_MAX_X - margin %}
  {% set avail_w = max_right - start_x_abs %}
  {% if avail_w < 0 %}{% set avail_w = 0 %}{% endif %}

  ##### --- Flow geometry & safe-capped E/mm (prevents max_extrude_cross_section trips) ---
  {% set lw = (params.LINE_W|default(line_w))|float %}
  {% set lh = (params.LAYER_H|default(layer_h))|float %}
  {% set fd = (params.FIL_D|default(fil_d))|float %}
  {% set flow = (params.FLOW|default(flow))|float %}
  {% set fil_area = 3.1415926535 * (fd/2.0) * (fd/2.0) %}
  {% if fil_area <= 0 %}{% set fil_area = 2.405 %}{% endif %}  ; sane fallback for 1.75
  {% set geom_xsec = lw * lh * flow %}
  {% set max_xsec = (printer.configfile.settings.extruder.max_extrude_cross_section|float) if printer.configfile.settings.extruder is defined and printer.configfile.settings.extruder.max_extrude_cross_section is defined else 0.0 %}
  {% if max_xsec > 0 and geom_xsec > max_xsec %}
    {% set scale = max_xsec / geom_xsec %}
  {% else %}
    {% set scale = 1.0 %}
  {% endif %}
  {% set e_per_mm = (geom_xsec * scale) / fil_area %}
  {% if scale < 0.999 %}
    RESPOND MSG="SMART_PURGE: scaling E by {{'%0.2f' % scale}} to respect max_extrude_cross_section={{'%0.3f' % max_xsec}} mm^2."
  {% endif %}

  ##### --- Z approach (don’t crash if already higher) ---
  {% set curz = printer.gcode_move.gcode_position.z|float %}
  {% set target_z = (params.LINE_Z|default(line_z))|float %}
  {% if target_z < 0.0 %}{% set target_z = 0.0 %}{% endif %}
  {% set z_go = target_z if curz < target_z else curz %}

  ##### --- Prime anchor ---
  G90
  G1 Z{z_go} F{travel_f}
  G1 X{start_x_abs} Y{target_y} F{travel_f}
  G92 E0
  G1 E{prime_e} F{prime_f}

  ##### --- Optional check mark (clamped to bed) ---
  {% if style != "DOT" %}
    ; nominal check vectors
    {% set dx1n, dy1n = 3.0, 1.0 %}
    {% set dx2n, dy2n = 6.0, 4.0 %}
    {% set dx_sum = dx1n + dx2n %}
    {% set edge_room = max_right - start_x_abs %}
    {% set check_scale = 1.0 if edge_room >= (dx_sum + 0.5) else ( (edge_room - 0.5) / dx_sum if (edge_room - 0.5) > 0 else 0.0 ) %}
    ; scaled (never beyond margin)
    {% set dx1, dy1 = dx1n*check_scale, dy1n*check_scale %}
    {% set dx2, dy2 = dx2n*check_scale, dy2n*check_scale %}
    {% set len1 = (dx1*dx1 + dy1*dy1) ** 0.5 %}
    {% set len2 = (dx2*dx2 + dy2*dy2) ** 0.5 %}
    {% set e1 = e_per_mm * len1 %}
    {% set e2 = e_per_mm * len2 %}

    G91
    G1 X{dx1} Y{dy1} E{e1} F{line_f}
    G1 X{dx2} Y{dy2} E{e2} F{line_f}
    ; drop back down vertically to baseline (travel)
    G1 Y{- (dy1 + dy2)} F{travel_f}
    G90
  {% endif %}

  ##### --- Purge line (auto-clamped length) ---
  {% set dx_check = (dx1 + dx2) if style != "DOT" else 0.0 %}
  {% set line_start_x = start_x_abs + dx_check + 10.0 %}
  {% if line_start_x > max_right %}{% set line_start_x = max_right %}{% endif %}
  {% if line_start_x < (BED_MIN_X + margin) %}{% set line_start_x = BED_MIN_X + margin %}{% endif %}
  {% set room = max_right - line_start_x %}
  {% if room < 0.1 %}{% set room = 0.1 %}{% endif %}
  {% set line_len = req_len if req_len <= room else room %}
  {% set e_move = e_per_mm * line_len %}

  G90
  G1 X{line_start_x} Y{target_y} F{travel_f}
  G91
  G1 X{line_len} E{e_move} F{line_f}
  G90
  G92 E0

  RESTORE_GCODE_STATE NAME=SMART_PURGE_STATE

```

>[!NOTE]
>How it works:
><br>
> If there’s ≥ 30 mm between the part’s front edge and the bed front (after margin), it puts the purge exactly 30 mm in front of the part.
>
>If there’s < 30 mm available, it uses the midpoint of the remaining gap so you still stay in front without crowding.
