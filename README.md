<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>まいにちパタカラ（最新音つき版）</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; background-color: #f0f8ff; padding: 20px; color: #333; }
        .container { background-color: white; border-radius: 15px; padding: 20px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); max-width: 400px; margin: 0 auto; }
        h1 { color: #0076ff; font-size: 24px; }
        button { background-color: #0076ff; color: white; border: none; padding: 15px 3px; width: 100%; border-radius: 10px; font-size: 20px; font-weight: bold; margin-top: 20px; cursor: pointer; }
        button:disabled { background-color: #ccc; }
        .countdown { font-size: 48px; font-weight: bold; color: #ff9900; margin: 20px 0; min-height: 60px; }
        .result { font-size: 22px; font-weight: bold; margin-top: 20px; padding: 10px; border-radius: 5px; }
        .success { background-color: #ddffdd; color: #008800; }
        .warning { background-color: #ffdddd; color: #cc0000; }
        .promo-box { margin-top: 30px; border: 2px dashed #ff9900; background-color: #fffaf0; padding: 15px; border-radius: 10px; font-size: 14px; text-align: left; }
        .promo-title { font-weight: bold; color: #ff9900; font-size: 16px; text-align: center; margin-bottom: 5px; }
    </style>
</head>
<body>

<div class="container">
    <h1>まいにちパタカラ</h1>
    <p>【最新WEBお試し版・音つき】</p>
    <p style="font-size:14px; color:#666;">「スタート」を押すと音に合わせてカウントダウンが始まります！</p>
    
    <div id="countdownDisplay" class="countdown">⏱️</div>
    
    <button id="startBtn">計測スタート！</button>
    
    <div id="resultOutput"></div>

    <div class="promo-box">
        <div class="promo-title">✨ 完全版アプリならもっと凄い！ ✨</div>
        <p>近日公開のスマホアプリ製品版（iPhone/Android）では、以下のフル機能が使えます！</p>
        <ul style="padding-left: 20px; margin: 5px 0;">
            <li><b>「パ・タ・カ・ラ」すべての音の自動計測</b></li>
            <li>カレンダーへの毎日の自動履歴保存</li>
            <li>お口を鍛える「パタカラリズムゲーム」搭載</li>
        </ul>
        <p style="text-align:center; margin-bottom:0; font-weight:bold; color:#ff9900;">お試し版の結果画面から先行予約受付中！</p>
    </div>
</div>

<script>
let audioContext;
let analyser;
let microphone;

const startBtn = document.getElementById('startBtn');
const countdownDisplay = document.getElementById('countdownDisplay');
const resultOutput = document.getElementById('resultOutput');

function playTone(freq, type, duration) {
    if (!audioContext) return;
    const osc = audioContext.createOscillator();
    const gain = audioContext.createGain();
    osc.type = type;
    osc.frequency.value = freq;
    gain.gain.setValueAtTime(0.1, audioContext.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + duration);
    osc.connect(gain);
    gain.connect(audioContext.destination);
    osc.start();
    osc.stop(audioContext.currentTime + duration);
}

startBtn.addEventListener('click', async () => {
    resultOutput.innerHTML = "";
    startBtn.disabled = true;
    
    try {
        // iPhone対策：タップした瞬間にオーディオContextを新しく生成
        audioContext = new (window.AudioContext || window.webkitAudioContext)();
        
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        analyser = audioContext.createAnalyser();
        microphone = audioContext.createMediaStreamSource(stream);
        
        analyser.fftSize = 256;
        microphone.connect(analyser);
        
        // カウントダウン (3 -> 2 -> 1)
        let countNum = 3;
        countdownDisplay.style.color = "#ff9900";
        countdownDisplay.innerText = countNum;
        playTone(600, 'sine', 0.1); 

        const interval = setInterval(() => {
            countNum--;
            if (countNum > 0) {
                countdownDisplay.innerText = countNum;
                playTone(600, 'sine', 0.1); 
            } else {
                clearInterval(interval);
                startMeasurement(stream);
            }
        }, 1000);
        
    } catch (err) {
        alert("マイクの許可が必要です。iPhoneの設定やブラウザの許可を確認してください。");
        startBtn.disabled = false;
        countdownDisplay.innerText = "⏱️";
    }
});

function startMeasurement(stream) {
    countdownDisplay.style.color = "red";
    countdownDisplay.innerText = "スタート！";
    playTone(1000, 'sine', 0.3); 
    
    let count = 0;
    let isSpeaking = false;
    const bufferLength = analyser.frequencyBinCount;
    const dataArray = new Uint8Array(bufferLength);
    
    const checkAudio = setInterval(() => {
        analyser.getByteFrequencyData(dataArray);
        let sum = 0;
        for(let i=0; i<bufferLength; i++) { sum += dataArray[i]; }
        let average = sum / bufferLength;
        
        // 感度を「10」まで下げて、どんな小さな音でも拾うように超高感度化！
        if (average > 10) { 
            if (!isSpeaking) {
                count++;
                isSpeaking = true;
            }
        } else {
            isSpeaking = false;
        }
    }, 40);
    
    setTimeout(() => {
        clearInterval(checkAudio);
        stream.getTracks().forEach(track => track.stop());
        
        countdownDisplay.style.color = "#333";
        countdownDisplay.innerText = "終了！";
        playTone(300, 'sawtooth', 0.2); 
        
        startBtn.disabled = false;
        showResult(count);
    }, 1000);
}

function showResult(score) {
    let html = "";
    if (score >= 6) {
        html = `<div class="result success">結果：${score} 回 / 秒<br>🎉 すばらしい！健康です！</div>`;
    } else {
        html = `<div class="result warning">結果：${score} 回 / 秒<br>⚠️ 要注意（お口の老化のサイン）</div>`;
    }
    
    html += `
        <div style="margin-top:20px; padding:15px; background:#f9f9f9; border-radius:10px; border:1px solid #ddd;">
            <p style="font-size:15px; font-weight:bold; margin-top:0;">💡 あなたへのおすすめアドバイス</p>
            <p style="font-size:13px; color:#555; text-align:left; line-height:1.4;">
                ${score >= 6 ? '今のお口の若さをキープするために、毎日パタカラ体操を続けましょう！' : '「パ」の回数が少し少なめです。お口の筋肉が弱っているかもしれません。毎日「パタカラ」と発音するトレーニングで回復できます！'}
            </p>
            <button onclick="window.open('https://note.com/', '_blank')" style="background-color:#ff9900; margin-top:10px;">📖 お口の若さを保つトレーニング方法（note記事）を見る</button>
        </div>
    `;
    resultOutput.innerHTML = html;
}
</script>

</body>
</html>

