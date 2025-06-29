const express = require('express');
const multer = require('multer');
const path = require('path');
const sqlite3 = require('sqlite3').verbose();
const fs = require('fs');

const app = express();
const PORT = process.env.PORT || 4000;

// Utwórz /uploads, jeśli nie istnieje
const UPLOADS_DIR = path.join(__dirname, 'uploads');
if (!fs.existsSync(UPLOADS_DIR)) {
  fs.mkdirSync(UPLOADS_DIR);
}

// Multer do uploadu plików
const storage = multer.diskStorage({
  destination: (req, file, cb) => cb(null, UPLOADS_DIR),
  filename: (req, file, cb) => {
    const unique = Date.now() + '-' + Math.round(Math.random() * 1e9);
    cb(null, unique + path.extname(file.originalname));
  }
});
const upload = multer({ storage });

// Inicjalizacja bazy
const db = new sqlite3.Database(path.join(__dirname, 'factions.sqlite'));
db.serialize(() => {
  db.run(`
    CREATE TABLE IF NOT EXISTS factions(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      filter TEXT NOT NULL,
      imagePath TEXT NOT NULL,
      audioPath TEXT NOT NULL
    )
  `);
});

// Middleware do parsowania JSON i uploadu
app.use(express.json());
app.use('/uploads', express.static(UPLOADS_DIR));

// API - dodaj frakcję
app.post('/api/factions', upload.fields([{ name: 'image'}, { name: 'audio'}]), (req, res) => {
  const { name, filter } = req.body;
  const imageFile = req.files['image'] ? req.files['image'][0] : null;
  const audioFile = req.files['audio'] ? req.files['audio'][0] : null;

  if(!name || !filter || !imageFile || !audioFile) {
    return res.status(400).json({ error: 'Podaj nazwę, filtr oraz dodaj oba pliki (zdjęcie i dźwięk).' });
  }

  const imagePath = `/uploads/${imageFile.filename}`;
  const audioPath = `/uploads/${audioFile.filename}`;

  db.run(
    'INSERT INTO factions (name, filter, imagePath, audioPath) VALUES (?, ?, ?, ?)',
    [name, filter, imagePath, audioPath],
    function(err) {
      if(err) {
        console.error(err);
        return res.status(500).json({ error: 'Błąd serwera' });
      }
      res.json({ id: this.lastID, name, filter, imagePath, audioPath });
    }
  );
});

// API - pobierz frakcje z opcjonalnym filtrem
app.get('/api/factions', (req, res) => {
  const filter = req.query.filter || '';
  let sql = 'SELECT * FROM factions';
  const params = [];
  if(filter) {
    sql += ' WHERE filter LIKE ?';
    params.push(`%${filter}%`);
  }
  db.all(sql, params, (err, rows) => {
    if(err) {
      console.error(err);
      return res.status(500).json({ error: 'Błąd serwera' });
    }
    res.json(rows);
  });
});

// Serwuje frontend (strona HTML + JS)
app.get('/', (req, res) => {
  res.send(`
<!DOCTYPE html>
<html lang="pl">
<head>
  <meta charset="UTF-8" />
  <title>Frakcje</title>
  <style>
    body { font-family: Arial, sans-serif; max-width: 700px; margin: 2rem auto; }
    form div { margin-bottom: 0.5rem; }
    img { max-width: 200px; height: auto; display: block; margin: 0.5rem 0; }
    audio { width: 100%; }
    ul { list-style:none; padding:0; }
    li { border: 1px solid #ccc; margin-bottom: 1rem; padding: 1rem; }
  </style>
</head>
<body>
  <h1>Frakcje - dodawaj i wyszukuj</h1>

  <form id="searchForm" style="margin-bottom: 1rem;">
    <input type="text" id="searchInput" placeholder="Wpisz filtr np. 'man'" />
    <button type="submit">Szukaj</button>
    <button type="button" id="clearSearch">Wyczyść</button>
  </form>

  <h2>Dodaj frakcję</h2>
  <form id="addForm" enctype="multipart/form-data">
    <div><input required name="name" placeholder="Nazwa" /></div>
    <div><input required name="filter" placeholder="Filtr (np. man)" /></div>
    <div><label>Zdjęcie: <input type="file" required name="image" accept="image/*" /></label></div>
    <div><label>Dźwięk: <input type="file" required name="audio" accept="audio/*" /></label></div>
    <div><button type="submit">Dodaj frakcję</button></div>
  </form>
  <div id="error" style="color: red;"></div>

  <h2>Frakcje</h2>
  <ul id="list"></ul>

  <script>
    const listEl = document.getElementById('list');
    const errorEl = document.getElementById('error');
    const searchForm = document.getElementById('searchForm');
    const searchInput = document.getElementById('searchInput');
    const clearSearchBtn = document.getElementById('clearSearch');
    const addForm = document.getElementById('addForm');

    async function fetchFactions(filter='') {
      errorEl.textContent = '';
      try {
        let url = '/api/factions';
        if(filter) url += '?filter=' + encodeURIComponent(filter);
        const res = await fetch(url);
        const data = await res.json();
        renderFactions(data);
      } catch(e) {
        errorEl.textContent = 'Błąd podczas pobierania danych.';
      }
    }

    function renderFactions(items) {
      if(items.length === 0) {
        listEl.innerHTML = '<li>Brak frakcji do wyświetlenia.</li>';
        return;
      }
      listEl.innerHTML = '';
      for(const f of items) {
        const li = document.createElement('li');
        li.innerHTML = \`
          <h3>\${f.name}</h3>
          <p><b>Filtr:</b> \${f.filter}</p>
          <img src="\${f.imagePath}" alt="\${f.name}" />
          <audio controls src="\${f.audioPath}"></audio>
        \`;
        listEl.appendChild(li);
      }
    }

    addForm.addEventListener('submit', async (e) => {
      e.preventDefault();
      errorEl.textContent = '';
      const formData = new FormData(addForm);
      try {
        const res = await fetch('/api/factions', {
          method: 'POST',
          body: formData,
        });
        if(!res.ok) {
          const err = await res.json();
          throw new Error(err.error || 'Błąd serwera');
        }
        const newFaction = await res.json();
        addForm.reset();
        fetchFactions(searchInput.value);
      } catch(err) {
        errorEl.textContent = err.message;
      }
    });

    searchForm.addEventListener('submit', (e) => {
      e.preventDefault();
      fetchFactions(searchInput.value);
    });

    clearSearchBtn.addEventListener('click', () => {
      searchInput.value = '';
      fetchFactions();
    });

    // start
    fetchFactions();
  </script>
</body>
</html>
  `);
});

app.listen(PORT, () => {
  console.log(`Serwer działa na http://localhost:${PORT}`);
});
