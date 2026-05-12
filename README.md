<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Turnier Info</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.3.2/papaparse.min.js"></script>
    <style>
        .active-tab { border-bottom: 4px solid #3b82f6; color: #3b82f6; }
        /* Verhindert das "Blitzen" beim Laden */
        body { background-color: #111827; } 
    </style>
</head>
<body class="bg-gray-900 min-h-screen pb-10 text-gray-100">

<!-- Login Maske -->
<div id="login-screen" class="fixed inset-0 bg-gray-900 flex justify-center z-50 p-4 items-start pt-20">
    <div class="bg-gray-800 p-8 rounded-xl shadow-2xl w-full max-w-sm text-center border border-gray-700">
        <h2 class="text-xl font-bold mb-4 text-white">Turnier Login</h2>
        <!-- onkeydown hinzugefügt -->
        <input type="password" id="pw-input" 
               onkeydown="if(event.key === 'Enter') checkPassword()"
               class="w-full border-2 border-gray-700 p-3 rounded-lg mb-4 bg-gray-900 text-white focus:border-blue-500 outline-none" 
               placeholder="Passwort">
        <button onclick="checkPassword()" 
                class="w-full bg-blue-600 hover:bg-blue-700 text-white py-3 rounded-lg font-bold transition-colors">
            Anmelden
        </button>
        <p id="error-msg" class="text-red-400 text-sm mt-3 hidden">Falsches Passwort!</p>
    </div>
</div>

    <!-- Hauptinhalt -->
    <div id="main-content" class="hidden max-w-md mx-auto p-2 sm:p-4">
        
        <!-- Tab Navigation -->
        <div class="flex mb-4 bg-gray-800 rounded-xl shadow-sm overflow-hidden border border-gray-700">
            <button onclick="switchTab('pairings')" id="tab-pairings" class="flex-1 py-3 font-bold uppercase text-xs tracking-wider active-tab">Pairings</button>
            <button onclick="switchTab('standings')" id="tab-standings" class="flex-1 py-3 font-bold uppercase text-xs tracking-wider text-gray-400">Standings</button>
        </div>

        <!-- Steuerung -->
        <div class="sticky top-2 bg-gray-800/95 backdrop-blur shadow-lg rounded-xl p-3 space-y-3 z-40 border border-gray-700">
            <div class="flex gap-2">
                <div id="round-select-container" class="w-28 shrink-0">
                    <select id="round-select" onchange="loadRound(this.value)" 
                            class="w-full border border-gray-600 py-2 px-1 rounded-lg bg-gray-900 text-white font-bold outline-none text-sm text-center">
                    </select>
                </div>
                <div class="flex-[2] relative">
                    <input type="text" id="search-input" onkeyup="filterTable()" placeholder="Namen suchen..." 
                           class="w-full p-2 pl-8 border border-gray-600 rounded-lg bg-gray-900 text-white outline-none focus:ring-2 focus:ring-blue-500 text-sm">
                    <span class="absolute left-2.5 top-2.5 opacity-50">🔍</span>
                </div>
            </div>
        </div>

        <!-- PAIRINGS TABELLE -->
        <div id="view-pairings" class="mt-4 bg-gray-800 rounded-xl shadow-sm border border-gray-700 overflow-hidden">
            <table class="w-full text-sm">
                <thead class="bg-black/50 text-gray-400 uppercase text-[10px] tracking-[0.2em]">
                    <tr>
                        <th class="p-3 text-center w-12 border-r border-gray-700">Tisch</th>
                        <th class="p-3 text-center">Pairing</th>
                    </tr>
                </thead>
                <tbody class="divide-y divide-gray-700" id="table-body"></tbody>
            </table>
        </div>

        <!-- STANDINGS TABELLE -->
        <div id="view-standings" class="mt-4 bg-gray-800 rounded-xl shadow-sm border border-gray-700 overflow-hidden hidden">
            <table class="w-full text-sm">
                <thead class="bg-black/50 text-gray-400 uppercase text-[10px] tracking-[0.2em]">
                    <tr>
                        <th class="p-3 text-center w-12 border-r border-gray-700">Rang</th>
                        <th class="p-3 text-center">Spieler</th> 
                        <th class="p-3 text-center w-12">Pkt.</th>
                        <th class="p-3 text-center w-12">TB</th>
                    </tr>
                </thead>
                <tbody class="divide-y divide-gray-700" id="standings-body"></tbody>
            </table>
        </div>

        <div id="no-results" class="p-10 text-center text-gray-500 hidden italic">Keine Einträge gefunden.</div>
        <div id="loading-msg" class="p-10 text-center text-gray-500 italic">Lade Daten...</div>
    </div>

    <script>
        const CORRECT_PW = "login"; 
        let currentTab = 'pairings';

        window.onload = () => {
            if(sessionStorage.getItem('auth') === 'true') showContent();
        };

        function checkPassword() {
            if(document.getElementById('pw-input').value === CORRECT_PW) {
                sessionStorage.setItem('auth', 'true');
                showContent();
            } else {
                document.getElementById('error-msg').classList.remove('hidden');
            }
        }

        async function showContent() {
            document.getElementById('login-screen').classList.add('hidden');
            document.getElementById('main-content').classList.remove('hidden');
            await checkExistingRounds();
            loadStandings();
        }

        function switchTab(tab) {
            currentTab = tab;
            document.getElementById('view-pairings').classList.toggle('hidden', tab !== 'pairings');
            document.getElementById('view-standings').classList.toggle('hidden', tab !== 'standings');
            document.getElementById('round-select-container').classList.toggle('invisible', tab !== 'pairings');
            document.getElementById('tab-pairings').className = tab === 'pairings' ? "flex-1 py-3 font-bold uppercase text-xs tracking-wider active-tab" : "flex-1 py-3 font-bold uppercase text-xs tracking-wider text-gray-400";
            document.getElementById('tab-standings').className = tab === 'standings' ? "flex-1 py-3 font-bold uppercase text-xs tracking-wider active-tab" : "flex-1 py-3 font-bold uppercase text-xs tracking-wider text-gray-400";
            filterTable();
        }

        async function checkExistingRounds() {
            const select = document.getElementById('round-select');
            select.innerHTML = '';
            let foundRounds = [];
            for (let i = 1; i <= 20; i++) {
                try {
                    const response = await fetch(`runde${i}.csv`, { method: 'HEAD' });
                    if (response.ok) foundRounds.push({ id: i, file: `runde${i}.csv` });
                } catch (e) {}
            }
            foundRounds.sort((a, b) => b.id - a.id);
            if (foundRounds.length > 0) {
                foundRounds.forEach((round, index) => {
                    const opt = document.createElement('option');
                    opt.value = round.file;
                    opt.textContent = `Runde ${round.id}`;
                    if (index === 0) {
                        opt.classList.add('font-bold', 'text-blue-400');
                    } else {
                        opt.style.color = "#ef4444";
                        opt.style.textDecoration = "line-through";
                    }
                    select.appendChild(opt);
                });
                document.getElementById('loading-msg').classList.add('hidden');
                loadRound(select.value);
            }
        }

        function loadRound(fileName) {
            Papa.parse(fileName, { download: true, complete: (results) => renderPairings(results.data) });
        }

        function loadStandings() {
            Papa.parse("standings.csv", { download: true, complete: (results) => renderStandings(results.data) });
        }

        const processName = (name) => {
            if (!name) return "";
            let clean = name.replace(/\s*\([^)]*\)/g, "").trim();
            if (clean.includes("BYE")) return "— BYE —";
            if (clean.includes(",")) {
                const parts = clean.split(",");
                return `${parts[1].trim()} ${parts[0].trim()}`;
            }
            return clean;
        };

        function renderPairings(data) {
            const tbody = document.getElementById('table-body');
            tbody.innerHTML = '';
            for (let i = 1; i < data.length; i++) {
                const row = data[i];
                if(!row[0]) continue;
                const tr = document.createElement('tr');
                tr.className = "border-b border-gray-700 odd:bg-gray-800 even:bg-gray-700/30";
                tr.innerHTML = `
                    <td class="p-3 text-center font-mono font-bold text-blue-400 bg-black/20 w-12 border-r border-gray-700">${row[0]}</td>
                    <td class="p-3">
                        <div class="flex items-center justify-center gap-4 w-full">
                            <div class="flex-1 text-right font-medium text-gray-100">${processName(row[4])}</div>
                            <div class="text-[10px] font-bold text-gray-500 uppercase tracking-widest">vs</div>
                            <div class="flex-1 text-left font-medium text-gray-100">${processName(row[6])}</div>
                        </div>
                    </td>`;
                tbody.appendChild(tr);
            }
        }

        function renderStandings(data) {
            const tbody = document.getElementById('standings-body');
            tbody.innerHTML = '';
            for (let i = 1; i < data.length; i++) {
                const row = data[i];
                if(!row[0]) continue;
                const tr = document.createElement('tr');
                tr.className = "border-b border-gray-700 odd:bg-gray-800 even:bg-gray-700/30";
                tr.innerHTML = `
                    <td class="p-3 text-center font-mono font-bold text-blue-400 bg-black/20 w-12 border-r border-gray-700">${row[0]}</td>
                    <td class="p-3 font-medium text-gray-100 text-center">${processName(row[4])}</td>
                    <td class="p-3 text-center font-bold text-gray-300">${row[6] || '0'}</td>
                    <td class="p-3 text-center text-[10px] text-gray-500 font-mono">${row[7] || '0'}</td>
                `;
                tbody.appendChild(tr);
            }
        }

        function filterTable() {
            const input = document.getElementById('search-input').value.toLowerCase();
            const targetId = currentTab === 'pairings' ? '#table-body tr' : '#standings-body tr';
            const rows = document.querySelectorAll(targetId);
            let hasResults = false;
            rows.forEach(row => {
                const visible = row.innerText.toLowerCase().includes(input);
                row.style.display = visible ? "" : "none";
                if(visible) hasResults = true;
            });
            document.getElementById('no-results').classList.toggle('hidden', hasResults);
        }
    </script>
</body>
</html>
