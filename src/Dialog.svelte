<svelte:options tag="fds-image-editor-sam-floating-toolbar"></svelte:options>
<script>

/**
 * the tool layer object
 * @type {object}
 */   
export let tool_layer
export let showImageButtons=false
export function refresh() {
    tool_layer=tool_layer
}
</script>
{#if globalThis.gyre.iPad && tool_layer}

<div class="iPadTool">
    <div class="iPadToolPallete">
        {#if tool_layer.subTool  === 'points'}
            {#if tool_layer.pointMode === 0}
                <!-- svelte-ignore a11y-click-events-have-key-events -->
                <fds-image-editor-button
                    icon="background"
                    title="Define Background"
                    class="iconDark"
                    style="margin-bottom: 10px"
                    on:click={() => {
                        tool_layer.pointMode = true
                        tool_layer.component.refresh()
                    }}
                />
            {:else}
                <!-- svelte-ignore a11y-click-events-have-key-events -->
                <fds-image-editor-button
                    icon="foreground"
                    title="Define Foreground"
                    class="iconDark"
                    style="margin-bottom: 10px"
                    on:click={() => {
                        tool_layer.pointMode = false
                        tool_layer.component.refresh()
                    }}
                />
            {/if}
        {/if}


        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <fds-image-editor-button
            icon="AutoMask"
            title="Generate Matte automatically"
            class="iconDark"
            style="margin-top:10px"
            on:click={() => {
                tool_layer.component.action('autoMatte')
            }}
        />
    </div>
</div>
{:else if tool_layer}
<div class="aiSelection">
    {#if tool_layer.subTool === 'points'}
        <fds-image-editor-toggle on:change={(e) => {
            tool_layer.pointMode= e.detail
            tool_layer.component.refresh()
        }} value={tool_layer.pointMode} name="bg"
        class="toggle"
          />
            <label for="bg">Background</label>
        |
    {/if}

    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <fds-image-editor-button
        style="margin-left:10px"
        type="button"
        on:click={() => {
            tool_layer.component.action('autoMatte')
        }}
    >
        Auto Matte
    </fds-image-editor-button>
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <fds-image-editor-button
        icon="fds-image-editor-sam-eye"
        title="Select by Points"
        class="iconDark"
        style="vertical-align: -10px;display: inline-block; height: 40px"
        on:click={() => {
            tool_layer.previewResult = !tool_layer.previewResult
            tool_layer.component.refresh()
        }}
        state={tool_layer.previewResult ? 'active' : ''}
    />
    {#if showImageButtons}
       <!-- svelte-ignore a11y-click-events-have-key-events -->
       <fds-image-editor-button
       style="margin-left:10px"
       type="button"
       on:click={() => {
           tool_layer.component.action('loadImage')
       }}
   >
       New Layer
   </fds-image-editor-button>     
       <!-- svelte-ignore a11y-click-events-have-key-events -->
       <fds-image-editor-button
       style="margin-left:10px"
       type="button"
       on:click={() => {
           tool_layer.component.action('setMask')
       }}
   >
       Set Mask
   </fds-image-editor-button>        
    {/if}
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <fds-image-editor-button
        style="margin-left:10px"
        type="button"
        on:click={() => {
            tool_layer.component.action('clear')
        }}
    >
        Clear
    </fds-image-editor-button>    

<!-- svelte-ignore a11y-click-events-have-key-events -->
<fds-image-editor-button
    icon="moreSettings"
    title="Settings"
    class="iconDark"
    style="vertical-align: -10px;display: inline-block; height: 40px"
    on:click={() => {
        tool_layer.component.action('settings')
    }}
/>
</div>
{/if}
<style>
    .toggle {
        vertical-align: middle;
    }

</style>
