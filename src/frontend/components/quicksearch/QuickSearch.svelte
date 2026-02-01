<!-- Quickly find shows/elements globally in the program -->

<script lang="ts">
    import { fade } from "svelte/transition"
    import { quickSearchActive, special, theme, themes } from "../../stores"
    import { translateText } from "../../utils/language"
    import { formatSearch } from "../../utils/search"
    import { hexToRgb } from "../helpers/color"
    import Icon from "../helpers/Icon.svelte"
    import T from "../helpers/T.svelte"
    import Button from "../inputs/Button.svelte"
    import TextInput from "../inputs/TextInput.svelte"
    import Center from "../system/Center.svelte"
    import { quicksearch, selectQuicksearchValue } from "./quicksearch"

    let values: any[] = []
    let searchValue = ""
    let searchId = 0

    async function search(e: any) {
        searchValue = e.target.value
        const currentId = ++searchId

        // Clear values immediately if search is empty
        if (!searchValue) {
             values = []
             return
        }

        const results = await quicksearch(formatSearch(searchValue))

        // Prevent race conditions
        if (currentId !== searchId) return

        values = results
        selectedIndex = 0
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

    // let light = false
    // $: if ($theme) light = !isDarkTheme()

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
</script>

<svelte:window on:keydown={keydown} />

{#if $quickSearchActive}
    <div class="quicksearch" transition:fade={{ duration: 50 }}>
        <div class="box" style="--background: rgb({rgb.r} {rgb.g} {rgb.b} / 0.9);" class:isOptimized>
            <TextInput value={searchValue} placeholder={translateText("main.quick_search...")} style="padding: 10px 15px;font-size: 1.4em;" autofocus autoselect on:input={search} />

            {#if searchValue}
                {#if values.length}
                    <div class="values">
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
                                            <p class="description">{value.description}</p>
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
        top: 20%; /* Move it up a bit like spotlight */
        transform: translate(-50%, 0);
        /* max-width: calc(100% - var(--navigation-width) * 2); */
        width: 600px;
        max-width: 90%;

        z-index: 5001;
    }

    .box {
        display: flex;
        flex-direction: column;
        gap: 10px;

        background-color: var(--primary);
        border-radius: 12px;
        padding: 12px;

        box-shadow: 0 10px 30px rgb(0 0 0 / 0.5);
        border: 1px solid var(--primary-lighter);

        --background: rgba(35, 35, 45, 0.95);
        background-color: var(--background);
        backdrop-filter: blur(12px);
    }

    .box :global(input) {
        border-radius: 6px;
        border: none;
        background: transparent;
        color: var(--text);
    }
    .box :global(input):focus {
        outline: none;
    }

    .values {
        display: flex;
        flex-direction: column;
        max-height: 60vh;
        overflow-y: auto;
        padding-right: 5px; /* space for scrollbar */
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
</style>
