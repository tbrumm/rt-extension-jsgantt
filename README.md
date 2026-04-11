# RT::Extension::JSGantt

Gantt charts for your [Request Tracker](https://bestpractical.com/request-tracker) tickets — redesigned for RT 6 with Bootstrap 5 styling, full-width responsive layout, and inline editing of owner and dates directly in the chart.

---

## Screenshots

### Before — original narrow layout

<img src="Screenshots/Old View.png" alt="Old View: narrow fixed-width Gantt chart" width="100%">

The original extension rendered the chart in a fixed-width area that left most of the page unused, used small fonts with poor contrast, and offered no way to edit tickets inline.

### After — RT 6 redesign

<img src="Screenshots/New View.png" alt="New View: full-width Bootstrap Gantt chart with inline editing" width="100%">

The redesigned extension fills the full content area, matches the RT 6 Bootstrap 5 look and feel, and allows owners and dates to be changed directly in the chart view.

---

## Features

- **Full-width responsive layout** — the chart expands to fill the entire RT content area and redraws automatically on window resize
- **Bootstrap 5 styling** — fonts, colours, borders and buttons use Bootstrap CSS variables; dark mode is supported via `[data-bs-theme=dark]`
- **Inline owner editing** — an Owner dropdown per ticket lets you reassign tickets without leaving the Gantt view; only users with the `OwnTicket` right in the ticket's queue are shown
- **Inline date editing** — native browser date pickers for the Starts (Start) and Due (End) fields; changes are saved to RT immediately via the REST 2.0 API
- **Visual save feedback** — green border flash on success, red border and alert on error
- **Format-specific default date ranges** — separate `DefaultDays` values for Day, Week, Month and Quarter views
- **Bootstrap button group** for the Day / Week / Month / Quarter format selector (replaces old radio buttons)
- **Dependency arrows** rendered correctly over the full-width chart area

---

## RT Version

Works with RT **6.0** and **5.0**.

---

## Installation

```
perl Makefile.PL
make
make install        # may need root/sudo
```

Edit `/opt/rt6/etc/RT_SiteConfig.pm` and add:

```perl
Plugin('RT::Extension::JSGantt');
```

Clear the Mason cache and restart your web server:

```bash
rm -rf /opt/rt6/var/mason_data/obj
systemctl restart apache2    # or your web server
```

---

## Configuration

Add a `Set( %JSGanttOptions, … )` block to `/opt/rt6/etc/RT_SiteConfig.pm`.
All keys are optional; the values below are the defaults.

```perl
Set(
    %JSGanttOptions,

    # ── Chart appearance ───────────────────────────────────────────────────
    # Initial view format when the chart loads
    DefaultFormat => 'day',      # 'day' | 'week' | 'month' | 'quarter'

    # Which columns to show in the left panel
    ShowOwner    => 1,           # Owner column (inline-editable dropdown)
    ShowDuration => 1,           # Duration column
    ShowProgress => 1,           # % Complete column
    # ShowStartDate => 1,        # Start date column (inline-editable date picker)
    # ShowEndDate   => 1,        # End date column   (inline-editable date picker)

    # Caption shown inside each task bar
    # CaptionType => 'Resource', # 'Resource' | 'Caption' | 'Duration' | 'Complete' | 'None'

    # Date formats (mm/dd/yyyy | dd/mm/yyyy | yyyy-mm-dd)
    # DateInputFormat   => 'mm/dd/yyyy',
    # DateDisplayFormat => 'mm/dd/yyyy',

    # Which zoom buttons to show at the bottom of the left panel
    # FormatArr => q|'day','week','month','quarter'|,

    # ── Colours ────────────────────────────────────────────────────────────
    # Colour tickets by owner for easy identification (default: on)
    # ColorSchemeByOwner => 1,

    # Fixed colour palette (six hex codes, no leading #)
    # ColorScheme => ['ff0000', 'ffff00', 'ff00ff', '00ff00', '00ffff', '0000ff'],

    # Per-owner colour overrides (unspecified owners get an auto-assigned colour)
    # ColorSchemeByOwner => { root => 'ff0000', alice => '00cc00' },

    # Colour used when a ticket has no Starts/Due dates
    NullDatesColor => '333333',

    # ── Date-range defaults ────────────────────────────────────────────────
    # When a ticket has no dates and no time estimate, the chart displays a
    # bar spanning this many days.  DefaultDays is the global fallback;
    # the per-format keys take precedence when that format is active.
    DefaultDays        => 31,
    # DefaultDaysDay     => 31,
    # DefaultDaysWeek    => 90,
    # DefaultDaysMonth   => 180,
    # DefaultDaysQuarter => 365,

    # ── Time calculation ───────────────────────────────────────────────────
    # Used to convert TimeEstimated / TimeLeft (minutes) into days
    WorkingHoursPerDay => 8,
);
```

---

## Inline Editing

### Owner

The Owner column renders a dropdown (`<select>`) for every non-group ticket row.
The list contains only **privileged users who hold the `OwnTicket` right** in the
queue of that ticket — across all queues shown in the current chart.

Selecting a new owner immediately sends a `PUT` request to the RT REST 2.0 API
(`/REST/2.0/ticket/{id}`) and updates the ticket.

### Start and End Dates

The Start and End columns render native `<input type="date">` date pickers.
Changing a date immediately sends a `PUT` request to the REST 2.0 API, setting
the ticket's `Starts` (Start) or `Due` (End) field.

Both fields give visual feedback:
- **Green border flash** — change saved successfully
- **Red border + alert** — RT returned an error (the error message is shown)

> **Permissions note:** The logged-in user must have `ModifyTicket` (and
> `ReassignTicket` for owner changes) in the relevant queue for the save to
> succeed. RT will return an error otherwise.

---

## Methods

### `AllRelatedTickets`

Given a ticket, returns all related tickets (parent, children, dependencies)
including the original ticket itself.

```perl
my @tickets = RT::Extension::JSGantt->AllRelatedTickets(
    Ticket      => $ticket,
    CurrentUser => $session{CurrentUser},
);
```

### `TicketsInfo`

Given an array of ticket objects, resolves the data needed by the chart
(name, start, end, colour, owner, progress, dependencies, …).
Returns a two-element list: an arrayref of IDs and a hashref of info.

```perl
my ($ids, $info) = RT::Extension::JSGantt->TicketsInfo(
    Tickets     => \@tickets,
    CurrentUser => $session{CurrentUser},
);
```

### `GetTimeRange`

Given a ticket, resolves its start and end dates from the ticket fields,
transaction history, and time estimates.
Returns `($start_obj, $start_str, $end_obj, $end_str)`.

---

## Upgrading

### From versions before 1.02 — `DateDayBeforeMonth`

If you had the undocumented `DateDayBeforeMonth` option set, replace it with:

```perl
Set( %JSGanttOptions, DateDisplayFormat => 'dd/mm/yyyy' );
```

### `DefaultDays` default changed from 7 to 31

The built-in fallback for `DefaultDays` has been raised from 7 to 31 days so
that charts look populated out of the box.  If you had `DefaultDays => 7` in
your config and want to keep that behaviour, set it explicitly.

---

## Author

Best Practical Solutions, LLC &lt;modules@bestpractical.com&gt;

RT 6 redesign (Bootstrap 5 layout, inline editing, format-specific defaults)
contributed by the RT::Extension::JSGantt community.

---

## Bugs

Report bugs by email to
[bug-RT-Extension-JSGantt@rt.cpan.org](mailto:bug-RT-Extension-JSGantt@rt.cpan.org)
or via the web at
[rt.cpan.org](http://rt.cpan.org/Public/Dist/Display.html?Name=RT-Extension-JSGantt).

---

## License

This software is Copyright (c) 2014–2025 by Best Practical Solutions.

This is free software, licensed under the
[GNU General Public License, Version 2, June 1991](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html).
