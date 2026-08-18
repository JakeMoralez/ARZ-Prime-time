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
// @name         ARZ Leaders — Онлайн в прайм-тайм (+Похвалы)
// @namespace    https://lead.arztools.ru/
// @version      1.8.0
// @description  Подсчёт онлайна в прайм-тайм и проверка похвал (макс 2 в день)
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
  const PERIOD_MODES = {
    CURRENT: 'current',
    PREVIOUS: 'previous',
    CUSTOM: 'custom',
    ALL_TIME: 'all_time',
  };
  const DEFAULT_CONFIG = {
    startHour: 10,
    endHour: 21,
    normHours: 21,
    periodMode: 'current',
    customPeriodStart: null,
    customPeriodEnd: null,
    ignoredPromotions: [],
    promoMinRank: 1,
    promoJump: 1,
    enabled: true,
    debug: false,
  };

  let config = loadConfig();
  if (!Array.isArray(config.ignoredPromotions)) config.ignoredPromotions = [];
  let lastResult = null;
  let lastDisplaySignature = '';

  GM_addStyle(`
    #${WIDGET_ID} .new-leader-time-details-icon { position: relative; cursor: pointer; }
    #${MODAL_ID} tfoot td { border-top: 1px solid rgba(255,255,255,0.12); }
    #${MODAL_ID} .arz-prime-summary .new-leader-stat-badge { min-width: 7rem; }
    #${WIDGET_ID} .arz-prime-period-select {
      max-width: 11rem;
      font-size: small;
      padding: 0.1rem 1.5rem 0.1rem 0.35rem;
      cursor: pointer;
    }
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

    /* --- Кастомный дизайн для ARZ Tools --- */
    .btn-arz {
      background-color: rgba(255,255,255,0.05);
      border: 1px solid rgba(255,255,255,0.1);
      color: #aab2bd;
      transition: all 0.2s ease;
      box-shadow: none !important;
      border-radius: 6px;
    }
    .btn-arz:hover {
      background-color: rgba(255,255,255,0.1);
      color: #fff;
      border-color: rgba(255,255,255,0.2);
    }
    .btn-arz-primary {
      background-color: rgba(13, 110, 253, 0.1);
      border-color: rgba(13, 110, 253, 0.3);
      color: #9ac3fe;
    }
    .btn-arz-primary:hover {
      background-color: rgba(13, 110, 253, 0.2);
      color: #fff;
    }
    .btn-arz-warning {
      background-color: rgba(255, 193, 7, 0.1);
      border-color: rgba(255, 193, 7, 0.3);
      color: #ffda6a;
    }
    .btn-arz-warning:hover {
      background-color: rgba(255, 193, 7, 0.2);
      color: #fff;
    }
    .btn-arz-success {
      background-color: rgba(25, 135, 84, 0.1);
      border-color: rgba(25, 135, 84, 0.3);
      color: #a3cfbb;
    }
    .btn-arz-success:hover {
      background-color: rgba(25, 135, 84, 0.2);
      color: #fff;
    }
    .arz-glass-card {
      background-color: rgba(255,255,255,0.03);
      border: 1px solid rgba(255,255,255,0.08);
      border-radius: 8px;
      transition: background-color 0.2s;
    }
    .arz-glass-card:hover {
      background-color: rgba(255,255,255,0.06);
    }
    .arz-glass-card-warning {
      border-left: 3px solid rgba(255, 193, 7, 0.5);
    }
    .arz-list-group-item {
      background-color: transparent !important;
      border: 1px solid rgba(255,255,255,0.08) !important;
      color: #aab2bd !important;
      margin-bottom: 4px;
      border-radius: 6px !important;
    }
    .arz-list-group-item.violation {
      background-color: rgba(220, 53, 69, 0.05) !important;
      border-left: 3px solid rgba(220, 53, 69, 0.5) !important;
    }
    .arz-modal-dark {
      background-color: #1e2125;
      color: #dee2e6;
      border: 1px solid rgba(255,255,255,0.1);
    }
    /* -------------------------------------- */
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

  function openPromoSettings() {
    const minRank = prompt('С какого ранга начинать искать блат (старый ранг)?\\nПо умолчанию: 1', String(config.promoMinRank ?? 1));
    if (minRank === null) return;
    const jump = prompt('Прыжок больше скольких рангов считать блатом?\\nПо умолчанию: 1 (т.е. прыжок на 2 и более рангов)', String(config.promoJump ?? 1));
    if (jump === null) return;

    const parsedMin = parseInt(minRank, 10);
    const parsedJump = parseInt(jump, 10);

    if (isNaN(parsedMin) || parsedMin < 1 || isNaN(parsedJump) || parsedJump < 1) {
      alert('Ошибка: введите корректные числа (1 или больше).');
      return;
    }

    config = { ...config, promoMinRank: parsedMin, promoJump: parsedJump };
    saveConfig(config);

    // Если открыто окно Анти-Блата, обновляем его
    if (document.getElementById('arz-generic-modal')?.classList.contains('show')) {
      checkPromotions();
    }
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

  function parseDateInput(text) {
    if (!text) return null;
    const trimmed = String(text).trim();
    const dotted = trimmed.match(/^(\d{2})\.(\d{2})\.(\d{4})$/);
    if (dotted) {
      const [, dd, mm, yyyy] = dotted;
      return new Date(Number(yyyy), Number(mm) - 1, Number(dd));
    }
    const iso = trimmed.match(/^(\d{4})-(\d{2})-(\d{2})$/);
    if (iso) {
      const [, yyyy, mm, dd] = iso;
      return new Date(Number(yyyy), Number(mm) - 1, Number(dd));
    }
    return null;
  }

  function formatDateInput(date) {
    return `${pad2(date.getDate())}.${pad2(date.getMonth() + 1)}.${date.getFullYear()}`;
  }

  function getWeekRangeMonday(offsetWeeks = 0) {
    const now = new Date();
    const day = now.getDay();
    const diffToMonday = day === 0 ? 6 : day - 1;
    const start = new Date(now);
    start.setDate(now.getDate() - diffToMonday + offsetWeeks * 7);
    start.setHours(0, 0, 0, 0);
    const end = new Date(start);
    end.setDate(start.getDate() + 6);
    end.setHours(23, 59, 59, 999);
    return { start, end };
  }

  function getActiveRange(events = []) {
    if (config.periodMode === PERIOD_MODES.ALL_TIME) {
      if (!events || events.length === 0) return getWeekRangeMonday(0);
      const start = new Date(events[0].time);
      start.setHours(0, 0, 0, 0);
      const end = getNowCap();
      end.setHours(23, 59, 59, 999);
      return { start, end };
    }

    if (config.periodMode === PERIOD_MODES.PREVIOUS) {
      return getWeekRangeMonday(-1);
    }

    if (config.periodMode === PERIOD_MODES.CUSTOM) {
      const start = parseDateInput(config.customPeriodStart);
      const end = parseDateInput(config.customPeriodEnd);
      if (start && end && start <= end) {
        const rangeStart = new Date(start);
        rangeStart.setHours(0, 0, 0, 0);
        const rangeEnd = new Date(end);
        rangeEnd.setHours(23, 59, 59, 999);
        return { start: rangeStart, end: rangeEnd };
      }
      warn('некорректный свой период, используется текущая неделя');
    }

    return getWeekRangeMonday(0);
  }

  function getDayCountInRange(range) {
    return getDaysInRange(range).length;
  }

  function getDaysInRange(range) {
    const days = [];
    const cursor = new Date(range.start);
    cursor.setHours(0, 0, 0, 0);
    const end = new Date(range.end);
    end.setHours(0, 0, 0, 0);

    while (cursor <= end) {
      days.push(new Date(cursor));
      cursor.setDate(cursor.getDate() + 1);
    }

    return days;
  }

  function getPeriodLabel(range) {
    if (config.periodMode === PERIOD_MODES.CURRENT) {
      return 'тек. неделя, с ПН';
    }
    if (config.periodMode === PERIOD_MODES.PREVIOUS) {
      return 'прошлая неделя';
    }
    if (config.periodMode === PERIOD_MODES.ALL_TIME) {
      return 'весь срок';
    }
    return `${formatShortDate(range.start)}–${formatShortDate(range.end)}`;
  }

  function openCustomPeriodSettings() {
    const fallback = config.periodMode === PERIOD_MODES.CUSTOM
      ? getActiveRange()
      : getWeekRangeMonday(-1);
    const startStr = prompt(
      'Начало периода (ДД.ММ.ГГГГ):',
      config.customPeriodStart || formatDateInput(fallback.start)
    );
    if (startStr === null) return;

    const endStr = prompt(
      'Конец периода (ДД.ММ.ГГГГ):',
      config.customPeriodEnd || formatDateInput(fallback.end)
    );
    if (endStr === null) return;

    const start = parseDateInput(startStr);
    const end = parseDateInput(endStr);
    if (!start || !end) {
      alert('Некорректная дата. Формат: ДД.ММ.ГГГГ, например 11.08.2026');
      return;
    }
    if (start > end) {
      alert('Начало периода не может быть позже конца.');
      return;
    }

    start.setHours(0, 0, 0, 0);
    end.setHours(23, 59, 59, 999);
    if (getDayCountInRange({ start, end }) > 31) {
      alert('Максимальная длина периода — 31 день.');
      return;
    }

    config = {
      ...config,
      periodMode: PERIOD_MODES.CUSTOM,
      customPeriodStart: formatDateInput(start),
      customPeriodEnd: formatDateInput(end),
    };
    saveConfig(config);
    lastDisplaySignature = '';
    refresh(true);
    alert(`Период: ${formatDateInput(start)} – ${formatDateInput(end)}`);
  }

  function setPeriodMode(mode) {
    if (!Object.values(PERIOD_MODES).includes(mode)) return;
    if (mode === PERIOD_MODES.CUSTOM) {
      openCustomPeriodSettings();
      return;
    }

    config = {
      ...config,
      periodMode: mode,
      customPeriodStart: null,
      customPeriodEnd: null,
    };
    saveConfig(config);
    lastDisplaySignature = '';
    refresh(true);
  }

  function openPeriodSettings() {
    const choice = prompt(
      'Период расчёта:\n1 — текущая неделя (с понедельника)\n2 — прошлая неделя\n3 — весь срок\n4 — свой период (даты)',
      config.periodMode === PERIOD_MODES.PREVIOUS ? '2'
        : config.periodMode === PERIOD_MODES.ALL_TIME ? '3'
        : config.periodMode === PERIOD_MODES.CUSTOM ? '4' : '1'
    );
    if (choice === null) return;

    const normalized = choice.trim().toLowerCase();
    if (normalized === '1' || normalized === 'текущая') {
      setPeriodMode(PERIOD_MODES.CURRENT);
      alert('Период: текущая неделя (с понедельника).');
      return;
    }
    if (normalized === '2' || normalized === 'прошлая') {
      setPeriodMode(PERIOD_MODES.PREVIOUS);
      const range = getActiveRange();
      alert(`Период: прошлая неделя (${formatShortDate(range.start)} – ${formatShortDate(range.end)}).`);
      return;
    }
    if (normalized === '3' || normalized === 'весь' || normalized === 'весь срок') {
      setPeriodMode(PERIOD_MODES.ALL_TIME);
      alert('Период: весь срок (с первой активности в логах).');
      return;
    }
    if (normalized === '4' || normalized === 'свой') {
      openCustomPeriodSettings();
      return;
    }

    alert('Введите от 1 до 4');
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
    GM_registerMenuCommand('Период расчёта…', openPeriodSettings);
    GM_registerMenuCommand('Текущая неделя', () => setPeriodMode(PERIOD_MODES.CURRENT));
    GM_registerMenuCommand('Прошлая неделя', () => setPeriodMode(PERIOD_MODES.PREVIOUS));
    GM_registerMenuCommand('Свой период (даты)…', openCustomPeriodSettings);
    GM_registerMenuCommand('Настройки прайм-тайма…', openSettings);
    GM_registerMenuCommand('Норма онлайна (часов)…', openNormSettings);
    GM_registerMenuCommand('Настройки Анти-Блата…', openPromoSettings);
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

  function getDailyNormMs(normMs, dayCount = 7) {
    return normMs && dayCount > 0 ? normMs / dayCount : null;
  }

  function getElapsedDaysInRange(range) {
    const now = getNowCap();
    let count = 0;

    for (const day of getDaysInRange(range)) {
      if (day > now) break;
      count += 1;
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

  function clipSessionToRange(session, range) {
    const now = getNowCap().getTime();
    const overlapStart = Math.max(session.start.getTime(), range.start.getTime());
    const overlapEnd = Math.min(session.end.getTime(), range.end.getTime(), now);
    if (overlapEnd <= overlapStart) return null;
    return {
      start: new Date(overlapStart),
      end: new Date(overlapEnd),
    };
  }

  function buildRangeSessions(events, range) {
    return mergeSessions(
      buildAllSessionsFromLogs(events)
        .map((s) => clipSessionToRange(s, range))
        .filter(Boolean)
    );
  }

  function getNormMs() {
    const hours = Number(config.normHours);
    const normHours = Number.isFinite(hours) && hours > 0 ? hours : DEFAULT_CONFIG.normHours;
    return normHours * 3600 * 1000;
  }

  function getEffectiveNormMs(range, elapsedDays) {
    const baseNorm = getNormMs();
    if (config.periodMode === PERIOD_MODES.CURRENT) return baseNorm;
    return baseNorm * (elapsedDays / 7);
  }

  function calculateDailyBreakdown(sessions, range, events) {
    const days = [];
    const now = getNowCap();

    for (const dayDate of getDaysInRange(range)) {
      const dayStart = new Date(dayDate);
      dayStart.setHours(0, 0, 0, 0);
      const dayEnd = new Date(dayDate);
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
        date: new Date(dayStart),
        primeMs,
        totalMs,
        sessionCount: sessionIds.size,
        logins,
        maxPrimeMs,
        isFuture,
        hasActivity: primeMs > 0 || totalMs > 0 || logins > 0,
      });
    }

    return days;
  }

  function calculatePrimeTimeOnline() {
    const events = collectLoginLogEvents();
    const periodRange = getActiveRange(events);
    const sessions = buildRangeSessions(events, periodRange);

    let primeMs = 0;
    let totalMs = 0;
    for (const session of sessions) {
      const sessionMs = session.end.getTime() - session.start.getTime();
      totalMs += sessionMs;
      primeMs += sessionPrimeTimeMs(session.start, session.end, config.startHour, config.endHour);
    }
    primeMs = Math.min(primeMs, totalMs);

    const elapsedDays = getElapsedDaysInRange(periodRange);
    const normMs = getEffectiveNormMs(periodRange, elapsedDays);
    const days = calculateDailyBreakdown(sessions, periodRange, events);
    const dailyNormMs = getDailyNormMs(normMs, elapsedDays);

    const totals = days.reduce((acc, day) => ({
      primeMs: acc.primeMs + day.primeMs,
      totalMs: acc.totalMs + day.totalMs,
      sessionCount: acc.sessionCount + day.sessionCount,
      logins: acc.logins + day.logins,
    }), { primeMs: 0, totalMs: 0, sessionCount: 0, logins: 0 });

    return {
      primeMs,
      totalMs,
      periodRange,
      weekRange: periodRange,
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
      const { periodRange, normMs, dailyNormMs, elapsedDays, totals, primeMs } = lastResult;
      const { start, end } = periodRange;
      const avgPrimeMs = primeMs / elapsedDays;
      const totalPercentNorm = formatPercent(primeMs, normMs);
      const totalPercentWindow = formatPercent(primeMs, getMaxPrimeMsPerDay() * elapsedDays);
      const periodLabel = getPeriodLabel(periodRange);

      title.textContent = `${getLeaderNick()} — прайм-тайм по дням`;
      subtitle.textContent = `${periodLabel}: ${formatShortDate(start)} – ${formatShortDate(end)} · ${pad2(config.startHour)}:00–${pad2(config.endHour)}:00 МСК · расчёт по логам`;

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
            <small>От нормы периода</small>
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
    const { start, end } = result.periodRange;
    return [
      formatDuration(result.primeMs),
      result.normMs ? formatDuration(result.normMs) : '',
      percent,
      result.elapsedDays,
      result.sessionCount,
      config.startHour,
      config.endHour,
      config.normHours,
      config.periodMode,
      formatDateInput(start),
      formatDateInput(end),
    ].join('|');
  }

  function bindPeriodSelect(select) {
    select.addEventListener('change', () => {
      const value = select.value;
      if (value === PERIOD_MODES.CUSTOM) {
        select.value = config.periodMode;
        openCustomPeriodSettings();
        return;
      }
      setPeriodMode(value);
    });
  }

  function createPeriodSelect() {
    const select = document.createElement('select');
    select.className = 'form-select form-select-sm arz-prime-period-select';
    select.setAttribute('aria-label', 'Период расчёта');
    select.innerHTML = `
      <option value="${PERIOD_MODES.CURRENT}">Текущая неделя</option>
      <option value="${PERIOD_MODES.PREVIOUS}">Прошлая неделя</option>
      <option value="${PERIOD_MODES.ALL_TIME}">Весь срок</option>
      <option value="${PERIOD_MODES.CUSTOM}">Свой период…</option>
    `;
    select.value = Object.values(PERIOD_MODES).includes(config.periodMode)
      ? config.periodMode
      : PERIOD_MODES.CURRENT;
    bindPeriodSelect(select);
    return select;
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
      hoursSpan.textContent = `(${pad2(config.startHour)}:00–${pad2(config.endHour)}:00 · ${getPeriodLabel(result.periodRange)})`;
    }

    const periodSelect = row.querySelector('.arz-prime-period-select');
    if (periodSelect && periodSelect.value !== config.periodMode) {
      periodSelect.value = Object.values(PERIOD_MODES).includes(config.periodMode)
        ? config.periodMode
        : PERIOD_MODES.CURRENT;
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
          (${pad2(config.startHour)}:00–${pad2(config.endHour)}:00 · ${getPeriodLabel(result.periodRange)})
        </span>
      </label>
      <div class="d-flex align-items-center gap-2 mb-1 arz-prime-period-wrap"></div>
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

    row.querySelector('.arz-prime-period-wrap')?.appendChild(createPeriodSelect());

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

  // --- ПРОВЕРКИ И УМНЫЕ ФИЛЬТРЫ ---
  function ignorePromotion(signature) {
    if (!config.ignoredPromotions) config.ignoredPromotions = [];
    if (!config.ignoredPromotions.includes(signature)) {
      config.ignoredPromotions.push(signature);
      saveConfig(config);
      checkPromotions();
    }
  }

  function showGenericModal(title, html) {
    let modal = document.getElementById('arz-generic-modal');
    if (!modal) {
      modal = document.createElement('div');
      modal.className = 'modal fade modal-leader-stat';
      modal.id = 'arz-generic-modal';
      modal.tabIndex = -1;
      modal.setAttribute('aria-hidden', 'true');
      modal.innerHTML = `
        <div class="modal-dialog modal-lg">
          <div class="modal-content arz-modal-dark">
            <div class="modal-header" style="border-bottom: 1px solid rgba(255,255,255,0.05);">
              <h1 class="modal-title fs-5" id="arz-generic-modal-title" style="color: #e9ecef;"></h1>
              <button type="button" class="btn-close btn-close-white" data-bs-dismiss="modal" aria-label="Закрыть"></button>
            </div>
            <div class="modal-body text-start" id="arz-generic-modal-body">
            </div>
          </div>
        </div>
      `;
      document.body.appendChild(modal);
    }
    document.getElementById('arz-generic-modal-title').textContent = title;
    document.getElementById('arz-generic-modal-body').innerHTML = html;
    if (typeof bootstrap !== 'undefined' && bootstrap.Modal) {
      bootstrap.Modal.getOrCreateInstance(modal).show();
    }
  }

  function checkPraises() {
    const logs = document.querySelectorAll('#actions .new-leader-log-item');
    if (!logs.length) {
      alert('Логи действий в организации не найдены. Убедитесь, что вы находитесь во вкладке "Действия в организации".');
      return;
    }

    const praises = [];
    logs.forEach(log => {
      const textNode = log.querySelector('span');
      const metaNode = log.querySelector('.text-muted.small');
      if (!textNode || !metaNode) return;

      const text = textNode.textContent.trim();
      if (text.toLowerCase().includes('похвал')) {
        const meta = metaNode.textContent.trim();
        const dateMatch = meta.match(/(\d{2}\.\d{2}\.\d{4})/);
        const date = dateMatch ? dateMatch[1] : 'Неизвестная дата';

        const nicks = text.match(/[A-Za-z0-9]+_[A-Za-z0-9]+/g) || [];
        const receiver = nicks.length >= 2 ? nicks[1] : (nicks[0] || 'Неизвестный');

        praises.push({ date, receiver, giver: nicks[0] || 'Неизвестный', text, fullMeta: meta });
      }
    });

    if (praises.length === 0) {
      alert('В загруженных логах не найдено ни одной выдачи похвалы.');
      return;
    }

    const stats = {};
    praises.forEach(p => {
      if (!stats[p.date]) stats[p.date] = {};
      if (!stats[p.date][p.receiver]) stats[p.date][p.receiver] = [];
      stats[p.date][p.receiver].push(p);
    });

    let resultHtml = `<div class="p-2">`;
    let hasViolations = false;

    for (const [date, receivers] of Object.entries(stats)) {
      for (const [receiver, items] of Object.entries(receivers)) {
        if (items.length > 2) {
          hasViolations = true;
          resultHtml += `<div class="alert alert-danger mb-3">
            <strong class="d-block mb-1">Нарушение! Игрок ${receiver} получил ${items.length} похвал(ы) за ${date}.</strong>
            <ul class="mb-0 mt-2 ps-3" style="font-size: 0.9em; opacity: 0.9;">
              ${items.map(i => `<li class="mb-1"><strong>${i.fullMeta}</strong> — ${i.text}</li>`).join('')}
            </ul>
          </div>`;
        }
      }
    }

    if (!hasViolations) {
      resultHtml += `<div class="alert alert-success arz-glass-card px-3 py-2" style="border-left: 3px solid rgba(25,135,84,0.5); color: #a3cfbb; background-color: rgba(25,135,84,0.1);">Нарушений не найдено. Все выдачи похвал в пределах нормы (не более 2-х в день на человека).</div>`;
    }

    resultHtml += `<h6 class="mt-4 mb-3" style="color: #dee2e6;"><i class="fa-solid fa-chart-simple me-2"></i>Статистика (найдено ${praises.length} похвал):</h6>
      <ul class="list-group list-group-sm" style="max-height: 400px; overflow-y: auto; padding-right: 5px;">`;
    for (const [date, receivers] of Object.entries(stats)) {
      for (const [receiver, items] of Object.entries(receivers)) {
        const isViolation = items.length > 2;
        resultHtml += `<li class="list-group-item d-flex justify-content-between align-items-center arz-list-group-item ${isViolation ? 'violation' : ''}">
          <div>
            <span class="fw-bold" style="color: ${isViolation ? '#ff8787' : '#e9ecef'};">${receiver}</span>
            <span class="text-muted ms-2" style="font-size: 0.85em;">(${date})</span>
          </div>
          <span class="badge ${isViolation ? 'bg-danger' : 'bg-primary'} rounded-pill fs-6" style="opacity: 0.9;">${items.length}</span>
        </li>`;
      }
    }
    resultHtml += `</ul></div>`;

    showGenericModal('Проверка похвал (макс 2 в день)', resultHtml);
  }

  function getPromotionSignature(log) {
    return `${log.date}_${log.receiver}_${log.oldRank}_${log.newRank}`;
  }

  function checkPromotions() {
    const logs = document.querySelectorAll('#actions .new-leader-log-item');
    if (!logs.length) {
      alert('Логи действий в организации не найдены. Убедитесь, что вы находитесь во вкладке "Действия в организации".');
      return;
    }

    const pMinRank = config.promoMinRank ?? 1;
    const pJump = config.promoJump ?? 1;

    const promos = [];
    logs.forEach(log => {
      const textNode = log.querySelector('span');
      const metaNode = log.querySelector('.text-muted.small');
      if (!textNode || !metaNode) return;

      const text = textNode.textContent.trim();
      const meta = metaNode.textContent.trim();

      const fullMatch = text.match(/с\s+(\d+)\s+на\s+(\d+)/i);
      if (fullMatch) {
        const oldRank = parseInt(fullMatch[1], 10);
        const newRank = parseInt(fullMatch[2], 10);

        if (newRank > oldRank && (newRank - oldRank > pJump) && oldRank >= pMinRank) {
          const dateMatch = meta.match(/(\d{2}\.\d{2}\.\d{4})/);
          const date = dateMatch ? dateMatch[1] : 'Неизвестная дата';
          const nicks = text.match(/[A-Za-z0-9]+_[A-Za-z0-9]+/g) || [];
          const receiver = nicks.length >= 2 ? nicks[1] : (nicks[0] || 'Неизвестный');

          const promo = { date, receiver, oldRank, newRank, text, fullMeta: meta };
          promo.signature = getPromotionSignature(promo);

          if (!config.ignoredPromotions.includes(promo.signature)) {
            promos.push(promo);
          }
        }
      }
    });

    let html = `<div class="p-2">`;
    html += `
    <div class="d-flex justify-content-between align-items-center mb-3 pb-2" style="border-bottom: 1px solid rgba(255,255,255,0.05);">
      <span class="text-muted small"><i class="fa-solid fa-filter me-1"></i>Фильтр: от <strong>${pMinRank}</strong> ранга, прыжок > <strong>${pJump}</strong></span>
      <button class="btn btn-sm btn-arz btn-arz-warning arz-promo-settings-btn py-1 px-2" style="font-size: 0.85em;">
        <i class="fa-solid fa-gear me-1"></i>Настроить
      </button>
    </div>
    `;
    if (promos.length === 0) {
      html += `<div class="alert alert-success arz-glass-card px-3 py-2" style="border-left: 3px solid rgba(25,135,84,0.5); color: #a3cfbb; background-color: rgba(25,135,84,0.1);">Подозрительных повышений не найдено. Все чисто!</div>`;
    } else {
      html += `<div class="alert alert-warning arz-glass-card px-3 py-2 mb-3" style="border-left: 3px solid rgba(255,193,7,0.5); color: #ffda6a; background-color: rgba(255,193,7,0.1);">Найдено <strong>${promos.length}</strong> подозрительных повышений:</div>`;
      promos.forEach(p => {
        html += `
        <div class="arz-glass-card arz-glass-card-warning mb-2">
          <div class="p-3 d-flex justify-content-between align-items-center">
            <div>
              <strong style="color: #ff8787; font-size: 1.05em;">${p.receiver}</strong>
              <span class="badge" style="background-color: rgba(255,255,255,0.1); color: #dee2e6; margin-left: 8px; font-size: 0.9em;">${p.oldRank} <i class="fa-solid fa-arrow-right mx-1" style="opacity: 0.5; font-size: 0.8em;"></i> ${p.newRank}</span>
              <div class="small mt-1" style="color: #aab2bd;">${p.fullMeta} — <span style="opacity: 0.8;">${p.text}</span></div>
            </div>
            <button class="btn btn-sm btn-arz btn-arz-success ms-3 text-nowrap arz-ignore-promo-btn" data-signature="${p.signature}">
              <i class="fa-solid fa-check me-1"></i>Норма
            </button>
          </div>
        </div>`;
      });
    }
    html += `</div>`;
    showGenericModal('Проверка повышений (Анти-Блат)', html);

    // Вешаем обработчики после рендера модалки, так как inline-onclick не работает в песочнице Tampermonkey
    const modalBody = document.getElementById('arz-generic-modal-body');
    if (modalBody) {
      modalBody.querySelectorAll('.arz-ignore-promo-btn').forEach(btn => {
        btn.addEventListener('click', (e) => {
          e.preventDefault();
          ignorePromotion(btn.dataset.signature);
        });
      });
      const settingsBtn = modalBody.querySelector('.arz-promo-settings-btn');
      if (settingsBtn) {
        settingsBtn.addEventListener('click', (e) => {
          e.preventDefault();
          openPromoSettings();
        });
      }
    }
  }

  function injectActionButtons() {
    const searchWrapper = document.querySelector('#actions .search-in-accord');
    if (searchWrapper && !document.getElementById('btn-check-praises')) {
      searchWrapper.classList.add('d-flex', 'align-items-center', 'gap-2', 'flex-wrap');

      const input = searchWrapper.querySelector('input');
      if (input) {
        input.classList.add('flex-grow-1');
      }

      const btnPraises = document.createElement('button');
      btnPraises.id = 'btn-check-praises';
      btnPraises.className = 'btn btn-sm btn-arz btn-arz-primary';
      btnPraises.style.whiteSpace = 'nowrap';
      btnPraises.innerHTML = '<i class="fa-solid fa-ranking-star me-2"></i>Похвалы';
      btnPraises.addEventListener('click', (e) => {
        e.preventDefault();
        checkPraises();
      });

      const btnPromos = document.createElement('button');
      btnPromos.id = 'btn-check-promos';
      btnPromos.className = 'btn btn-sm btn-arz btn-arz-warning';
      btnPromos.style.whiteSpace = 'nowrap';
      btnPromos.innerHTML = '<i class="fa-solid fa-angles-up me-2"></i>Анти-Блат';
      btnPromos.addEventListener('click', (e) => {
        e.preventDefault();
        checkPromotions();
      });

      searchWrapper.appendChild(btnPraises);
      searchWrapper.appendChild(btnPromos);
    }
  }

  // ------------------------------------

  function isOurMutation(mutation) {
    const checkNode = (node) => {
      if (!node || node.nodeType !== 1) return false;
      return node.id === WIDGET_ID ||
        node.id === MODAL_ID ||
        node.id === 'btn-check-praises' ||
        node.id === 'btn-check-promos' ||
        node.id === 'arz-generic-modal' ||
        !!node.closest?.(`#${WIDGET_ID}, #${MODAL_ID}, #btn-check-praises, #btn-check-promos, #arz-generic-modal`);
    };

    if (checkNode(mutation.target)) return true;

    return [...mutation.addedNodes, ...mutation.removedNodes].some(checkNode);
  }

  function refresh(forceLog) {
    const debug = config.debug || forceLog;

    if (!document.body) return;

    const onPage = /checklead\.php/i.test(location.href) || /fractions\.php/i.test(location.href) ||
      document.body.innerText.includes('Статистика за неделю');
    if (!onPage) return;

    // Добавляем кнопки действий (похвалы, анти-блат)
    injectActionButtons();

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
        period: getPeriodLabel(result.periodRange),
        range: `${formatShortDate(result.periodRange.start)} – ${formatShortDate(result.periodRange.end)}`,
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
  console.log(LOG_PREFIX, 'скрипт v1.8.0 запущен на', location.href);
})();
```
