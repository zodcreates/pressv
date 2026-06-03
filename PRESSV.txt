// ==UserScript==
// @name         Shell Shockers Aimbot + ESP
// @namespace    REPLACE_AFTER_GREASYFORK_SIGNUP
// @version      2.1
// @author       REPLACE_AFTER_GREASYFORK_SIGNUP
// @license      GPL-3.0
// @match        https://shellshock.io/*
// @grant        unsafeWindow
// @run-at       document-start
// @require      https://cdn.jsdelivr.net/npm/babylonjs@7.15.0/babylon.min.js
// @description  shellshock.io aimbot. Hold RMB to snap to nearest enemy (with optional lead/prediction). Press V to toggle red wireframe ESP boxes + tracers (see through walls). Press ` (backtick) to show/hide the settings menu.
// ==/UserScript==
 
(function () {
    'use strict';
 
    // ────────────────────────────────────────────────────────────────────────
    // STATE + SETTINGS
    // ────────────────────────────────────────────────────────────────────────
    let RMB = false;
    let H = {};
    const ss = {};
 
    // Per-frame snapshot. `hasLock` is true only on frames where RMB is held
    // and a valid enemy was picked — it drives the overlay's
    // "locked, idle / lead / waiting" state line. `me` is a cached reference
    // to the local player used by the no-spread helper to identify our own
    // bullets at spawn time.
    const _aim = {
        hasLock: false,
        me: null,
    };
    const _pred = {
        enabled: false, active: false,
        speed: 0, t: 0, leadDist: 0, projSpeed: 0,
    };
    let nyxBannerEl = null;
    // Tuning constants — picked, not exposed in the menu. Lifted from the
    // working babylon.js prediction tuner.
    const SENSITIVITY           = 0.0025; // mouse-pixels → yaw radians
    const PROJECTILE_SPEED      = 80;     // fallback bullet speed (units/sec) when weapon lookup fails
    const VELOCITY_DEADZONE     = 0.02;   // below this speed, treat target as stationary
    const VELOCITY_SMOOTHING    = 0.25;   // EMA factor on raw velocity (lower = calmer)
    const PREDICTION_MAX_T      = 1.2;    // cap lead horizon — beyond this, overshoot dominates
    // Projectile + gravity timebase, ported from "other script". That script
    // works in per-tick units (gravity -0.012/tick², terminal 0.29/tick, and a
    // +1-tick lead bias); babylon2 runs in per-second, so convert through the
    // 30 Hz tick rate (shellshock's projectile/physics tick).
    const TICK_RATE             = 30;
    const TICK                  = 1 / TICK_RATE;          // ≈ 0.0333 s — the "+1 tick" lead bias
    const GRAVITY               = -0.012 / (TICK * TICK); // ≈ -10.8 u/s² (other script's -0.012/tick²)
    const V_TERMINAL            = 0.29 / TICK;            // ≈ 8.7 u/s — terminal fall speed (0.29/tick)
 
    const SETTINGS_KEY = 'ssh_settings_v1';
    const DEFAULT_SETTINGS = {
        aimEnabled:  true,
        crosshairTarget: false,
        espEnabled:  false,
        predEnabled: true,
        noSpread:    true,
        unlockSkins: true,
        itemEsp:     true,
        menuVisible: true,
    };
    const settings = Object.assign({}, DEFAULT_SETTINGS);
    try {
        const saved = JSON.parse(localStorage.getItem(SETTINGS_KEY) || '{}');
        Object.assign(settings, saved);
    } catch (e) {}
    function saveSettings() {
        try { localStorage.setItem(SETTINGS_KEY, JSON.stringify(settings)); } catch (e) {}
        // Mirror to the page's global so bundle-side patches can read these
        // flags without crossing the userscript sandbox.
        try { unsafeWindow.ssh_noSpread   = settings.noSpread;   } catch (e) {}
        try { unsafeWindow.ssh_skinUnlock  = settings.unlockSkins; } catch (e) {}
    }
    try { unsafeWindow.ssh_noSpread  = settings.noSpread;   } catch (e) {}
    try { unsafeWindow.ssh_skinUnlock = settings.unlockSkins; } catch (e) {}
    let refreshMenu = () => {}; // replaced by buildMenu() once DOM exists
 
    const log = (...a) =>
        console.log('%cShellhax', 'color:#000;background:#ff0;padding:2px 6px;border-radius:4px;font-weight:bold', ...a);
    log('started');
 
    // Scan an object's own properties for one whose value has `propName` set.
    // Used to discover the obfuscated `actor` key dynamically — much more
    // robust than relying on a regex that may not match every build.
    //
    // `skip` excludes known false-positive keys. `weapon` carries its own
    // `.mesh` (the gun model), and was previously matching first — making
    // the prediction read gun-tip position instead of the body actor, so
    // velocity tracked gun rotation rather than player movement.
    function findKeyWithProperty(obj, propName, skip) {
        for (const k in obj) {
            if (!obj.hasOwnProperty(k)) continue;
            if (skip && skip.indexOf(k) !== -1) continue;
            const v = obj[k];
            if (v && typeof v === 'object' && v.hasOwnProperty(propName)) return k;
        }
        return null;
    }
 
    // ────────────────────────────────────────────────────────────────────────
    // INPUT — RMB for aim, V for ESP, ` to toggle menu
    // ────────────────────────────────────────────────────────────────────────
    document.addEventListener('mousedown', e => { if (e.button === 2) RMB = true;  }, true);
    document.addEventListener('mouseup',   e => { if (e.button === 2) RMB = false; }, true);
    document.addEventListener('keydown', e => {
        const t = document.activeElement;
        if (t && (t.tagName === 'INPUT' || t.tagName === 'TEXTAREA')) return;
        if (e.repeat) return;
        if (e.code === 'KeyV') {
            settings.espEnabled = !settings.espEnabled;
            saveSettings(); refreshMenu();
            log('ESP', settings.espEnabled ? 'ON' : 'OFF');
        } else if (e.code === 'Backquote') {
            settings.menuVisible = !settings.menuVisible;
            saveSettings(); refreshMenu();
        }
    }, true);
 
    // ────────────────────────────────────────────────────────────────────────
    // MENU — tiny on-screen panel, persists to localStorage.
    // ────────────────────────────────────────────────────────────────────────
    function buildMenu() {
        if (document.getElementById('ssh-menu')) return;
        if (!document.body) { setTimeout(buildMenu, 50); return; }
 
        // Collapsible-category styling: each header is clickable and toggles
        // its body open/closed. Everything else matches the original panel.
        const css = document.createElement('style');
        css.textContent = `
            #ssh-menu .ssh-cat-hdr{cursor:pointer;padding:4px 0;font-weight:bold;
                border-top:1px solid #333;display:flex;justify-content:space-between;align-items:center}
            #ssh-menu .ssh-cat-body{display:none;padding-left:6px}
            #ssh-menu .ssh-cat.open .ssh-cat-body{display:block}
            #ssh-menu .ssh-arr{transition:transform .2s;font-size:9px;opacity:0.6}
            #ssh-menu .ssh-cat.open .ssh-arr{transform:rotate(90deg)}
        `;
        document.head.appendChild(css);
 
        const wrap = document.createElement('div');
        wrap.id = 'ssh-menu';
        wrap.style.cssText = [
            'position:fixed', 'top:12px', 'left:12px', 'z-index:2147483647',
            'background:rgba(0,0,0,0.82)', 'color:#fff',
            'font:12px/1.4 -apple-system,system-ui,sans-serif',
            'padding:10px 12px', 'border-radius:6px', 'min-width:210px',
            'border:1px solid #000000', 'user-select:none',
            'box-shadow:0 2px 10px rgba(101, 0, 0, 0)',
        ].join(';');
        wrap.innerHTML = `
            <div style="font-weight:bold;color:#ff3b3b;margin-bottom:6px;display:flex;justify-content:space-between;align-items:center;">
                <span>Shellhax</span>
                <span style="opacity:0.55;font-weight:normal;font-size:11px;">\` to hide</span>
            </div>
            <div class="ssh-cat open">
                <div class="ssh-cat-hdr">Combat <span class="ssh-arr">&#9654;</span></div>
                <div class="ssh-cat-body">
                    <label style="display:block;margin:3px 0;"><input type="checkbox" data-k="aimEnabled"> Aimbot <span style="opacity:0.55;">(hold RMB)</span></label>
                    <label style="display:block;margin:3px 0;"><input type="checkbox" data-k="crosshairTarget"> Target Crosshair <span style="opacity:0.55;">(else closest)</span></label>
                    <label style="display:block;margin:3px 0;"><input type="checkbox" data-k="predEnabled"> Prediction</label>
                    <label style="display:block;margin:3px 0;"><input type="checkbox" data-k="noSpread"> No Spread</label>
                </div>
            </div>
            <div class="ssh-cat open">
                <div class="ssh-cat-hdr">ESP <span class="ssh-arr">&#9654;</span></div>
                <div class="ssh-cat-body">
                    <label style="display:block;margin:3px 0;"><input type="checkbox" data-k="espEnabled"> ESP <span style="opacity:0.55;">(V)</span></label>
                    <label style="display:block;margin:3px 0;"><input type="checkbox" data-k="itemEsp"> Item ESP <span style="opacity:0.55;">(ammo · grenades)</span></label>
                </div>
            </div>
            <div class="ssh-cat">
                <div class="ssh-cat-hdr">Skins <span class="ssh-arr">&#9654;</span></div>
                <div class="ssh-cat-body">
                    <label style="display:block;margin:3px 0;"><input type="checkbox" data-k="unlockSkins"> Unlock Skins</label>
                </div>
            </div>
            <div id="ssh-nyx" style="margin-top:6px;padding:6px 8px;border-radius:4px;background:rgba(180,80,255,0.18);border:1px solid rgba(180,80,255,0.55);color:#e9d5ff;font-size:11px;line-height:1.35;">
                Join the <b>Nyx</b> wave — start your name with <b>Nyx</b> so we can spot each other.
            </div>
        `;
        document.body.appendChild(wrap);
 
        nyxBannerEl = wrap.querySelector('#ssh-nyx');
        const inputs = wrap.querySelectorAll('[data-k]');
 
        // Collapsible categories: clicking a header toggles its body open/closed.
        wrap.querySelectorAll('.ssh-cat-hdr').forEach(hdr => {
            hdr.addEventListener('click', () => hdr.parentElement.classList.toggle('open'));
        });
 
        refreshMenu = () => {
            inputs.forEach(el => {
                const k = el.dataset.k;
                if (el.type === 'checkbox') el.checked = !!settings[k];
                else                        el.value   = settings[k];
            });
            wrap.style.display = settings.menuVisible ? '' : 'none';
        };
        refreshMenu();
 
        inputs.forEach(el => {
            el.addEventListener('input', () => {
                const k = el.dataset.k;
                if (el.type === 'checkbox') {
                    settings[k] = el.checked;
                } else {
                    const n = parseFloat(el.value);
                    settings[k] = Number.isFinite(n) ? n : DEFAULT_SETTINGS[k];
                }
                saveSettings(); refreshMenu();
            });
        });
 
        // Stop menu interactions from bleeding into the game.
        ['keydown','keyup','mousedown','mouseup','wheel','contextmenu'].forEach(ev =>
            wrap.addEventListener(ev, e => e.stopPropagation(), true));
 
        log('menu built');
    }
    if (document.body) buildMenu();
    else document.addEventListener('DOMContentLoaded', buildMenu);
 
    // ────────────────────────────────────────────────────────────────────────
    // YAW/PITCH MECHANISM (trimmed port of babylon.js's `yawpitch` helper)
    //
    // shellshock.io's camera state lives in WASM. You cannot move the camera
    // by mutating player.yaw / player.pitch directly — the WASM module is the
    // source of truth. The only thing that DOES affect the camera is the
    // game's `pointermove` listener (named "real" in the bundle), which
    // converts movementX/Y into WASM look deltas.
    //
    // Steps:
    //   1. Hook addEventListener early to capture that listener.
    //   2. To turn the camera N radians, synthesize a fake pointermove event
    //      with movementX = N/sensitivity and call the listener directly.
    //   3. After moving, read back the resulting yaw/pitch from the bundle's
    //      `unsafeWindow.get_yaw_pitch()` helper.
    // ────────────────────────────────────────────────────────────────────────
    let realPointerListener = null;
    const _origAEL = EventTarget.prototype.addEventListener;
    EventTarget.prototype.addEventListener = function (type, listener, options) {
        try {
            if (type === 'pointermove' && listener && listener.name === 'real') {
                realPointerListener = listener;
                log('captured real pointermove listener');
            }
        } catch (e) { /* never break the page's own event hooks */ }
        return _origAEL.call(this, type, listener, options);
    };
 
    function getCurrentYawPitch() {
        try { return unsafeWindow.get_yaw_pitch(); } catch (e) { return null; }
    }
 
    function movePointer(mx, my) {
        mx = Math.round(mx); my = Math.round(my);
        if (mx === 0 && my === 0) return;
        if (!realPointerListener) return;
        realPointerListener({ movementX: mx, movementY: my, x: 1, isTrusted: true });
    }
 
    // Signed shortest-arc difference between two angles, in (-π, π].
    function radianDiff(a, b) {
        const TAU = 2 * Math.PI;
        a = ((a % TAU) + TAU) % TAU;
        b = ((b % TAU) + TAU) % TAU;
        let d = Math.abs(a - b);
        d = Math.min(d, TAU - d);
        return (((a - b + TAU) % TAU) > Math.PI) ? -d : d;
    }
 
    // Aim camera at the given yaw/pitch (in radians, get_yaw_pitch convention).
    function setToYawPitch(targetYaw, targetPitch) {
        const cur = getCurrentYawPitch();
        if (!cur) return;
        const dy = radianDiff(cur.yaw,   targetYaw);
        const dp = radianDiff(cur.pitch, targetPitch);
        movePointer(dy / SENSITIVITY, dp / SENSITIVITY);
    }
 
    // ────────────────────────────────────────────────────────────────────────
    // NO-SPREAD HELPER — called from inside the bundle's fireThis hook.
    // Replaces the bullet's spawn direction with the camera-forward unit
    // vector, so every shot flies exactly along the crosshair — no spread
    // even while moving or after a sustained burst. The local camera is
    // already where the player is aiming, so the server's hit detection
    // sees no aim divergence and registers hits normally.
    // ────────────────────────────────────────────────────────────────────────
    // Local-player identification — try multiple markers because the bundle
    // can hand fireThis a player reference that differs from what we cached
    // in PLAYERS (different object instances). Any single match is enough.
    const _isLocalPlayer = (p) => {
        if (!p || typeof p !== 'object') return false;
        if (_aim.me && p === _aim.me) return true;
        if (p.hasOwnProperty('ws')) return true;
        return false;
    };
 
    // Inspect anytime in the page console by typing `ssh_diag` — gives a
    // live snapshot of whether the bullet-spawn inject is running and what
    // the helper sees. Vital for debugging "no-spread doesn't work" cases
    // where the issue might be (a) inject never runs (different fire path),
    // (b) noSpread off, (c) owner gate failing, or (d) BABYLON / yaw lookup
    // unavailable.
    unsafeWindow.ssh_diag = {
        calls: 0,
        bailNoSetting: 0,
        bailNotLocal: 0,
        bailNoBabylon: 0,
        bailNoYawPitch: 0,
        redirected: 0,
        lastOwnerKeys: '',
        lastOwnerIsLocal: null,
        // No-spread cone instrumentation (set by the spread-zero patch):
        coneFire: 0,   // times the patched weapon cone ran (any shot)
        coneZero: 0,   // times it actually zeroed spread (flag was on)
    };
 
    unsafeWindow.ssh_silentAimRedirect = function (_bullet, owner, _origin, _dir, _weapon) {
        try {
            const d = unsafeWindow.ssh_diag;
            d.calls++;
            if (owner && typeof owner === 'object') {
                d.lastOwnerKeys = Object.keys(owner).slice(0, 40).join(',');
                d.lastOwnerIsLocal = _isLocalPlayer(owner);
            }
            if (!settings.noSpread) { d.bailNoSetting++; return null; }
            if (!_isLocalPlayer(owner)) { d.bailNotLocal++; return null; }
            if (typeof BABYLON === 'undefined') { d.bailNoBabylon++; return null; }
            const cur = getCurrentYawPitch();
            if (!cur) { d.bailNoYawPitch++; return null; }
            const y = cur.yaw, p = cur.pitch;
            // Shellshock's yaw/pitch → forward convention (verified against
            // the working aim path: targetYaw = atan2(-dx,-dz), targetPitch
            // chosen so positive pitch = looking up). The previous formula
            // had every component negated, which spawned bullets aimed
            // INTO the player's own body — invisible and instantly stopped.
            const v = new BABYLON.Vector3(
                -Math.sin(y) * Math.cos(p),
                 Math.sin(p),
                -Math.cos(y) * Math.cos(p),
            );
            d.redirected++;
            return v;
        } catch (e) { return null; }
    };
 
    // ────────────────────────────────────────────────────────────────────────
    // BUNDLE INTERCEPTION — same pattern as babylon.js (which works).
    // ────────────────────────────────────────────────────────────────────────
    const _origReplace = String.prototype.replace;
    String.prototype.sshReplace = function () { return _origReplace.apply(this, arguments); };
 
    const CB_NAME = 'ssh_' + Math.random().toString(36).slice(2, 10);
 
    const DEFAULTS = {
        x: 'Fh', y: 'Qh', z: 'Th',
        yaw: 'Eh', pitch: '$h', coords: 'aa',
        playing: 'Gh',
        actor: 'eh', mesh: 'mesh',
        weapon: 'Ah',
        renderingGroupId: 'renderingGroupId',
        SCENE: 'eI', PLAYERS: 'rI',
        CULL: 'Xw',
        items: 'kr',
    };
 
    function extractKeys(js) {
        const out = Object.assign({}, DEFAULTS);
        try {
            let m;
            // WASM mouse-look bridge: `(zO.Kw=e.yaw,zO.Ew=e.pitch,zO.Qz=e.coords)`
            m = /\(([a-zA-Z_$0-9]+)\.([a-zA-Z_$0-9]+)=e\.yaw,\1\.([a-zA-Z_$0-9]+)=e\.pitch,\1\.([a-zA-Z_$0-9]+)=e\.coords\)/.exec(js);
            if (m) { out.yaw = m[2]; out.pitch = m[3]; out.coords = m[4]; }
            // Players-array iteration
            m = /for\s*\(\s*(?:var|let)\s+\w+=0;\w+<([a-zA-Z_$0-9]+);\w+\+\+\)\s*\{\s*(?:var|let)\s+\w+=([a-zA-Z_$0-9]+)\[\w+\];\s*\w+&&\w+\.([a-zA-Z_$0-9]+)&&/.exec(js);
            if (m) { out.PLAYERS = m[2]; out.playing = m[3]; }
            // Spectator-info builder pos fields
            m = /posX:[a-zA-Z_$0-9]+\.([a-zA-Z_$0-9]+),posY:[a-zA-Z_$0-9]+\.([a-zA-Z_$0-9]+),posZ:[a-zA-Z_$0-9]+\.([a-zA-Z_$0-9]+)/.exec(js);
            if (m) { out.x = m[1]; out.y = m[2]; out.z = m[3]; }
            // Actor + mesh
            m = /actorX:[a-zA-Z_$0-9]+\.([a-zA-Z_$0-9]+)\.([a-zA-Z_$0-9]+)\.position\.x/.exec(js);
            if (m) { out.actor = m[1]; out.mesh = m[2]; }
            // SCENE: first render() in the two-render() pattern
            m = /([a-zA-Z_$0-9]+)\.render\(\),[a-zA-Z_$0-9]+\.render\(\)\}\)\)/.exec(js);
            if (m) out.SCENE = m[1];
            // Items manager: the network-protocol handler calls
            // `<X>.spawnItem(s,p,m,v,y);break;` with exactly those 5 args.
            // Capture <X> as the items-manager instance.
            m = /([a-zA-Z_$0-9]+)\.spawnItem\([a-zA-Z_$0-9]+,[a-zA-Z_$0-9]+,[a-zA-Z_$0-9]+,[a-zA-Z_$0-9]+,[a-zA-Z_$0-9]+\);break;/.exec(js);
            if (m) out.items = m[1];
        } catch (e) { log('key extraction error:', e); }
        return out;
    }
 
    function patchBundle(js) {
        H = extractKeys(js);
        log('H map:', H);
 
        const argFields = Object.keys(H)
            .map(k => `${k}:(()=>{try{return ${H[k]}}catch(_){return null}})()`)
            .join(',');
        const find    = H.SCENE + '.render';
        const replace = `window["${CB_NAME}"]({${argFields}},true)||${H.SCENE}.render`;
        const before  = js;
        js = js.sshReplace(find, replace);
        if (before === js) log('WARNING: SCENE.render patch did not match');
        else               log('SCENE.render hook installed');
 
        // Cull inhibition: the bundle hides off-screen / occluded players via
        // `{if(<CULL>)`. Patch to `{if(true)` so all players keep rendering
        // even when behind walls — otherwise our ESP boxes (parented to the
        // player mesh) disappear with them.
        const cullBefore = js;
        js = js.sshReplace('{if(' + H.CULL + ')', '{if(true)');
        if (cullBefore === js) log('WARNING: cull-inhibition patch did not match (H.CULL=' + H.CULL + ')');
        else                   log('cull inhibition installed');
 
        // ── Bullet-spawn hook (client-side direction redirect / no-spread) ──
        // The original method name was `fireThis` in babylon.js's era; in the
        // current build it may be minified, and the parameter names vary by
        // build (saw `(e,t,i,n)` historically, `(e,i,r,n)` in one snapshot,
        // and the production bundle uses something else entirely).
        //
        // Anchor instead on the convention that's been stable across builds:
        // a prototype method with FOUR parameters whose body normalizes a
        // direction vector with `this.direction.copyFrom(<p3>).normalize()
        // .scaleInPlace(<p4>.velocity)`. The lookahead requires that body
        // pattern, with the 3rd param as the direction and the 4th as the
        // weapon — that pins the meaning of the params regardless of how
        // they're named in this build.
        // Use [^}] (not [\s\S]) so the lookahead can't cross a function-body
        // close-brace into the next function — without this, the regex hit
        // Qv.prototype.clearRect because the *next* function's body contained
        // the direction-normalize pattern within 1000 chars of clearRect's
        // open-brace.
        const fireRegex = /([a-zA-Z_$0-9]+)\.prototype\.([a-zA-Z_$0-9]+)=function\(([a-zA-Z_$0-9]+),([a-zA-Z_$0-9]+),([a-zA-Z_$0-9]+),([a-zA-Z_$0-9]+)\)\{(?=[^}]{0,800}?this\.direction\.copyFrom\(\5\)\.normalize\(\)\.scaleInPlace\(\6\.velocity\))/g;
        const fireStubs = [];
        let _fm;
        while ((_fm = fireRegex.exec(js)) !== null) {
            fireStubs.push({
                match: _fm[0], cls: _fm[1], method: _fm[2],
                p1: _fm[3], p2: _fm[4], p3: _fm[5], p4: _fm[6],
            });
        }
        if (fireStubs.length) {
            for (const s of fireStubs) {
                const inject = 'var lmd;if(window.ssh_silentAimRedirect&&(lmd=window.ssh_silentAimRedirect(this,' + s.p1 + ',' + s.p2 + ',' + s.p3 + ',' + s.p4 + ')))' + s.p3 + '=lmd;';
                js = js.sshReplace(s.match, s.match + inject);
                log('bullet-spawn hook: ' + s.cls + '.prototype.' + s.method + '(' + s.p1 + ',' + s.p2 + ',' + s.p3 + ',' + s.p4 + ') — dir=' + s.p3);
            }
        } else {
            log('bullet-spawn direction hook not found (expected on this build — spread-zero patch below handles no-spread)');
        }
 
        // ── No-spread (spread-zero) patch — the reliable path ──
        // The redirect hook above only changes the LOCAL bullet's direction.
        // The spread the server simulates comes from the weapon's fire() cone,
        // built BEFORE the direction is sent to the network:
        //   let e=n, i=<M>._i((<rng>.getFloat()-.5)*e, …*e, …*e); t=t.multiply(i)
        // where <M>._i is RotationYawPitchRoll and `e` (=`n`) is the spread
        // angle. When e === 0 that rotation is identity → the bullet flies
        // exactly along yaw/pitch, on client AND server (the cleaned direction
        // is what gets transmitted). So we zero the spread angle at its source.
        // Different builds rename the vars, so we anchor on the stable cone
        // shape (`._i((<x>.getFloat()-.5)*<v>,`) rather than property names.
        const spreadBefore = js;
        const spreadPatterns = [
            // os snapshot: n=this.player.shotSpread+this.inaccuracy
            { re: /this\.player\.shotSpread\+this\.inaccuracy/g,
              fn: (m) => '(window.ssh_noSpread?0:(' + m + '))' },
            // generic: <a>.shotSpread+<b>.inaccuracy  (receivers may differ)
            { re: /[a-zA-Z_$0-9.]+\.shotSpread\+[a-zA-Z_$0-9.]+\.inaccuracy/g,
              fn: (m) => '(window.ssh_noSpread?0:(' + m + '))' },
            // current build (May 2026): zero the cone's spread angle `e`.
            // Match `let e=n,i=<M>._i((<P>.getFloat()-.5)*e,` and gate `e` to 0.
            { re: /let ([\w$]+)=([\w$]+),([\w$]+)=([\w$.]+)\._i\(\(([\w$.]+)\.getFloat\(\)-\.5\)\*\1,/,
              fn: (_m, v1, src, v3, M, P) =>
                  'let ' + v1 + '=(window.ssh_diag&&(window.ssh_diag.coneFire++,' +
                  'window.ssh_noSpread&&window.ssh_diag.coneZero++),' +
                  'window.ssh_noSpread?0:' + src + '),' +
                  v3 + '=' + M + '._i((' + P + '.getFloat()-.5)*' + v1 + ',' },
        ];
        for (const p of spreadPatterns) {
            js = js.sshReplace(p.re, p.fn);
            if (spreadBefore !== js) break;
        }
        if (spreadBefore === js) {
            log('WARNING: spread-zero pattern not found — running cone DIAG');
            try {
                const count = (tk) => js.split(tk).length - 1;
                const tokens = ['shotSpread', 'inaccuracy', 'instability',
                                'randomGen', 'fireMunitions', 'fireThis'];
                log('no-spread DIAG token counts: ' +
                    tokens.map(tk => tk + '=' + count(tk)).join(', '));
                // The spread vars are minified in this build (shotSpread/etc =
                // 0), but the spread cone is built by the caller right before
                // .fire()/fireMunitions. Dump WIDE slices around fireMunitions
                // so we can see the real cone math + variable names.
                const anchor = ['fireMunitions', 'fireThis']
                    .find(a => js.indexOf(a) !== -1);
                if (anchor) {
                    const hits = [];
                    let from = 0, idx;
                    while ((idx = js.indexOf(anchor, from)) !== -1 && hits.length < 10) {
                        hits.push(js.slice(Math.max(0, idx - 520), idx + 200));
                        from = idx + anchor.length;
                    }
                    log('no-spread DIAG anchor "' + anchor + '": ' + hits.length + ' site(s) — paste these back:');
                    hits.forEach((h, i) => log('SPR[' + i + ']: ' + h));
                } else {
                    log('no-spread DIAG: no fireMunitions/fireThis anchor found');
                }
            } catch (e) { log('no-spread DIAG failed: ' + (e && e.message)); }
        } else {
            log('spread-zero (no-spread) patch installed');
        }
 
        // ── Item ESP hooks (ammo + grenade drops) ──
        // Anchor on the items-manager prototype methods. Both spawnItem
        // and collectItem have a stable shape:
        //   spawnItem(e,t,i,r,n){var a=this.pools[t].retrieve(e); ...}
        //   collectItem(e,t){var i=this.pools[e]; i.recycle(...); ...}
        // Param names are captured generically so renaming between builds
        // doesn't break the patch.
        // Anchor on the function body (`var a=this.pools[t].retrieve(e);`)
        // and accept both `<Cls>.prototype.spawnItem=function(...)` (old
        // bundle style) and `spawnItem(...){` (ES6 class syntax). The body
        // pattern is what disambiguates from call sites like
        // `kr.spawnItem(s,p,m,v,y);`.
        const spawnItemBefore = js;
        js = js.sshReplace(
            /((?:\.prototype\.spawnItem=function|\bspawnItem)\(([a-zA-Z_$0-9]+),([a-zA-Z_$0-9]+),([a-zA-Z_$0-9]+),([a-zA-Z_$0-9]+),([a-zA-Z_$0-9]+)\)\{var ([a-zA-Z_$0-9]+)=this\.pools\[\3\]\.retrieve\(\2\);)/,
            '$1window.ssh_onItemSpawn&&window.ssh_onItemSpawn($7,$3,$4,$5,$6);'
        );
        if (spawnItemBefore === js) log('WARNING: spawnItem pattern not found — item ESP disabled');
        else                        log('spawnItem hook installed');
 
        const collectItemBefore = js;
        js = js.sshReplace(
            /((?:\.prototype\.collectItem=function|\bcollectItem)\(([a-zA-Z_$0-9]+),([a-zA-Z_$0-9]+)\)\{var ([a-zA-Z_$0-9]+)=this\.pools\[\2\];)/,
            '$1window.ssh_onItemCollect&&window.ssh_onItemCollect($2,$4.objects[$3]);'
        );
        if (collectItemBefore === js) log('WARNING: collectItem pattern not found');
        else                          log('collectItem hook installed');
 
        // ── Skin unlock ──
        // The bundle's "do I own this skin?" check is:
        //   `inventory[X].id===Y.id) return true; return false`
        // Patch the trailing comparison so it ALSO passes when our flag
        // `window.ssh_skinUnlock` is set — every skin then appears owned.
        // Gated client-side; the server may still validate at use, but the
        // shop/inventory UI will let you try them on.
        const skinBefore = js;
        js = js.sshReplace(
            /inventory\[[a-zA-Z$_]+\]\.id===[a-zA-Z$_]+\.id\)return!0;return!1/,
            (m) => m + '||window.ssh_skinUnlock'
        );
        if (skinBefore === js) log('WARNING: skin-unlock pattern not found in this bundle');
        else                   log('skin-unlock hook installed');
 
        return js;
    }
 
    const _origAppendChild = HTMLElement.prototype.appendChild;
    HTMLElement.prototype.appendChild = function (node) {
        if (node && node.tagName === 'SCRIPT' && node.innerHTML &&
            node.innerHTML.startsWith('(()=>{')) {
            log('intercepting bundle, ' + node.innerHTML.length + ' chars');
            node.innerHTML = patchBundle(node.innerHTML);
        }
        return _origAppendChild.call(this, node);
    };
 
    // ────────────────────────────────────────────────────────────────────────
    // ESP — wireframe box per enemy, parented to their mesh so it follows
    // smoothly. Created on first sight; visibility toggled per frame.
    // ────────────────────────────────────────────────────────────────────────
    let _espWarned = false;
    // Tracks every player we've created an ESP box for. Used by the sweep
    // in the per-frame callback to dispose orphan boxes when a player leaves
    // the match (no longer in ss.PLAYERS) or switches to our team.
    const _espPlayers = new Set();
    function disposeEspFor(P) {
        if (P._ssh_box)    { try { P._ssh_box.dispose();    } catch (e) {} P._ssh_box    = null; }
        if (P._ssh_tracer) { try { P._ssh_tracer.dispose(); } catch (e) {} P._ssh_tracer = null; }
        velState.delete(P);
        _espPlayers.delete(P);
    }
 
    // ESP color palette. Default enemies render in red; players whose name
    // starts with "Nyx" render in purple so the community is visible at a
    // glance across the map. Reused across boxes + tracers; one Color3
    // instance each so we don't allocate per-frame.
    const ESP_COLOR_RED = (typeof BABYLON !== 'undefined') ? new BABYLON.Color3(1.0, 0.2, 0.2) : null;
    const ESP_COLOR_NYX = (typeof BABYLON !== 'undefined') ? new BABYLON.Color3(0.7, 0.3, 1.0) : null;
    const ESP_COLOR_AMMO    = (typeof BABYLON !== 'undefined') ? new BABYLON.Color3(1.0, 0.95, 0.2) : null;
    const ESP_COLOR_GRENADE = (typeof BABYLON !== 'undefined') ? new BABYLON.Color3(1.0, 0.5, 0.0)  : null;
    const _isNyxName = (n) => typeof n === 'string' && n.toLowerCase().startsWith('nyx');
 
    // Item ESP — markers for ammo/grenade pickups. The bundle's items
    // manager pools meshes (Qo.constructors = [Yo, Ko] → type 0 = ammo,
    // type 1 = grenade), so we keep our marker keyed by the pooled mesh
    // and dispose it on collect.
    // Map (not WeakMap) so we can iterate it each frame to dispose markers
    // whose item went inactive without us seeing the collect event.
    const itemMarkers = new Map();
    function makeItemMarker(x, y, z, color) {
        if (typeof BABYLON === 'undefined' || !ss.SCENE) return null;
        const s = 0.35;
        const V = BABYLON.Vector3;
        const m = BABYLON.MeshBuilder.CreateLineSystem(
            'sshitem_' + Math.random().toString(36).slice(2, 8),
            { lines: [
                [new V(-s, 0, 0), new V(s, 0, 0)],
                [new V(0, -s, 0), new V(0, s, 0)],
                [new V(0, 0, -s), new V(0, 0, s)],
            ]},
            ss.SCENE
        );
        m.color = color;
        m.position.x = x; m.position.y = y; m.position.z = z;
        m[H.renderingGroupId] = 1;
        m.alwaysSelectAsActiveMesh = true;
        m.isPickable = false;
        pierceWalls(m);
        return m;
    }
    unsafeWindow.ssh_onItemSpawn = function (item, type, x, y, z) {
        try {
            if (!settings.itemEsp) return;
            if (!item || !item.mesh) return;
            // Clean up any leftover marker for this pooled mesh.
            const old = itemMarkers.get(item.mesh);
            if (old) { try { old.dispose(); } catch (e) {} itemMarkers.delete(item.mesh); }
            const color = (type === 0 ? ESP_COLOR_AMMO : ESP_COLOR_GRENADE)
                       || new BABYLON.Color3(1, 1, 1);
            const marker = makeItemMarker(x, y, z, color);
            if (marker) itemMarkers.set(item.mesh, marker);
        } catch (e) {}
    };
    unsafeWindow.ssh_onItemCollect = function (_type, item) {
        try {
            if (!item || !item.mesh) return;
            const marker = itemMarkers.get(item.mesh);
            if (marker) {
                try { marker.dispose(); } catch (e) {}
                itemMarkers.delete(item.mesh);
            }
        } catch (e) {}
    };
 
    // Force a mesh to ignore the depth buffer entirely while rendering. This
    // is what actually makes the box/tracer punch through walls — neither
    // renderingGroupId nor setRenderingAutoClearDepthStencil is reliable on
    // its own in this Babylon version.
    function pierceWalls(mesh) {
        if (!mesh || mesh._ssh_pierced) return;
        mesh._ssh_pierced = true;
        let saved = null;
        mesh.onBeforeRenderObservable.add(() => {
            try {
                const eng = ss.SCENE && ss.SCENE.getEngine && ss.SCENE.getEngine();
                if (!eng) return;
                saved = eng.getDepthFunction();
                eng.setDepthFunction(BABYLON.Engine.ALWAYS);
            } catch (e) {}
        });
        mesh.onAfterRenderObservable.add(() => {
            try {
                const eng = ss.SCENE && ss.SCENE.getEngine && ss.SCENE.getEngine();
                if (!eng || saved === null) return;
                eng.setDepthFunction(saved);
                saved = null;
            } catch (e) {}
        });
    }
    function ensureEspBox(P) {
        if (P._ssh_box) return P._ssh_box;
        if (typeof BABYLON === 'undefined') {
            if (!_espWarned) { _espWarned = true; log('ESP: BABYLON not loaded'); }
            return null;
        }
        if (!ss.SCENE) {
            if (!_espWarned) { _espWarned = true; log('ESP: no SCENE'); }
            return null;
        }
        // Build geometry in local space (centered around 0, 0, 0). Position is
        // set per-frame from network coords below — no parenting, so we don't
        // inherit the game's interpolation freeze when it thinks the enemy
        // is occluded.
        const w = 0.4, h = 0.65, d = 0.4;
        const V = BABYLON.Vector3;
        const v = [
            new V(-w/2, 0,   -d/2), new V(w/2, 0,   -d/2),
            new V( w/2, h,   -d/2), new V(-w/2, h,  -d/2),
            new V(-w/2, 0,    d/2), new V(w/2, 0,    d/2),
            new V( w/2, h,    d/2), new V(-w/2, h,   d/2),
        ];
        const lines = [];
        for (let i = 0; i < 4; i++) {
            lines.push([v[i],   v[(i+1)%4]]);
            lines.push([v[i+4], v[(i+1)%4+4]]);
            lines.push([v[i],   v[i+4]]);
        }
        const box = BABYLON.MeshBuilder.CreateLineSystem(
            'sshesp_' + Math.random().toString(36).slice(2, 8),
            { lines },
            ss.SCENE
        );
        box.color = ESP_COLOR_RED || new BABYLON.Color3(1.0, 0.2, 0.2);
        box[H.renderingGroupId] = 1;
        box.alwaysSelectAsActiveMesh = true;
        box.isPickable = false;
        pierceWalls(box);
        P._ssh_box = box;
        _espPlayers.add(P);
        log('ESP: built box for', P.name || P.nickname || '?');
        return box;
    }
 
    // Tracer line from each enemy to a point 5 units BEHIND the camera (so it
    // appears to converge at your eye and project outward to the enemy).
    function ensureEspTracer(P) {
        if (P._ssh_tracer) return P._ssh_tracer;
        if (typeof BABYLON === 'undefined' || !ss.SCENE) return null;
        const placeholder = [new BABYLON.Vector3(0,0,0), new BABYLON.Vector3(0,0,0)];
        const tr = BABYLON.MeshBuilder.CreateLines(
            'sshtracer_' + Math.random().toString(36).slice(2, 8),
            { points: placeholder, updatable: true },
            ss.SCENE
        );
        tr.color = ESP_COLOR_RED || new BABYLON.Color3(1.0, 0.2, 0.2);
        tr.isPickable = false;
        tr.alwaysSelectAsActiveMesh = true;
        tr.doNotSyncBoundingInfo = true;
        tr[H.renderingGroupId] = 1;
        pierceWalls(tr);
        P._ssh_tracer = tr;
        return tr;
    }
    function updateEspTracer(P, crosshairs) {
        const tr = ensureEspTracer(P);
        if (!tr) return null;
        // Use network coords (always live) rather than mesh.position (freezes
        // when the game thinks the enemy is occluded).
        const from = new BABYLON.Vector3(P[H.x], P[H.y] + 0.4, P[H.z]);
        BABYLON.MeshBuilder.CreateLines(undefined, {
            points: [from, crosshairs.clone()],
            instance: tr,
            updatable: true,
        });
        return tr;
    }
 
    // ────────────────────────────────────────────────────────────────────────
    // VELOCITY + ACCELERATION TRACKING (network-coord based).
    //
    // Previously this used `actor.mesh.position` because it's interpolated
    // and produced calmer accel. But in the May 2026 build, the running
    // animation laterally sways the mesh in roughly the direction the gun is
    // pointing — that contaminates the derivative, so the lead ended up
    // pointing where the gun was facing rather than where the enemy was
    // actually moving. Network coords (P[H.x/y/z]) only update on real
    // movement, so velocity tracks true displacement. They're stepped per
    // server tick — EMA smoothing handles that just fine.
    // ────────────────────────────────────────────────────────────────────────
    const velState = new WeakMap();
    function trackVelocity(P, nowMs) {
        const px = P[H.x], py = P[H.y], pz = P[H.z];
        if (typeof px !== 'number' || typeof py !== 'number' || typeof pz !== 'number') return null;
        let s = velState.get(P);
        if (!s) {
            s = { x: px, y: py, z: pz, t: nowMs,
                  vx: 0, vy: 0, vz: 0, primed: false };
            velState.set(P, s);
            return s;
        }
        // Network coords update at ~20 Hz tick rate while this function gets
        // called at frame rate (60–120 Hz). Sampling every frame produces a
        // zero-delta for most frames then a huge spike on the tick frame —
        // EMA can't smooth that. Only recompute velocity on frames where the
        // position ACTUALLY changed (a real server tick). dt is then the
        // time between ticks, giving a clean ~50 ms baseline.
        if (px !== s.x || py !== s.y || pz !== s.z) {
            const dt = (nowMs - s.t) / 1000;
            if (dt > 0.001 && dt < 0.5) {
                const rawVx = (px - s.x) / dt;
                const rawVy = (py - s.y) / dt;
                const rawVz = (pz - s.z) / dt;
                s.vx = s.vx + (rawVx - s.vx) * VELOCITY_SMOOTHING;
                s.vy = s.vy + (rawVy - s.vy) * VELOCITY_SMOOTHING;
                s.vz = s.vz + (rawVz - s.vz) * VELOCITY_SMOOTHING;
                s.primed = true;
            }
            s.x = px; s.y = py; s.z = pz; s.t = nowMs;
        }
        return s;
    }
 
    // Per-weapon bullet speed (Crackshot ≈150, EggK-47 ≈80…). Big difference
    // in long-range lead — fixed-time lead under/overshoots across weapons.
    function getProjectileSpeed(me) {
        try {
            const w = me && me[H.weapon];
            if (w && w.subClass && typeof w.subClass.velocity === 'number' && w.subClass.velocity > 0) {
                return w.subClass.velocity;
            }
        } catch (e) {}
        return PROJECTILE_SPEED;
    }
 
    // Best-effort floor height under (x, z): cast a ray straight down from just
    // above the target and return the first solid hit's y. "other script" uses
    // the game's map-only collider (Collider.grenadeCollidesWithCell) for this;
    // that filtered collider isn't exposed to us, so we pick the nearest solid
    // below via the scene — excluding our own ESP geometry. Returns null when
    // the scene/ray isn't available (then the caller skips the clamp).
    function getGroundY(x, z, fromY) {
        if (!ss.SCENE || typeof BABYLON === 'undefined') return null;
        try {
            const origin = new BABYLON.Vector3(x, fromY + 0.5, z);
            const ray = new BABYLON.Ray(origin, new BABYLON.Vector3(0, -1, 0), 60);
            const pick = ss.SCENE.pickWithRay(ray, (mesh) => {
                if (!mesh || mesh.isPickable === false) return false;
                if (mesh._ssh_box || mesh._ssh_tracer || mesh._ssh_pierced) return false;
                return true;
            });
            return (pick && pick.hit && pick.pickedPoint) ? pick.pickedPoint.y : null;
        } catch (e) { return null; }
    }
 
    // Lead prediction ported from "other script"'s predictAim:
    //   • Single pass (no iteration): flight time t = dist/projSpeed + 1 tick.
    //   • Horizontal: straight-line lead (velocity only, no acceleration term).
    //   • Vertical: integrate the target under gravity until it reaches terminal
    //     velocity, then constant fall — this leads jumping/falling targets
    //     instead of leaving Y alone. Finally clamp to the ground so the aim
    //     point never sinks below the floor under it.
    function predictPosition(basePos, vel, mePos, projSpeed) {
        if (!vel || !vel.primed) return { x: basePos.x, y: basePos.y, z: basePos.z };
        const speed = Math.hypot(vel.vx, vel.vy, vel.vz);
        if (speed <= VELOCITY_DEADZONE) return { x: basePos.x, y: basePos.y, z: basePos.z };
 
        const dist = Math.hypot(basePos.x - mePos.x, basePos.y - mePos.y, basePos.z - mePos.z);
        let t = dist / projSpeed + TICK;          // flight time + 1-tick lead bias
        if (t > PREDICTION_MAX_T) t = PREDICTION_MAX_T;
 
        // Horizontal lead.
        const x = basePos.x + vel.vx * t;
        const z = basePos.z + vel.vz * t;
 
        // Vertical: gravity until terminal velocity, then constant terminal fall.
        const g = GRAVITY;
        const vYTerminal = -V_TERMINAL;           // downward (negative)
        const tAccel = Math.min(t, (vYTerminal - vel.vy) / g);
        const tConst = Math.max(t - tAccel, 0);
        let y = basePos.y + vel.vy * tAccel + 0.5 * g * tAccel * tAccel + vYTerminal * tConst;
 
        // Ground clamp — never aim below the floor.
        const groundY = getGroundY(x, z, basePos.y);
        if (groundY !== null && y < groundY) y = groundY;
 
        return { x, y, z };
    }
 
    // Nyx-tag banner: visible until the local player's in-game name starts
    // with "Nyx" (case-insensitive). The banner lives inside the menu and is
    // hidden once the player adopts the tag — no menu spam after they
    // comply. Hidden also when the menu itself is hidden via backtick.
    function updateNyxBanner() {
        if (!nyxBannerEl) return;
        const name = _aim.me && typeof _aim.me.name === 'string' ? _aim.me.name : '';
        const hasTag = name.toLowerCase().startsWith('nyx');
        nyxBannerEl.style.display = hasTag ? 'none' : '';
    }
 
    // ────────────────────────────────────────────────────────────────────────
    // PER-FRAME CALLBACK — ESP refresh + aim
    // ────────────────────────────────────────────────────────────────────────
    unsafeWindow[CB_NAME] = function (vars) {
        try {
            Object.assign(ss, vars);
            if (!ss.PLAYERS) return false;
 
            const nowMs = performance.now();
 
            // Find local player (the one with the .ws WebSocket attached).
            let me = null;
            for (const P of ss.PLAYERS) {
                if (P && P.hasOwnProperty('ws')) { me = P; break; }
            }
            if (!me) return false;
 
            // Discover the obfuscated `actor` key dynamically — it's whichever
            // key on the player object has a `.mesh` child, excluding the
            // weapon key (whose `.mesh` is the gun model, NOT the body —
            // picking it up made our velocity tracker derive from gun-tip
            // motion and sent the lead in the gun's facing direction).
            // Always run the scan — if H.actor was previously set to the
            // weapon key, the guard `me[H.actor].mesh` would otherwise pass
            // and the bad key would stick.
            const actorKey = findKeyWithProperty(me, H.mesh, [H.weapon]);
            if (actorKey && actorKey !== H.actor) {
                log('actor key resolved:', H.actor, '→', actorKey);
                H.actor = actorKey;
            }
 
            // One-time scene tweak: clear the depth buffer between rendering
            // group 0 (world) and group 1 (our boxes/tracers).
            if (ss.SCENE && !ss.SCENE._ssh_depthCleared && typeof BABYLON !== 'undefined') {
                try {
                    ss.SCENE.setRenderingAutoClearDepthStencil(1, true, true, true);
                    ss.SCENE._ssh_depthCleared = true;
                    log('depth-clear enabled for renderingGroupId=1 (see through walls)');
                } catch (e) { log('depth-clear setup failed:', e && e.message); }
            }
 
            // Crosshair convergence point: 5 units in front of the camera, so
            // tracer lines from each enemy visually fan out from the center of
            // your view.
            let crosshairs = null;
            const cur = getCurrentYawPitch();
            if (cur && typeof BABYLON !== 'undefined' && me[H.actor] && me[H.actor][H.mesh]) {
                crosshairs = new BABYLON.Vector3();
                crosshairs.copyFrom(me[H.actor][H.mesh].position);
                crosshairs.y += 0.4;
                const yaw = cur.yaw;
                const pitch = -cur.pitch;
                const off = -5;
                crosshairs.x += Math.sin(yaw) * Math.cos(pitch) * off;
                crosshairs.y += Math.sin(pitch) * off;
                crosshairs.z += Math.cos(yaw) * Math.cos(pitch) * off;
            }
 
            // ── ESP cleanup sweep ──
            // The main loop below only visits players who are STILL valid
            // enemies. Anyone who left the match (gone from ss.PLAYERS) or
            // switched onto our team would otherwise keep an orphan box at
            // their last position forever. Build the set of who should
            // currently have ESP, then dispose everything else we're tracking.
            const _shouldHaveEsp = new Set();
            for (const P of ss.PLAYERS) {
                if (!P || P === me) continue;
                if (me.team !== 0 && P.team === me.team) continue;
                _shouldHaveEsp.add(P);
            }
            for (const P of _espPlayers) {
                if (!_shouldHaveEsp.has(P)) disposeEspFor(P);
            }
 
            // ── ESP boxes + tracers + velocity tracking ──
            for (const P of ss.PLAYERS) {
                if (!P || P === me) continue;
                if (me.team !== 0 && P.team === me.team) continue;
                if (!P[H.playing]) {
                    if (P._ssh_box)    P._ssh_box.visibility    = 0;
                    if (P._ssh_tracer) P._ssh_tracer.visibility = 0;
                    velState.delete(P);
                    continue;
                }
                // Track velocity every frame so prediction is primed the
                // instant RMB goes down.
                trackVelocity(P, nowMs);
 
                // Recolor based on whether this enemy is a Nyx user. Cache
                // the last-applied state on the player so we only touch the
                // mesh color when it actually changes (cheap most frames).
                const nyx = _isNyxName(P.name);
                const desiredColor = nyx ? ESP_COLOR_NYX : ESP_COLOR_RED;
 
                const box = ensureEspBox(P);
                if (box) {
                    box.position.x = P[H.x];
                    box.position.y = P[H.y];
                    box.position.z = P[H.z];
                    box.visibility = settings.espEnabled ? 1 : 0;
                    if (P._ssh_nyxColor !== nyx) {
                        box.color = desiredColor;
                        P._ssh_nyxColor = nyx;
                    }
                }
                if (crosshairs) {
                    const tr = updateEspTracer(P, crosshairs);
                    if (tr) {
                        tr.visibility = settings.espEnabled ? 1 : 0;
                        if (tr._ssh_nyxColor !== nyx) {
                            tr.color = desiredColor;
                            tr._ssh_nyxColor = nyx;
                        }
                    }
                }
            }
 
            // Reset per-frame state read by the overlay + no-spread helper.
            _aim.hasLock = false;
            _aim.me = me;
            // Clear aim-smoothing memory when RMB is up so the next press
            // snaps onto target instantly instead of easing in from a stale
            // cached point.
            if (!RMB) _aim.smoothTarget = null;
            _pred.enabled = !!settings.predEnabled;
            _pred.active = false;
            _pred.speed = 0; _pred.t = 0; _pred.leadDist = 0;
            _pred.projSpeed = getProjectileSpeed(me);
 
            // ── Aim: RMB held + aimbot enabled → snap to closest enemy ──
            if (RMB && settings.aimEnabled) {
                const meMesh = me[H.actor] && me[H.actor][H.mesh];
                if (meMesh && meMesh.position) {
                    const meP = meMesh.position; // local-player mesh.y = head/eye
 
                    // Target selection. Two modes, chosen by the
                    // `crosshairTarget` setting:
                    //   • OFF (default) — closest enemy by 3D distance.
                    //   • ON  — enemy nearest the crosshair, i.e. smallest
                    //     angle between my camera-forward vector and the
                    //     (me → enemy) direction. Picks whoever I'm already
                    //     pointing closest to, regardless of distance.
                    // Both are tracked in one pass; the active mode just
                    // decides which winner we use.
                    let bestClose = null;
                    let bestCloseDist = Infinity;
                    let bestCloseMeshPos = null;
 
                    // Camera-forward unit vector for crosshair mode. Inverse of
                    // the yaw/pitch→direction formula used below for aiming:
                    // forward = (-sin(yaw)cos(pitch), sin(pitch), -cos(yaw)cos(pitch)).
                    let bestCross = null;
                    let bestCrossDot = -Infinity;
                    let bestCrossMeshPos = null;
                    let fwdX = 0, fwdY = 0, fwdZ = 0, haveFwd = false;
                    if (settings.crosshairTarget) {
                        const cur = getCurrentYawPitch();
                        if (cur) {
                            const cp = Math.cos(cur.pitch);
                            fwdX = -Math.sin(cur.yaw) * cp;
                            fwdY =  Math.sin(cur.pitch);
                            fwdZ = -Math.cos(cur.yaw) * cp;
                            haveFwd = true;
                        }
                    }
 
                    for (const P of ss.PLAYERS) {
                        if (!P || P === me) continue;
                        if (!P[H.playing]) continue;
                        if (me.team !== 0 && P.team === me.team) continue;
                        const pMesh = P[H.actor] && P[H.actor][H.mesh];
                        if (!pMesh || !pMesh.position) continue;
                        const pp = pMesh.position; // enemy mesh.y = body center → good aim point
                        const dx = meP.x - pp.x;
                        const dy = meP.y - pp.y;
                        const dz = meP.z - pp.z;
                        const d = Math.hypot(dx, dy, dz);
                        if (d <= 0) continue;
                        if (d < bestCloseDist) {
                            bestCloseDist = d; bestClose = P; bestCloseMeshPos = pp;
                        }
                        // Crosshair proximity: dot of camera-forward with the
                        // (me → enemy) unit vector. Higher dot = smaller angle.
                        if (haveFwd) {
                            const dot = (fwdX * -dx + fwdY * -dy + fwdZ * -dz) / d;
                            if (dot > bestCrossDot) {
                                bestCrossDot = dot; bestCross = P; bestCrossMeshPos = pp;
                            }
                        }
                    }
                    const useCross    = settings.crosshairTarget && bestCross;
                    const best        = useCross ? bestCross        : bestClose;
                    const bestMeshPos = useCross ? bestCrossMeshPos : bestCloseMeshPos;
 
                    if (best && bestMeshPos) {
                        // Pick the aim point. With prediction ON, use the
                        // mesh-interpolated body position and apply the
                        // single-pass gravity-aware lead solve (predictPosition).
                        // With prediction OFF, snap to the server-authoritative
                        // network coords — these don't bob with the running
                        // animation, so the crosshair stays locked to dead center.
                        let aimX, aimY, aimZ;
                        if (settings.predEnabled) {
                            aimX = bestMeshPos.x;
                            aimY = bestMeshPos.y;
                            aimZ = bestMeshPos.z;
                            const v = velState.get(best);
                            if (v && v.primed) {
                                const pred = predictPosition(bestMeshPos, v, meP, _pred.projSpeed);
                                aimX = pred.x; aimY = pred.y; aimZ = pred.z;
                                _pred.active = true;
                                _pred.speed = Math.hypot(v.vx, v.vy, v.vz);
                                _pred.leadDist = Math.hypot(aimX - bestMeshPos.x, aimZ - bestMeshPos.z);
                                const distToPred = Math.hypot(aimX - meP.x, aimY - meP.y, aimZ - meP.z);
                                _pred.t = Math.min(distToPred / _pred.projSpeed + TICK, PREDICTION_MAX_T);
                            }
                        } else {
                            aimX = best[H.x];
                            aimY = best[H.y];
                            aimZ = best[H.z];
                        }
 
                        // Final aim-point smoothing — absorbs the tick-boundary
                        // velocity steps that otherwise make the predicted point
                        // (and therefore the camera) jitter every ~50 ms. Reset
                        // when the target changes so the initial snap onto a
                        // new target is instant (user rejects slow-initial-snap
                        // behavior — see memory).
                        const AIM_SMOOTH = 0.35;
                        if (_aim.smoothTarget !== best) {
                            _aim.smoothTarget = best;
                            _aim.smoothX = aimX; _aim.smoothY = aimY; _aim.smoothZ = aimZ;
                        } else {
                            _aim.smoothX += (aimX - _aim.smoothX) * AIM_SMOOTH;
                            _aim.smoothY += (aimY - _aim.smoothY) * AIM_SMOOTH;
                            _aim.smoothZ += (aimZ - _aim.smoothZ) * AIM_SMOOTH;
                            aimX = _aim.smoothX; aimY = _aim.smoothY; aimZ = _aim.smoothZ;
                        }
 
                        // Compute target yaw/pitch in get_yaw_pitch() convention. The
                        // negated direction-vector + atan2 formulas match babylon.js's
                        // F.calculateYaw / F.calculatePitch verbatim.
                        const negDx = -(aimX - meP.x);
                        const negDy = -(aimY - meP.y);
                        const negDz = -(aimZ - meP.z);
                        const targetYaw   = Math.atan2(negDx, negDz) % (2 * Math.PI);
                        let   targetPitch = -Math.atan2(negDy, Math.hypot(negDx, negDz));
                        if (targetPitch >  1.5) targetPitch =  1.5;
                        if (targetPitch < -1.5) targetPitch = -1.5;
 
                        _aim.hasLock = true;
 
                        // A single big synthetic movement gets clamped by the game's
                        // mouse-input pipeline; several smaller ones converge cleanly.
                        for (let i = 0; i < 5; i++) setToYawPitch(targetYaw, targetPitch);
                    }
                }
            }
 
            // Item ESP sweep — runs every frame so markers track items that
            // existed before our spawnItem hook installed (pre-join state),
            // and cleans up markers whose item became inactive via any
            // code path the collectItem hook didn't catch.
            if (settings.itemEsp && ss.items && ss.items.pools && typeof BABYLON !== 'undefined') {
                const seen = new Set();
                for (let t = 0; t < ss.items.pools.length; t++) {
                    const pool = ss.items.pools[t];
                    if (!pool || typeof pool.forEachActive !== 'function') continue;
                    const poolColor = (t === 0 ? ESP_COLOR_AMMO : ESP_COLOR_GRENADE);
                    pool.forEachActive((it) => {
                        if (!it || !it.mesh) return;
                        seen.add(it.mesh);
                        if (!itemMarkers.has(it.mesh)) {
                            const pos = it.mesh.position;
                            const m = makeItemMarker(pos.x, pos.y, pos.z, poolColor);
                            if (m) itemMarkers.set(it.mesh, m);
                        }
                    });
                }
                for (const [meshKey, marker] of itemMarkers) {
                    if (!seen.has(meshKey)) {
                        try { marker.dispose(); } catch (e) {}
                        itemMarkers.delete(meshKey);
                    }
                }
            } else if (!settings.itemEsp && itemMarkers.size > 0) {
                for (const marker of itemMarkers.values()) {
                    try { marker.dispose(); } catch (e) {}
                }
                itemMarkers.clear();
            }
 
            updateNyxBanner();
            return false;
        } catch (e) {
            log('per-frame error:', e && e.message);
            return false;
        }
    };
 
    // ────────────────────────────────────────────────────────────────────────
    // NOTES
    // ────────────────────────────────────────────────────────────────────────
    // • Hold RMB → aim at the closest enemy by 3D distance, or the one nearest
    //   the crosshair when "Target Crosshair" is enabled.
    // • Press V → toggle red wireframe ESP on enemies.
    // • Press ` → show/hide the settings menu (top-left).
    // • Aim target + velocity are read from mesh.position (interpolated), not
    //   from the stepped network coords on the player object.
    // • Prediction is ported from "other script": single-pass lead
    //   (t = dist/projSpeed + 1 tick), with a gravity + terminal-velocity
    //   vertical drop model and a ground clamp. Gravity/terminal are per-tick
    //   game constants converted to per-second via the 30 Hz TICK_RATE.
    // • Lead time is in seconds. Velocity smoothing is an EMA factor where
    //   0 = use raw instantaneous velocity, 1 = heavily filtered.
    // • If aim under/overshoots, your in-game mouse sensitivity differs from
    //   the menu's "Sensitivity" value (default 0.0025); tweak it.
})();