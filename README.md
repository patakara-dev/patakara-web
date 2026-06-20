<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>まいにちパタカラ（高感度版）</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; background-color: #f0f8ff; padding: 20px; color: #333; }
        .container { background-color: white; border-radius: 15px; padding: 20px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); max-width: 400px; margin: 0 auto; }
        h1 { color: #0076ff; font-size: 24px; margin-bottom: 5px; }
        .subtitle { font-size: 14px; color: #666; font-weight: bold; margin-bottom: 20px; }
        .guide-box { background-color: #e6f2ff; border-left: 5px solid #0076ff; padding: 12px; border-radius: 5px; text-align: left; font-size: 14px; line-height: 1.5; margin-bottom: 20px; }
        .target-text { font-weight: bold; color: #ff5500; }
        button { background-color: #0076ff; color: white; border: none; padding: 15px 3px; width: 100%; border-radius: 10px; font-size: 20px; font-weight: bold; margin-top: 10px; cursor: pointer; }
        button:disabled { background-color: #ccc; }
        .countdown { font-size: 44px; font-weight: bold; color: #ff9900; margin: 20px 0; min-height: 120px; display: flex; flex-direction: column; justify-content: center; align-items: center; background: #fffde6; border-radius: 10px; border: 1px solid #ffe699; }
        .action-msg { font-size: 24px; color: red; font-weight: bold; }
        .result { font-size: 22px; font-weight: bold; margin-top: 20px; padding: 10px; border-radius: 5px; }
        .success { background-color: #ddffdd; color: #008800; }
        .warning { background-color: #ffdddd; color: #cc0000; }
    </style>
</head>
<body>

<div class="container">
    <h1>まいにちパタカラ</h1>
    <div class="subtitle">【WEBお試し版・超高感度マイク版】</div>
    
    <div class="guide-box">
        <b>💡 計測のコツ：</b><br>
        「3・2 wilderness 1」のあと、画面が赤くなったら<br>
        スマホのマイクに向かって、少し<b>大きめの声でハッキリ</b>と<br>
        「パパパパパパパ！」と発音してください！<br>
        🎯 <b>目標：1秒間に <span class="target-text">6回以上</span></b>
    </div>
    
    <div id="countdownDisplay" class="countdown">
        <span style="font-size: 18px; color: #888;">ここに「3・2・1」が出ます</span>
    </div>
    
    <button id="startBtn">計測スタート！</button>
    
    <div id="resultOutput"></div>
</div>

<script>
let audioContext;
let analyser;
let microphone;

const startBtn = document.getElementById('startBtn');
const countdownDisplay = document.getElementById('countdownDisplay');
const resultOutput = document.getElementById('resultOutput');

startBtn.addEventListener('click', async () => {
    resultOutput.innerHTML = "";
    startBtn.disabled = true;
    
    try {
        // iPhoneのノイズキャンセリング等をできるだけ無効化して生音を拾う設定
        const stream = await navigator.mediaDevices.getUserMedia({ 
            audio: {
                echoCancellation: false,
                noiseSuppression: false,
                autoGainControl: false
            } 
        });
        
        audioContext = new (window.AudioContext || window.webkitAudioContext)();
        analyser = audioContext.createAnalyser();
        microphone = stream.getTracks()[0];
        
        const source = audioContext.createMediaStreamSource(stream);
        analyser.fftSize = 256;
        source.connect(analyser);
        
        let countNum = 3;
        countdownDisplay.innerHTML = `<span style="font-size:60px;">${countNum}</span>`;

        const interval = setInterval(() => {
            countNum--;
            if (countNum > 0) {
                countdownDisplay.innerHTML = `<span style="font-size:60px;">${countNum}</span>`;
            } else {
                clearInterval(interval);
                startMeasurement(stream);
            }
        }, 1000);
        
    } catch (err) {
        alert("マイクの許可が取れませんでした。iPhoneの設定を確認してください。");
        startBtn.disabled = false;
    }
});

function startMeasurement(stream) {
    countdownDisplay.innerHTML = `<span class="action-msg">パパパパパ！<br>と言って！</span>`;
    
    let count = 0;
    let isSpeaking = false;
    const bufferLength = analyser.frequencyBinCount;
    const dataArray = new Uint8Array(bufferLength);
    
    // 1秒間に50回（20ミリ秒ごと）超細かくチェック
    const checkAudio = setInterval(() => {
        analyser.getByteFrequencyData(dataArray);
        let sum = 0;
        for(let i=0; i<bufferLength; i++) { sum += dataArray[i]; }
        let average = sum / bufferLength;
        
        // 判定基準を「2」まで下げて、iPhoneに消されかけた小さな「パ」も全部拾う！
        if (average > 2) { 
            if (!isSpeaking) {
                count++;
                isSpeaking = true;
            }
        } else {
            isSpeaking = false;
        }
    }, 20);
    
    setTimeout(() => {
        clearInterval(checkAudio);
        stream.getTracks().forEach(track => track.stop());
        if (audioContext) audioContext.close();
        
        countdownDisplay.innerHTML = `<span style="font-size:30px; color:#333;">終了しました！</span>`;
        startBtn.disabled = false;
        showResult(count);
    }, 1000);
}

function showResult(score) {
    let html = "";
    if (score >= 6) {
        html = `<div class="result success">結果：${score} 回 / 秒<br>🎉 目標達成！</div>`;
    } else {
        html = `<div class="result warning">結果：${score} 回 / 秒<br>⚠️ 目標は6回以上</div>`;
    }
    resultOutput.innerHTML = html;
}
</script>

</body>
</html>
