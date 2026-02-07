# Specifikation: Dynamisk strömbegränsare Go-e v5

Detta dokument beskriver exakt vad automationen i `dynamicloadbalance.yaml` ska göra.

## Syfte
- Säkerställa att husets säkring inte överbelastas genom att pausa eller sänka EV‑laddning.
- Vid överbelastning, först försöka **nödsänka** laddströmmen i ett steg innan paus.
- Återuppta laddning när det finns tillräcklig marginal.
- Justera laddström i små steg upp/ner baserat på aktuell husbelastning.

## Ingångar (sensorer/entiteter som läses)
- `sensor.current_phase_1`, `sensor.current_phase_2`, `sensor.current_phase_3` (husets fasströmmar).
- `sensor.go_echarger_407894_car` (bilens tillstånd).
- `select.go_echarger_407894_frc` (FRC‑läge för laddaren).
- `number.go_echarger_407894_amp` (aktuellt inställt laddströmvärde).
- `input_datetime.go_e_last_raise` (tidpunkt för senaste höjning).
- `input_datetime.go_e_overload_start` (tidpunkt då överbelastning upptäcktes).
- `input_boolean.go_e_auto_paused` (markerar att paus gjorts av automationen).

## Utgångar (entiteter som skrivs)
- Sätter FRC‑läge via `select.go_echarger_407894_frc`.
- Sätter laddström via `number.go_echarger_407894_amp`.
- Sätter `input_boolean.go_e_auto_paused` on/off.
- Skriver loggar via `logbook.log`.
- Uppdaterar `input_datetime.go_e_last_raise`.
- Uppdaterar `input_datetime.go_e_overload_start`.

## Konstanter/parametrar
- `fuse_a`: säkringsstorlek i A.
- `safety_margin_a`: marginal innan sänkning börjar.
- `min_ev_a`: minsta tillåtna laddström.
- `max_ev_a`: högsta tillåtna laddström.
- `step_a`: storlek på varje justeringssteg.
- `increase_buffer_a`: extra marginal för att tillåta höjning.
- `resume_buffer_a`: extra marginal för återupptag utöver minsta laddström.
- `resume_cooldown_s`: minsta väntetid innan återupptag får ske (sekunder).
- `increase_rate_limit_s`: minsta tid mellan höjningar.
- `overload_timeout_s`: hur länge nödsänkning får pågå innan paus utlöses (sekunder).

## Beräknade värden
- `house_max_a`: maxvärde av L1/L2/L3.
- `reduce_threshold_a`: `fuse_a - safety_margin_a`.
- `pause_threshold_a`: `fuse_a`.
- `resume_threshold_a`: `fuse_a - min_ev_a - resume_buffer_a`.
- `frc_state`: FRC‑läge i lower‑case.
- `car_state`: bilens tillstånd i lower‑case.
- `is_charging`: true om `car_state == "charging"`.
- `is_connected`: true om `car_state` är `"connected"` eller `"charging"`.
- `current_setpoint_a`: aktuellt inställt laddströmsvärde.
- `seconds_since_last_raise`: sekunder sedan `input_datetime.go_e_last_raise`.
- `seconds_since_auto_pause`: sekunder sedan `input_boolean.go_e_auto_paused` senast slogs på.
- `is_overloaded`: true om `house_max_a_f >= pause_threshold_a`.
- `excess_a`: `max(house_max_a_f - reduce_threshold_a, 0)` — hur mycket vi överstiger sänktröskeln.
- `safe_setpoint_a`: `max(floor(current_setpoint_a - excess_a), min_ev_a)` — högsta säkra ampere‑värde.
- `seconds_since_overload_start`: sekunder sedan `input_datetime.go_e_overload_start`.
- `overload_start_is_stale`: true om `seconds_since_overload_start > overload_timeout_s + 10` eller entiteten saknar giltigt värde.
- `overload_timed_out`: true om överbelastad OCH ej stale OCH `>= overload_timeout_s`.
- `next_setpoint_a_raw`:
  - Om `house_max_a > reduce_threshold_a`: sänk med `step_a`, aldrig under `min_ev_a`.
  - Om `house_max_a < (reduce_threshold_a - increase_buffer_a)`: höj med `step_a`, aldrig över `max_ev_a`.
  - Annars: behåll `current_setpoint_a`.

## Triggers
Automationen ska köras när:
- någon av fasströmmarna ändras och har varit stabil i 3 sekunder, eller
- var 10:e sekund.

## Beteende: paus/nödsänkning/återuppta (först i actions)
### Pausa laddning
Automationen ska pausa laddning om **alla** villkor är uppfyllda:
- `is_connected` är true.
- `is_overloaded` är true.
- `frc_state != "don't charge"`.
- Minst ett av:
  - `safe_setpoint_a < min_ev_a` (sänkning till säker nivå ej möjlig).
  - `overload_timed_out` (överbelastningen har pågått i >= `overload_timeout_s` trots nödsänkning).
  - `current_setpoint_a <= min_ev_a` och `is_overloaded` (redan på lägsta nivå, fortfarande överbelastad).

Åtgärder:
1. Sätt FRC‑läge till `Don't charge`.
2. Sätt `input_boolean.go_e_auto_paused` till on.
3. Logga i logbook att laddning pausas.

### Nödsänkning
Automationen ska nödsänka laddström om **alla** villkor är uppfyllda:
- `is_connected` är true.
- `is_overloaded` är true.
- `frc_state != "don't charge"`.
- `safe_setpoint_a >= min_ev_a` (sänkning till säker nivå är möjlig).
- `current_setpoint_a > min_ev_a` (vi är inte redan på lägsta).
- `overload_timed_out` är false (tidsgränsen ej nådd).

Åtgärder:
1. Sätt laddström till `safe_setpoint_a`.
2. Om `overload_start_is_stale`: sätt `input_datetime.go_e_overload_start` till aktuell tid (ny överbelastningsepisod).
3. Logga i logbook att nödsänkning skett.

### Återuppta laddning
Automationen ska återuppta laddning om **alla** villkor är uppfyllda:
- `frc_state == "don't charge"`.
- `house_max_a <= resume_threshold_a`.
- `input_boolean.go_e_auto_paused` är on.
- `seconds_since_auto_pause >= resume_cooldown_s`.

Åtgärder:
1. Sätt FRC‑läge till `Neutral`.
2. Vänta 3 sekunder.
3. Sätt laddström till `min_ev_a`.
4. Sätt `input_boolean.go_e_auto_paused` till off.
5. Logga i logbook att laddning återupptas.

## Beteende: dynamisk strömjustering (därefter i actions)
### Sänk laddström
Automationen ska sänka laddström om **alla** villkor är uppfyllda:
- `frc_state != "don't charge"`.
- `is_charging` är true.
- `is_overloaded` är false (normal sänkning sker ej vid överbelastning — nödsänkning hanterar det).
- `next_setpoint_a_raw < current_setpoint_a`.

Åtgärder:
1. Sätt laddström till `next_setpoint_a_raw`.
2. Logga i logbook att ström sänkts.

### Höj laddström
Automationen ska höja laddström om **alla** villkor är uppfyllda:
- `frc_state != "don't charge"`.
- `is_charging` är true (höjning sker bara vid aktiv laddning, inte vid t.ex. "complete").
- `next_setpoint_a_raw > current_setpoint_a`.
- `house_max_a <= (reduce_threshold_a - increase_buffer_a)`.
- `seconds_since_last_raise >= increase_rate_limit_s`.

Åtgärder:
1. Sätt laddström till `next_setpoint_a_raw`.
2. Sätt `input_datetime.go_e_last_raise` till aktuell tid.
3. Logga i logbook att ström höjts.

## Loggning
Logbook‑meddelanden ska skrivas vid:
- paus,
- nödsänkning,
- återupptag,
- sänkning,
- höjning.

Alla `logbook.log`‑anrop inkluderar `entity_id: amp_number_entity` så att posterna
kopplas till laddströmsentiteten och kan filtreras i logbook‑kort på dashboards.

## Körläge
- `mode: restart` innebär att en ny trigger avbryter och startar om automationen.

## Begränsningar och avsiktliga val
- Vid återupptag ska laddning alltid starta på `min_ev_a` (inte på en beräknad högre nivå).
- Höjningar rate‑limitas; sänkningar gör det inte.
- Nödsänkning sker i ett enda steg till `safe_setpoint_a` (inte stegvis).
- Om indata är `unknown/unavailable` tolkas de som 0 via `float(0)`.
- Åtgärder utförs endast när alla tre fas‑sensorer har giltiga värden
  (dvs inte `unknown/unavailable/none/''`).

## Implementationsnot (läsbarhet)
Själva YAML‑implementationen använder samlade "guard"‑variabler för villkoren:
- `can_pause`, `can_emergency_reduce`, `can_resume`, `should_decrease`, `should_increase`.
Detta ändrar inte beteendet, men gör villkoren lättare att läsa och återanvända.
