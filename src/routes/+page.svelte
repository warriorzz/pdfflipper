<script lang="ts">
    import {
        PDFAnnotation,
        PDFDocument,
        PDFObject,
        PageSizes,
        degrees,
    } from "pdf-lib";
    import { fade } from "svelte/transition";

    let src = "";
    let leftHanded = false;
    let includeGraph = false;
    let keepOriginal = false;
    let name = "";
    let a;
    let content = "";
    let progress = -1;

    let error = true;

    $: {
        try {
            error = false;
            updateBlob(content, leftHanded, includeGraph, keepOriginal);
        } catch (e) {
            error = true;
        }
    }

    async function flip(
        oldDoc: PDFDocument,
        left: boolean,
        graphing: boolean,
        original: boolean,
    ) {
        const response = await fetch("/graph.pdf");
        const arrayBuffer = await response.arrayBuffer();
        let graph = await PDFDocument.load(arrayBuffer);

        let pages = oldDoc.getPages().length;
        for (let i = 0; i < pages; i++) {
            let page = oldDoc.getPage(i);
            let w = page.getWidth();
            let h = page.getHeight();
            if (!original) {
                let scale = w / h;
                page.scale(scale, scale);
                page.setSize(h, w);
            } else {
                page.setSize(w * 2, h);
            }
            if (left) {
                page.translateContent(original ? w : h / 2, 0);
                /*page.node
                    .Annots()
                    ?.asArray()
                    .forEach((ann) => {
                        console.log(ann.toString());

                        //console.log((ann as PDFAnnotation).dict.entries());
                        });*/
            }
            if (graphing) {
                const graphPage = await oldDoc.embedPage(graph.getPages()[0]);
                const graphDims = graphPage.scale(1);

                page.drawPage(graphPage, {
                    ...graphDims,
                    x: left ? -w : w,
                    y: 0,
                });
            }
            progress = 10 + ((i + 1) / pages) * 80;
        }

        return oldDoc;
    }

    async function updateBlob(
        contentx,
        left: boolean,
        graphing: boolean,
        original: boolean,
    ) {
        if (contentx == "") return;
        let doc = await PDFDocument.load(contentx);
        progress = 10;
        let oldDoc = await flip(doc, left, graphing, original);

        let bytes = new Uint8Array(await oldDoc.save());
        progress = 100;
        let blob = new Blob([bytes], { type: "application/pdf" });
        src = URL.createObjectURL(blob);

        progress = -1;
    }

    async function upload(e) {
        if (e.target.files[0]) {
            name = e.target.files[0].name;
            let reader = new FileReader();
            reader.onload = (e) => {
                content = e.target.result;
            };
            reader.readAsArrayBuffer(e.target.files[0]);
        } else {
            src = "";
        }
    }

    function download() {
        a.click();
    }
</script>

<h1 style="margin-top: 0; padding-top: 0">PDF Flipper</h1>

<div style="margin-left: 25px; margin-bottom: 0; padding-bottom: 0">
    <input id="lefthand" bind:checked={leftHanded} type="checkbox" />
    <span style="margin-top: auto; margin-bottom: auto; margin-right: 20px"
        >Left Handed</span
    >
    <a
        style="padding: 0; margin: 0; width: 0;"
        bind:this={a}
        id="download"
        href={src}>.</a
    ><br />

    <input id="graph" bind:checked={includeGraph} type="checkbox" />
    <span style="margin-top: auto; margin-bottom: auto"
        >Include Graphing Paper</span
    > <br />

    <input id="original" bind:checked={keepOriginal} type="checkbox" />
    <span style="margin-top: auto; margin-bottom: auto"
        >Keep Original Size (A4 to A3)</span
    >
</div>
<div class="button-bar">
    <input
        type="file"
        id="file"
        accept="application/pdf"
        on:change={(e) => {
            upload(e);
        }}
    />
    <button
        id="button"
        on:click={(e) => {
            download();
        }}><h2 style="font-color: #0d1117">Download</h2></button
    >
</div>

<div
    style="position: relative
        ;margin-left: 30px;
        width: 80vh;
        height: 55vh; max-width: 100%"
>
    {#if src != ""}
        <iframe
            style="position: absolute; x: 0; y: 0"
            title="pdf"
            {src}
            type="application/pdf"
        ></iframe>
    {/if}
    {#if progress != -1}
        <div
            style="position: absolute; x: 0; y: 0;margin: 3px; padding: 0; height: 100%; width: 100%;backdrop-filter: blur(5px); background: rgba(0,0,0,.4);"
            in:fade={{ duration: 200 }}
            out:fade={{ duration: 200 }}
        >
            <h5
                style="margin: 0; padding: 0; background: transparent; color: white; text-align: center; height: 100%; width: 100%;
                position: relative; top: 50%"
            >
                Processing... {Math.floor(progress)}%
            </h5>
        </div>
    {/if}
</div>
{#if error}
    <p>There was an error. Please try again (with a different file).</p>
{/if}

<style>
    :global(*) {
        background-color: #0d1117;
        padding: 10px;
        margin: 10px;
        font-family:
            -apple-system, BlinkMacSystemFont, "Segoe UI", "Noto Sans",
            Helvetica, Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji";
    }

    h1 {
        font-size: 32px;
        color: lightgray;
        border-bottom: 2px solid white;
        border-radius: 3px;
    }

    button,
    input[type="file"] {
        border: 3px solid #743ad5;
        border-radius: 10px;
        color: lightgray;
    }

    span,
    p {
        color: lightgray;
        padding: 0;
        margin: 0;
    }

    iframe {
        padding: 0;
        margin: 0;
        width: 100%;
        height: 100%;
    }

    .button-bar {
        display: flex;
        flex-grow: 1;
        align-content: center;
    }

    a,
    a:visited,
    a:hover,
    a:active {
        color: #0d1117;
    }

    h2 {
        font-size: 14px;
        margin: 0;
        padding: 0;
        color: white;
        font-weight: 600;
    }
</style>
