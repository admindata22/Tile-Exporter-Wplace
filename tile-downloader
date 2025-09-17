// ==UserScript==
// @name         Tile Exporter
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  Adaptive tile downloader with live progress, preview, and PNG export.
// @author       bautti_.
// @match        https://wplace.live/*
// @grant        GM_addStyle
// @icon         https://files.catbox.moe/bylc9c.png
// ==/UserScript==

(function() {
    'use strict';

    const BASE_URL = "https://backend.wplace.live/files/s0/tiles/{x}/{y}.png";
    const MAX_RETRIES = 5, DELAY = 500;

    const uiHTML = `
    <div id="tm-ui-container" class="minimized">
        <div class="tm-ui-header">
            <h2>Tile Exporter</h2>
            <button id="tm-ui-toggle">&#x25B2;</button>
        </div>
        <div class="tm-ui-content">
            <div class="tm-card">
                <h3>Download Parts Of The Map</h3>
                <label>Start (Tl X_Tl Y_Px X_Px Y)</label>
                <input type="text" id="tm-start" placeholder="0_0_0_0">
                <label>End (Tl X_Tl Y_Px X_Px Y)</label>
                <input type="text" id="tm-end" placeholder="2_2_256_256">
                <button id="tm-download-btn" class="tm-btn">Download Tiles</button>
                <p id="tm-status"></p>
            </div>
            <div class="tm-card">
                <h3>Download Progress</h3>
                <div class="tm-stat">
                    <span id="tm-progress-text">0%</span>
                    <div class="tm-progress">
                        <div id="tm-progress-bar" class="tm-progress-bar" style="width:0%"></div>
                    </div>
                </div>
            </div>
            <div class="tm-card tm-card-preview">
                <h3>Preview</h3>
                <canvas id="tm-canvas"></canvas>
                <button id="tm-save-btn" class="tm-btn">Save as PNG</button>
            </div>
        </div>
    </div>`;
    document.body.insertAdjacentHTML('beforeend', uiHTML);

    GM_addStyle(`
        #tm-ui-container {position:fixed;top:70%;left:70%;width:350px;height:auto;background:#1e1e2f;color:#fff;border-radius:15px;box-shadow:0 10px 30px rgba(0,0,0,0.5);font-family:'Segoe UI',Tahoma,Geneva,Verdana,sans-serif;transition:all 0.4s ease;z-index:9999;}
        #tm-ui-container.minimized {width:180px;height:50px;border-radius:25px;overflow:hidden;}
        .tm-ui-header {display:flex;justify-content:space-between;align-items:center;padding:10px 15px;background:linear-gradient(90deg,#4b6cb7,#182848);cursor:grab;}
        .tm-ui-header h2{margin:0;font-size:1em;}
        #tm-ui-toggle{background:transparent;border:none;color:#fff;font-size:1.2em;cursor:pointer;}
        .tm-ui-content{padding:15px 20px;display:block;transition:all 0.4s ease;width:100%;box-sizing:border-box;max-height:70vh;overflow-y:auto;}
        #tm-ui-container.minimized .tm-ui-content{display:none;}
        .tm-card{display:flex;flex-direction:column;background:#2e2e3e;margin-bottom:15px;padding:15px;border-radius:10px;transition:transform 0.3s,box-shadow 0.3s;width:100%;box-sizing:border-box;}
        .tm-card:hover{transform:translateY(-3px);box-shadow:0 8px 20px rgba(0,0,0,0.4);}
        .tm-card-preview{flex:1 1 auto;display:flex;flex-direction:column;overflow:hidden;}
        #tm-canvas{flex:1 1 auto;width:100%;border:1px solid #555;display:block;}
        .tm-btn{background:#4b6cb7;border:none;padding:10px 15px;border-radius:8px;color:#fff;cursor:pointer;transition:background 0.3s;margin-top:5px;flex-shrink:0;}
        .tm-btn:hover{background:#182848;}
        input{width:100%;box-sizing:border-box;margin-bottom:5px;padding:5px;border-radius:5px;border:none;}
        input:focus{outline:none;}
        label{font-size:0.9em;display:block;margin-top:10px;}
        #tm-status{margin-top:5px;font-size:0.9em;color:#4b6cb7;}
        .tm-stat span{font-size:1em;display:block;margin-bottom:5px;}
        .tm-progress{width:100%;background:#444;height:10px;border-radius:5px;overflow:hidden;}
        .tm-progress-bar{height:100%;background:#4b6cb7;width:0%;}`);

    const container = document.getElementById('tm-ui-container');
    const toggleBtn = document.getElementById('tm-ui-toggle');
    const canvas = document.getElementById('tm-canvas');
    const progressText = document.getElementById('tm-progress-text');
    const progressBar = document.getElementById('tm-progress-bar');
    let originalCanvasWidth = 0, originalCanvasHeight = 0;

    toggleBtn.addEventListener('click', ()=>{
        container.classList.toggle('minimized');
        toggleBtn.innerHTML = container.classList.contains('minimized')?'&#x25B2;':'&#x25BC;';
        resizeCanvas();
    });

    let dragging=false, offsetX, offsetY;
    container.querySelector('.tm-ui-header').addEventListener('mousedown', e=>{
        if(e.target.tagName==='INPUT') return;
        dragging=true;
        offsetX=e.clientX-container.offsetLeft;
        offsetY=e.clientY-container.offsetTop;
        container.style.transition='none';
        e.preventDefault();
    });
    document.addEventListener('mousemove', e=>{
        if(!dragging) return;
        let left=Math.min(Math.max(0,e.clientX-offsetX), window.innerWidth-container.offsetWidth);
        let top=Math.min(Math.max(0,e.clientY-offsetY), window.innerHeight-container.offsetHeight);
        container.style.left=left+'px';
        container.style.top=top+'px';
    });
    document.addEventListener('mouseup', ()=>{if(dragging){dragging=false;container.style.transition='all 0.4s ease';}});

    function resizeCanvas(){
        if(!originalCanvasWidth||!originalCanvasHeight) return;
        const card=canvas.parentElement;
        const headerH=card.querySelector('h3').offsetHeight;
        const btnH=card.querySelector('button').offsetHeight;
        let availW=card.clientWidth, availH=card.clientHeight-headerH-btnH-10;
        const ratio=originalCanvasHeight/originalCanvasWidth;
        let w=availW, h=w*ratio;
        if(h>availH){h=availH; w=h/ratio;}
        canvas.width=w; canvas.height=h;
        if(canvas._image){const ctx=canvas.getContext('2d'); ctx.clearRect(0,0,w,h); ctx.drawImage(canvas._image,0,0,w,h);}
    }

    async function downloadTile(x,y){
        const url=BASE_URL.replace('{x}',x).replace('{y}',y);
        for(let i=0;i<MAX_RETRIES;i++){
            try{
                const res=await fetch(url);
                if(!res.ok) throw new Error(`HTTP ${res.status}`);
                return await createImageBitmap(await res.blob());
            }catch(e){console.warn(`Tile ${x}_${y} attempt ${i+1}: ${e}`); await new Promise(r=>setTimeout(r,1000));}
        }
        return null;
    }

    async function buildCroppedImage(startChunk,startInner,endChunk,endInner){
        const [xMin,yMin]=startChunk,[xMax,yMax]=endChunk,[xStart,yStart]=startInner,[xEnd,yEnd]=endInner;
        const status=document.getElementById('tm-status');
        const total=(xMax-xMin+1)*(yMax-yMin+1); let downloaded=0;
        const tiles=[];
        for(let y=yMin;y<=yMax;y++){
            for(let x=xMin;x<=xMax;x++){
                status.textContent=`📥 Downloading tile ${x}_${y}...`;
                const img=await downloadTile(x,y); if(img) tiles.push({x,y,img});
                downloaded++; const p=Math.round((downloaded/total)*100);
                progressText.textContent=p+'%'; progressBar.style.width=p+'%';
                await new Promise(r=>setTimeout(r,DELAY));
            }
        }
        if(!tiles.length){status.textContent="❌ No tiles downloaded"; return;}
        const tw=tiles[0].img.width, th=tiles[0].img.height, totalW=(xMax-xMin+1)*tw, totalH=(yMax-yMin+1)*th;
        const tempCanvas=document.createElement('canvas'); tempCanvas.width=totalW; tempCanvas.height=totalH;
        const ctx=tempCanvas.getContext('2d'); tiles.forEach(t=>ctx.drawImage(t.img,(t.x-xMin)*tw,(t.y-yMin)*th));
        const cropW=(xMax-xMin)*tw+xEnd-xStart, cropH=(yMax-yMin)*th+yEnd-yStart;
        const cropped=document.createElement('canvas'); cropped.width=cropW; cropped.height=cropH;
        cropped.getContext('2d').drawImage(tempCanvas,xStart,yStart,cropW,cropH,0,0,cropW,cropH);
        canvas._image=cropped; originalCanvasWidth=cropW; originalCanvasHeight=cropH; resizeCanvas();
        status.textContent="✅ Cropped image ready"; progressText.textContent='100%'; progressBar.style.width='100%';
    }

    document.getElementById('tm-download-btn').addEventListener('click',()=>{
        const start=document.getElementById('tm-start').value.trim().split('_').map(Number);
        const end=document.getElementById('tm-end').value.trim().split('_').map(Number);
        if(start.length!==4||end.length!==4){alert('Incorrect format. Must be 4 numbers separated by _'); return;}
        buildCroppedImage([start[0],start[1]],[start[2],start[3]],[end[0],end[1]],[end[2],end[3]]);
    });

    document.getElementById('tm-save-btn').addEventListener('click', () => {
        if (!canvas._image) return alert('No image to save');
        const fullCanvas = document.createElement('canvas');
        fullCanvas.width = canvas._image.width;
        fullCanvas.height = canvas._image.height;
        fullCanvas.getContext('2d').drawImage(canvas._image, 0, 0);
        const a = document.createElement('a');
        a.download = 'tiles.png';
        a.href = fullCanvas.toDataURL('image/png');
        a.click();
    });

})();
