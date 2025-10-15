<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1"/>
<title>Arb Radar v2 — Kalshi × Polymarket (Live)</title>
<style>
  body {font-family: system-ui, sans-serif; background:#0b0f14; color:#e8eef7; margin:0;}
  header {background:#0e141b; padding:14px; border-bottom:1px solid #1e2a3a;}
  h1 {margin:0; font-size:18px;}
  .container {padding:16px;}
  .card {background:#0e141b; padding:12px; border:1px solid #1e2a3a; border-radius:10px; margin-bottom:10px;}
  .mono {font-family: monospace;}
  .good {color:#72f2b0;} .bad {color:#ff6b6b;}
</style>
</head>
<body>
<header>
  <h1>Arb Radar v2 — Kalshi × Polymarket (Live)</h1>
  <p>Automatically fetching live odds. Refreshes every 10 seconds.</p>
</header>
<div class="container">
  <div id="data">Loading…</div>
</div>

<script>
async function fetchPolymarket(){
  try{
    const r=await fetch("https://gamma-api.polymarket.com/markets?limit=200");
    return await r.json();
  }catch(e){return [];}
}
async function fetchKalshi(){
  const proxy="https://api.allorigins.win/raw?url=";
  try{
    const r=await fetch(proxy+encodeURIComponent("https://trading-api.kalshi.com/v2/markets?status=active&limit=200"));
    const d=await r.json();
    return d.markets||[];
  }catch(e){return [];}
}
function key(a,b){return[a,b].map(x=>x.toLowerCase()).sort().join(" vs ");}
function parse(d,src){
  return d.map(m=>{
    const t=(m.title||m.question||"").toLowerCase();
    let a=null,b=null;
    if(t.includes(" vs ")) [a,b]=t.split(" vs ");
    else if(t.includes(" @ ")) [a,b]=t.split(" @ ");
    const yes=src==="poly"
      ? Number(m.bestBid||m.price||0)/100
      : Number((m.order_book?.best_bid||m.last_price||0))/100;
    return {src,a,b,yes};
  }).filter(x=>x.a&&x.b&&x.yes>0&&x.yes<1);
}
function findArbs(poly,kalshi){
  const idx=new Map();
  kalshi.forEach(k=>{
    const ky=key(k.a,k.b);
    if(!idx.has(ky)) idx.set(ky,[]);
    idx.get(ky).push(k);
  });
  const res=[];
  poly.forEach(p=>{
    const mates=idx.get(key(p.a,p.b))||[];
    mates.forEach(k=>{
      const sum=p.yes+k.yes;
      const edge=1-sum;
      if(edge>0.02) res.push({pair:key(p.a,p.b),sum,edge});
    });
  });
  return res.sort((a,b)=>b.edge-a.edge);
}
async function update(){
  document.getElementById("data").innerHTML="Fetching…";
  const [p,k]=await Promise.all([fetchPolymarket(),fetchKalshi()]);
  const P=parse(p,"poly"),K=parse(k,"kalshi");
  const arbs=findArbs(P,K);
  const root=document.getElementById("data");
  root.innerHTML="";
  if(!arbs.length){root.innerHTML="<p>No current arbitrage opportunities.</p>";return;}
  arbs.forEach(a=>{
    const div=document.createElement("div");
    div.className="card";
    div.innerHTML=`<strong>${a.pair}</strong><br>Sum: ${(a.sum*100).toFixed(1)}¢<br>
    <span class="${a.edge>0?"good":"bad"}">Edge: ${(a.edge*100).toFixed(2)}%</span>`;
    root.appendChild(div);
  });
}
update();
setInterval(update,10000);
</script>
</body>
</html>
