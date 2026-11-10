// ==UserScript==
// @name         Ops Portal Delegation Helper
// @namespace    https://ads.opsportal.annotation.amazon.dev/
// @version      3.7.0
// @description  Inline helpers on the Ops Portal delegation form: pick the delegate manager from a dropdown next to "Whom to delegate", a one-click "Till end of day" button next to "Time range", and a list of recent delegations (with the DAs involved) you can re-apply with one click.
// @author       gexu
// @match        https://ads.opsportal.annotation.amazon.dev/*
// @match        https://*.annotation.amazon.dev/*
// @grant        GM_setValue
// @grant        GM_getValue
// @run-at       document-idle
// ==/UserScript==
//
// The REAL "Create Delegation Request" form is rendered in a (cross-origin)
// iframe from delegation.services.annotation.amazon.dev — the ads.opsportal.*
// page is only a shell. This script matches every annotation.amazon.dev origin
// and does NOT use @noframes, so it runs INSIDE the form frame and injects its
// controls directly next to the native fields (no floating panel, no cross-frame
// messaging — everything is same-document DOM).
//
// Form layout (confirmed live):
//   "Who to delegate"  -> a list of CHECKBOXES, one per DA alias (+ "Select All").
//   "Whom to delegate" -> a TEXTAREA, placeholder "Please enter user Alias
//                         (lowercase only)" — the MANAGER login.
//   "Time range"       -> a Cloudscape date-range picker. The collapsed trigger
//                         is a <button> reading "Filter by a date and time range".
//                         Opening it shows a "Relative range | Absolute range"
//                         tab set; Absolute has Start/End date (YYYY/MM/DD) and
//                         Start/End time (hh:mm:ss) inputs plus "Apply".

(function () {
  'use strict';

  // ---------------------------------------------------------------------------
  // Config
  // ---------------------------------------------------------------------------
  const MANAGER_LOGINS = [
    'depalama', 'mcwaic', 'cgnemani', 'gexu', 'ggngray', 'jonrebel', 'ohota', 'svvarana',
  ];

  const MEMORY_KEY = 'opsportal_delegation_memory_v1';
  const MEMORY_MAX = 5;

  // Quick time-range types and their chip labels.
  const TYPE_LABEL = {
    eod: 'Till end of day',
    tomorrow: 'Till tomorrow',
    eow: 'Till end of week',
  };

  const UI_ID = 'dh-inline';
  const DEBUG = true;
  const log = (...a) =>
    DEBUG && console.log('%c[DelegationHelper]', 'color:#ff9900;font-weight:bold', ...a);

  // ---------------------------------------------------------------------------
  // Small helpers
  // ---------------------------------------------------------------------------
  const pad = (n) => String(n).padStart(2, '0');
  const fmtDate = (d) => `${d.getFullYear()}/${pad(d.getMonth() + 1)}/${pad(d.getDate())}`;
  const fmtTime = (d) => `${pad(d.getHours())}:${pad(d.getMinutes())}:${pad(d.getSeconds())}`;
  const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
  const text = (el) => (el ? el.textContent.replace(/\s+/g, ' ').trim() : '');

  function debounce(fn, ms) {
    let t;
    return function (...args) {
      clearTimeout(t);
      t = setTimeout(() => fn.apply(this, args), ms);
    };
  }

  // All quick ranges start at now + 1 min; only the end differs.
  function rangeStart() {
    const start = new Date();
    start.setMinutes(start.getMinutes() + 1);
    start.setSeconds(0, 0);
    return start;
  }
  function endOfDay(d) {
    const e = new Date(d);
    e.setHours(23, 59, 59, 0);
    return e;
  }
  // Build a quick range by type: eod=today, tomorrow=+1 day, eow=this Sunday.
  function quickRange(type) {
    const start = rangeStart();
    let end;
    if (type === 'tomorrow') {
      const d = new Date(start); d.setDate(d.getDate() + 1); end = endOfDay(d);
    } else if (type === 'eow') {
      const d = new Date(start);
      // Days until Sunday (end of week). If already Sunday, stay today.
      const toSunday = (7 - d.getDay()) % 7;
      d.setDate(d.getDate() + toSunday);
      end = endOfDay(d);
    } else {
      end = endOfDay(start); // 'eod'
    }
    return { start, end };
  }

  // ---------------------------------------------------------------------------
  // React-aware native field writer
  // ---------------------------------------------------------------------------
  function setNativeValue(el, value) {
    if (!el) return false;
    const proto =
      el.tagName === 'TEXTAREA' ? window.HTMLTextAreaElement.prototype
      : el.tagName === 'SELECT' ? window.HTMLSelectElement.prototype
      : window.HTMLInputElement.prototype;
    const setter = Object.getOwnPropertyDescriptor(proto, 'value');
    if (setter && setter.set) setter.set.call(el, value);
    else el.value = value;
    el.dispatchEvent(new Event('input', { bubbles: true }));
    el.dispatchEvent(new Event('change', { bubbles: true }));
    return true;
  }

  // A "real" click. NOTE: do NOT pass `view: window` — in a cross-origin frame
  // the window ref fails the MouseEvent constructor and throws.
  function clickReal(el) {
    const opts = { bubbles: true, cancelable: true };
    try { el.dispatchEvent(new PointerEvent('pointerdown', opts)); } catch (e) {
      try { el.dispatchEvent(new MouseEvent('pointerdown', opts)); } catch (e2) {}
    }
    try { el.dispatchEvent(new MouseEvent('mousedown', opts)); } catch (e) {}
    try { el.dispatchEvent(new MouseEvent('mouseup', opts)); } catch (e) {}
    try { el.click(); } catch (e) {}
    el.dispatchEvent(new Event('input', { bubbles: true }));
    el.dispatchEvent(new Event('change', { bubbles: true }));
  }

  function typeInto(el, value) {
    el.focus();
    setNativeValue(el, value);
    el.dispatchEvent(new KeyboardEvent('keyup', { bubbles: true }));
    el.dispatchEvent(new Event('blur', { bubbles: true }));
  }

  function labelFor(input) {
    const aria = input.getAttribute('aria-label');
    if (aria) return aria;
    const ph = input.getAttribute('placeholder');
    if (ph) return ph;
    if (input.id) {
      const lbl = document.querySelector(`label[for="${CSS.escape(input.id)}"]`);
      if (lbl) return text(lbl);
    }
    const field = input.closest('[class*="form-field"], div');
    if (field) {
      const lbl = field.querySelector('label, [class*="label"]');
      if (lbl) return text(lbl);
    }
    return '';
  }

  // ---------------------------------------------------------------------------
  // Native-form field discovery
  // ---------------------------------------------------------------------------
  // The specific "Whom to delegate" textarea, identified by its placeholder
  // ("Please enter user Alias (lowercase only)") / delegate-ish label. NO generic
  // fallback — other pages (e.g. One-on-One notes) also have textareas, and we
  // must not grab those.
  function findManagerInput() {
    const tas = Array.from(document.querySelectorAll('textarea'));
    const re = /(enter user alias|lowercase only|whom to delegate|delegate)/i;
    return tas.find((t) => re.test((t.placeholder || '') + ' ' + labelFor(t))) || null;
  }

  // Is THIS document the real delegation form? Require the specific manager
  // textarea AND the delegation URL/heading — a bare textarea+checkbox combo
  // (which the One-on-One notes page also has) is NOT enough.
  function looksLikeDelegationForm() {
    if (!findManagerInput()) return false;
    // Path or on-page heading must mention delegation.
    const onDelegationPath = /delegation/i.test(location.href);
    const hasHeading = /create delegation request|delegation/i.test(
      document.body ? document.body.innerText.slice(0, 4000) : ''
    );
    const hasCheckboxes = document.querySelectorAll('input[type="checkbox"]').length > 0;
    return (onDelegationPath || hasHeading) && hasCheckboxes;
  }

  // The collapsed date-range trigger button.
  function findTimeTrigger() {
    const re = /date and time range|time range|filter by a date/i;
    let el = Array.from(document.querySelectorAll('button, [role="button"], [role="combobox"], input'))
      .find((e) =>
        re.test(text(e) + ' ' + (e.getAttribute('placeholder') || '') + ' ' + (e.getAttribute('aria-label') || ''))
      );
    if (el) return el;
    const leaf = Array.from(document.querySelectorAll('*'))
      .filter((e) => e.childElementCount === 0)
      .find((e) => re.test(text(e)));
    return leaf ? (leaf.closest('button, [role="button"]') || leaf.parentElement || leaf) : null;
  }

  function findByText(candidates, sel) {
    const els = Array.from(document.querySelectorAll(sel));
    const wanted = candidates.map((c) => c.toLowerCase());
    return (
      els.find((e) => wanted.includes(text(e).toLowerCase())) ||
      els.find((e) => wanted.some((w) => text(e).toLowerCase().includes(w))) ||
      null
    );
  }

  // Resolve the alias text for a DA checkbox (alias lives in the row, not the
  // <input>). Walk up progressively until we find text.
  function checkboxLabelText(box) {
    if (box.id) {
      const l = document.querySelector(`label[for="${CSS.escape(box.id)}"]`);
      if (l && text(l)) return text(l);
    }
    const lab = box.closest('label');
    if (lab && text(lab)) return text(lab);
    const wrap = box.closest('[class*="checkbox"]');
    if (wrap) {
      if (text(wrap)) return text(wrap);
      if (wrap.parentElement && text(wrap.parentElement)) return text(wrap.parentElement);
    }
    let el = box.parentElement;
    for (let i = 0; i < 4 && el; i++) {
      if (text(el)) return text(el);
      el = el.parentElement;
    }
    return '';
  }

  // Tick the DA checkboxes matching the given aliases (and untick others that
  // aren't in the list, so re-applying a chip reproduces exactly its DA set).
  function tickDAs(aliases) {
    const want = new Set((aliases || []).map((a) => String(a).trim().toLowerCase()));
    const boxes = Array.from(document.querySelectorAll('input[type="checkbox"]'))
      .filter((b) => !b.closest('#' + UI_ID));
    let matched = 0;
    boxes.forEach((b) => {
      const rowText = checkboxLabelText(b).trim().toLowerCase();
      if (!rowText || /select all|permanent/.test(rowText)) return;
      const alias = rowText.split(/[^a-z0-9]+/).filter(Boolean)[0] || rowText;
      const shouldCheck = want.has(alias);
      if (shouldCheck && !b.checked) { clickReal(b); matched++; }
      else if (!shouldCheck && b.checked) { clickReal(b); } // clear stale picks
      else if (shouldCheck) { matched++; }
    });
    log('tickDAs: wanted', [...want], 'matched', matched);
    return matched;
  }

  // Aliases of the currently-ticked DA checkboxes.
  function getCheckedDAs() {
    return Array.from(document.querySelectorAll('input[type="checkbox"]'))
      .filter((b) => b.checked && !b.closest('#' + UI_ID))
      .map((b) => checkboxLabelText(b).trim().toLowerCase())
      .filter((t) => t && !/select all|permanent/.test(t))
      .map((t) => t.split(/[^a-z0-9]+/).filter(Boolean)[0] || t)
      .filter((v, i, arr) => arr.indexOf(v) === i);
  }

  // ---------------------------------------------------------------------------
  // Form actions
  // ---------------------------------------------------------------------------
  function setManager(login) {
    const el = findManagerInput();
    if (!el) { log('manager textarea not found'); return false; }
    el.focus();
    setNativeValue(el, login);
    el.dispatchEvent(new Event('blur', { bubbles: true }));
    return true;
  }

  // Drive the date-range popover: open -> Absolute tab -> fill 4 inputs -> Apply.
  async function setTimeRange(start, end) {
    const trig = findTimeTrigger();
    if (!trig) { log('time: trigger not found'); return false; }
    clickReal(trig);
    await sleep(350);

    const absBtn = findByText(['Absolute range', 'Absolute'], 'button, [role="tab"], a, span, div');
    if (absBtn) { clickReal(absBtn.closest('button, [role="tab"], a') || absBtn); await sleep(250); }

    const inputs = Array.from(document.querySelectorAll('input')).filter((i) => !i.closest('#' + UI_ID));
    const dateInputs = inputs.filter((i) => /yyyy[\/-]?mm[\/-]?dd/i.test(i.placeholder || ''));
    const timeInputs = inputs.filter((i) => /hh:mm:ss/i.test(i.placeholder || ''));
    if (!dateInputs.length && !timeInputs.length) { log('time: popover inputs not found'); return false; }

    if (dateInputs[0]) typeInto(dateInputs[0], fmtDate(start));
    if (timeInputs[0]) typeInto(timeInputs[0], fmtTime(start));
    if (dateInputs[1]) typeInto(dateInputs[1], fmtDate(end));
    if (timeInputs[1]) typeInto(timeInputs[1], fmtTime(end));
    await sleep(150);

    const applyBtn = findByText(['Apply'], 'button, [role="button"], span');
    if (applyBtn) { clickReal(applyBtn.closest('button, [role="button"]') || applyBtn); await sleep(120); }
    return true;
  }

  // ---------------------------------------------------------------------------
  // Memory (recent delegations) — activity based
  // ---------------------------------------------------------------------------
  function loadMemory() {
    const raw = GM_getValue(MEMORY_KEY, null);
    return Array.isArray(raw) ? raw : [];
  }
  function saveMemory(list) {
    GM_setValue(MEMORY_KEY, list.slice(0, MEMORY_MAX));
  }

  // entry: { manager, type, das[], ts }
  // One chip per activity = manager + type + the exact SET of DAs delegated
  // together. Different DAs (even same manager/type) are a different activity, so
  // they get their own chip; re-submitting the identical set just refreshes it.
  function rememberDelegation(entry) {
    const list = loadMemory();
    const key = (e) =>
      [e.manager, e.type || '', (e.das || []).slice().sort().join(',')].join('|');
    const filtered = list.filter((e) => key(e) !== key(entry));
    filtered.unshift(entry);
    saveMemory(filtered);
    return filtered;
  }

  // ---------------------------------------------------------------------------
  // Inline UI injection
  // ---------------------------------------------------------------------------
  let currentManager = '';
  let lastTimeType = ''; // 'eod' once "Till end of day" is applied this session

  function styleOnce() {
    if (document.getElementById(UI_ID + '-style')) return;
    const s = document.createElement('style');
    s.id = UI_ID + '-style';
    s.textContent = `
      .dh-wrap { margin:6px 0 2px; font:inherit; }
      .dh-inline-pick { display:inline-flex; align-items:center; gap:6px; margin-left:12px; font:inherit; font-weight:400; }
      .dh-row { display:flex; align-items:center; gap:8px; flex-wrap:wrap; }
      .dh-lbl { font-weight:600; color:#5f6b7a; font-size:12px; }
      .dh-sel { padding:5px 8px; border:1px solid #8c9196; border-radius:6px; background:#fff; color:inherit; font:inherit; }
      .dh-btn { padding:5px 12px; border:0; border-radius:6px; background:#0972d3; color:#fff; font:inherit; font-weight:700; cursor:pointer; }
      .dh-btn:hover { background:#0858a8; }
      .dh-note { font-size:11px; color:#5f6b7a; }
      .dh-ok { color:#037f0c; font-size:12px; }
      .dh-err { color:#d13212; font-size:12px; }
      .dh-mem { margin-top:8px; }
      .dh-mem h4 { margin:0 0 6px; font-size:12px; color:#5f6b7a; font-weight:700; }
      .dh-chip { display:flex; justify-content:space-between; align-items:flex-start;
        border:1px solid #d5dbdb; border-radius:6px; padding:6px 8px; margin-bottom:6px; gap:8px; cursor:pointer; max-width:560px; }
      .dh-chip:hover { background:#f2f8fd; border-color:#0972d3; }
      .dh-chip .t { font-size:11px; color:#5f6b7a; }
      .dh-chip .x { color:#d13212; font-weight:700; padding:0 4px; }
    `;
    document.head.appendChild(s);
  }

  // Insert `node` right after the form field that contains `anchorEl`.
  function insertAfterField(anchorEl, node) {
    const field = anchorEl.closest('[class*="form-field"]') || anchorEl.parentElement;
    if (field && field.parentElement) {
      field.parentElement.insertBefore(node, field.nextSibling);
    } else {
      anchorEl.parentElement.appendChild(node);
    }
  }

  // Manager dropdown placed to the RIGHT of the "Whom to delegate" label,
  // ABOVE the textarea box.
  function injectManagerDropdown() {
    const ta = findManagerInput();
    if (!ta || document.getElementById('dh-mgr-wrap')) return !!document.getElementById('dh-mgr-wrap');

    const wrap = document.createElement('span');
    wrap.id = 'dh-mgr-wrap';
    wrap.className = 'dh-inline-pick';
    wrap.innerHTML = `
      <select class="dh-sel" id="dh-mgr-sel">
        <option value="">— select manager —</option>
        ${MANAGER_LOGINS.map((m) => `<option value="${m}">${m}</option>`).join('')}
      </select>`;

    // Find the field label for this textarea and append the picker to its row so
    // it sits to the right of the "Whom to delegate" heading, above the box.
    const label = findFieldLabel(ta);
    if (label) {
      // Wrap the label text + picker in a flex row for right-alignment.
      label.style.display = 'inline-flex';
      label.style.alignItems = 'center';
      label.style.gap = '10px';
      label.appendChild(wrap);
    } else {
      // Fallback: insert just before the textarea.
      ta.parentElement.insertBefore(wrap, ta);
    }

    wrap.querySelector('#dh-mgr-sel').addEventListener('change', (ev) => {
      const login = ev.target.value;
      if (!login) return;
      currentManager = login;
      setManager(login);
    });
    return true;
  }

  // The Cloudscape form-field label element above a control.
  function findFieldLabel(control) {
    if (control.id) {
      const l = document.querySelector(`label[for="${CSS.escape(control.id)}"]`);
      if (l) return l;
      // Cloudscape often uses aria-labelledby -> an element id "...-label".
      const by = control.getAttribute('aria-labelledby');
      if (by) {
        const el = document.getElementById(by);
        if (el) return el;
      }
    }
    const field = control.closest('[class*="form-field"]');
    if (field) {
      return field.querySelector('label, [class*="label"], [id$="-label"]');
    }
    return null;
  }

  // "Till end of day" button + recent delegations under "Time range".
  function injectTimeControls() {
    const trig = findTimeTrigger();
    if (!trig || document.getElementById('dh-time-wrap')) return !!document.getElementById('dh-time-wrap');

    const wrap = document.createElement('div');
    wrap.id = 'dh-time-wrap';
    wrap.className = 'dh-wrap';
    wrap.innerHTML = `
      <div class="dh-row">
        <button type="button" class="dh-btn" data-type="eod">Till end of day</button>
        <button type="button" class="dh-btn" data-type="tomorrow">Till tomorrow</button>
        <button type="button" class="dh-btn" data-type="eow">Till end of week</button>
        <span id="dh-status"></span>
      </div>
      <div class="dh-mem" id="dh-mem">
        <h4>Recent delegations</h4>
        <div id="dh-mem-list"></div>
      </div>`;
    insertAfterField(trig, wrap);

    wrap.querySelectorAll('.dh-btn[data-type]').forEach((b) =>
      b.addEventListener('click', () => applyQuick(b.getAttribute('data-type')))
    );
    renderMemory();
    return true;
  }

  function setStatus(msg, ok) {
    const el = document.getElementById('dh-status');
    if (!el) return;
    el.textContent = msg || '';
    el.className = ok ? 'dh-ok' : 'dh-err';
  }

  async function applyQuick(type) {
    const { start, end } = quickRange(type);
    setStatus('Applying…', true);
    const ok = await setTimeRange(start, end);
    setStatus(ok ? '✓ Time range set' : 'Could not set the time range', ok);
    // Note the chosen time type; the activity is only RECORDED on Submit.
    if (ok) lastTimeType = type;
  }

  function renderMemory(list) {
    const box = document.getElementById('dh-mem-list');
    if (!box) return;
    list = list || loadMemory();
    if (!list.length) {
      box.innerHTML = '<div class="dh-note">No saved delegations yet.</div>';
      return;
    }
    box.innerHTML = '';
    list.forEach((e, i) => {
      const chip = document.createElement('div');
      chip.className = 'dh-chip';
      // Show the ACTION TYPE, not a stale absolute range (the actual time shifts
      // every run).
      const when = TYPE_LABEL[e.type] || e.type || '(custom range)';
      const das = (e.das && e.das.length)
        ? `<div class="t" style="margin-top:2px;">DAs: ${e.das.join(', ')}</div>` : '';
      chip.innerHTML =
        `<div><div><b>→ ${e.manager}</b></div><div class="t">${when}</div>${das}</div>` +
        `<div class="x" title="remove">✕</div>`;
      chip.addEventListener('click', (ev) => {
        if (ev.target.classList.contains('x')) {
          const next = loadMemory();
          next.splice(i, 1);
          saveMemory(next);
          renderMemory(next);
          return;
        }
        applyMemoryEntry(e);
      });
      box.appendChild(chip);
    });
  }

  // Re-apply a saved activity: tick its DAs, set the manager, then the time range.
  async function applyMemoryEntry(e) {
    currentManager = e.manager || '';
    const sel = document.getElementById('dh-mgr-sel');
    if (sel) sel.value = MANAGER_LOGINS.includes(e.manager) ? e.manager : '';
    tickDAs(e.das || []);
    setManager(e.manager);
    // Re-apply the saved type as a fresh range.
    await applyQuick(e.type || 'eod');
  }

  // (DAs are captured at submit time, so no live checkbox watcher is needed.)

  // Record the activity ONLY when the user actually submits the request. We
  // capture the manager, the chosen time type, and the DAs at submit time.
  function installSubmitWatcher() {
    document.addEventListener('click', (ev) => {
      const btn = ev.target.closest('button, [role="button"], input[type="submit"]');
      if (!btn || btn.closest('#' + UI_ID)) return;
      if (!/^submit$/i.test(text(btn)) && !/submit/i.test(btn.getAttribute('aria-label') || '')) return;

      const manager = (findManagerInput() && findManagerInput().value.trim()) || currentManager;
      if (!manager) return; // nothing meaningful to record

      // Read DAs NOW; if the form already cleared them on submit, fall back to
      // the last set we tracked while the user was ticking boxes.
      let das = getCheckedDAs();
      if (!das.length && lastCheckedDAs.length) das = lastCheckedDAs.slice();

      renderMemory(rememberDelegation({
        manager,
        type: lastTimeType || 'eod',
        das,
        ts: Date.now(),
      }));
      log('recorded delegation on submit:', manager, lastTimeType, 'DAs:', das);
    }, true); // capture phase, so we record even though the SPA may navigate away
  }

  // Track the DA set as the user ticks/unticks boxes, so we still have it at
  // submit even if the form clears the checkboxes on click.
  let lastCheckedDAs = [];
  function installDaTracker() {
    document.addEventListener('change', (ev) => {
      const t = ev.target;
      if (t && t.tagName === 'INPUT' && t.type === 'checkbox' && !t.closest('#' + UI_ID)) {
        const das = getCheckedDAs();
        if (das.length) lastCheckedDAs = das;
      }
    }, true);
  }

  // ---------------------------------------------------------------------------
  // Boot
  // ---------------------------------------------------------------------------
  function tryInject() {
    if (!looksLikeDelegationForm()) { removeInjected(); return false; }
    styleOnce();
    const a = injectManagerDropdown();
    const b = injectTimeControls();
    return a && b;
  }

  // Remove our injected controls (used when the SPA navigates away from the
  // delegation form so the dropdown doesn't leak onto other pages).
  function removeInjected() {
    ['dh-mgr-wrap', 'dh-time-wrap'].forEach((id) => {
      const el = document.getElementById(id);
      if (el) el.remove();
    });
  }

  function boot() {
    // Only act in the frame that actually has the form. The SPA mounts late and
    // can re-render, so keep trying and re-inject (or clean up) as it navigates.
    const mo = new MutationObserver(debounce(() => {
      if (!looksLikeDelegationForm()) {
        removeInjected();
      } else if (!document.getElementById('dh-mgr-wrap') || !document.getElementById('dh-time-wrap')) {
        tryInject();
      }
    }, 400));
    mo.observe(document.documentElement, { childList: true, subtree: true });

    installSubmitWatcher();
    installDaTracker();

    let tries = 0;
    const iv = setInterval(() => {
      tries += 1;
      if (tryInject() || tries > 80) clearInterval(iv);
    }, 400);

    log('booted on', location.host, '| in frame:', window.top !== window.self);
  }

  boot();
})();
