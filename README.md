index.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <meta name="apple-mobile-web-app-capable" content="yes" />
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
  <title>Scripture Reader</title>
  <style>
    :root {
      --bg: #111113;
      --surface: #1a1a1e;
      --surface-border: #2c2c34;
      --text: #eaeaea;
      --text-muted: #8e8e93;
      --accent: #5e9eff;
      --verse-num: #6a6a78;
      --font-body: "Charter", "Georgia", "Iowan Old Style", serif;
      --font-ui: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
    }

    [data-theme="light"] {
      --bg: #fafafa;
      --surface: #f0f0f2;
      --surface-border: #e0e0e6;
      --text: #1a1a1a;
      --text-muted: #737373;
      --accent: #0056b3;
      --verse-num: #a0a0a8;
    }

    * {
      box-sizing: border-box;
      margin: 0;
      padding: 0;
      -webkit-tap-highlight-color: transparent;
    }

    body {
      background: var(--bg);
      color: var(--text);
      font-family: var(--font-ui);
      display: flex;
      flex-direction: column;
      height: 100vh;
      overflow: hidden;
      user-select: text;
    }

    header {
      background: var(--surface);
      border-bottom: 1px solid var(--surface-border);
      padding: 10px 14px;
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 8px;
      z-index: 10;
    }

    .nav-selectors {
      display: flex;
      gap: 6px;
      align-items: center;
    }

    select, input, button {
      background: var(--bg);
      color: var(--text);
      border: 1px solid var(--surface-border);
      border-radius: 8px;
      padding: 6px 8px;
      font-size: 14px;
      font-family: var(--font-ui);
      outline: none;
    }

    select {
      font-weight: 600;
    }

    button {
      cursor: pointer;
      display: flex;
      align-items: center;
      justify-content: center;
      min-width: 36px;
      height: 34px;
    }

    #reader-viewport {
      flex: 1;
      position: relative;
      overflow-x: hidden;
      overflow-y: auto;
      touch-action: pan-y;
    }

    .reader-pane {
      max-width: 680px;
      margin: 0 auto;
      padding: 30px 20px 80px 20px;
      font-family: var(--font-body);
      font-size: 19px;
      line-height: 1.8;
      transition: transform 0.2s ease, opacity 0.2s ease;
    }

    .page-turn-left {
      transform: translateX(-40px);
      opacity: 0;
    }

    .page-turn-right {
      transform: translateX(40px);
      opacity: 0;
    }

    .chapter-heading {
      font-family: var(--font-ui);
      font-size: 22px;
      font-weight: 700;
      text-align: center;
      margin-bottom: 24px;
      color: var(--accent);
      letter-spacing: -0.5px;
    }

    .verse {
      margin-bottom: 10px;
      text-align: justify;
      hyphens: auto;
    }

    .verse-num {
      font-family: var(--font-ui);
      font-size: 11px;
      font-weight: bold;
      color: var(--verse-num);
      vertical-align: super;
      margin-right: 5px;
      user-select: none;
    }

    #search-results-overlay {
      display: none;
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      background: var(--bg);
      overflow-y: auto;
      padding: 16px;
      z-index: 20;
    }

    .search-card {
      background: var(--surface);
      border: 1px solid var(--surface-border);
      border-radius: 8px;
      padding: 12px 14px;
      margin-bottom: 10px;
      cursor: pointer;
    }

    .search-card-ref {
      font-size: 13px;
      font-weight: bold;
      color: var(--accent);
      margin-bottom: 4px;
    }

    footer {
      background: var(--surface);
      border-top: 1px solid var(--surface-border);
      padding: 8px 16px;
      display: flex;
      justify-content: space-between;
      align-items: center;
      font-size: 12px;
      color: var(--text-muted);
    }

    .loading-pulse {
      animation: pulse 1.2s infinite;
    }

    @keyframes pulse {
      0% { opacity: 0.4; }
      50% { opacity: 1; }
      100% { opacity: 0.4; }
    }
  </style>
</head>
<body data-theme="dark">

  <header>
    <div class="nav-selectors">
      <select id="trans-select" title="Translation">
        <option value="BSB" selected>BSB</option>
        <option value="WEB">WEB</option>
        <option value="KJV">KJV</option>
      </select>

      <select id="book-select"></select>
      <select id="chapter-select"></select>
    </div>

    <div style="display: flex; gap: 6px;">
      <input type="text" id="search-box" placeholder="Search..." style="width: 90px;" />
      <button id="theme-toggle" aria-label="Toggle Theme">☀️</button>
    </div>
  </header>

  <div id="reader-viewport">
    <div id="search-results-overlay"></div>
    <div id="reader-pane" class="reader-pane">
      <div id="chapter-header" class="chapter-heading">Loading...</div>
      <div id="chapter-text"></div>
    </div>
  </div>

  <footer>
    <span id="swipe-hint">Swipe left/right to turn chapter</span>
    <span id="cache-status">Ready</span>
  </footer>

  <script>
    const BOOKS = [
      { name: "Genesis", id: 1, chaps: 50 }, { name: "Exodus", id: 2, chaps: 40 },
      { name: "Leviticus", id: 3, chaps: 27 }, { name: "Numbers", id: 4, chaps: 36 },
      { name: "Deuteronomy", id: 5, chaps: 34 }, { name: "Joshua", id: 6, chaps: 24 },
      { name: "Judges", id: 7, chaps: 21 }, { name: "Ruth", id: 8, chaps: 4 },
      { name: "1 Samuel", id: 9, chaps: 31 }, { name: "2 Samuel", id: 10, chaps: 24 },
      { name: "1 Kings", id: 11, chaps: 22 }, { name: "2 Kings", id: 12, chaps: 25 },
      { name: "1 Chronicles", id: 13, chaps: 29 }, { name: "2 Chronicles", id: 14, chaps: 36 },
      { name: "Ezra", id: 15, chaps: 10 }, { name: "Nehemiah", id: 16, chaps: 13 },
      { name: "Esther", id: 17, chaps: 10 }, { name: "Job", id: 18, chaps: 42 },
      { name: "Psalms", id: 19, chaps: 150 }, { name: "Proverbs", id: 20, chaps: 31 },
      { name: "Ecclesiastes", id: 21, chaps: 12 }, { name: "Song of Solomon", id: 22, chaps: 8 },
      { name: "Isaiah", id: 23, chaps: 66 }, { name: "Jeremiah", id: 24, chaps: 52 },
      { name: "Lamentations", id: 25, chaps: 5 }, { name: "Ezekiel", id: 26, chaps: 48 },
      { name: "Daniel", id: 27, chaps: 12 }, { name: "Hosea", id: 28, chaps: 14 },
      { name: "Joel", id: 29, chaps: 3 }, { name: "Amos", id: 30, chaps: 9 },
      { name: "Obadiah", id: 31, chaps: 1 }, { name: "Jonah", id: 32, chaps: 4 },
      { name: "Micah", id: 33, chaps: 7 }, { name: "Nahum", id: 34, chaps: 3 },
      { name: "Habakkuk", id: 35, chaps: 3 }, { name: "Zephaniah", id: 36, chaps: 3 },
      { name: "Haggai", id: 37, chaps: 2 }, { name: "Zechariah", id: 38, chaps: 14 },
      { name: "Malachi", id: 39, chaps: 4 }, { name: "Matthew", id: 40, chaps: 28 },
      { name: "Mark", id: 41, chaps: 16 }, { name: "Luke", id: 42, chaps: 24 },
      { name: "John", id: 43, chaps: 21 }, { name: "Acts", id: 44, chaps: 28 },
      { name: "Romans", id: 45, chaps: 16 }, { name: "1 Corinthians", id: 46, chaps: 16 },
      { name: "2 Corinthians", id: 47, chaps: 13 }, { name: "Galatians", id: 48, chaps: 6 },
      { name: "Ephesians", id: 49, chaps: 6 }, { name: "Philippians", id: 50, chaps: 4 },
      { name: "Colossians", id: 51, chaps: 4 }, { name: "1 Thessalonians", id: 52, chaps: 5 },
      { name: "2 Thessalonians", id: 53, chaps: 3 }, { name: "1 Timothy", id: 54, chaps: 6 },
      { name: "2 Timothy", id: 55, chaps: 4 }, { name: "Titus", id: 56, chaps: 3 },
      { name: "Philemon", id: 57, chaps: 1 }, { name: "Hebrews", id: 58, chaps: 13 },
      { name: "James", id: 59, chaps: 5 }, { name: "1 Peter", id: 60, chaps: 5 },
      { name: "2 Peter", id: 61, chaps: 3 }, { name: "1 John", id: 62, chaps: 5 },
      { name: "2 John", id: 63, chaps: 1 }, { name: "3 John", id: 64, chaps: 1 },
      { name: "Jude", id: 65, chaps: 1 }, { name: "Revelation", id: 66, chaps: 22 }
    ];

    let currentTrans = "BSB";
    let currentBookIndex = 42;
    let currentChapter = 1;

    const bookSelect = document.getElementById("book-select");
    const chapterSelect = document.getElementById("chapter-select");
    const transSelect = document.getElementById("trans-select");
    const chapterHeader = document.getElementById("chapter-header");
    const chapterText = document.getElementById("chapter-text");
    const readerPane = document.getElementById("reader-pane");
    const readerViewport = document.getElementById("reader-viewport");
    const searchBox = document.getElementById("search-box");
    const searchOverlay = document.getElementById("search-results-overlay");
    const cacheStatus = document.getElementById("cache-status");
    const themeToggle = document.getElementById("theme-toggle");

    BOOKS.forEach((b, idx) => {
      const opt = document.createElement("option");
      opt.value = idx;
      opt.textContent = b.name;
      bookSelect.appendChild(opt);
    });

    function updateChapterDropdown() {
      chapterSelect.innerHTML = "";
      const totalChaps = BOOKS[currentBookIndex].chaps;
      for (let i = 1; i <= totalChaps; i++) {
        const opt = document.createElement("option");
        opt.value = i;
        opt.textContent = `${i}`;
        chapterSelect.appendChild(opt);
      }
      chapterSelect.value = currentChapter;
    }

    async function fetchChapterData(trans, bookId, chap) {
      const cacheKey = `bible_${trans}_${bookId}_${chap}`;
      const localCached = localStorage.getItem(cacheKey);
      if (localCached) {
        return JSON.parse(localCached);
      }

      const url = `https://bolls.life/get-chapter/${trans}/${bookId}/${chap}/`;
      const res = await fetch(url);
      if (!res.ok) throw new Error("Network error");
      const data = await res.json();

      try {
        localStorage.setItem(cacheKey, JSON.stringify(data));
      } catch (e) {
        localStorage.clear();
      }
      return data;
    }

    async function loadChapter(direction = null) {
      const book = BOOKS[currentBookIndex];
      bookSelect.value = currentBookIndex;
      updateChapterDropdown();

      chapterHeader.textContent = `${book.name} ${currentChapter} (${currentTrans})`;
      
      if (direction === "next") {
        readerPane.classList.add("page-turn-left");
      } else if (direction === "prev") {
        readerPane.classList.add("page-turn-right");
      }

      cacheStatus.textContent = "Loading...";
      cacheStatus.classList.add("loading-pulse");

      try {
        const verses = await fetchChapterData(currentTrans, book.id, currentChapter);

        setTimeout(() => {
          chapterText.innerHTML = verses.map(v => `
            <div class="verse">
              <span class="verse-num">${v.verse}</span>${v.text}
            </div>
          `).join("");

          readerViewport.scrollTop = 0;
          readerPane.classList.remove("page-turn-left", "page-turn-right");
          cacheStatus.textContent = "Saved Offline";
          cacheStatus.classList.remove("loading-pulse");
        }, direction ? 120 : 0);

        prefetchAdjacent();
      } catch (err) {
        chapterText.innerHTML = `<p style="color: var(--text-muted); padding-top: 20px;">Could not load chapter. Please check internet connection.</p>`;
        cacheStatus.textContent = "Offline error";
        cacheStatus.classList.remove("loading-pulse");
      }
    }

    function prefetchAdjacent() {
      const book = BOOKS[currentBookIndex];
      if (currentChapter < book.chaps) {
        fetchChapterData(currentTrans, book.id, currentChapter + 1).catch(() => {});
      }
      if (currentChapter > 1) {
        fetchChapterData(currentTrans, book.id, currentChapter - 1).catch(() => {});
      }
    }

    function nextPage() {
      const book = BOOKS[currentBookIndex];
      if (currentChapter < book.chaps) {
        currentChapter++;
        loadChapter("next");
      } else if (currentBookIndex < BOOKS.length - 1) {
        currentBookIndex++;
        currentChapter = 1;
        loadChapter("next");
      }
    }

    function prevPage() {
      if (currentChapter > 1) {
        currentChapter--;
        loadChapter("prev");
      } else if (currentBookIndex > 0) {
        currentBookIndex--;
        currentChapter = BOOKS[currentBookIndex].chaps;
        loadChapter("prev");
      }
    }

    let touchStartX = 0;
    let touchStartY = 0;

    readerViewport.addEventListener("touchstart", (e) => {
      touchStartX = e.changedTouches[0].screenX;
      touchStartY = e.changedTouches[0].screenY;
    }, { passive: true });

    readerViewport.addEventListener("touchend", (e) => {
      const deltaX = e.changedTouches[0].screenX - touchStartX;
      const deltaY = e.changedTouches[0].screenY - touchStartY;

      if (Math.abs(deltaX) > 60 && Math.abs(deltaY) < 45) {
        if (deltaX < 0) {
          nextPage();
        } else {
          prevPage();
        }
      }
    }, { passive: true });

    let searchTimeout = null;
    searchBox.addEventListener("input", (e) => {
      clearTimeout(searchTimeout);
      const query = e.target.value.trim();

      if (!query) {
        searchOverlay.style.display = "none";
        return;
      }

      searchTimeout = setTimeout(async () => {
        searchOverlay.style.display = "block";
        searchOverlay.innerHTML = `<p style="color:var(--text-muted); font-size:14px;">Searching ${currentTrans}...</p>`;

        try {
          const res = await fetch(`https://bolls.life/v2/find/?search=${encodeURIComponent(query)}&translation=${currentTrans}&match_whole=true`);
          const data = await res.json();
          const results = data.results || [];

          if (results.length === 0) {
            searchOverlay.innerHTML = `<p style="color:var(--text-muted); font-size:14px;">No results found for "${query}".</p>`;
            return;
          }

          searchOverlay.innerHTML = results.slice(0, 40).map(r => {
            const b = BOOKS.find(item => item.id === r.book);
            const bookName = b ? b.name : `Book ${r.book}`;
            return `
              <div class="search-card" onclick="goToVerse(${r.book - 1}, ${r.chapter})">
                <div class="search-card-ref">${bookName} ${r.chapter}:${r.verse} (${currentTrans})</div>
                <div style="font-size: 15px; font-family: var(--font-body);">${r.text}</div>
              </div>
            `;
          }).join("");
        } catch (e) {
          searchOverlay.innerHTML = `<p style="color:var(--text-muted);">Search request failed.</p>`;
        }
      }, 350);
    });

    window.goToVerse = function(bIdx, chap) {
      currentBookIndex = bIdx;
      currentChapter = chap;
      searchOverlay.style.display = "none";
      searchBox.value = "";
      loadChapter();
    };

    bookSelect.addEventListener("change", (e) => {
      currentBookIndex = parseInt(e.target.value, 10);
      currentChapter = 1;
      loadChapter();
    });

    chapterSelect.addEventListener("change", (e) => {
      currentChapter = parseInt(e.target.value, 10);
      loadChapter();
    });

    transSelect.addEventListener("change", (e) => {
      currentTrans = e.target.value;
      loadChapter();
    });

    themeToggle.addEventListener("click", () => {
      const isDark = document.body.getAttribute("data-theme") === "dark";
      const nextTheme = isDark ? "light" : "dark";
      document.body.setAttribute("data-theme", nextTheme);
      themeToggle.textContent = nextTheme === "dark" ? "☀️" : "🌙";
    });

    updateChapterDropdown();
    loadChapter();
  </script>
</body>
</html>
