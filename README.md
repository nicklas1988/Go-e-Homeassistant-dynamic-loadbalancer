# Dynamisk strömbegränsare för go-eCharger (Home Assistant)

Den här automationen pausar och justerar laddströmmen för en go-eCharger baserat på
husets fasströmmar. Den är gjord för att undvika att huvudsäkringen löser ut.

Se `SPEC.md` för exakt beteende.

## Förutsättningar
- Home Assistant med go-eCharger‑integration.
- Sensorer för husets fasströmmar (A) på tre faser.
- Entiteter för laddaren:
  - FRC‑läge (select)
  - Laddström (number)
  - Bilens status (sensor)
- `input_boolean` för auto‑paus.
- `input_datetime` för senaste höjning.

## Snabbstart
1. Kopiera `dynamicloadbalance.example.yaml` till `dynamicloadbalance.yaml`.
2. Importera `dynamicloadbalance.yaml` i dina automationer (eller använd packages).
3. Uppdatera entitets‑ID:n i variablerna så att de matchar din installation.
4. Skapa nödvändiga `input_*`‑entiteter (se lista nedan).
5. Justera parametrar som `fuse_a`, `min_ev_a`, `max_ev_a`, `resume_buffer_a`.
6. Ladda om automationer i Home Assistant.

## Entiteter du behöver
Uppdatera variablerna i YAML:
- `phase_1_sensor`, `phase_2_sensor`, `phase_3_sensor`
- `amp_number_entity`
- `frc_select_entity`
- `car_state_sensor`
- `auto_pause_entity` (input_boolean)
- `last_raise_entity` (input_datetime)

Skapa manuellt i HA om de saknas:
- `input_boolean.go_e_auto_paused`
- `input_datetime.go_e_last_raise`

## Rekommenderade parametrar
- `fuse_a`: storlek på huvudsäkring (A).
- `safety_margin_a`: marginal innan sänkning (A).
- `min_ev_a` / `max_ev_a`: laddströmintervall (A).
- `resume_buffer_a`: extra buffert för återupptag (A).
- `resume_cooldown_s`: minsta väntetid före återupptag (s).
- `increase_buffer_a`: extra marginal för höjning (A).
- `increase_rate_limit_s`: minsta tid mellan höjningar (s).
