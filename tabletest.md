<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Google Sheet Table</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }

    body {
      font-family: Arial, sans-serif;
      background: #f4f6f9;
      padding: 30px;
      color: #333;
    }

    h1 {
      margin-bottom: 20px;
      font-size: 1.6rem;
      color: #2c3e50;
    }

    .controls {
      display: flex;
      flex-wrap: wrap;
      gap: 12px;
      margin-bottom: 16px;
      align-items: center;
    }

    .controls input[type="text"] {
      padding: 8px 12px;
      border: 1px solid #ccc;
      border-radius: 6px;
      font-size: 0.95rem;
      flex: 1;
      min-width: 200px;
    }

    .controls select {
      padding: 8px 12px;
      border: 1px solid #ccc;
      border-radius: 6px;
      font-size: 0.95rem;
      background: white;
    }

    .controls button {
      padding: 8px 16px;
      border: none;
      border-radius: 6px;
      background: #3498db;
      color: white;
      font-size: 0.95rem;
      cursor: pointer;
    }

    .controls button:hover { background: #2980b9; }

    #row-count {
      font-size: 0.9rem;
      color: #666;
      margin-bottom: 10px;
    }

    .table-wrapper {
      overflow-x: auto;
      border-radius: 8px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    }

    table {
      width: 100%;
      border-collapse: collapse;
      background: white;
      font-size: 0.92rem;
    }

    thead tr {
      background: #2c3e50;
      color: white;
    }

    thead th {
      padding: 12px 14px;
      text-align: left;
      cursor: pointer;
      user-select: none;
      white-space: nowrap;
    }

    thead th:hover { background: #34495e; }

    thead th .sort-icon {
      margin-left: 6px;
      font-size: 0.8rem;
      opacity: 0.6;
    }

    thead th.sort-asc .sort-icon::after { content: "▲"; }
    thead th.sort-desc .sort-icon::after { content: "▼"; }
    thead th:not(.sort-asc):not(.sort-desc) .sort-icon::after { content: "⇅"; }

    thead tr.filter-row th {
      background: #ecf0f1;
      padding: 6px 8px;
      cursor: default;
    }

    thead tr.filter-row th input {
      width: 100%;
      padding: 5px 8px;
      border: 1px solid #ccc;
      border-radius: 4px;
      font-size: 0.85rem;
    }

    tbody tr:nth-child(even) { background: #f8f9fa; }
    tbody tr:hover { background: #eaf4fb; }

    tbody td {
      padding: 10px 14px;
      border-bottom: 1px solid #e0e0e0;
      vertical-align: top;
    }

    .no-data {
      text-align: center;
      padding: 30px;
      color: #888;
      font-style: italic;
    }

    #status {
      margin-bottom: 14px;
      font-size: 0.9rem;
      color: #888;
    }
  </style>
</head>
<body>

  <h1>Google Sheet Data Table</h1>

  <div class="controls">
    <input type="text" id="global-search" placeholder="🔍 Search all columns..." oninput="applyFilters()" />
    <select id="rows-per-page" onchange="applyFilters()">
      <option value="10">10 rows</option>
      <option value="25" selected>25 rows</option>
      <option value="50">50 rows</option>
      <option value="100">100 rows</option>
      <option value="0">All rows</option>
    </select>
    <button onclick="clearAllFilters()">Clear Filters</button>
    <button onclick="exportCSV()">Export CSV</button>
  </div>

  <div id="status">Loading data...</div>
  <div id="row-count"></div>

  <div class="table-wrapper">
    <table id="data-table">
      <thead>
        <tr id="header-row"></tr>
        <tr class="filter-row" id="filter-row"></tr>
      </thead>
      <tbody id="table-body"></tbody>
    </table>
  </div>

  <div class="controls" style="margin-top: 14px; justify-content: center;">
    <button onclick="changePage(-1)">◀ Prev</button>
    <span id="page-info" style="padding: 8px 12px; font-size:0.9rem;"></span>
    <button onclick="changePage(1)">Next ▶</button>
  </div>

  <script>
    // ─────────────────────────────────────────────
    // CONFIGURATION
    // ─────────────────────────────────────────────
    const CSV_URL = "https://docs.google.com/spreadsheets/d/e/2PACX-1vRtyGEmEVnftuJGoHfJM-VaWdDE65gXUgG1dsWWzT9MzXJggi_qDz7i46H4R9LWWA/pub?gid=1027676349&single=true&output=csv";
    // ─────────────────────────────────────────────

    let allRows      = [];
    let headers      = [];
    let filteredRows = [];
    let sortCol      = -1;
    let sortDir      = 1;
    let currentPage  = 1;

    async function loadSheet() {
      try {
        const res = await fetch(CSV_URL);
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const text = await res.text();
        parseCSV(text);
        document.getElementById("status").textContent = "Data loaded successfully.";
      } catch (e) {
        document.getElementById("status").textContent = `Error loading sheet: ${e.message}. Make sure the sheet is public and the SHEET_ID is correct.`;
      }
    }

    function parseCSV(text) {
      const rows = [];
      let row = [], cell = "", inQuote = false;

      for (let i = 0; i < text.length; i++) {
        const ch = text[i], next = text[i + 1];
        if (inQuote) {
          if (ch === '"' && next === '"') { cell += '"'; i++; }
          else if (ch === '"') inQuote = false;
          else cell += ch;
        } else {
          if (ch === '"') inQuote = true;
          else if (ch === ',') { row.push(cell.trim()); cell = ""; }
          else if (ch === '\n' || (ch === '\r' && next === '\n')) {
            if (ch === '\r') i++;
            row.push(cell.trim()); rows.push(row); row = []; cell = "";
          } else cell += ch;
        }
      }
      if (cell || row.length) { row.push(cell.trim()); rows.push(row); }

      if (rows.length === 0) return;

      headers = rows[0];
      allRows = rows.slice(1).filter(r => r.some(c => c !== ""));
      buildHeaders();
      applyFilters();
    }

    function buildHeaders() {
      const headerRow = document.getElementById("header-row");
      const filterRow = document.getElementById("filter-row");
      headerRow.innerHTML = "";
      filterRow.innerHTML = "";

      headers.forEach((h, i) => {
        const th = document.createElement("th");
        th.innerHTML = `${h} <span class="sort-icon"></span>`;
        th.dataset.col = i;
        th.addEventListener("click", () => sortByCol(i, th));
        headerRow.appendChild(th);

        const ftd = document.createElement("th");
        const inp = document.createElement("input");
        inp.type = "text";
        inp.placeholder = `Filter ${h}…`;
        inp.dataset.col = i;
        inp.addEventListener("input", applyFilters);
        ftd.appendChild(inp);
        filterRow.appendChild(ftd);
      });
    }

    function sortByCol(col, th) {
      if (sortCol === col) sortDir *= -1;
      else { sortCol = col; sortDir = 1; }

      document.querySelectorAll("#header-row th").forEach(t => {
        t.classList.remove("sort-asc", "sort-desc");
      });
      th.classList.add(sortDir === 1 ? "sort-asc" : "sort-desc");

      applyFilters();
    }

    function applyFilters() {
      const globalSearch = document.getElementById("global-search").value.toLowerCase();
      const colFilters = [...document.querySelectorAll("#filter-row input")]
        .map(i => i.value.toLowerCase());

      filteredRows = allRows.filter(row => {
        if (globalSearch && !row.some(c => c.toLowerCase().includes(globalSearch))) return false;
        return colFilters.every((f, i) => !f || (row[i] || "").toLowerCase().includes(f));
      });

      if (sortCol >= 0) {
        filteredRows.sort((a, b) => {
          const av = a[sortCol] || "", bv = b[sortCol] || "";
          const an = parseFloat(av), bn = parseFloat(bv);
          if (!isNaN(an) && !isNaN(bn)) return (an - bn) * sortDir;
          return av.localeCompare(bv) * sortDir;
        });
      }

      currentPage = 1;
      renderPage();
    }

    function renderPage() {
      const rowsPerPage = parseInt(document.getElementById("rows-per-page").value);
      const total       = filteredRows.length;
      const pages       = rowsPerPage > 0 ? Math.ceil(total / rowsPerPage) : 1;
      currentPage       = Math.max(1, Math.min(currentPage, pages || 1));

      const start    = rowsPerPage > 0 ? (currentPage - 1) * rowsPerPage : 0;
      const end      = rowsPerPage > 0 ? start + rowsPerPage : total;
      const pageRows = filteredRows.slice(start, end);

      const tbody = document.getElementById("table-body");
      tbody.innerHTML = "";

      if (pageRows.length === 0) {
        tbody.innerHTML = `<tr><td colspan="${headers.length}" class="no-data">No matching records found.</td></tr>`;
      } else {
        pageRows.forEach(row => {
          const tr = document.createElement("tr");
          headers.forEach((_, i) => {
            const td = document.createElement("td");
            td.textContent = row[i] || "";
            tr.appendChild(td);
          });
          tbody.appendChild(tr);
        });
      }

      document.getElementById("row-count").textContent =
        `Showing ${pageRows.length ? start + 1 : 0}–${Math.min(end, total)} of ${total} row(s)` +
        (total < allRows.length ? ` (filtered from ${allRows.length} total)` : "");

      document.getElementById("page-info").textContent =
        pages > 1 ? `Page ${currentPage} of ${pages}` : "";
    }

    function changePage(dir) {
      currentPage += dir;
      renderPage();
    }

    function clearAllFilters() {
      document.getElementById("global-search").value = "";
      document.querySelectorAll("#filter-row input").forEach(i => i.value = "");
      sortCol = -1; sortDir = 1;
      document.querySelectorAll("#header-row th").forEach(t =>
        t.classList.remove("sort-asc", "sort-desc"));
      applyFilters();
    }

    function exportCSV() {
      const csvRows = [headers, ...filteredRows]
        .map(r => r.map(c => `"${(c || "").replace(/"/g, '""')}"`).join(","))
        .join("\n");
      const a = document.createElement("a");
      a.href = "data:text/csv;charset=utf-8," + encodeURIComponent(csvRows);
      a.download = "export.csv";
      a.click();
    }

    loadSheet();
  </script>
</body>
</html>
