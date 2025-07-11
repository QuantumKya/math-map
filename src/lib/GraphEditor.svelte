<script lang="ts">
    import { onMount } from 'svelte';
    import { base } from '$app/paths';
    
    import NodeEditor from './NodeEditor.svelte';
    import ModePicker from './ModePicker.svelte';

    import Victor from 'victor';
    import data from './nodes.json';

    type MNode = { "id": number, "name": string, "description": string, "image": string, "position": number[] };

    let canvas: HTMLCanvasElement;

    let translation: Victor = new Victor(0, 0);
    let zoom: number = 1;

    let draggingCanvas: boolean = false;
    let lastMousePos: Victor = new Victor(0, 0);

    let dragOffset: Victor = new Victor(0, 0);

    let node_data = data["nodes"];
    let link_data = data["links"];

    let imageBuffer: HTMLImageElement[] = [];

    let focusNode: MNode = $state(node_data[0]);
    let draggingNode: boolean = false;
    let editingNode: boolean = $state(false);

    const draw = (ctx: CanvasRenderingContext2D) => {
        ctx.clearRect(0, 0, canvas.width, canvas.height);

        link_data.forEach((link) => {
            ctx.beginPath();
            const node1 = node_data[link["from"]];
            const node2 = node_data[link["to"]];
            ctx.moveTo((node1.position[0] - translation.x) / zoom, (node1.position[1] - translation.y) / zoom);
            ctx.lineTo((node2.position[0] - translation.x) / zoom, (node2.position[1] - translation.y) / zoom);
            ctx.lineWidth = 7.5;
            ctx.strokeStyle = "black";
            ctx.stroke();
        });

        node_data.forEach((node, index) => {
            imageBuffer[index].src = node.image;
            
            const pos = Victor.fromArray(node.position).subtract(translation).divideScalar(zoom);
            const imgWidth = 25 / zoom;

            ctx.beginPath();
            ctx.arc(
                pos.x, pos.y,
                20 / zoom,
                0, Math.PI * 2
            );
            ctx.fillStyle = "black";
            ctx.fill();
            ctx.drawImage(imageBuffer[index], pos.x - imgWidth / 2, pos.y - imgWidth / 2, imgWidth, imgWidth);
        });

        requestAnimationFrame(() => draw(ctx));
    }

    const getMousePos = (event: MouseEvent) => {
        const rect = canvas.getBoundingClientRect();
        const x = (event.clientX - rect.left + translation.x) / zoom;
        const y = (event.clientY - rect.top + translation.y) / zoom;
        return new Victor(x, y);
    };

    const onmousedown = (event: MouseEvent) => {
        if (editingNode) return;

        const pos = getMousePos(event);
        let node = node_data.find(
            node => Victor.fromArray(node.position).distance(pos) < 20
        );
        focusNode = node ?? focusNode;
        
        if (node) {
            if (event.button === 0) {
                dragOffset = pos.clone().subtract(Victor.fromArray(focusNode.position));
                draggingNode = true;
            }
            else if (event.button === 2) {
                editingNode = true;
            }
        }
        else {
            draggingCanvas = true;
            lastMousePos = pos;
        }
    }

    const onmousemove = (event: MouseEvent) => {
        if (editingNode) return;

        const pos = getMousePos(event);

        if (draggingNode) {
            let node = node_data.find(n => n.id === focusNode.id);
            if (node === undefined) throw new Error("uhhh focusNode error, it's not in the list...");
            else {
                node.position = pos.clone().subtract(dragOffset).toArray();
            }
        }
        else if (draggingCanvas) {
            const dMouse = pos.clone().subtract(lastMousePos);
            translation.subtract(dMouse);
        }
    }

    const onmouseup = (event: MouseEvent) => {
        if (editingNode) return;

        draggingNode = false;
        draggingCanvas = false;
    }

    
    const saveNode = (id: number, title: string, desc: string, img: string): boolean => {
        if (!window.confirm("Save this node with new contents?")) return false;

        node_data[id].name = title;
        node_data[id].description = desc;
        node_data[id].image = img;
        return true;
    }
    
    const exitNode = (saved: boolean, changed: boolean) => {
        if (!saved && changed) if (!window.confirm("Stop editing without saving?")) return;
        editingNode = false;
    }
    

    onMount(() => {
        canvas.addEventListener('contextmenu', event => event.preventDefault()); 

        for (const node of node_data) {
            let img = new Image();
            img.src = node.image;
            imageBuffer.push(img);
        }

        let ctx = canvas?.getContext("2d");
        if (ctx !== null) draw(ctx);
    });
</script>

<div id="canvas-box" class="relative inline-block">

    <canvas width="1700" height="800" class="bg-neutral-100 active:cursor-grabbing"
        bind:this={canvas}
        {onmousemove} {onmousedown} {onmouseup}
    ></canvas>

    <div id="edit-panel" class="absolute right-2.5 top-2.5">
        {#if (editingNode)}
            <NodeEditor node_id={focusNode.id} {saveNode} {exitNode} />
        {/if}
    </div>

    <ModePicker />
</div>