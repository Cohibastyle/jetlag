# Functional Specification Document (FSD)

## Product Name
Jet Lag Recovery System (JL-01)

## Document Purpose
Define the functional requirements for a single-page, HTML-based experience that helps a traveler generate a personalized jet lag recovery timeline based on:
- Starting city or airport code
- Destination city or airport code
- User-selected recovery aggressiveness

The page updates a destination-time sleep schedule table in real time as the user changes inputs. The aggressiveness slider also drives a live animated sky background that reflects the selected recovery intensity.

## Scope
In scope:
- One responsive web page built with HTML, CSS, and JavaScript (no build tools or external dependencies).
- Searchable input controls for origin and destination with a bundled airport dataset (~340 airports worldwide).
- Instant search suggestions while typing for airport and city choices.
- Continuous aggressiveness slider (range 1.0-5.0, step 0.04) that influences adaptation speed and drives a live sky animation.
- Real-time generation of a destination-time sleep schedule table.
- Flight Analysis summary panel (raw offset, effective offset, adaptation days, destination timezone).
- Cinematic cockpit/instrument-panel visual design with glassmorphism panels and HUD corner brackets.
- Basic validation and error states.

Out of scope:
- Flight booking or airline integration.
- User accounts and authentication.
- Medical diagnosis or treatment claims.
- Offline mode and native mobile apps.
- 24-hour time display toggle (future enhancement).
- Date picker for future/past trips (future enhancement).

## Target Users
- Travelers preparing for international or cross-time-zone trips.
- Users who want a practical sleep adjustment plan in destination local time.

## User Goals
- Quickly find and select origin and destination airports or cities.
- Tune recovery style from conservative to aggressive.
- Instantly see a day-by-day sleep schedule in destination time.

## Assumptions
- The airport/city dataset is bundled directly in the HTML file (no external API required).
- Time zone offsets, including DST, are resolved via the browser's Intl.DateTimeFormat API using IANA timezone strings.
- Offsets are computed for the current calendar date; DST transitions that occur within the adaptation window are not modelled per-day.
- Default bedtime is 23:00 and default wake time is 07:00.

## Primary User Stories
1. As a traveler, I want to type part of an airport code or city and get suggestions immediately so I can select locations quickly.
2. As a traveler, I want to set how aggressive my recovery plan should be so the timeline matches my comfort level.
3. As a traveler, I want the sleep table to update instantly when I change any input so I can compare options in real time.
4. As a traveler, I want all times shown in destination local time so I can follow the schedule after arrival.
5. As a traveler, I want to see how many days it will take to adapt based on my aggressiveness choice so I can plan accordingly.
6. As a traveler, I want to see clear validation messages if I enter invalid input so I can correct it and get accurate results.
7. As a traveler, I want to use the page on my mobile device with the same functionality as desktop so I can plan on the go.
8. As a traveler, I want to understand the meaning of the aggressiveness levels so I can choose the right one for me.
9. As a traveler, I want to see a clear "No matches" message if my search input does not match any locations so I know to adjust my query.
10. As a traveler, I want to see the time zone difference between origin and destination so I can understand the scale of adjustment needed.

## Functional Requirements

### FR-1: Page Layout
The page is composed of five visual zones:
1. Header panel - product name, subtitle (JL-01), and a live weather condition badge.
2. Controls panel (left of main grid on desktop) - origin, destination, bedtime, wake time, and aggressiveness slider.
3. Flight Analysis panel (right of main grid on desktop) - four stat tiles: raw TZ offset, effective adjustment, adaptation days, destination timezone.
4. Destination-Time Sleep Plan panel - schedule table or placeholder empty state.
5. Fixed sky canvas behind all content - animated weather background driven by the aggressiveness slider.

Layout breakpoints:
- >= 861 px: controls and summary side-by-side (1.15fr : 1fr grid).
- <= 860 px: controls and summary stack vertically.
- <= 600 px: compact mobile layout - reduced padding, hidden subtitle, origin and destination inputs span full width, controls and summary fit above fold on a 375x812 viewport.

### FR-2: Origin and Destination Search Inputs
- Each input supports free typing with instant suggestion results.
- Suggestion matching rules:
  - Match against city name, airport name, IATA code, and country name.
  - Case-insensitive.
  - Starts-with matches ranked first, then contains matches.
- Suggestion list behavior:
  - Renders on every keystroke with no artificial debounce delay.
  - Shows up to 8 results.
  - Supports keyboard navigation: ArrowDown / ArrowUp moves highlight, Enter selects, Escape closes.
  - Mouse click selects an item.
  - Closes on blur (120 ms timeout to allow click events to fire first).
- On selection, the input field is populated in the format: CODE - City (Airport Name).
- A teal pill chip below the input shows: CODE | City, Country.
- If no results found, the list shows "No matches".
- Clearing the input clears the selection and hides the chip.

### FR-3: Input Validation
All validation runs on every state change before schedule generation:
- Origin and destination must both be selected from the suggestion list.
- Origin and destination cannot be the same IATA code.
- Bedtime and wake time must be valid HH:MM clock values (0-23 hours, 0-59 minutes).
- Bedtime and wake time cannot be identical (zero-length sleep window).
- If any validation fails: the error banner is shown, the table is hidden, and the Flight Analysis tiles show dash.
- The error banner hides itself via CSS :empty when the error string is cleared.

### FR-4: Aggressiveness Slider
- Continuous range: 1.0 to 5.0, step 0.04.
- Default value: 3.0.
- Named breakpoints derived from the continuous value:
  - 1.0-1.5 => Gentle
  - 1.5-2.5 => Moderate-Gentle
  - 2.5-3.5 => Moderate
  - 3.5-4.5 => Moderate-Aggressive
  - 4.5-5.0 => Aggressive
- Label format: "Level X.X - <Name> - Y.Y h/day"
- The slider thumb is styled as an amber radial-gradient sphere with a drop-shadow glow.
- The filled track left of the thumb updates color on every animation frame: teal (levels 1-2.5), amber (levels 2.5-3.8), red (levels 3.8-5).
- Slider changes trigger immediate schedule recalculation and sky animation update.

### FR-5: Timeline and Sleep Schedule Generation

Inputs:
- Origin IANA timezone string.
- Destination IANA timezone string.
- User normal bedtime (minutes from midnight).
- User normal wake time (minutes from midnight).
- Aggressiveness level (continuous 1.0-5.0).

Timezone offset resolution:
- Both origin and destination UTC offsets are computed via Intl.DateTimeFormat using the current date.
- The offset is sampled at local noon of the current calendar day in each timezone to avoid DST transition edge cases (transitions always occur near 2 AM).
- DST is correctly reflected for today's date. Transitions that fall within the adaptation window are not applied per-day.

Offset calculation:
- rawOffsetHours = destinationOffsetFromUTC minus originOffsetFromUTC
- effectiveOffsetHours = shortestOffset(rawOffsetHours) where:
  - effective = ((raw + 12) % 24 + 24) % 24 - 12
  - If effective == -12 and raw > 0, use +12 instead.
  - This constrains the adjustment to [-12, +12], always taking the shorter direction around the clock.

Adaptation rate (continuous linear interpolation):
- shiftRate = 0.5 + (level - 1) x 0.5 hours per day.
- Level 1 => 0.5 h/day, Level 3 => 1.5 h/day, Level 5 => 2.5 h/day.

Adaptation days:
- days = ceiling(|effectiveOffsetHours| / shiftRate), minimum 1 row shown.

Schedule baseline:
- On arrival the traveler's body clock still reads home time.
- arrivalBedtime in destination local time = userBedtime + effectiveOffsetHours x 60 (minutes).
- arrivalWakeTime similarly offset.

Per-day entries (day d, 1-indexed):
- cumulative = sign(effectiveOffset) x min(|effectiveOffset|, shiftRate x d) hours.
- bedLocal = arrivalBedtime - cumulative x 60 minutes.
- wakeLocal = arrivalWakeTime - cumulative x 60 minutes.
- Note: "Shift sleep Xh earlier" / "Shift sleep Xh later" / "No shift needed".
- Times displayed in 12-hour AM/PM format.

Table columns:
- # - day number (1, 2, 3...)
- Bedtime - 12-hour AM/PM, white-space: nowrap, fixed 7rem wide.
- Wake Time - 12-hour AM/PM, white-space: nowrap, fixed 7rem wide.
- Shift Note - variable width, max-width: 160px.

### FR-6: Real-Time Update Rules
The page recalculates and rerenders on every change to:
- Origin selection (including clearing)
- Destination selection (including clearing)
- Bedtime input
- Wake time input
- Aggressiveness slider (fires on both input and change events)

### FR-7: Flight Analysis Panel
Four stat tiles in a 2x2 grid:
- TZ Difference (raw) - rawOffsetHours formatted as +/-Xh (one decimal).
- Adjustment Used - effectiveOffsetHours formatted as +/-Xh (one decimal).
- Adaptation Days - integer count.
- Destination TZ - IANA timezone string (e.g. Asia/Tokyo), small font with word-break.

All tiles show dash when inputs are invalid or incomplete.

### FR-8: Weather Sky Animation
A canvas element fixed behind all content animates continuously based on the aggressiveness slider, normalized to t = (value - 1) / 4 (range 0-1).

Sky gradient - top and bottom colours interpolate through 5 stops as t increases:
- t = 0.00: bright blue sky
- t = 0.25: partly cloudy
- t = 0.50: overcast grey
- t = 0.75: storm
- t = 1.00: deep storm / near-black

Sun - visible when t < 0.38; fades out linearly. Two stacked radial gradients (glow halo + disc) at 75% width, 14% height.

Clouds - 22 objects, each drawn as 7 overlapping ellipses with radial gradient softness (no ctx.filter blur for performance). Cloud brightness lerps 248 to 68 (white to dark grey). Coverage and wind speed scale up with t.

Rain - 320 line-segment drops. Fades in at t > 0.44, full density at t = 0.72. Drop speed and wind angle increase with t.

Lightning - activates at t > 0.76. Branching bolt as a random walk. Flash interval: ~550 frames (t = 0.76) to ~70 frames (t = 1.0). Each bolt fades over ~18 frames with a blue screen-flash at peak.

Weather badge thresholds:
- t 0.00-0.12: CLEAR
- t 0.12-0.32: FEW CLOUDS
- t 0.32-0.52: OVERCAST
- t 0.52-0.72: RAIN
- t 0.72-0.88: HEAVY RAIN
- t 0.88-1.00: THUNDERSTORM

At t > 0.65 the badge switches to red (.stormy class).

### FR-9: Accessibility and Usability
- Origin and destination inputs use role="combobox" with aria-expanded and aria-controls pointing to the suggestion role="listbox".
- Suggestion items use role="option".
- Error banner uses role="alert" with aria-live="assertive".
- Main grid uses aria-live="polite".
- HUD corner overlay uses aria-hidden="true".
- All controls are keyboard accessible.
- Visible amber focus indicator on all interactive elements.

### FR-10: Error and Empty States
- No airports selected: schedule area shows placeholder text.
- Validation error: inline error banner with a plain-language message; hides via CSS :empty when cleared.
- Table hidden until valid: table-wrap is hidden when validation fails; empty state shown instead.

## Data Requirements

### Airport/City Dataset
Bundled directly in the JavaScript. ~340 airports worldwide. Each record:
- code - IATA 3-letter code (string)
- city - city name (string)
- airport - airport name (string)
- country - country name (string)
- tz - IANA timezone string (e.g. "America/Los_Angeles")

Coverage: North America, South America, Europe, Middle East, Africa, South Asia, Southeast Asia, East Asia, Central Asia, Oceania, Pacific.

### Derived Values at Runtime
- originOffsetFromUTC: UTC offset for origin, sampled at local noon today
- destinationOffsetFromUTC: UTC offset for destination, sampled at local noon today
- rawOffsetHours: destination minus origin offset
- effectiveOffsetHours: shortest-direction offset in [-12, +12]
- shiftRate: h/day = 0.5 + (level - 1) x 0.5
- adaptationDays: ceiling(|effectiveOffset| / shiftRate)
- arrivalBedtime: user bedtime expressed in destination local time on arrival
- arrivalWakeTime: user wake time expressed in destination local time on arrival

## Visual Design
- Theme: Cinematic cockpit / instrument panel.
- Background: #080D1A dark navy, covered by the animated sky canvas.
- Panels: Dark glassmorphism - rgba(6,12,26,0.87) background, amber border rgba(255,184,0,0.28), backdrop-filter: blur(20px).
- Scanlines: subtle repeating-linear-gradient texture via ::after on every panel (pointer-events: none).
- HUD corners: Four amber L-bracket divs at viewport corners (decorative, pointer-events: none).
- Accent colours:
  - Amber #FFB800 - labels, borders, slider thumb, panel dots.
  - Teal #00E5CC - data values (stat tiles, table cells, chips).
  - Danger #FF5555 - error states, storm badge.
- Typography: Space Grotesk (body/headings), IBM Plex Mono (all labels, stat values, table data, chips).
- Table rows animate in with a 180 ms fade-slide on each recalculation.
- Table supports horizontal scroll on narrow viewports.

## Non-Functional Requirements
- Initial page load target: under 2 seconds on standard broadband.
- No full-page reload for any interaction.
- No external JavaScript dependencies; all logic is vanilla JS.
- Works on current versions of Chrome, Safari, Edge, and Firefox.
- Canvas animation targets 60 fps on modern hardware; ctx.filter blur is intentionally avoided for performance.
- No personal data is stored or transmitted.

## Privacy and Safety Notes
- No personal data storage of any kind.
- Travel advice is informational only and not medical advice. Footer note shown on every page load.

## Acceptance Criteria
1. Typing "LAX" or "Los" in origin shows matching suggestions immediately.
2. Typing "NRT" or "Tokyo" in destination shows matching suggestions and selecting one populates the chip.
3. If origin equals destination, an inline error appears and no schedule is shown.
4. Moving the aggressiveness slider from 2 to 5 instantly updates adaptation days, table rows, sky background, and weather badge.
5. Every table time is in 12-hour AM/PM format and does not wrap across lines.
6. A keyboard-only user can complete origin/destination selection, set bedtime/wake time, and adjust the slider to view the updated table.
7. If no suggestion matches a query, the dropdown shows "No matches".
8. For a LAX to NRT route (raw offset ~+17h), the effective offset used is -7h (shorter direction) and the schedule reflects that.
9. At slider = 1 (Gentle), the sky is bright blue with a visible sun and sparse clouds; weather badge shows CLEAR.
10. At slider = 5 (Aggressive), the sky is near-black with heavy rain and lightning; weather badge shows THUNDERSTORM.
11. On a 375 px wide viewport, controls and summary are visible without scrolling.
12. If bedtime or wake time is blank, invalid, or identical, a validation error appears and no schedule is generated.

## Future Enhancements
- Flight departure/arrival time integration to set the adaptation start date automatically.
- Light exposure and caffeine timing guidance overlaid on the schedule.
- Save/export schedule as PDF or calendar (.ics) events.
- Per-day DST-aware offset recalculation for routes spanning a DST boundary.
- 24-hour time display toggle.
- Date picker for planning future trips.
- User preference memory (aggressiveness default, time format).

## Implementation Notes
- UI structure as implemented:
  - Header panel: title, subtitle, weather badge.
  - Controls panel: origin, destination, bedtime, wake time, aggressiveness slider.
  - Flight Analysis panel: four stat tiles.
  - Sleep Plan panel: numbered schedule table.
  - Sky canvas: fixed fullscreen animated background.
- Client approach:
  - No debounce - suggestions render on every keystroke.
  - recalcAndRender() is a single pure function that reads all state and rerenders the full schedule.
  - Weather animation runs in a separate requestAnimationFrame loop; it reads the slider value directly on each frame.
