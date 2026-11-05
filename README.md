import { api } from './repository.js';
import * as V from './views.js';
import { STATUS } from './constants.js';
import { debounce, normalizeArabic } from './utils.js';
import { validateWish, validateOffer } from './validators.js';

const $ = (sel, root = document) => root.querySelector(sel);
const $$ = (sel, root = document) => [...root.querySelectorAll(sel)];
const viewEl = $('#view');
const modal = $('#modal');
const toastEl = $('#toast');

const state = {
  route: 'home',
  q: '', category: 'all', city: 'all',
  tab: 'published',
  wishes: [], offers: [],
  loading: true, error: false,
};

/* ================= selectors ================= */
const findWish = (id) => state.wishes.find((w) => w.id === id);

function filteredWishes() {
  const q = normalizeArabic(state.q.trim());
  return state.wishes.filter((w) =>
    (state.category === 'all' || w.category === state.category) &&
    (state.city === 'all' || w.city === state.city) &&
    (!q || normalizeArabic(`${w.title} ${w.desc} ${w.city} ${w.author}`).includes(q)));
}

const mineBuckets = () => ({
  published: state.wishes.filter((w) => w.isMine),
  joined: state.wishes.filter((w) => w.joined),
  saved: state.wishes.filter((w) => w.saved),
  offers: state.offers,
});

/* ================= rendering ================= */
function renderHome() {
  viewEl.innerHTML = V.homeShell(state);
  renderList();
}

function renderList() {
  const list = $('#list');
  if (!list) return;
  $('#stats').innerHTML = state.loading ? '' : V.stats(state.wishes);

  if (state.loading) { list.innerHTML = V.skeletons(6); $('#count').textContent = ''; return; }
  if (state.error) { list.innerHTML = V.errorState(); $('#count').textContent = ''; return; }

  const items = filteredWishes();
  $('#count').textContent = `${items.length} من ${state.wishes.length} رغبة`;

  if (items.length) { list.innerHTML = items.map(V.wishCard).join(''); return; }

  const filtering = state.q || state.category !== 'all' || state.city !== 'all';
  list.innerHTML = filtering
    ? V.emptyState({ art: '🔍', title: 'ما لقينا شيء', text: 'جرّب كلمة ثانية أو غيّر التصنيف والمدينة.', action: '<button class="btn btn--dark" data-action="clear-filters">مسح التصفية</button>' })
    : V.emptyState({ art: '🌱', title: 'ما أحد كتب ودّه بعد', text: 'كن أول من يبدأ.', action: '<button class="btn btn--primary" data-action="new">+ أضف ودّي</button>' });
}

function renderMine() {
  const buckets = mineBuckets();
  const counts = Object.fromEntries(Object.entries(buckets).map(([k, v]) => [k, v.length]));
  viewEl.innerHTML = V.mineShell(state, counts);
  const list = $('#mine-list');

  if (state.loading) { list.innerHTML = V.skeletons(3); return; }
  if (state.error) { list.innerHTML = V.errorState(); return; }

  const items = buckets[state.tab];
  if (!items.length) {
    const e = V.MINE_EMPTY[state.tab];
    list.innerHTML = V.emptyState({
      art: e.art, title: e.title, text: e.text,
      action: e.cta
        ? '<button class="btn btn--primary" data-action="new">+ أضف ودّي</button>'
        : '<a class="btn btn--dark" href="#/">تصفّح الرغبات</a>',
    });
    return;
  }
  list.innerHTML = state.tab === 'offers' ? items.map(V.offerItem).join('') : items.map(V.wishCard).join('');
}

const renderCurrent = () => (state.route === 'mine' ? renderMine() : renderList());

function render() {
  state.route === 'mine' ? renderMine() : renderHome();
  $$('[data-nav]').forEach((a) =>
    a.dataset.nav === state.route ? a.setAttribute('aria-current', 'page') : a.removeAttribute('aria-current'));
}

/* ================= data ================= */
async function fetchAll() {
  [state.wishes, state.offers] = await Promise.all([api.listWishes(), api.listMyOffers()]);
}

async function bootstrap() {
  state.loading = true; state.error = false;
  renderCurrent();
  try { await fetchAll(); } catch { state.error = true; }
  state.loading = false;
  renderCurrent();
}

/** Run a mutation, re-sync, show feedback. Returns true on success. */
async function run(task, successMsg) {
  try {
    await task();
    await fetchAll();
    renderCurrent();
    if (successMsg) toast(successMsg);
    return true;
  } catch (err) {
    toast(err.message || 'صار خطأ، حاول مرة ثانية.', 'error');
    return false;
  }
}

/* ================= UI helpers ================= */
let toastTimer;
function toast(msg, kind = 'ok') {
  toastEl.textContent = msg;
  toastEl.className = `toast show ${kind === 'error' ? 'error' : ''}`;
  clearTimeout(toastTimer);
  toastTimer = setTimeout(() => toastEl.classList.remove('show'), 2600);
}

function openModal(html) {
  modal.innerHTML = html;
  if (!modal.open) modal.showModal();
  bindCounters(modal);
  (modal.querySelector('input:not([type=hidden]):not(.sr-only), textarea') ?? modal.querySelector('button'))?.focus();
}
const closeModal = () => modal.open && modal.close();

function bindCounters(root) {
  $$('[data-count-for]', root).forEach((el) => {
    const input = $(`[name="${el.dataset.countFor}"]`, root);
    const max = input.maxLength;
    const update = () => { el.textContent = `${input.value.length}/${max}`; };
    input.addEventListener('input', update);
    update();
  });
}

function showErrors(form, errors) {
  $$('[data-error-for]', form).forEach((el) => {
    const msg = errors[el.dataset.errorFor] ?? '';
    el.textContent = msg;
    const input = $(`[name="${el.dataset.errorFor}"]`, form);
    input?.toggleAttribute('aria-invalid', Boolean(msg));
    if (input && msg) input.setAttribute('aria-invalid', 'true');
  });
  form.querySelector('[aria-invalid="true"]')?.focus();
}

/* ================= actions (event delegation) ================= */
const actions = {
  new: () => openModal(V.wishForm(null, api.getName())),
  close: closeModal,
  retry: bootstrap,

  category(_, el) { state.category = el.dataset.value; $('#chips').innerHTML = V.chips(state); renderList(); },
  'clear-filters'() {
    Object.assign(state, { q: '', category: 'all', city: 'all' });
    renderHome();
  },
  tab(_, el) { state.tab = el.dataset.value; renderMine(); },

  join: (id) => run(() => api.toggleJoin(id)),
  save: (id) => run(() => api.toggleSave(id), findWish(id)?.saved ? 'انشالت من المحفوظات' : 'انحفظت ✓'),
  done: (id) => {
    const done = findWish(id)?.status === STATUS.DONE;
    return run(() => api.updateWish(id, { status: done ? STATUS.OPEN : STATUS.DONE }), done ? 'رجعت الرغبة مفتوحة' : 'مبروك! تحققت ودّك 🎉');
  },
  edit: (id) => openModal(V.wishForm(findWish(id))),
  delete: (id) => openModal(V.confirmDelete(findWish(id))),
  offer: (id) => openModal(V.offerForm(findWish(id), api.getName())),
  report: (id) => openModal(V.reportForm(findWish(id))),
};

document.addEventListener('click', (e) => {
  if (e.target === modal) return closeModal(); // backdrop
  const el = e.target.closest('[data-action]');
  if (!el) return;
  const id = el.closest('[data-id]')?.dataset.id;
  actions[el.dataset.action]?.(id, el);
});

/* ================= forms ================= */
const formHandlers = {
  async wish(data, form) {
    const { valid, errors, values } = validateWish(data);
    if (!valid) return showErrors(form, errors);
    api.setName(values.author);
    const ok = data.id
      ? await run(() => api.updateWish(data.id, values), 'انحفظ التعديل ✓')
      : await run(() => api.createWish(values), 'اننشر ودّك 🎉');
    if (ok) closeModal();
  },

  async offer(data, form) {
    const { valid, errors, values } = validateOffer(data);
    if (!valid) return showErrors(form, errors);
    api.setName(values.author);
    if (await run(() => api.addOffer(data.id, values), 'وصل اقتراحك ✓')) closeModal();
  },

  async report(data) {
    if (await run(() => api.reportWish(data.id, data.reason), 'وصلنا بلاغك، شكراً لك.')) closeModal();
  },

  async delete(data) {
    if (await run(() => api.deleteWish(data.id), 'انحذفت الرغبة')) closeModal();
  },
};

document.addEventListener('submit', async (e) => {
  const form = e.target.closest('form[data-form]');
  if (!form) return;
  e.preventDefault();
  const submit = form.querySelector('[type=submit]');
  submit.disabled = true;
  try { await formHandlers[form.dataset.form](Object.fromEntries(new FormData(form)), form); }
  finally { submit.disabled = false; }
});

/* ================= search / city ================= */
document.addEventListener('input', debounce((e) => {
  if (e.target.id !== 'search') return;
  state.q = e.target.value;
  renderList();
}, 150));

document.addEventListener('change', (e) => {
  if (e.target.id !== 'city') return;
  state.city = e.target.value;
  renderList();
});

/* ================= routing ================= */
function onRoute() {
  state.route = location.hash === '#/mine' ? 'mine' : 'home';
  render();
  window.scrollTo({ top: 0 });
}
window.addEventListener('hashchange', onRoute);

/* ================= boot ================= */
onRoute();
bootstrap();

if ('serviceWorker' in navigator && location.protocol.startsWith('http')) {
  navigator.serviceWorker.register('sw.js').catch(() => {});
}
