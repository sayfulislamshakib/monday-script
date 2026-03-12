# Monday.com Status Injector

A Tampermonkey userscript that allows you to instantly inject beautifully formatted project statuses (Working on it, Stakeholder Review, Stuck, Complete) into Monday.com documents.

**Author:** Sayful  
**Version:** 4.3  

## ✨ Features
* **Lightning Fast:** Trigger the menu instantly with a keyboard shortcut (`Alt + W`).
* **Hybrid Interaction:** Select your status using keyboard numbers (`1-4`) or by clicking with your mouse.
* **Ghost Cursor Technology:** Safely injects text without highlighting, overwriting, or deleting your existing paragraphs.
* **Strict Formatting:** Locks in Monday.com's native colors (Egg Yolk, Dark Orange, Stuck Red, Done Green) and prevents the editor from adding unwanted double-spaces or extra empty lines.
* **Custom Inputs:** Built-in floating UI for typing custom update messages without breaking browser focus.

---

## ⚙️ Required Extension Settings

Modern browsers require a few specific settings for Tampermonkey scripts to run smoothly. Before installing, please configure your extension:

1. **Install Tampermonkey:** Get the Tampermonkey extension for your browser (Chrome, Edge, Firefox, etc.).
2. **Enable Developer Mode (Chrome/Edge):**
   * Open a new tab and go to your browser's extension page (e.g., type `chrome://extensions/` in the URL bar).
   * Toggle **Developer Mode** to **ON** (usually located in the top right corner).
3. **Allow File Permissions:**
   * On that same extensions page, find the Tampermonkey extension card and click **Details**.
   * Scroll down and toggle **"Allow access to file URLs"** to **ON**.

---

## 🚀 Installation

1. Click the Tampermonkey extension icon in your browser toolbar and select **Dashboard**.
2. Click the **+** icon (Create a new script) tab at the top.
3. Delete any placeholder code currently in the editor.
4. Copy and paste the entire script below into the editor.
5. Go to **File > Save** (or press `Ctrl + S` / `Cmd + S`).
6. Refresh your Monday.com tab for the script to take effect.

### The Code

```javascript
// ==UserScript==
// @name         Monday Status Injector
// @namespace    [http://tampermonkey.net/](http://tampermonkey.net/)
// @version      4.3
// @description  Instantly injects formatted project statuses (Working on it, Stuck, etc.) into Monday.com using Alt+W without overwriting existing text.
// @author       Sayful
// @match        *://*[.monday.com/](https://.monday.com/)*
// @grant        none
// @run-at       document-end
// ==/UserScript==

(function() {
    'use strict';

    let savedRange = null;
    let targetEditor = null;
    let menuState = 'NONE'; 

    window.addEventListener('keydown', (e) => {
        if (menuState === 'NONE') {
            if (e.altKey && (e.key === 'w' || e.key === 'W')) {
                e.preventDefault();
                e.stopPropagation();
                showMainMenu();
            }
            return;
        }

        if (menuState === 'MAIN') {
            if (e.key === "Escape") {
                e.preventDefault();
                e.stopPropagation();
                closeMainMenu();
            } else if (["1", "2", "3", "4"].includes(e.key)) {
                e.preventDefault();
                e.stopPropagation();
                const selectedKey = e.key;
                closeMainMenu();
                processSelection(selectedKey);
            } else if (e.key !== 'Alt') {
                e.preventDefault();
                e.stopPropagation();
            }
            return;
        }

        if (menuState === 'INPUT') {
            if (e.key === "Escape") {
                e.preventDefault();
                e.stopPropagation();
                closeInputMenu();
                if (targetEditor) targetEditor.focus();
            } else if (e.key === "Enter") {
                e.preventDefault();
                e.stopPropagation();
                const inputEl = document.getElementById('status-custom-input');
                const msg = inputEl.value;
                const label = inputEl.getAttribute('data-label');
                const color = inputEl.getAttribute('data-color');
                closeInputMenu();
                restoreAndInject(label, color, msg);
            }
            return;
        }
    }, true);

    function showMainMenu() {
        const existing = document.getElementById('status-injector-menu');
        if (existing) existing.remove();

        const menu = document.createElement('div');
        menu.id = 'status-injector-menu';
        menu.style = "position:fixed; top:30%; left:50%; transform:translate(-50%, -30%); z-index:2147483647; background:#1c1d1f; color:#ffffff; padding:20px; border-radius:12px; border:1px solid #444; font-family:sans-serif; box-shadow:0 20px 60px rgba(0,0,0,0.8); text-align:center; min-width:320px;";
        
        const options = [
            { id: '1', label: 'Working on it', color: '#ffcb00' },
            { id: '2', label: 'Stakeholder Review', color: '#ff9d00' },
            { id: '3', label: 'Stuck', color: '#df2f4a' },
            { id: '4', label: 'Complete', color: '#00ca72' }
        ];

        let menuHTML = `<div style="font-size:18px; font-weight:bold; margin-bottom:15px; color:#ffcb00;">Select Status</div>`;
        options.forEach(opt => {
            menuHTML += `<div class="status-opt" data-key="${opt.id}" style="padding:10px 15px; margin:5px 0; border-radius:6px; cursor:pointer; text-align:left; transition: background 0.2s; display:flex; align-items:center;">
                <b style="color:${opt.color}; margin-right:12px; font-size:18px;">${opt.id}.</b>
                <span style="font-size:16px;">${opt.label}</span>
            </div>`;
        });
        menuHTML += `<div style="font-size:11px; margin-top:10px; color:#888;">[ESC to cancel]</div>`;
        menu.innerHTML = menuHTML;
        document.body.appendChild(menu);

        const style = document.createElement('style');
        style.id = 'status-menu-styles';
        style.innerHTML = `.status-opt:hover { background: #333 !important; }`;
        document.head.appendChild(style);

        menu.addEventListener('mousedown', (e) => {
            e.preventDefault(); 
        });

        menu.addEventListener('click', (e) => {
            const row = e.target.closest('.status-opt');
            if (row) {
                const key = row.getAttribute('data-key');
                closeMainMenu();
                processSelection(key);
            }
        });

        menuState = 'MAIN';
    }

    function closeMainMenu() {
        const menu = document.getElementById('status-injector-menu');
        if (menu) menu.remove();
        const styles = document.getElementById('status-menu-styles');
        if (styles) styles.remove();
        menuState = 'NONE';
    }

    function closeInputMenu() {
        const menu = document.getElementById('status-input-menu');
        if (menu) menu.remove();
        menuState = 'NONE';
    }

    function processSelection(key) {
        let label, color, msg, defaultText, skipInput = false;
        switch(key) {
            case "1": label = "Working on it"; color = "var(--color-egg_yolk)"; defaultText = "Start working on the"; break;
            case "2": label = "Stakeholder Review"; color = "var(--color-dark-orange)"; msg = "Moved for Stakeholder Review."; skipInput = true; break;
            case "3": label = "Stuck"; color = "var(--color-stuck-red)"; defaultText = "Stopped working due to a high-priority"; break;
            case "4": label = "Complete"; color = "var(--color-done-green)"; msg = "Moved for development."; skipInput = true; break;
        }

        if (!skipInput) {
            const sel = window.getSelection();
            if (sel.rangeCount > 0) {
                savedRange = sel.getRangeAt(0).cloneRange();
                const node = savedRange.commonAncestorContainer;
                targetEditor = node.nodeType === 1 ? node.closest('[contenteditable="true"]') : node.parentElement?.closest('[contenteditable="true"]');
            }
            showInputMenu(label, color, defaultText);
        } else {
            inject(label, color, msg, document.activeElement);
        }
    }

    function showInputMenu(label, color, defaultText) {
        const inputMenu = document.createElement('div');
        inputMenu.id = 'status-input-menu';
        inputMenu.style = "position:fixed; top:30%; left:50%; transform:translate(-50%, -30%); z-index:2147483647; background:#1c1d1f; color:#ffffff; padding:20px; border-radius:12px; border:1px solid #444; font-family:sans-serif; box-shadow:0 20px 60px rgba(0,0,0,0.8); text-align:center; min-width:320px;";
        
        inputMenu.innerHTML = `
            <div style="font-size:16px; font-weight:bold; margin-bottom:15px; color:${color.replace('var(--color-egg_yolk)', '#ffcb00').replace('var(--color-stuck-red)', '#df2f4a')};">
                Message for ${label}
            </div>
            <input type="text" id="status-custom-input" data-label="${label}" data-color="${color}" value="${defaultText}" style="width:100%; box-sizing:border-box; padding:10px; border-radius:6px; border:1px solid #555; background:#2a2b2d; color:#fff; font-size:16px; margin-bottom:15px; outline:none;" />
            
            <button id="status-submit-btn" style="background:#00ca72; color:#1c1d1f; border:none; padding:10px 20px; border-radius:6px; font-size:16px; font-weight:bold; cursor:pointer; width:100%; margin-bottom:10px; transition: background 0.2s;">
                Add
            </button>
            
            <div style="font-size:11px; color:#888;">[Enter] or Click Add &nbsp;&nbsp; [ESC] to cancel</div>
        `;
        document.body.appendChild(inputMenu);

        inputMenu.addEventListener('mousedown', (e) => {
            if (e.target.tagName !== 'INPUT' && e.target.tagName !== 'BUTTON') {
                e.preventDefault(); 
            }
        });

        menuState = 'INPUT';
        const inputField = document.getElementById('status-custom-input');
        const submitBtn = document.getElementById('status-submit-btn');
        
        submitBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            const msg = inputField.value;
            closeInputMenu();
            restoreAndInject(label, color, msg);
        });

        submitBtn.addEventListener('mouseover', () => { submitBtn.style.background = '#00e682'; });
        submitBtn.addEventListener('mouseout', () => { submitBtn.style.background = '#00ca72'; });

        inputField.focus();
        inputField.select();
    }

    function restoreAndInject(label, color, msg) {
        if (targetEditor) targetEditor.focus();
        const sel = window.getSelection();
        if (savedRange && sel) {
            try {
                sel.removeAllRanges();
                sel.addRange(savedRange);
                sel.collapseToEnd();
            } catch (e) {}
        }
        inject(label, color, msg, targetEditor || document.activeElement);
    }

    function inject(label, color, message, targetNode) {
        const now = new Date();
        const timeStr = now.toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit', hour12: true });
        const dateStr = now.toLocaleDateString('en-GB', { day: '2-digit', month: 'short', year: 'numeric' });

        const html = '<div style="margin:-18px 0 !important; padding:0 !important; display:block; line-height:1.1 !important; font-family:sans-serif;">' +
            '<h1 style="margin:0 0 4px 0 !important; padding:0 !important; font-weight:bold; line-height:1 !important;">' +
                '<span style="color:' + color + ' !important;">' + label + '</span>' +
            '</h1>' +
            '<div style="display:flex; align-items:center; margin:0 0 6px 0 !important; padding:0 !important; line-height:1 !important;">' +
                '<span style="color:#b3b3b3 !important; font-size:14px;">at&nbsp;</span>' +
                '<span style="font-size:14px; color:inherit;">' + timeStr + ',&nbsp;</span>' +
                '<span style="font-size:14px; color:inherit;">' + dateStr + '</span>' +
            '</div>' +
            '<p style="margin:0 !important; padding:0 !important; font-size:18px; line-height:1.2 !important;">' + message + '</p>' +
        '</div>';

        const dataTransfer = new DataTransfer();
        dataTransfer.setData('text/html', html);
        dataTransfer.setData('text/plain', label);
        
        const event = new ClipboardEvent('paste', { clipboardData: dataTransfer, bubbles: true, cancelable: true });
        targetNode.dispatchEvent(event);
    }
})();
