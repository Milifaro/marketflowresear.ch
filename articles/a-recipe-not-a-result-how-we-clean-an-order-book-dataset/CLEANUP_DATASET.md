# ОЧИСТКА ДАТАСЕТУ ПОДІЙ-210 — виконуваний рецепт

Як з повного датасету вікон 210 с × 8 фіч отримати **очищений датасет**, у якому
залишається майже вся розворотна маса, і як при цьому будуються три службові
множини: **повне тихе ПСА**, **шум 1.1** і **група дрібних класів**, що
викидаються слідом.

Документ написаний за `DIARY_STANDARD.md`: це рецепт, а не опис результату.
Числа нижче — з BTC (зріз до 08.08.2026), вони **орієнтир з нашого зрізу, не
константи**: на іншій монеті й іншому періоді вони будуть іншими, збігатися має
СТРУКТУРА висновків, а не цифри. Код у розділі «Повний код» вставляється
автоматично з робочих файлів скриптом `build_cleanup_doc.py` — руками його тут
не правити.

---

## 1. Вхід

**Формат.** `media/analyst/class4/cache08/<COIN>_data.npz` — масив `X` форми
(вікон, 8, 210): 8 ордербук-фіч, посекундно, вікно 210 с; `ts` — час початку
вікна (UTC, секунди). Порядок фіч: `const_resist, resist_plus, resist_minus,
const_support, support_plus, support_minus, vol_buy, vol_sell`.

**Мінімальний обсяг.** Мітка «тихе ПСА» причинна і потребує норми за 28 діб, тож
датасет коротший за ~35 діб дає порожній прогрів і нема на чому будувати полюси.
Практичний мінімум — 3 місяці посекундних даних; у нас 56263 вікон
за 175 діб.

**Головне правило задачі.** На вхід алгоритмів іде РІВНО вміст вікна:
`log1p(x/0.01)` плюс робастний масштаб по каналу (медіана та IQR), порахований
**на тій самій підмножині**, яку кластеризуємо. Ні ціни, ні часу, ні відповідей
наших детекторів на вході немає — розмітка зигзага і мітка розвороту
прикладаються ПОТІМ, як перевірка. Це і робить числа «скільки розворотів у
класі» осмисленими: клас знайдено не за розворотами.

**Що потрібно мати заздалегідь** (нічого з цього ланцюг не будує):

| що | чим будується | навіщо в цьому рецепті |
|---|---|---|
| `cache08/<COIN>_data.npz` | `class4.py data08 <COIN>` | сам датасет вікон |
| `cache08/<COIN>_poles.npz` | той самий етап | полюси ПСА → мітка тихого ПСА |
| `models08/<COIN>_pcaquiet.pt` + `.json` | `class4.py prod08 <COIN>` | модель тихого ПСА для прогріву |

---

## 2. Порядок команд

```bash
python3 class4.py data08 <COIN>      # кеш датасету + полюси ПСА
python3 class4.py prod08 <COIN>      # продакшен-моделі 4 класів (треба pcaquiet)
python3 zzclust.py clean <COIN>      # весь механічний ланцюг нижче, кроки 1-8
# далі — РІШЕННЯ ЛЮДИНИ за таблицею кандидатів, яку надрукує clean:
python3 zzclust.py rev4b <COIN>      # зняти обрані дрібні класи
python3 zzclust.py rev4c <COIN>      # (за потреби) зняти клас цілком, крім вибраних вікон
python3 zzclust.py revmap <COIN> rev4b   # перевірка на РІВНІ ПЕРЕЛОМІВ
```

`clean` пропускає кроки, чий вихід уже лежить у `media/analyst/zz`; `--force`
перераховує все.

---

## 3. Кроки

### Крок 1 — розбиття всього датасету
**Команда:** `zzclust.py all <COIN>` (усередині `clean`).
**Чому саме так.** Спершу треба знати, на що датасет розпадається САМ, без
жодних наших міток. PCA-32 → k-середні K=2..10 (K обирається за **відривом
силуету над жорстким нулем**, не за силуетом: k-середні завжди повертають рівно
K класів, а силует майже завжди падає з K) і паралельно PCA-32 → UMAP-10 →
HDBSCAN. UMAP-2 — тільки карта для очей, кластери на площині вдруге не шукаються.
**Контроль.** HDBSCAN: 3 класи, шум
71.7%, стійкість 0.7243
проти 0.1709 у жорсткого нуля. Нуль при цьому дав
11 «класів» при шумі 78.3% —
**ні k, ні частка шуму не розрізняють структуру від її відсутності**, розрізняє
лише стійкість повної перебудови на іншому сіді.

### Крок 2 — другий рівень: розбиття самого шуму
**Команда:** `zzclust.py subhdb <COIN> all`.
**Чому саме так.** Шум HDBSCAN — не «невпізнані» вікна, а тиха частина шкали;
всередині нього k-середні стабільно знаходять гучніший шар. Масштаб фіч
перераховується на цій підмножині, `mcs` масштабується з її розміром.
**Контроль.** 40334 вікон шуму, K=2: «гучний» клас
4463 вікон з ліфтом розворотів
×4.578, «тихий»
35871 з ×0.555.

### Крок 3 — третій рівень: клас **шум 1.1**
**Команда:** `zzclust.py deep <COIN> all_nz 1 2`.
**Чому саме так.** Тихий клас другого рівня ділиться навпіл ще раз; нижня
половина і є «шум 1.1» — практично мертва на розвороти частина датасету.
**Контроль.** 1.1 — 18320 вікон, 16 розворотних
(×0.28); відрив силуету над жорстким нулем
0.059 (на першому рівні він у 4 рази більший) — це
**робочий розріз континууму, а не окремий стан**, і так його й треба називати.

### Крок 4 — **повне тихе ПСА**
**Команда:** `zzclust.py pqwarm <COIN>`.
**Чому саме так.** Мітка тихого ПСА причинна (норма 28 діб), тож на початку
датасету її не існує. Але МОДЕЛІ норма не потрібна — на вхід їй ідуть самі 1680
чисел вікна. Тому продакшен-модель проганяється по ВСЬОМУ датасету, включно з
прогрівом; ваги і поріг беруться з картки як є, нічого не навчається.
**Контроль.** Разом 4598 спрацювань на 56263 вікон. На
розміченій частині 4363/46390
(9.41%) при мітці 9.55%,
точність 0.778, повнота 0.766; на
прогріві 235/9873 (2.38%).
**Чесно:** на прогріві звіряти нема з чим — спрацювання там не є ні влучанням,
ні помилкою.

### Крок 5 — **очищений датасет**
**Команда:** `zzclust.py revset <COIN>`.
**Чому саме так.** Очищений датасет = всі вікна − шум 1.1 − повне тихе ПСА.
Множини перетинаються, тож викидається їх **обʼєднання**, а не сума.
**Контроль.** 37519 вікон з 56263; розворотних
847 — це 98.1% усієї
розворотної маси датасету (863 вікон). Тобто **третину датасету
викинуто ціною одиниць розворотних вікон** — це і є критерій, що очистка вдалась.

### Крок 6 — подія 210×6 (без const-стін)
**Команда:** `zzclust.py rev6 <COIN>`.
**Чому саме так.** Перевірка, чи беруть стіни участь у поділі. Змінюється РІВНО
склад входу; масштаб перераховується на шести каналах.
**Контроль.** k-середні майже не помічають втрати (ARI з прогоном на 8 фічах
0.968), а HDBSCAN втрачає саме
СТІННІ дрібні класи. Найрозворотніший клас: 8015 вікон,
430 розворотів, ×2.376.

### Крок 7 — подія 210×4 і зняття тихих класів
**Команда:** `zzclust.py rev4 <COIN>`.
**Чому саме так.** Два рухи: (а) з множини викидаються всі класи HDBSCAN
попереднього кроку, крім найрозворотнішого — правило механічне, бо номери
класів довільні, а змістовним є лише порядок за ліфтом (`_auto_drop`); шум
**не чіпаємо** — у ньому щоразу лишається близько третини розворотних вікон;
(б) на вхід ідуть лише 4 потокові фічі: приплив і відтік обох стін.
**Контроль.** 34949 вікон, 839 розворотних (база
2.40%); найрозворотніший клас HDBSCAN 11053
вікон / 557 розворотів / ×2.099. Звірка з кроком 6:
ARI k-середніх 0.691.

### Крок 8 — «а HDBSCAN на 10 класів?»
**Команда:** `zzclust.py mcs <COIN> rev4 10`.
**Чому саме так.** У HDBSCAN **немає K** — кількість класів виходить зі
щільності, задається лише `min_cluster_size`. Перебір іде на один раз
побудованому просторі (інакше кожне значення коштувало б повного прогону), а
обране значення проганяється ПОВНИМ конвеєром із нулем і стійкістю.
**Контроль.** Перебір немонотонний: сусідні `mcs` дають то 3,
то 12 класів. Обране mcs=25 на
фіксованому просторі дало 12
класів, а повна перебудова — 5 (шум
59.5%). **Це головне попередження кроку.**

### Крок 9 — зняття дрібних класів (РІШЕННЯ ЛЮДИНИ)
**Команда:** `zzclust.py rev4b <COIN>` зі списком `drop`.
**Чому не автоматично.** Дрібні класи в десятки-сотні вікон статистично від бази
не відрізняються (q 0.18-0.59), і викидати їх — вибір, а не висновок.
`_drop_candidates` друкує таблицю кандидатів (ліфт < 1), рішення приймає людина.

**Контроль.** 32736 вікон, 832 розворотних
(база 2.54%); найрозворотніший клас 12340 вікон /
583 розворотів / ×1.859; звірка з кроком 7: ARI
k-середніх 0.881, Жаккар
розворотного класу 0.841
— **вісь має лишитись на місці**, інакше звуження зайшло надто далеко.


### Крок 10 — перевірка на РІВНІ ПЕРЕЛОМІВ
**Команда:** `zzclust.py revmap <COIN> rev4b`.
**Чому саме так.** «N розворотних вікон у класі» нічого не каже, поки не відомо,
СКІЛЬКОМ різним переломам вони належать: вікна йдуть кроком 210 с, і в смугу
±15 хв навколо пивота їх влазить до девʼяти. Метрика — **частка перелому в
класі** (перелом цілком в одному класі → 1.0; 3% вікон у шумі і 97% у класі →
0.03 і 0.97), бо сира кількість зважує довгі грона.
**Контроль.** 832 вікон = 109 переломів
(7.63 вікна на перелом). Клас, який ми знімаємо на кроці 11,
має виглядати так: 10 вікон від 8 РІЗНИХ переломів,
домінує в 0, тримає цілком 0, середня частка
перелому 0.017. Це не «свої» розвороти, а поодинокі краї гронів.


### Крок 11 — зняти клас цілком, крім названих вікон
**Команда:** `zzclust.py rev4c <COIN>`.
**Чому саме так.** Якщо клас не домінує в жодному переломі, він знімається
повністю; вікна, які вирішено лишити, шукаються **за часом пивота**, а не за
номером рядка — номери класів залежать від сіда, час пивота ні.
**Контроль і СТОП-СИГНАЛ.** 31212 вікон, 823
розворотних (база 2.64%). Але HDBSCAN тут **вироджується**:
max_share 0.9821 (стеля лінії 0.65) при стійкості
0.056 — проти
0.6851 на попередньому кроці. Читати
це треба як межу: **датасет дозволяє приблизно три звуження, четверте зʼїдає
саму структуру**; k-середні при цьому ще тримаються (ARI з попереднім кроком
0.938), тобто вироджується
саме щільнісне розбиття.


---

## 4. Пастки

1. **k і частка шуму нічого не доводять.** Жорсткий нуль (клітинки перемішано
   МІЖ вікнами по кожній колонці окремо) дає то 6, то 11, то 39 «класів» при
   такій самій частці шуму, як у реальних даних. Судить **лише стійкість** —
   ARI повної перебудови на іншому сіді проти такої ж перебудови нуля.
2. **Стійкість при фіксованому просторі завищена.** Перебір `mcs` іде на одному
   разі побудованому UMAP і дає більше класів, ніж повний прогін (у нас 12
   проти 5). Рапортувати можна лише числа повної перебудови.
3. **max_share ≤ 0.65, і шум у нього не входить.** Поділ «майже все проти дрібки»
   відтворюється сам собою і дає високу стійкість ні з чого. Якщо max_share
   вище стелі — розбиття тримається на одному великому класі, і його «стійкість»
   не є доказом.
4. **Вироджений нуль не порівнюється.** Коли жорсткий нуль не знаходить ЖОДНОГО
   класу (100% шуму), його ARI дорівнює 1.000 — це не «нуль стійкіший», це
   відсутність розбиття. У JSON для цього пишеться `null.k` і `null.max_share`.
5. **Шум не викидати.** У ньому щоразу лишається близько третини розворотних
   вікон, і це не «невпізнані» події, а тихі краї гронів, гучна середина яких
   лежить у класі.
6. **Гронування.** Рахувати переломи, а не вікна: одне грона з девʼяти вікон
   легко виглядає як девʼять подій. Будь-яке твердження про клас перевіряти
   етапом `revmap`.
7. **База розворотів росте механічно.** Кожне звуження робиться за міткою,
   знайденою на ТИХ САМИХ даних, тож частка розворотів у залишку зростає
   арифметично. Ліфти сусідніх кроків між собою **не порівнюються** — у них
   різні бази; порівнювати можна лише ARI і збіг міток.
8. **Масштаб фіч — на своїй підмножині.** Інакше в підвибірку заходить
   константа ззовні, і два етапи вже не є незалежними.
9. **Склад входу тягнеться за тегом.** Похідні етапи (`kmaps`, `mcs`) мусять
   брати `cfg.cols` прогону, інакше вони ріжуть у просторі 8 фіч поверх карти,
   побудованої на 4.
10. **`min_cluster_size` масштабувати з розміром множини** (у нас 250 на повний
    датасет, далі пропорційно), інакше на підмножинах він або зʼїдає всі класи,
    або плодить сотні.

---

## 5. Контрольні числа (BTC, зріз до 08.08.2026)

Допуски — для повторного прогону на ТОМУ САМОМУ зрізі (недетермінізм UMAP і
порядку потоків). На іншій монеті збігатися має структура, не числа.

| крок | величина | значення | допуск |
|---|---|---|---|
| `all` | вікон у датасеті | 56263 | точно |
| `all` | розворотних вікон (геометрія) | 863 | точно |
| `all` | класів HDBSCAN | 3 | ±1 |
| `all` | шум HDBSCAN | 71.7% | ±3 в.п. |
| `all` | стійкість / нуль | 0.7243 / 0.1709 | ±0.10 |
| `deep` | шум 1.1: вікон | 18320 | ±2% |
| `deep` | шум 1.1: розворотних | 16 | ±3 |
| `pqwarm` | повне тихе ПСА: спрацювань | 4598 | точно |
| `pqwarm` | з них на прогріві | 235 | точно |
| `revset` | очищений датасет: вікон | 37519 | точно |
| `revset` | у ньому розворотних | 847 з 863 | точно |
| `rev6` | ARI k-середніх з прогоном на 8 фічах | 0.9675 | ±0.05 |
| `rev4` | вікон | 34949 | точно |
| `rev4` | найрозворотніший клас: ліфт | 2.099 | ±0.15 |
| `mcs` | класів: фіксований простір / повний прогін | 12 / 5 | ±2 / ±1 |
| `rev4b` | вікон | 32736 | точно |
| `rev4b` | ARI k-середніх з попереднім кроком | 0.881 | ±0.05 |
| `rev4c` | max_share (СТОП-СИГНАЛ) | 0.9821 | > 0.9 = вироджено |

---

## 6. Що зміниться на іншій монеті або іншому датасеті

- **Номери класів довільні.** Ніде не покладатись на «клас 0» — лише на порядок
  за ліфтом розворотів (`_auto_drop`) або на явний список після перегляду
  таблиці кандидатів.
- **`mcs` масштабується з обсягом.** У ланцюгу це вже зроблено пропорційно, але
  для дуже малих монет (менше ~20 тис. вікон) перевіряти вручну.
- **Модель тихого ПСА — своя на кожну монету.** Ваги BTC на іншу монету не
  переносяться; `class4.py prod08 <COIN>` обовʼязковий, інакше кроку 4 немає і
  очищений датасет вийде іншим за складом.
- **Кількість звужень.** На BTC витримало три; далі розбиття вироджується.
  Критерій зупинки не «скільки разів», а `max_share > 0.65` разом із падінням
  стійкості — щойно бачите це, крок відкочується.
- **Зигзаг і мітка розвороту** тут ні на що не впливають на вході, вони лише
  міряють результат. Змінюючи поріг зигзага, ви змінюєте вимірювальний прилад,
  а не очистку.

---

## 7. Чек-лист перед тим, як довіряти результату

- [ ] жорсткий нуль порахований на кожному кроці, де є HDBSCAN;
- [ ] стійкість — через ПОВНУ перебудову на іншому сіді, не перекластеризацію;
- [ ] `max_share` реального розбиття перевірено, не лише нуля;
- [ ] вироджений нуль (0 класів) позначений і не використаний як порівняння;
- [ ] твердження про класи перевірене на РІВНІ ПЕРЕЛОМІВ (`revmap`);
- [ ] ліфти різних кроків не порівнюються між собою;
- [ ] очищений датасет тримає ≥95% розворотних вікон вихідного;
- [ ] склад входу (`cfg.cols`) однаковий у прогоні й у похідних етапах.

---

## 8. Повний код

Вставлено автоматично з робочих файлів. Якщо тут зʼявився коментар
`# !! немає функції` — рецепт розʼїхався з кодом, збирач треба перезапустити.


### `zzclust.py`

```python
def log(*a):
    print(time.strftime("[%H:%M:%S]"), *a, flush=True)


def load_marks(coin="BTC", with_det=True):
    """Вікна датасету + період кожного вікна + дві мітки розвороту."""
    d = np.load(os.path.join(T.CACHE08, f"{coin}_data.npz"), mmap_mode="r")
    n = int(d["X"].shape[0])
    ts_abs = np.asarray(d["ts"]).astype(np.int64)
    px, present, meta = T.price_grid(coin)
    t0 = int(meta["t0"])
    ts = ts_abs - t0
    legs, pivots, sm = T.build_legs(np.asarray(px))
    pt = np.array([p["t"] for p in pivots])
    pk = np.array([p["kind"] for p in pivots])
    per = T._class_vec(ts, np.full(n, WIN), legs, pt, pk).astype(str)
    mid = ts + WIN // 2
    j = np.clip(np.searchsorted(pt, mid), 0, len(pt) - 1)
    j2 = np.clip(j - 1, 0, len(pt) - 1)
    dist = np.minimum(np.abs(pt[j] - mid), np.abs(pt[j2] - mid))
    rev_geo = dist <= T.PIVOT_TOL
    rev_det = np.zeros(n, bool)
    if with_det:
        for tg, _ in SR.ALL_TARGETS:
            R = SR.split_windows(coin, tg)
            rev_det[R["w_rev"]] = True
    return dict(d=d, n=n, ts=ts, ts_abs=ts_abs, per=per, rev_geo=rev_geo,
                rev_det=rev_det, meta=meta, legs=legs, pivots=pivots)


def build_F(d, idx=None, chunk=4096, cols=None):
    """Вхід усіх алгоритмів: log1p(x/0.01) + робастний масштаб по каналу.

    Масштаб рахується на ТІЙ САМІЙ множині, яку далі кластеризуємо (у /zzup це
    підмножина циклу «дно → вершина») — інакше в підвибірку заходила б
    константа ззовні, і два етапи вже не були б незалежні.

    `cols` — які з 8 каналів ідуть НА ВХІД (подія 210×6 = без const-стін).
    RAW завжди лишається на всі 8 фіч: рівні викинутих каналів однаково треба
    показувати в таблиці класів, інакше не видно, чим вони зайняті.
    """
    X = d["X"]
    N = X.shape[0] if idx is None else len(idx)
    cc = np.arange(8) if cols is None else np.asarray(cols, int)
    nc = len(cc)
    F = np.empty((N, nc * WIN), np.float32)
    for a in range(0, N, chunk):
        sl = slice(a, min(a + chunk, N))
        blk = np.asarray(X[sl] if idx is None else X[idx[sl]])[:, cc]
        F[sl] = np.log1p(blk / 0.01).reshape(len(blk), -1)
    A = F.reshape(N, nc, WIN)
    med = np.median(A, axis=(0, 2), keepdims=True)
    iqr = (np.quantile(A, .75, axis=(0, 2), keepdims=True) -
           np.quantile(A, .25, axis=(0, 2), keepdims=True))
    np.divide(A - med, np.where(iqr > 1e-9, iqr, 1.0), out=A)
    RAW = np.empty((N, 8), np.float32)          # сирий середній рівень фічі
    for a in range(0, N, chunk):
        sl = slice(a, min(a + chunk, N))
        blk = np.asarray(X[sl] if idx is None else X[idx[sl]])
        RAW[sl] = blk.mean(axis=2)
    return F, RAW


def kmeans_sweep(F, seed=SEED, npca=32, ks=KS):
    """PCA-32 → k-середні K=2..10 із силуетом, стійкістю і жорстким нулем.

    K обирається за ВІДРИВОМ силуету над нулем: сам силует падає з K майже
    завжди, і «найкращий силует» вибрав би K=2 незалежно від даних.
    """
    from sklearn.decomposition import PCA
    from sklearn.cluster import KMeans
    from sklearn.metrics import silhouette_score, adjusted_rand_score as ARI
    pca = PCA(n_components=min(npca, F.shape[1]), random_state=seed,
              svd_solver="randomized").fit(F)
    Z = pca.transform(F).astype(np.float32)
    evr = pca.explained_variance_ratio_
    rng = np.random.default_rng(seed)
    Fn = F.copy()
    for c in range(Fn.shape[1]):
        rng.shuffle(Fn[:, c])
    Zn = PCA(n_components=min(npca, F.shape[1]), random_state=seed,
             svd_solver="randomized").fit_transform(Fn).astype(np.float32)
    del Fn
    sub = rng.choice(len(Z), min(SIL_N, len(Z)), replace=False)
    rows = []
    for K in ks:
        km = KMeans(n_clusters=K, n_init=10, random_state=seed).fit(Z)
        km2 = KMeans(n_clusters=K, n_init=10, random_state=seed + 1).fit(Z)
        kn = KMeans(n_clusters=K, n_init=10, random_state=seed).fit(Zn)
        kn2 = KMeans(n_clusters=K, n_init=10, random_state=seed + 1).fit(Zn)
        s = float(silhouette_score(Z[sub], km.labels_[sub]))
        sn = float(silhouette_score(Zn[sub], kn.labels_[sub]))
        rows.append({"k": K, "sil": round(s, 4), "sil_null": round(sn, 4),
                     "gap": round(s - sn, 4),
                     "stab": round(float(ARI(km.labels_, km2.labels_)), 4),
                     "stab_null": round(float(ARI(kn.labels_, kn2.labels_)), 4),
                     "inertia": round(float(km.inertia_), 1),
                     "sizes": np.bincount(km.labels_, minlength=K).tolist()})
        log(f"   K={K:2d}: силует {s:.4f} (нуль {sn:.4f}, відрив {s-sn:+.4f}) · "
            f"стійкість {rows[-1]['stab']:.3f} (нуль {rows[-1]['stab_null']:.3f})")
    kbest = max(rows, key=lambda r: r["gap"])["k"]
    klab = KMeans(n_clusters=kbest, n_init=10, random_state=seed).fit(Z).labels_
    log(f"   обрано K={kbest} за відривом силуету над нулем")
    return {"rows": rows, "k_best": kbest, "lab": klab, "Z": Z,
            "evr": [round(float(v), 5) for v in evr],
            "evr_first": round(float(evr[0]), 4),
            "evr_sum": round(float(evr.sum()), 4)}


def cls_table(lab, RAW, rev_geo, rev_det, per, with_noise=True):
    """Паспорт кожного класу: розмір · рівні 8 фіч · РОЗВОРОТИ двома мірами.

    Ліфт — у скільки разів частка розворотів у класі більша за базу всієї
    множини; p — біноміальний тест проти цієї бази, q — поправка БХ на всі
    класи таблиці. Склад за періодами показує, чим клас є геометрично.
    """
    from scipy import stats as sps
    n = len(lab)
    bg = float(rev_geo.mean()) or 1e-9
    bd = float(rev_det.mean()) or 1e-9
    out, pv, pv2 = [], [], []
    lo = -1 if with_noise else 0
    for c in range(lo, int(lab.max()) + 1):
        m = lab == c
        if not m.any():
            continue
        cnt = int(m.sum())
        ng, nd = int(rev_geo[m].sum()), int(rev_det[m].sum())
        p1 = float(sps.binomtest(ng, cnt, min(bg, 1.0)).pvalue)
        p2 = float(sps.binomtest(nd, cnt, min(bd, 1.0)).pvalue)
        pv.append(p1); pv2.append(p2)
        out.append({
            "cls": int(c), "n": cnt, "share": round(cnt / n, 4),
            "n_rev": ng, "rev_share": round(ng / cnt, 4),
            "lift": round(ng / cnt / bg, 3), "p": p1,
            "n_revd": nd, "revd_share": round(nd / cnt, 4),
            "lift_d": round(nd / cnt / bd, 3), "p_d": p2,
            "per": {k: round(float((per[m] == k).mean()), 4) for k in PERIODS},
            "lev": {FEATS8[f]: round(float(RAW[m, f].mean()), 4) for f in range(8)}})
    for r, q1, q2 in zip(out, SR._bh(np.array(pv)), SR._bh(np.array(pv2))):
        r["q"] = float(q1); r["q_d"] = float(q2)
    return out


def _pivot_of(coin, ts_abs):
    """Кожному вікну — найближчий пивот зигзага 2.3% (його номер, час і бік)."""
    px, present, meta = T.price_grid(coin)
    legs, pivots, sm = T.build_legs(np.asarray(px))
    pt = np.array([p["t"] for p in pivots])
    pk = np.array([p["kind"] for p in pivots])
    mid = (np.asarray(ts_abs).astype(np.int64) - int(meta["t0"])) + WIN // 2
    j = np.clip(np.searchsorted(pt, mid), 0, len(pt) - 1)
    j2 = np.clip(j - 1, 0, len(pt) - 1)
    take2 = np.abs(pt[j2] - mid) < np.abs(pt[j] - mid)
    jj = np.where(take2, j2, j)
    return jj, np.abs(pt[jj] - mid), pt, pk, int(meta["t0"])


def _stage(coin, tag, ttl, mask=None, mcs=250, seed=SEED, npca=32, nn=30,
           cols=None):
    """Спільне тіло етапів ① і ②: одна множина вікон, чотири методи, таблиці."""
    os.makedirs(OUT_DIR, exist_ok=True)
    M = load_marks(coin)
    idx = np.flatnonzero(mask) if mask is not None else None
    n = M["n"] if idx is None else len(idx)
    per = M["per"] if idx is None else M["per"][idx]
    rg = M["rev_geo"] if idx is None else M["rev_geo"][idx]
    rd = M["rev_det"] if idx is None else M["rev_det"][idx]
    ts = M["ts_abs"] if idx is None else M["ts_abs"][idx]
    log(f"{coin}: {ttl} — {n} вікон 210 с з {M['n']} · розворотних (геометрія) "
        f"{int(rg.sum())} ({rg.mean()*100:.2f}%) · детекторних {int(rd.sum())} "
        f"({rd.mean()*100:.2f}%) · mcs={mcs}")
    cc = list(range(8)) if cols is None else list(cols)
    if cols is not None:
        log(f"   вхід — подія 210×{len(cc)}: " + ", ".join(FEATS8[c] for c in cc) +
            " (викинуто " + ", ".join(FEATS8[c] for c in range(8) if c not in cc) + ")")
    F, RAW = build_F(M["d"], idx, cols=cc)

    t0 = time.time()
    H = SR._hdb_block(F, seed, mcs, npca=npca, nn=nn, want_u2=True)
    hlab, U2 = H["lab"], H["u2"]
    log(f"   HDBSCAN: класів {H['real']['k']} · шум {H['real']['noise']*100:.1f}% · "
        f"evr {H['real']['evr']} · стійкість {H['real']['ari_seed']} проти "
        f"{H['null']['ari_seed']} у нуля · max_share {H['real']['max_share']} "
        f"(нуль: k={H['null']['k']}, шум {H['null']['noise']*100:.1f}%) · "
        f"{int(time.time()-t0)} с")
    KM = kmeans_sweep(F, seed=seed, npca=npca)
    klab, Z = KM["lab"], KM["Z"]

    from sklearn.metrics import adjusted_rand_score as ARI
    kcls = cls_table(klab, RAW, rg, rd, per, with_noise=False)
    hcls = cls_table(hlab, RAW, rg, rd, per)
    _log_table(kcls, f"k-середні K={KM['k_best']}:")
    _log_table(hcls, "HDBSCAN:")
    pcode = np.array([PERIODS.index(x) if x in PERIODS else len(PERIODS) for x in per])
    ari = {"km_vs_hdb": round(float(ARI(klab, hlab)), 4),
           "km_vs_rev": round(float(ARI(klab, rg.astype(int))), 4),
           "hdb_vs_rev": round(float(ARI(hlab, rg.astype(int))), 4),
           "km_vs_period": round(float(ARI(klab, pcode)), 4),
           "hdb_vs_period": round(float(ARI(hlab, pcode)), 4)}
    log("   ARI: " + " · ".join(f"{k} {v}" for k, v in ari.items()))

    out = {"coin": coin, "stage": tag, "title": ttl, "built": int(time.time()),
           "cfg": {"pca": npca, "umap": 10, "n_neighbors": nn, "mcs": int(mcs),
                   "seed": seed, "win": WIN, "ks": list(KS),
                   "cols": cc, "feats_in": [FEATS8[c] for c in cc],
                   "feats_out": [FEATS8[c] for c in range(8) if c not in cc],
                   "scale": "log1p(x/0.01) + робастний по каналу",
                   "zz_pct": T.LEG_THR * 100, "tol_s": T.PIVOT_TOL},
           "n": {"win": int(n), "dataset": int(M["n"]),
                 "rev_geo": int(rg.sum()), "rev_det": int(rd.sum())},
           "base": {"geo": round(float(rg.mean()), 5),
                    "det": round(float(rd.mean()), 5)},
           "periods": [{"name": k, "n": int((per == k).sum()),
                        "share": round(float((per == k).mean()), 4)}
                       for k in PERIODS],
           "legs": leg_stats(coin),
           "pca": {"evr_first": KM["evr_first"], "evr_sum": KM["evr_sum"],
                   "evr": KM["evr"]},
           "k_sweep": KM["rows"], "k_best": KM["k_best"], "kmeans": kcls,
           "hdb": {"real": H["real"], "null": H["null"]}, "hdb_classes": hcls,
           "ari": ari, "feats": list(FEATS8),
           "t_from": int(ts.min()), "t_to": int(ts.max())}
    p = os.path.join(OUT_DIR, f"{coin}_{tag}.json")
    json.dump(out, open(p, "w"), ensure_ascii=False)
    log(f"{coin}: збережено {os.path.basename(p)} (числа ДО малювання)")
    np.savez_compressed(os.path.join(OUT_DIR, f"{coin}_{tag}.npz"),
                        klab=klab, hlab=hlab, u2=U2, z2=Z[:, :2], ts=ts,
                        rev_geo=rg, rev_det=rd, per=per,
                        idx=(idx if idx is not None else np.arange(n)))
    try:
        _charts(coin, tag, ttl, F, Z, U2, klab, hlab, RAW, rg, per, KM["rows"],
                KM["k_best"], H, kcls, hcls, KM["evr"])
    except Exception as e:
        log(f"{coin}: ГРАФІКИ НЕ ВИЙШЛИ ({type(e).__name__}: {e}) — числа в JSON цілі")
    return out


def stage_all(coin="BTC", mcs=250, seed=SEED):
    """② УВЕСЬ ДАТАСЕТ: ті самі чотири методи + розвороти під кожним класом."""
    return _stage(coin, "all", "увесь датасет", mask=None, mcs=mcs, seed=seed)


def stage_subhdb(coin="BTC", tag="down", mcs=None, seed=SEED):
    """ДРУГИЙ РІВЕНЬ: той самий конвеєр, але ТІЛЬКИ на вікнах, які перший
    прогін відніс до ШУМУ.

    Навіщо. Шум — не залишок, а 56-72% датасету, і саме він ковтає розворотні
    вікна (у /zzdown 156 з 431, тобто 36%). Питання: чи розпадається сам шум на
    щось, чи це справді однорідна тиха маса. Масштаб фіч рахується ЗАНОВО на
    цій множині — інакше всі вікна стояли б у вузькій смузі старої шкали і
    HDBSCAN не мав би чого різати.

    Що НЕ змінюється: жорсткий нуль, стійкість повною перебудовою на іншому
    сіді, стеля max_share 0.65 і вибір K за відривом силуету над нулем.
    min_cluster_size зменшується пропорційно розміру підмножини — інакше на
    вчетверо меншій вибірці той самий поріг вимагав би вчетверо щільніших
    згустків.
    """
    p = os.path.join(OUT_DIR, f"{coin}_{tag}.npz")
    if not os.path.exists(p):
        raise SystemExit(f"нема {os.path.basename(p)} — спершу "
                         f"python3 zzclust.py {tag} {coin}")
    z = np.load(p, allow_pickle=True)
    idx, hlab = z["idx"], z["hlab"]
    M = load_marks(coin, with_det=False)
    mask = np.zeros(M["n"], bool)
    mask[idx[hlab < 0]] = True
    base = json.load(open(os.path.join(OUT_DIR, f"{coin}_{tag}.json")))
    del M
    if mcs is None:
        mcs = max(50, int(round(int(base["cfg"]["mcs"]) * mask.sum() /
                                int(base["n"]["win"]))))
    return _stage(coin, f"{tag}_nz",
                  f"ШУМ прогону /{'zz' + tag} окремо ({int(mask.sum())} вікон)",
                  mask=mask, mcs=mcs, seed=seed)


def stage_deep(coin="BTC", tag="all_nz", cls=1, K=2, by="klab", seed=SEED, npca=32):
    """ЩЕ ОДИН РІВЕНЬ УСЕРЕДИНІ ОДНОГО КЛАСУ: k-середні K=2 на його вікнах.

    Дає підкласи з номером батька — «шум клас 1» → «1.1» і «1.2». Масштаб фіч
    перераховується на цій множині (інакше всі вікна стоять у вузькій смузі
    старої шкали), PCA будується заново, мітки зберігаються в npz, тож у той
    самий спосіб можна спуститись іще глибше.

    Поруч із поділом одразу рахується ВІДРИВ СИЛУЕТУ НАД ЖОРСТКИМ НУЛЕМ: сам
    поділ k-середні видадуть завжди, і без відриву це просто розріз континууму.
    Одиниця висновку — не вікно, а ПЕРЕЛОМ: для кожного пивота рахується частка
    його розворотних вікон у кожному підкласі й скільки переломів лежать у
    підкласі цілком.
    """
    from sklearn.decomposition import PCA
    from sklearn.cluster import KMeans
    from sklearn.metrics import silhouette_score
    p = os.path.join(OUT_DIR, f"{coin}_{tag}.npz")
    if not os.path.exists(p):
        raise SystemExit(f"нема {os.path.basename(p)}")
    z = np.load(p, allow_pickle=True)
    sel = np.asarray(z[by]) == cls
    idx = np.asarray(z["idx"])[sel]
    rg, rd = np.asarray(z["rev_geo"])[sel], np.asarray(z["rev_det"])[sel]
    per = np.asarray(z["per"]).astype(str)[sel]
    ts = np.asarray(z["ts"]).astype(np.int64)[sel]
    M = load_marks(coin, with_det=False)
    F, RAW = build_F(M["d"], idx)
    del M
    n = len(idx)
    log(f"{coin}: {tag} клас {cls} → k-середні K={K} · {n} вікон · розворотних "
        f"{int(rg.sum())} ({rg.mean()*100:.2f}%)")
    Z = PCA(n_components=npca, random_state=seed,
            svd_solver="randomized").fit_transform(F).astype(np.float32)
    rng = np.random.default_rng(seed)
    Fn = F.copy()
    for c in range(Fn.shape[1]):
        rng.shuffle(Fn[:, c])
    Zn = PCA(n_components=npca, random_state=seed,
             svd_solver="randomized").fit_transform(Fn).astype(np.float32)
    del Fn
    lab = KMeans(n_clusters=K, n_init=10, random_state=seed).fit(Z).labels_
    labn = KMeans(n_clusters=K, n_init=10, random_state=seed).fit(Zn).labels_
    sub = rng.choice(n, min(SIL_N, n), replace=False)
    sil = float(silhouette_score(Z[sub], lab[sub]))
    siln = float(silhouette_score(Zn[sub], labn[sub]))
    log(f"   силует {sil:.4f} при нулі {siln:.4f} → відрив {sil-siln:+.4f}")
    cls_rows = cls_table(lab, RAW, rg, rd, per, with_noise=False)
    for r in cls_rows:
        r["name"] = f"{cls}.{r['cls'] + 1}"
    _log_table(cls_rows, f"підкласи {cls}.1 … {cls}.{K}:")

    jj, dist, pt, pk, t0 = _pivot_of(coin, ts)
    w = np.flatnonzero(rg)
    ev = jj[w]
    piv = sorted(set(ev.tolist()))
    events = []
    for pv in piv:
        mm = w[ev == pv]
        cnt = {}
        for c in lab[mm]:
            cnt[int(c)] = cnt.get(int(c), 0) + 1
        tot = len(mm)
        events.append({"t": int(pt[pv] + t0), "kind": int(pk[pv]), "n": tot,
                       "cnt": {str(k2): v for k2, v in sorted(cnt.items())},
                       "frac": {str(k2): round(v / tot, 4) for k2, v in sorted(cnt.items())},
                       "dom": int(max(cnt, key=lambda x: cnt[x]))})
    for r in cls_rows:
        fr = np.array([e["frac"].get(str(r["cls"]), 0.0) for e in events])
        r["piv_n"] = int((fr > 0).sum())
        r["piv_full"] = int((fr >= 0.999).sum())
        r["piv_frac_mean"] = round(float(fr.mean()), 4)
        log(f"   підклас {r['name']}: переломів торкається {r['piv_n']} з {len(piv)}, "
            f"цілком у ньому {r['piv_full']}, середня частка перелому {r['piv_frac_mean']}")

    out = {"coin": coin, "parent": tag, "parent_cls": int(cls), "by": by,
           "built": int(time.time()),
           "cfg": {"pca": npca, "k": K, "seed": seed},
           "n": {"win": n, "rev_geo": int(rg.sum()), "pivots": len(piv)},
           "base": {"geo": round(float(rg.mean()), 5),
                    "det": round(float(rd.mean()), 5)},
           "silhouette": {"real": round(sil, 4), "null": round(siln, 4),
                          "gap": round(sil - siln, 4)},
           "classes": cls_rows, "events": events, "feats": list(FEATS8)}
    q = os.path.join(OUT_DIR, f"{coin}_{tag}_c{cls}.json")
    json.dump(out, open(q, "w"), ensure_ascii=False)
    np.savez_compressed(os.path.join(OUT_DIR, f"{coin}_{tag}_c{cls}.npz"),
                        klab=lab, hlab=lab, idx=idx, ts=ts, rev_geo=rg,
                        rev_det=rd, per=per)
    log(f"{coin}: збережено {os.path.basename(q)} + npz")
    return out


def stage_pqwarm(coin="BTC", tag="all_nz", cls=1, sub=0, bs=2048):
    """ПРОДАКШЕН-МОДЕЛЬ ТИХОГО ПСА, ПРОГНАНА ПО ПРОГРІВУ (перші 28 діб).

    Мітка тихого ПСА причинна: вона потребує норми за 28 діб, тож перших 9873
    вікон датасету не має ВЗАГАЛІ. Але МОДЕЛІ норма не потрібна — на вхід їй
    ідуть самі 1680 чисел вікна, тож її можна опитати і там, де мітки нема.
    Це і робиться тут: ваги беруться з models08 як є (нічого не навчається),
    поріг — той самий, що записаний у картці моделі.

    ЧЕСНО ПРО ЦІ ЧИСЛА: на прогріві звіряти нема з чим — мітки там не існує,
    тому спрацювання моделі там НЕ є ні влучанням, ні помилкою. Єдине, що
    можна сказати, — з якою частотою вона горить і чи збігається це з тим, як
    вона горить на розміченій частині.
    """
    import torch
    import class4 as C4
    import mywclass as MW
    card = json.load(open(os.path.join(C4.MODELS08, f"{coin}_pcaquiet.json")))
    cfg, thr = card["cfg"], float(card["thr"])
    d = np.load(os.path.join(T.CACHE08, f"{coin}_data.npz"), mmap_mode="r")
    n = int(d["X"].shape[0])
    P = np.load(os.path.join(T.CACHE08, f"{coin}_poles.npz"))
    pm = json.load(open(os.path.join(T.CACHE08, f"{coin}_poles.json")))
    q = np.zeros(n, bool); hq = np.zeros(n, bool)
    q[P["wid"]] = P["lab"][P["wid"]] == pm["lo_id"]; hq[P["wid"]] = True
    dev = C4._dev()
    net = C4._dil(ch=cfg["ch"], dils=tuple(cfg["dils"]), k=cfg["k"], cin=8).to(dev)
    net.load_state_dict(torch.load(os.path.join(C4.MODELS08, f"{coin}_pcaquiet.pt"),
                                   map_location=dev))
    net.eval()
    log(f"{coin}: модель тихого ПСА (ch{cfg['ch']} k{cfg['k']} d{cfg['dils']}, "
        f"поріг {thr:.5f}) → прогін по ВСІХ {n} вікнах, з них прогріву "
        f"{int((~hq).sum())}")
    p = np.empty(n, np.float32)
    t0 = time.time()
    with torch.no_grad():
        for a in range(0, n, bs):
            L = MW.prep_fixed(np.asarray(d["X"][a:a + bs]))
            p[a:a + bs] = torch.sigmoid(
                net(torch.from_numpy(L).to(dev))).cpu().numpy().ravel()
    fire = p >= thr
    log(f"   {int(time.time()-t0)} с · горить усього {int(fire.sum())} "
        f"({fire.mean()*100:.2f}%)")
    lab_part = {"n": int(hq.sum()), "fire": int(fire[hq].sum()),
                "rate": round(float(fire[hq].mean()), 4),
                "label": int(q[hq].sum()), "label_rate": round(float(q[hq].mean()), 4),
                "hit": int((fire & q)[hq].sum())}
    lab_part["prec"] = round(lab_part["hit"] / max(lab_part["fire"], 1), 4)
    lab_part["rec"] = round(lab_part["hit"] / max(lab_part["label"], 1), 4)
    warm = {"n": int((~hq).sum()), "fire": int(fire[~hq].sum()),
            "rate": round(float(fire[~hq].mean()), 4)}
    log(f"   РОЗМІЧЕНА частина: {lab_part['fire']} спрацювань на {lab_part['n']} "
        f"({lab_part['rate']*100:.2f}%), мітка {lab_part['label']} "
        f"({lab_part['label_rate']*100:.2f}%), точність {lab_part['prec']*100:.1f}%, "
        f"повнота {lab_part['rec']*100:.1f}%")
    log(f"   ПРОГРІВ (мітки нема): {warm['fire']} спрацювань на {warm['n']} "
        f"({warm['rate']*100:.2f}%)")

    zc = np.load(os.path.join(OUT_DIR, f"{coin}_{tag}_c{cls}.npz"), allow_pickle=True)
    a11 = np.zeros(n, bool)
    a11[np.asarray(zc["idx"])[np.asarray(zc["klab"]) == sub]] = True
    ov = {"n11": int(a11.sum()),
          "warm_fire_in_11": int((fire & ~hq & a11).sum()),
          "warm_11": int((~hq & a11).sum()),
          "all_fire_in_11": int((fire & a11).sum())}
    log(f"   на прогріві 1.1 накриває {ov['warm_11']} вікон, з них модель горить "
        f"на {ov['warm_fire_in_11']} ({100*ov['warm_fire_in_11']/max(ov['warm_11'],1):.1f}%)")

    out = {"coin": coin, "built": int(time.time()), "thr": thr, "cfg": cfg,
           "n": {"win": n, "labeled": int(hq.sum()), "warm": int((~hq).sum())},
           "labeled": lab_part, "warm": warm, "overlap11": ov,
           "fire_total": int(fire.sum())}
    json.dump(out, open(os.path.join(OUT_DIR, f"{coin}_pqwarm.json"), "w"),
              ensure_ascii=False)
    np.savez_compressed(os.path.join(OUT_DIR, f"{coin}_pqwarm.npz"),
                        p=p, fire=fire, have=hq, label=q)
    log(f"{coin}: збережено {coin}_pqwarm.json + npz")

    # ---- карти: та сама UMAP датасету, тепер з відповідями моделі ----
    zz = np.load(os.path.join(OUT_DIR, f"{coin}_all.npz"), allow_pickle=True)
    U2 = zz["u2"]
    plt = T._style()
    fig, ax = plt.subplots(1, 4, figsize=(18, 4.7))
    A = ax[0]
    for nm, m, c, sz, al in (("не горить", ~fire, "#3d444d", 1.0, .15),
                             ("модель горить (розмічена частина)", fire & hq, "#3fb950", 1.4, .40),
                             ("модель горить НА ПРОГРІВІ", fire & ~hq, "#d29922", 2.0, .55)):
        if m.any():
            A.scatter(U2[m, 0], U2[m, 1], s=sz, alpha=al, c=c, linewidths=0,
                      label=f"{nm} ({int(m.sum())})")
    A.set_title("відповіді моделі на ВСЬОМУ датасеті", fontsize=9.5)
    A.legend(fontsize=6.5, markerscale=5)
    B = ax[1]
    for nm, m, c in (("поза обома", hq & ~fire & ~q, "#3d444d"),
                     ("збіг: мітка і сигнал", hq & fire & q, "#3fb950"),
                     ("мітка без сигналу", hq & ~fire & q, "#58a6ff"),
                     ("сигнал поза міткою", hq & fire & ~q, "#f85149")):
        if m.any():
            B.scatter(U2[m, 0], U2[m, 1], s=(1.0 if "поза обома" == nm else 1.6),
                      alpha=(.15 if "поза обома" == nm else .45), c=c, linewidths=0,
                      label=f"{nm} ({int(m.sum())})")
    B.set_title("розмічена частина: модель проти мітки", fontsize=9.5)
    B.legend(fontsize=6.5, markerscale=4)
    C = ax[2]
    C.scatter(U2[:, 0], U2[:, 1], s=1.0, alpha=.10, c="#3d444d", linewidths=0)
    C.scatter(U2[~hq, 0], U2[~hq, 1], s=1.3, alpha=.30, c="#6e7681", linewidths=0,
              label=f"прогрів, мітки нема ({int((~hq).sum())})")
    C.scatter(U2[fire & ~hq, 0], U2[fire & ~hq, 1], s=2.2, alpha=.6, c="#d29922",
              linewidths=0, label=f"з них модель горить ({int((fire & ~hq).sum())})")
    C.set_title("прогрів 28 діб окремо", fontsize=9.5)
    C.legend(fontsize=6.5, markerscale=4)
    D = ax[3]
    D.scatter(U2[:, 0], U2[:, 1], s=1.0, alpha=.10, c="#3d444d", linewidths=0)
    D.scatter(U2[a11, 0], U2[a11, 1], s=1.3, alpha=.28, c="#58a6ff", linewidths=0,
              label=f"шум 1.1 ({int(a11.sum())})")
    D.scatter(U2[fire, 0], U2[fire, 1], s=1.6, alpha=.35, c="#3fb950", linewidths=0,
              label=f"модель ПСА горить ({int(fire.sum())})")
    D.scatter(U2[fire & a11, 0], U2[fire & a11, 1], s=1.8, alpha=.45, c="#f0883e",
              linewidths=0, label=f"обидва ({int((fire & a11).sum())})")
    D.set_title("модель ПСА проти класу 1.1", fontsize=9.5)
    D.legend(fontsize=6.5, markerscale=4)
    fig.suptitle(f"{coin}: продакшен-модель тихого ПСА на всьому датасеті — на прогріві "
                 f"горить {warm['rate']*100:.2f}% проти {lab_part['rate']*100:.2f}% "
                 f"на розміченій частині", fontsize=11.5)
    fig.tight_layout(rect=(0, 0, 1, 0.93))
    fig.savefig(os.path.join(OUT_DIR, f"{coin}_pqwarm.png"), dpi=112)
    plt.close(fig)
    log(f"{coin}: намальовано {coin}_pqwarm.png")
    return out


def _revset_mask(coin="BTC", tag="all_nz", cls=1, sub=0):
    """Маска «датасету з розворотами» + пропорційний mcs (спільна для rev і rev6)."""
    M = load_marks(coin, with_det=False)
    n = M["n"]
    del M
    zc = np.load(os.path.join(OUT_DIR, f"{coin}_{tag}_c{cls}.npz"), allow_pickle=True)
    a11 = np.zeros(n, bool)
    a11[np.asarray(zc["idx"])[np.asarray(zc["klab"]) == sub]] = True
    pw = np.load(os.path.join(OUT_DIR, f"{coin}_pqwarm.npz"))
    fire = np.asarray(pw["fire"]).astype(bool)
    drop = a11 | fire
    keep = ~drop
    log(f"{coin}: «датасет з розворотами» = {n} − шум {cls}.{sub+1} ({int(a11.sum())}) "
        f"− всі події210 тихеПСА ({int(fire.sum())}); перетин {int((a11 & fire).sum())}, "
        f"викинуто {int(drop.sum())} → лишилось {int(keep.sum())}")
    return keep, max(50, int(round(250 * keep.sum() / n)))


def stage_revset(coin="BTC", tag="all_nz", cls=1, sub=0, mcs=None, seed=SEED):
    """«ДАТАСЕТ З РОЗВОРОТАМИ» — усе, що лишається після викидання тихого.

    Визначення користувача (14.08): усі події-210 датасету МІНУС клас «шум 1.1»
    МІНУС «всі події210 тихеПСА» (спрацювання продакшен-моделі pcaquiet по
    всьому датасету, включно з прогрівом, де мітки не існує). Дві множини
    перетинаються, тож викидається їх ОБʼЄДНАННЯ, а не сума.

    Далі — той самий конвеєр лінії без змін: PCA-32 → k-середні K=2..10 за
    відривом силуету над жорстким нулем і PCA-32 → UMAP-10 → HDBSCAN зі
    стійкістю на іншому сіді та стелею max_share; масштаб фіч рахується заново
    на цій множині. min_cluster_size зменшено пропорційно її розміру.
    """
    keep, mcs0 = _revset_mask(coin, tag, cls, sub)
    return _stage(coin, "rev", "«датасет з розворотами» (усе, крім шуму 1.1 і подій тихого ПСА)",
                  mask=keep, mcs=(mcs or mcs0), seed=seed)


def stage_rev6(coin="BTC", tag="all_nz", cls=1, sub=0, mcs=None, seed=SEED):
    """ПОДІЯ 210×6: та сама очищена множина, але БЕЗ обох const-стін на вході.

    Вимога користувача 14.08: стандарт події — 6 фіч, `const_resist` і
    `const_support` виключені. Решта конвеєра не змінюється жодним рядком
    (PCA-32 → k-середні K=2..10 за відривом силуету над жорстким нулем і
    PCA-32 → UMAP-10 → HDBSCAN зі стійкістю на іншому сіді), тож різниця з
    прогоном на 8 фічах є РІВНО внеском стін.

    Масштаб рахується заново на шести каналах — інакше медіана й IQR тягнули б
    у себе розкид викинутих. РІВНІ в таблицях лишаються на всі 8 фіч: треба
    бачити, що роблять стіни в класах, знайдених без них.
    """
    keep, mcs0 = _revset_mask(coin, tag, cls, sub)
    out = _stage(coin, "rev6",
                 "«датасет з розворотами», подія 210×6 (без const-стін)",
                 mask=keep, mcs=(mcs or mcs0), seed=seed, cols=[1, 2, 4, 5, 6, 7])
    try:
        out = _cmp8(coin)
    except Exception as e:
        log(f"{coin}: звірка з прогоном на 8 фічах не вийшла ({type(e).__name__}: {e})")
    return out


def _auto_drop(coin, tag, key="hdb_classes"):
    """Які класи прогону є «тихими»: усі, крім найрозворотнішого (шум не чіпаємо).

    Правило механічне і тому переноситься на іншу монету: номери класів у
    HDBSCAN довільні, змістовним є ЛИШЕ порядок за ліфтом розворотів. Шум
    ніколи не викидається — у ньому щоразу лишається третина всіх розворотних
    вікон, і це не «невпізнані» вікна, а тихі краї гронів.
    """
    d = json.load(open(os.path.join(OUT_DIR, f"{coin}_{tag}.json")))
    rows = [r for r in d[key] if r["cls"] >= 0]
    if len(rows) < 2:
        return ()
    keep = max(rows, key=lambda r: r["lift"])["cls"]
    out = tuple(sorted(r["cls"] for r in rows if r["cls"] != keep))
    log(f"{coin}/{tag}: найрозворотніший клас {keep} (×{max(r['lift'] for r in rows)}), "
        f"викидаються {list(out)}")
    return out


def _drop_candidates(coin, tag, key="classes"):
    """Підказка людині: класи, бідні на розвороти (ліфт < 1), з їхніми числами.

    НЕ автоматичне рішення. У ланцюгу очистки два кроки (rev4b і rev4c) робились
    рішенням користувача, а не правилом: дрібні класи в 37-200 вікон статистично
    від бази не відрізняються (q 0.18-0.59), і викидати їх чи ні — вибір, а не
    висновок. Функція лише показує таблицю кандидатів.
    """
    d = json.load(open(os.path.join(OUT_DIR, f"{coin}_{tag}.json")))
    rows = [r for r in d[key] if r["cls"] >= 0]
    log(f"{coin}/{tag}: кандидати на викидання (ліфт < 1):")
    log(f"   {'клас':>6s} {'вікон':>7s} {'розв.':>6s} {'ліфт':>6s} {'q':>8s}")
    out = []
    for r in sorted(rows, key=lambda r: r["lift"]):
        mark = "  ←" if r["lift"] < 1 else ""
        log(f"   {r['cls']:6d} {r['n']:7d} {r['n_rev']:6d} {r['lift']:6.2f} "
            f"{r['q']:8.4f}{mark}")
        if r["lift"] < 1:
            out.append(r["cls"])
    return tuple(out)


def stage_rev4(coin="BTC", tag="all_nz", cls=1, sub=0, drop=None, mcs=None,
               seed=SEED):
    """ПОДІЯ 210×4: тільки приплив і відтік обох стін, обʼєми теж викинуті.

    Дві дії за раз (вимога користувача 14.08):
    (1) з очищеної множини викидаються ДВА ТИХІ КЛАСИ прогону 210×6 (класи 0 і 1
        HDBSCAN: 238 і 2332 вікна, разом 1 і 7 розворотів) — вони й так були
        порожні на розвороти, тож база розворотів піднімається;
    (2) на вхід ідуть ЧОТИРИ фічі: resist_plus · resist_minus · support_plus ·
        support_minus. Викинуто vol_buy, vol_sell і обидві const-стіни.

    Множина визначається ЧЕРЕЗ МІТКИ 210×6, тож етап rev6 має бути прогнаний
    раніше; шум HDBSCAN у викидання НЕ входить (у ньому 409 розворотних вікон).
    """
    keep, _ = _revset_mask(coin, tag, cls, sub)
    if drop is None:
        drop = _auto_drop(coin, "rev6")
    a = np.load(os.path.join(OUT_DIR, f"{coin}_rev6.npz"), allow_pickle=True)
    idx6, h6 = np.asarray(a["idx"]), np.asarray(a["hlab"])
    bad = np.zeros(len(keep), bool)
    bad[idx6[np.isin(h6, list(drop))]] = True
    keep2 = keep & ~bad
    log(f"{coin}: очищений датасет {int(keep.sum())} − тихі класи 210×6 "
        f"{list(drop)} ({int(bad.sum())} вікон) → {int(keep2.sum())}")
    if mcs is None:
        mcs = max(50, int(round(250 * keep2.sum() / len(keep))))
    out = _stage(coin, "rev4",
                 "очищений датасет без двох тихих класів, подія 210×4 "
                 "(лише приплив і відтік)",
                 mask=keep2, mcs=mcs, seed=seed, cols=[1, 2, 4, 5])
    try:
        out = _cmp_prev(coin, "rev4", "rev6")
    except Exception as e:
        log(f"{coin}: звірка з прогоном 210×6 не вийшла ({type(e).__name__}: {e})")
    return out


def stage_mcs(coin="BTC", tag="rev4", target=10,
              mcss=(155, 110, 80, 60, 45, 35, 25, 18, 12, 8), seed=SEED,
              npca=32, nn=30):
    """HDBSCAN «на K класів»: у нього НЕМА K, є min_cluster_size.

    Питання користувача «зроби HDBSCAN K=10» не має прямої відповіді: кількість
    класів у HDBSCAN не задається, вона ВИХОДИТЬ з щільності. Найближче чесне —
    перебрати min_cluster_size і взяти те, що дає приблизно стільки класів.

    Перебір іде на ОДНОМУ РАЗ ПОБУДОВАНОМУ просторі (PCA-32 → UMAP-10), інакше
    девʼять повних перебудов коштували б години; тому числа перебору — лише
    орієнтир. Обране mcs далі проганяється ПОВНИМ конвеєром із жорстким нулем і
    стійкістю на іншому сіді (повна перебудова), і саме ці числа є результатом:
    у /wclass стійкість при фіксованому просторі була завищена вдвічі.
    """
    from sklearn.decomposition import PCA
    from sklearn.cluster import HDBSCAN
    import umap
    p = os.path.join(OUT_DIR, f"{coin}_{tag}.npz")
    if not os.path.exists(p):
        raise SystemExit(f"нема {os.path.basename(p)}")
    z = np.load(p, allow_pickle=True)
    idx, rg, rd = z["idx"], z["rev_geo"], z["rev_det"]
    per = z["per"].astype(str)
    cfg = json.load(open(os.path.join(OUT_DIR, f"{coin}_{tag}.json"))).get("cfg", {})
    cols = cfg.get("cols") or list(range(8))
    M = load_marks(coin, with_det=False)
    F, RAW = build_F(M["d"], idx, cols=cols)
    del M
    n, base = len(idx), float(rg.mean())
    log(f"{coin}/{tag}: перебір min_cluster_size під ~{target} класів · {n} вікон · "
        f"розворотних {int(rg.sum())} ({base*100:.2f}%) · вхід {len(cols)} фіч")
    t0 = time.time()
    Z = PCA(n_components=npca, random_state=seed,
            svd_solver="randomized").fit_transform(F).astype(np.float32)
    U = umap.UMAP(n_components=10, n_neighbors=nn, min_dist=0.0,
                  random_state=seed, verbose=False).fit_transform(Z)
    log(f"   простір побудовано за {int(time.time()-t0)} с — далі лише HDBSCAN")
    sweep, labs = [], {}
    for m in mcss:
        lab = HDBSCAN(min_cluster_size=int(m)).fit(U).labels_
        labs[m] = lab
        k = int(lab.max()) + 1
        sweep.append({"mcs": int(m), "k": k,
                      "noise": round(float((lab < 0).mean()), 4),
                      "max_share": SR._maxshare(lab)})
        log(f"   mcs={m:4d}: класів {k:3d} · шум {sweep[-1]['noise']*100:5.1f}% · "
            f"max_share {sweep[-1]['max_share']}")
    pick = min(sweep, key=lambda r: (abs(r["k"] - target), -r["mcs"]))["mcs"]
    log(f"   обрано mcs={pick} (найближче до {target} класів) — далі ПОВНИЙ прогін "
        f"з нулем і стійкістю")
    H = SR._hdb_block(F, seed, pick, npca=npca, nn=nn, want_u2=True)
    hlab, U2 = H["lab"], H["u2"]
    log(f"   ПОВНИЙ: класів {H['real']['k']} · шум {H['real']['noise']*100:.1f}% · "
        f"стійкість {H['real']['ari_seed']} проти {H['null']['ari_seed']} у нуля · "
        f"max_share {H['real']['max_share']} (нуль: k={H['null']['k']}, "
        f"шум {H['null']['noise']*100:.1f}%)")
    hcls = cls_table(hlab, RAW, rg, rd, per)
    _log_table(hcls, f"HDBSCAN mcs={pick}:")
    out = {"coin": coin, "stage": f"{tag}_mcs", "built": int(time.time()),
           "cfg": {"pca": npca, "umap": 10, "n_neighbors": nn, "seed": seed,
                   "target_k": int(target), "mcs_list": [int(m) for m in mcss],
                   "cols": list(cols), "feats_in": [FEATS8[c] for c in cols],
                   "fixed_space": True},
           "n": {"win": n, "rev_geo": int(rg.sum()), "rev_det": int(rd.sum())},
           "base": {"geo": round(base, 5), "det": round(float(rd.mean()), 5)},
           "sweep": sweep, "mcs": int(pick),
           "hdb": {"real": H["real"], "null": H["null"]},
           "classes": hcls, "feats": list(FEATS8)}
    json.dump(out, open(os.path.join(OUT_DIR, f"{coin}_{tag}_mcs.json"), "w"),
              ensure_ascii=False)
    np.savez_compressed(os.path.join(OUT_DIR, f"{coin}_{tag}_mcs.npz"),
                        hlab=hlab, u2=U2, idx=idx,
                        **{f"m{m}": labs[m] for m in mcss})
    log(f"{coin}: збережено {coin}_{tag}_mcs.json (числа ДО малювання)")
    try:
        plt = T._style()
        fig, ax = plt.subplots(1, 3, figsize=(16.5, 4.6),
                               gridspec_kw={"width_ratios": [1, 1.3, 1]})
        mm = [r["mcs"] for r in sweep]
        ax[0].plot(mm, [r["k"] for r in sweep], marker="o", ms=4, color="#58a6ff",
                   label="класів")
        ax[0].axhline(target, color=GRID, ls="--", lw=1)
        a2 = ax[0].twinx()
        a2.plot(mm, [100 * r["noise"] for r in sweep], marker="o", ms=4,
                color="#f0883e", label="шум, %")
        a2.set_ylabel("шум, %", fontsize=8, color="#f0883e")
        ax[0].set_xscale("log"); ax[0].set_xlabel("min_cluster_size", fontsize=8)
        ax[0].set_title(f"скільки класів дає mcs (простір фіксований)\nобрано {pick}",
                        fontsize=9)
        ax[0].axvline(pick, color="#3fb950", lw=1, ls=":")
        for c in range(-1, int(hlab.max()) + 1):
            m = hlab == c
            if m.any():
                ax[1].scatter(U2[m, 0], U2[m, 1], s=1.1, alpha=.30, linewidths=0,
                              c=("#3d444d" if c < 0 else None),
                              label=("шум" if c < 0 else f"{c} ({int(m.sum())})"))
        ax[1].scatter(U2[rg, 0], U2[rg, 1], s=15, facecolors="none",
                      edgecolors="#f0883e", linewidths=0.5, alpha=.9)
        ax[1].set_title(f"HDBSCAN mcs={pick}: {H['real']['k']} класів, "
                        f"шум {H['real']['noise']*100:.0f}%; обведено "
                        f"{int(rg.sum())} розворотних", fontsize=9)
        ax[1].legend(fontsize=5.5, markerscale=4, ncol=2)
        ax[1].set_xticks([]); ax[1].set_yticks([])
        nm = [("шум" if c["cls"] < 0 else str(c["cls"])) for c in hcls]
        ax[2].bar(nm, [c["lift"] for c in hcls], color="#f0883e")
        ax[2].axhline(1, color=GRID, lw=1, ls="--")
        ax[2].set_title(f"розвороти в класі: ліфт до бази {base*100:.2f}%", fontsize=9)
        ax[2].tick_params(axis="x", labelsize=6.5)
        for a in (ax[0], ax[2]):
            a.grid(alpha=.15, color=GRID)
        fig.suptitle(f"{coin}: {tag} — HDBSCAN не має K; найближче до {target} класів "
                     f"дає min_cluster_size={pick}", fontsize=11.5)
        fig.tight_layout(rect=(0, 0, 1, 0.92))
        fig.savefig(os.path.join(OUT_DIR, f"{coin}_{tag}_mcs.png"), dpi=110)
        plt.close(fig)
        log(f"{coin}: намальовано {coin}_{tag}_mcs.png")
    except Exception as e:
        log(f"{coin}: ГРАФІК НЕ ВИЙШОВ ({type(e).__name__}: {e}) — числа в JSON цілі")
    return out


def stage_rev4b(coin="BTC", drop=(1, 2), mcs=None, seed=SEED):
    """ЩЕ ОДНЕ ЗВУЖЕННЯ: з множини 210×4 викидаються два тихі класи прогону mcs.

    Викидаються класи 1 і 2 повного прогону HDBSCAN mcs=25 (119 і 2094 вікна,
    разом 7 розворотних) — обидва майже порожні на розвороти. Шум знову НЕ
    чіпаємо: у ньому 241 розворотне вікно.

    Вхід лишається тим самим — 4 фічі (приплив і відтік обох стін), решта
    конвеєра без змін, масштаб перераховується на новій множині.
    """
    z4 = np.load(os.path.join(OUT_DIR, f"{coin}_rev4.npz"), allow_pickle=True)
    zm = np.load(os.path.join(OUT_DIR, f"{coin}_rev4_mcs.npz"), allow_pickle=True)
    idx4 = np.asarray(z4["idx"])
    if not np.array_equal(idx4, np.asarray(zm["idx"])):
        raise ValueError("mcs-прогін ішов по іншій множині — мітки не прикладаються")
    M = load_marks(coin, with_det=False)
    n = M["n"]
    del M
    keep = np.zeros(n, bool)
    keep[idx4] = True
    bad = np.zeros(n, bool)
    bad[idx4[np.isin(np.asarray(zm["hlab"]), list(drop))]] = True
    keep2 = keep & ~bad
    log(f"{coin}: множина 210×4 {int(keep.sum())} − класи mcs {list(drop)} "
        f"({int(bad.sum())} вікон) → {int(keep2.sum())}")
    if mcs is None:
        mcs = max(50, int(round(250 * keep2.sum() / n)))
    out = _stage(coin, "rev4b",
                 "210×4 без двох тихих класів прогону mcs=25",
                 mask=keep2, mcs=mcs, seed=seed, cols=[1, 2, 4, 5])
    try:
        out = _cmp_prev(coin, "rev4b", "rev4")
    except Exception as e:
        log(f"{coin}: звірка з rev4 не вийшла ({type(e).__name__}: {e})")
    return out


def stage_rev4c(coin="BTC", cls=0, keep_pivot="2026-03-14 10:34", mcs=None, seed=SEED):
    """ЧЕТВЕРТЕ ЗВУЖЕННЯ: клас 0 прогону rev4b знято ЦІЛКОМ, крім одного вікна.

    Рішення користувача: клас 0 (1525 вікон) не має власних розворотів — його
    10 розворотних вікон розкидані по 8 переломах і скрізь є краєм чужого грона.
    Лишається рівно одне вікно — розворотне вікно перелому 14.03 10:34 (дно), де
    клас тримав половину перелому з двох вікон.

    Вікно шукається за ЧАСОМ ПИВОТА, а не за порядковим номером: номери класів
    і рядків залежать від сіда, а час пивота — ні.
    """
    z = np.load(os.path.join(OUT_DIR, f"{coin}_rev4b.npz"), allow_pickle=True)
    idx, hlab = np.asarray(z["idx"]), np.asarray(z["hlab"])
    rg, ts = np.asarray(z["rev_geo"]).astype(bool), np.asarray(z["ts"])
    jj, dist, pt, pk, t0 = _pivot_of(coin, ts)
    want = int(dtm.datetime.strptime(keep_pivot, "%Y-%m-%d %H:%M")
               .replace(tzinfo=dtm.timezone.utc).timestamp())
    ptt = pt[jj] + t0
    keep_row = np.flatnonzero((hlab == cls) & rg & (np.abs(ptt - want) <= 60))
    if len(keep_row) == 0:
        raise SystemExit(f"у класі {cls} нема розворотного вікна пивота {keep_pivot}")
    log(f"{coin}: у класі {cls} {int((hlab==cls).sum())} вікон, лишаємо "
        f"{len(keep_row)} (розворот {keep_pivot})")
    M = load_marks(coin, with_det=False)
    n = M["n"]
    del M
    keep = np.zeros(n, bool)
    keep[idx] = True
    bad = np.zeros(n, bool)
    bad[idx[hlab == cls]] = True
    bad[idx[keep_row]] = False
    keep2 = keep & ~bad
    log(f"{coin}: {int(keep.sum())} − клас {cls} без збереженого вікна "
        f"({int(bad.sum())}) → {int(keep2.sum())}")
    if mcs is None:
        mcs = max(50, int(round(250 * keep2.sum() / n)))
    out = _stage(coin, "rev4c",
                 f"210×4, клас {cls} знято цілком (лишено 1 розворотне вікно {keep_pivot})",
                 mask=keep2, mcs=mcs, seed=seed, cols=[1, 2, 4, 5])
    try:
        out = _cmp_prev(coin, "rev4c", "rev4b")
    except Exception as e:
        log(f"{coin}: звірка з rev4b не вийшла ({type(e).__name__}: {e})")
    return out


def _cmp_prev(coin="BTC", a="rev4", b="rev6"):
    """Звірка двох прогонів на СПІЛЬНИХ вікнах (множини різні за розміром).

    Прогін a — підмножина прогону b, тож обидві розмітки зводяться до перетину
    їхніх idx; поза перетином порівнювати нема з чим за побудовою.
    """
    from sklearn.metrics import adjusted_rand_score as ARI
    A = np.load(os.path.join(OUT_DIR, f"{coin}_{a}.npz"), allow_pickle=True)
    B = np.load(os.path.join(OUT_DIR, f"{coin}_{b}.npz"), allow_pickle=True)
    ia, ib = np.asarray(A["idx"]), np.asarray(B["idx"])
    com = np.intersect1d(ia, ib)
    sa = np.searchsorted(ia, com); sb = np.searchsorted(ib, com)
    ka, kb = np.asarray(A["klab"])[sa], np.asarray(B["klab"])[sb]
    ha, hb = np.asarray(A["hlab"])[sa], np.asarray(B["hlab"])[sb]
    rg = np.asarray(A["rev_geo"]).astype(bool)[sa]
    p = os.path.join(OUT_DIR, f"{coin}_{a}.json")
    d = json.load(open(p)); e = json.load(open(os.path.join(OUT_DIR, f"{coin}_{b}.json")))
    best = lambda rows: max([r for r in rows if r["cls"] >= 0], key=lambda r: r["lift"])
    j = lambda m1, m2: round(float((m1 & m2).sum() / max((m1 | m2).sum(), 1)), 4)
    ca, cb = ka == best(d["kmeans"])["cls"], kb == best(e["kmeans"])["cls"]
    ma, mb = ha == best(d["hdb_classes"])["cls"], hb == best(e["hdb_classes"])["cls"]
    d["cmp_prev"] = {
        "vs": b, "n_common": int(len(com)),
        "ari_km": round(float(ARI(ka, kb)), 4),
        "agree_km": round(float(max((ka == kb).mean(), (ka != kb).mean())), 4),
        "ari_hdb": round(float(ARI(ha, hb)), 4),
        "km_best": {"n6": int(ca.sum()), "n8": int(cb.sum()),
                    "inter": int((ca & cb).sum()), "jac": j(ca, cb),
                    "rev6": int(rg[ca].sum()), "rev8": int(rg[cb].sum())},
        "hdb_best": {"n6": int(ma.sum()), "n8": int(mb.sum()),
                     "inter": int((ma & mb).sum()), "jac": j(ma, mb),
                     "rev6": int(rg[ma].sum()), "rev8": int(rg[mb].sum())},
        "k8": e["k_best"], "hdb_k8": e["hdb"]["real"]["k"],
        "noise8": e["hdb"]["real"]["noise"],
        "base8": e["base"]["geo"], "n_win8": e["n"]["win"], "rev8": e["n"]["rev_geo"]}
    json.dump(d, open(p, "w"), ensure_ascii=False)
    log(f"{coin}: звірка {a} з {b} на {len(com)} спільних вікнах — ARI k-середніх "
        f"{d['cmp_prev']['ari_km']} (згода {d['cmp_prev']['agree_km']}), "
        f"ARI HDBSCAN {d['cmp_prev']['ari_hdb']}, Жаккар розворотного класу "
        f"{d['cmp_prev']['hdb_best']['jac']}")
    return d


def stage_revmap(coin="BTC", tag="up"):
    """ЧИЇ САМЕ РОЗВОРОТНІ ВІКНА ЛЕЖАТЬ У КОЖНОМУ КЛАСІ (і в шумі теж).

    Навіщо окремий етап. У таблиці класів розворотних вікон десятки, і саме
    число нічого не каже, поки не відомо, СКІЛЬКОМ РІЗНИМ ПЕРЕЛОМАМ вони
    належать: 42 вікна можуть бути 42 різними розворотами, а можуть — пʼятьма
    гронами навколо одного пивота (вікна йдуть кроком 210 с, у смугу ±15 хв їх
    влазить до 9). Це та сама пастка гронування, що в /trade08, тільки збоку.

    Тому кожне розворотне вікно приписується СВОЄМУ ПИВОТУ, і питання
    ставиться на рівні перелому:
      · у скількох переломів узагалі є вікно в цьому класі;
      · для скількох саме цей клас тримає НАЙБІЛЬШУ частку вікон перелому;
      · скільки переломів лежать у класі ЦІЛКОМ — тобто унікальні для нього.
    Останнє і є відповідь на «чи ці вікна унікальні»: якщо перелом цілком у
    шумі, HDBSCAN його ніде більше не бачив; якщо ні — той самий перелом уже
    порахований в іншому класі, і вікна в шумі є лише його краєм.
    """
    p = os.path.join(OUT_DIR, f"{coin}_{tag}.npz")
    if not os.path.exists(p):
        raise SystemExit(f"нема {os.path.basename(p)} — спершу "
                         f"python3 zzclust.py {tag} {coin}")
    z = np.load(p, allow_pickle=True)
    hlab, klab, ts_abs, rg = z["hlab"], z["klab"], z["ts"], z["rev_geo"]
    base = json.load(open(os.path.join(OUT_DIR, f"{coin}_{tag}.json")))
    jj, dist, pt, pk, t0 = _pivot_of(coin, ts_abs)
    w = np.flatnonzero(rg)
    ev = jj[w]
    piv = sorted(set(ev.tolist()))
    log(f"{coin}/{tag}: розворотних вікон {len(w)} · різних переломів {len(piv)} "
        f"· у середньому {len(w)/max(len(piv),1):.1f} вікна на перелом")

    def block(lab, name, with_noise):
        lo = -1 if with_noise else 0
        cls = {}
        for c in range(lo, int(lab.max()) + 1):
            cls[c] = {"cls": int(c), "n_win": 0, "n_piv": 0, "n_dom": 0, "n_excl": 0}
        evs = []
        for pv in piv:
            m = w[ev == pv]
            cnt = {}
            for c in lab[m]:
                cnt[int(c)] = cnt.get(int(c), 0) + 1
            dom = max(cnt, key=lambda k: (cnt[k], -k))
            for c, v in cnt.items():
                if c not in cls:
                    continue
                cls[c]["n_win"] += v
                cls[c]["n_piv"] += 1
                if len(cnt) == 1:
                    cls[c]["n_excl"] += 1
            if dom in cls:
                cls[dom]["n_dom"] += 1
            # ЧАСТКА ПЕРЕЛОМУ В КЛАСІ: перелом цілком в одному класі → 1.0;
            # 3% вікон у шумі і 97% в класі → 0.03 і 0.97. Саме частка, а не
            # кількість вікон: переломи різні за довжиною (від 1 до 9 вікон),
            # і сира кількість зважує довгі грона.
            tot = max(len(m), 1)
            evs.append({"piv": int(pv), "t": int(pt[pv] + t0),
                        "kind": int(pk[pv]), "n": int(len(m)),
                        "dom": int(dom), "cnt": {str(k): v for k, v in sorted(cnt.items())},
                        "frac": {str(k): round(v / tot, 4) for k, v in sorted(cnt.items())}})
        rows = list(cls.values())
        log(f"   {name}:")
        log(f"   {'клас':>6s} {'вікон':>6s} {'переломів':>10s} {'домінує':>8s} {'ЦІЛКОМ у класі':>15s}")
        for r in rows:
            nm = "шум" if r["cls"] < 0 else str(r["cls"])
            log(f"   {nm:>6s} {r['n_win']:6d} {r['n_piv']:10d} {r['n_dom']:8d} "
                f"{r['n_excl']:15d}")
        return rows, evs

    hrows, evs = block(hlab, "HDBSCAN", True)
    krows, _ = block(klab, f"k-середні K={base['k_best']}", False)

    # ЧАСТКИ ПО КЛАСАХ на рівні перелому: середня частка, скільки переломів
    # лежать у класі ЦІЛКОМ (частка 1.0) і скільки — більшістю (≥0.5).
    for r in hrows:
        fr = np.array([e["frac"].get(str(r["cls"]), 0.0) for e in evs])
        r["frac_mean"] = round(float(fr.mean()), 4)
        r["frac_full"] = int((fr >= 0.999).sum())
        r["frac_half"] = int((fr >= 0.5).sum())
    top = sorted(evs, key=lambda e: -e["frac"].get("-1", 0.0))[:10]
    nz = np.array([e["frac"].get("-1", 0.0) for e in evs])
    log(f"   ЧАСТКА ШУМУ на переломі: середня {nz.mean():.3f} · переломів з "
        f"часткою >0 {int((nz>0).sum())} · ≥0.5 {int((nz>=0.5).sum())} · "
        f"=1.0 {int((nz>=0.999).sum())}")
    log("   10 переломів з найбільшою часткою шуму:")
    log(f"   {'перелом (UTC)':>17s} {'бік':>8s} {'вікон':>6s} {'шум':>6s}  решта")
    for e in top:
        oth = " · ".join(f"клас {k}: {v:.2f}" for k, v in sorted(e["frac"].items())
                         if k != "-1")
        log(f"   {time.strftime('%Y-%m-%d %H:%M', time.gmtime(e['t'])):>17s} "
            f"{'дно' if e['kind'] < 0 else 'вершина':>8s} {e['n']:6d} "
            f"{e['frac'].get('-1', 0.0):6.2f}  {oth or '—'}")

    # РІВНІ ФІЧ окремо для розворотних і звичайних вікон УСЕРЕДИНІ класу: без
    # цього не видно, чим саме є розворотні вікна в шумі — повноцінною подією
    # чи тихим краєм грона, у якого гучна середина лежить в іншому класі.
    dd = np.load(os.path.join(T.CACHE08, f"{coin}_data.npz"), mmap_mode="r")
    gi = z["idx"]

    def lev(mask):
        ii = np.sort(gi[mask])
        if not len(ii):
            return None
        A = np.empty((len(ii), 8), np.float32)
        for a in range(0, len(ii), 1024):
            A[a:a + 1024] = np.asarray(dd["X"][ii[a:a + 1024]]).mean(axis=2)
        return {FEATS8[f]: round(float(A[:, f].mean()), 4) for f in range(8)}

    levs = []
    for c in range(-1, int(hlab.max()) + 1):
        m = hlab == c
        if not m.any():
            continue
        levs.append({"cls": int(c), "n_rev": int((m & rg).sum()),
                     "n_oth": int((m & ~rg).sum()),
                     "rev": lev(m & rg), "oth": lev(m & ~rg)})
    log("   рівні фіч усередині класу (розворотні / звичайні вікна):")
    for r in levs:
        nm = "шум" if r["cls"] < 0 else f"клас {r['cls']}"
        rp = (r["rev"] or {}).get("resist_plus")
        op = (r["oth"] or {}).get("resist_plus")
        log(f"   {nm:>8s}: resist_plus розв. {rp if rp is not None else '—'} · "
            f"звич. {op} · вікон {r['n_rev']}/{r['n_oth']}")
    out = {"coin": coin, "stage": tag, "built": int(time.time()),
           "cfg": {"zz_pct": T.LEG_THR * 100, "tol_s": T.PIVOT_TOL, "win": WIN},
           "n": {"rev_win": int(len(w)), "pivots": len(piv),
                 "pivots_all": int(len(pt)),
                 "per_pivot": round(len(w) / max(len(piv), 1), 2)},
           "hdb": hrows, "kmeans": krows, "k_best": base["k_best"],
           "levels": levs, "feats": list(FEATS8), "events": evs,
           "noise_frac": {"mean": round(float(nz.mean()), 4),
                          "gt0": int((nz > 0).sum()), "half": int((nz >= 0.5).sum()),
                          "full": int((nz >= 0.999).sum())},
           "top_noise": top}
    q = os.path.join(OUT_DIR, f"{coin}_{tag}_rev.json")
    json.dump(out, open(q, "w"), ensure_ascii=False)
    log(f"{coin}: збережено {os.path.basename(q)}")
    return out


def stage_clean(coin="BTC", force=False):
    """ВЕСЬ ЛАНЦЮГ ОЧИСТКИ ОДНІЄЮ КОМАНДОЮ — до першого людського рішення.

    Порядок кроків жорсткий: кожен наступний читає мітки попереднього. Крок,
    чий вихід уже лежить у media/analyst/zz, пропускається (force=True
    перераховує все). Повний опис — CLEANUP_DATASET.md.

    ПЕРЕДУМОВИ для іншої монети (нічого з цього тут не будується):
      · кеш датасету media/analyst/class4/cache08/<COIN>_data.npz — `class4.py data08 <COIN>`;
      · полюси ПСА <COIN>_poles.npz там же (той самий етап);
      · продакшен-модель тихого ПСА models08/<COIN>_pcaquiet.pt + .json —
        `class4.py prod08 <COIN>`.
    Без них крок pqwarm не запуститься, і «повне тихе ПСА» не буде побудоване.

    ДЕ ЛАНЦЮГ ЗУПИНЯЄТЬСЯ САМ: після кроку `mcs` рішення «які дрібні класи
    викидати» приймає людина (див. _drop_candidates) — правила для цього немає,
    класи в 37-200 вікон статистично від бази не відрізняються.
    """
    import class4 as C4
    need = [(os.path.join(T.CACHE08, f"{coin}_data.npz"), f"class4.py data08 {coin}"),
            (os.path.join(T.CACHE08, f"{coin}_poles.npz"), f"class4.py data08 {coin}"),
            (os.path.join(C4.MODELS08, f"{coin}_pcaquiet.pt"), f"class4.py prod08 {coin}")]
    miss = [(p, c) for p, c in need if not os.path.exists(p)]
    if miss:
        for p, c in miss:
            log(f"НЕМА {p} → спершу {c}")
        raise SystemExit("передумови не виконані")
    steps = [
        ("all",    f"{coin}_all.npz",      lambda: stage_all(coin)),
        ("subhdb", f"{coin}_all_nz.npz",   lambda: stage_subhdb(coin, "all")),
        ("deep",   f"{coin}_all_nz_c1.npz", lambda: stage_deep(coin, "all_nz", 1, 2)),
        ("pqwarm", f"{coin}_pqwarm.npz",   lambda: stage_pqwarm(coin)),
        ("revset", f"{coin}_rev.npz",      lambda: stage_revset(coin)),
        ("rev6",   f"{coin}_rev6.npz",     lambda: stage_rev6(coin)),
        ("rev4",   f"{coin}_rev4.npz",     lambda: stage_rev4(coin)),
        ("mcs",    f"{coin}_rev4_mcs.json", lambda: stage_mcs(coin, "rev4", 10)),
    ]
    for name, out, fn in steps:
        p = os.path.join(OUT_DIR, out)
        if os.path.exists(p) and not force:
            log(f"[{name}] вже пораховано ({out}) — пропускаю")
            continue
        log(f"[{name}] рахую…")
        t0 = time.time()
        fn()
        log(f"[{name}] готово за {int(time.time()-t0)} с")
    log(f"{coin}: механічна частина ланцюга пройдена. Далі — РІШЕННЯ ЛЮДИНИ:")
    _drop_candidates(coin, "rev4_mcs")
    log(f"   обрані класи викидаються командою `zzclust.py rev4b {coin}` "
        f"(список drop у stage_rev4b), далі за потреби `rev4c {coin}`.")
    return True
```


### `stormrev.py`

```python
def _hdb_block(F, seed, mcs, npca=32, numap=10, nn=30, want_u2=False):
    """Один повний прогін HDBSCAN із жорстким нулем і стійкістю.

    Стійкість — ПОВНА перебудова на іншому сіді (не перекластеризація в тому
    самому просторі: у /events це завищувало число вдвічі).
    Нуль — клітинки перемішано МІЖ подіями незалежно.
    """
    from sklearn.decomposition import PCA
    from sklearn.cluster import HDBSCAN
    from sklearn.metrics import adjusted_rand_score as ARI
    import umap

    def run(M, sd_, u2=False):
        p = PCA(n_components=min(npca, M.shape[1]), random_state=sd_,
                svd_solver="randomized").fit(M)
        Z = p.transform(M).astype(np.float32)
        u = umap.UMAP(n_components=numap, n_neighbors=nn, min_dist=0.0,
                      random_state=sd_, verbose=False).fit_transform(Z)
        h = HDBSCAN(min_cluster_size=int(mcs)).fit(u)
        U2 = (umap.UMAP(n_components=2, n_neighbors=nn, min_dist=0.0,
                        random_state=sd_).fit_transform(Z) if u2 else None)
        return h.labels_, U2, float(p.explained_variance_ratio_.sum())

    lab, U2, evr = run(F, seed, want_u2)
    rng = np.random.default_rng(seed)
    Fn = F.copy()
    for c in range(Fn.shape[1]):
        rng.shuffle(Fn[:, c])
    lab_n, _, evr_n = run(Fn, seed)
    lab2, _, _ = run(F, seed + 1)
    lab_n2, _, _ = run(Fn, seed + 1)
    return {
        "lab": lab, "u2": U2,
        # max_share і для РЕАЛЬНОГО розбиття, не лише для нуля: поділ «майже все
        # проти дрібки» відтворюється сам собою і дає високу стійкість ні з чого.
        # ШУМ У ЦЕ ЧИСЛО НЕ ВХОДИТЬ — він не клас, інакше при 80% шуму max_share
        # завжди виглядав би виродженим (наступив на це 13.08 у переборі mcs).
        "real": {"k": int(lab.max()) + 1, "noise": round(float((lab < 0).mean()), 4),
                 "evr": round(evr, 4), "ari_seed": round(float(ARI(lab, lab2)), 4),
                 "max_share": _maxshare(lab)},
        "null": {"k": int(lab_n.max()) + 1, "noise": round(float((lab_n < 0).mean()), 4),
                 "evr": round(evr_n, 4),
                 "ari_seed": round(float(ARI(lab_n, lab_n2)), 4),
                 "max_share": _maxshare(lab_n)}}


def _maxshare(lab):
    """Частка найбільшого КЛАСУ серед кластеризованих точок (шум не клас).
    Ознака виродженого поділу «майже все проти дрібки»; стеля лінії /events 0.65."""
    m = lab >= 0
    if not m.any():
        return 0.0
    return round(float(np.bincount(lab[m]).max() / m.sum()), 4)
```


---

*Зібрано `build_cleanup_doc.py` з робочих файлів і JSON у `media/analyst/zz`.
Монета BTC.*
