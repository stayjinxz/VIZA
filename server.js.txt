const express = require('express');
const path = require('path');
const fetch = globalThis.fetch;
const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());
app.use(express.static(__dirname));

// ORIGINAL Visa X-PAY key & endpoints
const XPAY_KEY = "WVJVB50233CC192ONFXW21m84NYaktglFj2gbYvcBR-ZCHl8Y";
const ENDPOINTS = [
  "https://api.visa.com",
  "https://secure.checkout.visa.com",
  "https://b2b.api.visa.com",
  "https://api.visa.com/fallback"
];

// Helper POST function
async function attemptPost(url,payload,headers){
  try{
    const resp = await fetch(url,{
      method:'POST',
      headers: Object.assign({'Content-Type':'application/json','x-pay-token':XPAY_KEY}, headers||{}),
      body: JSON.stringify(payload)
    });
    const text = await resp.text();
    let body = null;
    try{body=JSON.parse(text);}catch(e){body={raw:text};}
    return {ok:resp.ok,status:resp.status,body,url};
  }catch(err){return {ok:false,error:err.toString(),url};}
}

async function sendWithFallbacks(payload){
  let lastFailure = null;
  for(const ep of ENDPOINTS){
    const fullUrl = ep.replace(/\/$/,'') + '/';
    const attempt = await attemptPost(fullUrl,payload);
    if(attempt.ok) return {mode:'no-client-auth',result:attempt};
    lastFailure = attempt;
  }
  return {mode:'failed',result:lastFailure || {ok:false,error:'No attempt made'}};
}

app.post('/process-payment',async(req,res)=>{
  const payload=req.body||{};
  console.log('[PROCESS] payload preview:',{
    amount:payload.amount,
    recipient:!!payload.recipientPrimaryAccountNumber,
    sender:!!payload.senderAccountNumber
  });
  try{
    if(payload.method && payload.method.toLowerCase()==='square'){
      return res.json({mode:'square-fallback',approved:true,message:'Square fallback (enable real sandbox token if desired)'});}
    const outcome=await sendWithFallbacks(payload);
    if(outcome.mode==='failed'){res.status(502).json({error:'All endpoints failed',detail:outcome.result});}
    else{res.status(outcome.result.status||200).json({mode:outcome.mode,url:outcome.result.url,status:outcome.result.status,body:outcome.result.body});}
  }catch(e){console.error('Server error',e);res.status(500).json({error:e.toString()});}
});

app.get('/',(req,res)=>res.sendFile(path.join(__dirname,'index.html')));

app.listen(PORT,()=>console.log('Visa Live tester running at http://localhost:'+PORT));