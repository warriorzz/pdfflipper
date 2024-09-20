<script lang="ts">
    import { PDFDocument, PageSizes } from "pdf-lib";

    let src = "";
    let leftHanded = false;
    let includeGraph = false;
    let name = "";
    let a;
    let content = "";
    let progress = -1;

    let error = true;

    $: {
        try {
            error = false;
            updateBlob(content, leftHanded, includeGraph);
        } catch (e) {
            error = true;
        }
    }

    async function updateBlob(contentx, left, graphing) {
        if (contentx == "") return;
        let newDoc = await PDFDocument.create();
        let oldDoc = await PDFDocument.load(contentx);

        // load static graph.pdf page
        const response = await fetch("/graph.pdf");
        const arrayBuffer = await response.arrayBuffer();
        let graph = await PDFDocument.load(arrayBuffer);

        let pages = oldDoc.getPages().length;
        for (let i = 0; i < pages; i++) {
            let page = oldDoc.getPages()[i];

            // let { width, height } = page.getSize();

            let lastPage = newDoc.addPage([PageSizes.A4[1], PageSizes.A4[0]]);

            const preamble = await newDoc.embedPage(page);
            const preambleDims = preamble.scale(0.7071);

            lastPage.drawPage(preamble, {
                ...preambleDims,
                x: left ? PageSizes.A4[1] / 2 : 0,
                y: 0,
            });

            if (graphing) {
                const graphPage = await newDoc.embedPage(graph.getPages()[0]);
                const graphDims = graphPage.scale(0.7071);

                lastPage.drawPage(graphPage, {
                    ...graphDims,
                    x: leftHanded ? 0 : PageSizes.A4[1] / 2,
                    y: 0,
                });
            }

            progress = Math.floor(((i + 1) / pages) * 100);
        }
        let bytes = new Uint8Array(await newDoc.save());
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
        }
    }

    function download() {
        a.click();
    }
</script>

<h1>PDF Flipper</h1>
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
        }}>Download</button
    >
    <input id="lefthand" bind:checked={leftHanded} type="checkbox" />
    <span style="margin-top: auto; margin-bottom: auto; margin-right: 20px"
        >Left Handed</span
    >
    <input id="graph" bind:checked={includeGraph} type="checkbox" />
    <span style="margin-top: auto; margin-bottom: auto"
        >Include Graphing Paper</span
    >
</div>
{#if progress != -1}
    <p>Processing... {progress}%</p>
{/if}
{#if src != ""}
    <iframe title="pdf" {src} type="application/pdf"></iframe>
{/if}
<a
    style="padding: 0; margin: 0; width: 0;"
    bind:this={a}
    id="download"
    href={src}
/>
{#if error}
    <p>There was an error. Please try again (with a different file).</p>
{/if}

<style>
    :global(*) {
        background-color: #0d1117;
        padding: 10px;
        margin: 10px;
    }

    h1 {
        font-size: 32px;
        color: lightgray;
        border-bottom: 1px solid white;
    }

    button,
    input[type="file"] {
        border: 3px solid #743ad5;
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
        height: 60vh;
    }

    .button-bar {
        display: flex;
        flex-width: 1;
        align-content: center;
    }
</style>
