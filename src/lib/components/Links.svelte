<script>
    import { settings } from '../stores/settings-store.svelte.js'
    import { isValidSlug } from '../utils/link-icons.js'

    const columns = $derived.by(() => {
        const result = []
        const linksPerColumn = Math.max(
            1,
            parseInt(settings.linksPerColumn) || 1
        )
        for (let i = 0; i < settings.links.length; i += linksPerColumn) {
            result.push(settings.links.slice(i, i + linksPerColumn))
        }
        return result
    })
</script>

<div class="panel-wrapper">
    <div class="panel-label">links</div>
    <div class="panel">
        {#each columns as column}
            <div
                class="column"
                class:icon-mode={settings.linkIconMode === 'icons'}
            >
                {#each column as link}
                    <a
                        href={link.url}
                        target={settings.linkTarget}
                        rel="noopener noreferrer"
                        class="link"
                    >
                        {#if settings.linkIconMode === 'icons' && isValidSlug(link.icon)}
                            <span class="icon si si-{link.icon}"></span>
                        {:else}
                            <span class="prefix">></span>
                        {/if}
                        {link.title}
                    </a>
                    <br />
                {/each}
            </div>
        {/each}
    </div>
</div>

<style>
    .panel {
        display: flex;
        flex-wrap: wrap;
        gap: 1.5rem;
    }
    .link:hover .prefix,
    .link:hover .icon {
        color: var(--txt-2);
    }
    .prefix,
    .icon {
        display: inline-block;
        color: var(--txt-3);
        text-align: center;
    }
    .icon-mode .prefix,
    .icon {
        width: 1.25rem;
    }
    .icon {
        font-size: 0.875rem;
        vertical-align: text-bottom;
    }
    .column {
        flex-grow: 1;
    }
</style>
