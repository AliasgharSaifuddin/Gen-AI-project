<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>EcoGuardian AI — Smart Environmental Platform</title>
  <script src="https://unpkg.com/react@18/umd/react.development.js"></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
  <style>
    *{margin:0;padding:0;box-sizing:border-box}
    html,body,#root{height:100%;width:100%}
    body{background:#020c1b;font-family:'Segoe UI',system-ui,sans-serif;color:#e2e8f0}
    input,select,textarea{font-family:inherit;color-scheme:dark}
    input[type=range]{accent-color:#10b981;cursor:pointer}
    ::-webkit-scrollbar{width:5px}
    ::-webkit-scrollbar-track{background:#0a1628}
    ::-webkit-scrollbar-thumb{background:#1e3a5f;border-radius:3px}
    @keyframes spin{to{transform:rotate(360deg)}}
    @keyframes pulse{0%,100%{opacity:1}50%{opacity:.4}}
    @keyframes fadeIn{from{opacity:0;transform:translateY(10px)}to{opacity:1;transform:translateY(0)}}
    @keyframes glow{0%,100%{box-shadow:0 0 8px #10b98122}50%{box-shadow:0 0 22px #10b98144}}
    @keyframes blink{0%,100%{opacity:1}50%{opacity:.3}}
    .fade{animation:fadeIn .35s ease}
    button{transition:all .15s}
    button:active{transform:scale(.96)}
  </style>
</head>
<body>
<div id="root"></div>
<script type="text/babel">
const{useState,useEffect,useRef,useCallback}=React;

// ╔══════════════════════════════════════════════════╗
// ║  PASTE YOUR FREE GROQ API KEY BELOW              ║
// ║  Get it FREE → https://console.groq.com          ║
// ╚══════════════════════════════════════════════════╝
const GROQ_KEY = "YOUR_GROQ_API_KEY_HERE";

// ── AI Call ──────────────────────────────────────────────────────────────────
async function askAI(system, user, history=[]) {
  if(GROQ_KEY==="YOUR_GROQ_API_KEY_HERE")
    return "⚠️ Please add your free Groq API key at the top of this file.\nGet it at: https://console.groq.com (takes 2 minutes, completely free)";
  try{
    const msgs=[{role:"system",content:system},...history,{role:"user",content:user}];
    const r=await fetch("https://api.groq.com/openai/v1/chat/completions",{
      method:"POST",
      headers:{"Content-Type":"application/json","Authorization":"Bearer "+GROQ_KEY},
      body:JSON.stringify({model:"llama3-70b-8192",max_tokens:1024,messages:msgs})
    });
    const d=await r.json();
    if(d.error) return "❌ API Error: "+d.error.message;
    return d.choices?.[0]?.message?.content||"No response.";
  }catch(e){return "❌ Error: "+e.message;}
}

// ── Real Data APIs (all FREE, no key needed) ──────────────────────────────────
// OpenAQ — real government air quality sensors worldwide
async function fetchOpenAQ(lat, lng, cityName) {
  try {
    // Try to get nearest station measurements
    const url = `https://api.openaq.org/v2/latest?coordinates=${lat},${lng}&radius=50000&limit=20&order_by=distance`;
    const r = await fetch(url, {headers:{"Accept":"application/json"}});
    const d = await r.json();
    if(d.results && d.results.length > 0) {
      // Parse results into our format
      const stations = {};
      d.results.forEach(result => {
        const name = result.name || result.location || "Station";
        if(!stations[name]) stations[name]={name,lat:result.coordinates?.latitude||lat,lng:result.coordinates?.longitude||lng,params:{}};
        result.measurements?.forEach(m => {
          stations[name].params[m.parameter] = {value:m.value, unit:m.unit, lastUpdated:m.lastUpdated};
        });
      });
      return {success:true, stations:Object.values(stations).slice(0,8), source:"OpenAQ Live"};
    }
    return {success:false};
  } catch(e) {
    return {success:false, error:e.message};
  }
}

// Open-Meteo — free real weather data (no key needed)
async function fetchWeather(lat, lng) {
  try {
    const url = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lng}&current=temperature_2m,relative_humidity_2m,wind_speed_10m,wind_direction_10m,weather_code&hourly=temperature_2m,relative_humidity_2m&timezone=auto&forecast_days=1`;
    const r = await fetch(url);
    const d = await r.json();
    if(d.current) {
      return {
        success: true,
        temp: Math.round(d.current.temperature_2m),
        humidity: d.current.relative_humidity_2m,
        windSpeed: d.current.wind_speed_10m,
        windDir: d.current.wind_direction_10m,
        source: "Open-Meteo Live"
      };
    }
    return {success:false};
  } catch(e) { return {success:false}; }
}

// Nominatim geocoding — free, no key needed
async function geocodeCity(cityName) {
  try {
    const url = `https://nominatim.openstreetmap.org/search?q=${encodeURIComponent(cityName)}&format=json&limit=5`;
    const r = await fetch(url, {headers:{"User-Agent":"EcoGuardianAI/1.0"}});
    const d = await r.json();
    if(d.length > 0) {
      return d.map(item => ({
        name: item.display_name.split(",").slice(0,2).join(", "),
        lat: parseFloat(item.lat),
        lng: parseFloat(item.lon),
        type: item.type
      }));
    }
    return [];
  } catch(e) { return []; }
}

// ── Helpers ──────────────────────────────────────────────────────────────────
const aqiColor=v=>v<=50?"#22c55e":v<=100?"#eab308":v<=150?"#f97316":v<=200?"#ef4444":v<=300?"#9333ea":"#7f1d1d";
const aqiLabel=v=>v<=50?"Good":v<=100?"Moderate":v<=150?"Unhealthy (Sensitive)":v<=200?"Unhealthy":v<=300?"Very Unhealthy":"Hazardous";
const aqiEmoji=v=>v<=50?"😊":v<=100?"😐":v<=150?"😷":v<=200?"🤢":v<=300?"☠️":"💀";

// Convert raw pollutant values to rough AQI estimate
function estimateAQI(params={}) {
  let aqi = 50; // default moderate
  if(params.pm25) {
    const v = params.pm25.value;
    if(v<=12) aqi=Math.round(v/12*50);
    else if(v<=35.4) aqi=Math.round(50+(v-12)/23.4*50);
    else if(v<=55.4) aqi=Math.round(100+(v-35.4)/20*50);
    else if(v<=150.4) aqi=Math.round(150+(v-55.4)/95*50);
    else aqi=Math.min(500,Math.round(200+(v-150.4)/100*100));
  } else if(params.pm10) {
    const v = params.pm10.value;
    aqi = Math.round(Math.min(500, v * 1.2));
  } else if(params.no2) {
    const v = params.no2.value;
    aqi = Math.round(Math.min(500, v * 0.8));
  }
  return Math.max(1, aqi);
}

// ── UI Components ─────────────────────────────────────────────────────────────
function Spin({size=16,color="#10b981"}){
  return <div style={{width:size,height:size,border:`2px solid ${color}33`,borderTop:`2px solid ${color}`,borderRadius:"50%",animation:"spin .8s linear infinite",flexShrink:0}}/>;
}
function Loader({text="Loading…"}){
  return(
    <div style={{display:"flex",alignItems:"center",gap:10,color:"#10b981",fontSize:13,padding:"10px 0"}}>
      <Spin/><span style={{animation:"pulse 1.5s ease infinite"}}>{text}</span>
    </div>
  );
}
function Pill({children,color="#10b981",lg=false}){
  return(
    <span style={{background:color+"22",color,border:`1px solid ${color}44`,borderRadius:20,padding:lg?"4px 12px":"2px 8px",fontSize:lg?12:11,fontWeight:600,whiteSpace:"nowrap"}}>
      {children}
    </span>
  );
}
function Card({children,style={},color="#1e3a5f"}){
  return(
    <div style={{background:"#0d1b2e",border:`1px solid ${color}`,borderRadius:14,padding:20,...style}}>
      {children}
    </div>
  );
}
function NavBtn({icon,label,active,onClick,badge}){
  return(
    <button onClick={onClick} style={{width:"100%",display:"flex",alignItems:"center",gap:10,padding:"9px 12px",borderRadius:8,background:active?"#10b98118":"transparent",border:active?"1px solid #10b98144":"1px solid transparent",color:active?"#10b981":"#4b6280",cursor:"pointer",fontSize:13,fontWeight:active?600:400,textAlign:"left"}}>
      <span style={{fontSize:17,minWidth:20,textAlign:"center"}}>{icon}</span>
      <span style={{flex:1}}>{label}</span>
      {badge&&<Pill color="#ef4444">{badge}</Pill>}
    </button>
  );
}

// ── Live AQI Gauge ────────────────────────────────────────────────────────────
function AQIGauge({aqi, size=120}){
  const color = aqiColor(aqi);
  const pct = Math.min(1, aqi/300);
  const angle = -140 + pct*280;
  const r=46, cx=60, cy=65;
  const startAngle = -140*(Math.PI/180);
  const endAngle = (angle)*(Math.PI/180);
  const x1=cx+r*Math.cos(startAngle), y1=cy+r*Math.sin(startAngle);
  const x2=cx+r*Math.cos(endAngle), y2=cy+r*Math.sin(endAngle);
  const largeArc=pct>0.5?1:0;
  return(
    <svg width={size} height={size*0.75} viewBox="0 0 120 90">
      <path d={`M ${cx+r*Math.cos(-140*Math.PI/180)} ${cy+r*Math.sin(-140*Math.PI/180)} A ${r} ${r} 0 1 1 ${cx+r*Math.cos(-40*Math.PI/180)} ${cy+r*Math.sin(-40*Math.PI/180)}`}
        fill="none" stroke="#1e293b" strokeWidth="8" strokeLinecap="round"/>
      {aqi>0&&<path d={`M ${x1} ${y1} A ${r} ${r} 0 ${largeArc} 1 ${x2} ${y2}`}
        fill="none" stroke={color} strokeWidth="8" strokeLinecap="round"/>}
      <text x={cx} y={cy+2} textAnchor="middle" fontSize="22" fontWeight="800" fill={color} fontFamily="monospace">{aqi}</text>
      <text x={cx} y={cy+14} textAnchor="middle" fontSize="7" fill="#475569">AQI</text>
      <text x={cx} y={cy+24} textAnchor="middle" fontSize="7" fill={color} fontWeight="700">{aqiLabel(aqi)}</text>
    </svg>
  );
}

// ── Location Picker (CORE FEATURE) ────────────────────────────────────────────
function LocationPicker({onLocationSet}){
  const [mode,setMode]=useState("auto"); // auto | search | manual
  const [loading,setLoading]=useState(false);
  const [search,setSearch]=useState("");
  const [results,setResults]=useState([]);
  const [searching,setSearching]=useState(false);
  const [error,setError]=useState("");

  const PRESET_CITIES=[
    {name:"Karachi, Pakistan",lat:24.8607,lng:67.0011},
    {name:"Lahore, Pakistan",lat:31.5204,lng:74.3587},
    {name:"Islamabad, Pakistan",lat:33.6844,lng:73.0479},
    {name:"Delhi, India",lat:28.6139,lng:77.2090},
    {name:"Mumbai, India",lat:19.0760,lng:72.8777},
    {name:"Dhaka, Bangladesh",lat:23.8103,lng:90.4125},
    {name:"Beijing, China",lat:39.9042,lng:116.4074},
    {name:"Bangkok, Thailand",lat:13.7563,lng:100.5018},
    {name:"Cairo, Egypt",lat:30.0444,lng:31.2357},
    {name:"Lagos, Nigeria",lat:6.5244,lng:3.3792},
    {name:"London, UK",lat:51.5074,lng:-0.1278},
    {name:"New York, USA",lat:40.7128,lng:-74.0060},
    {name:"Los Angeles, USA",lat:34.0522,lng:-118.2437},
    {name:"Tokyo, Japan",lat:35.6762,lng:139.6503},
    {name:"Paris, France",lat:48.8566,lng:2.3522},
    {name:"Dubai, UAE",lat:25.2048,lng:55.2708},
  ];

  function autoDetect(){
    setLoading(true);setError("");
    if(!navigator.geolocation){setError("GPS not supported by your browser.");setLoading(false);return;}
    navigator.geolocation.getCurrentPosition(
      async pos=>{
        const{latitude:lat,longitude:lng}=pos.coords;
        // Reverse geocode
        try{
          const r=await fetch(`https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lng}&format=json`,{headers:{"User-Agent":"EcoGuardianAI/1.0"}});
          const d=await r.json();
          const name=d.address?.city||d.address?.town||d.address?.state||"Your Location";
          const country=d.address?.country||"";
          onLocationSet({name:`${name}, ${country}`,lat,lng,autoDetected:true});
        }catch{
          onLocationSet({name:`Lat ${lat.toFixed(2)}, Lng ${lng.toFixed(2)}`,lat,lng,autoDetected:true});
        }
        setLoading(false);
      },
      err=>{
        setError("Location access denied. Please allow GPS or search manually.");
        setLoading(false);
      },
      {timeout:10000,enableHighAccuracy:true}
    );
  }

  async function searchCity(){
    if(!search.trim()||search.length<2) return;
    setSearching(true);setResults([]);
    const r=await geocodeCity(search);
    setResults(r.slice(0,5));
    setSearching(false);
    if(r.length===0) setError("No results found. Try a different city name.");
    else setError("");
  }

  return(
    <div style={{minHeight:"100vh",background:"#020c1b",display:"flex",alignItems:"center",justifyContent:"center",padding:20}}>
      <div style={{position:"fixed",inset:0,background:"radial-gradient(ellipse at 20% 40%,#10b98109,transparent 55%),radial-gradient(ellipse at 80% 70%,#0891b20a,transparent 55%)",pointerEvents:"none"}}/>
      <div style={{width:"100%",maxWidth:560,position:"relative"}}>
        {/* Header */}
        <div style={{textAlign:"center",marginBottom:32}}>
          <div style={{fontSize:60,marginBottom:8,filter:"drop-shadow(0 0 24px #10b98155)"}}>🌍</div>
          <div style={{fontSize:30,fontWeight:900,background:"linear-gradient(90deg,#10b981,#0891b2,#6366f1)",WebkitBackgroundClip:"text",WebkitTextFillColor:"transparent",letterSpacing:-1.5}}>EcoGuardian AI</div>
          <div style={{fontSize:12,color:"#334155",letterSpacing:3,fontWeight:600,marginTop:4}}>ENVIRONMENTAL INTELLIGENCE PLATFORM</div>
          <div style={{fontSize:13,color:"#475569",marginTop:10}}>Select your location to load real-time air quality data</div>
        </div>

        <Card color="#1e3a5f">
          {/* Mode tabs */}
          <div style={{display:"grid",gridTemplateColumns:"1fr 1fr 1fr",gap:6,marginBottom:20,background:"#060e1a",borderRadius:10,padding:4}}>
            {[["auto","📍 Auto GPS"],["search","🔍 Search City"],["manual","🗺️ Pick City"]].map(([m,l])=>(
              <button key={m} onClick={()=>{setMode(m);setError("");setResults([]);}} style={{padding:"9px 4px",borderRadius:7,border:"none",background:mode===m?"#1e3a5f":"transparent",color:mode===m?"#93c5fd":"#334155",fontSize:12,fontWeight:mode===m?700:400,cursor:"pointer"}}>
                {l}
              </button>
            ))}
          </div>

          {/* Auto GPS */}
          {mode==="auto"&&(
            <div style={{textAlign:"center",padding:"10px 0"}}>
              <div style={{fontSize:48,marginBottom:12}}>📡</div>
              <div style={{fontSize:14,color:"#64748b",marginBottom:6}}>Automatically detect your location</div>
              <div style={{fontSize:12,color:"#334155",marginBottom:20}}>Uses your device GPS to find the nearest air quality sensors</div>
              <button onClick={autoDetect} disabled={loading} style={{background:"linear-gradient(90deg,#059669,#0891b2)",color:"#fff",border:"none",borderRadius:10,padding:"13px 32px",fontSize:14,fontWeight:700,cursor:"pointer",display:"inline-flex",alignItems:"center",gap:10,opacity:loading?.7:1}}>
                {loading?<><Spin color="#fff" size={16}/> Detecting location…</>:<>📍 Detect My Location</>}
              </button>
              {error&&<div style={{marginTop:12,color:"#fca5a5",fontSize:12,padding:"8px 12px",background:"#ef444411",borderRadius:8}}>{error}</div>}
              <div style={{marginTop:16,fontSize:11,color:"#1e3a5f"}}>Your location is only used locally — never stored or sent anywhere except to fetch public air quality data.</div>
            </div>
          )}

          {/* Search */}
          {mode==="search"&&(
            <div>
              <div style={{display:"flex",gap:8,marginBottom:12}}>
                <input value={search} onChange={e=>setSearch(e.target.value)} onKeyDown={e=>e.key==="Enter"&&searchCity()}
                  placeholder="Type any city name — e.g. Lahore, Tokyo, London…"
                  style={{flex:1,background:"#060e1a",border:"1px solid #1e3a5f",borderRadius:8,padding:"11px 14px",color:"#e2e8f0",fontSize:13,outline:"none"}}
                  autoFocus/>
                <button onClick={searchCity} disabled={searching||!search.trim()} style={{background:"#10b981",color:"#fff",border:"none",borderRadius:8,padding:"11px 18px",fontSize:13,fontWeight:700,cursor:"pointer",opacity:searching?.6:1}}>
                  {searching?<Spin color="#fff"/>:"Search"}
                </button>
              </div>
              {searching&&<Loader text="Searching cities…"/>}
              {error&&<div style={{color:"#fca5a5",fontSize:12,marginBottom:10}}>{error}</div>}
              {results.map((r,i)=>(
                <div key={i} onClick={()=>onLocationSet({...r,autoDetected:false})}
                  style={{padding:"12px 14px",background:"#060e1a",borderRadius:8,marginBottom:6,cursor:"pointer",border:"1px solid #1e293b",display:"flex",justifyContent:"space-between",alignItems:"center",transition:"border-color .15s"}}
                  onMouseEnter={e=>e.currentTarget.style.borderColor="#10b981"}
                  onMouseLeave={e=>e.currentTarget.style.borderColor="#1e293b"}>
                  <div>
                    <div style={{fontSize:13,color:"#e2e8f0",fontWeight:500}}>{r.name}</div>
                    <div style={{fontSize:11,color:"#334155"}}>Lat {r.lat.toFixed(4)}, Lng {r.lng.toFixed(4)}</div>
                  </div>
                  <span style={{color:"#10b981",fontSize:18}}>→</span>
                </div>
              ))}
            </div>
          )}

          {/* Preset cities */}
          {mode==="manual"&&(
            <div>
              <div style={{fontSize:12,color:"#475569",marginBottom:12}}>Select from major cities worldwide:</div>
              <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:6,maxHeight:320,overflowY:"auto"}}>
                {PRESET_CITIES.map(c=>(
                  <button key={c.name} onClick={()=>onLocationSet({...c,autoDetected:false})}
                    style={{background:"#060e1a",border:"1px solid #1e293b",borderRadius:8,padding:"10px 12px",cursor:"pointer",textAlign:"left",transition:"border-color .15s"}}
                    onMouseEnter={e=>e.currentTarget.style.borderColor="#10b981"}
                    onMouseLeave={e=>e.currentTarget.style.borderColor="#1e293b"}>
                    <div style={{fontSize:12,color:"#cbd5e1",fontWeight:500}}>{c.name.split(",")[0]}</div>
                    <div style={{fontSize:10,color:"#334155"}}>{c.name.split(",").slice(1).join(",").trim()}</div>
                  </button>
                ))}
              </div>
            </div>
          )}
        </Card>
      </div>
    </div>
  );
}

// ── Data Loading Screen ───────────────────────────────────────────────────────
function DataLoader({location, onDataLoaded}){
  const [step,setStep]=useState(0);
  const [error,setError]=useState("");
  const steps=["📡 Connecting to sensors…","🌡️ Fetching weather data…","💨 Loading air quality readings…","🤖 Running AI analysis…","✅ All systems ready!"];

  useEffect(()=>{
    let mounted=true;
    async function load(){
      try{
        setStep(0); await new Promise(r=>setTimeout(r,600));
        if(!mounted) return;
        setStep(1);
        const weather=await fetchWeather(location.lat,location.lng);
        if(!mounted) return;
        setStep(2);
        const aqData=await fetchOpenAQ(location.lat,location.lng,location.name);
        if(!mounted) return;
        setStep(3); await new Promise(r=>setTimeout(r,700));
        if(!mounted) return;
        setStep(4); await new Promise(r=>setTimeout(r,500));
        if(!mounted) return;
        onDataLoaded({weather,aqData,location,loadedAt:new Date()});
      }catch(e){
        if(mounted) setError("Failed to load data: "+e.message);
      }
    }
    load();
    return()=>{mounted=false;};
  },[]);

  return(
    <div style={{minHeight:"100vh",background:"#020c1b",display:"flex",alignItems:"center",justifyContent:"center"}}>
      <div style={{textAlign:"center",maxWidth:400,padding:20}}>
        <div style={{fontSize:56,marginBottom:16,animation:"pulse 2s ease infinite"}}>🌍</div>
        <div style={{fontSize:18,fontWeight:700,color:"#10b981",marginBottom:6}}>{location.name}</div>
        <div style={{fontSize:13,color:"#475569",marginBottom:28}}>Loading environmental data…</div>
        <div style={{display:"flex",flexDirection:"column",gap:10}}>
          {steps.map((s,i)=>(
            <div key={i} style={{display:"flex",alignItems:"center",gap:12,padding:"10px 16px",background:i<step?"#10b98111":i===step?"#0d1b2e":"#060e1a",border:`1px solid ${i<step?"#10b98133":i===step?"#10b98155":"#0d1b2e"}`,borderRadius:8,transition:"all .3s"}}>
              {i<step?<span style={{fontSize:16}}>✅</span>:i===step?<Spin color="#10b981"/>:<span style={{fontSize:16,opacity:.2}}>⏳</span>}
              <span style={{fontSize:13,color:i<step?"#10b981":i===step?"#e2e8f0":"#1e3a5f"}}>{s}</span>
            </div>
          ))}
        </div>
        {error&&<div style={{marginTop:16,color:"#fca5a5",fontSize:12,padding:"10px",background:"#ef444411",borderRadius:8}}>{error}</div>}
      </div>
    </div>
  );
}

// ── Smart Dashboard ───────────────────────────────────────────────────────────
function Dashboard({envData, location, onRefresh, refreshing}){
  const {weather,aqData}=envData;
  const [aiInsight,setAiInsight]=useState("");
  const [loadingInsight,setLoadingInsight]=useState(false);
  const hasRealData = aqData.success && aqData.stations?.length>0;

  // Build summary from real data or use estimates
  const stations = hasRealData ? aqData.stations : [];
  const avgAQI = hasRealData
    ? Math.round(stations.reduce((a,s)=>a+estimateAQI(s.params),0)/stations.length)
    : 85; // fallback

  async function getAIInsight(){
    setLoadingInsight(true);
    const ctx=`Location: ${location.name}\nAQI: ${avgAQI}\nWeather: ${weather.success?`Temp:${weather.temp}°C, Humidity:${weather.humidity}%, Wind:${weather.windSpeed}km/h`:"unavailable"}\nData source: ${hasRealData?aqData.source:"estimated"}\nStations found: ${stations.length}`;
    const r=await askAI("You are EcoGuardian AI, an expert environmental scientist. Provide a concise, actionable 3-sentence environmental intelligence briefing based on current air quality data. Include health impact, main cause, and top recommendation.",ctx);
    setAiInsight(r);setLoadingInsight(false);
  }

  useEffect(()=>{
    if(!aiInsight) getAIInsight();
  },[envData]);

  return(
    <div className="fade" style={{display:"flex",flexDirection:"column",gap:18}}>
      {/* Location + refresh bar */}
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",padding:"14px 18px",background:"#0d1b2e",border:"1px solid #1e3a5f",borderRadius:12}}>
        <div style={{display:"flex",alignItems:"center",gap:12}}>
          <span style={{fontSize:28}}>📍</span>
          <div>
            <div style={{fontSize:16,fontWeight:700,color:"#e2e8f0"}}>{location.name}</div>
            <div style={{fontSize:11,color:"#334155"}}>Lat {location.lat.toFixed(4)}, Lng {location.lng.toFixed(4)} · {location.autoDetected?"Auto-detected GPS":"Manual selection"}</div>
          </div>
        </div>
        <div style={{display:"flex",gap:10,alignItems:"center"}}>
          <Pill color={hasRealData?"#10b981":"#fbbf24"}>{hasRealData?"✓ Live Sensor Data":"⚡ Estimated Data"}</Pill>
          <Pill color="#60a5fa">{aqData.source||"Fallback"}</Pill>
          <button onClick={onRefresh} disabled={refreshing} style={{background:"#1e3a5f",color:"#93c5fd",border:"1px solid #2563eb44",borderRadius:8,padding:"8px 14px",fontSize:12,fontWeight:600,cursor:"pointer",display:"flex",alignItems:"center",gap:6}}>
            {refreshing?<Spin size={12} color="#93c5fd"/>:<span>🔄</span>}
            Refresh
          </button>
        </div>
      </div>

      {/* Weather + AQI main cards */}
      <div style={{display:"grid",gridTemplateColumns:"200px 1fr",gap:16}}>
        {/* AQI Gauge */}
        <Card style={{display:"flex",flexDirection:"column",alignItems:"center",justifyContent:"center",padding:"20px 10px"}}>
          <AQIGauge aqi={avgAQI} size={160}/>
          <div style={{marginTop:8,fontSize:28}}>{aqiEmoji(avgAQI)}</div>
          <div style={{fontSize:11,color:"#334155",marginTop:4,textAlign:"center"}}>
            {hasRealData?`${stations.length} live sensors`:"Estimated value"}
          </div>
        </Card>
        {/* Weather + stats */}
        <div style={{display:"grid",gridTemplateColumns:"1fr 1fr 1fr 1fr",gap:10}}>
          {weather.success&&[
            ["🌡️","Temperature",weather.temp+"°C","#f472b6"],
            ["💧","Humidity",weather.humidity+"%","#60a5fa"],
            ["💨","Wind Speed",weather.windSpeed+" km/h","#34d399"],
            ["🧭","Wind Dir",weather.windDir+"°","#a78bfa"],
          ].map(([icon,label,value,color])=>(
            <Card key={label} style={{padding:"14px 12px"}}>
              <div style={{fontSize:20,marginBottom:6}}>{icon}</div>
              <div style={{fontSize:10,color:"#334155",letterSpacing:.8,textTransform:"uppercase",marginBottom:4}}>{label}</div>
              <div style={{fontSize:24,fontWeight:800,color,fontFamily:"monospace"}}>{value}</div>
              <Pill color="#60a5fa">Live</Pill>
            </Card>
          ))}
          {!weather.success&&<div style={{gridColumn:"span 4",display:"flex",alignItems:"center",justifyContent:"center",color:"#334155",fontSize:13}}>Weather data unavailable</div>}
        </div>
      </div>

      {/* Pollutant readings from real sensors */}
      {hasRealData&&(
        <Card>
          <div style={{fontSize:14,fontWeight:700,color:"#94a3b8",marginBottom:14,display:"flex",justifyContent:"space-between",alignItems:"center"}}>
            <span>📡 Live Sensor Readings</span>
            <Pill color="#10b981">Updated {envData.loadedAt.toLocaleTimeString()}</Pill>
          </div>
          <div style={{display:"grid",gridTemplateColumns:"repeat(auto-fill,minmax(220px,1fr))",gap:10}}>
            {stations.slice(0,6).map((s,i)=>{
              const aqi=estimateAQI(s.params);
              const mainParam=s.params.pm25||s.params.pm10||s.params.no2||Object.values(s.params)[0];
              return(
                <div key={i} style={{background:"#060e1a",border:`1px solid ${aqiColor(aqi)}33`,borderRadius:10,padding:"12px 14px"}}>
                  <div style={{fontSize:12,fontWeight:600,color:"#94a3b8",marginBottom:6,whiteSpace:"nowrap",overflow:"hidden",textOverflow:"ellipsis"}}>{s.name.length>30?s.name.slice(0,28)+"…":s.name}</div>
                  <div style={{display:"flex",justifyContent:"space-between",alignItems:"center"}}>
                    <div>
                      <div style={{fontSize:24,fontWeight:800,color:aqiColor(aqi),fontFamily:"monospace"}}>{aqi}</div>
                      <div style={{fontSize:10,color:"#334155"}}>AQI (estimated)</div>
                    </div>
                    <div style={{textAlign:"right"}}>
                      {mainParam&&<div style={{fontSize:12,color:"#64748b"}}>{Object.keys(s.params)[0].toUpperCase()}: <strong style={{color:"#e2e8f0"}}>{mainParam.value?.toFixed(1)} {mainParam.unit}</strong></div>}
                      <div style={{marginTop:4}}><Pill color={aqiColor(aqi)}>{aqiLabel(aqi)}</Pill></div>
                    </div>
                  </div>
                </div>
              );
            })}
          </div>
        </Card>
      )}

      {/* No real data notice */}
      {!hasRealData&&(
        <div style={{background:"#fbbf2411",border:"1px solid #fbbf2433",borderRadius:10,padding:"14px 18px",display:"flex",gap:12,alignItems:"center"}}>
          <span style={{fontSize:24}}>ℹ️</span>
          <div>
            <div style={{fontSize:13,fontWeight:600,color:"#fbbf24",marginBottom:3}}>Limited sensor coverage for this area</div>
            <div style={{fontSize:12,color:"#92400e"}}>No OpenAQ sensors found within 50km. Dashboard shows estimated values. You can still use all AI features and upload your own data in the Data tab.</div>
          </div>
        </div>
      )}

      {/* AI Insight */}
      <Card color="#10b98133" style={{background:"linear-gradient(135deg,#0d2040,#060e1a)",animation:"glow 4s ease infinite"}}>
        <div style={{display:"flex",gap:14,alignItems:"flex-start"}}>
          <span style={{fontSize:32,lineHeight:1}}>🤖</span>
          <div style={{flex:1}}>
            <div style={{fontSize:14,fontWeight:700,color:"#10b981",marginBottom:8}}>EcoGuardian AI — Live Environmental Briefing</div>
            {loadingInsight?<Loader text="AI analyzing your local environmental data…"/>:(
              <div className="fade" style={{fontSize:13,color:"#94a3b8",lineHeight:1.8}}>{aiInsight||"Click refresh to get AI analysis."}</div>
            )}
            <button onClick={getAIInsight} disabled={loadingInsight} style={{marginTop:10,background:"transparent",border:"1px solid #10b98144",borderRadius:7,padding:"6px 14px",color:"#10b981",fontSize:12,cursor:"pointer",fontWeight:600}}>
              {loadingInsight?"Analyzing…":"🔄 Refresh AI Insight"}
            </button>
          </div>
        </div>
      </Card>
    </div>
  );
}

// ── AI Chat ───────────────────────────────────────────────────────────────────
function Chatbot({envData,location}){
  const {weather,aqData}=envData;
  const hasReal=aqData.success&&aqData.stations?.length>0;
  const ctx=`Location: ${location.name} (${location.lat.toFixed(3)}, ${location.lng.toFixed(3)})\nLive AQI: ${hasReal?estimateAQI(aqData.stations[0]?.params||{}):"~85 (estimated)"}\nWeather: ${weather.success?`${weather.temp}°C, ${weather.humidity}% humidity, wind ${weather.windSpeed}km/h`:"unavailable"}\nSensors found: ${aqData.stations?.length||0}\nData: ${hasReal?"Live OpenAQ sensors":"Estimated"}`;
  const SYS=`You are EcoGuardian AI, a world-class environmental intelligence assistant. You help cities, governments, schools, and citizens make smart environmental decisions.\n\nCurrent monitoring context:\n${ctx}\n\nExpertise: air quality (AQI, PM2.5, PM10, CO, NO₂, SO₂), climate science, environmental health, urban sustainability, policy, green technology, carbon management.\n\nAlways be specific, data-driven, and action-oriented. Reference the user's real local data when relevant.`;

  const [msgs,setMsgs]=useState([{role:"assistant",content:`Hello! I'm EcoGuardian AI 🌿\n\nI've loaded real-time environmental data for **${location.name}**. I can see ${aqData.stations?.length||0} air quality sensor${aqData.stations?.length!==1?"s":""} in your area.\n\nAsk me anything about your local air quality, health risks, sustainability strategies, or environmental policy!`}]);
  const [input,setInput]=useState("");
  const [loading,setLoading]=useState(false);
  const bottomRef=useRef(null);
  useEffect(()=>{bottomRef.current?.scrollIntoView({behavior:"smooth"});},[msgs,loading]);

  async function send(msg){
    const m=msg||input.trim();if(!m||loading)return;
    setInput("");
    const newMsgs=[...msgs,{role:"user",content:m}];
    setMsgs(newMsgs);setLoading(true);
    const hist=newMsgs.slice(1,-1).map(x=>({role:x.role,content:x.content}));
    const r=await askAI(SYS,m,hist);
    setMsgs([...newMsgs,{role:"assistant",content:r}]);
    setLoading(false);
  }

  const QUICK=[
    `Is it safe to go outside in ${location.name.split(",")[0]} today?`,
    "What's causing the current pollution levels?",
    "Health tips for today's air quality",
    "Best time of day to exercise outdoors?",
    "How does wind affect pollution dispersal?",
    "Green initiatives for this city"
  ];

  function renderText(t){
    return t.split(/(\*\*[^*]+\*\*)/g).map((p,i)=>
      p.startsWith("**")&&p.endsWith("**")?<strong key={i} style={{color:"#10b981"}}>{p.slice(2,-2)}</strong>:p
    );
  }

  return(
    <div className="fade" style={{display:"flex",flexDirection:"column",height:580}}>
      {/* Context bar */}
      <div style={{padding:"8px 12px",background:"#060e1a",borderRadius:8,marginBottom:12,display:"flex",gap:10,alignItems:"center",flexWrap:"wrap"}}>
        <span style={{fontSize:11,color:"#334155",fontWeight:600}}>AI CONTEXT:</span>
        <Pill color="#10b981">📍 {location.name.split(",")[0]}</Pill>
        {aqData.stations?.length>0&&<Pill color="#60a5fa">📡 {aqData.stations.length} sensors</Pill>}
        {weather.success&&<Pill color="#f472b6">🌡️ {weather.temp}°C</Pill>}
        {weather.success&&<Pill color="#60a5fa">💧 {weather.humidity}%</Pill>}
      </div>
      <div style={{flex:1,overflowY:"auto",display:"flex",flexDirection:"column",gap:10,paddingBottom:10}}>
        {msgs.map((m,i)=>(
          <div key={i} style={{display:"flex",justifyContent:m.role==="user"?"flex-end":"flex-start",gap:8,alignItems:"flex-end"}}>
            {m.role==="assistant"&&<div style={{width:30,height:30,borderRadius:"50%",background:"#10b98122",border:"1px solid #10b98144",display:"flex",alignItems:"center",justifyContent:"center",fontSize:14,flexShrink:0}}>🌿</div>}
            <div style={{maxWidth:"78%",background:m.role==="user"?"#1a3357":"#0d1b2e",border:`1px solid ${m.role==="user"?"#2563eb33":"#1e293b"}`,borderRadius:m.role==="user"?"14px 14px 4px 14px":"14px 14px 14px 4px",padding:"10px 14px",fontSize:13,color:"#cbd5e1",lineHeight:1.75,whiteSpace:"pre-wrap"}}>
              {renderText(m.content)}
            </div>
            {m.role==="user"&&<div style={{width:30,height:30,borderRadius:"50%",background:"#1e3a5f",border:"1px solid #2563eb44",display:"flex",alignItems:"center",justifyContent:"center",fontSize:13,flexShrink:0}}>👤</div>}
          </div>
        ))}
        {loading&&(
          <div style={{display:"flex",gap:8,alignItems:"flex-end"}}>
            <div style={{width:30,height:30,borderRadius:"50%",background:"#10b98122",border:"1px solid #10b98144",display:"flex",alignItems:"center",justifyContent:"center",fontSize:14}}>🌿</div>
            <div style={{background:"#0d1b2e",border:"1px solid #1e293b",borderRadius:"14px 14px 14px 4px",padding:"10px 14px"}}><Loader/></div>
          </div>
        )}
        <div ref={bottomRef}/>
      </div>
      <div style={{display:"flex",gap:5,flexWrap:"wrap",borderTop:"1px solid #1e293b",padding:"8px 0",marginBottom:8}}>
        {QUICK.map(q=>(
          <button key={q} onClick={()=>send(q)} style={{background:"#060e1a",border:"1px solid #1e293b",borderRadius:20,padding:"4px 10px",fontSize:11,color:"#475569",cursor:"pointer"}}
            onMouseEnter={e=>{e.currentTarget.style.borderColor="#10b981";e.currentTarget.style.color="#10b981";}}
            onMouseLeave={e=>{e.currentTarget.style.borderColor="#1e293b";e.currentTarget.style.color="#475569";}}>
            {q}
          </button>
        ))}
      </div>
      <div style={{display:"flex",gap:8}}>
        <input value={input} onChange={e=>setInput(e.target.value)} onKeyDown={e=>e.key==="Enter"&&!e.shiftKey&&send()}
          placeholder={`Ask about ${location.name.split(",")[0]}'s air quality, health, sustainability…`}
          style={{flex:1,background:"#060e1a",border:"1px solid #1e3a5f",borderRadius:10,padding:"12px 16px",color:"#e2e8f0",fontSize:13,outline:"none"}}/>
        <button onClick={()=>send()} disabled={loading||!input.trim()} style={{background:"linear-gradient(90deg,#059669,#0891b2)",color:"#fff",border:"none",borderRadius:10,padding:"12px 20px",fontSize:13,fontWeight:700,cursor:"pointer",opacity:loading?.6:1}}>Send ➤</button>
      </div>
    </div>
  );
}

// ── Data Upload & Analysis ────────────────────────────────────────────────────
function DataTab({location}){
  const [csv,setCsv]=useState([]);
  const [fname,setFname]=useState("");
  const [analyzing,setAnalyzing]=useState(false);
  const [insight,setInsight]=useState("");
  const [dragOver,setDragOver]=useState(false);

  function parseFile(file){
    if(!file) return;
    setFname(file.name);setInsight("");
    const r=new FileReader();
    r.onload=ev=>{
      const lines=ev.target.result.split("\n").filter(l=>l.trim());
      setCsv(lines.map(l=>l.split(",")));
    };
    r.readAsText(file);
  }

  async function analyze(){
    if(!csv.length)return;
    setAnalyzing(true);setInsight("");
    const preview=csv.slice(0,8).map(r=>r.join(", ")).join("\n");
    const r=await askAI(
      "You are an expert environmental data scientist. Analyze pollution CSV datasets and provide actionable insights with specific numbers.",
      `File: ${fname} | Location context: ${location.name}\nHeaders: ${csv[0].join(", ")}\nTotal rows: ${csv.length-1}\nSample data:\n${preview}\n\nProvide:\n1. Data Quality Assessment (completeness, validity)\n2. Key Pollutant Analysis (identify worst parameters)\n3. Anomaly Detection (flag unusual values)\n4. Statistical Highlights (min, max, averages if visible)\n5. Health Risk Summary\n6. Top 3 Recommendations\n7. Data Collection Improvements`
    );
    setInsight(r);setAnalyzing(false);
  }

  const SAMPLE_CSV=`zone,date,time,aqi,pm25,pm10,co,no2,so2,temperature,humidity
Industrial District,2024-01-15,08:00,187,68,112,9.2,78,42,22,65
Downtown Core,2024-01-15,08:00,142,47,88,6.1,52,28,23,60
Port Area,2024-01-15,08:00,165,58,98,8.1,65,35,21,70
Residential North,2024-01-15,08:00,89,28,54,3.2,31,14,24,55
University District,2024-01-15,08:00,62,18,38,1.8,22,9,25,50
Coastal Belt,2024-01-15,08:00,44,12,26,1.1,14,5,26,75
Green Valley,2024-01-15,08:00,31,8,18,0.7,9,3,27,45`;

  function loadSample(){
    const lines=SAMPLE_CSV.split("\n");
    setCsv(lines.map(l=>l.split(",")));
    setFname("sample_pollution_data.csv");
    setInsight("");
  }

  return(
    <div className="fade" style={{display:"flex",flexDirection:"column",gap:16}}>
      {/* Upload zone */}
      <Card
        style={{border:dragOver?"2px dashed #10b981":"2px dashed #1e3a5f",textAlign:"center",padding:"28px 20px",transition:"border-color .2s",cursor:"pointer"}}
        onDragOver={e=>{e.preventDefault();setDragOver(true);}}
        onDragLeave={()=>setDragOver(false)}
        onDrop={e=>{e.preventDefault();setDragOver(false);parseFile(e.dataTransfer.files[0]);}}>
        <div style={{fontSize:44,marginBottom:10}}>📂</div>
        <div style={{fontSize:14,fontWeight:600,color:"#64748b",marginBottom:4}}>
          {fname?`✅ ${fname}`:"Drag & drop your CSV file here"}
        </div>
        <div style={{fontSize:12,color:"#334155",marginBottom:16}}>
          {csv.length>0?`${csv.length-1} data rows loaded`:"Supports CSV files with pollution data"}
        </div>
        <div style={{display:"flex",gap:10,justifyContent:"center",flexWrap:"wrap"}}>
          <label style={{background:"linear-gradient(90deg,#059669,#0891b2)",color:"#fff",borderRadius:8,padding:"10px 20px",fontSize:13,fontWeight:700,cursor:"pointer",display:"inline-block"}}>
            📁 Choose File
            <input type="file" accept=".csv,.txt" onChange={e=>parseFile(e.target.files[0])} style={{display:"none"}}/>
          </label>
          <button onClick={loadSample} style={{background:"#1e3a5f",color:"#93c5fd",border:"1px solid #2563eb44",borderRadius:8,padding:"10px 20px",fontSize:13,fontWeight:600,cursor:"pointer"}}>
            📋 Load Sample Data
          </button>
        </div>
      </Card>

      {/* CSV format guide */}
      {!csv.length&&(
        <Card>
          <div style={{fontSize:13,fontWeight:700,color:"#64748b",marginBottom:12}}>📝 Expected CSV Format</div>
          <div style={{background:"#060e1a",borderRadius:8,padding:14,overflowX:"auto"}}>
            <pre style={{fontSize:11,color:"#10b981",margin:0,lineHeight:1.8,fontFamily:"monospace"}}>
{`zone, date, time, aqi, pm25, pm10, co, no2, so2, temperature, humidity
Industrial District, 2024-01-15, 08:00, 187, 68, 112, 9.2, 78, 42, 22, 65
Downtown Core, 2024-01-15, 08:00, 142, 47, 88, 6.1, 52, 28, 23, 60`}
            </pre>
          </div>
          <div style={{marginTop:12,display:"grid",gridTemplateColumns:"1fr 1fr 1fr",gap:8}}>
            {[["zone","Area/district name"],["aqi","Air Quality Index"],["pm25","Fine particles µg/m³"],["pm10","Coarse particles µg/m³"],["co","Carbon monoxide ppm"],["no2","Nitrogen dioxide µg/m³"]].map(([k,v])=>(
              <div key={k} style={{padding:"6px 10px",background:"#060e1a",borderRadius:6}}>
                <div style={{fontSize:11,color:"#10b981",fontWeight:700,fontFamily:"monospace"}}>{k}</div>
                <div style={{fontSize:10,color:"#334155"}}>{v}</div>
              </div>
            ))}
          </div>
        </Card>
      )}

      {/* Data preview */}
      {csv.length>0&&(
        <Card>
          <div style={{fontSize:13,fontWeight:700,color:"#64748b",marginBottom:10,display:"flex",justifyContent:"space-between"}}>
            <span>📋 Data Preview</span>
            <span style={{color:"#10b981"}}>{csv.length-1} rows × {csv[0].length} columns</span>
          </div>
          <div style={{overflowX:"auto",borderRadius:8,border:"1px solid #1e293b"}}>
            <table style={{width:"100%",borderCollapse:"collapse",fontSize:12}}>
              <thead>
                <tr style={{background:"#060e1a"}}>
                  {csv[0].map((h,i)=><th key={i} style={{padding:"8px 12px",color:"#10b981",textAlign:"left",borderBottom:"1px solid #1e293b",whiteSpace:"nowrap",fontWeight:700}}>{h.trim()}</th>)}
                </tr>
              </thead>
              <tbody>
                {csv.slice(1,7).map((row,i)=>(
                  <tr key={i} style={{borderBottom:"1px solid #0d1b2e",background:i%2===0?"transparent":"#060e1a"}}>
                    {row.map((c,j)=><td key={j} style={{padding:"8px 12px",color:"#94a3b8",whiteSpace:"nowrap"}}>{c.trim()}</td>)}
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
          {csv.length>8&&<div style={{fontSize:11,color:"#334155",marginTop:6,textAlign:"center"}}>Showing 6 of {csv.length-1} rows</div>}
        </Card>
      )}

      {csv.length>0&&(
        <button onClick={analyze} disabled={analyzing} style={{background:"linear-gradient(90deg,#059669,#0891b2)",color:"#fff",border:"none",borderRadius:10,padding:"14px",fontSize:14,fontWeight:800,cursor:"pointer",opacity:analyzing?.6:1}}>
          {analyzing?"🤖 AI Analyzing Your Data…":"🤖 Analyze Data with AI"}
        </button>
      )}
      {analyzing&&<Card><Loader text="Running environmental data analysis…"/></Card>}
      {insight&&(
        <Card className="fade" color="#10b98133">
          <div style={{fontSize:14,fontWeight:700,color:"#10b981",marginBottom:12}}>🤖 AI Data Analysis — {fname}</div>
          <div style={{fontSize:13,color:"#cbd5e1",lineHeight:1.85,whiteSpace:"pre-wrap"}}>{insight}</div>
        </Card>
      )}
    </div>
  );
}

// ── Report Generator ──────────────────────────────────────────────────────────
function Reports({envData,location}){
  const {weather,aqData}=envData;
  const hasReal=aqData.success&&aqData.stations?.length>0;
  const avgAQI=hasReal?Math.round(aqData.stations.reduce((a,s)=>a+estimateAQI(s.params),0)/aqData.stations.length):85;
  const ctx=`Location: ${location.name} | AQI: ${avgAQI} | Weather: ${weather.success?weather.temp+"°C, "+weather.humidity+"% humidity":"N/A"} | Sensors: ${aqData.stations?.length||0} | Data: ${hasReal?"Live":"Estimated"}`;

  const [type,setType]=useState("exec");
  const [loading,setLoading]=useState(false);
  const [report,setReport]=useState("");
  const [copied,setCopied]=useState(false);

  const TYPES=[
    {id:"exec",icon:"📄",label:"Executive Summary",color:"#60a5fa"},
    {id:"health",icon:"🏥",label:"Health Report",color:"#f472b6"},
    {id:"policy",icon:"🏛️",label:"Policy Brief",color:"#a78bfa"},
    {id:"sustain",icon:"♻️",label:"Sustainability",color:"#34d399"},
    {id:"alert",icon:"🚨",label:"Alert Report",color:"#ef4444"},
  ];
  const t=TYPES.find(x=>x.id===type);
  const PROMPTS={
    exec:`Generate a 400-word executive environmental intelligence report for ${location.name}.\n${ctx}\nInclude: 1) Executive Summary 2) Current Air Quality Status 3) Key Risk Zones 4) Top 5 Actions 5) 30-Day Roadmap`,
    health:`Generate a 400-word public health environmental report for ${location.name}.\n${ctx}\nInclude: 1) Population Risk 2) Vulnerable Groups 3) Disease Burden Estimate 4) Hospital Advisory 5) Protective Measures`,
    policy:`Generate a 400-word environmental policy brief for ${location.name} government.\n${ctx}\nInclude: 1) Policy Context 2) Regulatory Gaps 3) Five Recommended Ordinances 4) Enforcement Plan 5) Budget Estimates`,
    sustain:`Generate a 400-word sustainability assessment for ${location.name}.\n${ctx}\nInclude: 1) Sustainability Score 2) SDG Alignment 3) Six Green Initiatives 4) Renewable Energy Opportunities 5) 2030 Targets`,
    alert:`Generate a 400-word environmental alert bulletin for ${location.name}.\n${ctx}\nInclude: 1) Alert Level 2) Immediate Risks 3) Affected Populations 4) Emergency Response 5) Recovery Timeline`,
  };

  async function gen(){
    setLoading(true);setReport("");
    const r=await askAI("You are a senior environmental consultant. Generate professional reports with ## headers and • bullet points. Be specific with data and actionable.",PROMPTS[type]);
    setReport(r);setLoading(false);
  }

  function renderReport(text){
    return text.split("\n").map((l,i)=>{
      if(l.startsWith("## ")||l.startsWith("# ")) return<div key={i} style={{fontSize:14,fontWeight:800,color:"#10b981",margin:"14px 0 6px",borderBottom:"1px solid #1e293b",paddingBottom:4}}>{l.replace(/^#+\s/,"")}</div>;
      if(l.match(/^[•\-\*] /)) return<div key={i} style={{display:"flex",gap:8,padding:"2px 0 2px 8px",color:"#94a3b8",fontSize:13,lineHeight:1.7}}><span style={{color:"#10b981",flexShrink:0}}>•</span><span>{l.slice(2)}</span></div>;
      if(/^\d+\./.test(l)) return<div key={i} style={{color:"#94a3b8",fontSize:13,lineHeight:1.7,paddingLeft:8}}>{l}</div>;
      if(!l.trim()) return<div key={i} style={{height:6}}/>;
      return<div key={i} style={{color:"#cbd5e1",fontSize:13,lineHeight:1.8}}>{l}</div>;
    });
  }

  return(
    <div className="fade" style={{display:"flex",flexDirection:"column",gap:16}}>
      <div style={{display:"grid",gridTemplateColumns:"repeat(5,1fr)",gap:8}}>
        {TYPES.map(r=>(
          <button key={r.id} onClick={()=>{setType(r.id);setReport("");}} style={{background:type===r.id?r.color+"1a":"#060e1a",border:`1px solid ${type===r.id?r.color:"#1e3a5f"}`,borderRadius:10,padding:"12px 6px",cursor:"pointer",textAlign:"center"}}>
            <div style={{fontSize:24,marginBottom:5}}>{r.icon}</div>
            <div style={{fontSize:10,fontWeight:700,color:type===r.id?r.color:"#4b6280",lineHeight:1.3}}>{r.label}</div>
          </button>
        ))}
      </div>
      <Card style={{border:`1px solid ${t.color}33`}}>
        <div style={{display:"flex",justifyContent:"space-between",alignItems:"center"}}>
          <div style={{display:"flex",alignItems:"center",gap:10}}>
            <span style={{fontSize:26}}>{t.icon}</span>
            <div>
              <div style={{fontSize:15,fontWeight:700,color:t.color}}>{t.label}</div>
              <div style={{fontSize:12,color:"#334155"}}>For {location.name} · {hasReal?"Live data":"Estimated"}</div>
            </div>
          </div>
          <button onClick={gen} disabled={loading} style={{background:`linear-gradient(90deg,${t.color},#0891b2)`,color:"#fff",border:"none",borderRadius:10,padding:"11px 22px",fontSize:13,fontWeight:700,cursor:"pointer",opacity:loading?.6:1}}>
            {loading?"⏳ Generating…":"🚀 Generate"}
          </button>
        </div>
      </Card>
      {loading&&<Card><Loader text="AI generating your report…"/></Card>}
      {report&&(
        <Card className="fade">
          <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:14,paddingBottom:10,borderBottom:"1px solid #1e293b"}}>
            <div style={{display:"flex",gap:8,alignItems:"center"}}>
              <span>{t.icon}</span>
              <div>
                <div style={{fontSize:13,fontWeight:700,color:t.color}}>{t.label} — {location.name.split(",")[0]}</div>
                <div style={{fontSize:11,color:"#334155"}}>Generated {new Date().toLocaleString()}</div>
              </div>
            </div>
            <button onClick={()=>{navigator.clipboard.writeText(report);setCopied(true);setTimeout(()=>setCopied(false),2000);}} style={{background:"#060e1a",border:"1px solid #1e3a5f",borderRadius:7,padding:"6px 14px",color:copied?"#10b981":"#64748b",fontSize:12,fontWeight:600,cursor:"pointer"}}>
              {copied?"✓ Copied!":"📋 Copy"}
            </button>
          </div>
          <div style={{maxHeight:480,overflowY:"auto",paddingRight:6}}>{renderReport(report)}</div>
        </Card>
      )}
    </div>
  );
}

// ── AI Agents ─────────────────────────────────────────────────────────────────
const AGENTS=[
  {id:"health",icon:"🫁",name:"Health AI",color:"#f472b6",
   desc:"Respiratory risk, health impacts, public safety",
   system:"You are HealthGuard AI, an environmental health specialist. Analyze health impacts from real-time pollution data. Give clinical, evidence-based, actionable safety recommendations with specific AQI thresholds and vulnerable population guidance.",
   prompts:["Is it safe to go outside today?","Risk for asthma patients","School closure recommendation?","Emergency health advisory draft","Protective measures for children"]},
  {id:"traffic",icon:"🚦",name:"Traffic AI",color:"#fb923c",
   desc:"Traffic pollution, congestion, routing strategies",
   system:"You are TrafficGuard AI, specializing in traffic emission analysis for smart cities. Analyze congestion-pollution correlations, recommend traffic management interventions, and calculate emission reduction scenarios.",
   prompts:["Best hours to avoid traffic pollution","Electric vehicle impact estimate","Traffic restriction policy","Emission hotspot analysis","Public transit recommendations"]},
  {id:"climate",icon:"🌡️",name:"Climate AI",color:"#60a5fa",
   desc:"Weather impact, AQI forecast, seasonal patterns",
   system:"You are ClimateGuard AI, an atmospheric scientist. Analyze weather-pollution interactions, forecast AQI trends based on meteorological conditions, and identify seasonal pollution patterns.",
   prompts:["How does today's weather affect AQI?","Weekend pollution forecast","Monsoon/winter pollution patterns","Temperature inversion risks","Climate change long-term impact"]},
  {id:"policy",icon:"🏛️",name:"Policy AI",color:"#a78bfa",
   desc:"Government policy, regulations, legal frameworks",
   system:"You are PolicyGuard AI, an environmental governance expert. Draft policies, recommend regulations, create enforcement frameworks, and advise on international environmental standards compliance.",
   prompts:["Draft emergency pollution ordinance","Industrial permit requirements","Environmental fine structure","International standards compliance","Community engagement policy"]},
  {id:"sustain",icon:"♻️",name:"Sustainability AI",color:"#34d399",
   desc:"Sustainability scores, green initiatives, net-zero",
   system:"You are SustainGuard AI, a sustainability intelligence specialist. Calculate sustainability metrics, design green initiatives, create net-zero roadmaps, and identify renewable energy opportunities.",
   prompts:["Calculate my city's sustainability score","Top 5 green initiatives","Net-zero timeline estimate","Solar energy potential","Green building recommendations"]},
];

function AgentsTab({envData,location}){
  const {weather,aqData}=envData;
  const hasReal=aqData.success;
  const ctx=`Location: ${location.name} | AQI: ${hasReal&&aqData.stations?.length?estimateAQI(aqData.stations[0].params):"~85"} | ${aqData.stations?.length||0} sensors | Weather: ${weather.success?weather.temp+"°C, "+weather.humidity+"% humidity, wind "+weather.windSpeed+"km/h":"unavailable"} | Data: ${hasReal?"Live sensors":"Estimated"}`;

  const [active,setActive]=useState(null);
  const [q,setQ]=useState("");
  const [loading,setLoading]=useState(false);
  const [resp,setResp]=useState("");
  const [hist,setHist]=useState([]);
  const ag=AGENTS.find(a=>a.id===active);

  async function run(){
    if(!q.trim()||!ag)return;
    setLoading(true);setResp("");
    const fullQ=`[Real-time context: ${ctx}]\n\nQuestion: ${q}`;
    const r=await askAI(ag.system,fullQ,hist);
    setResp(r);
    setHist(h=>[...h,{role:"user",content:fullQ},{role:"assistant",content:r}]);
    setLoading(false);
  }

  return(
    <div className="fade">
      <div style={{display:"grid",gridTemplateColumns:"repeat(5,1fr)",gap:8,marginBottom:18}}>
        {AGENTS.map(a=>(
          <button key={a.id} onClick={()=>{setActive(a.id);setQ("");setResp("");setHist([]);}} style={{background:active===a.id?a.color+"1a":"#060e1a",border:`1px solid ${active===a.id?a.color:"#1e3a5f"}`,borderRadius:12,padding:"14px 6px",cursor:"pointer",textAlign:"center"}}>
            <div style={{fontSize:26,marginBottom:5}}>{a.icon}</div>
            <div style={{fontSize:11,fontWeight:700,color:active===a.id?a.color:"#4b6280"}}>{a.name}</div>
            <div style={{fontSize:9,color:"#1e3a5f",marginTop:3,lineHeight:1.3}}>{a.desc.split(",")[0]}</div>
          </button>
        ))}
      </div>
      {ag?(
        <Card style={{border:`1px solid ${ag.color}33`}}>
          <div style={{display:"flex",alignItems:"center",gap:12,marginBottom:16,paddingBottom:12,borderBottom:"1px solid #1e293b"}}>
            <span style={{fontSize:28}}>{ag.icon}</span>
            <div>
              <div style={{fontSize:15,fontWeight:700,color:ag.color}}>{ag.name}</div>
              <div style={{fontSize:12,color:"#475569"}}>{ag.desc}</div>
            </div>
            <div style={{marginLeft:"auto",textAlign:"right"}}>
              <Pill color={ag.color}>Active</Pill>
              <div style={{fontSize:10,color:"#1e3a5f",marginTop:4}}>Using live data from {location.name.split(",")[0]}</div>
            </div>
          </div>
          <div style={{display:"flex",gap:6,flexWrap:"wrap",marginBottom:12}}>
            {ag.prompts.map(p=>(
              <button key={p} onClick={()=>setQ(p)} style={{background:"#060e1a",border:`1px solid ${ag.color}33`,borderRadius:20,padding:"5px 11px",fontSize:11,color:"#64748b",cursor:"pointer"}}
                onMouseEnter={e=>{e.currentTarget.style.color=ag.color;e.currentTarget.style.borderColor=ag.color;}}
                onMouseLeave={e=>{e.currentTarget.style.color="#64748b";e.currentTarget.style.borderColor=ag.color+"33";}}>
                {p}
              </button>
            ))}
          </div>
          <div style={{display:"flex",gap:8,marginBottom:12}}>
            <input value={q} onChange={e=>setQ(e.target.value)} onKeyDown={e=>e.key==="Enter"&&run()}
              placeholder={`Ask ${ag.name} about ${location.name.split(",")[0]}…`}
              style={{flex:1,background:"#060e1a",border:"1px solid #1e3a5f",borderRadius:8,padding:"11px 14px",color:"#e2e8f0",fontSize:13,outline:"none"}}/>
            <button onClick={run} disabled={loading||!q.trim()} style={{background:ag.color,color:"#fff",border:"none",borderRadius:8,padding:"11px 20px",fontSize:13,fontWeight:700,cursor:"pointer",opacity:loading?.6:1}}>
              {loading?"…":"▶ Run"}
            </button>
          </div>
          {loading&&<Loader text={`${ag.name} analyzing live data for ${location.name.split(",")[0]}…`}/>}
          {resp&&<div className="fade" style={{background:"#060e1a",border:`1px solid ${ag.color}22`,borderRadius:10,padding:16,fontSize:13,color:"#cbd5e1",lineHeight:1.8,whiteSpace:"pre-wrap",maxHeight:380,overflowY:"auto"}}>{resp}</div>}
          {hist.length>0&&<button onClick={()=>{setHist([]);setResp("");}} style={{marginTop:8,background:"transparent",border:"1px solid #1e293b",borderRadius:6,padding:"5px 12px",color:"#334155",fontSize:11,cursor:"pointer"}}>🗑️ Clear ({Math.floor(hist.length/2)} turns)</button>}
        </Card>
      ):(
        <Card style={{textAlign:"center",padding:"40px 20px",color:"#1e3a5f"}}>
          <div style={{fontSize:44,marginBottom:10}}>🤖</div>
          <div style={{fontSize:15,fontWeight:600,marginBottom:6}}>Select an AI Agent</div>
          <div style={{fontSize:13}}>5 specialized environmental AI agents — all powered by live local data</div>
        </Card>
      )}
    </div>
  );
}

// ── Auth ──────────────────────────────────────────────────────────────────────
function Auth({onLogin}){
  const [mode,setMode]=useState("login");
  const [f,setF]=useState({name:"",email:"",pass:"",role:"user"});
  const [err,setErr]=useState("");
  const DEMO={admin:{email:"admin@eco.ai",pass:"admin123",name:"Admin User",role:"admin"},user:{email:"demo@eco.ai",pass:"demo123",name:"Demo User",role:"user"}};

  function submit(e){
    e.preventDefault();setErr("");
    if(mode==="login"){
      const m=Object.values(DEMO).find(u=>u.email===f.email&&u.pass===f.pass);
      if(m) onLogin(m); else setErr("Invalid. Try demo@eco.ai / demo123");
    } else {
      if(!f.name||!f.email||!f.pass){setErr("All fields required");return;}
      if(f.pass.length<6){setErr("Password min 6 chars");return;}
      onLogin({name:f.name,email:f.email,role:f.role});
    }
  }

  return(
    <div style={{minHeight:"100vh",background:"#020c1b",display:"flex",alignItems:"center",justifyContent:"center",padding:20}}>
      <div style={{width:"100%",maxWidth:420}}>
        <div style={{textAlign:"center",marginBottom:30}}>
          <div style={{fontSize:52,marginBottom:8,filter:"drop-shadow(0 0 20px #10b98166)"}}>🌍</div>
          <div style={{fontSize:26,fontWeight:900,background:"linear-gradient(90deg,#10b981,#0891b2)",WebkitBackgroundClip:"text",WebkitTextFillColor:"transparent",letterSpacing:-1}}>EcoGuardian AI</div>
          <div style={{fontSize:10,color:"#334155",letterSpacing:3,fontWeight:600,marginTop:3}}>SMART ENVIRONMENTAL INTELLIGENCE</div>
        </div>
        <Card color="#1e3a5f">
          <div style={{display:"flex",background:"#060e1a",borderRadius:9,padding:4,marginBottom:20,gap:4}}>
            {["login","signup"].map(m=>(
              <button key={m} onClick={()=>{setMode(m);setErr("");}} style={{flex:1,padding:"9px",borderRadius:6,border:"none",background:mode===m?"#1e3a5f":"transparent",color:mode===m?"#93c5fd":"#334155",fontSize:13,fontWeight:700,cursor:"pointer"}}>
                {m==="login"?"🔑 Sign In":"✨ Sign Up"}
              </button>
            ))}
          </div>
          <form onSubmit={submit} style={{display:"flex",flexDirection:"column",gap:13}}>
            {mode==="signup"&&<input value={f.name} onChange={e=>setF({...f,name:e.target.value})} placeholder="Full Name" style={{background:"#060e1a",border:"1px solid #1e3a5f",borderRadius:8,padding:"11px 14px",color:"#e2e8f0",fontSize:13,outline:"none",width:"100%"}}/>}
            <input value={f.email} onChange={e=>setF({...f,email:e.target.value})} type="email" placeholder="Email address" style={{background:"#060e1a",border:"1px solid #1e3a5f",borderRadius:8,padding:"11px 14px",color:"#e2e8f0",fontSize:13,outline:"none",width:"100%"}}/>
            <input value={f.pass} onChange={e=>setF({...f,pass:e.target.value})} type="password" placeholder="Password" style={{background:"#060e1a",border:"1px solid #1e3a5f",borderRadius:8,padding:"11px 14px",color:"#e2e8f0",fontSize:13,outline:"none",width:"100%"}}/>
            {mode==="signup"&&(
              <select value={f.role} onChange={e=>setF({...f,role:e.target.value})} style={{background:"#060e1a",border:"1px solid #1e3a5f",borderRadius:8,padding:"11px 14px",color:"#e2e8f0",fontSize:13,outline:"none",width:"100%"}}>
                <option value="user">👤 User — Researcher / Citizen</option>
                <option value="admin">👑 Admin — Government / Organization</option>
              </select>
            )}
            {err&&<div style={{color:"#fca5a5",fontSize:12,padding:"8px 12px",background:"#ef444411",borderRadius:7}}>{err}</div>}
            <button type="submit" style={{background:"linear-gradient(90deg,#059669,#0891b2)",color:"#fff",border:"none",borderRadius:9,padding:"13px",fontSize:14,fontWeight:800,cursor:"pointer"}}>
              {mode==="login"?"🚀 Enter Platform →":"✨ Create Account →"}
            </button>
          </form>
          {mode==="login"&&(
            <div style={{marginTop:16,padding:13,background:"#060e1a",borderRadius:9}}>
              <div style={{fontSize:10,color:"#334155",marginBottom:7,fontWeight:700,letterSpacing:.5}}>DEMO ACCOUNTS (click to fill)</div>
              {[["👤","demo@eco.ai","demo123","#10b981"],["👑","admin@eco.ai","admin123","#a78bfa"]].map(([icon,email,pass,c])=>(
                <div key={email} onClick={()=>setF({...f,email,pass})} style={{display:"flex",gap:8,alignItems:"center",cursor:"pointer",padding:"5px 0"}}>
                  <span>{icon}</span>
                  <span style={{fontSize:12}}><strong style={{color:c}}>{email}</strong><span style={{color:"#475569"}}> / </span><strong style={{color:c}}>{pass}</strong></span>
                </div>
              ))}
            </div>
          )}
        </Card>
      </div>
    </div>
  );
}

// ── Main App ──────────────────────────────────────────────────────────────────
function App(){
  const [user,setUser]=useState(null);
  const [location,setLocation]=useState(null);
  const [envData,setEnvData]=useState(null);
  const [page,setPage]=useState("dashboard");
  const [refreshing,setRefreshing]=useState(false);

  async function refresh(){
    if(!location) return;
    setRefreshing(true);
    const [weather,aqData]=await Promise.all([fetchWeather(location.lat,location.lng),fetchOpenAQ(location.lat,location.lng,location.name)]);
    setEnvData({weather,aqData,location,loadedAt:new Date()});
    setRefreshing(false);
  }

  const NAV=[
    {id:"dashboard",icon:"🌍",label:"Dashboard"},
    {id:"chat",icon:"💬",label:"AI Assistant"},
    {id:"agents",icon:"🤖",label:"AI Agents"},
    {id:"data",icon:"📊",label:"Data Upload"},
    {id:"reports",icon:"📄",label:"Reports"},
  ];

  if(!user) return <Auth onLogin={u=>{setUser(u);}}/>;
  if(!location) return <LocationPicker onLocationSet={loc=>setLocation(loc)}/>;
  if(!envData) return <DataLoader location={location} onDataLoaded={d=>{setEnvData(d);setPage("dashboard");}}/>;

  const hasReal=envData.aqData.success&&envData.aqData.stations?.length>0;
  const avgAQI=hasReal?Math.round(envData.aqData.stations.reduce((a,s)=>a+estimateAQI(s.params),0)/envData.aqData.stations.length):85;

  return(
    <div style={{height:"100vh",display:"flex",overflow:"hidden"}}>
      {/* Sidebar */}
      <div style={{width:220,background:"#060e1a",borderRight:"1px solid #0d1b2e",padding:"18px 10px",display:"flex",flexDirection:"column"}}>
        <div style={{paddingLeft:6,marginBottom:22}}>
          <div style={{fontSize:15,fontWeight:900,background:"linear-gradient(90deg,#10b981,#0891b2)",WebkitBackgroundClip:"text",WebkitTextFillColor:"transparent"}}>EcoGuardian AI</div>
          <div style={{fontSize:8,color:"#1e3a5f",letterSpacing:1.8,marginTop:1,fontWeight:600}}>SMART ENVIRONMENTAL INTELLIGENCE</div>
        </div>
        {/* Current location mini card */}
        <div style={{background:"#0d1b2e",border:"1px solid #1e3a5f",borderRadius:10,padding:"10px 12px",marginBottom:16}}>
          <div style={{fontSize:10,color:"#334155",marginBottom:4,fontWeight:600}}>📍 MONITORING</div>
          <div style={{fontSize:12,fontWeight:600,color:"#e2e8f0",marginBottom:4,lineHeight:1.3}}>{location.name}</div>
          <div style={{display:"flex",justifyContent:"space-between",alignItems:"center"}}>
            <span style={{fontSize:20,fontWeight:800,color:aqiColor(avgAQI),fontFamily:"monospace"}}>{avgAQI}</span>
            <Pill color={aqiColor(avgAQI)}>{aqiLabel(avgAQI).split(" ")[0]}</Pill>
          </div>
          <div style={{display:"flex",gap:4,marginTop:6,flexWrap:"wrap"}}>
            <Pill color={hasReal?"#10b981":"#fbbf24"}>{hasReal?"● Live":"● Est."}</Pill>
            {envData.weather.success&&<Pill color="#60a5fa">🌡️{envData.weather.temp}°C</Pill>}
          </div>
        </div>
        <div style={{display:"flex",flexDirection:"column",gap:3,flex:1}}>
          {NAV.map(n=><NavBtn key={n.id} icon={n.icon} label={n.label} active={page===n.id} onClick={()=>setPage(n.id)}/>)}
        </div>
        <div style={{borderTop:"1px solid #0d1b2e",paddingTop:10,marginTop:8}}>
          <div style={{display:"flex",alignItems:"center",gap:8,padding:"7px 8px",marginBottom:5}}>
            <div style={{width:30,height:30,borderRadius:"50%",background:"#10b98122",border:"1px solid #10b98144",display:"flex",alignItems:"center",justifyContent:"center",fontSize:14}}>{user.role==="admin"?"👑":"👤"}</div>
            <div>
              <div style={{fontSize:12,fontWeight:600,color:"#e2e8f0"}}>{user.name}</div>
              <div style={{fontSize:10,color:"#334155",textTransform:"capitalize"}}>{user.role}</div>
            </div>
          </div>
          <button onClick={()=>setLocation(null)} style={{width:"100%",background:"transparent",border:"1px solid #1e293b",borderRadius:7,padding:"6px",color:"#334155",fontSize:11,cursor:"pointer",marginBottom:4}}
            onMouseEnter={e=>e.currentTarget.style.color="#10b981"} onMouseLeave={e=>e.currentTarget.style.color="#334155"}>
            📍 Change Location
          </button>
          <button onClick={()=>{setUser(null);setLocation(null);setEnvData(null);}} style={{width:"100%",background:"transparent",border:"1px solid #1e293b",borderRadius:7,padding:"6px",color:"#334155",fontSize:11,cursor:"pointer"}}
            onMouseEnter={e=>e.currentTarget.style.color="#ef4444"} onMouseLeave={e=>e.currentTarget.style.color="#334155"}>
            ← Sign Out
          </button>
        </div>
      </div>

      {/* Main */}
      <div style={{flex:1,display:"flex",flexDirection:"column",overflow:"hidden"}}>
        {/* Topbar */}
        <div style={{padding:"12px 22px",borderBottom:"1px solid #0d1b2e",display:"flex",justifyContent:"space-between",alignItems:"center",background:"#060e1a",flexShrink:0}}>
          <div>
            <div style={{fontSize:15,fontWeight:700,color:"#e2e8f0"}}>{NAV.find(n=>n.id===page)?.icon} {NAV.find(n=>n.id===page)?.label}</div>
            <div style={{fontSize:11,color:"#334155"}}>{location.name} · {new Date().toLocaleString()}</div>
          </div>
          <div style={{display:"flex",gap:8,alignItems:"center",flexWrap:"wrap"}}>
            <Pill color={aqiColor(avgAQI)}>AQI {avgAQI} — {aqiLabel(avgAQI)}</Pill>
            <Pill color={hasReal?"#10b981":"#fbbf24"}>{hasReal?"📡 Live Sensors":"⚡ Estimated"}</Pill>
            {GROQ_KEY==="YOUR_GROQ_API_KEY_HERE"&&<Pill color="#fbbf24">⚠️ Add Groq Key</Pill>}
            {GROQ_KEY!=="YOUR_GROQ_API_KEY_HERE"&&<Pill color="#22c55e">✓ AI Ready</Pill>}
          </div>
        </div>
        {/* Page */}
        <div style={{flex:1,overflowY:"auto",padding:20}}>
          {page==="dashboard"&&<Dashboard envData={envData} location={location} onRefresh={refresh} refreshing={refreshing}/>}
          {page==="chat"&&<Chatbot envData={envData} location={location}/>}
          {page==="agents"&&<AgentsTab envData={envData} location={location}/>}
          {page==="data"&&<DataTab location={location}/>}
          {page==="reports"&&<Reports envData={envData} location={location}/>}
        </div>
      </div>
    </div>
  );
}

ReactDOM.createRoot(document.getElementById("root")).render(<App/>);
</script>
</body>
</html>
