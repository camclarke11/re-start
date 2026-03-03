<script>
    import { onMount, onDestroy } from 'svelte'
    import { settings } from '../stores/settings-store.svelte.js'

    let currentHrs = $state('')
    let currentMin = $state('')
    let currentSec = $state('')
    let currentAmPm = $state('')
    let currentDate = $state('')
    let currentQuoteIndex = $state(0)
    let quoteDateKey = $state('')
    let quoteGlowX = $state(0)
    let quoteGlowY = $state(0)
    let quoteGlowActive = $state(false)
    let clockInterval = null
    const motivationalQuotes = [
        {
            text: 'The secret of getting ahead is getting started.',
            author: 'Mark Twain',
        },
        {
            text: 'Inspiration is for amateurs; the rest of us just show up and get to work.',
            author: 'Chuck Close',
        },
        {
            text: "You can't build a reputation on what you are going to do.",
            author: 'Henry Ford',
        },
        {
            text: "If you spend too much time thinking about a thing, you'll never get it done.",
            author: 'Bruce Lee',
        },
        {
            text: 'Discipline equals freedom.',
            author: 'Jocko Willink',
        },
        {
            text: 'Action is the foundational key to all success.',
            author: 'Pablo Picasso',
        },
        {
            text: 'Do what you can, with what you have, where you are.',
            author: 'Theodore Roosevelt',
        },
        {
            text: 'Someday is not a day of the week.',
            author: 'Janet Dailey',
        },
        {
            text: "You don't have to be great to start, but you have to start to be great.",
            author: 'Zig Ziglar',
        },
        {
            text: 'If it is important to you, you will find a way. If not, you will find an excuse.',
            author: 'Ryan Blair',
        },
        {
            text: 'The most effective way to do it, is to do it.',
            author: 'Amelia Earhart',
        },
        {
            text: "Don't watch the clock; do what it does. Keep going.",
            author: 'Sam Levenson',
        },
    ]

    function getDateKey(date) {
        const y = date.getFullYear()
        const m = String(date.getMonth() + 1).padStart(2, '0')
        const d = String(date.getDate()).padStart(2, '0')
        return `${y}-${m}-${d}`
    }

    function quoteIndexForDateKey(dateKey) {
        let hash = 0
        for (let i = 0; i < dateKey.length; i++) {
            hash = (hash * 31 + dateKey.charCodeAt(i)) >>> 0
        }
        return hash % motivationalQuotes.length
    }

    function updateQuoteForDate(date) {
        const dateKey = getDateKey(date)
        if (dateKey === quoteDateKey) return
        quoteDateKey = dateKey
        currentQuoteIndex = quoteIndexForDateKey(dateKey)
    }

    function showNextQuote() {
        currentQuoteIndex =
            (currentQuoteIndex + 1) % motivationalQuotes.length
        quoteDateKey = getDateKey(new Date())
    }

    function handleQuotePointerMove(event) {
        const targetRect = event.currentTarget.getBoundingClientRect()
        quoteGlowX = event.clientX - targetRect.left
        quoteGlowY = event.clientY - targetRect.top
        quoteGlowActive = true
    }

    function handleQuotePointerLeave() {
        quoteGlowActive = false
    }

    function updateTime() {
        const now = new Date()

        let hours = now.getHours()

        if (settings.timeFormat === '12hr') {
            currentAmPm = hours >= 12 ? 'pm' : 'am'
            hours = hours % 12
            if (hours === 0) hours = 12
        } else {
            currentAmPm = ''
        }

        currentHrs = hours.toString().padStart(2, '0')
        currentMin = now.getMinutes().toString().padStart(2, '0')
        currentSec = now.getSeconds().toString().padStart(2, '0')

        const locale = settings.dateFormat === 'dmy' ? 'en-GB' : 'en-US'
        currentDate = new Intl.DateTimeFormat(locale, {
            weekday: 'long',
            year: 'numeric',
            month: 'long',
            day: 'numeric',
        })
            .format(now)
            .toLowerCase()

        if (settings.showMotivationalQuote) {
            updateQuoteForDate(now)
        }
    }

    function startClock() {
        updateTime()

        const now = new Date()
        const msUntilNextSecond = 1000 - now.getMilliseconds()

        setTimeout(() => {
            updateTime()
            clockInterval = setInterval(updateTime, 1000)
        }, msUntilNextSecond)
    }

    function handleVisibilityChange() {
        if (document.visibilityState === 'visible') {
            startClock()
        } else {
            if (clockInterval) {
                clearInterval(clockInterval)
                clockInterval = null
            }
        }
    }

    onMount(() => {
        startClock()
        document.addEventListener('visibilitychange', handleVisibilityChange)
    })

    onDestroy(() => {
        if (clockInterval) {
            clearInterval(clockInterval)
        }
    })
</script>

<div class="panel-wrapper">
    <div class="panel-label">datetime</div>
    <div class="panel">
        <div class="datetime-main">
            <div class="datetime-left">
                <div class="clock">
                    {currentHrs}<span class="colon">:</span>{currentMin}<span
                        class="colon">:</span
                    >{currentSec}
                    {#if settings.timeFormat === '12hr'}
                        <span class="ampm">{currentAmPm}</span>
                    {/if}
                </div>
                <div class="date">{currentDate}</div>
            </div>
            {#if settings.showMotivationalQuote}
                <button
                    class="quote"
                    onclick={showNextQuote}
                    title="show another quote"
                    aria-label="show another quote"
                    onpointermove={handleQuotePointerMove}
                    onpointerenter={handleQuotePointerMove}
                    onpointerleave={handleQuotePointerLeave}
                    style="--qx: {quoteGlowX}px; --qy: {quoteGlowY}px; --q-opacity: {quoteGlowActive
                        ? '1'
                        : '0'}"
                >
                    <span class="quote-text">
                        <span class="quote-base"
                            >"{motivationalQuotes[currentQuoteIndex].text}"</span
                        >
                        <span class="quote-glow" aria-hidden="true"
                            >"{motivationalQuotes[currentQuoteIndex].text}"</span
                        >
                    </span>
                    <span class="quote-author"
                        >- {motivationalQuotes[currentQuoteIndex].author}</span
                    >
                </button>
            {/if}
        </div>
    </div>
</div>

<style>
    .panel-wrapper {
        flex-grow: 1;
    }
    .datetime-main {
        display: flex;
        justify-content: space-between;
        gap: 1.5rem;
        align-items: center;
    }
    .datetime-left {
        min-width: 0;
    }
    .clock {
        font-size: 3.125rem;
        font-weight: 300;
        color: var(--txt-1);
        line-height: 3.5rem;
        margin: 0 0 0.5rem 0;
    }
    .colon,
    .ampm {
        color: var(--txt-2);
    }
    .date {
        font-size: 1.5rem;
        color: var(--txt-3);
        line-height: 2rem;
        margin: 0;
    }
    .quote {
        color: var(--txt-3);
        line-height: 1.36;
        text-align: right;
        max-width: min(24rem, 42vw);
        align-self: stretch;
        justify-content: center;
        padding: 0.1rem 0;
        position: relative;
        display: inline-flex;
        flex-direction: column;
        align-items: flex-end;
        gap: 0.38rem;
        isolation: auto;
        transition:
            transform 180ms ease-out,
            filter 180ms ease-out;
    }
    .quote-text {
        display: inline-grid;
        width: auto;
        max-width: 100%;
    }
    .quote-base,
    .quote-glow {
        grid-area: 1 / 1;
        white-space: normal;
        text-align: inherit;
    }
    .quote-base {
        color: var(--txt-3);
        transition: color 140ms ease-out;
    }
    .quote-glow {
        color: color-mix(in oklch, var(--txt-1) 88%, var(--txt-2) 12%);
        opacity: var(--q-opacity, 0);
        pointer-events: none;
        text-shadow:
            0 0 0.39rem color-mix(in oklch, var(--txt-1) 46%, transparent),
            0 0 0.94rem color-mix(in oklch, var(--txt-2) 31%, transparent);
        -webkit-mask-image: radial-gradient(
            3.6rem circle at var(--qx, 50%) var(--qy, 50%),
            rgb(0 0 0 / 90%) 0%,
            rgb(0 0 0 / 22%) 45%,
            transparent 78%
        );
        mask-image: radial-gradient(
            3.6rem circle at var(--qx, 50%) var(--qy, 50%),
            rgb(0 0 0 / 90%) 0%,
            rgb(0 0 0 / 22%) 45%,
            transparent 78%
        );
        transition: opacity 140ms ease-out;
    }
    .quote:hover .quote-base {
        color: var(--txt-2);
    }
    .quote-author {
        display: inline-flex;
        align-items: center;
        gap: 0.42rem;
        align-self: flex-end;
        font-size: 0.74rem;
        letter-spacing: 0.08em;
        text-transform: uppercase;
        color: color-mix(in oklch, var(--txt-2) 78%, var(--txt-3) 22%);
        padding-top: 0.16rem;
        line-height: 1.1;
        opacity: 0.92;
        transition: color 140ms ease-out;
    }
    .quote-author::before {
        content: '';
        width: 1.45rem;
        height: 1px;
        background: color-mix(in oklch, var(--txt-2) 35%, transparent);
        transform: translateY(-0.5px);
    }
    .quote:hover .quote-author {
        color: color-mix(in oklch, var(--txt-1) 66%, var(--txt-2) 34%);
    }
    .quote:hover {
        transform: perspective(28rem) translateY(-0.5px);
        filter: saturate(1.04);
    }
    .quote-base::selection,
    .quote-glow::selection {
        background: color-mix(in oklch, var(--txt-2) 22%, transparent);
        color: var(--txt-1);
    }
    @media (prefers-reduced-motion: reduce) {
        .quote {
            transition: none;
        }
        .quote:hover {
            transform: none;
            filter: none;
        }
    }
    @media (max-width: 62rem) {
        .datetime-main {
            flex-direction: column;
        }
        .quote {
            text-align: left;
            align-self: flex-start;
            align-items: flex-start;
            max-width: min(30rem, 100%);
        }
        .quote-author {
            align-self: flex-start;
        }
    }
</style>
