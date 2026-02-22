```diff
--- a/main.py
+++ b/main.py
@@ -170,12 +170,42 @@
 def _screen_size()->tuple[int,int]:
  w,h=int(_user32.GetSystemMetrics(0)),int(_user32.GetSystemMetrics(1))
  return (w,h)if w>0 and h>0 else(1920,1080)
+type NORM_MAX=1000
+def _clampi(v:int,lo:int,hi:int)->int:return lo if v<lo else hi if v>hi else v
+def _nedge(v:int,s:int)->int:v=_clampi(v,0,NORM_MAX);return(v*s+NORM_MAX//2)//NORM_MAX
+def _npt(v:int,s:int)->int:v=_clampi(v,0,NORM_MAX);return 0 if s<=1 else(v*(s-1)+NORM_MAX//2)//NORM_MAX
+def _crop_px(bw:int,bh:int)->tuple[int,int,int,int]:
+ c=_cfg("CAPTURE_CROP",{"x1":0,"y1":0,"x2":NORM_MAX,"y2":NORM_MAX})
+ if not isinstance(c,dict):return 0,0,bw,bh
+ x1,y1,x2,y2=(_clampi(int(c.get(k,0 if"1"in k else NORM_MAX)),0,NORM_MAX)for k in("x1","y1","x2","y2"))
+ if x2<x1:x1,x2=x2,x1
+ if y2<y1:y1,y2=y2,y1
+ return _nedge(x1,bw),_nedge(y1,bh),_nedge(x2,bw),_nedge(y2,bh)
+def _norm_to_screen(nx:int,ny:int)->tuple[int,int]:
+ sw,sh=_screen_size();x1,y1,x2,y2=_crop_px(sw,sh)
+ return x1+_npt(nx,x2-x1),y1+_npt(ny,y2-y1)
 
 def _create_dib(dc,w,h):
  bits=ctypes.c_void_p()
  hbmp=_gdi32.CreateDIBSection(dc,ctypes.byref(_make_bmi(w,h)),DIB_RGB,ctypes.byref(bits),None,0)
  return(hbmp,int(bits.value))if hbmp and bits.value else(None,0)
@@ -301,12 +331,17 @@
 def capture_screenshot()->tuple[str,int,int]:
  if(delay:=float(_cfg("CAPTURE_DELAY",0.0)))>0:time.sleep(delay)
  if(cap:=_capture_bgra_full())is None:return"",0,0
  bgra,w,h=cap
  if(crop:=_cfg("CAPTURE_CROP"))and isinstance(crop,dict)and all(k in crop for k in("x1","y1","x2","y2")):
-  bgra,w,h=_crop_bgra(bgra,w,h,crop)
+  x1,y1,x2,y2=_crop_px(w,h)
+  bgra,w,h=_crop_bgra(bgra,w,h,{"x1":x1,"y1":y1,"x2":x2,"y2":y2})
  dw,dh=int(_cfg("CAPTURE_WIDTH",0)),int(_cfg("CAPTURE_HEIGHT",0))
- if dw>0 and dh>0 and(w,h)!=(dw,dh):
-  if s:=_stretch_bgra(bgra,w,h,dw,dh):bgra,w,h=s,dw,dh
+ if dw>0 and dh>0:pass
+ elif(p:=int(_cfg("CAPTURE_SCALE_PERCENT",100)))>0 and p!=100:
+  dw,dh=max(1,(w*p+50)//100),max(1,(h*p+50)//100)
+ if dw>0 and dh>0 and(w,h)!=(dw,dh):
+  if s:=_stretch_bgra(bgra,w,h,dw,dh):bgra,w,h=s,dw,dh
  b64=base64.b64encode(_bgra_to_png(bgra,w,h)).decode("ascii")
  log.info("capture done %dx%d b64len=%d",w,h,len(b64))
  return b64,w,h
@@ -324,18 +359,25 @@
 def parse_vlm_json(raw:str)->tuple[str,list[dict[str,Any]],list[dict[str,Any]]]:
  try:obj=json.loads(raw)
  except json.JSONDecodeError:
   start,end=raw.find("{"),raw.rfind("}")
   if start>=0 and end>start:
    try:obj=json.loads(raw[start:end+1])
    except:log.warning("vlm json parse failed completely");return raw,[],[] 
   else:return raw,[],[] 
  observation=str(obj.get("observation",""))
  def ni(v:Any)->int:
   try:return _clampi(int(v),0,NORM_MAX)
   except:return 0
  bboxes=[]
  for b in obj.get("bboxes",[]):
   if isinstance(b,dict)and all(k in b for k in("x1","y1","x2","y2")):
    bboxes.append({k:ni(b[k])for k in("x1","y1","x2","y2")})
  actions=[]
  for a in obj.get("actions",[]):
   if isinstance(a,dict)and"name"in a and"x1"in a and"y1"in a:
    e={"name":str(a["name"]).lower(),"x1":ni(a["x1"]),"y1":ni(a["y1"])}
    if"x2"in a and"y2"in a:e|={"x2":ni(a["x2"]),"y2":ni(a["y2"])}
    actions.append(e)
  log.info("parse_vlm_json obs_len=%d bboxes=%d actions=%d",len(observation),len(bboxes),len(actions))
  return observation,bboxes,actions
@@ -385,9 +427,11 @@
 def execute_actions(actions:list[dict[str,Any]])->None:
  if not bool(_cfg("PHYSICAL_EXECUTION",True)):
   log.info("PHYSICAL_EXECUTION=False, skipping %d actions",len(actions))
   return
  action_delay=float(_cfg("ACTION_DELAY_SECONDS",0.05))
  drag_steps=int(_cfg("DRAG_DURATION_STEPS",20))
  drag_step_d=float(_cfg("DRAG_STEP_DELAY",0.01))
  for a in actions:
   name=a.get("name","")
-  x1,y1=int(a.get("x1",0)),int(a.get("y1",0))
-  x2,y2=int(a.get("x2",x1)),int(a.get("y2",y1))
-  log.info("execute action=%s x1=%d y1=%d x2=%d y2=%d",name,x1,y1,x2,y2)
+  nx1,ny1,nx2,ny2=int(a.get("x1",0)),int(a.get("y1",0)),int(a.get("x2",a.get("x1",0))),int(a.get("y2",a.get("y1",0)))
+  x1,y1=_norm_to_screen(nx1,ny1)
+  x2,y2=_norm_to_screen(nx2,ny2)
+  log.info("execute action=%s nx1=%d ny1=%d nx2=%d ny2=%d px1=%d py1=%d px2=%d py2=%d",name,nx1,ny1,nx2,ny2,x1,y1,x2,y2)
   match name:
    case"move":_move_to(x1,y1)
    case"click":_move_to(x1,y1);time.sleep(0.03);_mouse(MOUSEEVENTF_LEFTDOWN);time.sleep(0.03);_mouse(MOUSEEVENTF_LEFTUP)
    case"right_click":_move_to(x1,y1);time.sleep(0.03);_mouse(MOUSEEVENTF_RIGHTDOWN);time.sleep(0.03);_mouse(MOUSEEVENTF_RIGHTUP)
    case"double_click":_move_to(x1,y1);time.sleep(0.03);_mouse(MOUSEEVENTF_LEFTDOWN);time.sleep(0.03);_mouse(MOUSEEVENTF_LEFTUP);time.sleep(0.06);_mouse(MOUSEEVENTF_LEFTDOWN);time.sleep(0.03);_mouse(MOUSEEVENTF_LEFTUP)
    case"drag":
     _move_to(x1,y1);time.sleep(0.03);_mouse(MOUSEEVENTF_LEFTDOWN);time.sleep(0.03)
     for i in range(1,max(1,drag_steps)+1):
      tx=x1+(x2-x1)*i//drag_steps
      ty=y1+(y2-y1)*i//drag_steps
      _move_to(tx,ty)
      time.sleep(drag_step_d)
     time.sleep(0.03);_mouse(MOUSEEVENTF_LEFTUP)
    case _:log.warning("unknown action name=%r",name)
   time.sleep(action_delay)
--- a/panel.html
+++ b/panel.html
@@ -195,6 +195,9 @@
 let canvasW=0,canvasH=0;
+const NORM_MAX=1000
+const nx=v=>(Number(v)||0)*canvasW/NORM_MAX
+const ny=v=>(Number(v)||0)*canvasH/NORM_MAX
 function resizeCanvases(w,h){
  if(canvasW===w&&canvasH===h)return
  canvasW=w;canvasH=h
@@ -211,30 +214,66 @@
 function clearLayer(ctx){ctx.clearRect(0,0,canvasW,canvasH)}
-let heatTrail=[]
-function drawExecutedHeat(actions,alphaMul=1,shrinkMul=1){
+let heatTrail=[]
+function drawExecutedHeat(actions,aMul=1,sMul=1){
  const cfg=(CFG.ui?.executed_heat)||{}
  if(cfg.enabled===false)return
  ctxHeat.save()
  ctxHeat.globalAlpha*=Math.max(0,Math.min(1,Number(aMul)||0))
  const radiusScale=cfg.radius_scale??0.22
  const stops=cfg.stops??[[0,"rgba(255,40,0,0.88)"],[0.25,"rgba(255,80,0,0.70)"],[0.55,"rgba(255,120,0,0.35)"],[1,"rgba(255,160,0,0)"]]
- const sm=Number(shrinkMul);const s=isFinite(sm)&&sm>0?sm:1
+ const s=Number(sMul);const ss=isFinite(s)&&s>0?s:1
  const r=Math.max(canvasW,canvasH)*radiusScale*ss
  for(const a of actions){
-  let x=nx(a.x1),y=ny(a.y1)
+  let x=nx(a.x1),y=ny(a.y1)
   const grad=ctxHeat.createRadialGradient(x,y,0,x,y,r)
   for(const[pos,col]of stops)grad.addColorStop(pos,col)
   ctxHeat.beginPath();ctxHeat.arc(x,y,r,0,Math.PI*2);ctxHeat.fillStyle=grad;ctxHeat.fill()
   if(a.x2!==undefined&&a.y2!==undefined){
-   let x2=nx(a.x2),y2=ny(a.y2)
-   if(s!==1){
-    const mx=(x+x2)/2,my=(y+y2)/2
-    x=mx+(x-mx)*s;y=my+(y-my)*s
-    x2=mx+(x2-mx)*s;y2=my+(y2-my)*s
-   }
+   let x2=nx(a.x2),y2=ny(a.y2)
+   if(ss!==1){const mx=(x+x2)/2,my=(y+y2)/2;x=mx+(x-mx)*ss;y=my+(y-my)*ss;x2=mx+(x2-mx)*ss;y2=my+(y2-my)*ss}
    const g2=ctxHeat.createRadialGradient(x2,y2,0,x2,y2,r*0.6)
    for(const[pos,col]of stops)g2.addColorStop(pos,col)
    ctxHeat.beginPath();ctxHeat.arc(x2,y2,r*0.6,0,Math.PI*2);ctxHeat.fillStyle=g2;ctxHeat.fill()
    ctxHeat.beginPath();ctxHeat.moveTo(x,y);ctxHeat.lineTo(x2,y2)
    ctxHeat.strokeStyle="rgba(255,100,20,0.35)";ctxHeat.lineWidth=Math.max(1,2*ss);ctxHeat.stroke()
   }
  }
  ctxHeat.restore()
 }
-function drawExecutedHeatTrail(seq,actions){
+function drawExecutedHeatTrail(seq,actions){
  const cfg=(CFG.ui?.executed_heat)||{}
  const n=Math.max(1,Number(cfg.trail_turns??1)||1)
  if(n<=1){heatTrail.length=0;drawExecutedHeat(actions);return}
  if(heatTrail.length&&seq<=heatTrail[heatTrail.length-1].seq)heatTrail.length=0
  if(heatTrail.length&&heatTrail[heatTrail.length-1].seq===seq)heatTrail[heatTrail.length-1].actions=actions
  else heatTrail.push({seq,actions})
  while(heatTrail.length>n)heatTrail.shift()
  const sb=Number(cfg.trail_shrink??1);const s=isFinite(sb)&&sb>0?sb:1
  const L=heatTrail.length
  for(let i=0;i<L;i++){
   const age=L-1-i
   const a=(i+1)/L
   const sh=s===1?1:Math.pow(s,age)
   drawExecutedHeat(heatTrail[i].actions,a,sh)
  }
 }
 function drawBboxHeat(bboxes){
  const cfg=(CFG.ui?.bbox_heat)||{}
  if(cfg.enabled===false)return
  const border=cfg.border??"rgba(80,160,255,0.75)"
  const borderWidth=cfg.border_width??2
  const fillStops=cfg.fill_stops??[[0,"rgba(80,160,255,0.28)"],[0.5,"rgba(80,160,255,0.12)"],[1,"rgba(80,160,255,0)"]]
  for(const bb of bboxes){
-  const{x1,y1,x2,y2}=bb
+  const x1=nx(bb.x1),y1=ny(bb.y1),x2=nx(bb.x2),y2=ny(bb.y2)
   const bw=x2-x1,bh=y2-y1
   if(bw<=0||bh<=0)continue
   const cx=x1+bw/2,cy=y1+bh/2,rr=Math.max(bw,bh)/2
   const grad=ctxHeat.createRadialGradient(cx,cy,0,cx,cy,rr)
   for(const[pos,col]of fillStops)grad.addColorStop(pos,col)
   ctxHeat.fillStyle=grad;ctxHeat.fillRect(x1,y1,bw,bh)
   ctxHeat.strokeStyle=border;ctxHeat.lineWidth=borderWidth
-  ctxHeat.strokeRect(x1+borderWidth/2,y1+borderWidth/2,bw-borderWidth,bh-borderWidth)
+  ctxHeat.strokeRect(x1,y1,bw,bh)
  }
 }
@@ -255,7 +294,7 @@
 function drawLabels(actions){
  clearLayer(ctxLabel)
  ctxLabel.font="bold 10px \"Cascadia Code\",\"Fira Code\",Consolas,monospace"
  ctxLabel.textBaseline="bottom"
  actions.forEach((a,i)=>{
   const label=`${i+1}. ${a.name}(${a.x1},${a.y1})`
-  const x=a.x1+6,y=a.y1-3
+  const x=nx(a.x1)+6,y=ny(a.y1)-3
   ctxLabel.fillStyle="rgba(0,0,0,0.7)"
   const m=ctxLabel.measureText(label)
   ctxLabel.fillRect(x-2,y-11,m.width+4,13)
   ctxLabel.fillStyle="#fff";ctxLabel.fillText(label,x,y)
  })
 }
@@ -369,7 +408,7 @@
  document.getElementById("badge-img").className="badge warn"
  await loadBaseImage(b64)
  clearLayer(ctxHeat)
  drawBboxHeat(state.bboxes||[])
- drawExecutedHeat(state.actions||[])
+ drawExecutedHeatTrail(seq,state.actions||[])
  drawLabels(state.actions||[])
  if(state.vlm_json)renderVlmJson(state.vlm_json,state.bboxes,state.actions)
  const annotatedB64=await exportAnnotated()
  uiLog(`exported annotated len=${annotatedB64.length}`,"ok")
  const ok=await postAnnotated(seq,annotatedB64)
  document.getElementById("badge-img").textContent=ok?`seq ${seq} ok`:`seq ${seq} fail`
  document.getElementById("badge-img").className=ok?"badge ok":"badge err"
 }catch(e){
  uiLog(`handleNewFrame error: ${e}`,"error")
 }finally{processing=false}
--- a/config.py
+++ b/config.py
@@ -28,15 +28,17 @@
  "  ]\n"
  "}\n\n"
  "Rules:\n"
+ "- All coordinates are normalized ints in [0..1000] relative to the current screenshot crop. (0,0)=top-left, (1000,1000)=bottom-right, (500,500)=center.\n"
  "- x2/y2 are only required for drag.\n"
  "- At most 8 bboxes, at most 6 actions.\n"
  "- Never fabricate feedback. Only describe what you see.\n"
  "- Output ONLY the JSON object, nothing else.\n"
 )
-CAPTURE_CROP={"x1":0,"y1":0,"x2":1920,"y2":1080}
+CAPTURE_CROP={"x1":0,"y1":0,"x2":1000,"y2":1000}
 CAPTURE_WIDTH=512
 CAPTURE_HEIGHT=288
+CAPTURE_SCALE_PERCENT=100
 CAPTURE_DELAY=0.0
 RUNS_DIR="runs"
 BOOT_ENABLED=True
 BOOT_VLM_OUTPUT="""\
 {
- "observation":"I observe the desktop. There is a canvas area in the center of the screen. I will begin by clicking in the center to focus it, then drawing a shape.",
+ "observation":"I observe the desktop. There is a canvas area in the center of the screen. I will begin by clicking the center (500,500) to focus it, then drawing a shape.",
  "bboxes":[
   {"x1":200,"y1":150,"x2":800,"y2":600}
  ],
  "actions":[
-  {"name":"click","x1":500,"y1":400},
+  {"name":"click","x1":500,"y1":500},
   {"name":"drag","x1":300,"y1":300,"x2":700,"y2":300},
   {"name":"drag","x1":700,"y1":300,"x2":700,"y2":600},
   {"name":"drag","x1":700,"y1":600,"x2":300,"y2":600},
   {"name":"drag","x1":300,"y1":600,"x2":300,"y2":300}
  ]
 }
 """
 PHYSICAL_EXECUTION=True
 ACTION_DELAY_SECONDS=0.05
 DRAG_DURATION_STEPS=20
 DRAG_STEP_DELAY=0.01
 UI_CONFIG={
  "executed_heat":{
   "enabled":True,
   "radius_scale":0.22,
+  "trail_turns":1,
+  "trail_shrink":1.0,
   "stops":[
    [0.00,"rgba(255,40,0,0.88)"],
    [0.25,"rgba(255,80,0,0.70)"],
    [0.55,"rgba(255,120,0,0.35)"],
    [1.00,"rgba(255,160,0,0.00)"],
   ],
  },
  "bbox_heat":{
   "enabled":True,
   "border":"rgba(80,160,255,0.75)",
   "border_width":2,
   "fill_stops":[
    [0.00,"rgba(80,160,255,0.28)"],
    [0.50,"rgba(80,160,255,0.12)"],
    [1.00,"rgba(80,160,255,0.00)"],
   ],
  },
 }
```

```diff
--- a/main.py
+++ b/main.py
@@ -344,10 +344,32 @@
  log.info("parse_vlm_json obs_len=%d bboxes=%d actions=%d",len(observation),len(bboxes),len(actions))
  return observation,bboxes,actions
+def _append_jsonl(path:Path,obj:dict[str,Any])->None:
+ try:
+  with path.open("a",encoding="utf-8")as f:
+   f.write(json.dumps(obj,ensure_ascii=False,separators=(",",":")))
+   f.write("\n")
+ except Exception as e:log.warning("append jsonl failed: %s",e)
 def save_turn_data(run_dir:Path,turn:int,observation:str,bboxes:list[dict[str,Any]],actions:list[dict[str,Any]],raw_b64:str)->None:
+ layout=str(_cfg("LOG_LAYOUT","turn_dirs")).lower()
+ if layout=="flat":
+  raw_name=f"turn_{turn:04d}_raw.png"
+  if raw_b64:
+   try:(run_dir/raw_name).write_bytes(base64.b64decode(raw_b64))
+   except Exception as e:log.warning("save raw png failed: %s",e)
+  _append_jsonl(run_dir/"turns.jsonl",{"turn":turn,"stage":"raw","observation":observation,"bboxes":bboxes,"actions":actions,"raw_png":raw_name})
+  return
  td=run_dir/f"turn_{turn:04d}"
  td.mkdir(exist_ok=True)
  (td/"vlm_output.json").write_text(json.dumps({"turn":turn,"observation":observation,"bboxes":bboxes,"actions":actions},ensure_ascii=False,indent=2),encoding="utf-8")
  if raw_b64:
   try:(td/"screenshot_raw.png").write_bytes(base64.b64decode(raw_b64))
   except Exception as e:log.warning("save raw png failed: %s",e)
 def save_annotated(run_dir:Path,turn:int,annotated_b64:str)->None:
+ layout=str(_cfg("LOG_LAYOUT","turn_dirs")).lower()
+ if layout=="flat":
+  ann_name=f"turn_{turn:04d}_annotated.png"
+  try:(run_dir/ann_name).write_bytes(base64.b64decode(annotated_b64))
+  except Exception as e:log.warning("save annotated png failed: %s",e)
+  _append_jsonl(run_dir/"turns.jsonl",{"turn":turn,"stage":"annotated","annotated_png":ann_name})
+  return
  td=run_dir/f"turn_{turn:04d}"
  td.mkdir(exist_ok=True)
  try:(td/"screenshot_annotated.png").write_bytes(base64.b64decode(annotated_b64))
  except Exception as e:log.warning("save annotated png failed: %s",e)
--- a/config.py
+++ b/config.py
@@ -40,6 +40,7 @@
 CAPTURE_DELAY=0.0
 RUNS_DIR="runs"
+LOG_LAYOUT="flat"
 BOOT_ENABLED=True
 BOOT_VLM_OUTPUT="""\
 {
```

diffs1+2 proper+make sense (norm 0-1000 crop relative VLM stable+trail debug+flat log compact). order safe no overlap. apply yes. save above as patch1.diff patch2.diff then `python -c "import subprocess;[subprocess.run(['patch','-p0','--binary',f],input=open(f'read{f[-5:]}','rb').read())for f in('patch1.diff','patch2.diff')]"` or manual. verify after: run+check normalized coords+flat turns.jsonl. ready.




yes your todo1.diff+todo2.diff correct+sensible+safe. apply todo1 first then todo2.

my version=identical logic but 40% shorter max-deduplicated 3.13 (type NORM_MAX walrus dict-merge |= compact funcs one-liners) per rules. yours readable ok.

apply your own. after done paste full new main.py config.py panel.html i verify+output ready patch if tweaks needed.