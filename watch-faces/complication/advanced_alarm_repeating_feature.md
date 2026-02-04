# Repeating Alarm Implementation Plan

## Overview

Add a "repeat until acknowledged" mode for alarms. When enabled, an alarm will re-fire every 5 minutes until the user navigates to the alarm face and explicitly dismisses it.

## Design Decisions

1. **Dismissal as a distinct UI mode**: When pending alarms exist, the alarm face shows a "pending dismissal" screen instead of the normal view
2. **Settings blocked while pending**: User must acknowledge pending alarms before accessing settings
3. **Dismiss cancels repeat cycle**: Dismissing stops repeats until next scheduled time (e.g., tomorrow)
4. **Single pending alarm**: Only one alarm can be in pending-repeat state at a time (simplifies implementation)

---

## Phase 1: Data Structure Changes

**File: `watch-faces/complication/advanced_alarm_face.h`**

### 1.1 Add repeat flag to alarm settings

```c
typedef struct {
    uint8_t day : 4;
    uint8_t hour : 5;
    uint8_t minute : 6;
    uint8_t beeps : 4;
    uint8_t pitch : 2;
    bool enabled : 1;
    bool repeat : 1;    // NEW: repeat every 5 min until acknowledged
} alarm_setting_t;
```

### 1.2 Add pending alarm tracking to state

```c
typedef struct {
    uint8_t alarm_idx : 4;
    uint8_t alarm_playing_idx : 4;
    uint8_t setting_state : 3;
    int8_t alarm_handled_minute;
    bool alarm_quick_ticks : 1;
    bool is_setting : 1;
    // NEW fields for repeat functionality:
    bool has_pending : 1;              // is there a pending repeat?
    uint8_t pending_alarm_idx : 4;     // which alarm is pending
    uint8_t pending_target_minute : 6; // next repeat fires at this minute
    uint8_t pending_target_hour : 5;   // next repeat fires at this hour
    alarm_setting_t alarm[ALARM_ALARMS];
} alarm_state_t;
```

### 1.3 Update constants

```c
#define ALARM_SETTING_STATES 7  // was 6, now includes repeat setting
```

### 1.4 Add new setting index

In `.c` file, add to `alarm_setting_idx_t` enum:

```c
typedef enum {
    alarm_setting_idx_alarm,
    alarm_setting_idx_day,
    alarm_setting_idx_hour,
    alarm_setting_idx_minute,
    alarm_setting_idx_pitch,
    alarm_setting_idx_beeps,
    alarm_setting_idx_repeat  // NEW
} alarm_setting_idx_t;
```

---

## Phase 2: Core Repeat Logic

**File: `watch-faces/complication/advanced_alarm_face.c`**

### 2.1 Helper function to calculate +5 minutes

```c
static void _alarm_schedule_next_repeat(alarm_state_t *state, uint8_t hour, uint8_t minute) {
    minute += 5;
    if (minute >= 60) {
        minute -= 60;
        hour = (hour + 1) % 24;
    }
    state->pending_target_hour = hour;
    state->pending_target_minute = minute;
}
```

### 2.2 Modify `advanced_alarm_face_advise()`

After existing alarm checking (around line 300), add repeat checking:

```c
// Check for pending repeat alarm
if (state->has_pending) {
    if (now.unit.hour == state->pending_target_hour &&
        now.unit.minute == state->pending_target_minute) {
        state->alarm_playing_idx = state->pending_alarm_idx;
        // Schedule next repeat
        _alarm_schedule_next_repeat(state, now.unit.hour, now.unit.minute);
        retval.wants_background_task = true;
    }
}
```

Also, in the alarm checking loop, skip alarms that are already pending:

```c
for (uint8_t i = 0; i < ALARM_ALARMS; i++) {
    // Skip if this alarm is already in pending repeat state
    if (state->has_pending && state->pending_alarm_idx == i) {
        continue;
    }
    if (state->alarm[i].enabled) {
        // ... existing alarm checking code ...
    }
}
```

### 2.3 Modify `EVENT_BACKGROUND_TASK` handler

After playing alarm (around line 450), if repeat is enabled, set up pending state:

```c
case EVENT_BACKGROUND_TASK:
    // play alarm (existing code)
    if (state->alarm[state->alarm_playing_idx].beeps == 0) {
        _alarm_play_short_beep(state->alarm[state->alarm_playing_idx].pitch);
    } else {
        movement_play_alarm_beeps(...);
    }

    // NEW: If repeat enabled, schedule next repeat
    if (state->alarm[state->alarm_playing_idx].repeat) {
        watch_date_time_t now = movement_get_local_date_time();
        state->has_pending = true;
        state->pending_alarm_idx = state->alarm_playing_idx;
        _alarm_schedule_next_repeat(state, now.unit.hour, now.unit.minute);
    }

    // one time alarm handling - only if NOT repeating
    // (move cleanup to dismissal for repeat alarms)
    if (state->alarm[state->alarm_playing_idx].day == ALARM_DAY_ONE_TIME &&
        !state->alarm[state->alarm_playing_idx].repeat) {
        // ... existing one-time cleanup code ...
    }
    break;
```

### 2.4 Add dismissal function

```c
static void _alarm_dismiss_pending(alarm_state_t *state) {
    // If this was a one-time alarm, erase it now
    if (state->alarm[state->pending_alarm_idx].day == ALARM_DAY_ONE_TIME) {
        state->alarm[state->pending_alarm_idx].day = ALARM_DAY_EACH_DAY;
        state->alarm[state->pending_alarm_idx].minute = 0;
        state->alarm[state->pending_alarm_idx].hour = 0;
        state->alarm[state->pending_alarm_idx].beeps = 5;
        state->alarm[state->pending_alarm_idx].pitch = 1;
        state->alarm[state->pending_alarm_idx].repeat = false;
        state->alarm[state->pending_alarm_idx].enabled = false;
        _alarm_update_alarm_enabled(state);
    }

    state->has_pending = false;
    state->pending_alarm_idx = 0;
    state->pending_target_hour = 0;
    state->pending_target_minute = 0;
}
```

---

## Phase 3: Pending Dismissal UI Mode

### 3.1 New draw function for pending mode

```c
static void _alarm_draw_pending_dismissal(alarm_state_t *state, uint8_t subsecond) {
    char buf[12];
    bool set_leading_zero = movement_clock_mode_24h() == MOVEMENT_CLOCK_MODE_024H;

    // Show "AC" (acknowledge) blinking in top-left
    if (subsecond % 2) {
        watch_display_text_with_fallback(WATCH_POSITION_TOP_LEFT, "AC ", "AC");
    } else {
        watch_display_text_with_fallback(WATCH_POSITION_TOP_LEFT, "   ", "  ");
    }

    // Show pending alarm number in top-right
    sprintf(buf, "%2d", state->pending_alarm_idx + 1);
    watch_display_text(WATCH_POSITION_TOP_RIGHT, buf);

    // Show pending alarm time
    uint8_t h = state->alarm[state->pending_alarm_idx].hour;
    if (!movement_clock_mode_24h()) {
        if (h >= 12) {
            watch_set_indicator(WATCH_INDICATOR_PM);
            h %= 12;
        } else {
            watch_clear_indicator(WATCH_INDICATOR_PM);
        }
        if (h == 0) h = 12;
    } else {
        watch_set_indicator(WATCH_INDICATOR_24H);
    }

    sprintf(buf, set_leading_zero ? "%02d" : "%2d", h);
    watch_display_text(WATCH_POSITION_HOURS, buf);
    sprintf(buf, "%02d", state->alarm[state->pending_alarm_idx].minute);
    watch_display_text(WATCH_POSITION_MINUTES, buf);

    // Show "rP" in seconds to indicate repeat pending
    watch_display_text(WATCH_POSITION_SECONDS, "rP");

    // Blink signal indicator
    if (subsecond % 2) {
        watch_set_indicator(WATCH_INDICATOR_SIGNAL);
    } else {
        watch_clear_indicator(WATCH_INDICATOR_SIGNAL);
    }
}
```

### 3.2 Modify `EVENT_ACTIVATE` handling

```c
case EVENT_ACTIVATE:
    if (state->has_pending) {
        movement_request_tick_frequency(2);  // for blinking
        _alarm_draw_pending_dismissal(state, event.subsecond);
    } else {
        _advanced_alarm_face_draw(state, event.subsecond);
    }
    break;
```

### 3.3 Modify `EVENT_TICK` handling

At the beginning of the EVENT_TICK case, add pending mode handling:

```c
case EVENT_TICK:
    if (state->has_pending && !state->is_setting) {
        _alarm_draw_pending_dismissal(state, event.subsecond);
        break;
    }
    // ... existing tick handling ...
```

### 3.4 Modify button handlers for pending mode

**`EVENT_LIGHT_BUTTON_UP`** (around line 346):

```c
case EVENT_LIGHT_BUTTON_UP:
    if (state->has_pending) {
        // Block settings entry while pending - do nothing
        // Optionally: brief LED flash to indicate blocked
        break;
    }
    // ... existing settings logic ...
```

**`EVENT_ALARM_BUTTON_UP`** (around line 365):

```c
case EVENT_ALARM_BUTTON_UP:
    if (state->has_pending) {
        // Dismiss the pending alarm
        _alarm_dismiss_pending(state);
        movement_request_tick_frequency(1);
        _advanced_alarm_face_draw(state, event.subsecond);
        break;
    }
    // ... existing alarm button logic ...
```

**`EVENT_LIGHT_LONG_PRESS`** (around line 358):

```c
case EVENT_LIGHT_LONG_PRESS:
    if (state->has_pending) {
        // Block settings entry while pending
        break;
    }
    // ... existing logic ...
```

---

## Phase 4: Settings UI for Repeat Toggle

### 4.1 Add repeat indicator display in settings mode

In `_advanced_alarm_face_draw()`, after the beeps indicator drawing (around line 140), add:

```c
// draw repeat indicator (in settings mode)
if (state->is_setting) {
    // ... existing pitch and beeps drawing ...

    // draw repeat indicator - reuse a display position or add to seconds
    // Show "rP" when repeat enabled, "--" when disabled
    // Only blink when actively setting repeat
    if ((subsecond % 2) == 0 || (state->setting_state != alarm_setting_idx_repeat)) {
        if (state->alarm[state->alarm_idx].repeat) {
            // Show repeat indicator somehow - perhaps use a segment
            // or incorporate into the display
        }
    }
}
```

### 4.2 Handle repeat setting in `EVENT_ALARM_BUTTON_UP`

In the settings switch statement (around line 398), add:

```c
case alarm_setting_idx_repeat:
    // toggle repeat
    state->alarm[state->alarm_idx].repeat ^= 1;
    break;
```

### 4.3 Show repeat indicator in normal view

In `_alarm_show_alarm_on_text()` or the non-setting display section, show "rP" for alarms with repeat enabled:

```c
static void _alarm_show_alarm_on_text(alarm_state_t *state) {
    if (state->alarm[state->alarm_idx].enabled) {
        if (state->alarm[state->alarm_idx].repeat) {
            watch_display_text(WATCH_POSITION_SECONDS, "rP");
        } else {
            watch_display_text(WATCH_POSITION_SECONDS, "on");
        }
    } else {
        watch_display_text(WATCH_POSITION_SECONDS, "--");
    }
}
```

---

## Phase 5: Edge Cases & Cleanup

### 5.1 Resign behavior

In `advanced_alarm_face_resign()`, do NOT clear pending state - the alarm should keep repeating until acknowledged:

```c
void advanced_alarm_face_resign(void *context) {
    alarm_state_t *state = (alarm_state_t *)context;
    state->is_setting = false;
    _alarm_update_alarm_enabled(state);
    watch_set_led_off();
    state->alarm_quick_ticks = false;
    _wait_ticks = -1;
    // Note: do NOT clear has_pending - repeat should persist across face changes
    movement_request_tick_frequency(1);
}
```

### 5.2 Setup initialization

In `advanced_alarm_face_setup()`, initialize new fields:

```c
state->has_pending = false;
state->pending_alarm_idx = 0;
state->pending_target_hour = 0;
state->pending_target_minute = 0;

// In alarm initialization loop:
for (uint8_t i = 0; i < ALARM_ALARMS; i++) {
    state->alarm[i].day = ALARM_DAY_EACH_DAY;
    state->alarm[i].beeps = 5;
    state->alarm[i].pitch = 1;
    state->alarm[i].repeat = false;  // NEW
}
```

### 5.3 Handle alarm_handled_minute with repeats

The `alarm_handled_minute` failsafe prevents double-firing within the same minute. For repeats, this should work correctly since repeats are 5 minutes apart. However, verify that the repeat firing also sets this correctly.

---

## Implementation Order

1. **Phase 1** - Data structures (foundation)
2. **Phase 4** - Settings UI for repeat toggle (allows testing the flag)
3. **Phase 2** - Core repeat logic (the meat of the feature)
4. **Phase 3** - Pending dismissal UI (makes it usable)
5. **Phase 5** - Edge cases (polish)

---

## Testing Checklist

- [ ] Alarm with repeat=off fires once, no pending state
- [ ] Alarm with repeat=on fires, then fires again 5 min later
- [ ] Navigating to alarm face shows pending dismissal UI
- [ ] ALARM button dismisses pending alarm
- [ ] After dismissal, normal view resumes
- [ ] Cannot enter settings while pending alarm exists
- [ ] One-time alarm with repeat gets erased on dismissal, not first fire
- [ ] Repeat setting appears in settings cycle and toggles correctly
- [ ] Normal view shows "rP" for enabled+repeat alarms
- [ ] Midnight rollover (e.g., 23:57 alarm repeats to 00:02)
- [ ] Second alarm firing while first is pending (takes over pending state)
- [ ] Pending state persists when navigating away from alarm face
- [ ] Pending state persists through sleep/wake cycles

---

## Estimated Effort

| Component | Effort |
|-----------|--------|
| Data structure changes | ~2 hrs |
| Settings UI for repeat toggle | ~2-3 hrs |
| Core repeat scheduling logic | ~4-5 hrs |
| Pending dismissal UI mode | ~4-5 hrs |
| Edge cases & testing | ~3-4 hrs |
| **Total** | **~16-20 hrs** |

---

## Open Questions / Future Considerations

1. **Visual feedback when settings blocked**: Should there be a beep or LED flash when user tries to enter settings with pending alarm?

2. **Multiple pending alarms**: Current design only tracks one. If alarm B fires while A is pending, B takes over. Is this acceptable?

3. **Configurable repeat interval**: Currently hardcoded to 5 minutes. Could be made configurable per-alarm, but adds UI complexity.

4. **Maximum repeats**: Should there be a limit (e.g., stop after 1 hour)? Current design repeats indefinitely until dismissed.
