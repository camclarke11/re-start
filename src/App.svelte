<script>
    import { onMount } from 'svelte'
    import '@fontsource-variable/geist-mono'
    import 'virtual:simple-icons.css'
    import {
        settings,
        MAIN_WIDGET_IDS,
        normalizeMainWidgetOrder,
    } from './lib/stores/settings-store.svelte.js'
    import { defaultTheme, defaultCustomColors } from './lib/config/themes.js'
    import Clock from './lib/components/Clock.svelte'
    import Calendar from './lib/components/Calendar.svelte'
    import Links from './lib/components/Links.svelte'
    import CommandLauncher from './lib/components/CommandLauncher.svelte'
    import Settings from './lib/components/Settings.svelte'
    import Stats from './lib/components/Stats.svelte'
    import Tasks from './lib/components/Tasks.svelte'
    import Weather from './lib/components/Weather.svelte'
    import { saveSettings } from './lib/stores/settings-store.svelte.js'
    import { isChrome } from './lib/utils/browser-detect.js'

    let showSettings = $state(false)
    let appReady = $state(false)

    // Check if Google Tasks is available (Chrome only)
    const googleTasksAvailable = isChrome()
    const googleCalendarAvailable = isChrome()

    let needsConfiguration = $derived(
        (settings.locationMode === 'manual' &&
            (settings.latitude === null || settings.longitude === null)) ||
            (settings.taskBackend === 'todoist' && !settings.todoistApiToken) ||
            (settings.taskBackend === 'google-tasks' &&
                googleTasksAvailable &&
                !settings.googleTasksSignedIn) ||
            (settings.showCalendar &&
                googleCalendarAvailable &&
                !settings.googleCalendarSignedIn)
    )

    const widgetComponents = {
        weather: Weather,
        tasks: Tasks,
        calendar: Calendar,
    }

    function isWidgetVisible(widgetId) {
        switch (widgetId) {
            case 'weather':
                return settings.showWeather
            case 'tasks':
                return settings.showTasks
            case 'calendar':
                return googleCalendarAvailable && settings.showCalendar
            default:
                return false
        }
    }

    let orderedVisibleWidgets = $derived.by(() => {
        const normalizedOrder = normalizeMainWidgetOrder(settings.mainWidgetOrder)
        const uniqueOrderedWidgets = []
        const seen = new Set()

        for (const widgetId of normalizedOrder) {
            if (!MAIN_WIDGET_IDS.includes(widgetId) || seen.has(widgetId)) {
                continue
            }
            seen.add(widgetId)
            uniqueOrderedWidgets.push(widgetId)
        }

        return uniqueOrderedWidgets.filter((widgetId) => isWidgetVisible(widgetId))
    })

    function closeSettings() {
        showSettings = false
    }

    function openSettings() {
        showSettings = true
    }

    onMount(() => {
        if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) {
            appReady = true
            return
        }

        // Delay one paint so initial styles apply before the reveal transition.
        requestAnimationFrame(() => {
            requestAnimationFrame(() => {
                appReady = true
            })
        })
    })

    function applyTheme(themeName) {
        document.documentElement.className =
            'theme-' + (themeName || defaultTheme)
    }

    function applyCustomThemeColors(colors) {
        let styleEl = document.getElementById('custom-theme-vars')
        if (!styleEl) {
            styleEl = document.createElement('style')
            styleEl.id = 'custom-theme-vars'
            document.head.appendChild(styleEl)
        }
        const c = colors || defaultCustomColors
        styleEl.textContent = `:root.theme-custom {
            --bg-1: ${c.bg1}; --bg-2: ${c.bg2}; --bg-3: ${c.bg3};
            --txt-1: ${c.txt1}; --txt-2: ${c.txt2}; --txt-3: ${c.txt3};
            --txt-4: ${c.txt4}; --txt-err: ${c.txtErr};
        }`
    }

    $effect(() => {
        const fontName = settings.font?.trim() || 'Geist Mono Variable'
        document.documentElement.style.setProperty(
            '--font-family',
            `'${fontName}', monospace`
        )
    })

    $effect(() => {
        applyTheme(settings.currentTheme)
    })

    $effect(() => {
        applyCustomThemeColors(settings.customThemeColors)
    })

    $effect(() => {
        document.title = settings.tabTitle || '~'
    })

    $effect(() => {
        let styleEl = document.getElementById('custom-css')
        if (!styleEl) {
            styleEl = document.createElement('style')
            styleEl.id = 'custom-css'
            document.head.appendChild(styleEl)
        }
        styleEl.textContent = settings.customCSS || ''
    })

    $effect(() => {
        saveSettings(settings)
    })
</script>

<main>
    <div class="command-row">
        <CommandLauncher />
    </div>

    <div class="container" class:ready={appReady}>
        {#if settings.showClock || settings.showStats}
            <div class="top">
                {#if settings.showClock}
                    <Clock />
                {/if}
                {#if settings.showStats}
                    <Stats class={!settings.showClock ? 'expand' : ''} />
                {/if}
            </div>
        {/if}
        {#if orderedVisibleWidgets.length}
            <div class="widgets">
                {#each orderedVisibleWidgets as widgetId (widgetId)}
                    {@const WidgetComponent = widgetComponents[widgetId]}
                    <div class="widget-slot" class:tasks-wide={widgetId === 'tasks'}>
                        <WidgetComponent />
                    </div>
                {/each}
            </div>
        {/if}
        {#if settings.showLinks}
            <div class="links">
                <Links />
            </div>
        {/if}
    </div>

    <button
        class="settings-btn"
        class:needs-config={needsConfiguration}
        onclick={openSettings}
        aria-label="Open settings"
    >
        settings
    </button>

    <Settings {showSettings} {closeSettings} />
</main>

<style>
    main {
        position: relative;
        overflow: hidden;
        display: flex;
        flex-direction: column;
        min-height: 100vh;
        justify-content: flex-start;
        align-items: center;
        width: min(84rem, calc(100vw - 2rem));
        margin: 0 auto;
        padding: 1rem 1rem 2rem;
        isolation: isolate;
    }
    .command-row {
        width: 100%;
        margin-bottom: 1rem;
        z-index: 1;
    }
    .container {
        position: relative;
        z-index: 1;
        display: flex;
        flex-direction: column;
        width: 100%;
        gap: 1.5rem;
        opacity: 0;
    }
    .top,
    .widgets {
        display: flex;
        width: 100%;
        gap: 1.5rem;
    }
    .widget-slot {
        flex: 1 1 0;
        min-width: 0;
        display: flex;
    }
    .widget-slot.tasks-wide {
        flex: 1.4 1 0;
    }
    .links {
        width: 100%;
    }
    .top,
    .links {
        opacity: 0;
        transform: translate3d(0, 0.5rem, 0);
    }
    .widgets {
        opacity: 0;
    }
    .container.ready {
        opacity: 1;
    }
    .container.ready .top,
    .container.ready .links {
        opacity: 1;
        transform: translate3d(0, 0, 0);
        transition:
            opacity 220ms cubic-bezier(0.22, 1, 0.36, 1),
            transform 280ms cubic-bezier(0.22, 1, 0.36, 1);
    }
    .container.ready .widgets {
        opacity: 1;
        transition: opacity 180ms cubic-bezier(0.22, 1, 0.36, 1);
    }
    .container.ready .top {
        transition-delay: 30ms;
    }
    .container.ready .widgets {
        transition-delay: 70ms;
    }
    .container.ready .links {
        transition-delay: 110ms;
    }
    .widgets :global(.panel-wrapper) {
        flex: 1 1 auto;
        min-width: 0;
        width: 100%;
    }
    .top :global(.panel-wrapper) {
        flex: 1 1 0;
        min-width: 0;
    }
    .settings-btn {
        position: fixed;
        top: 0;
        right: 0;
        padding: 1rem 1.5rem;
        opacity: 0;
        z-index: 100;
        color: var(--txt-3);
    }
    .settings-btn:hover {
        opacity: 1;
    }
    .settings-btn.needs-config {
        opacity: 1;
        animation: pulse 1s ease-in-out infinite;
    }
    .settings-btn.needs-config:hover {
        opacity: 1;
        animation: none;
    }

    @media (prefers-reduced-motion: reduce) {
        .container,
        .top,
        .widgets,
        .links {
            opacity: 1;
            transform: none;
            transition: none;
        }
    }

    @media (max-width: 720px) {
        main {
            width: calc(100vw - 1rem);
            padding-top: 0.75rem;
        }
        .command-row {
            margin-bottom: 0.75rem;
        }
    }
</style>
