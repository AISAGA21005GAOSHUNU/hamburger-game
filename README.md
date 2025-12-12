const el = (id) => document.getElementById(id);

const menuBtn = el("menuBtn");
const drawer = el("drawer");
const closeDrawer = el("closeDrawer");
const overlay = el("overlay");

const promptInput = el("promptInput");
const currentPrompt = el("currentPrompt");
const answerText = el("answerText");

const generateBtn = el("generateBtn");
const randomBtn = el("randomBtn");
const fakeBtn = el("fakeBtn");

const startRoundBtn = el("startRoundBtn");
const revealNowBtn = el("revealNowBtn");
const secondsInput = el("secondsInput");
const revealMode = el("revealMode");

const endpointInput = el("endpointInput");
const copyPromptBtn = el("copyPromptBtn");

const artImg = el("artImg");
const curtain = el("curtain");
const loading = el("loading");

const timerText = el("timerText");
const roundText = el("roundText");

const toggleAnswerBtn = el("toggleAnswerBtn");
const answerBox = el("answerBox");

const toast = el("toast");

let prompts = [];
let round = 0;
let timer = null;
let remaining = 0;
let isAnswerVisible = true;

// --- UI helpers
function openDrawer(){
  drawer.classList.add("open");
  drawer.setAttribute("aria-hidden","false");
  overlay.hidden = false;
}
function closeDrawerFn(){
  drawer.classList.remove("open");
  drawer.setAttribute("aria-hidden","true");
  overlay.hidden = true;
}
function showToast(msg){
  toast.textContent = msg;
  toast.hidden = false;
  setTimeout(()=> toast.hidden = true, 1600);
}

menuBtn.onclick = openDrawer;
closeDrawer.onclick = closeDrawerFn;
overlay.onclick = closeDrawerFn;

// --- Load prompts
fetch("./prompts.json")
  .then(r => r.json())
  .then(j => { prompts = j.prompts || []; })
  .catch(()=> { prompts = ["牛","豬","貓","狗"]; });

// --- Core actions
function setPrompt(p){
  const s = (p || "").trim();
  promptInput.value = s;
  currentPrompt.textContent = s || "（尚未設定）";
  answerText.textContent = s || "（題目）";
}

randomBtn.onclick = () => {
  if (!prompts.length) return setPrompt("牛");
  const p = prompts[Math.floor(Math.random()*prompts.length)];
  setPrompt(p);
  showToast("🎲 隨機題已就位");
};

copyPromptBtn.onclick = async () => {
  const p = promptInput.value.trim();
  if (!p) return showToast("先輸入題目");
  try{
    await navigator.clipboard.writeText(p);
    showToast("📋 題目已複製");
  }catch{
    showToast("你的瀏覽器不讓複製（也沒關係）");
  }
};

toggleAnswerBtn.onclick = () => {
  isAnswerVisible = !isAnswerVisible;
  answerBox.style.display = isAnswerVisible ? "block" : "none";
};

function stopTimer(){
  if (timer) clearInterval(timer);
  timer = null;
}
function setCurtainHidden(hidden){
  if (hidden) curtain.classList.add("hidden");
  else curtain.classList.remove("hidden");
}

// Pixel reveal by CSS filter
function setPixelLevel(level){
  // level: 0(clear) .. 1(very pixel)
  const px = Math.round(2 + level * 22);
  const blur = (level * 2).toFixed(2);
  artImg.style.filter = `blur(${blur}px) contrast(1.05)`;
  artImg.style.transform = `scale(1.0)`;
  artImg.style.imageRendering = "pixelated";
}

// Reveal logic per tick
function applyReveal(progress){
  // progress: 0..1 (0 = hidden, 1 = fully revealed)
  const mode = revealMode.value;
  if (mode === "curtain"){
    setCurtainHidden(false);
    // curtain moves up as progress increases
    const y = (1 - progress) * 100;
    curtain.style.transform = `translateY(${Math.max(0, y)}%)`;
  } else {
    // pixel mode: keep curtain hidden
    curtain.style.transform = `translateY(-100%)`;
    const level = 1 - progress;
    setPixelLevel(level);
  }
}

function fullyReveal(){
  stopTimer();
  timerText.textContent = "已揭曉";
  curtain.style.transform = `translateY(-100%)`;
  setCurtainHidden(true);
  artImg.style.filter = "none";
  artImg.style.imageRendering = "auto";
  showToast("👀 揭曉！");
}

revealNowBtn.onclick = fullyReveal;

startRoundBtn.onclick = () => {
  const p = promptInput.value.trim();
  if (!p) return showToast("先輸入題目或按隨機題");
  if (!artImg.src) return showToast("先生成圖片（或用假圖）");

  round += 1;
  roundText.textContent = String(round);

  stopTimer();
  remaining = Math.max(5, Math.min(180, Number(secondsInput.value || 30)));
  timerText.textContent = `${remaining}s`;

  // start hidden
  if (revealMode.value === "curtain"){
    setCurtainHidden(false);
    curtain.style.transform = `translateY(0%)`;
  } else {
    setCurtainHidden(true);
    artImg.style.imageRendering = "pixelated";
    setPixelLevel(1);
  }

  timer = setInterval(() => {
    remaining -= 1;
    timerText.textContent = `${remaining}s`;

    const total = Math.max(1, Number(secondsInput.value || 30));
    const progress = 1 - (remaining / total); // 0..1
    applyReveal(progress);

    if (remaining <= 0){
      fullyReveal();
    }
  }, 1000);

  showToast("⏱️ 回合開始");
};

// --- Image generation (proxy endpoint)
async function generateWithEndpoint(){
  const p = promptInput.value.trim();
  if (!p) return showToast("請輸入題目");
  setPrompt(p);

  const endpoint = endpointInput.value.trim();
  if (!endpoint) return showToast("先填 AI 端點（或用假圖）");

  loading.hidden = false;
  try{
    // Expect JSON: { image_url: "https://..." } or { image_base64: "..." }
    const res = await fetch(endpoint, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ prompt: p })
    });

    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const data = await res.json();

    if (data.image_url){
      artImg.src = data.image_url;
    } else if (data.image_base64){
      artImg.src = `data:image/png;base64,${data.image_base64}`;
    } else {
      throw new Error("No image_url/image_base64 in response");
    }

    // reset reveal state
    setCurtainHidden(true);
    curtain.style.transform = `translateY(0%)`;
    artImg.style.filter = "none";
    artImg.style.imageRendering = "auto";
    timerText.textContent = "未開始";
    showToast("🍳 圖片生成完成");
  } catch (e){
    console.error(e);
    showToast("生成失敗：請檢查端點或改用假圖");
  } finally {
    loading.hidden = true;
  }
}
generateBtn.onclick = generateWithEndpoint;

// --- Fake image mode (no AI)
fakeBtn.onclick = () => {
  const p = promptInput.value.trim() || "牛";
  setPrompt(p);
  // Cute placeholder: dynamic SVG with burger theme + prompt
  const svg = `
  <svg xmlns="http://www.w3.org/2000/svg" width="1200" height="800">
    <defs>
      <linearGradient id="g" x1="0" y1="0" x2="1" y2="1">
        <stop offset="0" stop-color="#fff2d6"/>
        <stop offset="1" stop-color="#ffd0a1"/>
      </linearGradient>
      <pattern id="p" width="120" height="120" patternUnits="userSpaceOnUse">
        <text x="10" y="70" font-size="48" opacity="0.18">🍔</text>
        <text x="60" y="40" font-size="38" opacity="0.14">🍟</text>
      </pattern>
    </defs>
    <rect width="100%" height="100%" fill="url(#g)"/>
    <rect width="100%" height="100%" fill="url(#p)" />
    <rect x="80" y="90" width="1040" height="620" rx="40" fill="rgba(255,255,255,0.80)" stroke="rgba(0,0,0,0.12)" />
    <text x="120" y="210" font-size="56" font-family="system-ui" font-weight="800">AI 產生圖（假圖模式）</text>
    <text x="120" y="310" font-size="44" font-family="system-ui" font-weight="700">題目：${escapeXml(p)}</text>
    <text x="120" y="420" font-size="28" font-family="system-ui" opacity="0.75">上課先玩流程完全沒問題，之後再接 AI 端點就能出真圖。</text>
    <text x="120" y="560" font-size="120">🍔</text>
    <text x="280" y="560" font-size="120">🎨</text>
    <text x="440" y="560" font-size="120">🧠</text>
  </svg>`;
  artImg.src = "data:image/svg+xml;charset=utf-8," + encodeURIComponent(svg);

  setCurtainHidden(true);
  curtain.style.transform = `translateY(0%)`;
  timerText.textContent = "未開始";
  showToast("🧃 假圖就緒（免 AI）");
};

function escapeXml(s){
  return s.replace(/[<>&'"]/g, (c) => ({
    "<":"&lt;", ">":"&gt;", "&":"&amp;", "'":"&apos;", "\"":"&quot;"
  }[c]));
}
