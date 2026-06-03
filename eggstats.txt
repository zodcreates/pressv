// ==UserScript==
// @name         Shell Shockers EggTools + EggStats (FINAL KDR + LIVE)
// @namespace    [https://shellshock.io/](https://shellshock.io/)
// @version      2.7
// @author       Lanrad | Co-Auth: Midnight & Chat-GPT
// @description  EggStats + Live Leaderboard (Auto) + Discord Embed + Castle Teams Creator + New Stats
// @match        https://shellshock.io/*
// @grant        none
// ==/UserScript==

// LMAO this took too long to make, even w ai. Anyways props out to Claude, Gemini, and Chatgpt for this. They made the UI and the key functions. Most of the code for how functions act was made by Lanrad.
(function () {
  'use strict';

  /***********************
   * MATCH STATS
   ***********************/
  const matchStats = {};
  function ensurePlayer(name) {
    if (!matchStats[name]) matchStats[name] = { kills: 0, deaths: 0 };
  }

  /***********************
   * MATCH TIMER + FINAL LOCK
   ***********************/
  let matchStartTime = null;
  let finalEmbedLocked = false;

  function getMatchDuration() {
    if (!matchStartTime) return "00:00";
    const s = Math.floor((Date.now() - matchStartTime) / 1000);
    return `${String(Math.floor(s / 60)).padStart(2, "0")}:${String(s % 60).padStart(2, "0")}`;
  }

  /***********************
   * DISCORD EMBED STATE
   ***********************/
  let liveEmbedMessageId = localStorage.getItem("eggEmbedMsgId") || null;
  let liveEmbedInterval = null;

  // track last successful send time so we can auto‑recover
  let lastEmbedOkAt = Date.now();
  let consecutiveErrors = 0;

  function buildDiscordEmbed() {
    // sort by kills desc, then deaths asc, then KDR desc
    const players = Object.entries(matchStats)
      .map(([name, s]) => {
        const kdr = s.deaths === 0 ? s.kills : s.kills / s.deaths;
        return { name, ...s, kdr };
      })
      .sort((a, b) => {
        if (b.kills !== a.kills) return b.kills - a.kills;        // more kills first
        if (a.deaths !== b.deaths) return a.deaths - b.deaths;    // fewer deaths first
        if (b.kdr !== a.kdr) return b.kdr - a.kdr;                // higher KDR as final tiebreak
        return a.name.localeCompare(b.name);
      });

    const medals = ["🥇", "🥈", "🥉"];

    return {
      embeds: [{
        title: finalEmbedLocked ? "🏁 Final Match Results" : "🏆 Live Shell Shockers Leaderboard",
        description: players.slice(0, 10).map((p, i) =>
          `${medals[i] || "▫️"} **${p.name}** — ${p.kills}K / ${p.deaths}D (KDR ${p.kdr.toFixed(2)})`
        ).join("\n") || "No stats yet",
        color: finalEmbedLocked ? 0x2ecc71 : 0xf1c40f,
        footer: { text: `EggStats • Duration ${getMatchDuration()}` },
        timestamp: new Date().toISOString()
      }]
    };
  }

  function safePatch(webhookURL, body) {
    // PATCH wrapper to auto‑recover from errors and expired webhooks
    return fetch(webhookURL, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body)
    }).then(r => {
      if (!r.ok) {
        consecutiveErrors++;
        // invalid / deleted webhook or forbidden
        if (r.status === 404 || r.status === 401 || r.status === 403) {
          liveEmbedMessageId = null;
          localStorage.removeItem("eggEmbedMsgId");
        }
        // too many errors -> stop this interval; will be recreated
        if (consecutiveErrors >= 10) {
          clearInterval(liveEmbedInterval);
          liveEmbedInterval = null;
        }
      } else {
        consecutiveErrors = 0;
        lastEmbedOkAt = Date.now();
      }
      return r;
    }).catch(() => {
      consecutiveErrors++;
      if (consecutiveErrors >= 10) {
        clearInterval(liveEmbedInterval);
        liveEmbedInterval = null;
      }
    });
  }

  function safePost(webhookURL, body) {
    // POST wrapper that stores message id and resets error counters
    return fetch(webhookURL + "?wait=true", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body)
    })
      .then(r => r.json())
      .then(d => {
        if (d && d.id) {
          liveEmbedMessageId = d.id;
          localStorage.setItem("eggEmbedMsgId", liveEmbedMessageId);
          lastEmbedOkAt = Date.now();
          consecutiveErrors = 0;
        }
        return d;
      })
      .catch(() => {
        consecutiveErrors++;
      });
  }

  function startEmbedUpdates(webhookURL) {
    if (liveEmbedInterval || finalEmbedLocked) return;

    liveEmbedInterval = setInterval(() => {
      if (finalEmbedLocked) return;

      const urlBase = localStorage.getItem("eggWebhook");
      if (!urlBase) return;

      // if we lost the message id (deleted message / invalid webhook), recreate
      if (!liveEmbedMessageId) {
        safePost(urlBase, buildDiscordEmbed());
        return;
      }

      // if we haven't had a successful update in 15 minutes, drop id and recreate
      if (Date.now() - lastEmbedOkAt > 15 * 60 * 1000) {
        liveEmbedMessageId = null;
        localStorage.removeItem("eggEmbedMsgId");
        safePost(urlBase, buildDiscordEmbed());
        return;
      }

      safePatch(`${urlBase}/messages/${liveEmbedMessageId}`, buildDiscordEmbed());
    }, 10000);
  }

  /***********************
   * UI PANEL
   ***********************/
  const panel = document.createElement("div");
  panel.id = "eggToolsPanel";
  panel.style.cssText = `
    position: fixed; top: 15px; left: 50%; transform: translateX(-50%);
    background: rgba(20, 20, 20, 0.95); color: #eee; padding: 15px;
    border-radius: 15px; font-family: 'Segoe UI', Tahoma, sans-serif;
    z-index: 999999; width: 280px; border: 1px solid #444;
    box-shadow: 0 8px 32px rgba(0,0,0,0.5); backdrop-filter: blur(4px);
  `;

  panel.innerHTML = `
    <style>
      #eggToolsPanel button {
        width: 100%; margin: 4px 0; padding: 8px; border: none; border-radius: 6px;
        background: #333; color: white; cursor: pointer; font-size: 12px; transition: 0.2s;
      }
      #eggToolsPanel button:hover { background: #444; }
      #eggToolsPanel .ml-btn { background: #2c3e50; font-weight: bold; border-left: 4px solid #3498db; }
      #eggToolsPanel .ml-btn:hover { background: #34495e; }
      #eggToolsPanel input {
        width: 100%; padding: 8px; margin-bottom: 8px; border-radius: 6px; border: 1px solid #444;
        background: #000; color: #1abc9c; font-size: 11px; box-sizing: border-box;
      }
      .section-label { font-size: 10px; color: #888; text-transform: uppercase; margin-top: 10px; margin-bottom: 4px; font-weight: bold; }
    </style>
    <div style="display:flex; justify-content: space-between; align-items: center; margin-bottom: 10px;">
      <b id="eggTab" style="cursor:pointer; font-size: 16px;">🥚 EggTools</b>
      <span id="eggStatus" style="padding:3px 10px; border-radius:20px; background:#a00; font-size:10px; font-weight:bold;">OFF</span>
    </div>
    <div id="eggContent" style="display:none;">
      <input id="webhookInput" placeholder="Discord Webhook URL">

      <div class="section-label">Presets</div>
      <button class="ml-btn" id="mlNA">🔗 Link to ML NA</button>
      <button class="ml-btn" id="mlEU">🔗 Link to ML EU</button>
      <button class="ml-btn" id="mlCS">🔗 Link to ML CS</button>

      <div class="section-label">Controls</div>
      <button id="enableStats" style="background: #1abc9c; color: #000; font-weight: bold;">Toggle EggStats ( \\ )</button>
      <button id="newStats">⭐ Start New Stats</button>
      <button id="finalLock">🔒 Lock Final Stats</button>
      <button id="createGame">🏰 Create Castle Teams</button>
    </div>
  `;

  document.body.appendChild(panel);

  panel.querySelector("#eggTab").onclick = () => {
    const content = panel.querySelector("#eggContent");
    content.style.display = content.style.display === "none" ? "block" : "none";
  };

  function updateStatusUI(on) {
    const eggStatus = document.getElementById("eggStatus");
    eggStatus.textContent = on ? "ON" : "OFF";
    eggStatus.style.background = on ? "#1abc9c" : "#a00";
  }

  /***********************
   * WEBHOOK FIX (AUTO RESET)
   ***********************/
  const webhookInput = panel.querySelector("#webhookInput");
  webhookInput.value = localStorage.getItem("eggWebhook") || "";

  const lastWebhook = localStorage.getItem("eggWebhookLast") || "";
  if (webhookInput.value && webhookInput.value !== lastWebhook) {
    localStorage.removeItem("eggEmbedMsgId");
    liveEmbedMessageId = null;
    localStorage.setItem("eggWebhookLast", webhookInput.value);
  }

  // ML Preset Webhook Helper
  const setWebhook = (url) => {
    webhookInput.value = url;
    localStorage.setItem("eggWebhook", url);
    localStorage.removeItem("eggEmbedMsgId");
    liveEmbedMessageId = null;
    alert("Webhook linked successfully!");
  };

  panel.querySelector("#mlNA").onclick = () => setWebhook("https://discord.com/api/webhooks/1477710254027313375/5CMAww-iNf0YsYGKjrxOmhyqLh11ipnvX0rjd3KBY5EpxRWn3iNURmQnhk_xE6av9MBZ");
  panel.querySelector("#mlEU").onclick = () => setWebhook("https://discord.com/api/webhooks/1477709090661597207/cn59II6BPn_xn5ui8LZMxR2Px8vG3Lv6994jF4LIEW9Pq4YGyLY8kVSFWxgKGUwlR2hI");
  panel.querySelector("#mlCS").onclick = () => setWebhook("https://discord.com/api/webhooks/1477710488271065101/t4OiefCGWJFsTF-YiDi3rYDyJYVPuvtNEK3WFuHWLCGVG--4e9Oc-emoupTbybtcuTs1");

  /***********************
   * START NEW STATS
   ***********************/
  function startNewStats() {
    const webhookURL = webhookInput.value.trim();
    if (!webhookURL) {
      alert("Enter webhook URL before starting new stats.");
      return;
    }

    // clear all current match stats
    for (const p in matchStats) {
      delete matchStats[p];
    }

    // reset match timer and unlock final flag
    matchStartTime = Date.now();
    finalEmbedLocked = false;

    // force a brand‑new Discord message for this fresh match
    liveEmbedMessageId = null;
    localStorage.removeItem("eggEmbedMsgId");
    localStorage.setItem("eggWebhook", webhookURL);

    // create new embed and restart updates if EggStats is enabled
    safePost(webhookURL, buildDiscordEmbed()).then(() => {
      if (eggStatsEnabled && liveEmbedMessageId) {
        clearInterval(liveEmbedInterval);
        liveEmbedInterval = null;
        startEmbedUpdates(webhookURL);
      }
    });

    // update leaderboard if visible
    if (leaderboardVisible) renderLeaderboard();
  }

  /***********************
   * FINAL LOCK BUTTON
   ***********************/
  panel.querySelector("#finalLock").onclick = () => {
    if (finalEmbedLocked) return alert("Final stats already locked.");
    finalEmbedLocked = true;
    clearInterval(liveEmbedInterval);
    liveEmbedInterval = null;

    const webhookURL = localStorage.getItem("eggWebhook");
    if (!webhookURL || !liveEmbedMessageId) return;

    safePatch(`${webhookURL}/messages/${liveEmbedMessageId}`, buildDiscordEmbed());

    alert("Final stats locked.");
  };

  /***********************
   * CREATE CASTLE + TEAMS GAME
   ***********************/
  panel.querySelector("#createGame").onclick = () => {
    document.getElementById("joinPrivateGamePopup").style.display = "";

    const mapBtn = document.querySelector("#mapRight");
    const mapName = document.querySelector("#mapText");

    const mapLoop = setInterval(() => {
      if (mapName?.innerText === "Castle") clearInterval(mapLoop);
      else mapBtn?.click();
    }, 50);

    setTimeout(() => {
      document.querySelector("#createPrivateGame > div > div > div > button")?.click();
      setTimeout(() => {
        [...document.querySelectorAll("#createPrivateGame ul li")]
          .find(li => li.innerText.includes("Teams"))?.click();
      }, 300);
    }, 3000);

    setTimeout(() => {
      document.querySelector("#createPrivateGame > div > div > div > :nth-last-child(1)")?.click();
    }, 4500);
  };

  /***********************
   * EGGSTATS CORE
   ***********************/
  let eggStatsEnabled = false;
  let observer = null;

  panel.querySelector("#enableStats").onclick = toggleEggStats;
  panel.querySelector("#newStats").onclick = startNewStats;

  function toggleEggStats() {
    if (eggStatsEnabled) {
      // turn OFF
      observer?.disconnect();
      observer = null;
      clearInterval(liveEmbedInterval);
      liveEmbedInterval = null;
      eggStatsEnabled = false;
      updateStatusUI(false);
      return;
    }

    // turn ON
    if (!matchStartTime) matchStartTime = Date.now();

    const webhookURL = webhookInput.value.trim();
    if (!webhookURL) return alert("Enter webhook URL");
    localStorage.setItem("eggWebhook", webhookURL);

    // NEW: always start a fresh message when turning ON
    liveEmbedMessageId = null;
    localStorage.removeItem("eggEmbedMsgId");

    safePost(webhookURL, buildDiscordEmbed()).then(() => {
      if (liveEmbedMessageId) startEmbedUpdates(webhookURL);
    });

    const ticker = document.querySelector("#killTicker");
    if (!ticker) return alert("Kill ticker not found");

    let lastMsg = "";

    observer = new MutationObserver(() => {
      const killerEl = document.querySelector("#killTicker > *:nth-last-child(3)");
      const killedEl = document.querySelector("#killTicker > *:nth-last-child(2)");
      if (!killedEl || finalEmbedLocked) return;

      const killer = killerEl?.innerText?.trim() || "";
      const killed = killedEl.innerText.trim();

      ensurePlayer(killed);

      let msg;
      if (!killer || killer === killed) {
        matchStats[killed].deaths++;
        msg = `${killed} eliminated themselves`;
      } else {
        ensurePlayer(killer);
        matchStats[killer].kills++;
        matchStats[killed].deaths++;
        msg = `${killer} killed ${killed}`;
      }

      if (msg === lastMsg) return;
      lastMsg = msg;

      if (leaderboardVisible) renderLeaderboard();
    });

    observer.observe(ticker, { childList: true });
    eggStatsEnabled = true;
    updateStatusUI(true);
  }

  /***********************
   * LIVE LEADERBOARD (AUTO)
   ***********************/
  let leaderboardVisible = false;
  let leaderboardEl = null;
  let leaderboardInterval = null;

  function renderLeaderboard() {
    if (!leaderboardEl) {
      leaderboardEl = document.createElement("div");
      leaderboardEl.style.cssText = `
        position: fixed;
        top: 80px;
        right: 20px;
        background: #111;
        color: white;
        padding: 10px;
        border-radius: 12px;
        font-family: Arial;
        z-index: 999999;
        width: 340px;
      `;
      document.body.appendChild(leaderboardEl);
    }

    const players = Object.entries(matchStats)
      .map(([n, s]) => {
        const kdr = s.deaths === 0 ? s.kills : s.kills / s.deaths;
        return { name: n, ...s, kdr };
      })
      .sort((a, b) => {
        if (b.kills !== a.kills) return b.kills - a.kills;
        if (a.deaths !== b.deaths) return a.deaths - b.deaths;
        if (b.kdr !== a.kdr) return b.kdr - a.kdr;
        return a.name.localeCompare(b.name);
      });

    const medals = ["🥇", "🥈", "🥉"];

    leaderboardEl.innerHTML = `
      <b style="display:block;text-align:center;margin-bottom:6px;">🏆 Live Leaderboard</b>
      <table style="width:100%;font-size:11px;">
        <tr><th align="left">Player</th><th>K</th><th>D</th><th>KDR</th></tr>
        ${players.map((p, i) => `
          <tr style="
            ${i === 0 ? "box-shadow:0 0 12px #f1c40f;background:#2c2c00;" : ""}
          ">
            <td>${i === 0 ? "⭐ " : ""}${medals[i] || ""} ${p.name}</td>
            <td>${p.kills}</td>
            <td>${p.deaths}</td>
            <td>${p.kdr.toFixed(2)}</td>
          </tr>
        `).join("")}
      </table>
    `;
  }

  function startLeaderboardUpdates() {
    if (leaderboardInterval) return;
    leaderboardInterval = setInterval(() => {
      if (leaderboardVisible) renderLeaderboard();
    }, 1000);
  }

  function stopLeaderboardUpdates() {
    clearInterval(leaderboardInterval);
    leaderboardInterval = null;
  }

  /***********************
   * KEYBINDS
   ***********************/
  document.addEventListener("keydown", e => {
    if (["INPUT", "TEXTAREA"].includes(e.target.tagName)) return;

    if (e.key === "\\") toggleEggStats();

    if (e.key === "]") {
      leaderboardVisible = !leaderboardVisible;
      if (leaderboardVisible) {
        renderLeaderboard();
        leaderboardEl.style.display = "block";
        startLeaderboardUpdates();
      } else {
        leaderboardEl.style.display = "none";
        stopLeaderboardUpdates();
      }
    }

    if (e.key === "[") {
      panel.style.display = panel.style.display === "none" ? "block" : "none";
    }
  });

  /***********************
   * SERVER CHANGE DETECTION
   ***********************/
  let tickerObserverRetry = null;

  function attachKillObserver() {
    if (!eggStatsEnabled || observer) return;

    const ticker = document.querySelector("#killTicker");
    if (!ticker) return;

    observer = new MutationObserver(() => {
      const killerEl = document.querySelector("#killTicker > *:nth-last-child(3)");
      const killedEl = document.querySelector("#killTicker > *:nth-last-child(2)");
      if (!killedEl || finalEmbedLocked) return;

      const killer = killerEl?.innerText?.trim() || "";
      const killed = killedEl.innerText.trim();

      ensurePlayer(killed);

      if (!killer || killer === killed) {
        matchStats[killed].deaths++;
      } else {
        ensurePlayer(killer);
        matchStats[killer].kills++;
        matchStats[killed].deaths++;
      }

      if (leaderboardVisible) renderLeaderboard();
    });

    observer.observe(ticker, { childList: true });
    console.log("[EggStats] Kill observer attached");
  }

  tickerObserverRetry = setInterval(() => {
    if (!eggStatsEnabled) return;

    const ticker = document.querySelector("#killTicker");

    if (!ticker && observer) {
      observer.disconnect();
      observer = null;
      console.log("[EggStats] Kill ticker lost, waiting...");
    }

    if (ticker && !observer) {
      attachKillObserver();
    }
  }, 2000);
/***********************
   * DOWNLOAD STATS AS PNG
   ***********************/
  // Add button to panel
  const dlBtn = document.createElement("button");
  dlBtn.textContent = "📸 Download Stats PNG";
  dlBtn.style.cssText = `
    width:100%; margin-top:4px; padding:8px; border:none; border-radius:6px;
    background:#8e44ad; color:white; cursor:pointer; font-size:12px; transition:0.2s;
  `;
  dlBtn.onmouseover = () => dlBtn.style.background = "#9b59b6";
  dlBtn.onmouseout  = () => dlBtn.style.background = "#8e44ad";
  panel.querySelector("#eggContent").appendChild(dlBtn);

  dlBtn.onclick = () => {
    const players = Object.entries(matchStats)
      .map(([name, s]) => {
        const kdr = s.deaths === 0 ? s.kills : s.kills / s.deaths;
        return { name, ...s, kdr };
      })
      .sort((a, b) => {
        if (b.kills !== a.kills) return b.kills - a.kills;
        if (a.deaths !== b.deaths) return a.deaths - b.deaths;
        return b.kdr - a.kdr;
      });

    if (!players.length) return alert("No stats to download yet.");

    const rowH = 36, headerH = 90, footerH = 40;
    const W = 500, H = headerH + players.length * rowH + footerH;
    const medals = ["🥇", "🥈", "🥉"];

    const canvas = document.createElement("canvas");
    canvas.width = W;
    canvas.height = H;
    const ctx = canvas.getContext("2d");

    // Background
    ctx.fillStyle = "#0f0f0f";
    ctx.fillRect(0, 0, W, H);

    // Header bar
    const grad = ctx.createLinearGradient(0, 0, W, 0);
    grad.addColorStop(0, "#1a1a2e");
    grad.addColorStop(1, "#16213e");
    ctx.fillStyle = grad;
    ctx.fillRect(0, 0, W, headerH);

    // Title
    ctx.fillStyle = "#f1c40f";
    ctx.font = "bold 22px Segoe UI, Arial";
    ctx.textAlign = "center";
    ctx.fillText(finalEmbedLocked ? "🏁 Final Match Results" : "🏆 Live Leaderboard", W / 2, 36);

    // Subtitle (duration)
    ctx.fillStyle = "#aaa";
    ctx.font = "13px Segoe UI, Arial";
    ctx.fillText(`Duration: ${getMatchDuration()}  •  EggStats`, W / 2, 62);

    // Column headers
    ctx.fillStyle = "#555";
    ctx.fillRect(0, headerH - 1, W, 1);
    ctx.fillStyle = "#888";
    ctx.font = "bold 11px Segoe UI, Arial";
    ctx.textAlign = "left";
    ctx.fillText("PLAYER", 50, headerH + 14 - rowH + 24);

    ["K", "D", "KDR"].forEach((label, i) => {
      ctx.textAlign = "center";
      ctx.fillText(label, [360, 410, 465][i], headerH + 14 - rowH + 24);
    });

    // Rows
    players.forEach((p, i) => {
      const y = headerH + i * rowH;

      // Alternating row bg
      ctx.fillStyle = i % 2 === 0 ? "#161616" : "#111";
      ctx.fillRect(0, y, W, rowH);

      // Gold highlight for first place
      if (i === 0) {
        ctx.fillStyle = "rgba(241,196,15,0.08)";
        ctx.fillRect(0, y, W, rowH);
      }

      const textY = y + rowH / 2 + 5;

      // Medal / rank
      ctx.font = "15px Segoe UI, Arial";
      ctx.textAlign = "center";
      ctx.fillStyle = "#fff";
      ctx.fillText(medals[i] || `#${i + 1}`, 22, textY);

      // Player name
      ctx.font = i === 0 ? "bold 14px Segoe UI, Arial" : "14px Segoe UI, Arial";
      ctx.fillStyle = i === 0 ? "#f1c40f" : "#eee";
      ctx.textAlign = "left";
      ctx.fillText(p.name, 50, textY);

      // Stats
      ctx.textAlign = "center";
      ctx.font = "13px Segoe UI, Arial";
      ctx.fillStyle = "#2ecc71"; ctx.fillText(p.kills,          360, textY);
      ctx.fillStyle = "#e74c3c"; ctx.fillText(p.deaths,         410, textY);
      ctx.fillStyle = "#3498db"; ctx.fillText(p.kdr.toFixed(2), 465, textY);

      // Row divider
      ctx.fillStyle = "#222";
      ctx.fillRect(0, y + rowH - 1, W, 1);
    });

    // Footer
    const footerY = headerH + players.length * rowH;
    ctx.fillStyle = "#1a1a1a";
    ctx.fillRect(0, footerY, W, footerH);
    ctx.fillStyle = "#555";
    ctx.font = "11px Segoe UI, Arial";
    ctx.textAlign = "center";
    ctx.fillText(`Generated by EggTools • ${new Date().toLocaleString()}`, W / 2, footerY + 25);

    // Download
    const link = document.createElement("a");
    link.download = `eggstats_${Date.now()}.png`;
    link.href = canvas.toDataURL("image/png");
    link.click();
  };
})();
