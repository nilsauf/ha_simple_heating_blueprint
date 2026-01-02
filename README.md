# Simple Heating Blueprint for Home Assistant

## Overview

**Simple Heating** is a Home Assistant automation blueprint that implements a **time-based heating controller** with support for:

- Weekly schedules with multiple Comfort time ranges per day
- Comfort / Eco / Off modes
- A seasonal heating-off period (date-based, recurring yearly)
- Manual overrides via `input_select`
- Precise, timer-driven scheduling without polling

The blueprint is designed to be **predictable, deterministic, and extensible**, making it suitable for developers who want a clean and transparent heating control logic.

---

## Key Concepts

### Mode-Based Control

The heating system is controlled indirectly via an `input_select` helper with three states:

- **Comfort** → Heating on with Comfort temperature
- **Eco** → Heating on with Eco temperature
- **Off** → Heating completely disabled

The automation ensures **one-way synchronization**:
- Changes to the `input_select` immediately update the thermostat
- Scheduled logic updates the `input_select`, not the thermostat directly

This separation allows:
- Manual overrides
- External automations or UI controls
- Easy debugging

---

### Timer-Driven Scheduling

Instead of using time or time_pattern triggers, this blueprint uses a **single timer entity**:

- The timer always represents the *next relevant state change*
- When the timer finishes, the schedule is recalculated
- The timer duration is dynamically computed

Advantages:
- No polling
- Minimal triggers
- Exact transitions, even across midnight
- Easy pause/resume behavior

---

## Features

### Weekly Schedule

- Individual schedule per weekday (Monday–Sunday)
- Multiple Comfort time ranges per day
- Outside defined ranges, Eco mode is active
- Empty day schedules default to Eco mode

Example logic:
- 06:00–08:00 → Comfort
- 08:00–17:00 → Eco
- 17:00–22:00 → Comfort
- 22:00–06:00 → Eco

---

### Seasonal Heating-Off Period

- Configurable start and end dates
- **Year is ignored** → period repeats every year
- Correctly handles year-crossing ranges (e.g. Oct–Mar)
- During this period:
  - Mode is forced to `Off`
  - Thermostat HVAC mode is set to `off`
  - Schedule logic is suspended

---

### Manual Overrides

- Changing the `input_select` manually:
  - Immediately updates the thermostat
  - Does **not** break the schedule
- The next timer event will re-evaluate and reapply scheduled logic

---

## Inputs

### Required Entities

| Input | Description |
|-----|------------|
| `thermostat` | Climate entity to control |
| `state_helper` | `input_select` with options: `Comfort`, `Eco`, `Off` |
| `plan_timer` | Timer entity used for scheduling |

### Temperatures

| Input | Description |
|-----|------------|
| `heat_temp` | Target temperature for Comfort mode |
| `eco_temp` | Target temperature for Eco mode |

### Heating-Free Period

| Input | Description |
|-----|------------|
| `off_start` | Start date of heating-off period (MM-DD used) |
| `off_end` | End date of heating-off period (MM-DD used) |

### Weekly Schedules

Each weekday accepts:
- A list of `{ start, end }` time objects
- Zero or more entries
- Multiple Comfort windows per day

---

## Internal Logic Breakdown

### Trigger Types

1. **Timer finished**
   - Recalculate desired mode
   - Schedule next transition

2. **Timer restarted**
   - Same as finished, but avoids feedback loops

3. **State helper changed**
   - Immediate thermostat update
   - No timer recalculation

---

### Decision Flow (Schedule Evaluation)

1. Check if current date is within heating-off period
   - If yes → `Off`
2. Load today's schedule
3. If no entries → `Eco`
4. If current time is within any Comfort range → `Comfort`
5. Otherwise → `Eco`

---

### Next Timer Calculation

Depending on the active mode, the next timer target is:

- **Comfort** → next Comfort end
- **Eco** → next Comfort start
- **Off** → end of heating-off period

The nearest future timestamp is selected, converted into a duration, and scheduled.

---

## Design Decisions

### Why `input_select` Instead of Direct Control?

- Enables manual overrides
- Allows integration with other automations
- Decouples decision logic from hardware control

### Why a Single Timer?

- Prevents race conditions
- No duplicated logic
- Lower system load
- Clear execution points

### Why Ignore the Year in Heating-Off Period?

- Enables recurring seasonal behavior
- Avoids yearly reconfiguration
- Simplifies logic for long-term setups

---

## Limitations & Assumptions

- Thermostat supports:
  - `hvac_mode: heat`
  - `hvac_mode: off`
  - `set_temperature`
- No support for:
  - Cooling
  - Multiple climate entities
  - Per-day temperature overrides
- Time ranges must not overlap (not enforced)