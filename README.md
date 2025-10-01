# abitalalvmx.github.io
import React, { useRef, useState, useEffect } from "react";

// Wizard completo de 5 pasos para estimar e implementar una propuesta Wi‑Fi
// Pasos (como solicitaste):
// 1) Definir entorno (subir dibujo/plano + preguntas de materiales/ventanas/puertas/espejos)
// 2) Requisitos de conectividad (ancho de banda, usuarios, aplicaciones)
// 3) Selección de modelo de AP (lista predefinida)
// 4) Ubicaciones ideales (colocación manual o automática mediante heurística "IA ligera")
// 5) Generar informe PDF con mapa de calor aproximado, lista de materiales y plan de implementación
//
// Notas:
// - Esta es una implementación client-side (single file React) pensada para integrarse en una SPA.
// - El 'motor IA' es heurístico y puede mejorarse conectando servicios externos o bibliotecas de modelado RF.
// - Para generar PDFs complejos en producción conviene usar generación server-side, aquí se usa ventana para imprimir/guardar.

export default function WizardEstimadorWiFi(){
  // Stepper
  const [step, setStep] = useState(1);

  // Paso 1 — entorno y archivo
  const [uploadedImage, setUploadedImage] = useState(null); // data URL
  const [imageName, setImageName] = useState("");
  const [dimensionsMeters, setDimensionsMeters] = useState({width:20, length:15});
  const [walls, setWalls] = useState([]); // optional user-specified wall lines (not implemented as draggable, but as count/typology)
  const [doors, setDoors] = useState(0);
  const [windows, setWindows] = useState(0);
  const [mirrorArea, setMirrorArea] = useState(false);
  const [environmentType, setEnvironmentType] = useState("office");
  const [constructionMaterial, setConstructionMaterial] = useState("drywall");
  const [isExterior, setIsExterior] = useState(false);

  // Paso 2 — requisitos de conectividad
  const [internetMbps, setInternetMbps] = useState(200);
  const [usersRecurrent, setUsersRecurrent] = useState(30);
  const [usersPeak, setUsersPeak] = useState(60);
  const [apps, setApps] = useState("ofimática, videollamada ocasional");
  const [businessType, setBusinessType] = useState("oficina");

  // Paso 3 — selección de APs (sin precios)
  const defaultCatalog = [
    {id:'aruba505', name:'Aruba 505'},
    {id:'aruba515', name:'Aruba 515'},
    {id:'aruba565', name:'Aruba 565'},
    {id:'ruckus350', name:'Ruckus 350'},
    {id:'ruckus350e', name:'Ruckus 350E'},
    {id:'ruckusT310n', name:'Ruckus T310N'},
  ];
  const [catalog, setCatalog] = useState(defaultCatalog);
  const [chosenModelId, setChosenModelId] = useState(defaultCatalog[0].id);

  // Paso 4 — ubicaciones
  const canvasRef = useRef(null);
  const [placedAPs, setPlacedAPs] = useState([]); // manual placements: {xPx,yPx}
  const [autoPlacedAPs, setAutoPlacedAPs] = useState([]);
  const [useAutoPlacement, setUseAutoPlacement] = useState(true);

  // Resultado y reporte
  const [estimation, setEstimation] = useState(null);

  // Canvas and image scaling helpers
  const canvasSize = {w:800, h:500};
  const [imgScale, setImgScale] = useState(1);
  const [imgOffset, setImgOffset] = useState({x:0,y:0});

  useEffect(()=>{ drawCanvas(); }, [uploadedImage, placedAPs, autoPlacedAPs]);

  // --- Helpers ---
  function onUploadImage(file){
    if(!file) return;
    const reader = new FileReader();
    reader.onload = (e)=>{ setUploadedImage(e.target.result); setImageName(file.name); setTimeout(()=>drawCanvas(),100); };
    reader.readAsDataURL(file);
  }

  function drawCanvas(){
    const canvas = canvasRef.current; if(!canvas) return; const ctx = canvas.getContext('2d'); ctx.clearRect(0,0,canvas.width,canvas.height);
    // background
    ctx.fillStyle = '#ffffff'; ctx.fillRect(0,0,canvas.width,canvas.height);

    // draw uploaded image centered
    if(uploadedImage){
      const img = new Image(); img.onload = ()=>{
        // scale to fit canvas while keeping aspect
        const ratio = Math.min(canvas.width / img.width, canvas.height / img.height);
        const drawW = img.width * ratio; const drawH = img.height * ratio; const offX = (canvas.width - drawW)/2; const offY = (canvas.height - drawH)/2;
        setImgScale(ratio); setImgOffset({x:offX,y:offY});
        ctx.drawImage(img, offX, offY, drawW, drawH);
        // overlay placed APs and auto APs
        drawAPMarkers(ctx, offX, offY, drawW, drawH);
      };
      img.src = uploadedImage;
    } else {
      // draw a placeholder rectangle representing the area
      const margin = 40; const w = canvas.width - margin*2; const h = canvas.height - margin*2; ctx.strokeStyle='#0f172a'; ctx.lineWidth=2; ctx.strokeRect(margin, margin, w, h);
      ctx.font='14px sans-serif'; ctx.fillStyle='#0f172a'; ctx.fillText(`${dimensionsMeters.width}m x ${dimensionsMeters.length}m`, margin+10, margin+20);
      drawAPMarkers(ctx, margin, margin, w, h);
    }
  }

  function drawAPMarkers(ctx, offX, offY, drawW, drawH){
    // draw autoPlacedAPs with blue, manual with orange
    const drawList = (list, color) => {
      ctx.fillStyle = color;
      list.forEach(p=>{
        const x = offX + p.x * drawW; const y = offY + p.y * drawH; ctx.beginPath(); ctx.arc(x,y,8,0,Math.PI*2); ctx.fill(); ctx.fillStyle='#fff'; ctx.font='10px sans-serif'; ctx.fillText('AP', x-10, y+4); ctx.fillStyle=color;
      });
    };
    drawList(autoPlacedAPs, '#0b84ff'); drawList(placedAPs, '#ff8a00');

    // optionally draw coverage heat circles (simple approximation)
    const coverageM = estimateCoveragePerAPMeters();
    if(coverageM){
      // scale meters to pixels using dimensionsMeters and drawW/drawH
      const pixelPerMeterX = drawW / dimensionsMeters.width; const pixelPerMeterY = drawH / dimensionsMeters.length; const rPx = Math.round(((pixelPerMeterX + pixelPerMeterY)/2) * coverageM);
      ctx.globalAlpha = 0.12; ctx.fillStyle = '#0b84ff'; [...autoPlacedAPs,...placedAPs].forEach(p=>{ const x = offX + p.x*drawW; const y = offY + p.y*drawH; ctx.beginPath(); ctx.arc(x,y,rPx,0,Math.PI*2); ctx.fill(); }); ctx.globalAlpha = 1;
    }
  }

  // Convert pixel click into relative coords 0..1 inside image area
  function onCanvasClick(e){
    if(!uploadedImage && !imgScale) return; const rect = canvasRef.current.getBoundingClientRect(); const x = e.clientX - rect.left; const y = e.clientY - rect.top;
    // determine if click is inside image area (use imgOffset/scale) else ignore
    const imgOff = imgOffset; const imgW = (uploadedImage ? (canvasRef.current.width - imgOff.x*2) : canvasRef.current.width - 80);
    const imgH = (uploadedImage ? (canvasRef.current.height - imgOff.y*2) : canvasRef.current.height - 80);
    const relX = (x - imgOff.x) / imgW; const relY = (y - imgOff.y) / imgH;
    if(relX < 0 || relX > 1 || relY < 0 || relY > 1) return; // outside
    // add manual placement
    setPlacedAPs(prev => [...prev, {x: relX, y: relY}]);
  }

  // Simple heuristic: estimate per-AP coverage radius in meters based on environment & materials
  function estimateCoveragePerAPMeters(){
    const baseByEnv = {open: 20, office: 10, dense: 6, warehouse: 8};
    const materialMult = {drywall:1.0, glass:0.9, brick:0.7, concrete:0.5}[constructionMaterial] || 1.0;
    const mirrorMult = mirrorArea ? 0.9 : 1.0;
    const wifiGenMult = {'wifi5':1.0,'wifi6':0.95,'wifi6e':0.8}[wifiGen] || 1.0;
    const base = baseByEnv[environmentType] || 10;
    // lower radius for exterior or heavy metal
    const metalMult = (mirrorArea || isExterior) ? 0.8 : 1.0;
    const radius = base * materialMult * mirrorMult * wifiGenMult * metalMult;
    return Math.max(4, radius); // minimum
  }

  // Auto placement algorithm (greedy grid covering): returns array of normalized positions {x,y}
  function computeAutoPlacement(){
    const coverageM = estimateCoveragePerAPMeters();
    const w = dimensionsMeters.width; const h = dimensionsMeters.length;
    // approximate number across/vertical based on spacing ~= radius*sqrt(2) to get overlap
    const spacing = coverageM * Math.sqrt(2);
    const nx = Math.max(1, Math.ceil(w / spacing));
    const ny = Math.max(1, Math.ceil(h / spacing));
    const positions = [];
    for(let iy=0; iy<ny; iy++){
      for(let ix=0; ix<nx; ix++){
        const px = (ix + 0.5)/nx; const py = (iy + 0.5)/ny; positions.push({x:px, y:py});
      }
    }
    setAutoPlacedAPs(positions);
    return positions;
  }

  // Compute final estimation (choose manual placements if any and use auto if selected)
  function finalizeEstimation(){
    let finalAPs = useAutoPlacement ? autoPlacedAPs.length ? autoPlacedAPs : computeAutoPlacement() : placedAPs;
    const total = finalAPs.length;
    const covPerAP = estimateCoveragePerAPMeters();
    const area = dimensionsMeters.width * dimensionsMeters.length;
    const explanation = { total, area, covPerAP, method: useAutoPlacement ? 'auto' : 'manual', placements: finalAPs };
    setEstimation(explanation);
    return explanation;
  }

  // Generate printable proposal (opens new window with markup + simple heatmap image from canvas)
  function generatePDFProposal(){
    const final = estimation || finalizeEstimation();
    // capture canvas as image
    const canvas = canvasRef.current; const dataUrl = canvas.toDataURL();
    const model = catalog.find(c=>c.id===chosenModelId);
    const bom = `${final.total} x ${model.name}`;
    const html = `<!doctype html><html><head><meta charset='utf-8'><title>Propuesta Wi-Fi</title>
      <style>body{font-family:Arial,Helvetica,sans-serif;padding:20px;color:#0f172a}h1{color:#0b63a5}table{width:100%;border-collapse:collapse}td,th{padding:8px;border-bottom:1px solid #ddd} .right{text-align:right}</style></head><body>
      <h1>Propuesta técnica - Estimado inicial</h1>
      <h2>Resumen</h2>
      <p><strong>Proyecto:</strong> ${imageName || 'Plano sin nombre'}</p>
      <p><strong>Modelo AP seleccionado:</strong> ${model.name}</p>
      <p><strong>Total estimado:</strong> ${final.total} AP(s)</p>
      <h3>Mapa (estimado)</h3>
      <img src='${dataUrl}' style='width:100%;max-width:900px;border:1px solid #ddd' />
      <h3>Lista de materiales (BOM)</h3>
      <ul><li>${bom}</li><li>Cables y conectores (a definir tras site survey)</li><li>Switch PoE requerido (según consumo final por AP)</li></ul>
      <h3>Detalles de dimensionamiento</h3>
      <ul>
        <li>Área: ${final.area} m²</li>
        <li>Radio de cobertura estimado por AP: ${Math.round(final.covPerAP)} m</li>
        <li>Método de colocación: ${final.method}</li>
        <li>Requisitos de conectividad: ${internetMbps} Mbps • usuarios pico ${usersPeak}</li>
      </ul>
      <p>Nota: Este documento es una propuesta inicial basada en un modelo heurístico. Recomendamos realizar un site survey con equipo profesional para validar y ajustar ubicaciones y cantidades.</p>
      <p><button onclick='window.print()'>Imprimir / Guardar como PDF</button></p>
    </body></html>`;
    const w = window.open('','_blank'); w.document.write(html); w.document.close();
  }

  // Navigation / validation helpers
  function canProceedFromStep1(){
    // require either uploaded image or valid positive dimensions
    return uploadedImage || (dimensionsMeters.width>0 && dimensionsMeters.length>0);
  }
  function canProceedFromStep2(){ return internetMbps>0 && usersPeak>0; }
  function canProceedFromStep3(){ return !!chosenModelId; }

  // --- UI ---
  return (
    <div className="p-6 max-w-6xl mx-auto">
      <h1 className="text-2xl font-bold mb-4">Asistente: diseño rápido de red Wi‑Fi (5 pasos)</h1>

      <div className="mb-4 flex gap-2 text-sm">
        {[1,2,3,4,5].map(s=> (
          <div key={s} className={`px-3 py-1 rounded-full ${step===s? 'bg-sky-600 text-white':'bg-gray-100 text-gray-700'}`}>Paso {s}</div>
        ))}
      </div>

      {step===1 && (
        <div className="p-4 bg-white rounded shadow">
          <h2 className="font-semibold">1 — Definir entorno (sube dibujo/plano)</h2>
          <p className="text-sm text-gray-600">Sube un JPG/PNG/HEIC del plano o foto del espacio. Si no tienes plano, ingresa dimensiones en metros.</p>

          <div className="mt-3">
            <input type="file" accept="image/*" onChange={(e)=>onUploadImage(e.target.files[0])} />
            {uploadedImage && <div className="mt-2 text-sm">Archivo: {imageName}</div>}
          </div>

          <div className="grid grid-cols-2 gap-3 mt-3">
            <div>
              <label className="text-sm">Ancho (m)</label>
              <input type="number" value={dimensionsMeters.width} onChange={(e)=>setDimensionsMeters({...dimensionsMeters, width: Number(e.target.value)})} className="w-full p-2 border rounded mt-1" />
            </div>
            <div>
              <label className="text-sm">Largo (m)</label>
              <input type="number" value={dimensionsMeters.length} onChange={(e)=>setDimensionsMeters({...dimensionsMeters, length: Number(e.target.value)})} className="w-full p-2 border rounded mt-1" />
            </div>
            <div>
              <label className="text-sm">Pisos</label>
              <input type="number" min={1} value={1} readOnly className="w-full p-2 border rounded mt-1 bg-gray-50" />
            </div>
            <div>
              <label className="text-sm">Tipo de entorno</label>
              <select value={environmentType} onChange={(e)=>setEnvironmentType(e.target.value)} className="w-full p-2 border rounded mt-1">
                <option value="office">Oficina</option>
                <option value="open">Espacio abierto</option>
                <option value="dense">Ambiente denso</option>
                <option value="warehouse">Almacén / techo alto</option>
              </select>
            </div>
          </div>

          <div className="grid grid-cols-3 gap-3 mt-3">
            <div>
              <label className="text-sm">Material principal</label>
              <select value={constructionMaterial} onChange={(e)=>setConstructionMaterial(e.target.value)} className="w-full p-2 border rounded mt-1">
                <option value="drywall">Tablaroca</option>
                <option value="glass">Vidrio</option>
                <option value="brick">Ladrillo</option>
                <option value="concrete">Concreto</option>
              </select>
            </div>
            <div>
              <label className="text-sm">Puertas</label>
              <input type="number" value={doors} onChange={(e)=>setDoors(Number(e.target.value))} className="w-full p-2 border rounded mt-1" />
            </div>
            <div>
              <label className="text-sm">Ventanas</label>
              <input type="number" value={windows} onChange={(e)=>setWindows(Number(e.target.value))} className="w-full p-2 border rounded mt-1" />
            </div>
          </div>

          <label className="flex items-center gap-2 mt-3"><input type="checkbox" checked={mirrorArea} onChange={(e)=>setMirrorArea(e.target.checked)} /> <span className="text-sm">¿Hay espejos grandes o superficies metálicas?</span></label>
          <label className="flex items-center gap-2 mt-2"><input type="checkbox" checked={isExterior} onChange={(e)=>setIsExterior(e.target.checked)} /> <span className="text-sm">¿Área exterior / semi-exterior?</span></label>

          <div className="mt-4 flex gap-2">
            <button disabled={!canProceedFromStep1()} onClick={()=>setStep(2)} className={`px-4 py-2 rounded ${canProceedFromStep1()? 'bg-sky-600 text-white':'bg-gray-200'}`}>Siguiente — Requisitos</button>
          </div>
        </div>
      )}

      {step===2 && (
        <div className="p-4 bg-white rounded shadow">
          <h2 className="font-semibold">2 — Requisitos de conectividad</h2>
          <div className="grid grid-cols-2 gap-3 mt-3">
            <div>
              <label className="text-sm">Ancho de banda Internet (Mbps)</label>
              <input type="number" value={internetMbps} onChange={(e)=>setInternetMbps(Number(e.target.value))} className="w-full p-2 border rounded mt-1" />
            </div>
            <div>
              <label className="text-sm">Usuarios recurrentes</label>
              <input type="number" value={usersRecurrent} onChange={(e)=>setUsersRecurrent(Number(e.target.value))} className="w-full p-2 border rounded mt-1" />
            </div>
            <div>
              <label className="text-sm">Usuarios pico esperados</label>
              <input type="number" value={usersPeak} onChange={(e)=>setUsersPeak(Number(e.target.value))} className="w-full p-2 border rounded mt-1" />
            </div>
            <div>
              <label className="text-sm">Tipo de aplicaciones</label>
              <input type="text" value={apps} onChange={(e)=>setApps(e.target.value)} className="w-full p-2 border rounded mt-1" />
            </div>
          </div>

          <div className="mt-4 flex gap-2">
            <button onClick={()=>setStep(1)} className="px-4 py-2 rounded border">Atrás</button>
            <button disabled={!canProceedFromStep2()} onClick={()=>setStep(3)} className={`px-4 py-2 rounded ${canProceedFromStep2()? 'bg-sky-600 text-white':'bg-gray-200'}`}>Siguiente — Elegir AP</button>
          </div>
        </div>
      )}

      {step===3 && (
        <div className="p-4 bg-white rounded shadow">
          <h2 className="font-semibold">3 — Selecciona modelo de punto de acceso</h2>
          <p className="text-sm text-gray-600">Selecciona el modelo que usarás en la propuesta. Esto ayuda a generar ubicaciones basadas en patrones de cobertura estimados.</p>

          <div className="grid grid-cols-2 gap-3 mt-3">
            {catalog.map(item=> (
              <label key={item.id} className={`p-3 border rounded cursor-pointer ${chosenModelId===item.id? 'bg-sky-50 border-sky-200':''}`}>
                <input type="radio" name="apModel" value={item.id} checked={chosenModelId===item.id} onChange={()=>setChosenModelId(item.id)} />
                <span className="ml-2">{item.name}</span>
              </label>
            ))}
          </div>

          <div className="mt-4 flex gap-2">
            <button onClick={()=>setStep(2)} className="px-4 py-2 rounded border">Atrás</button>
            <button disabled={!canProceedFromStep3()} onClick={()=>{ setStep(4); computeAutoPlacement(); }} className={`px-4 py-2 rounded ${canProceedFromStep3()? 'bg-sky-600 text-white':'bg-gray-200'}`}>Siguiente — Ubicaciones</button>
          </div>
        </div>
      )}

      {step===4 && (
        <div className="p-4 bg-white rounded shadow">
          <h2 className="font-semibold">4 — Identificar ubicaciones ideales</h2>
          <p className="text-sm text-gray-600">Puedes colocar APs manualmente haciendo clic en el plano, o usar la colocación automática (IA ligera) para obtener una propuesta rápida.</p>

          <div className="flex items-center gap-3 mt-2">
            <label className="flex items-center gap-2"><input type="radio" checked={useAutoPlacement} onChange={()=>setUseAutoPlacement(true)} /> Automática</label>
            <label className="flex items-center gap-2"><input type="radio" checked={!useAutoPlacement} onChange={()=>setUseAutoPlacement(false)} /> Manual</label>
          </div>

          <div className="mt-3">
            <canvas ref={canvasRef} width={canvasSize.w} height={canvasSize.h} className="w-full border rounded" onClick={onCanvasClick} />
            <div className="text-xs text-gray-600 mt-2">Click sobre el plano para colocar AP (modo manual). APs automáticos están en azul; manuales en naranja.</div>
          </div>

          <div className="mt-3 flex gap-2">
            <button onClick={()=>{ setPlacedAPs([]); setAutoPlacedAPs([]); }} className="px-3 py-2 border rounded">Limpiar</button>
            <button onClick={()=>computeAutoPlacement()} className="px-3 py-2 bg-sky-600 text-white rounded">Recalcular automáticos</button>
            <button onClick={()=>{ const final = finalizeEstimation(); setStep(5); }} className="px-3 py-2 bg-emerald-600 text-white rounded">Finalizar y generar informe</button>
          </div>
        </div>
      )}

      {step===5 && (
        <div className="p-4 bg-white rounded shadow">
          <h2 className="font-semibold">5 — Implementar con confianza (Informe)</h2>
          <p className="text-sm text-gray-600">Genera un informe PDF con el mapa de calor estimado, ubicaciones de AP y lista de materiales inicial.</p>

          <div className="mt-3">
            <button onClick={generatePDFProposal} className="px-4 py-2 bg-sky-600 text-white rounded">Generar propuesta (Imprimir/Guardar PDF)</button>
            <button onClick={()=>{ setStep(1); setEstimation(null); setPlacedAPs([]); setAutoPlacedAPs([]); }} className="ml-2 px-4 py-2 border rounded">Nuevo proyecto</button>
          </div>

          <div className="mt-4">
            <h3 className="font-medium">Resumen</h3>
            {estimation? (
              <div>
                <p>Total APs: <strong>{estimation.total}</strong></p>
                <p>Radio aproximado por AP: <strong>{Math.round(estimation.covPerAP)} m</strong></p>
                <p>Método: <strong>{estimation.method}</strong></p>
              </div>
            ) : <p className="text-sm text-gray-600">Pulse 'Finalizar y generar informe' en el paso 4 para producir el resumen.</p>}
          </div>
        </div>
      )}

      <div className="mt-6 text-sm text-gray-600">Si quieres, puedo:  
        <ul className="ml-4 list-disc">
          <li>Conectar el "IA ligero" con modelos de patrón de antena reales (para cada modelo d
