memory-trips/
├── .gitignore
├── package.json
├── server.js
├── README.md
└── public/
    └── index.html
    {
  "name": "memory-trips",
  "version": "1.0.0",
  "description": "A website where people upload their memories and trips",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "license": "MIT",
  "dependencies": {
    "express": "^4.19.2",
    "multer": "^1.4.5-lts.1"
  }
}
node_modules/
uploads/
data/
.env
const express = require("express");
const multer = require("multer");
const path = require("path");
const fs = require("fs");
const crypto = require("crypto");

const PORT = process.env.PORT || 3000;
const UPLOAD_DIR = path.join(__dirname, "uploads");
const DATA_FILE = path.join(__dirname, "data", "memories.json");

fs.mkdirSync(UPLOAD_DIR, { recursive: true });
fs.mkdirSync(path.dirname(DATA_FILE), { recursive: true });
if (!fs.existsSync(DATA_FILE)) fs.writeFileSync(DATA_FILE, "[]");

const readMemories = () => JSON.parse(fs.readFileSync(DATA_FILE, "utf8"));
const writeMemories = (list) => fs.writeFileSync(DATA_FILE, JSON.stringify(list, null, 2));

const ALLOWED = { "image/jpeg": ".jpg", "image/png": ".png", "image/webp": ".webp", "image/gif": ".gif" };

const upload = multer({
  storage: multer.diskStorage({
    destination: UPLOAD_DIR,
    filename: (req, file, cb) =>
      cb(null, crypto.randomUUID() + ALLOWED[file.mimetype]),
  }),
  limits: { fileSize: 8 * 1024 * 1024, files: 6 }, // 8 MB each, up to 6 photos
  fileFilter: (req, file, cb) =>
    ALLOWED[file.mimetype] ? cb(null, true) : cb(new Error("Only JPG, PNG, WebP or GIF images are allowed.")),
});

const app = express();
app.use(express.static(path.join(__dirname, "public")));
app.use("/uploads", express.static(UPLOAD_DIR));

// List memories (newest first); optional ?q= search
app.get("/api/memories", (req, res) => {
  const q = (req.query.q || "").toString().toLowerCase();
  let list = readMemories().sort((a, b) => b.createdAt.localeCompare(a.createdAt));
  if (q) list = list.filter((m) => [m.title, m.place, m.story, m.author].join(" ").toLowerCase().includes(q));
  res.json(list);
});

// Add a memory
app.post("/api/memories", upload.array("photos", 6), (req, res) => {
  const clean = (v, max) => (v || "").toString().trim().slice(0, max);
  const title = clean(req.body.title, 100);
  if (!title) return res.status(400).json({ error: "Please add a title." });
  if (!req.files || !req.files.length) return res.status(400).json({ error: "Please add at least one photo." });

  const memory = {
    id: crypto.randomUUID(),
    title,
    place: clean(req.body.place, 100),
    date: clean(req.body.date, 10),
    author: clean(req.body.author, 60) || "Anonymous",
    story: clean(req.body.story, 2000),
    photos: req.files.map((f) => "/uploads/" + f.filename),
    createdAt: new Date().toISOString(),
  };
  const list = readMemories();
  list.push(memory);
  writeMemories(list);
  res.status(201).json(memory);
});

// Upload errors -> friendly JSON
app.use((err, req, res, next) => {
  res.status(400).json({ error: err.message || "Something went wrong." });
});

app.listen(PORT, () => console.log(`Memory Trips running at http://localhost:${PORT}`));
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Memory Trips</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;700&family=Young+Serif&display=swap" rel="stylesheet">
<style>
  :root {
    --paper: #eef2f4; --card: #ffffff; --ink: #1b2a33; --muted: #5c6f7a;
    --sea: #2f6f8f; --sun: #e8b04a; --line: #d3dde2;
  }
  * { box-sizing: border-box; }
  body { margin: 0; background: var(--paper); color: var(--ink); font: 16px/1.5 "DM Sans", system-ui, sans-serif; }
  header { max-width: 1100px; margin: 0 auto; padding: 48px 20px 24px; display: flex; flex-wrap: wrap; gap: 20px; align-items: end; justify-content: space-between; }
  h1 { font: 400 clamp(2.2rem, 6vw, 3.6rem)/1.05 "Young Serif", Georgia, serif; margin: 0; max-width: 14ch; }
  header p { margin: 8px 0 0; color: var(--muted); max-width: 42ch; }
  .actions { display: flex; gap: 10px; flex-wrap: wrap; }
  input, textarea, button { font: inherit; }
  input[type=search] { padding: 10px 14px; border: 1px solid var(--line); border-radius: 8px; background: var(--card); min-width: 220px; }
  .btn { background: var(--sea); color: #fff; border: 0; padding: 10px 18px; border-radius: 8px; font-weight: 700; cursor: pointer; }
  .btn:hover { background: #255a75; }
  .btn.ghost { background: transparent; color: var(--ink); border: 1px solid var(--line); }
  :focus-visible { outline: 3px solid var(--sun); outline-offset: 2px; }

  main { max-width: 1100px; margin: 0 auto; padding: 8px 20px 80px; columns: 3 280px; column-gap: 18px; }
  .memory { break-inside: avoid; background: var(--card); border-radius: 6px; margin: 0 0 18px; overflow: hidden; border: 1px solid var(--line); }
  .memory img { width: 100%; display: block; cursor: zoom-in; }
  .thumbs { display: flex; gap: 4px; padding: 4px 4px 0; }
  .thumbs img { width: 56px; height: 56px; object-fit: cover; border-radius: 3px; }
  .body { padding: 14px 16px 18px; }
  .body h2 { font: 400 1.35rem/1.2 "Young Serif", Georgia, serif; margin: 0 0 4px; }
  .meta { color: var(--muted); font-size: .9rem; margin: 0 0 8px; }
  .place { display: inline-block; background: var(--sun); color: #3a2a05; font-weight: 700; font-size: .85rem; padding: 2px 10px; border-radius: 999px; margin-bottom: 8px; }
  .story { margin: 0; white-space: pre-wrap; }
  .empty { column-span: all; text-align: center; color: var(--muted); padding: 60px 0; }

  dialog { border: 0; border-radius: 12px; padding: 0; width: min(520px, 94vw); box-shadow: 0 20px 60px rgba(27,42,51,.35); }
  dialog::backdrop { background: rgba(27,42,51,.55); }
  form { padding: 24px; display: grid; gap: 14px; }
  form h3 { font: 400 1.6rem "Young Serif", Georgia, serif; margin: 0; }
  label { display: grid; gap: 4px; font-weight: 500; font-size: .95rem; }
  form input, form textarea { padding: 10px 12px; border: 1px solid var(--line); border-radius: 8px; width: 100%; }
  .row { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
  .error { color: #b3261e; min-height: 1.2em; margin: 0; }
  .form-actions { display: flex; justify-content: flex-end; gap: 10px; }
  #viewer img { max-width: 94vw; max-height: 90vh; display: block; }
  #viewer { background: transparent; box-shadow: none; width: auto; }
  @media (max-width: 480px) { .row { grid-template-columns: 1fr; } }
</style>
</head>
<body>
<header>
  <div>
    <h1>Places we kept</h1>
    <p>Share a trip, a day out, or a moment you don't want to forget.</p>
  </div>
  <div class="actions">
    <input type="search" id="search" placeholder="Search places or stories" aria-label="Search memories">
    <button class="btn" id="openForm">Add a memory</button>
  </div>
</header>

<main id="grid" aria-live="polite"></main>

<dialog id="formDialog">
  <form id="memoryForm">
    <h3>Add a memory</h3>
    <label>Title <input name="title" required maxlength="100" placeholder="Sunrise at Munnar"></label>
    <div class="row">
      <label>Place <input name="place" maxlength="100" placeholder="Munnar, Kerala"></label>
      <label>Date <input name="date" type="date"></label>
    </div>
    <label>Your name <input name="author" maxlength="60" placeholder="Anonymous"></label>
    <label>Your story <textarea name="story" rows="4" maxlength="2000" placeholder="What made this day special?"></textarea></label>
    <label>Photos (up to 6, 8 MB each) <input name="photos" type="file" accept="image/*" multiple required></label>
    <p class="error" id="formError" role="alert"></p>
    <div class="form-actions">
      <button type="button" class="btn ghost" id="cancel">Cancel</button>
      <button class="btn" id="submitBtn">Post memory</button>
    </div>
  </form>
</dialog>

<dialog id="viewer"><img alt=""></dialog>

<script>
  const grid = document.getElementById("grid");
  const dlg = document.getElementById("formDialog");
  const viewer = document.getElementById("viewer");
  const form = document.getElementById("memoryForm");
  const errorEl = document.getElementById("formError");

  function el(tag, props = {}, ...kids) {
    const n = Object.assign(document.createElement(tag), props);
    kids.forEach(k => n.append(k));
    return n;
  }

  function show(src) { viewer.querySelector("img").src = src; viewer.showModal(); }
  viewer.addEventListener("click", () => viewer.close());

  function card(m) {
    const main = el("img", { src: m.photos[0], alt: m.title, loading: "lazy" });
    main.onclick = () => show(m.photos[0]);
    const c = el("article", { className: "memory" }, main);
    if (m.photos.length > 1) {
      const t = el("div", { className: "thumbs" });
      m.photos.slice(1).forEach(p => {
        const i = el("img", { src: p, alt: "", loading: "lazy" });
        i.onclick = () => show(p);
        t.append(i);
      });
      c.append(t);
    }
    const body = el("div", { className: "body" });
    if (m.place) body.append(el("span", { className: "place" }, m.place));
    body.append(el("h2", {}, m.title));
    const when = m.date ? new Date(m.date).toLocaleDateString(undefined, { year: "numeric", month: "long", day: "numeric" }) + " by " : "By ";
    body.append(el("p", { className: "meta" }, when + m.author));
    if (m.story) body.append(el("p", { className: "story" }, m.story));
    c.append(body);
    return c;
  }

  async function load(q = "") {
    const res = await fetch("/api/memories?q=" + encodeURIComponent(q));
    const list = await res.json();
    grid.replaceChildren();
    if (!list.length) grid.append(el("p", { className: "empty" }, q ? "No memories match your search." : "No memories yet. Add the first one."));
    list.forEach(m => grid.append(card(m)));
  }

  let timer;
  document.getElementById("search").addEventListener("input", e => {
    clearTimeout(timer);
    timer = setTimeout(() => load(e.target.value), 250);
  });
  document.getElementById("openForm").onclick = () => { errorEl.textContent = ""; dlg.showModal(); };
  document.getElementById("cancel").onclick = () => dlg.close();

  form.addEventListener("submit", async e => {
    e.preventDefault();
    const btn = document.getElementById("submitBtn");
    btn.disabled = true; btn.textContent = "Posting...";
    errorEl.textContent = "";
    try {
      const res = await fetch("/api/memories", { method: "POST", body: new FormData(form) });
      const data = await res.json();
      if (!res.ok) throw new Error(data.error);
      form.reset(); dlg.close(); load();
    } catch (err) {
      errorEl.textContent = err.message;
    } finally {
      btn.disabled = false; btn.textContent = "Post memory";
    }
  });

  load();
</script>
</body>
</html>
# Memory Trips

A simple website where people upload photos and stories from their memories and trips.

**Features:** photo upload (up to 6 per memory), title, place, date, story, search, full-size photo viewer.

## Run it

```bash
npm install
npm start
```

Open http://localhost:3000

## Push to GitHub

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/memory-trips.git
git push -u origin main
```

## Structure

- `server.js` - Express API, image upload (multer), JSON file storage
- `public/index.html` - the whole front end
- `uploads/` and `data/` - created automatically, ignored by git

## Before going public

This starter has no login, so anyone can post. Add authentication, moderation and a real database (SQLite/PostgreSQL) plus cloud storage (S3, Cloudinary) before putting it online.
end
