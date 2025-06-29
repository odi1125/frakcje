<!DOCTYPE html>
<html lang="pl">
<head>
  <meta charset="UTF-8">
  <title>Dodaj swój wóz</title>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap" rel="stylesheet">
  <style>
    * {
      box-sizing: border-box;
    }

    body {
      font-family: 'Inter', sans-serif;
      background: linear-gradient(to bottom right, #e3f2fd, #f8bbd0);
      margin: 0;
      padding: 20px;
    }

    h1 {
      text-align: center;
      margin-bottom: 40px;
      color: #333;
      font-size: 2.5em;
    }

    form {
      max-width: 600px;
      margin: 0 auto 50px auto;
      background: white;
      padding: 30px;
      border-radius: 16px;
      box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
    }

    input, select, button {
      width: 100%;
      margin-top: 15px;
      padding: 12px 15px;
      font-size: 16px;
      border: 1px solid #ccc;
      border-radius: 10px;
      outline: none;
      transition: border 0.2s ease;
    }

    input:focus, select:focus {
      border-color: #2196f3;
    }

    button {
      background: #2196f3;
      color: white;
      font-weight: bold;
      border: none;
      cursor: pointer;
      transition: background 0.3s ease;
    }

    button:hover {
      background: #1976d2;
    }

    .woz {
      max-width: 600px;
      margin: 20px auto;
      background: white;
      padding: 20px;
      border-radius: 16px;
      box-shadow: 0 10px 25px rgba(0,0,0,0.1);
      transition: transform 0.3s;
    }

    .woz:hover {
      transform: translateY(-5px);
    }

    .woz h2 {
      margin-top: 0;
      color: #0d47a1;
    }

    .woz img {
      width: 100%;
      max-height: 300px;
      object-fit: cover;
      border-radius: 12px;
      margin: 15px 0;
    }

    audio {
      width: 100%;
      outline: none;
    }

    @media (max-width: 600px) {
      h1 {
        font-size: 2em;
      }

      form, .woz {
        padding: 20px;
      }
    }
  </style>
</head>
<body>

  <h1>🚗 Dodaj swój wóz</h1>

  <form id="formularz">
    <input type="text" id="nazwa" placeholder="Nazwa wozu" required />
    <select id="marka" required>
      <option value="">Wybierz markę</option>
      <option value="MAN">MAN</option>
      <option value="Scania">Scania</option>
      <option value="Mercedes">Mercedes</option>
    </select>
    <input type="file" id="zdjecie" accept="image/*" required />
    <input type="file" id="dzwiek" accept="audio/*" required />
    <button type="submit">📤 Dodaj wóz</button>
  </form>

  <div id="listaWozow"></div>

  <script>
    const form = document.getElementById('formularz');
    const lista = document.getElementById('listaWozow');

    function pokazWozy() {
      lista.innerHTML = '';
      const wozy = JSON.parse(localStorage.getItem('wozy') || '[]');

      wozy.forEach((woz, index) => {
        const div = document.createElement('div');
        div.className = 'woz';
        div.innerHTML = `
          <h2>${woz.nazwa} (${woz.marka})</h2>
          <img src="${woz.zdjecie}" />
          <audio controls src="${woz.dzwiek}"></audio>
        `;
        lista.appendChild(div);
      });
    }

    form.addEventListener('submit', async (e) => {
      e.preventDefault();

      const nazwa = document.getElementById('nazwa').value.trim();
      const marka = document.getElementById('marka').value;
      const zdjeciePlik = document.getElementById('zdjecie').files[0];
      const dzwiekPlik = document.getElementById('dzwiek').files[0];

      if (!zdjeciePlik || !dzwiekPlik) return;

      const zdjecieBase64 = await toBase64(zdjeciePlik);
      const dzwiekBase64 = await toBase64(dzwiekPlik);

      const nowyWoz = { nazwa, marka, zdjecie: zdjecieBase64, dzwiek: dzwiekBase64 };
      const wozy = JSON.parse(localStorage.getItem('wozy') || '[]');
      wozy.push(nowyWoz);
      localStorage.setItem('wozy', JSON.stringify(wozy));

      form.reset();
      pokazWozy();
    });

    function toBase64(file) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = () => resolve(reader.result);
        reader.onerror = reject;
        reader.readAsDataURL(file);
      });
    }

    pokazWozy();
  </script>

</body>
</html>

