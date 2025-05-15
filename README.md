<!DOCTYPE html>
<html lang="uz">
<head>
  <meta charset="UTF-8" />
  <title>A5/1 Shifrlash Algoritmi</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <style>
    body {
      font-family: sans-serif;
      background: #eef2f7;
      padding: 20px;
    }
    .container {
      max-width: 800px;
      margin: auto;
      background: white;
      padding: 20px;
      border-radius: 12px;
      box-shadow: 0 0 10px rgba(0,0,0,0.1);
    }
    h2 {
      text-align: center;
      margin-bottom: 20px;
    }
    .tabs {
      display: flex;
      justify-content: center;
      margin-bottom: 20px;
    }
    .tab-btn {
      flex: 1;
      padding: 10px;
      background: #d9e2ec;
      border: none;
      cursor: pointer;
      font-size: 16px;
    }
    .tab-btn.active {
      background: #0077cc;
      color: white;
    }
    .section {
      display: none;
    }
    .section.active {
      display: block;
    }
    input, textarea, button {
      width: 100%;
      padding: 10px;
      margin-top: 10px;
      border-radius: 6px;
      border: 1px solid #ccc;
      font-size: 16px;
    }
    button {
      background: #0077cc;
      color: white;
      margin-top: 15px;
      cursor: pointer;
    }
    button:hover {
      background: #005fa3;
    }
    pre {
      background: #f9f9f9;
      padding: 10px;
      margin-top: 15px;
      border-radius: 6px;
      white-space: pre-wrap;
      word-wrap: break-word;
    }
    label {
      font-weight: bold;
      margin-top: 10px;
      display: block;
    }
  </style>
</head>
<body>
  <div class="container">
    <h2>A5/1 Shifrlash / Deshifrlash</h2>

    <div class="tabs">
      <button class="tab-btn active" onclick="showTab(event, 'encrypt')">🔐 Shifrlash</button>
      <button class="tab-btn" onclick="showTab(event, 'decrypt')">🔓 Deshifrlash</button>
    </div>

    <div id="encrypt" class="section active">
      <label>64-bit Kalit:</label>
      <input id="keyInput1" maxlength="64" placeholder="Masalan: 1001..." />

      <label>22-bit Frame:</label>
      <input id="frameInput1" maxlength="22" placeholder="Masalan: 0000000000000000000001" />

      <label>Matn:</label>
      <textarea id="plainText" rows="3" placeholder="Shifrlanadigan matn"></textarea>

      <label>Yoki Word (.docx) fayl yuklang:</label>
      <input type="file" id="wordFile" accept=".docx" onchange="loadWordFile()" />

      <button onclick="encryptText()">🛡 Shifrlash</button>

      <h3>Natija:</h3>
      <pre id="outputEncrypt">Shifrlangan bitlar bu yerda chiqadi...</pre>
    </div>

    <div id="decrypt" class="section">
      <label>64-bit Kalit:</label>
      <input id="keyInput2" maxlength="64" placeholder="Masalan: 1001..." />

      <label>22-bit Frame:</label>
      <input id="frameInput2" maxlength="22" placeholder="Masalan: 0000000000000000000001" />

      <label>Shifrlangan bitlar:</label>
      <textarea id="cipherBits" rows="3" placeholder="0 va 1 lardan iborat"></textarea>

      <button onclick="decryptText()">🔓 Deshifrlash</button>

      <h3>Natija:</h3>
      <pre id="outputDecrypt">Deshifrlangan matn bu yerda chiqadi...</pre>
    </div>
  </div>

  <script src="https://unpkg.com/mammoth/mammoth.browser.min.js"></script>
  <script>
    function showTab(event, tabId) {
      document.querySelectorAll('.section').forEach(s => s.classList.remove('active'));
      document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
      document.getElementById(tabId).classList.add('active');
      event.target.classList.add('active');
    }

    function loadWordFile() {
      const file = document.getElementById("wordFile").files[0];
      if (!file) return;
      const reader = new FileReader();
      reader.onload = function(event) {
        const arrayBuffer = event.target.result;
        mammoth.extractRawText({ arrayBuffer: arrayBuffer })
          .then(function(result) {
            document.getElementById("plainText").value = result.value.trim();
          })
          .catch(err => alert("Xatolik: " + err));
      };
      reader.readAsArrayBuffer(file);
    }

    class LFSR {
      constructor(size, taps, clockingBit) {
        this.size = size;
        this.taps = taps;
        this.clockingBit = clockingBit;
        this.reg = Array(size).fill(0);
      }
      shift() {
        let feedback = this.taps.reduce((acc, t) => acc ^ this.reg[t], 0);
        this.reg.pop();
        this.reg.unshift(feedback);
      }
      getClockingBit() {
        return this.reg[this.clockingBit];
      }
      getOutputBit() {
        return this.reg[this.reg.length - 1];
      }
      xorBit(bit) {
        this.reg[0] ^= bit;
        this.shift();
      }
    }

    class A51 {
      constructor(key, frame) {
        this.R1 = new LFSR(19, [13, 16, 17, 18], 8);
        this.R2 = new LFSR(22, [20, 21], 10);
        this.R3 = new LFSR(23, [7, 20, 21, 22], 10);
        this.loadKey(key);
        this.loadFrame(frame);
        for (let i = 0; i < 100; i++) this.majorityClock();
      }
      majorityClock() {
        const m = this.majority(this.R1.getClockingBit(), this.R2.getClockingBit(), this.R3.getClockingBit());
        if (this.R1.getClockingBit() === m) this.R1.shift();
        if (this.R2.getClockingBit() === m) this.R2.shift();
        if (this.R3.getClockingBit() === m) this.R3.shift();
      }
      majority(a, b, c) {
        return (a + b + c) >= 2 ? 1 : 0;
      }
      loadKey(key) {
        for (let i = 0; i < 64; i++) {
          const bit = parseInt(key[i]);
          this.R1.xorBit(bit);
          this.R2.xorBit(bit);
          this.R3.xorBit(bit);
        }
      }
      loadFrame(frame) {
        for (let i = 0; i < 22; i++) {
          const bit = parseInt(frame[i]);
          this.R1.xorBit(bit);
          this.R2.xorBit(bit);
          this.R3.xorBit(bit);
        }
      }
      getKeystream(length) {
        const stream = [];
        for (let i = 0; i < length; i++) {
          this.majorityClock();
          const bit = this.R1.getOutputBit() ^ this.R2.getOutputBit() ^ this.R3.getOutputBit();
          stream.push(bit);
        }
        return stream;
      }
    }

   function textToBits(text) {
  const encoder = new TextEncoder();
  const bytes = encoder.encode(text);
  return Array.from(bytes).map(byte => byte.toString(2).padStart(8, '0')).join('');
}

function bitsToText(bits) {
  const bytes = [];
  for (let i = 0; i < bits.length; i += 8) {
    const byte = bits.slice(i, i + 8);
    if (byte.length === 8) {
      bytes.push(parseInt(byte, 2));
    }
  }
  const decoder = new TextDecoder();
  return decoder.decode(new Uint8Array(bytes));
}

    function xorBits(a, b) {
      return a.split('').map((bit, i) => bit ^ b[i]).join('');
    }

    function encryptText() {
      const key = document.getElementById("keyInput1").value.trim();
      const frame = document.getElementById("frameInput1").value.trim();
      const plain = document.getElementById("plainText").value;
      const output = document.getElementById("outputEncrypt");

      if (key.length !== 64 || frame.length !== 22 || plain.length === 0) {
        output.textContent = "⚠️ Kirish ma’lumotlari to‘liq emas!";
        return;
      }

      const bits = textToBits(plain);
      const cipher = new A51(key, frame);
      const keystream = cipher.getKeystream(bits.length).join('');
      const cipherBits = xorBits(bits, keystream);

      output.textContent = cipherBits;
    }

    function decryptText() {
      const key = document.getElementById("keyInput2").value.trim();
      const frame = document.getElementById("frameInput2").value.trim();
      const cipherBits = document.getElementById("cipherBits").value.trim();
      const output = document.getElementById("outputDecrypt");

      if (key.length !== 64 || frame.length !== 22 || cipherBits.length === 0) {
        output.textContent = "⚠️ Kirish ma’lumotlari to‘liq emas!";
        return;
      }

      const cipher = new A51(key, frame);
      const keystream = cipher.getKeystream(cipherBits.length).join('');
      const plainBits = xorBits(cipherBits, keystream);
      const text = bitsToText(plainBits);

      output.textContent = text;
    }
  </script>
</body>
</html>
