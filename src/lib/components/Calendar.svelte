<script>
    import { onMount, onDestroy, untrack } from 'svelte'
    import { settings } from '../stores/settings-store.svelte.js'
    import { isChrome } from '../utils/browser-detect.js'
    import GoogleCalendarAPI from '../api/google-calendar-api.js'

    let { class: className = '' } = $props()

    let events = $state([])
    let loading = $state(false)
    let error = $state('')
    let initialLoad = $state(true)
    let noCalendarsSelected = $state(false)
    let monthAnchor = $state(new Date())
    let selectedDateKey = $state(null)
    let eventsListEl = $state(null)
    let activeMonthKey = $state(null)

    const calendarApi = new GoogleCalendarAPI()
    const weekdayLabels = ['su', 'mo', 'tu', 'we', 'th', 'fr', 'sa']

    function clampMaxEvents(value) {
        const parsed = parseInt(value, 10)
        if (Number.isNaN(parsed)) return 5
        return Math.min(20, Math.max(1, parsed))
    }

    function handleVisibilityChange() {
        if (document.visibilityState === 'visible') {
            loadEvents()
        }
    }

    function startOfDay(date) {
        return new Date(date.getFullYear(), date.getMonth(), date.getDate())
    }

    function toDateKey(date) {
        const year = date.getFullYear()
        const month = String(date.getMonth() + 1).padStart(2, '0')
        const day = String(date.getDate()).padStart(2, '0')
        return `${year}-${month}-${day}`
    }

    function parseDateKey(dateKey) {
        const [year, month, day] = dateKey.split('-').map((value) => Number(value))
        return new Date(year, month - 1, day)
    }

    function shiftMonth(offset) {
        monthAnchor = new Date(
            monthAnchor.getFullYear(),
            monthAnchor.getMonth() + offset,
            1
        )
    }

    function getEventDayRange(event) {
        const startDay = startOfDay(event.start)
        let endExclusive = new Date(
            startDay.getFullYear(),
            startDay.getMonth(),
            startDay.getDate() + 1
        )

        if (event.end) {
            endExclusive = event.allDay
                ? startOfDay(event.end)
                : new Date(
                      event.end.getFullYear(),
                      event.end.getMonth(),
                      event.end.getDate() + 1
                  )
        }

        if (endExclusive.getTime() <= startDay.getTime()) {
            endExclusive = new Date(
                startDay.getFullYear(),
                startDay.getMonth(),
                startDay.getDate() + 1
            )
        }

        return { startDay, endExclusive }
    }

    function eventOccursOnDate(event, dateKey) {
        const dayStart = parseDateKey(dateKey)
        const dayEnd = new Date(
            dayStart.getFullYear(),
            dayStart.getMonth(),
            dayStart.getDate() + 1
        )
        const { startDay, endExclusive } = getEventDayRange(event)
        return startDay.getTime() < dayEnd.getTime() &&
            endExclusive.getTime() > dayStart.getTime()
    }

    function formatMonthLabel(date) {
        return date
            .toLocaleDateString('en-US', {
                month: 'long',
                year: 'numeric',
            })
            .toLowerCase()
    }

    function formatSelectedDate(dateKey) {
        return parseDateKey(dateKey)
            .toLocaleDateString('en-US', {
                weekday: 'short',
                month: 'short',
                day: 'numeric',
            })
            .toLowerCase()
    }

    function formatMonthKey(date) {
        return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}`
    }

    function formatTickerMonthLabel(date) {
        return date
            .toLocaleDateString('en-US', { month: 'short' })
            .toLowerCase()
    }

    function setSelectedDate(dateKey) {
        selectedDateKey = dateKey
        if (eventsListEl) eventsListEl.scrollTop = 0
    }

    function clearSelectedDate() {
        selectedDateKey = null
        if (eventsListEl) eventsListEl.scrollTop = 0
    }

    let dayEventMeta = $derived.by(() => {
        const counts = new Map()
        const titles = new Map()

        for (const event of events) {
            const { startDay, endExclusive } = getEventDayRange(event)
            let cursor = new Date(startDay)
            let steps = 0

            while (
                cursor.getTime() < endExclusive.getTime() &&
                steps < 62
            ) {
                const key = toDateKey(cursor)
                counts.set(key, (counts.get(key) || 0) + 1)

                const titleList = titles.get(key) || []
                if (titleList.length < 3) {
                    titleList.push(event.title)
                    titles.set(key, titleList)
                }

                cursor = new Date(
                    cursor.getFullYear(),
                    cursor.getMonth(),
                    cursor.getDate() + 1
                )
                steps += 1
            }
        }

        return { counts, titles }
    })

    let monthDays = $derived.by(() => {
        const monthStart = new Date(
            monthAnchor.getFullYear(),
            monthAnchor.getMonth(),
            1
        )
        const gridStart = new Date(
            monthStart.getFullYear(),
            monthStart.getMonth(),
            monthStart.getDate() - monthStart.getDay()
        )
        const todayKey = toDateKey(new Date())

        return Array.from({ length: 42 }, (_, index) => {
            const date = new Date(
                gridStart.getFullYear(),
                gridStart.getMonth(),
                gridStart.getDate() + index
            )
            const key = toDateKey(date)
            const eventCount = dayEventMeta.counts.get(key) || 0
            const previewTitles = dayEventMeta.titles.get(key) || []
            const tooltip = previewTitles.length
                ? previewTitles.join(' | ')
                : 'no events'

            return {
                key,
                date,
                day: date.getDate(),
                inCurrentMonth: date.getMonth() === monthAnchor.getMonth(),
                isToday: key === todayKey,
                eventCount,
                tooltip,
            }
        })
    })

    let visibleEvents = $derived.by(() => {
        if (!selectedDateKey) return events
        return events.filter((event) => eventOccursOnDate(event, selectedDateKey))
    })

    let monthGroups = $derived.by(() => {
        const grouped = []
        const byMonth = new Map()
        let lastYear = null

        for (const event of visibleEvents) {
            const monthDate = new Date(
                event.start.getFullYear(),
                event.start.getMonth(),
                1
            )
            const key = formatMonthKey(monthDate)

            if (!byMonth.has(key)) {
                const year = monthDate.getFullYear()
                const group = {
                    key,
                    label: formatMonthLabel(monthDate),
                    tickerLabel: formatTickerMonthLabel(monthDate),
                    year,
                    showYear: year !== lastYear,
                    events: [],
                }
                byMonth.set(key, group)
                grouped.push(group)
                lastYear = year
            }

            byMonth.get(key).events.push(event)
        }

        return grouped
    })

    function handleEventsScroll() {
        if (!eventsListEl) return

        const sections = eventsListEl.querySelectorAll('.month-section')
        if (sections.length === 0) {
            activeMonthKey = null
            return
        }

        const scrollTop = eventsListEl.scrollTop
        let currentKey = null

        for (const section of sections) {
            if (section.offsetTop <= scrollTop + 8) {
                currentKey = section.getAttribute('data-month-key')
            } else {
                break
            }
        }

        if (!currentKey) {
            currentKey = sections[0].getAttribute('data-month-key')
        }

        activeMonthKey = currentKey
    }

    function scrollToMonth(monthKey) {
        if (!eventsListEl) return
        const target = eventsListEl.querySelector(
            `[data-month-key="${monthKey}"]`
        )
        if (!target) return

        eventsListEl.scrollTo({
            top: target.offsetTop,
            behavior: 'smooth',
        })
        activeMonthKey = monthKey
    }

    $effect(() => {
        const groups = monthGroups
        if (groups.length === 0) {
            activeMonthKey = null
            return
        }

        if (!groups.some((group) => group.key === activeMonthKey)) {
            activeMonthKey = groups[0].key
        }
    })

    $effect(() => {
        settings.googleCalendarSignedIn
        settings.googleCalendarMaxEvents
        settings.googleCalendarVisibleCalendarIds
        settings.timeFormat

        if (untrack(() => initialLoad)) {
            initialLoad = false
            return
        }

        loadEvents(true)
    })

    function formatEventDate(date, allDay) {
        const now = new Date()
        const today = new Date(now.getFullYear(), now.getMonth(), now.getDate())
        const tomorrow = new Date(today.getTime() + 24 * 60 * 60 * 1000)
        const eventDay = new Date(date.getFullYear(), date.getMonth(), date.getDate())

        let dayLabel = eventDay
            .toLocaleDateString('en-US', { month: 'short', day: 'numeric' })
            .toLowerCase()

        if (eventDay.getTime() === today.getTime()) dayLabel = 'today'
        if (eventDay.getTime() === tomorrow.getTime()) dayLabel = 'tmrw'

        if (allDay) {
            return `${dayLabel} all day`
        }

        const use12Hour = settings.timeFormat === '12hr'
        const timeLabel = date
            .toLocaleTimeString('en-US', {
                hour: 'numeric',
                minute: '2-digit',
                hour12: use12Hour,
            })
            .toLowerCase()

        return `${dayLabel} ${timeLabel}`
    }

    export async function loadEvents(showLoading = false) {
        if (!isChrome()) {
            error = 'google calendar only works in chrome'
            events = []
            noCalendarsSelected = false
            loading = false
            return
        }

        if (!settings.googleCalendarSignedIn) {
            error = 'not signed in to google calendar'
            events = []
            noCalendarsSelected = false
            loading = false
            return
        }

        if (
            Array.isArray(settings.googleCalendarVisibleCalendarIds) &&
            settings.googleCalendarVisibleCalendarIds.length === 0
        ) {
            error = ''
            events = []
            noCalendarsSelected = true
            loading = false
            return
        }

        if (showLoading) loading = true
        error = ''
        noCalendarsSelected = false

        try {
            const maxEvents = clampMaxEvents(settings.googleCalendarMaxEvents)
            events = await calendarApi.getUpcomingEvents(
                maxEvents,
                settings.googleCalendarVisibleCalendarIds
            )
        } catch (err) {
            if (err.message?.includes('Authentication expired')) {
                settings.googleCalendarSignedIn = false
                error = 'google sign in expired'
            } else {
                error = 'failed to load calendar'
            }
            console.error('calendar sync failed:', err)
        } finally {
            if (showLoading) loading = false
        }
    }

    export function refreshCalendar() {
        loadEvents(true)
    }

    onMount(() => {
        loadEvents(true)
        document.addEventListener('visibilitychange', handleVisibilityChange)
    })

    onDestroy(() => {
        document.removeEventListener('visibilitychange', handleVisibilityChange)
    })
</script>

<div class="panel-wrapper {className}">
    <button class="widget-label" onclick={refreshCalendar} disabled={loading}>
        {loading ? 'loading...' : 'calendar'}
    </button>
    <div class="panel">
        {#if error}
            <div class="error">{error}</div>
        {:else if noCalendarsSelected}
            <div class="dark">no calendars selected</div>
        {:else}
            <div class="calendar-layout">
                <div class="month-header">
                    <button
                        class="month-nav"
                        type="button"
                        aria-label="Previous month"
                        onclick={() => shiftMonth(-1)}
                    >
                        &lt;
                    </button>
                    <div class="month-label">{formatMonthLabel(monthAnchor)}</div>
                    <button
                        class="month-nav"
                        type="button"
                        aria-label="Next month"
                        onclick={() => shiftMonth(1)}
                    >
                        &gt;
                    </button>
                </div>

                <div class="weekday-row">
                    {#each weekdayLabels as weekday}
                        <span>{weekday}</span>
                    {/each}
                </div>

                <div class="month-grid">
                    {#each monthDays as day}
                        <button
                            type="button"
                            class="month-day"
                            class:muted={!day.inCurrentMonth}
                            class:today={day.isToday}
                            class:selected={selectedDateKey === day.key}
                            class:has-events={day.eventCount > 0}
                            onclick={() => setSelectedDate(day.key)}
                            title={day.tooltip}
                            aria-label={day.key}
                        >
                            <span class="day-number">{day.day}</span>
                            {#if day.eventCount > 0}
                                <span class="day-count">{day.eventCount}</span>
                            {/if}
                        </button>
                    {/each}
                </div>

                <div class="events-header">
                    <span class="events-label">
                        {selectedDateKey
                            ? formatSelectedDate(selectedDateKey)
                            : 'upcoming events'}
                    </span>
                    {#if selectedDateKey}
                        <button
                            class="clear-selection"
                            type="button"
                            onclick={clearSelectedDate}
                        >
                            show upcoming
                        </button>
                    {/if}
                </div>

                <div class="events-wrap">
                    {#if visibleEvents.length === 0}
                        <div class="dark">
                            {selectedDateKey
                                ? 'no events on selected day'
                                : 'no upcoming events'}
                        </div>
                    {:else}
                        <div class="events" bind:this={eventsListEl} onscroll={handleEventsScroll}>
                            {#each monthGroups as monthGroup}
                                <div
                                    class="month-section"
                                    data-month-key={monthGroup.key}
                                >
                                    <div class="month-marker">{monthGroup.label}</div>
                                    {#each monthGroup.events as event}
                                        <div class="event">
                                            <div class="event-time">
                                                {formatEventDate(
                                                    event.start,
                                                    event.allDay
                                                )}
                                            </div>
                                            {#if event.link}
                                                <a
                                                    class="event-title bright"
                                                    href={event.link}
                                                    target="_blank"
                                                    rel="noopener noreferrer"
                                                >
                                                    {event.title}
                                                </a>
                                            {:else}
                                                <div class="event-title bright">
                                                    {event.title}
                                                </div>
                                            {/if}
                                            {#if event.location}
                                                <div class="event-location">
                                                    {event.location}
                                                </div>
                                            {/if}
                                            {#if !event.calendarPrimary}
                                                <div class="event-calendar">
                                                    #{event.calendarName}
                                                </div>
                                            {/if}
                                        </div>
                                    {/each}
                                </div>
                            {/each}
                        </div>
                        {#if monthGroups.length > 1}
                            <div class="month-ticker">
                                {#each monthGroups as monthGroup}
                                    {#if monthGroup.showYear}
                                        <div class="ticker-year-divider">
                                            {monthGroup.year}
                                        </div>
                                    {/if}
                                    <button
                                        type="button"
                                        class="ticker-item"
                                        class:active={activeMonthKey === monthGroup.key}
                                        onclick={() => scrollToMonth(monthGroup.key)}
                                        title={monthGroup.label}
                                    >
                                        <span>{monthGroup.tickerLabel}</span>
                                    </button>
                                {/each}
                            </div>
                        {/if}
                    {/if}
                </div>
            </div>
        {/if}
    </div>
</div>

<style>
    .panel-wrapper {
        flex: 1;
        min-width: 0;
    }
    .calendar-layout {
        display: flex;
        flex-direction: column;
        gap: 0.35rem;
    }
    .month-header {
        display: flex;
        align-items: center;
        justify-content: space-between;
    }
    .month-label {
        color: var(--txt-2);
        text-transform: lowercase;
    }
    .month-nav {
        color: var(--txt-3);
        width: 1.5rem;
        text-align: center;
    }
    .weekday-row {
        display: grid;
        grid-template-columns: repeat(7, minmax(0, 1fr));
        color: var(--txt-4);
        font-size: 0.75rem;
        text-transform: lowercase;
    }
    .weekday-row span {
        text-align: center;
    }
    .month-grid {
        display: grid;
        grid-template-columns: repeat(7, minmax(0, 1fr));
        gap: 0.15rem;
    }
    .month-day {
        min-height: 1.45rem;
        border: 1px solid var(--bg-3);
        background: var(--bg-2);
        display: inline-flex;
        align-items: center;
        justify-content: center;
        gap: 0.2rem;
        color: var(--txt-3);
    }
    .month-day.muted {
        opacity: 0.45;
    }
    .month-day.today {
        border-color: var(--txt-3);
    }
    .month-day.selected {
        border-color: var(--txt-2);
        color: var(--txt-1);
    }
    .month-day.has-events .day-number {
        color: var(--txt-2);
    }
    .day-count {
        font-size: 0.625rem;
        color: var(--txt-4);
    }
    .events-header {
        display: flex;
        align-items: center;
        justify-content: space-between;
        margin-top: 0.1rem;
    }
    .events-label {
        color: var(--txt-3);
        text-transform: lowercase;
    }
    .clear-selection {
        color: var(--txt-3);
        font-size: 0.8rem;
    }
    .events-wrap {
        position: relative;
    }
    .events {
        display: grid;
        gap: 0.35rem;
        overflow: auto;
        scrollbar-width: none;
        height: 12.75rem;
        padding-right: 2.75rem;
        scroll-behavior: smooth;
    }
    .month-section {
        display: grid;
        gap: 0.35rem;
    }
    .month-marker {
        position: sticky;
        top: 0;
        z-index: 2;
        color: var(--txt-4);
        text-transform: lowercase;
        background: linear-gradient(
            to right,
            var(--bg-1) 0%,
            var(--bg-1) 80%,
            transparent 100%
        );
        border-bottom: 1px solid var(--bg-3);
        padding: 0.1rem 0;
    }
    .month-ticker {
        position: absolute;
        top: 0;
        right: 0;
        bottom: 0;
        display: flex;
        flex-direction: column;
        align-items: flex-end;
        gap: 0.1rem;
        padding-right: 0.1rem;
        pointer-events: none;
    }
    .ticker-item {
        display: block;
        flex-shrink: 0;
        pointer-events: auto;
        color: var(--txt-4);
        font-size: 0.7rem;
        text-transform: lowercase;
        text-align: right;
        border-right: 2px solid transparent;
        padding-right: 0.35rem;
        line-height: 1;
    }
    .ticker-item.active {
        color: var(--txt-2);
        border-right-color: var(--txt-3);
    }
    .ticker-item:hover {
        color: var(--txt-2);
    }
    .ticker-year-divider {
        color: var(--txt-4);
        font-size: 0.55rem;
        line-height: 1;
        text-align: right;
        padding-right: 0.35rem;
        opacity: 0.75;
        margin-top: 0.15rem;
    }
    .event {
        min-width: 0;
    }
    .event-time {
        color: var(--txt-3);
    }
    .event-title {
        display: block;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
    .event-location {
        color: var(--txt-3);
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
    .event-calendar {
        color: var(--txt-3);
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
</style>
