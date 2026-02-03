<!-- Quickly find shows/elements globally in the program -->

<script lang="ts">
    import { fade, fly } from "svelte/transition"
    import { quickSearchActive, special, theme, themes } from "../../stores"
    import { translateText } from "../../utils/language"
    import { formatSearch } from "../../utils/search"
    import { hexToRgb } from "../helpers/color"
    import Icon from "../helpers/Icon.svelte"
    import T from "../helpers/T.svelte"
    import Button from "../inputs/Button.svelte"
    import TextInput from "../inputs/TextInput.svelte"
    import Center from "../system/Center.svelte"
    import { quicksearch, selectQuicksearchValue, type SearchCategory } from "./quicksearch"

    let values: any[] = []
    let searchValue = ""
    let searchId = 0
    let activeCategory: SearchCategory = "all"

    const categories: { id: SearchCategory; label: string; icon: string }[] = [
        { id: "all", label: "All", icon: "search" },
        { id: "songs", label: "Songs", icon: "slide" },
        { id: "bible", label: "Bible", icon: "scripture" },
        { id: "media", label: "Media", icon: "media" },
        { id: "settings", label: "Settings", icon: "settings" }
    ]

    async function search(e: any) {
        searchValue = e.target.value
        const currentId = ++searchId

        // Clear values immediately if search is empty
        if (!searchValue) {
             values = []
             return
        }

        const results = await quicksearch(formatSearch(searchValue), searchValue, activeCategory)

        // Prevent race conditions
        if (currentId !== searchId) return

        values = results
        selectedIndex = 0
    }

    function selectCategory(id: SearchCategory) {
        activeCategory = id
        // Re-run search with new category
        if (searchValue) {
            search({ target: { value: searchValue } })
        }
    }

    let selectedIndex = 0

    function keydown(e: KeyboardEvent) {
        // CTRL + G or F8
        if (((e.ctrlKey || e.metaKey) && e.key === "g") || e.key === "F8") {
            // toggle quick search
            quickSearchActive.set(!$quickSearchActive)
            return
        }

        if (!$quickSearchActive || !values.length) return

        if (e.key === "Enter") {
            selectQuicksearchValue(values[selectedIndex], e.ctrlKey || e.metaKey)
            selectedIndex = 0
        } else if (e.key === "ArrowDown") {
             e.preventDefault()
             selectedIndex = Math.min(values.length - 1, selectedIndex + 1)
        } else if (e.key === "ArrowUp") {
             e.preventDefault()
             selectedIndex = Math.max(0, selectedIndex - 1)
        }
    }

    let rgb = { r: 35, g: 35, b: 45 }
    $: if ($theme) updateColor()
    function updateColor() {
        const color = $themes[$theme]?.colors["primary"]
        if (!color) return

        const newRgb = hexToRgb(color)
        rgb = { r: Math.max(0, newRgb.r - 1), g: Math.max(0, newRgb.g - 5), b: Math.max(0, newRgb.b - 5) }
    }

    $: isOptimized = $special.optimizedMode

    // Auto-scroll to selected item
    $: if (values.length && selectedIndex >= 0) {
        const selectedEl = document.getElementById(`qs-item-${selectedIndex}`)
        if (selectedEl) selectedEl.scrollIntoView({ block: 'nearest' })
    }

    // Function to highlight matching text
    function highlightMatch(text: string, search: string): string {
        if (!search || !text) return text

        // Try strict match first
        const escapedSearch = search.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
        const strictRegex = new RegExp(`(${escapedSearch})`, 'gi')
        if (strictRegex.test(text)) {
            return text.replace(strictRegex, '<mark>$1</mark>')
        }

        // Try tokenized match (words >= 3 chars)
        const words = search.split(/\s+/).filter(w => w.length >= 3)
        if (!words.length) return text

        // Group words for a single regex to avoid overlapping <mark> tags
        const pattern = words
            .sort((a, b) => b.length - a.length) // Longest words first
            .map(w => w.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'))
            .join('|')

        const tokenRegex = new RegExp(`(${pattern})`, 'gi')
        return text.replace(tokenRegex, '<mark>$1</mark>')
    }
</script>

<svelte:window on:keydown={keydown} />

{#if $quickSearchActive}
    <div class="quicksearch" transition:fade={{ duration: 50 }}>
        <div class="box" style="--background: rgb({rgb.r} {rgb.g} {rgb.b} / 0.9);" class:isOptimized>
            <TextInput value={searchValue} placeholder={translateText("main.quick_search...")} style="padding: 10px 15px;font-size: 1.4em;" autofocus autoselect on:input={search} />

            <!-- Glowing Indicator -->
            {#if !searchValue}
                <div class="indicator" transition:fade={{ duration: 100 }}>
                    <span class="glow-dot"></span>
                    <span class="indicator-text">{translateText("main.search_anything")}</span>
                </div>
            {/if}

            <!-- Category Filter Bar -->
            <div class="category-bar" transition:fly={{ y: -10, duration: 150 }}>
                {#each categories as cat}
                    <button
                        class="category-btn"
                        class:active={activeCategory === cat.id}
                        on:click={() => selectCategory(cat.id)}
                    >
                        <Icon id={cat.icon} size={0.9} />
                        <span>{cat.label}</span>
                    </button>
                {/each}
            </div>

            {#if searchValue}
                {#if values.length}
                    <div class="values" transition:fly={{ y: 10, duration: 150 }}>
                        {#each values as value, i}
                            {#if i === 0 || values[i - 1].category !== value.category}
                                <div class="category-header">
                                    {value.category}
                                </div>
                            {/if}

                            <div id="qs-item-{i}">
                                <Button
                                    style="gap: 10px;font-size: 1em;color: {value.color || 'unset'};"
                                    active={i === selectedIndex}
                                    on:click={(e) => selectQuicksearchValue(value, e.ctrlKey || e.metaKey)}
                                    bold={false}
                                >
                                    <Icon id={value.icon || value.type} />
                                    <div class="item-text" data-title={value.name}>
                                        <p>
                                            {value.name}

                                            {#if value.aliasMatch && !value.aliasMatch.startsWith("-")}
                                                <span style="opacity: 0.5;font-style: italic;margin-left: 5px;font-size: 0.8em;">{value.aliasMatch}</span>
                                            {/if}

                                            {#if value.id.includes("http")}
                                                <Icon id="launch" size={0.8} white />
                                            {/if}
                                        </p>
                                        {#if value.description}
                                            <p class="description">{@html highlightMatch(value.description, searchValue)}</p>
                                        {/if}
                                    </div>
                                </Button>
                            </div>
                        {/each}
                    </div>
                {:else}
                    <Center faded>
                        <T id="empty.search" />
                    </Center>
                {/if}
            {/if}
        </div>
    </div>
{/if}

<style>
    .quicksearch {
        position: absolute;
        left: 50%;
        top: 18%;
        transform: translate(-50%, 0);
        width: 650px;
        max-width: 90%;

        z-index: 5001;
    }

    .box {
        display: flex;
        flex-direction: column;
        gap: 10px;

        background-color: var(--primary);
        border-radius: 16px;
        padding: 14px;

        box-shadow: 0 12px 40px rgb(0 0 0 / 0.6), 0 0 0 1px rgba(255, 255, 255, 0.05) inset;
        border: 1px solid var(--primary-lighter);

        --background: rgba(35, 35, 45, 0.95);
        background-color: var(--background);
        backdrop-filter: blur(16px);
    }

    .box :global(input) {
        border-radius: 8px;
        border: none;
        background: rgba(255, 255, 255, 0.05);
        color: var(--text);
        transition: background 0.2s ease;
    }
    .box :global(input):focus {
        outline: none;
        background: rgba(255, 255, 255, 0.08);
    }

    /* Glowing Indicator */
    .indicator {
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 8px;
        padding: 8px 0;
        opacity: 0.6;
    }
    .glow-dot {
        width: 8px;
        height: 8px;
        border-radius: 50%;
        background: linear-gradient(135deg, #4ade80, #22d3ee);
        box-shadow: 0 0 8px 2px rgba(74, 222, 128, 0.5);
        animation: pulse 2s ease-in-out infinite;
    }
    @keyframes pulse {
        0%, 100% { opacity: 0.7; transform: scale(1); }
        50% { opacity: 1; transform: scale(1.1); }
    }
    .indicator-text {
        font-size: 0.85em;
        color: var(--text);
        opacity: 0.7;
    }

    /* Category Filter Bar */
    .category-bar {
        display: flex;
        gap: 6px;
        padding: 4px 0;
        border-bottom: 1px solid rgba(255, 255, 255, 0.08);
    }
    .category-btn {
        display: flex;
        align-items: center;
        gap: 5px;
        padding: 6px 12px;
        border: none;
        border-radius: 8px;
        background: transparent;
        color: var(--text);
        opacity: 0.6;
        cursor: pointer;
        font-size: 0.85em;
        transition: all 0.15s ease;
    }
    .category-btn:hover {
        background: rgba(255, 255, 255, 0.08);
        opacity: 0.9;
    }
    .category-btn.active {
        background: rgba(255, 255, 255, 0.12);
        opacity: 1;
    }

    .values {
        display: flex;
        flex-direction: column;
        max-height: 55vh;
        overflow-y: auto;
        padding-right: 5px;
    }

    .category-header {
        font-size: 0.75em;
        font-weight: bold;
        text-transform: uppercase;
        color: var(--text);
        opacity: 0.5;
        padding: 10px 10px 5px 10px;
        margin-top: 5px;
        border-bottom: 1px solid var(--primary-lighter);
    }
    .category-header:first-child {
        margin-top: 0;
    }

    .item-text {
        display: flex;
        flex-direction: column;
        align-items: flex-start;
        flex: 1;
        overflow: hidden;
    }
    .item-text p {
        width: 100%;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
        text-align: left;
    }
    .description {
        font-size: 0.8em;
        opacity: 0.6;
    }
    .description :global(mark) {
        background: rgba(250, 204, 21, 0.4);
        color: inherit;
        padding: 0 2px;
        border-radius: 2px;
    }
</style>
