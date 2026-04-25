# Functional Specification Document (FSD)

## Product Name
Jet Lag Recovery Timeline Page (HTML)

## Document Purpose
Define the functional requirements for a single-page, HTML-based experience that helps a traveler generate a personalized jet lag recovery timeline based on:
- Starting city or airport code
- Destination city or airport code
- User-selected recovery aggressiveness

The page updates a destination-time sleep schedule table in real time as the user changes inputs.

## Scope
In scope:
- One responsive web page built with HTML, CSS, and JavaScript.
- Searchable input controls for origin and destination.
- Instant search suggestions while typing for airport and city choices.
- Aggressiveness slider that influences adaptation speed.
- Real-time generation of a destination-time sleep table.
- Basic validation and error states.

Out of scope:
- Flight booking or airline integration.
- User accounts and authentication.
- Medical diagnosis or treatment claims.
- Offline mode and native mobile apps.

## Target Users
- Travelers preparing for international or cross-time-zone trips.
- Users who want a practical sleep adjustment plan in destination local time.

## User Goals
- Quickly find and select origin and destination airports or cities.
- Tune recovery style from conservative to aggressive.
- Instantly see a day-by-day sleep schedule in destination time.

## Assumptions
- The application has access to an airport/city dataset with IATA code, city name, airport name, and time zone.
- Time zone conversion is supported by browser APIs or a bundled data source.
- User can manually adjust date if needed (default is current date in destination).
- User can enter normal sleeping time and waking time patterns for the schedule generation algorithm.

## Primary User Stories
1. As a traveler, I want to type part of an airport code or city and get suggestions immediately so I can select locations quickly.
2. As a traveler, I want to set how aggressive my recovery plan should be so the timeline matches my comfort level.
3. As a traveler, I want the sleep table to update instantly when I change any input so I can compare options in real time.
4. As a traveler, I want all times shown in destination local time so I can follow the schedule after arrival.
5. As a traveler, I want to see how many days it will take to adapt based on my aggressiveness choice so I can plan accordingly.
6. As a traveler, I want to see clear validation messages if I enter invalid input so I can correct it and get accurate results.
7. As a traveler, I want to use the page on my mobile device with the same functionality as desktop so I can plan on the go.
8. As a traveler, I want to understand the meaning of the aggressiveness levels so I can choose the right one for me.
9. As a traveler, I want to see a clear "No matches" message if my search input doesn't match any locations so I know to adjust my query.
10. As a traveler, I want to see the time zone difference between origin and destination so I can understand the scale of adjustment needed.

## Functional Requirements

### FR-1: Page Layout
- The page must include:
  - Origin search input.
  - Destination search input.
  - Normal bedtime input (user's typical sleep start time).
  - Normal wake-time input (user's typical wake time).
  - Aggressiveness slider with label and current value.
  - Summary panel (time-zone difference and estimated adaptation days).
  - Sleep schedule table (destination local time).
- Layout must be responsive for desktop and mobile.

### FR-2: Origin and Destination Search Inputs
- Each input must support free typing with instant suggestion results.
- Suggestion matching rules:
  - Match by city name, airport name, and IATA code.
  - Case-insensitive.
  - Starts-with matches ranked first, then contains matches.
- Suggestion list behavior:
  - Appears within 150 ms of typing pause or faster.
  - Supports keyboard navigation (up/down/enter/escape).
  - Mouse click selects an item.
- On selection, show a compact display format:
  - Example: LAX - Los Angeles (Los Angeles International)
- If no results found, show a clear "No matches" state.

### FR-3: Input Validation
- Origin and destination are required.
- Origin and destination cannot be identical.
- Normal bedtime and wake time are required.
- Bedtime and wake time must be valid local clock times.
- If bedtime equals wake time, show validation error (sleep window cannot be zero length).
- If validation fails, schedule table is hidden or disabled and inline error text is shown.

### FR-4: Aggressiveness Slider
- Slider range: 1 to 5.
- Default value: 3.
- Labels:
  - 1 = Gentle
  - 2 = Moderate-Gentle
  - 3 = Moderate
  - 4 = Moderate-Aggressive
  - 5 = Aggressive
- Slider changes must trigger immediate schedule recalculation.

### FR-5: Timeline and Sleep Table Generation
- Inputs to the schedule algorithm:
  - Origin time zone.
  - Destination time zone.
  - User normal bedtime.
  - User normal wake time.
  - Aggressiveness level.
  - Start date (default current date in destination, optional date picker for future enhancement).
- Calculate raw time-zone offset difference in hours.
- The system must use the shorter adjustment direction across a 24-hour clock:
  - Compute rawOffsetHours = destinationOffsetFromUTC - originOffsetFromUTC.
  - Convert to effectiveOffsetHours in the range [-12, +12].
  - If abs(rawOffsetHours) > 12, use 24 - abs(rawOffsetHours) and reverse direction.
  - This prevents plans from using adjustment magnitudes greater than 12 hours.
- Estimate adaptation rate (hours shifted per day) based on aggressiveness:
  - Level 1: 0.5 h/day
  - Level 2: 1.0 h/day
  - Level 3: 1.5 h/day
  - Level 4: 2.0 h/day
  - Level 5: 2.5 h/day
- Estimate total adaptation days:
  - days = ceiling(abs(effectiveOffsetHours) / shiftRate)
- Generate daily target sleep window entries in destination local time:
  - Day index (Day 1, Day 2, ...)
  - Bedtime (destination local time)
  - Wake time (destination local time)
  - Daily shift note (for example: "Shift bedtime 1.5h earlier")
- The initial schedule baseline must be derived from the user's normal bedtime and wake time before daily shifts are applied.
- Table updates in real time on every valid input change.

### FR-6: Real-Time Update Rules
- The page must recalculate and rerender schedule results when any of the following changes:
  - Origin selection
  - Destination selection
  - Normal bedtime
  - Normal wake time
  - Aggressiveness slider value
- Update latency target: visible table update within 200 ms after input change under normal dataset size.

### FR-7: Destination-Time Display
- All schedule times shown in destination local time with clear timezone label.
- Display format:
  - 12-hour with AM/PM and optional 24-hour toggle (future enhancement).
- Include timezone abbreviation when available.

### FR-8: Accessibility and Usability
- All controls must be keyboard accessible.
- Inputs and slider must have visible labels.
- Search results must be announced for screen readers via ARIA live region or combobox pattern.
- Color contrast must meet WCAG AA for text and controls.

### FR-9: Error and Empty States
- If dataset fails to load, show retry message and disable schedule generation.
- If user has not selected both airports, show guidance placeholder in the table area.
- If algorithm cannot compute due to missing timezone data, show actionable error.

## Data Requirements

### Airport/City Record
Each searchable record should include:
- iataCode (string, 3 chars)
- cityName (string)
- airportName (string)
- countryName (string)
- timeZoneId (IANA timezone string, example: America/Los_Angeles)

### Derived Values
- originOffsetFromUTC (number, hours)
- destinationOffsetFromUTC (number, hours)
- rawOffsetHours (destination minus origin)
- effectiveOffsetHours (shortest-direction offset in range [-12, +12])
- baselineSleepStartLocal (time, from user normal bedtime)
- baselineSleepEndLocal (time, from user normal wake time)
- shiftRateByAggressiveness (number)
- adaptationDays (integer)

## UX Behavior Details
- Typing in either search box instantly filters and shows top 8 suggestions.
- Selected items remain visible below input as a pill or compact summary row.
- Bedtime and wake-time inputs show picker-friendly controls on mobile and desktop (for example, HTML time inputs) and update results immediately.
- Slider thumb movement updates value label continuously.
- Schedule table animates row updates subtly (fade/slide under 150 ms) to communicate recalculation without distraction.
- On mobile, inputs stack vertically and table supports horizontal scrolling if needed.

## Non-Functional Requirements
- Initial page load target: under 2 seconds on standard broadband.
- No full-page reload for interactions.
- Works on current versions of Chrome, Safari, Edge, and Firefox.
- JavaScript errors are handled gracefully with user feedback.

## Privacy and Safety Notes
- No personal data storage is required.
- Travel advice is informational only and not medical advice.

## Acceptance Criteria
1. User can type "LAX" or "Los" in origin input and see matching suggestions immediately.
2. User can type "NRT" or "Tokyo" in destination input and select a result.
3. If origin equals destination, an inline validation message appears and no schedule is shown.
4. Moving aggressiveness slider from 2 to 5 updates adaptation days and table rows instantly.
5. Every table time is labeled or otherwise clearly represented in destination local time.
6. Keyboard-only user can complete location selection and slider adjustment and view updated table.
7. If no suggestion matches, UI shows "No matches" and allows continued typing.
8. For routes where raw offset exceeds 12 hours, schedule generation uses the shorter-direction adjustment (for example, a raw +14 hour difference is treated as -10 hours for adaptation-day calculation).
9. User can enter normal bedtime and wake time, and the generated schedule reflects those times as the baseline in destination-time output.
10. If bedtime or wake time is missing, invalid, or identical, the page shows a clear validation message and does not generate a schedule.

## Future Enhancements
- Flight departure/arrival time integration.
- Light exposure and caffeine guidance blocks.
- Save/export schedule as PDF or calendar events.
- User preference memory for time display and aggressiveness defaults.

## Implementation Notes (Non-Binding)
- Suggested UI structure:
  - Header: title and short explainer.
  - Input panel: origin, destination, aggressiveness.
  - Results panel: offset summary and adaptation days.
  - Table panel: day-by-day destination-time sleep plan.
- Suggested client approach:
  - Debounced search (50-150 ms).
  - Pure function for schedule generation.
  - State-driven rerender for deterministic updates.
