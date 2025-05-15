<!DOCTYPE html>ga
<html lang="uz">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>A5/1 Shifrlash va Deshifrlash</title>
<style>
  body { font-family: Arial, sans-serif; margin: 20px; background: #f0f0f0; }
  h1 { text-align: center; }
  label { display: block; margin-top: 15px; }
  textarea, input[type=text] { width: 100%; padding: 8px; font-family: monospace; font-size: 14px; }
  button { margin-top: 10px; padding: 10px 20px; font-size: 16px; cursor: pointer; }
  .output { margin-top: 15px; white-space: pre-wrap; background: #fff; padding: 10px; border: 1px solid #ccc; min-height: 60px; }
</style>
</head>
<body>

<h1>A5/1 Shifrlash va Deshifrlash</h1>

<label for="keyInput">64-bitli Kalit (faqat 0 va 1):</label>
<input id="keyInput" type="text" maxlength="64" placeholder="64 bitli kalitni kiriting (0 va 1 dan)" />

<label for="frameInput">22-bitli Frame (faqat 0 va 1):</label>
<input id="frameInput" type="text" maxlength="22" placeholder="22 bitli frame kiriting (0 va 1 dan)" />

<label for="plainText">Matn (shifrlash uchun):</label>
<textarea id="plainText" rows="4" placeholder="Shifrlash uchun matn kiriting"></textarea>
<button id="btnEncrypt">Shifrlash</button>

<label for="cipherText">Shifrlangan bitlar (0 va 1):</label>
<textarea id="cipherText" rows="4" placeholder="Deshifrlash uchun shifrlangan bitlarni kiriting"></textarea>
<button id="btnDecrypt">Deshifrlash</button>

<h3>Natija:</h3>
<div id="output" class="output"></div>

<script>
  // A5/1 algoritmining asosiy qismi
  class A51 {
    constructor(key, frame) {
      this.R1 = new Array(19).fill(0);
      this.R2 = new Array(22).fill(0);
      this.R3 = new Array(23).fill(0);
      this.init(key, frame);
    }

    init(key, frame) {
      // Reset registorlar
      this.R1.fill(0);
      this.R2.fill(0);
      this.R3.fill(0);

      // Load key (64 bit)
      for (let i = 0; i < 64; i++) {
        this.clockAll();
        this.R1[0] ^= parseInt(key[i]);
        this.R2[0] ^= parseInt(key[i]);
        this.R3[0] ^= parseInt(key[i]);
      }

      // Load frame (22 bit)
      for (let i = 0; i < 22; i++) {
        this.clockAll();
        this.R1[0] ^= parseInt(frame[i]);
        this.R2[0] ^= parseInt(frame[i]);
        this.R3[0] ^= parseInt(frame[i]);
      }

      // 100 clock pulsesi
      for (let i = 0; i < 100; i++) {
        this.clockMajority();
      }
    }

    clockAll() {
      this.shiftR1();
      this.shiftR2();
      this.shiftR3();
    }

    clockMajority() {
      const m = this.majority(this.R1[8], this.R2[10], this.R3[10]);
      if (this.R1[8] === m) this.shiftR1();
      if (this.R2[10] === m) this.shiftR2();
      if (this.R3[10] === m) this.shiftR3();
    }

    majority(x, y, z) {
      return (x + y + z) > 1 ? 1 : 0;
    }

    shiftR1() {
      const newBit = this.R1[13] ^ this.R1[16] ^ this.R1[17] ^ this.R1[18];
      this.R1.pop();
      this.R1.unshift(newBit);
    }

    shiftR2() {
      const newBit = this.R2[20] ^ this.R2[21];
      this.R2.pop();
      this.R2.unshift(newBit);
    }

    shiftR3() {
      const newBit = this.R3[7] ^ this.R3[20] ^ this.R3[21] ^ this.R3[22];
      this.R3.pop();
      this.R3.unshift(newBit);
    }

    getKeystream(length) {
      const ks = [];
      for (let i = 0; i < length; i++) {
        this.clockMajority();
        const bit = this.R1[18] ^ this.R2[21] ^ this.R3[22];
        ks.push(bit);
      }
      return ks;
    }
  }

  // Matnni bitlarga aylantirish (8 bit/harf)
  function textToBits(text) {
    let bits = "";
    for (let i = 0; i < text.length; i++) {
      let bin = text.charCodeAt(i).toString(2);
      bits += "00000000".slice(bin.length) + bin;
    }
    return bits;
  }

  // Bitlarni matnga aylantirish
  function bitsToText(bits) {
    let text = "";
    for (let i = 0; i + 8 <= bits.length; i += 8) {
      const byte = bits.slice(i, i + 8);
      const charCode = parseInt(byte, 2);
      text += String.fromCharCode(charCode);
    }
    return text;
  }

  // XOR operatsiyasi ikkita bitli satrga
  function xorBits(a, b) {
    let res = "";
    for (let i = 0; i < a.length; i++) {
      res += (a[i] ^ b[i]);
    }
    return res;
  }

  const output = document.getElementById("output");

  document.getElementById("btnEncrypt").addEventListener("click", () => {
    const key = document.getElementById("keyInput").value.trim();
    const frame = document.getElementById("frameInput").value.trim();
    const plain = document.getElementById("plainText").value;

    if (key.length !== 64 || frame.length !== 22) {
      output.textContent = "⚠️ Iltimos, 64 bitli kalit va 22 bitli frame kiriting (0 va 1 dan).";
      return;
    }
    if (!/^[01]+$/.test(key) || !/^[01]+$/.test(frame)) {
      output.textContent = "⚠️ Kalit va frame faqat 0 va 1 dan iborat bo‘lishi kerak!";
      return;
    }
    if (plain.length === 0) {
      output.textContent = "⚠️ Iltimos, shifrlash uchun matn kiriting.";
      return;
    }

    const plainBits = textToBits(plain);
    const cipher = new A51(key, frame);
    const keystream = cipher.getKeystream(plainBits.length).join("");
    const cipherBits = xorBits(plainBits, keystream);

    // Natijani chiqaramiz
    output.textContent = "Shifrlangan bitlar:\n" + cipherBits;
    // Shifrlangan bitlarni pastdagi maydonga ham joylaymiz (deshifrlash uchun)
    document.getElementById("cipherText").value = cipherBits;
  });

  document.getElementById("btnDecrypt").addEventListener("click", () => {
    const key = document.getElementById("keyInput").value.trim();
    const frame = document.getElementById("frameInput").value.trim();
    const cipherBits = document.getElementById("cipherText").value.trim();

    if (key.length !== 64 || frame.length !== 22) {
      output.textContent = "⚠️ Iltimos, 64 bitli kalit va 22 bitli frame kiriting (0 va 1 dan).";
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
    if (cipherBits.length % 8 !== 0) {
      output.textContent = "⚠️ Shifrlangan bitlar uzunligi 8 ning ko‘paytmasi bo‘lishi kerak!";
      return;
    }

    const cipher = new A51(key, frame);
    const keystream = cipher.getKeystream(cipherBits.length).join("");
    const plainBits = xorBits(cipherBits, keystream);
    const text = bitsToText(plainBits);

    output.textContent = "Deshifrlangan matn:\n" + text;
  });
</script>

</body>
</html>
