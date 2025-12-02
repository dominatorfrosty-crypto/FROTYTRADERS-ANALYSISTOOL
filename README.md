import React, { useEffect, useRef, useState } from "react";

// CONFIG
const APP_ID = '1xAfSI8X9TgMEix';
const WS_URL = `wss://ws.derivws.com/websockets/v3?app_id=${APP_ID}`;
const API_TOKEN = "<84799>";
const STRATEGIES = ["Over/Under", "Even/Odd", "Matches", "Differs"];
const VOLATILITY_MARKETS = [
  "R_10", "R_25", "R_50", "R_75", "R_100", 
  "R_150", "R_200", "R_100_10", "R_100_15", "R_100_30", "R_100_90"
];

const ANALYSIS_PERIOD = 15; // ticks (~15 seconds)

const getDigitCounts = (ticks) =>
  ticks.reduce((cnt, tick) => {
    const d = tick % 10;
    cnt[d] = (cnt[d] || 0) + 1;
    return cnt;
  }, Array(10).fill(0));

// --- STRATEGY SIGNAL LOGIC --- //
function analyzeTicks(strategy, ticks) {
  const sample = ticks.slice(-ANALYSIS_PERIOD);
  const counts = getDigitCounts(sample);

  // Over/Under logic
  if (strategy === "Over/Under") {
    // Over 2/3, Under 7/6 (pick most frequent, avoid loss)
    // Entry = digit, Recovery = next most probable
    const overPool = [3,2];
    const underPool = [6,7];
    const getBest = (pool) =>
      pool.reduce((best, d) => (counts[d] > counts[best] ? d : best), pool[0]);
    const overBest = getBest(overPool);
    const underBest = getBest(underPool);
    const entry = counts[overBest] >= counts[underBest] ? `Over ${overBest}` : `Under ${underBest}`;
    const recovery = counts[overBest] < counts[underBest] ? `Over ${overPool.find(d=>d!==overBest)}` : `Under ${underPool.find(d=>d!==underBest)}`;
    return {
      strategy,
      signal: entry,
      entryPoint: entry,
      recoveryDigit: recovery,
      comment: `Trade ${entry} with recovery on ${recovery}.`
    };
  }
  // Even/Odd logic
  if (strategy === "Even/Odd") {
    const evenDigits = [0,2,4,6,8], oddDigits = [1,3,5,7,9];
    const evenSum = evenDigits.map(d=>counts[d]).reduce((a,b)=>a+b,0);
    const oddSum = oddDigits.map(d=>counts[d]).reduce((a,b)=>a+b,0);
    const bestEven = evenDigits.reduce((best,d)=>counts[d]>counts[best]?d:best,evenDigits[0]);
    const bestOdd = oddDigits.reduce((best,d)=>counts[d]>counts[best]?d:best,oddDigits[0]);
    const bestType = evenSum > oddSum ? "Even" : "Odd";
    const entryDigit = bestType==="Even"?bestEven:bestOdd;
    const recoveryDigit = bestType==="Even"?evenDigits.find(d=>d!==entryDigit):oddDigits.find(d=>d!==entryDigit);

    return {
      strategy,
      signal: `${bestType} - Enter on digit ${entryDigit}`,
      entryPoint: entryDigit,
      recoveryDigit,
      comment: `Trade ${bestType} (digit: ${entryDigit}), recover with digit ${recoveryDigit} for smoother win.`
    };
  }
  // Matches logic
  if (strategy === "Matches") {
    // Find most repeated match digit, next frequent for recovery
    const bestMatch = counts.reduce((best, val, idx) => val > counts[best] ? idx : best, 0);
    const recoveryMatch =
      counts.map((val, idx) => ({val, idx}))
      .filter(obj => obj.idx !== bestMatch)
      .sort((a, b) => b.val - a.val)[0]?.idx ?? (bestMatch+1)%10;
    return {
      strategy,
      signal: `Match digit ${bestMatch}`,
      entryPoint: bestMatch,
      recoveryDigit: recoveryMatch,
      comment: `Match with digit ${bestMatch}, recovery with digit ${recoveryMatch}`
    };
  }
  // Differs logic
  if (strategy === "Differs") {
    // Choose digit with least frequency (hard to match, ideal for differs)
    const bestDiffer = counts.reduce((least, val, idx) => val < counts[least] ? idx : least, 0);
    const recoveryDiffer =
      counts.map((val,idx)=>({val,idx}))
      .filter(obj=>obj.idx!==bestDiffer)
      .sort((a,b)=>a.val-b.val)[0]?.idx ?? (bestDiffer+1)%10;
    return {
      strategy,
      signal: `Differs digit ${bestDiffer}`,
      entryPoint: bestDiffer,
      recoveryDigit: recoveryDiffer,
      comment: `Differs entry on digit ${bestDiffer}, recovery with digit ${recoveryDiffer}`
    };
  }
  return {
    strategy,
    signal: "Not enough data",
    entryPoint: null,
    recoveryDigit: null,
    comment: "Waiting for more tick data."
  };
}

export default function TradingAnalysisTool() {
  const ws = useRef();
  const [connected, setConnected] = useState(false);
  const [ticks, setTicks] = useState([]);
  const [strategy, setStrategy] = useState(STRATEGIES[0]);
  const [market, setMarket] = useState(VOLATILITY_MARKETS[0]);
  const [signalResult, setSignalResult] = useState(null);
  const [animating, setAnimating] = useState(false);

  // WebSocket connection and data
  useEffect(() => {
    const socket = new WebSocket(WS_URL);
    ws.current = socket;
    socket.onopen = () => {
      setConnected(true);
      socket.send(JSON.stringify({ authorize: API_TOKEN }));
      socket.send(JSON.stringify({
        "ticks_history": market, "count": 1000, "end": "latest"
      }));
      socket.send(JSON.stringify({ ticks: market }));
    };
    socket.onmessage = msg => {
      const data = JSON.parse(msg.data);
      if (data?.history?.prices) setTicks(data.history.prices);
      if (data?.tick?.quote) setTicks(prev => [...prev.slice(-999), data.tick.quote]);
    };
    socket.onclose = () => setConnected(false);
    return () => socket.close();
  }, [market]);

  function handleAnalyze() {
    setAnimating(true);
    setTimeout(() => {
      const result = analyzeTicks(strategy, ticks);
      setSignalResult(result);
    }, 1000); // small delay for "animation" effect
    setTimeout(() => {
      setAnimating(false);
      setSignalResult(null);
    }, 15_000); // Signal bar goes off after 15s (analysis period exact)
  }

  // Bar animation
  const percentages = ticks.length ? calcTickDistribution(ticks) : Array(10).fill("0.00");
  const cursorIndex = ticks.length ? (ticks[ticks.length-1] % 10) : 0;

  return (
    <div style={{ fontFamily: 'sans-serif', background: '#eee', minHeight: '100vh' }}>
      <h2>Deriv Real-Time Trading Signal Tool</h2>
      <div>
        <label>Strategy:</label>
        {STRATEGIES.map(s => (
          <button key={s} onClick={() => setStrategy(s)} style={{
            margin: 2, background: strategy===s ? "#bbf" : "#fff" }}>{s}
          </button>
        ))}
      </div>
      <div>
        <label>Market: </label>
        <select value={market} onChange={e=>setMarket(e.target.value)}>
          {VOLATILITY_MARKETS.map(m => <option value={m} key={m}>{m}</option>)}
        </select>
        <span style={{marginLeft:8,color:connected?"green":"red"}}>
          {connected ? "Connected" : "Offline"}
        </span>
      </div>
      <div style={{ margin: "15px 0" }}>
        <button onClick={handleAnalyze} style={{ padding: "8px 16px" }}>
          Analyze
        </button>
      </div>
      {/* Tick Distribution Animation */}
      <div style={{
        display:'flex',gap:10,margin:'10px 0',
        alignItems:'flex-end',height:120
      }}>
        {percentages.map((pct,idx)=>(
          <div key={idx}
            style={{
              position:'relative',width:40,
              height: Math.max(+pct * 2,15),
              background:'#444',borderRadius:8,
              marginTop:100-(+pct*2)
            }}>
            <div style={{
              position:'absolute',bottom:-18,left:'50%',
              width:34, color:'#222',
              fontWeight:'bold', textAlign:'center',
              background: cursorIndex===idx?'#f00':'#eee',
              borderRadius:6, transform:'translateX(-50%)'
            }}>{idx}</div>
            <div style={{
              color:'#fff',fontSize:13,textAlign:'center'
            }}>{pct}%</div>
            {cursorIndex===idx && (
              <div style={{
                position:'absolute',top:-10,left:'50%',
                transform:'translateX(-50%)',
                width:0,height:0,borderLeft:'14px solid transparent',
                borderRight:'14px solid transparent',
                borderBottom:'12px solid red'
              }}/>
            )}
          </div>
        ))}
      </div>
      {/* Animated Signal Bar */}
      {animating && signalResult && (
        <div style={{
          animation: "fadein 1s",
          background:"#cff", 
          border:"2px solid #319",
          borderRadius:12,
          margin:"15px 0",padding:"12px 24px",
          position:'relative',
          maxWidth:340
        }}>
          <strong>Strategy:</strong> {signalResult.strategy}<br/>
          <strong>Signal:</strong> {signalResult.signal}<br/>
          <strong>Entry Digit:</strong> {signalResult.entryPoint}<br/>
          <strong>Recovery Digit:</strong> {signalResult.recoveryDigit}<br/>
          <strong>Analysis:</strong> <span style={{color:"#006"}}>{signalResult.comment}</span>
          <div style={{
            height:6,
            background:"#66f",
            marginTop:8,
            width: "100%",
            borderRadius:4,
            transition:"width 1s",
            animation: "indicator 15s linear forwards"
          }} />
          <style>{`
            @keyframes indicator {
              0% { width: 100%; }
              100% { width: 0%; }
            }
            @keyframes fadein {
              from { opacity: 0; }
              to { opacity: 1; }
            }
          `}</style>
        </div>
      )}
    </div>
  );
}
