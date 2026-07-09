# ARZ Leaders — Онлайн в прайм-тайм

Скрипт для [lead.arztools.ru](https://lead.arztools.ru) / [lead.arztools.tech](https://lead.arztools.tech). Считает **онлайн в прайм-тайм** по логам входов/выходов на странице лидера и показывает виджет рядом с «Онлайн за неделю».

---

## Что делает скрипт

- Парсит вкладку **«Входы/выходы»** на странице проверки лидера.
- Считает время, попавшее в окно **прайм-тайма** (по умолчанию **10:00–21:00 МСК**).
- Неделя считается **с понедельника 00:00** до воскресенья 23:59.
- Показывает прогресс-бар, % от нормы и среднее в день.
- Модалка **«по дням»** — разбивка прайм-тайма, онлайна, сессий и входов.
- Норма недели настраивается (по умолчанию **21 час**).

---

### Колонки в таблице по дням

| Колонка | Значение |
|---------|----------|
| **Прайм-тайм** | Время в окне 10:00–21:00 |
| **% нормы** | Доля от дневной нормы (норма недели ÷ 7) |
| **% окна** | Насколько заполнено прайм-окно за день (макс. 11 ч) |
| **Онлайн** | Полное время в игре за день (включая ночь) |
| **Сессий** | Число сессий за день |
| **Входов** | Число записей «авторизовался» за день |

---

## Установка в Tampermonkey

### 1. Установите Tampermonkey

Скачайте с [tampermonkey.net](https://www.tampermonkey.net/) и включите расширение.

### 2. Разрешите userscripts (Chrome)

`Настройки Chrome` → `Расширения` → Tampermonkey → включите **«Allow user scripts»**.

### 3. Создайте скрипт

1. Клик по иконке Tampermonkey → **Dashboard** → **+** (создать новый скрипт)
2. Удалите шаблонный код в редакторе
3. Скопируйте **весь код** из раздела [Код скрипта](#код-скрипта) ниже
4. Вставьте в редактор Tampermonkey
5. **Ctrl+S** — сохранить

### 4. CSP (если скрипт не запускается)

Tampermonkey → **Настройки** → **Config mode** → **Advanced** → **CSP** → **Remove entirely**.

### 5. Проверка

1. Откройте страницу лидера: `https://lead.arztools.ru/load/checklead.php?id=...`
2. Кликните вкладку **«Входы/выходы»** и дождитесь загрузки логов
3. Под «Онлайн за неделю» появится блок **«Онлайн в прайм-тайм»**
4. Обновите страницу: **Ctrl+F5**

---

## Код скрипта

Скопируйте **всё** содержимое блока ниже в редактор Tampermonkey:

```javascript
// ==UserScript==
// @name         ARZ Leaders — Онлайн в прайм-тайм
// @namespace    https://lead.arztools.ru/
// @version      1.6.6
// @description  Подсчёт онлайна в настраиваемые часы прайм-тайма (по умолчанию 10:00–21:00 МСК)
// @author       you
// @match        https://lead.arztools.ru/*
// @match        https://lead.arztools.tech/*
// @match        https://*.arztools.ru/*
// @match        https://*.arztools.tech/*
// @grant        GM_getValue
// @grant        GM_setValue
// @grant        GM_registerMenuCommand
// @grant        GM_addStyle
// @run-at       document-end
// ==/UserScript==

/* global bootstrap, GM_getValue, GM_setValue, GM_registerMenuCommand, GM_addStyle */

(function () {
  'use strict';

  const LOG_PREFIX = '[ARZ Prime]';
  const STORAGE_KEY = 'arz_prime_time_config';
  const WIDGET_ID = 'arz-prime-time-row';
  const MODAL_ID = 'arz-prime-stat-days';
  const DAY_NAMES = ['ВС', 'ПН', 'ВТ', 'СР', 'ЧТ', 'ПТ', 'СБ'];
  /** Макс. длина «живой» сессии без выхода в логах (11 ч = окно прайм-тайма). */
  const OPEN_SESSION_MAX_MS = 11 * 3600 * 1000;
  const DEFAULT_CONFIG = {
    startHour: 10,
    endHour: 21,
    normHours: 21,
    enabled: true,
    debug: false,
  };

  let config = loadConfig();
  let lastResult = null;
  let lastDisplaySignature = '';

  GM_addStyle(`
    #${WIDGET_ID} .new-leader-time-details-icon { position: relative; cursor: pointer; }
    #${MODAL_ID} tfoot td { border-top: 1px solid rgba(255,255,255,0.12); }
    #${MODAL_ID} .arz-prime-summary .new-leader-stat-badge { min-width: 7rem; }
    #content .col-lg-4.flex-grow-1.mb-4 > .new-leader-leader-card.arz-profile-card {
      display: flex !important;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      height: 100%;
    }
    #content .arz-profile-center {
      display: flex;
      justify-content: center;
      align-items: center;
      width: 100%;
    }
    #content .arz-profile-row {
      display: flex;
      align-items: center;
      justify-content: center;
      gap: 1rem;
      flex-wrap: wrap;
    }
    #content .arz-profile-card .new-leader-avatar-lg {
      margin: 0 !important;
      flex-shrink: 0;
    }
    #content .arz-profile-info {
      text-align: left;
      min-width: 0;
    }
    #content .arz-profile-info .d-flex {
      justify-content: flex-start !important;
    }
    #content .arz-profile-info p.text-muted {
      margin-bottom: 0 !important;
    }
    #content .arz-profile-badges {
      display: flex;
      gap: 0.5rem;
      flex-shrink: 0;
      margin: 0 !important;
    }
  `);

  /** Аватар + ник + должность по центру карточки, баллы/дни — полем рядом. */
  function fixProfileCard() {
    const col = document.querySelector('#content .row.d-flex > .col-lg-4.flex-grow-1.mb-4');
    const card = col?.querySelector(':scope > .new-leader-leader-card');
    if (!card?.querySelector('.new-leader-avatar-lg')) return;
    if (card.dataset.arzProfileLayout === '1') return;

    const avatar = card.querySelector('.new-leader-avatar-lg');
    const nickWrap = card.querySelector('.d-flex.align-items-center.justify-content-center');
    const badges = card.querySelector('.d-flex.justify-content-center.gap-2.mb-2');
    if (!avatar || !nickWrap || !badges) return;

    avatar.classList.remove('mb-3', 'mx-auto');

    const center = document.createElement('div');
    center.className = 'arz-profile-center';
    const row = document.createElement('div');
    row.className = 'arz-profile-row';

    const info = document.createElement('div');
    info.className = 'arz-profile-info';
    info.appendChild(nickWrap);
    card.querySelectorAll(':scope > p.text-muted').forEach((p) => info.appendChild(p));

    badges.classList.remove('mb-2', 'justify-content-center');
    badges.classList.add('arz-profile-badges');

    row.appendChild(avatar);
    row.appendChild(info);
    row.appendChild(badges);
    center.appendChild(row);

    const gear = card.querySelector('.position-absolute');
    if (gear) gear.after(center);
    else card.appendChild(center);

    card.classList.remove('new-leader-vertical-center');
    card.classList.add('h-100', 'arz-profile-card');
    card.dataset.arzProfileLayout = '1';
  }

  function log(...args) {
    if (config.debug) console.log(LOG_PREFIX, ...args);
  }

  function warn(...args) {
    console.warn(LOG_PREFIX, ...args);
  }

  function loadConfig() {
    try {
      return { ...DEFAULT_CONFIG, ...JSON.parse(GM_getValue(STORAGE_KEY, '{}')) };
    } catch {
      return { ...DEFAULT_CONFIG };
    }
  }

  function saveConfig(next) {
    GM_setValue(STORAGE_KEY, JSON.stringify(next));
  }

  function openNormSettings() {
    const norm = prompt('Норма онлайна за неделю (часов):', String(config.normHours ?? DEFAULT_CONFIG.normHours));
    if (norm === null) return;

    const normHours = Number(norm);
    if (!Number.isFinite(normHours) || normHours <= 0 || normHours > 168) {
      alert('Некорректное значение. Укажите число часов, например: 21');
      return;
    }

    config = { ...config, normHours };
    saveConfig(config);
    lastDisplaySignature = '';
    refresh(true);
    alert(`Норма: ${normHours} ч. (${formatDuration(normHours * 3600 * 1000)})`);
  }

  function openSettings() {
    const start = prompt('Начало прайм-тайма (час, 0–23, МСК):', String(config.startHour));
    if (start === null) return;
    const end = prompt('Конец прайм-тайма (час, 0–23, не включая):', String(config.endHour));
    if (end === null) return;
    const norm = prompt('Норма онлайна за неделю (часов):', String(config.normHours ?? DEFAULT_CONFIG.normHours));
    if (norm === null) return;

    const startHour = Number(start);
    const endHour = Number(end);
    const normHours = Number(norm);
    if (
      !Number.isInteger(startHour) || !Number.isInteger(endHour) ||
      startHour < 0 || startHour > 23 || endHour < 0 || endHour > 23 ||
      startHour >= endHour
    ) {
      alert('Некорректные часы. Пример: начало 10, конец 21 → с 10:00 до 20:59:59.');
      return;
    }
    if (!Number.isFinite(normHours) || normHours <= 0 || normHours > 168) {
      alert('Некорректная норма. Укажите число часов, например: 21');
      return;
    }

    config = { ...config, startHour, endHour, normHours };
    saveConfig(config);
    lastDisplaySignature = '';
    refresh(true);
    alert(`Прайм-тайм: ${pad2(startHour)}:00–${pad2(endHour)}:00\nНорма: ${normHours} ч.`);
  }

  try {
    GM_registerMenuCommand('Настройки прайм-тайма…', openSettings);
    GM_registerMenuCommand('Норма онлайна (часов)…', openNormSettings);
    GM_registerMenuCommand('Пересчитать прайм-тайм', () => refresh(true));
    GM_registerMenuCommand('Отладка: вкл/выкл', () => {
      config.debug = !config.debug;
      saveConfig(config);
      alert(`Отладка ${config.debug ? 'включена' : 'выключена'}. Откройте F12 → Console.`);
      refresh(true);
    });
  } catch (e) {
    console.error(LOG_PREFIX, 'Ошибка меню Tampermonkey:', e);
  }

  function pad2(n) {
    return String(n).padStart(2, '0');
  }

  function parseLogDate(text) {
    const m = text.match(/(\d{2})\.(\d{2})\.(\d{4})\s+(\d{2}):(\d{2}):(\d{2})/);
    if (!m) return null;
    const [, dd, mm, yyyy, hh, mi, ss] = m;
    return new Date(Number(yyyy), Number(mm) - 1, Number(dd), Number(hh), Number(mi), Number(ss));
  }

  function parseDurationToMs(text) {
    const m = text.match(/(\d{2}):(\d{2}):(\d{2})/);
    if (!m) return null;
    const [, h, mnt, s] = m;
    return (Number(h) * 3600 + Number(mnt) * 60 + Number(s)) * 1000;
  }

  function formatDuration(ms) {
    const totalSec = Math.max(0, Math.floor(ms / 1000));
    const h = Math.floor(totalSec / 3600);
    const m = Math.floor((totalSec % 3600) / 60);
    const s = totalSec % 60;
    return `${pad2(h)}:${pad2(m)}:${pad2(s)}`;
  }

  function formatDisplayDate(date) {
    return `${date.getFullYear()}-${pad2(date.getMonth() + 1)}-${pad2(date.getDate())} (${DAY_NAMES[date.getDay()]})`;
  }

  function formatShortDate(date) {
    return `${pad2(date.getDate())}.${pad2(date.getMonth() + 1)}.${String(date.getFullYear()).slice(-2)}`;
  }

  function formatPercent(value, total) {
    if (!total || total <= 0) return '—';
    return `${Math.round((value / total) * 100)}%`;
  }

  function getMaxPrimeMsPerDay() {
    return (config.endHour - config.startHour) * 3600 * 1000;
  }

  function getDailyNormMs(normMs) {
    return normMs ? normMs / 7 : null;
  }

  function getElapsedDaysInWeek(weekRange) {
    const now = getNowCap();
    let count = 0;
    const cursor = new Date(weekRange.start);
    for (let i = 0; i < 7; i += 1) {
      if (cursor > now) break;
      count += 1;
      cursor.setDate(cursor.getDate() + 1);
    }
    return count || 1;
  }

  function initWidgetTooltips(widget) {
    if (typeof bootstrap === 'undefined' || !bootstrap.Tooltip) return;
    widget.querySelectorAll('[data-bs-toggle="tooltips"]').forEach((el) => {
      const old = bootstrap.Tooltip.getInstance(el);
      if (old) old.dispose();
      new bootstrap.Tooltip(el);
    });
  }

  function createTableIcon() {
    const icon = document.createElement('i');
    icon.className = 'fa-solid fa-table new-leader-time-details-icon d-flex';
    icon.setAttribute('data-bs-toggle', 'tooltips');
    icon.setAttribute('data-bs-title', 'Статистика по дням');
    const hit = document.createElement('span');
    hit.style.cssText = 'width:20px;height:20px;position:absolute;cursor:pointer';
    hit.addEventListener('click', (e) => {
      e.preventDefault();
      e.stopPropagation();
      openDaysModal();
    });
    icon.appendChild(hit);
    return icon;
  }

  function createSettingsIcon() {
    const icon = document.createElement('i');
    icon.className = 'fa-solid fa-gear new-leader-time-details-icon d-flex';
    icon.setAttribute('data-bs-toggle', 'tooltips');
    icon.setAttribute('data-bs-title', 'Настройки прайм-тайма');
    const hit = document.createElement('span');
    hit.style.cssText = 'width:20px;height:20px;position:absolute;cursor:pointer';
    hit.addEventListener('click', (e) => {
      e.preventDefault();
      e.stopPropagation();
      openSettings();
    });
    icon.appendChild(hit);
    return icon;
  }

  function getLeaderNick() {
    return document.querySelector('.new-leader-nick')?.textContent?.trim() || 'Лидер';
  }

  /** Неделя строго с понедельника 00:00 до воскресенья 23:59 (МСК, локальное время браузера). */
  function getWeekRangeMonday() {
    const now = new Date();
    const day = now.getDay();
    const diffToMonday = day === 0 ? 6 : day - 1;
    const start = new Date(now);
    start.setDate(now.getDate() - diffToMonday);
    start.setHours(0, 0, 0, 0);
    const end = new Date(start);
    end.setDate(start.getDate() + 6);
    end.setHours(23, 59, 59, 999);
    return { start, end };
  }

  function getNowCap() {
    return new Date();
  }

  function primeWindowForDate(date, startHour, endHour) {
    const start = new Date(date);
    start.setHours(startHour, 0, 0, 0);
    const end = new Date(date);
    end.setHours(endHour, 0, 0, 0);
    return { start, end };
  }

  function overlapMs(aStart, aEnd, bStart, bEnd) {
    const start = Math.max(aStart.getTime(), bStart.getTime());
    const end = Math.min(aEnd.getTime(), bEnd.getTime());
    return Math.max(0, end - start);
  }

  function sessionPrimeTimeMs(sessionStart, sessionEnd, startHour, endHour) {
    let total = 0;
    const cursor = new Date(sessionStart);
    cursor.setHours(0, 0, 0, 0);

    while (cursor <= sessionEnd) {
      const { start: pStart, end: pEnd } = primeWindowForDate(cursor, startHour, endHour);
      total += overlapMs(sessionStart, sessionEnd, pStart, pEnd);
      cursor.setDate(cursor.getDate() + 1);
    }
    return total;
  }

  function collectLoginLogEvents() {
    const items = document.querySelectorAll('#loginLogs .new-leader-log-item');
    const events = [];

    items.forEach((item) => {
      const message = item.querySelector('.d-flex span')?.textContent?.trim() || '';
      const meta = item.querySelector('.text-muted.small')?.textContent?.trim() || '';
      const isLogin = /авторизовался на сервер/i.test(message);
      const isLogout = /вышел с сервера/i.test(message);
      if (!isLogin && !isLogout) return;

      const time = parseLogDate(meta);
      if (!time) return;

      events.push({
        type: isLogin ? 'login' : 'logout',
        time,
        sessionMs: isLogout ? parseDurationToMs(message.match(/время сессии:\s*(\d{2}:\d{2}:\d{2})/)?.[1] || '') : null,
      });
    });

    events.sort((a, b) => a.time - b.time);
    return events;
  }

  /** Собираем сессии только из логов входов/выходов. Один активный вход — один игрок. */
  function buildAllSessionsFromLogs(events) {
    const sessions = [];
    let openLogin = null;
    const now = getNowCap();

    for (const ev of events) {
      if (ev.type === 'login') {
        openLogin = ev;
        continue;
      }

      const sessionEnd = ev.time;
      let sessionStart = null;

      if (ev.sessionMs != null) {
        const impliedStart = new Date(sessionEnd.getTime() - ev.sessionMs);
        if (openLogin && openLogin.time <= sessionEnd) {
          sessionStart = impliedStart < openLogin.time ? impliedStart : openLogin.time;
          openLogin = null;
        } else {
          sessionStart = impliedStart;
        }
      } else if (openLogin && openLogin.time <= sessionEnd) {
        sessionStart = openLogin.time;
        openLogin = null;
      }

      if (sessionStart && sessionEnd > sessionStart) {
        sessions.push({ start: sessionStart, end: sessionEnd });
      }
    }

    if (openLogin && openLogin.time <= now) {
      const ageMs = now.getTime() - openLogin.time.getTime();
      const isLatestEvent = events.length > 0 && events[events.length - 1] === openLogin;
      if (isLatestEvent && ageMs > 0 && ageMs <= OPEN_SESSION_MAX_MS) {
        sessions.push({ start: openLogin.time, end: now });
      } else {
        log('висящий вход без выхода не учтён:', openLogin.time, `возраст ${formatDuration(ageMs)}`);
      }
    }

    return mergeSessions(sessions);
  }

  function mergeSessions(sessions) {
    if (!sessions.length) return [];

    const sorted = [...sessions].sort((a, b) => a.start - b.start);
    const merged = [{ start: new Date(sorted[0].start), end: new Date(sorted[0].end) }];

    for (let i = 1; i < sorted.length; i += 1) {
      const cur = sorted[i];
      const last = merged[merged.length - 1];

      if (cur.start <= last.end) {
        if (cur.end > last.end) last.end = new Date(cur.end);
      } else {
        merged.push({ start: new Date(cur.start), end: new Date(cur.end) });
      }
    }

    return merged;
  }

  function clipSessionToWeek(session, weekRange) {
    const now = getNowCap().getTime();
    const overlapStart = Math.max(session.start.getTime(), weekRange.start.getTime());
    const overlapEnd = Math.min(session.end.getTime(), weekRange.end.getTime(), now);
    if (overlapEnd <= overlapStart) return null;
    return {
      start: new Date(overlapStart),
      end: new Date(overlapEnd),
    };
  }

  function buildWeekSessions(events, weekRange) {
    return mergeSessions(
      buildAllSessionsFromLogs(events)
        .map((s) => clipSessionToWeek(s, weekRange))
        .filter(Boolean)
    );
  }

  function getNormMs() {
    const hours = Number(config.normHours);
    const normHours = Number.isFinite(hours) && hours > 0 ? hours : DEFAULT_CONFIG.normHours;
    return normHours * 3600 * 1000;
  }

  function calculateDailyBreakdown(sessions, weekRange, events) {
    const days = [];
    const cursor = new Date(weekRange.start);
    cursor.setHours(0, 0, 0, 0);
    const now = getNowCap();

    for (let i = 0; i < 7; i += 1) {
      const dayStart = new Date(cursor);
      dayStart.setHours(0, 0, 0, 0);
      const dayEnd = new Date(cursor);
      dayEnd.setHours(23, 59, 59, 999);
      const isFuture = dayStart > now;

      let primeMs = 0;
      let totalMs = 0;
      const sessionIds = new Set();

      if (!isFuture) {
        const effectiveDayEnd = new Date(Math.min(dayEnd.getTime(), now.getTime()));

        for (const session of sessions) {
          const overlapStart = new Date(Math.max(session.start.getTime(), dayStart.getTime()));
          const overlapEnd = new Date(Math.min(session.end.getTime(), effectiveDayEnd.getTime()));
          if (overlapEnd <= overlapStart) continue;

          sessionIds.add(`${session.start.getTime()}-${session.end.getTime()}`);
          totalMs += overlapEnd.getTime() - overlapStart.getTime();
          primeMs += sessionPrimeTimeMs(overlapStart, overlapEnd, config.startHour, config.endHour);
        }
      }

      const logins = events.filter((ev) =>
        ev.type === 'login' && ev.time >= dayStart && ev.time <= dayEnd
      ).length;

      const maxPrimeMs = getMaxPrimeMsPerDay();
      days.push({
        date: new Date(cursor),
        primeMs,
        totalMs,
        sessionCount: sessionIds.size,
        logins,
        maxPrimeMs,
        isFuture,
        hasActivity: primeMs > 0 || totalMs > 0 || logins > 0,
      });

      cursor.setDate(cursor.getDate() + 1);
    }

    return days;
  }

  function calculatePrimeTimeOnline() {
    const weekRange = getWeekRangeMonday();
    const events = collectLoginLogEvents();
    const sessions = buildWeekSessions(events, weekRange);

    let primeMs = 0;
    let totalMs = 0;
    for (const session of sessions) {
      const sessionMs = session.end.getTime() - session.start.getTime();
      totalMs += sessionMs;
      primeMs += sessionPrimeTimeMs(session.start, session.end, config.startHour, config.endHour);
    }
    primeMs = Math.min(primeMs, totalMs);

    const normMs = getNormMs();
    const days = calculateDailyBreakdown(sessions, weekRange, events);
    const dailyNormMs = getDailyNormMs(normMs);
    const elapsedDays = getElapsedDaysInWeek(weekRange);

    const totals = days.reduce((acc, day) => ({
      primeMs: acc.primeMs + day.primeMs,
      totalMs: acc.totalMs + day.totalMs,
      sessionCount: acc.sessionCount + day.sessionCount,
      logins: acc.logins + day.logins,
    }), { primeMs: 0, totalMs: 0, sessionCount: 0, logins: 0 });

    return {
      primeMs,
      totalMs,
      weekRange,
      sessionCount: sessions.length,
      eventCount: events.length,
      normMs,
      dailyNormMs,
      elapsedDays,
      totals,
      sessions,
      events,
      days,
    };
  }

  function findOnlineWeekRow() {
    const label = [...document.querySelectorAll('label.text-muted')].find((el) =>
      el.textContent.includes('Онлайн за неделю')
    );
    return label?.closest('.col-md-6') || null;
  }

  function ensureModal() {
    let modal = document.getElementById(MODAL_ID);
    if (modal) return modal;

    modal = document.createElement('div');
    modal.className = 'modal fade modal-leader-stat';
    modal.id = MODAL_ID;
    modal.tabIndex = -1;
    modal.setAttribute('aria-hidden', 'true');
    modal.innerHTML = `
      <div class="modal-dialog">
        <div class="modal-content">
          <div class="modal-header">
            <h1 class="modal-title fs-6" id="arz-prime-modal-title"></h1>
            <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Закрыть"></button>
          </div>
          <div class="modal-body">
            <p class="text-muted small mb-3" id="arz-prime-modal-subtitle"></p>
            <div class="row g-2 mb-3 arz-prime-summary" id="arz-prime-modal-summary"></div>
            <table class="table table-sm mb-0">
              <thead>
                <tr>
                  <th>Дата</th>
                  <th>Прайм-тайм</th>
                  <th>% нормы</th>
                  <th>% окна</th>
                  <th>Онлайн</th>
                  <th>Сессий</th>
                  <th>Входов</th>
                </tr>
              </thead>
              <tbody id="arz-prime-modal-tbody"></tbody>
              <tfoot id="arz-prime-modal-tfoot"></tfoot>
            </table>
          </div>
        </div>
      </div>
    `;

    document.body.appendChild(modal);
    return modal;
  }

  function updateModalTable() {
    if (!lastResult) {
      warn('нет данных для модалки');
      return;
    }

    const tbody = document.getElementById('arz-prime-modal-tbody');
    const tfoot = document.getElementById('arz-prime-modal-tfoot');
    const summary = document.getElementById('arz-prime-modal-summary');
    const title = document.getElementById('arz-prime-modal-title');
    const subtitle = document.getElementById('arz-prime-modal-subtitle');
    if (!tbody || !tfoot || !summary || !title || !subtitle) return;

    try {
      const { weekRange, normMs, dailyNormMs, elapsedDays, totals, primeMs } = lastResult;
      const { start, end } = weekRange;
      const avgPrimeMs = primeMs / elapsedDays;
      const totalPercentNorm = formatPercent(primeMs, normMs);
      const totalPercentWindow = formatPercent(primeMs, getMaxPrimeMsPerDay() * elapsedDays);

      title.textContent = `${getLeaderNick()} — прайм-тайм по дням`;
      subtitle.textContent = `Неделя: ${formatShortDate(start)} – ${formatShortDate(end)} · ${pad2(config.startHour)}:00–${pad2(config.endHour)}:00 МСК · расчёт по логам`;

      summary.innerHTML = `
        <div class="col-6 col-md-3">
          <div class="new-leader-stat-badge text-center py-2">
            <div class="text-accent">${formatDuration(primeMs)}</div>
            <small>Прайм-тайм</small>
          </div>
        </div>
        <div class="col-6 col-md-3">
          <div class="new-leader-stat-badge text-center py-2">
            <div class="text-accent">${totalPercentNorm}</div>
            <small>От нормы недели</small>
          </div>
        </div>
        <div class="col-6 col-md-3">
          <div class="new-leader-stat-badge text-center py-2">
            <div class="text-accent">${formatDuration(avgPrimeMs)}</div>
            <small>~в день</small>
          </div>
        </div>
        <div class="col-6 col-md-3">
          <div class="new-leader-stat-badge text-center py-2">
            <div class="text-accent">${totalPercentWindow}</div>
            <small>Заполнение окна</small>
          </div>
        </div>
      `;

      const visibleDays = lastResult.days.filter((day) => !day.isFuture);

      tbody.innerHTML = visibleDays.map((day) => `
        <tr>
          <td>${formatDisplayDate(day.date)}</td>
          <td>${formatDuration(day.primeMs)}</td>
          <td>${formatPercent(day.primeMs, dailyNormMs)}</td>
          <td>${formatPercent(day.primeMs, day.maxPrimeMs)}</td>
          <td>${formatDuration(day.totalMs)}</td>
          <td>${day.sessionCount}</td>
          <td>${day.logins}</td>
        </tr>
      `).join('');

      tfoot.innerHTML = `
        <tr>
          <td>Итого</td>
          <td>${formatDuration(totals.primeMs)}</td>
          <td>${totalPercentNorm}</td>
          <td>${totalPercentWindow}</td>
          <td>${formatDuration(totals.totalMs)}</td>
          <td>${lastResult.sessionCount}</td>
          <td>${totals.logins}</td>
        </tr>
      `;
    } catch (err) {
      console.error(LOG_PREFIX, 'ошибка модалки:', err);
      tbody.innerHTML = `<tr><td colspan="7" class="text-danger text-center">Ошибка загрузки данных</td></tr>`;
    }
  }

  function openDaysModal() {
    ensureModal();
    if (document.querySelector('#loginLogs .new-leader-log-item')) {
      lastResult = calculatePrimeTimeOnline();
    }
    updateModalTable();
    const modalEl = document.getElementById(MODAL_ID);
    if (typeof bootstrap !== 'undefined' && bootstrap.Modal) {
      bootstrap.Modal.getOrCreateInstance(modalEl).show();
    }
  }

  function getDisplaySignature(result) {
    const percent = result.normMs
      ? Math.round((result.primeMs / result.normMs) * 100)
      : 0;
    return [
      formatDuration(result.primeMs),
      result.normMs ? formatDuration(result.normMs) : '',
      percent,
      result.elapsedDays,
      result.sessionCount,
      config.startHour,
      config.endHour,
      config.normHours,
    ].join('|');
  }

  function patchWidget(row, result) {
    const percent = result.normMs
      ? Math.round((result.primeMs / result.normMs) * 100)
      : 0;
    const progressPercent = Math.min(100, percent);
    const valueText = result.normMs
      ? `${formatDuration(result.primeMs)}/${formatDuration(result.normMs)} (${percent}%)`
      : `${formatDuration(result.primeMs)}`;

    const progressBar = row.querySelector('.progress-bar');
    if (progressBar) progressBar.style.width = `${progressPercent}%`;

    const valueSpan = row.querySelector('.arz-prime-value');
    if (valueSpan) valueSpan.textContent = valueText;

    const dailySpan = row.querySelector('.arz-prime-daily');
    if (dailySpan) {
      dailySpan.textContent = `~${formatDuration(result.primeMs / result.elapsedDays)} в день · ${percent}% нормы`;
    }

    const hoursSpan = row.querySelector('.arz-prime-hours');
    if (hoursSpan) {
      hoursSpan.textContent = `(${pad2(config.startHour)}:00–${pad2(config.endHour)}:00, с ПН)`;
    }
  }

  function createWidget(result) {
    const row = document.createElement('div');
    row.id = WIDGET_ID;
    row.className = 'col-md-6 mb-3';

    const percent = result.normMs
      ? Math.round((result.primeMs / result.normMs) * 100)
      : 0;
    const progressPercent = Math.min(100, percent);
    const valueText = result.normMs
      ? `${formatDuration(result.primeMs)}/${formatDuration(result.normMs)} (${percent}%)`
      : `${formatDuration(result.primeMs)}`;

    row.innerHTML = `
      <label class="text-muted">
        <i class="fa-solid fa-sun me-2"></i>Онлайн в прайм-тайм
        <span class="arz-prime-hours" style="color: gray; margin-left: 4px; font-size: small;">
          (${pad2(config.startHour)}:00–${pad2(config.endHour)}:00, с ПН)
        </span>
      </label>
      <div class="d-flex align-items-center gap-2">
        <div class="progress flex-grow-1">
          <div class="progress-bar" style="width: ${progressPercent}%"></div>
        </div>
        <span class="arz-prime-value">${valueText}</span>
      </div>
      <div class="new-leader-stat-badge d-flex align-items-center gap-2 mt-2">
        <span class="arz-prime-daily">~${formatDuration(result.primeMs / result.elapsedDays)} в день · ${percent}% нормы</span>
      </div>
    `;

    const badge = row.querySelector('.new-leader-stat-badge');
    badge?.appendChild(createTableIcon());
    badge?.appendChild(createSettingsIcon());
    initWidgetTooltips(row);

    return row;
  }

  function updateWidget(result) {
    lastResult = result;
    const signature = getDisplaySignature(result);
    const existing = document.getElementById(WIDGET_ID);

    if (existing) {
      if (signature !== lastDisplaySignature) {
        patchWidget(existing, result);
        lastDisplaySignature = signature;
        log('виджет обновлён (без пересоздания)');
      }
      return true;
    }

    const widget = createWidget(result);
    lastDisplaySignature = signature;

    const onlineRow = findOnlineWeekRow();
    if (onlineRow?.parentElement) {
      onlineRow.insertAdjacentElement('afterend', widget);
      log('виджет вставлен после «Онлайн за неделю»');
      return true;
    }

    warn('не удалось найти место для вставки виджета');
    return false;
  }

  function isOurMutation(mutation) {
    const checkNode = (node) => {
      if (!node || node.nodeType !== 1) return false;
      return node.id === WIDGET_ID ||
        node.id === MODAL_ID ||
        !!node.closest?.(`#${WIDGET_ID}, #${MODAL_ID}`);
    };

    if (checkNode(mutation.target)) return true;

    return [...mutation.addedNodes, ...mutation.removedNodes].some(checkNode);
  }

  function refresh(forceLog) {
    const debug = config.debug || forceLog;

    if (!document.body) return;

    const onPage = /checklead\.php/i.test(location.href) ||
      document.body.innerText.includes('Статистика за неделю');
    if (!onPage) return;

    const logItems = document.querySelectorAll('#loginLogs .new-leader-log-item');
    if (!logItems.length) {
      if (debug) warn('логи #loginLogs ещё не загружены');
      return;
    }

    if (!config.enabled) {
      document.getElementById(WIDGET_ID)?.remove();
      document.getElementById(MODAL_ID)?.remove();
      return;
    }

    const result = calculatePrimeTimeOnline();
    ensureModal();
    const ok = updateWidget(result);

    if (debug || forceLog) {
      console.log(LOG_PREFIX, 'готово', {
        week: `${formatShortDate(result.weekRange.start)} – ${formatShortDate(result.weekRange.end)}`,
        prime: formatDuration(result.primeMs),
        totalFromLogs: formatDuration(result.totalMs),
        sessions: result.sessionCount,
        events: result.eventCount,
        widgetInserted: ok,
      });
    }
  }

  let debounceTimer;
  function scheduleRefresh() {
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(() => {
      fixProfileCard();
      refresh(false);
    }, 300);
  }

  function watchContent() {
    const content = document.getElementById('content');
    if (!content) {
      setTimeout(watchContent, 200);
      return;
    }

    const observer = new MutationObserver((mutations) => {
      if (mutations.length && mutations.every(isOurMutation)) return;
      scheduleRefresh();
    });
    observer.observe(content, { childList: true, subtree: true });
    scheduleRefresh();
  }

  // Раз в минуту — если игрок сейчас в сети, время растёт
  setInterval(() => {
    if (document.getElementById(WIDGET_ID)) refresh(false);
  }, 60000);

  document.addEventListener('shown.bs.tab', (e) => {
    if (e.target?.getAttribute('href') === '#loginLogs') {
      setTimeout(() => refresh(true), 150);
    }
  });

  document.addEventListener('shown.bs.modal', (e) => {
    if (e.target?.id === MODAL_ID) updateModalTable();
  });

  let attempts = 0;
  const retryTimer = setInterval(() => {
    attempts += 1;
    fixProfileCard();
    refresh(attempts <= 3);
    if (document.getElementById(WIDGET_ID) || attempts >= 40) {
      clearInterval(retryTimer);
    }
  }, 500);

  watchContent();
  console.log(LOG_PREFIX, 'скрипт v1.6.6 запущен на', location.href);
})();
```
