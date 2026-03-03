<script>
    import { onDestroy, onMount } from 'svelte'
    import { settings } from '../stores/settings-store.svelte.js'

    let value = $state('')
    let feedback = $state('')
    let inputEl = $state(null)
    let feedbackTimeoutId = null

    function showFeedback(message) {
        feedback = message
        if (feedbackTimeoutId) {
            clearTimeout(feedbackTimeoutId)
        }
        feedbackTimeoutId = setTimeout(() => {
            feedback = ''
            feedbackTimeoutId = null
        }, 2600)
    }

    function focusInput() {
        inputEl?.focus()
        inputEl?.select()
    }

    function buildChatGptUrl(query) {
        const params = new URLSearchParams()
        params.set('restart_autosend', '1')
        params.set('restart_request', `${Date.now()}-${Math.random()}`)
        params.set('restart_prompt', query)
        return `https://chatgpt.com/#${params.toString()}`
    }

    function buildGoogleUrl(query) {
        const params = new URLSearchParams({ q: query })
        return `https://www.google.com/search?${params.toString()}`
    }

    function buildGeminiUrl(query = '') {
        if (!query) return 'https://gemini.google.com/app'
        const params = new URLSearchParams({ q: query })
        return `https://gemini.google.com/app?${params.toString()}`
    }

    function buildYouTubeUrl(query) {
        const params = new URLSearchParams({ search_query: query })
        return `https://www.youtube.com/results?${params.toString()}`
    }

    function buildRedditUrl(query) {
        const params = new URLSearchParams({ q: query })
        return `https://www.reddit.com/search/?${params.toString()}`
    }

    function buildGithubUrl(query) {
        const params = new URLSearchParams({ q: query, type: 'repositories' })
        return `https://github.com/search?${params.toString()}`
    }

    function buildWikipediaUrl(query) {
        const params = new URLSearchParams({ search: query })
        return `https://en.wikipedia.org/w/index.php?${params.toString()}`
    }

    function normalizeUrl(url) {
        if (/^https?:\/\//i.test(url)) {
            return url
        }
        return `https://${url}`
    }

    function parseCommand(rawInput) {
        const input = rawInput.trim()
        if (!input) {
            return { error: 'type /h for commands or enter a search query' }
        }

        if (!input.startsWith('/')) {
            return {
                url: buildGoogleUrl(input),
                success: 'opened web search',
            }
        }

        const [command, ...rest] = input.split(/\s+/)
        const arg = rest.join(' ').trim()

        switch (command.toLowerCase()) {
            case '/c':
                if (!arg) return { error: 'add a prompt after /c' }
                return {
                    url: buildChatGptUrl(arg),
                    success: 'opened ChatGPT and auto-sent',
                }
            case '/g':
                return {
                    url: buildGeminiUrl(arg),
                    success: arg ? 'opened Gemini' : 'opened Gemini home',
                }
            case '/yt':
                if (!arg) return { error: 'add a query after /yt' }
                return { url: buildYouTubeUrl(arg), success: 'opened YouTube search' }
            case '/r':
                if (!arg) return { error: 'add a query after /r' }
                return { url: buildRedditUrl(arg), success: 'opened Reddit search' }
            case '/gh':
                if (!arg) return { error: 'add a query after /gh' }
                return { url: buildGithubUrl(arg), success: 'opened GitHub search' }
            case '/w':
                if (!arg) return { error: 'add a query after /w' }
                return {
                    url: buildWikipediaUrl(arg),
                    success: 'opened Wikipedia search',
                }
            case '/go':
                if (!arg) return { error: 'add a URL after /go' }
                return { url: normalizeUrl(arg), success: 'opened URL' }
            case '/h':
            case '/?':
                return {
                    help: true,
                    success: 'commands: /c /g /yt /r /gh /w /go | plain text = web',
                }
            default:
                return { error: 'unknown command, use /h' }
        }
    }

    function openUrl(url) {
        if (settings.linkTarget === '_self') {
            window.location.href = url
            return true
        }
        const popup = window.open(url, '_blank', 'noopener,noreferrer')
        return Boolean(popup)
    }

    function handleSubmit(event) {
        event.preventDefault()
        const submittedInput = value
        value = ''

        const parsed = parseCommand(submittedInput)
        if (parsed.error) {
            showFeedback(parsed.error)
            focusInput()
            return
        }

        if (parsed.help) {
            showFeedback(parsed.success)
            focusInput()
            return
        }

        const opened = openUrl(parsed.url)
        if (!opened) {
            showFeedback('popup blocked: allow popups for this page')
            focusInput()
            return
        }

        showFeedback(parsed.success)
    }

    function handleGlobalKeydown(event) {
        if (!(event.ctrlKey || event.metaKey)) return
        if (event.key.toLowerCase() !== 'k') return

        const activeElement = document.activeElement
        const isTypingTarget =
            activeElement?.tagName === 'INPUT' ||
            activeElement?.tagName === 'TEXTAREA' ||
            (activeElement instanceof HTMLElement &&
                activeElement.isContentEditable)

        if (isTypingTarget) return
        event.preventDefault()
        focusInput()
    }

    onMount(() => {
        document.addEventListener('keydown', handleGlobalKeydown)
    })

    onDestroy(() => {
        document.removeEventListener('keydown', handleGlobalKeydown)
        if (feedbackTimeoutId) {
            clearTimeout(feedbackTimeoutId)
            feedbackTimeoutId = null
        }
    })
</script>

<div class="command-bar">
    <form class="launcher" onsubmit={handleSubmit}>
        <span class="launcher-prefix">&gt;</span>
        <input
            bind:this={inputEl}
            bind:value
            class="launcher-input"
            type="text"
            placeholder="/c ask chatgpt | /g ask gemini | (ctrl+k focus)"
            aria-label="Command launcher"
        />
        <button type="submit" class="launcher-open">open</button>
    </form>
    {#if feedback}
        <div class="launcher-feedback">{feedback}</div>
    {/if}
</div>

<style>
    .command-bar {
        width: 100%;
    }
    .launcher {
        width: 100%;
        display: flex;
        align-items: center;
        gap: 0.5rem;
        border: 2px solid var(--bg-3);
        background: var(--bg-2);
        padding: 0.25rem 0.5rem;
    }
    .launcher-prefix {
        color: var(--txt-3);
        flex: 0 0 auto;
    }
    .launcher-input {
        flex: 1;
        min-width: 0;
        background: transparent;
        border: none;
        padding: 0;
    }
    .launcher-input:focus {
        outline: none;
    }
    .launcher-input::placeholder {
        color: var(--txt-4);
    }
    .launcher-open {
        color: var(--txt-2);
        flex: 0 0 auto;
    }
    .launcher-feedback {
        color: var(--txt-3);
        margin-top: 0.25rem;
    }
</style>
