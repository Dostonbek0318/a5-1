<!DOCTYPE html>
<html lang="uz">
<head>
  <meta charset="UTF-8" />
  <title>A5/1 Shifrlash va Deshifrlash</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <style>
    body {
      font-family: Arial, sans-serif;
      background: #f0f4f8;
      padding: 20px;
    }
    .container {
      max-width: 700px;
      margin: auto;
      background: white;
      padding: 20px;
      border-radius: 8px;
      box-shadow: 0 0 12px rgba(0,0,0,0.1);
    }
    h2 {
      text-align: center;
      margin-bottom: 20px;
      color: #333;
    }
    .tabs {
      display: flex;
      justify-content: center;
      margin-bottom: 20px;
    }
    .tab-btn {
      flex: 1;
      padding: 12px 0;
      border: none;
      cursor: pointer;
      background: #cbd5e1;
      color: #334155;
      font-size: 16px;
      font-weight: bold;
      transition: background-color 0.3s ease;
    }
    .tab-btn.active {
      background: #2563eb;
      color: white;
    }
    label {
      font-weight: 600;
      margin-top: 15px;
      display: block;
      color: #475569;
    }
    input[type="text"], textarea {
      width: 100%;
      padding: 10px;
      margin-top: 6px;
      border-radius: 6px;
      border: 1px solid #94a3b8;
      font-size: 16px;
      resize: vertical;
      font-family: monospace;
    }
    input[type="file"] {
      margin-top: 10px;
    }
    button {
      margin-top: 18px;
      padding: 12px;
      width: 100%;
      font-size: 16px;
      background: #2563eb;
      color: white;
      border: none;
      border-radius: 6px;
      cursor: pointer;
      font-weight: bold;
      transition: background-color 0.3s ease;
    }
    button:hover {
      background: #1e40af;
    }
    pre {
      background: #e2e8f0;
      padding: 12px;
      border-radius: 6px;
      margin-top: 15px;
      white-space: pre-wrap;
      word-wrap: break-word;
      font-family: monospace;
      min-height: 80px;
      color: #1e293b;
    }
  </style>
</head>
<body>
  <div class="container">
    <h2>A5/1 Shifrlash va Deshifrlash</h2>

    <div class="tabs">
      <button class="tab-btn active" onclick="switchTab('encrypt', this)">🔐 Shifrlash</button>
      <button class="tab-btn" onclick="switchTab('decrypt', this)">🔓 Deshifrlash</button>
    </div>

    <section id="encrypt" class="tab-content">
      <label for="keyEncrypt">64-bit Kalit (0 va 1 dan iborat):</label>
      <input id="keyEncrypt" type="text" maxlength="64" placeholder="Masalan: 010101...64 ta bit" />

      <label for="frameEncrypt">22-bit Frame (0 va 1 dan iborat):</label>
      <input id="frameEncrypt" type="text" maxlength="22" placeholder="Masalan: 0000000000000000000001" />

      <label for="plainText">Matn yoki Word (.docx) fayl yuklash orqali matn oling:</label>
      <textarea id="plainText" rows="4" placeholder="Matn kiriting..."></textarea>
      <input type="file" id="wordFile" accept=".docx" onchange="loadWordFile()" />

      <button onclick="encrypt()">Shifrlash</button>

      <label>Shifrlangan bitlar (0 va 1):</label>
      <pre id="cipherOutput">Natija shu yerda chiqadi...</pre>
    </section>

    <section id="decrypt" class="tab-content" style="display:none;">
      <label for="keyDecrypt">64-bit Kalit (0 va 1 dan iborat):</label>
      <input id="keyDecrypt" type="text" maxlength="64" placeholder="Masalan: 010101...64 ta bit" />

      <label for="frameDecrypt">22-bit Frame (0 va 1 dan iborat):</label>
      <input id="frameDecrypt" type="text" maxlength="22" placeholder="Masalan: 0000000000000000000001" />

      <label for="cipherText">Shifrlangan bitlar (0 va 1):</label>
      <textarea id="cipherText" rows="4" placeholder="Shifrlangan bitlarni kiriting..."></textarea>

      <button onclick="decrypt()">Deshifrlash</button>

      <label>Deshifrlangan matn:</label>
      <pre id="plainOutput">Natija shu yerda chiqadi...</pre>
    </section>
  </div>

  <!-- Mammoth.js (Word faylidan matn olish uchun) -->
  <script src="https://unpkg.com/mammoth/mammoth.browser.min.js"></script>

  <script>
    // Tablarni almashtirish funksiyasi
    function switchTab(id, btn) {
      document.querySelectorAll('.tab-content').forEach(section => {
        section.style.display = 'none';
      });
      document.getElementById(id).style.display = 'block';

      document.querySelectorAll('.tab-btn').forEach(button => {
        button.classList.remove('active');
      });
      btn.classList.add('active');
    }

    // DOCX fayldan matn olish
    function loadWordFile() {
      const fileInput = document.getElementById('wordFile');
      const file = fileInput.files[0];
      if (!file) return;

      const reader = new FileReader();
      reader.onload = function(event) {
        const arrayBuffer = event.target.result;
        mammoth.extractRawText({arrayBuffer: arrayBuffer})
          .then(function(result) {
            document.getElementById('plainText').value = result.value.trim();
          })
          .catch(function(err) {
            alert('Word faylni o‘qishda xatolik: ' + err);
          });
      };
      reader.readAsArrayBuffer(file);
    }

    // A5/1 LFSR klassi
    class LFSR {
      constructor(size, taps, clockingBit) {
        this.size = size;
        this.taps = taps;
        this.clockingBit = clockingBit;
        this.reg = Array(size).fill(0);
      }

      shift() {
        const feedback = this.taps.reduce((acc, t) => acc ^ this.reg[t], 0);
        this.reg.pop();
        this.reg.unshift(feedback);
      }

      getClockingBit() {
        return this.reg[this.clockingBit];
      }

      getOutputBit() {
        return this.reg[this.size - 1];
      }

      xorBit(bit) {
        this.reg[0] ^= bit;
        this.shift();
      }
    }

    // A5/1 algoritmi klassi
    class A51 {
      constructor(key, frame) {
        this.R1 = new LFSR(19, [13, 16, 17, 18], 8);
        this.R2 = new LFSR(22, [20, 21], 10);
        this.R3 = new LFSR(23, [7, 20, 21, 22], 10);

        this.loadKey(key);
        this.loadFrame(frame);

        // 100 marta majburiy aylantirish
        for (let i = 0; i < 100; i++) this.majorityClock();
      }

      majorityClock() {
        const m = this.majority(
          this.R1.getClockingBit(),
          this.R2.getClockingBit(),
          this.R3.getClockingBit()
        );
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

    // Matnni bitlarga aylantirish (har belgi ASCII 8 bit)
    function textToBits(text) {
      let bits = "";
      for (let i = 0; i < text.length; i++) {
        bits += text.charCodeAt(i).toString(2).padStart(8, "0");
      }
      return bits;
    }

    // Bitlardan matnga aylantirish
    function bitsToText(bits) {
      let text = "";
      for (let i = 0; i < bits.length; i += 8) {
        let byte = bits.substr(i, 8);
        if (byte.length < 8) break;
        text += String.fromCharCode(parseInt(byte, 2));
      }
      return text;
    }

    // Ikki bit ketma-ketligini XOR qilish
    function xorBits(bits1, bits2) {
      return bits1.split('').map((b, i) => b ^ bits2[i]).join('');
    }

    // Shifrlash funksiyasi
    function encrypt() {
      const key = document.getElementById('keyEncrypt').value.trim();
      const frame = document.getElementById('frameEncrypt').value.trim();
      const plainText = document.getElementById('plainText').value;

      const output = document.getElementById('cipherOutput');

      if (key.length !== 64 || frame.length !== 22 || plainText.length === 0) {
        output.textContent = "⚠️ Iltimos, 64 bit kalit, 22 bit frame va matn kiriting.";
        return;
      }

      if (!/^[01]+$/.test(key) || !/^[01]+$/.test(frame)) {
        output.textContent = "⚠️ Kalit va frame faqat 0 va 1 dan iborat bo‘lishi kerak!";
        return;
      }

      const plainBits = textToBits(plainText);
      const cipher =new A51(key, frame);
const keystream = cipher.getKeystream(plainBits.length).join('');
const cipherBits = xorBits(plainBits, keystream);

javascript
Copy
Edit
  output.textContent = cipherBits;
}

// Deshifrlash funksiyasi
function decrypt() {
  const key = document.getElementById('keyDecrypt').value.trim();
  const frame = document.getElementById('frameDecrypt').value.trim();
  const cipherBits = document.getElementById('cipherText').value.trim();

  const output = document.getElementById('plainOutput');

  if (key.length !== 64 || frame.length !== 22 || cipherBits.length === 0) {
    output.textContent = "⚠️ Iltimos, 64 bit kalit, 22 bit frame va shifrlangan bitlarni kiriting.";
    return;
  }

  if (!/^[01]+$/.test(key) || !/^[01]+$/.test(frame)) {
    output.textContent = "⚠️ Kalit va frame faqat 0 va 1 dan iborat bo‘lishi kerak!";
    return;
  }

  if (!/^[01]+$/.test(cipherBits)) {
    output.textContent = "⚠️ Shifrlangan bitlar faqat 0 va 1 dan iborat bo‘lishi kerak!";
    return;
  }

  const cipher = new A51(key, frame);
  const keystream = cipher.getKeystream(cipherBits.length).join('');
  const plainBits = xorBits(cipherBits, keystream);
  const text = bitsToText(plainBits);

  output.textContent = text;
}
</script> </body> </html> ```
