// ==UserScript==
// @name         Amazon Business Prime - Query Counter
// @namespace    http://tampermonkey.net/
// @version      2.7
// @description  Counts number of bot queries reviewed and submitted via feedback panel
// @author       arvindon
// @match        https://pre-prod.amazon.com/businessprime*
// @grant        GM_setValue
// @grant        GM_getValue
// ==/UserScript==

(function () {
  'use strict';

  // ─── Load persisted count ───────────────────────────────────────────────
  let queryCount = GM_getValue('queryCount', 0);
  let lastResetDate = GM_getValue('lastResetDate', null);

  const today = new Date().toDateString();
  if (lastResetDate !== today) {
    queryCount = 0;
    GM_setValue('queryCount', 0);
    GM_setValue('lastResetDate', today);
  }

  // ─── Create floating counter UI ─────────────────────────────────────────
  const counter = document.createElement('div');
  counter.id = 'query-counter';
  counter.innerHTML = `
    <div id="qc-header">📊 Query Counter</div>
    <div id="qc-body">
      <span id="qc-count">${queryCount}</span>
      <span id="qc-label"> queries today</span>
    </div>
    <button id="qc-reset">Reset</button>
  `;

  const style = document.createElement('style');
  style.textContent = `
    #query-counter {
      position: fixed;
      bottom: 20px;
      right: 20px;
      background: #232F3E;
      color: #fff;
      padding: 12px 16px;
      border-radius: 10px;
      font-family: Arial, sans-serif;
      font-size: 14px;
      z-index: 999999;
      box-shadow: 0 4px 12px rgba(0,0,0,0.4);
      min-width: 160px;
      text-align: center;
      cursor: move;
      user-select: none;
    }
    #qc-header {
      font-weight: bold;
      font-size: 13px;
      margin-bottom: 6px;
      color: #FF9900;
    }
    #qc-count {
      font-size: 32px;
      font-weight: bold;
      color: #FF9900;
      transition: color 0.3s ease;
    }
    #qc-label {
      font-size: 12px;
      color: #ccc;
    }
    #qc-reset {
      margin-top: 8px;
      background: #FF9900;
      border: none;
      color: #232F3E;
      padding: 4px 12px;
      border-radius: 5px;
      cursor: pointer;
      font-weight: bold;
      font-size: 12px;
      width: 100%;
      transition: background 0.2s ease;
    }
    #qc-reset:hover {
      background: #e68a00;
    }
    #qc-reset[data-confirming] {
      background: #cc0000;
      color: #fff;
    }
    #qc-reset[data-confirming]:hover {
      background: #aa0000;
    }
  `;

  document.head.appendChild(style);
  document.body.appendChild(counter);

  // ─── Normalize position to top/left immediately after mount ─────────────
  requestAnimationFrame(() => {
    const rect = counter.getBoundingClientRect();
    counter.style.top = rect.top + 'px';
    counter.style.left = rect.left + 'px';
    counter.style.bottom = 'auto';
    counter.style.right = 'auto';
  });

  // ─── Make counter draggable ──────────────────────────────────────────────
  let isDragging = false, offsetX, offsetY;

  counter.addEventListener('mousedown', (e) => {
    if (e.target.id === 'qc-reset') return;
    isDragging = true;
    offsetX = e.clientX - counter.getBoundingClientRect().left;
    offsetY = e.clientY - counter.getBoundingClientRect().top;
  });

  document.addEventListener('mousemove', (e) => {
    if (!isDragging) return;
    counter.style.left = `${e.clientX - offsetX}px`;
    counter.style.top = `${e.clientY - offsetY}px`;
  });

  document.addEventListener('mouseup', () => isDragging = false);

  // ─── Reset button — inline two-step confirm ──────────────────────────────
  document.getElementById('qc-reset').addEventListener('click', () => {
    const btn = document.getElementById('qc-reset');
    if (btn.dataset.confirming) {
      queryCount = 0;
      GM_setValue('queryCount', 0);
      document.getElementById('qc-count').textContent = 0;
      btn.textContent = 'Reset';
      delete btn.dataset.confirming;
    } else {
      btn.dataset.confirming = 'true';
      btn.textContent = 'Confirm?';
      setTimeout(() => {
        if (btn.dataset.confirming) {
          btn.textContent = 'Reset';
          delete btn.dataset.confirming;
        }
      }, 3000);
    }
  });

  // ─── Flash animation on count increment ──────────────────────────────────
  function updateCount() {
    queryCount++;
    GM_setValue('queryCount', queryCount);
    const countEl = document.getElementById('qc-count');
    countEl.textContent = queryCount;
    countEl.style.color = '#FF9900';
    countEl.style.transition = 'transform 0.2s ease';
    countEl.style.transform = 'scale(1.4)';
    setTimeout(() => countEl.style.transform = 'scale(1)', 300);
  }

  // ─── Flag: only listen for success toast after feedback Submit is clicked ─
  let feedbackSubmitClicked = false;
  let lastCountedTime = 0;

  // ─── Success toast watcher ────────────────────────────────────────────────
  // Only fires updateCount if feedback Submit was explicitly clicked first.
  // Resets the flag after 5 seconds whether or not a toast appeared.
  function startToastWatcher() {
    feedbackSubmitClicked = true;

    // Auto-reset flag after 5s in case toast never appears
    setTimeout(() => {
      feedbackSubmitClicked = false;
    }, 5000);
  }

  // ─── MutationObserver: watch for success toast in DOM ────────────────────
  const toastObserver = new MutationObserver((mutations) => {
    // Ignore toast entirely if feedback Submit was not clicked
    if (!feedbackSubmitClicked) return;

    for (const mutation of mutations) {
      for (const node of mutation.addedNodes) {
        if (node.nodeType !== Node.ELEMENT_NODE) continue;

        const allElements = [node, ...node.querySelectorAll('*')];
        for (const el of allElements) {
          const text = (el.innerText || el.textContent || '').trim().toLowerCase();
          if (text.includes('feedback submitted successfully')) {
            const now = Date.now();
            if (now - lastCountedTime > 2000) {
              lastCountedTime = now;
              feedbackSubmitClicked = false; // Reset flag immediately after counting
              updateCount();
            }
            return;
          }
        }
      }
    }
  });

  toastObserver.observe(document.body, { childList: true, subtree: true });

  // ─── MutationObserver: watch for feedback panel Submit button ────────────
  // Sets feedbackSubmitClicked flag when the feedback panel Submit is clicked.
  // This gates the toast watcher so it only counts feedback submissions,
  // not bot query submissions or any other form on the page.
  const buttonObserver = new MutationObserver(() => {
    const submitButtons = document.querySelectorAll('button');
    submitButtons.forEach((btn) => {
      if (
        btn.innerText.trim().toLowerCase() === 'submit' &&
        !btn.dataset.qcListening
      ) {
        btn.dataset.qcListening = 'true';
        btn.addEventListener('click', () => startToastWatcher());
      }
    });
  });

  buttonObserver.observe(document.body, { childList: true, subtree: true });

})();
