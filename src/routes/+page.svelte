<script lang="ts">
    import { PDFDocument, PageSizes } from "pdf-lib";

    let src = "";
    let leftHanded = false;
    let name = "";
    let a;
    let content = "";

    $: {
        updateBlob(content);
    }

    async function updateBlob(contentx) {
        if (contentx == "") return;
        let newDoc = await PDFDocument.create();
        let oldDoc = await PDFDocument.load(contentx);

        for (let i = 0; i < oldDoc.getPages().length; i++) {
            let page = oldDoc.getPages()[i];
            let lastPage = newDoc.addPage([PageSizes.A4[1], PageSizes.A4[0]]);

            const preamble = await newDoc.embedPage(page);
            const preambleDims = preamble.scale(0.7);

            lastPage.drawPage(preamble, {
                ...preambleDims,
                x: leftHanded ? PageSizes.A4[1] / 2 : 0,
                y: 0,
            });
        }
        src = await newDoc.saveAsBase64({ dataUri: true });
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
<form>
    <input
        type="file"
        id="file"
        accept="application/pdf"
        on:change={(e) => {
            upload(e);
        }}
    />
</form>
<div id="downloaddiv">
    <button
        id="button"
        on:click={(e) => {
            download();
        }}>Download</button
    >
    <input id="lefthand" bind:checked={leftHanded} type="checkbox" />
    <span>Left Handed</span>
</div>

<iframe title="pdf" {src}></iframe>
<a bind:this={a} id="download" href={src} />

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

    span {
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
</style>
