<svelte:options tag="fds-image-editor-sam"></svelte:options>
<script>
    import { onMount, tick } from "svelte"

    // needed otherwise it will be not availabe in bundle
    // eslint-disable-next-line no-unused-vars
    import Dialog from './Dialog.svelte'


  
    /**
     * the tool layer object
     * @type {object}
     */   
    export let tool_layer
    
    onMount(()=> {
        if (!tool_layer.points) tool_layer.points=[]
        if (!tool_layer.width) {
            tool_layer.width=200
            tool_layer.height=200
            tool_layer.x=20
            tool_layer.y=20         
            tool_layer.dragRestriction=true // limit to document size
            tool_layer.resizeRestriction=true // resize max to documennt size
            tool_layer.pointMode=false
        }
        refresh()
    })
    let canvas
    let canvasWidthZoomed,canvasHeightZoomed

    export function refresh() {
        if (!globalThis.gyre || !globalThis.gyre.canvas) return
        canvas=globalThis.gyre.canvas             // canvas API
        let dragDrop=globalThis.gyre.dragDrop   // Drag & Drop API        
        canvasWidthZoomed=canvas.calcPosition(canvas.width)        
        canvasHeightZoomed=canvas.calcPosition(canvas.height)  
        if (tool_layer.points) {
            for (let point of tool_layer.points) {
                point._x = canvas.calcPosition(point.x)
                point._y = canvas.calcPosition(point.y)
            }
        }      
        let l = tool_layer
        l._x = canvas.calcPosition(l.x) // rectangle
        l._y = canvas.calcPosition(l.y)
        l._width = canvas.calcPosition(l.width)
        l._height = canvas.calcPosition(l.height)
        if (l.element) {
            dragDrop.remove(l)
            dragDrop.makeDraggable(l)
            dragDrop.makeResizeAble(l)  
        }
        tool_layer.subTool=globalThis.gyre.selectedTool.subTool
        tool_layer=tool_layer  
    }
    let tmpMask
    let segImage
    let showProgress
    let formData={}
    export async function action(type) {
        if (!globalThis.gyre || !globalThis.gyre.canvas) return
        let gyre= globalThis.gyre
        if (!gyre.paletteValues.selectedLayer.url) return        
        if (type==="clear") {
            tool_layer.componentToolbar.showImageButtons=false
            tool_layer.componentToolbar.refresh()
            tool_layer.points=[]
            tmpMask=null
            refresh()
            return
        }
        if (type==="settings") {
            if (!gyre.openDialog) return;
            gyre.openDialog("fds-image-editor-sam box",formData,"Settings",(newData) => {formData=newData })
        }
        if (type==="setMask" && tmpMask) {      // set (inpainting) default mask
            gyre.maskManager.loadMask(tmpMask)
        }
        if (type==="loadImage" && segImage) {   // add new image layer here
            let newLayer = {
                type: 'image',
                name: 'AutoMatte',
                x: 0,
                y: 0,
                width: gyre.canvas.width,
                height: gyre.canvas.height,
                url: segImage
            }
            if (!gyre.layerManager) {
                alert("Only avalaible in main app")
            } else {
                gyre.layerManager.addLayer(newLayer)     
                gyre.refresh()
            }

        }
        if (type==="autoMatte") {   // execute ComfyUI workflow here
            showProgress=true
            await tick()
            let callBack_files = async (callbacktype,name,v2,v3,v4) => {    // callback for getting files from mappings

                console.log(callbacktype,name)
                if (callbacktype==="getLayerImage" && name==="currentLayer") {
                    return await gyre.imageAPI.urlToBase64(gyre.paletteValues.selectedLayer.url)
                }
                if (callbacktype==="getFile" && name==="json_file") {
                    let json
                    if (globalThis.gyre.selectedTool.subTool  !== 'points') {
                        json={boxes:[{x:tool_layer.x,y:tool_layer.y,w:tool_layer.width,h:tool_layer.height}]}
                    } else {
                        json={points:tool_layer.points}
                    }
                    return JSON.stringify(json)
                }                
            }
            let callback_error = async (nodeName) => {
                    console.log("Error at "+nodeName)
            }
            let workflowName 
            if (globalThis.gyre.selectedTool.subTool  !== 'points') workflowName = "fds-image-editor-sam box"
            else workflowName = "fds-image-editor-sam points"
            let data=gyre.ComfyUI.convertFormData(formData)
            data.currentLayer="empty"   // !important: set default empty values for files for calling callbacks
            data.json_file="empty"
            if (!data.erode_kernel_size) data.erode_kernel_size=3
            if (!data.dilate_kernel_size) data.dilate_kernel_size=3
            let callback_finished = async (result) => {
                        showProgress=false
                        let img=result[0].mime+";charset=utf-8;base64,"+result[0].base64
                        tmpMask= await gyre.imageAPI.setColorPreserveAlpha(img,255,0,0)
                        segImage=result[1].mime+";charset=utf-8;base64,"+result[1].base64
                        tool_layer.componentToolbar.showImageButtons=true
                        tool_layer.componentToolbar.refresh()
            }            
            await gyre.ComfyUI.executeWorkflow(workflowName, callback_finished, callBack_files,data,callback_error)

        }
       
    }
    function addLOIPoint(e) {
        if (globalThis.gyre.selectedTool.subTool  !== 'points') {
            refresh()
            return false
        }
        let coords=canvas.mouseCoords(e)
        let label=0
        if (!tool_layer.pointMode) label=1
        tool_layer.points.push({ x: parseInt(coords.x), y: parseInt(coords.y), label })
        refresh()
    }

    function deleteLOIPoint(e, i) {
        tool_layer.points.splice(i, 1)
        tool_layer = tool_layer
        e.preventDefault(true)
        e.stopPropagation(true)
        refresh()
    }

</script>
{#if canvasWidthZoomed}
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <div
        class="loiPointsBack"
        class:checkerBoard={tool_layer.previewResult}
        style="width:{canvasWidthZoomed}px;height:{canvasHeightZoomed}px;touch-action:pan-x pan-y;"
        on:click={addLOIPoint}
    >
    {#if showProgress}<fds-image-editor-progress-bar></fds-image-editor-progress-bar>{/if}
    {#if tmpMask && !tool_layer.previewResult}
        <!-- svelte-ignore a11y-missing-attribute -->
        <img src={tmpMask}  style="width:{canvasWidthZoomed}px;height:{canvasHeightZoomed}px;" class="mask" draggable="false">
    {/if}
    {#if segImage  && tool_layer.previewResult}
        <!-- svelte-ignore a11y-missing-attribute -->
        <img src={segImage}  style="width:{canvasWidthZoomed}px;height:{canvasHeightZoomed}px;"  draggable="false">
    {/if}
    {#if tool_layer.subTool !== 'points'}
    <div class="selectionOuter" bind:this={tool_layer.element}
        style="left:{tool_layer._x}px;top:{tool_layer._y}px;width:{tool_layer._width}px;height:{tool_layer._height}px;z-index:{200}"
    >
        <div class="selectionInner selectionRect"></div>
    </div>
    {:else}
        {#each tool_layer.points as point, i}
            <!-- svelte-ignore a11y-no-static-element-interactions -->
            <div
                class="loiPoint"
                class:loiPointBackground={point.label === 0}
                style="left:{point._x}px;top:{point._y}px;z-index:200"
                on:click={(e) => {
                    deleteLOIPoint(e, i)
                }}
            />
        {/each}    
    {/if}
    </div>
{/if}

<style>
    .selectionOuter {
        display: block;
        position: absolute;
        border: none;
        /*border-bottom: 5px solid transparent;
        border-right: 5px solid transparent;*/
        opacity: 1;
        touch-action: none;
    }
    .selectionInner {
        border: 1px solid white;
        display: block;
        position: absolute;
        top: 0;
        left: 0;
        bottom: 0;
        right: 0;
        opacity: 0.8;
        --angle: 0deg;
        border: 2px solid;
        border-image: conic-gradient(from var(--angle), red, yellow, lime, aqua, blue, magenta, red) 1;
        /* border-image: conic-gradient(from var(--angle), #39b0b0, #10db91, #2b7777, aqua, #27baba ) 1; */
        animation: 10s rotate linear infinite;
    }    
  .selectionRect {
        border-image: conic-gradient(
                from var(--angle),
                rgb(216, 216, 216),
                rgb(255, 255, 255),
                rgb(203, 203, 203),
                rgb(180, 180, 180),
                rgb(101, 101, 101),
                rgb(106, 106, 106),
                rgb(129, 129, 129)
            )
            1;
    }
    .loiPoint {
        border-radius: 100%;
        border: 1px solid white;
        width: 20px;
        height: 20px;
        position: absolute;
        margin-left: -10px;
        margin-top: -10px;
        background-color: green;
        cursor: not-allowed;
    }
    .loiPointBackground {
        background-color: red;
    }
    .loiPointsBack {
        position: absolute;
        left: 0;
        top: 0;
    }
    .loiPoint:hover {
        transition: 500ms ease-out;
        background-color: #000;
    }

    .mask {
        opacity: 0.5;
    }

    .checkerBoard {
        background-repeat: repeat;
        background-size: 20px;
        background-image: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEAAAABACAYAAACqaXHeAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAAFiUAABYlAUlSJPAAAACYSURBVHhe7dqxEcAgEMRAcEv03wHUZDtQDwTSJvr4ZsiY729cdM7huuOhWg1AtRqAajUA1WoAqtUAVKsBqFYDUK0GoFoNQLUagGo1ANVqAKrVAFSrAahWA1Ctufe++j9grcV1R0+AajUA1WoAqtUAVKsBqFYDUK0GoFoNQLUagGo1ANVqAKrVAFSrAahWA1CtBqBa8gHG+AAQDQreoC53nAAAAABJRU5ErkJggg==);
    }
</style>
