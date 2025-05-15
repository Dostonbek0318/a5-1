# a5-1
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>A5/1 Cipher Demo</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body {
      font-family: sans-serif;
      padding: 20px;
      background: #f0f0f0;
    }
    .container {
      background: white;
      border-radius: 10px;
      padding: 20px;
      max-width: 700px;
      margin: auto;
      box-shadow: 0 0 10px rgba(0,0,0,0.1);
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
      cursor: pointer;
    }
    button:hover {
      background: #005fa3;
    }
    pre {
      background: #f8f8f8;
      padding: 10px;
      border-radius: 6px;
      overflow-x: auto;
    }
  </style>
</head>
<body>
  <div class="container">
    <h2>A5/1 Shifrlash Algoritmi (JavaScript)</h2>
    <label>64-bit Kalit (masalan: 100100...):</label>
    <input id="keyInput" maxlength="64" placeholder="64 ta bitdan iborat bo‘lsin">

    <label>22-bit Frame Raqami (masalan: 0000000000000000000001):</label>
    <input id="frameInput" maxlength="22" placeholder="22 ta bitdan iborat bo‘lsin">

    <label>Keystream uzunligi (masalan: 114):</label>
    <input id="lengthInput" type="number" value="114">

    <button onclick="runA51()">Keystream hosil qilish</button>

    <h3>Natija:</h3>
    <pre id="output">Natija shu yerda ko‘rsatiladi...</pre>
  </div>

  <script>
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

        // 100 marta majority clock bilan siljitish
        for (let i = 0; i < 100; i++) {
          this.majorityClock();
        }
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
        return stream.join('');
      }
    }

    function runA51() {
      const key = document.getElementById("keyInput").value.trim();
      const frame = document.getElementById("frameInput").value.trim();
      const len = parseInt(document.getElementById("lengthInput").value.trim());

      const output = document.getElementById("output");

      if (key.length !== 64 || frame.length !== 22 || isNaN(len) || len <= 0) {
        output.textContent = "⚠️ Iltimos, to‘g‘ri formatda ma’lumot kiriting!";
        return;
      }

      try {
        const cipher = new A51(key, frame);
        const keystream = cipher.getKeystream(len);
        output.textContent = "🔑 Keystream:\n" + keystream;
      } catch (e) {
        output.textContent = "Xatolik: " + e.message;
      }
    }
  </script>
</body>
</html>
