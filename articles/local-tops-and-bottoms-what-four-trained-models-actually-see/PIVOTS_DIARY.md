# Щоденник дослідження /pivots — чи видно локальні вершини і дно в ордербуці

**Що це.** Виконуваний рецепт. Читаєш цей файл, береш свій датасет — отримуєш ту саму
розмітку локальних екстремумів, ті самі перевірки і ті самі висновки. Жодного
захардкодженого списку зон: зона визначається **процедурою**, і процедура перерахує її
на будь-яких даних — іншій монеті, іншому періоді, більшій історії.

Числа в тексті — це те, що вийшло **на нашому зрізі** (BTC, 171.0 доби, дані
з 1771091656 по 1785865310). Вони наведені як орієнтир «схоже чи ні», а не як
константи. Твій прогін дасть свої.

**Ця лінія не самостійна.** Вона перевіряє, чи бачать зони моделі, навчені в лінії
`/mywclass` — незалежній репліці лінії `/wclass`. Звідти беруться чотири детектори
станів ринку. Якщо ти повторюєш з нуля, спершу пройди:

- вкладка моделей: **`/mywclass`** (репліка) і **`/wclass`** (оригінал);
- щоденник моделей: **`WCLASS_DIARY.md`** — там описано, як знайдено класи
  `storm` · `mid` · `calm` · `pcaquiet` і як навчено їхні детектори;
- стандарт щоденників: **`DIARY_STANDARD.md`**.

---

## 0. Вхід

Посекундні дані ордербука, 8 фіч + ціна:

```
const_resist  resist_plus  resist_minus     — стіна опору: товщина, приплив, відтік
const_support support_plus support_minus    — стіна підтримки: те саме
vol_buy       vol_sell                      — реальні угоди: маркет-баї і маркет-сели
price_avg                                   — ціна
timestamp                                   — секунда
```

Формат: `{DATA}/2026/{MM}/{COIN}_USDT/1S_YYYY-MM-DD_COIN_USDT.csv`, один файл на добу.
Мінімум для повторення — **≥60 діб** посекундних даних однієї монети: на меншому зрізі
екстремумів набереться сотні дві, і парні тести втратять сенс.

Додатково потрібні **відповіді чотирьох моделей** на кожне вікно 210 с — з лінії
`/mywclass` (файли `media/analyst/mywclass/cache/BTC_oof_*.npz`). Беруться **out-of-fold**
відповіді, тобто дані моделлю, яка цього вікна не бачила. Це принципово: продакшен-модель
бачила 98% вікон у навчанні, і на ній усі числа були б памʼяттю, а не оцінкою.

**Правило, яке визначає всю конструкцію:** мітка зони визначена **з майбутнього** — вершина
стає вершиною лише після падіння на 1% після неї. Тому «ідентифікація зони» тут наполовину
є прогнозом розвороту, і кожне число міряється проти двох еталонів: нуль-контролю
(циклічний зсув відповідей моделей — руйнує звʼязок, зберігає автокореляцію) і **простої
ціни з того самого вікна**. Без цих двох еталонів жодне число з цієї лінії не має сенсу.

---

## 1. Крок перший: розмітити екстремуми і зони

`python3 pivot_zones.py zones COIN`

Екстремум береться ZigZag-ом з порогом **1.0%**: нова вершина (дно)
фіксується лише коли ціна відійшла від неї на цей поріг. Наслідок — обидві ноги, до
попереднього екстремуму і до наступного, за побудовою не менші за поріг. Додатково кожен
екстремум перевіряється явно, тому перший і останній пивот (з незавершеною ногою)
відкидаються.

**Зона — це не точка.** Це найдовший неперервний проміжок навколо екстремуму, поки ціна
тримається в смузі **0.5%** від нього: для вершини — не нижче, для дна —
не вище.

Три речі, без яких далі все буде брехнею:

1. **Ціна береться з РЕАЛЬНИХ спостережень, без інтерполяції.** У нас 18% секунд відсутні;
   інтерпольована ціна дає фальшиві локальні екстремуми рівно в дірках. Зигзаг не потребує
   рівномірної сітки — він працює на нерівномірному ряді як є.
2. **Дедуплікація timestamp** перед усім іншим.
3. **Межа даних заморожена** на тій самій, що в лінії моделей, інакше сітка вікон поїде і
   поточкове зіставлення «зона проти відповіді моделі» стане неможливим.

Зони вершин і дна **не можуть перетинатись** — це арифметика, а не удача: нога ≥1%, смуга
0.5%, тож верхній край зони дна завжди нижчий за нижній край зони вершини. Але це все одно
перевіряється фактично, і в нас перетинів **0**.

На нашому зрізі: **734 зон** (367 вершин і 367 дна —
порівну за побудовою чергування), 4.29 на добу, 15.02% усього
часу. Медіанна зона 1598 с, від 0
до 59083 с.

---

## 2. Крок другий: подивитись на це очима

`python3 pivot_zones.py chart COIN`

Шість панелей по ~28 діб зі спільною вертикальною логікою плюс збільшений відрізок, де
зони видно в натуральну ширину. Зелене — зона вершини, червоне — зона дна, малюються як
прямокутники: по горизонталі тривалість зони, по вертикалі — сама смуга 0.5%.

Цей крок не декоративний. На повній історії одразу видно те, чого не видно в числах:
у сильному тренді зон майже немає (ціна не повертається в смугу), а в боковику вони
злипаються в грона. Це пояснює половину результатів далі.

---

## 3. Крок третій: чи ідентифікують зону відповіді моделей

`python3 pivot_zones.py ident COIN`

Вхід — **тільки** видача чотирьох моделей: `storm` · `mid` · `calm` · `pcaquiet`. Набір
відповідей береться від одного вікна 210 с до восьми поспіль (умова: набір має влазити в
найкоротшу зону; в нас найкоротша зона 0 с, тож строго дозволено рівно
1 набір, решта — довідково).

Оцінка — блоками по 12 год з ембарго 1 год, 5 фолдів. Без
блокового спліту сусідні вікна (кореляція ~0.76) течуть з навчання у валідацію і всі
числа завищуються.

Задач шість, і **розділяти їх обовʼязково** — саме тут ховається головний результат:

| задача | що це | наш результат | нуль | розмах ціни |
|---|---|---|---|---|
| у зоні (будь-якій) | вікно всередині смуги 0.5% | 0.545 | 0.505 | 0.523 |
| **момент екстремуму** | вікно містить сам пивот | **0.859** | 0.526 | 0.872 |
| екстремум ±1 вікно | ±3.5 хв | 0.826 | 0.504 | 0.834 |
| **екстремум у наступну годину** | прогноз | **0.748** | 0.506 | 0.715 |
| вершина проти дна | напрям | 0.527 | 0.5 | — |

**Головне з цієї таблиці.** Саму зону не видно — вона живе десятки хвилин і більшу її
частину ринок просто спить. Момент екстремуму видно різко. Але **простий розмах ціни в
тому самому вікні бʼє ордербук на всіх трьох «часових» задачах**, а разом вони додають
лише +0.005 AUC. І напрям не читається взагалі.

Не пропусти протилежну асиметрію еталонів: цінове **положення** (перцентиль у ковзних
24 год) дає 0.689 на «вершина чи дно», але лише 0.566 на
«чи це взагалі зона». Ціна каже **бік**, ордербук каже **коли**. Вони не замінюють одне одного.

---

## 4. Крок четвертий: де реально є за що зачепитись

`python3 pivot_zones.py hooks COIN`

Пʼять окремих питань замість одного:

**A. Форма сигналу всередині зони.** Зона обмежена двома ногами ≥1%, і сигнал у ній має
форму бутерброда: на вході p(шторм) високе, у середині провалюється нижче бази, на виході
знову високе. Ордербук позначає **не зону, а її краї** — удар, яким ціна прийшла, і удар,
яким вона пішла.

**B. Правило «шторм, потім k тихих вікон».** Єдина зачіпка, яку один набір відповідей дати
не може — лише послідовність. При k=8 воно горить 0.5%
часу і дає ×1.208 до бази. Контроль: сам штиль без
шторму перед ним дає ×0.997 — тобто значення має саме
послідовність. **Другий контроль обовʼязковий**: те саме правило на розмаху ціни з тими ж
частотами спрацювання дає лише ×1.02-1.29. Тут — і майже тільки тут — ордербук попереду ціни.

**C. Серед вікон, де шторм уже горить** — чи відрізнити екстремум від сплеску всередині
ноги. Ордербук 0.7349, ціна 0.7753, разом 0.7572.
Ордербук не додає.

**D. Напрям — головна пастка лінії.** 89 описів вікна дають «вершина чи дно» на AUC 0.91,
і це виглядає як прорив. Перевірка вбиває: сама ціна дає **0.994**, а ціна+ордербук — ті
самі 0.994. Описи просто перечитують напрям імпульсу з асиметрії потоків.

**E. Покриття на рівні ЗОНИ, а не вікна.** Шторм торкнувся 50% усіх екстремумів рівно в
їхньому вікні (випадково було б 10%). Але розкладка за довжиною все пояснює: терциль
коротких зон — 87%, довгих — 20% при випадкових 28%. **Сигнал ловить різку шпильку, а не
пологе плато.**

---

## 5. Крок пʼятий: два ланцюги

`python3 pivot_zones.py chains COIN`

Усі шматки, після яких ціна падала, зшиваються в один ланцюг; усі, після яких росла — в
другий. Далі порівняння по 8 фічах, по відповідях моделей і по тривалості.

**Пастка, через яку цей крок легко зіпсувати:** рівні фіч їздять по місяцях сильніше, ніж
між вершиною і дном. Тому крім звичайного порівняння скрізь рахується **парне** — кожна
вершина проти сусіднього дна (вони чергуються за побудовою зигзага), різниця береться
всередині пари. Плюс BH-поправка на множинність.

**Другий контроль ще важливіший — тіло ноги.** Для кожного шматка береться шматок тієї ж
тривалості з середини руху, який до нього привів. Він відділяє властивість розвороту від
тіні того, куди ціна щойно йшла. Результат розколює знахідку навпіл:

- **обʼєм — чиста тінь тренду.** vol_buy/vol_sell розходиться між ланцюгами сильно
  (+0.13 проти −0.16), але на вершині не змінюється відносно ноги ані на йоту (Δ+0.001,
  p=0.96). Ніякого «вигасання покупок на верхах» немає;
- **стіни і приплив — властивість саме розвороту.** Приплив на сторону опору: у тілі
  підйому +0.010 → у зоні вершини +0.079 (p<0.001); у тілі падіння +0.065 → у зоні дна
  +0.021 (p=0.003). Зсуви в **протилежні боки**. Механіка: на вершині доростає стіна
  продавців, на дні — покупців.

Тривалість: вершини живуть довше за дно (парна різниця +91 с, q=0.043).

Скільки це варте: на задачі «шматок-екстремум проти шматка з тіла ноги» асиметрія книги
× знак напряму дає 0.579, рівні 8 фіч 0.657, а **ціна (хід+розмах) 0.919**, і ціна+книга
теж 0.919.

---

## 6. Крок шостий: розподіл сигналу в чотирьох станах

`python3 pivot_zones.py storm4 COIN`

Кожне вікно відноситься **за своєю серединою** до одного з чотирьох станів: зона вершини,
зона дна, тіло ноги вгору, тіло ноги вниз.

Перше, що треба знати: розподіл p(шторм) украй **двогорбий** — медіана майже нуль у всіх
станах, маса на краях. Тому порівнювати треба частоту спрацювання, а не медіану, і шкала
часток на графіку логарифмічна.

Три висновки:

- **вершина проти дна — різниці немає** (p≈0.11 і після поправки «всередині доби»);
- **нога вниз проти ноги вгору — різниця є** і переживає поправку: падіння штормовіші за
  підйоми. Це єдине місце, де сигнал має напрямний нахил, і живе він **не в екстремумах,
  а в тілі руху**;
- **зона проти ноги — скрізь дуже значуще**, +3.3-4.5 в.п.

Профіль уздовж ноги U-подібний: шторм сидить на кінцях руху, а не всередині. На нозі вниз
розгін починається раніше й крутіше.

Читання назад — уся практична цінність сигналу: коли шторм горить, шанс «ми в зоні
екстремуму» піднімається з 15.1% до 19.4%, а серед ніг частка «вниз» — з 48.9% до 51.9%.
Три відсоткові пункти напрямного нахилу при комісії 0.30% за оборот не торгуються.

---

## 7. Крок сьомий: усередині класів — самі 1680 чисел

`python3 pivot_zones.py inside COIN all`

Досі порівнювалась **відповідь** мережі. Тут порівнюється те, що вона **бачила**: матриця
8×210 у тому самому вигляді, в якому йде на вхід (`log1p(x/0.01)`, стала призначена
наперед). Для кожної з чотирьох цілей беруться лише її вікна.

Що розрізняє кожен клас на «вершина − дно» (рівень, d Коена, BH на 8 фіч):

- **шторм** — лише стіни (`const_support` +0.25, `const_resist` +0.17);
- **клас 2** — усі шість потоків **негативні**: на дні книга активніша;
- **штиль** — `vol_sell` +0.27 і `const_resist` +0.23: у тихому вікні на вершині одночасно
  товща стіна продавців і більше продажів;
- **тихе ПСА** — `vol_sell` **+0.67**, найбільший ефект усього дослідження (але лише
  211 і 221 вікон).

**Форма всередині вікна працює тільки в шторму**: нахил `vol_buy` d=−0.23 — на вершині
покупки за 210 с гаснуть, на дні розгоряються. У тихих класах нахили порожні.

І головне число лінії: на задачі «вершина проти дна» **ціна в тому ж вікні дає 0.42-0.50
в усіх чотирьох класах — рівно нуль**, а ордербук 0.52-0.59. Це єдина задача в усій
роботі, де ордербук бʼє ціну, і причина проста: в екстремумі ціна за 210 с нікуди не йде,
їй нічого сказати.

Повні 1680 чисел при цьому виграють лише там, де вибірка велика; на 1-2 тисячах вікон вони
програють шістнадцяти числам (8 рівнів + 8 нахилів). Вхід мережі завеликий для підвибірки.

---

## Три пастки, які тут неминуче зустрінуться

**1. Підміна цілі.** Лінія моделей тренує рівно чотири цілі: `storm`, `calm`, `mid`,
`pcaquiet`. Розріз штилю на дві половинки (`calm_hi`/`calm_lo`) — це **сирі класи
k-середніх до злиття**, а не окрема модель. У першій версії цього дослідження четвертою
ціллю помилково стояла половинка штилю, а `pcaquiet` був відсутній; це виправлено 06.08,
і всі числа перераховані. Якщо повторюєш — звіряй список цілей з `wclass.final()`.

**2. Епоха історії.** Будь-яке порівняння рівнів фіч між двома групами вікон насамперед
міряє **місяць**, а не стан ринку. Ліки: парне порівняння сусідніх подій (вони близькі в
часі) або поправка «всередині доби» (ранг серед вікон тієї ж доби). Без цього ви отримаєте
дуже значущі числа, які означають «літо не схоже на весну».

**3. Ціновий еталон.** Найпоширеніша помилка цієї лінії — зрадіти AUC 0.83 і не перевірити,
що розмах ціни в тому ж вікні дає 0.87. Правило: **на кожне питання, де ордербук щось
показує, кладеться ціновий еталон з того самого вікна**, і не один, а два — «положення»
(де ціна відносно недавньої історії) і «розмах» (скільки вона щойно пройшла). Вони
відповідають на різні питання, і плутати їх не можна.

---

## Що перевіряти при повторі

| величина | наше | допуск | що означає вихід |
|---|---|---|---|
| зон на добу | 4.29 | 3-6 | інший поріг зигзага або інша волатильність монети |
| частка часу в зонах | 15.02% | 10-20% | те саме |
| перетинів зон | 0 | рівно 0 | помилка в межах зони — арифметика цього не допускає |
| мінімальна нога | 1.004% | ≥ порогу | зигзаг зламаний |
| AUC «момент екстремуму» | 0.859 | 0.78-0.87 | нижче — моделі не ті або спліт не блоковий |
| AUC «у зоні» | 0.545 | 0.50-0.56 | вище 0.6 — майже напевно витік майбутнього |
| нуль-контроль | 0.505 | 0.48-0.52 | далеко від 0.5 — спліт негерметичний |
| ціна на «моменті» | 0.872 | 0.84-0.90 | має бити ордербук; якщо ні — перевір ціновий ряд |
| напрям (вершина/дно) | 0.527 | 0.48-0.53 | вище — витік ціни в ознаки |
| шторм: покриття коротких зон | 87% | 75-92% | нижче — інші пороги детектора |

---

## Відкинуті напрямки (щоб не витрачати час повторно)

- **Ідентифікувати зону як стан.** AUC 0.52 при нулі 0.49 і восьми довжинах набору. Зона
  занадто довга і всередині тиха. Мітку треба ставити на момент, не на проміжок.
- **Напрям з ордербука.** Чотири незалежні способи (відповіді моделей, 89 описів, сирі
  1680, асиметрія книги) — усі дають 0.50-0.53 після зняття тіні ціни.
- **Довгі зони.** Терциль найдовших зон детектується **гірше за випадковість** (×0.72).
  Розтягнуте плато ордербуком не ловиться, і збільшення допуску навколо екстремуму не
  допомагає: на ±14 хв уже ×1.00.
- **Обʼєм як ознака розвороту.** Розходиться між ланцюгами найсильніше з усього, і це
  повністю тінь тренду (Δ проти тіла ноги +0.001, p=0.96).
- **Повна матриця 1680 на малих підвибірках.** Програє 16 числам (рівні + нахили) скрізь,
  де вибірка менша за ~3 тисячі вікон.

---

## Порядок команд

```bash
# 0. спершу — моделі (інша лінія, свій щоденник WCLASS_DIARY.md)
python3 mywclass.py data BTC && python3 mywclass.py feat BTC
python3 mywclass.py labels BTC
python3 mywclass.py clf BTC storm 10        # ~12 хв на MPS
python3 mywclass.py clf BTC mid 10
python3 mywclass.py clf BTC calm 10
python3 mywclass.py poles BTC               # ~2 хв (UMAP+HDBSCAN)
python3 mywclass.py pq BTC 10               # ~15 хв
python3 mywclass.py final BTC && python3 mywclass.py four BTC

# 1. власне лінія /pivots
python3 pivot_zones.py zones  BTC           # ~30 с
python3 pivot_zones.py chart  BTC           # ~20 с
python3 pivot_zones.py ident  BTC           # ~3 хв
python3 pivot_zones.py hooks  BTC           # ~2 хв
python3 pivot_zones.py storm4 BTC           # ~1 хв
python3 pivot_zones.py inside BTC all       # ~2 хв
python3 pivot_zones.py tab    BTC

# 2. перевірка сторінки
node test_pages.js
```

---

## Повний код

### Головний скрипт лінії

`pivot_zones.py`:

```python
"""
ЗОНИ ЛОКАЛЬНИХ ВЕРШИН І ДНА (BTC, увесь датасет) + перевірка, чи їх видно
сигналами 4 моделей лінії /mywclass.

Постановка користувача дослівно:
  · шукаємо локальні вершини та дно РОЗМІРОМ 0.5% — зона це проміжок часу,
    поки ціна тримається в смузі 0.5% від самого екстремуму;
  · між крайніми точками попереднього екстремуму, поточного і наступного
    відстань не менше 1% — це рівно ZigZag з порогом 1% (кожна нога ≥1%),
    плюс явна перевірка обох ніг, бо перший і останній пивот незавершені;
  · графік BTC із зонами: червоне знизу (дно), зелене зверху (вершини);
  · чи можна ідентифікувати зону, маючи відповіді 4 моделей за ОДИН набір
    або більше — аж до тривалості НАЙКОРОТШОЇ зони.

Важливе про чесність. Мітка зони визначена з майбутнього: вершина стає
вершиною лише після падіння на 1% після неї. Тому «ідентифікація» тут — це
наполовину прогноз розвороту, і всі числа міряються проти нуль-контролю
(циклічний зсув відповідей моделей — руйнує звʼязок, зберігає автокореляцію)
та проти простої цінової ознаки (перцентиль ціни у ковзному вікні).

Дані: та сама сітка вікон 210 с, що в /mywclass (той самий t0/t1), тож вікно
в вікно зіставні. Ціна береться з РЕАЛЬНИХ спостережень (без інтерполяції) —
у BTC 18.4% секунд відсутні, інтерполяція наробила б фальшивих екстремумів.

Етапи:
    python3 pivot_zones.py zones BTC     # ZigZag 1% + зони 0.5%
    python3 pivot_zones.py chart BTC     # графік ціни із зонами (+zoom)
    python3 pivot_zones.py ident BTC     # чи ідентифікують зони 4 моделі
    python3 pivot_zones.py hooks BTC     # де в зоні реально є за що зачепитись
    python3 pivot_zones.py chains BTC    # ланцюг вершин проти ланцюга дна
    python3 pivot_zones.py storm4 BTC    # розподіл «шторму» у 4 станах ринку
    python3 pivot_zones.py inside BTC    # самі 1680 чисел усередині штормів
    python3 pivot_zones.py tab   BTC     # зведення для вкладки
"""
import os, sys, json, time
import numpy as np

import standard_signals as ST

ROOT = os.path.dirname(os.path.abspath(__file__))
OUT_DIR = os.path.join(ROOT, "media", "analyst", "pivots")
MYW = os.path.join(ROOT, "media", "analyst", "mywclass", "cache")

THR = 0.01           # мінімальна нога зигзага (1%)
BAND = 0.005         # глибина зони (0.5%)
WIN = 210            # вікно моделей, с
COVER = 0.5          # яка частка вікна має лежати в зоні, щоб вікно вважалось зонним
BLOCK_H = 12         # блок для чесного спліту
EMBARGO_S = 3600
FOLDS = 5
SEED = 20260806
TARGETS = ("storm", "mid", "calm", "pcaquiet")   # чотири цілі, які тренує лінія

FG, BG, GRID = "#e6edf3", "#0d1117", "#30363d"
C_TOP, C_BOT, C_PX = "#3fb950", "#f85149", "#8b949e"

T = {"uk": {"price": "ціна BTC, $", "day": "доба датасету",
            "top": "зона вершини (0.5%)", "bot": "зона дна (0.5%)",
            "title": "BTC · локальні вершини і дно: ноги ZigZag ≥1%, зона = смуга 0.5% від екстремуму",
            "zoom": "BTC · збільшено: зони вершин (зелене) і дна (червоне)",
            "hours": "години",
            "ident": "Чи видно зону за відповідями 4 моделей",
            "L": "довжина набору відповідей, вікон по 210 с",
            "auc": "AUC", "null": "нуль (зсув)", "px": "цінова ознака",
            "prof": "профіль відповідей моделей навколо екстремуму",
            "dt": "час від екстремуму, хв", "share": "частка вікон, %"},
      "en": {"price": "BTC price, $", "day": "day of dataset",
             "top": "top zone (0.5%)", "bot": "bottom zone (0.5%)",
             "title": "BTC · local tops and bottoms: ZigZag legs >=1%, zone = 0.5% band around the extremum",
             "zoom": "BTC · zoom: top zones (green) and bottom zones (red)",
             "hours": "hours",
             "ident": "Can the four models see the zone",
             "L": "length of the answer set, 210 s windows",
             "auc": "AUC", "null": "null (shift)", "px": "price feature",
             "prof": "model answers around the extremum",
             "dt": "time from the extremum, min", "share": "share of windows, %"}}


def log(*a):
    print(time.strftime("[%H:%M:%S]"), *a, flush=True)


# ======================= 1. ЦІНА + ЗИГЗАГ + ЗОНИ =======================
def price_series(coin):
    """Реальні спостереження в межах [t0,t1] лінії /mywclass, без інтерполяції."""
    S = ST.load(coin)
    if S is None:
        return None
    _, first = np.unique(S["ts"], return_index=True)
    first = np.sort(first)
    ts = S["ts"][first].astype(np.int64)
    px = S["price"][first].astype(np.float64)
    meta = json.load(open(os.path.join(MYW, f"{coin}_data.json")))
    t0, t1 = int(meta["t0"]), int(meta["t1"])
    m = (ts >= t0) & (ts <= t1) & np.isfinite(px) & (px > 0)
    log(f"{coin}: ціна — {int(m.sum())} реальних точок у межах /mywclass "
        f"({(t1-t0)/86400:.1f} діб)")
    return ts[m], px[m], t0, t1


def zigzag(ts, px, thr=THR):
    """Пивоти з чергуванням: новий екстремум фіксується, коли ціна відійшла
    від поточного на thr. Кожна нога за побудовою ≥ thr, але перший і
    останній пивот незавершені — їх ловить окрема перевірка нижче."""
    piv = []
    imax = imin = 0
    direction = 0
    for k in range(1, len(px)):
        if px[k] > px[imax]:
            imax = k
        if px[k] < px[imin]:
            imin = k
        if direction >= 0 and px[k] <= px[imax] * (1 - thr):
            piv.append((imax, 1)); direction = -1; imin = k
        elif direction <= 0 and px[k] >= px[imin] * (1 + thr):
            piv.append((imin, -1)); direction = 1; imax = k
    return piv


def zone_extent(px, i, kind, band=BAND):
    """Максимальний неперервний проміжок навколо i, де ціна тримається в смузі
    band від екстремуму (вершина: не нижче; дно: не вище)."""
    p = px[i]
    if kind > 0:
        lo, hi = p * (1 - band), np.inf
    else:
        lo, hi = -np.inf, p * (1 + band)
    a = i
    while a > 0 and lo <= px[a - 1] <= hi:
        a -= 1
    b = i
    n = len(px)
    while b < n - 1 and lo <= px[b + 1] <= hi:
        b += 1
    return a, b


def stage_zones(coin):
    os.makedirs(OUT_DIR, exist_ok=True)
    r = price_series(coin)
    if r is None:
        log("немає даних"); return None
    ts, px, t0, t1 = r
    piv = zigzag(ts, px)
    log(f"{coin}: пивотів ZigZag {THR*100:.1f}% — {len(piv)}")

    # ПЕРЕВІРКА ОБОХ НІГ: лишаємо лише ті екстремуми, де і попередня, і
    # наступна нога ≥ THR. Це знімає незавершені крайні пивоти.
    zones, legs = [], []
    for j in range(1, len(piv) - 1):
        i, kind = piv[j]
        ip, _ = piv[j - 1]
        inx, _ = piv[j + 1]
        lp = abs(px[i] / px[ip] - 1)
        ln = abs(px[inx] / px[i] - 1)
        legs.append([lp * 100, ln * 100])
        if lp < THR or ln < THR:
            continue
        a, b = zone_extent(px, i, kind)
        gaps = np.diff(ts[a:b + 1]) if b > a else np.array([0])
        zones.append({"kind": "top" if kind > 0 else "bot",
                      "t": int(ts[i]), "p": round(float(px[i]), 2),
                      "t_a": int(ts[a]), "t_b": int(ts[b]),
                      "dur": int(ts[b] - ts[a]),
                      "leg_prev": round(lp * 100, 3),
                      "leg_next": round(ln * 100, 3),
                      "gap_max": int(gaps.max()) if len(gaps) else 0})
    zones.sort(key=lambda z: z["t_a"])

    # зони не мають перетинатись: нога ≥1%, смуга 0.5% → верхній край зони дна
    # нижчий за нижній край зони вершини. Перевіряємо фактично.
    over = sum(1 for u, v in zip(zones[:-1], zones[1:]) if v["t_a"] <= u["t_b"])
    dur = np.array([z["dur"] for z in zones], float)
    tops = [z for z in zones if z["kind"] == "top"]
    bots = [z for z in zones if z["kind"] == "bot"]
    L = np.array(legs) if legs else np.zeros((1, 2))
    cover = dur.sum() / max(t1 - t0, 1) * 100

    out = {"coin": coin, "built": int(time.time()), "thr_pct": THR * 100,
           "band_pct": BAND * 100, "t0": t0, "t1": t1,
           "days": round((t1 - t0) / 86400, 1),
           "n_piv": len(piv), "n_zones": len(zones),
           "n_top": len(tops), "n_bot": len(bots),
           "overlaps": int(over),
           "cover_pct": round(float(cover), 2),
           "dur": {"min": int(dur.min()), "q10": int(np.quantile(dur, .1)),
                   "med": int(np.median(dur)), "q90": int(np.quantile(dur, .9)),
                   "max": int(dur.max()), "mean": int(dur.mean())},
           "leg": {"min": round(float(L[:, 0].min()), 3),
                   "med": round(float(np.median(L)), 3),
                   "max": round(float(L.max()), 3)},
           "per_day": round(len(zones) / max((t1 - t0) / 86400, 1e-9), 2),
           "zones": zones}
    json.dump(out, open(os.path.join(OUT_DIR, f"{coin}_zones.json"), "w"),
              ensure_ascii=False)
    log(f"{coin}: зон {len(zones)} (вершин {len(tops)}, дна {len(bots)}) · "
        f"{out['per_day']}/добу · перетинів {over}")
    log(f"{coin}: тривалість зони — мін {out['dur']['min']} с · медіана "
        f"{out['dur']['med']} с · макс {out['dur']['max']} с · покриття "
        f"{out['cover_pct']}% часу")
    log(f"{coin}: ноги — мін {out['leg']['min']}% · медіана {out['leg']['med']}%")
    return out


# ======================= 2. ГРАФІК ЦІНИ ІЗ ЗОНАМИ =======================
def _style():
    import matplotlib
    matplotlib.use("Agg")
    import matplotlib.pyplot as plt
    plt.rcParams.update({"figure.facecolor": BG, "axes.facecolor": BG,
                         "savefig.facecolor": BG, "text.color": FG,
                         "axes.labelcolor": FG, "xtick.color": FG,
                         "ytick.color": FG, "axes.edgecolor": GRID,
                         "font.size": 9})
    return plt


def stage_chart(coin, panels=6, zoom_days=4.0):
    from matplotlib.patches import Rectangle
    plt = _style()
    Z = json.load(open(os.path.join(OUT_DIR, f"{coin}_zones.json")))
    ts, px, t0, t1 = price_series(coin)
    # проріджуємо до хвилини — на 171 добу секунди все одно не видно
    step = 60
    tsd, pxd = ts[::step], px[::step]
    zs = Z["zones"]

    for lang in ("uk", "en"):
        t = T[lang]
        sfx = "" if lang == "uk" else "_en"
        span = (t1 - t0) / panels
        fig, ax = plt.subplots(panels, 1, figsize=(14, 2.35 * panels))
        for k in range(panels):
            a0, a1 = t0 + k * span, t0 + (k + 1) * span
            m = (tsd >= a0) & (tsd <= a1)
            axk = ax[k]
            axk.plot((tsd[m] - t0) / 86400, pxd[m], color=C_PX, lw=0.7)
            for z in zs:
                if z["t_b"] < a0 or z["t_a"] > a1:
                    continue
                x0 = (z["t_a"] - t0) / 86400
                w = max((z["t_b"] - z["t_a"]) / 86400, span / 86400 * 0.0016)
                if z["kind"] == "top":
                    y0, h = z["p"] * (1 - BAND), z["p"] * BAND
                    c = C_TOP
                else:
                    y0, h = z["p"], z["p"] * BAND
                    c = C_BOT
                axk.add_patch(Rectangle((x0, y0), w, h, color=c, alpha=0.55,
                                        lw=0, zorder=3))
            axk.set_xlim((a0 - t0) / 86400, (a1 - t0) / 86400)
            axk.grid(alpha=0.15, color=GRID)
            axk.set_ylabel(t["price"], fontsize=8)
            if k == panels - 1:
                axk.set_xlabel(t["day"])
        h = [Rectangle((0, 0), 1, 1, color=C_TOP, alpha=0.55),
             Rectangle((0, 0), 1, 1, color=C_BOT, alpha=0.55)]
        ax[0].legend(h, [t["top"], t["bot"]], loc="upper left", fontsize=8,
                     facecolor="#161b22", edgecolor=GRID, framealpha=0.95)
        fig.suptitle(t["title"], fontsize=11, y=0.995)
        fig.tight_layout(rect=[0, 0, 1, 0.985])
        p = os.path.join(OUT_DIR, f"{coin}_zones{sfx}.png")
        fig.savefig(p, dpi=110); plt.close(fig)
        log(f"{coin}: {p}")

        # ZOOM: найгустіший відрізок zoom_days
        W = int(zoom_days * 86400)
        tz = np.array([z["t"] for z in zs])
        best, bc = t0, -1
        for s in range(t0, t1 - W, 3600 * 6):
            c = int(((tz >= s) & (tz < s + W)).sum())
            if c > bc:
                bc, best = c, s
        m = (tsd >= best) & (tsd <= best + W)
        fig, axk = plt.subplots(figsize=(14, 5))
        axk.plot((tsd[m] - best) / 3600, pxd[m], color=C_PX, lw=1.0)
        for z in zs:
            if z["t_b"] < best or z["t_a"] > best + W:
                continue
            x0 = (z["t_a"] - best) / 3600
            w = max((z["t_b"] - z["t_a"]) / 3600, 0.02)
            if z["kind"] == "top":
                y0, h2, c = z["p"] * (1 - BAND), z["p"] * BAND, C_TOP
            else:
                y0, h2, c = z["p"], z["p"] * BAND, C_BOT
            axk.add_patch(Rectangle((x0, y0), w, h2, color=c, alpha=0.55, lw=0,
                                    zorder=3))
        axk.set_xlim(0, W / 3600)
        axk.set_xlabel(t["hours"]); axk.set_ylabel(t["price"])
        axk.grid(alpha=0.15, color=GRID)
        axk.legend(h, [t["top"], t["bot"]], loc="upper left", fontsize=8,
                   facecolor="#161b22", edgecolor=GRID, framealpha=0.95)
        axk.set_title(f'{t["zoom"]} · {bc} шт' if lang == "uk" else
                      f'{t["zoom"]} · {bc} zones', fontsize=11)
        fig.tight_layout()
        p = os.path.join(OUT_DIR, f"{coin}_zoom{sfx}.png")
        fig.savefig(p, dpi=110); plt.close(fig)
        log(f"{coin}: {p}")
    return True


# ======================= 3. ЧИ ВИДНО ЗОНУ МОДЕЛЯМ =======================
def window_labels(coin):
    """Мітка вікна 210 с: top / bot / none за часткою перекриття із зоною."""
    Z = json.load(open(os.path.join(OUT_DIR, f"{coin}_zones.json")))
    d = np.load(os.path.join(MYW, f"{coin}_data.npz"), mmap_mode="r")
    ts = np.asarray(d["ts"]).astype(np.int64)
    px = np.asarray(d["px"]).astype(np.float64)
    za = np.array([z["t_a"] for z in Z["zones"]], np.int64)
    zb = np.array([z["t_b"] for z in Z["zones"]], np.int64)
    zk = np.array([1 if z["kind"] == "top" else -1 for z in Z["zones"]], np.int8)
    ov = np.zeros((len(ts), 2))
    j = np.searchsorted(za, ts + WIN)
    for i in range(len(ts)):
        a, b = ts[i], ts[i] + WIN
        k = j[i] - 1
        while k >= 0 and zb[k] >= a:
            o = min(b, zb[k]) - max(a, za[k])
            if o > 0:
                ov[i, 0 if zk[k] > 0 else 1] += o
            k -= 1
    lab = np.zeros(len(ts), np.int8)
    lab[ov[:, 0] >= COVER * WIN] = 1
    lab[ov[:, 1] >= COVER * WIN] = -1
    return ts, px, lab, Z


def model_answers(coin):
    """4 відповіді на вікно: ймовірність переможця крос-валідації (out-of-fold,
    тобто моделлю, яка цього вікна не бачила) + її дискретний прапорець.

    УВАГА: ціль pcaquiet визначена лише на вікнах з 28-денною нормою (перші
    28 діб — прогрів, мітки немає). Там ставиться NaN, а всі етапи працюють
    на перетині, де є ВСІ чотири відповіді."""
    n = len(np.load(os.path.join(MYW, f"{coin}_data.npz"), mmap_mode="r")["ts"])
    cols, prob, flag = [], [], []
    for tg in TARGETS:
        z = np.load(os.path.join(MYW, f"{coin}_oof_{tg}.npz"))
        cv = json.load(open(os.path.join(MYW, f"{coin}_clf_{tg}.json")))
        eng = cv["best"]
        thr = cv["models"][eng]["thr"]
        p = np.full(n, np.nan)
        if "wid" in z.files:                       # ціль на підмножині вікон
            p[z["wid"]] = z[eng].astype(np.float64)
        else:
            p[:] = z[eng].astype(np.float64)
        prob.append(p)
        flag.append(np.where(np.isfinite(p), (p >= thr).astype(np.float64), np.nan))
        cols.append(f"{tg}:{eng}")
    P = np.array(prob).T; F = np.array(flag).T
    ok = np.isfinite(P).all(axis=1)
    return P, F, cols, ok


def stack(A, ts, Lmax, mode="trail"):
    """A[i] з L послідовних вікон, що йдуть стик у стик (без розривів).
    trail  — L вікон, що ЗАКІНЧУЮТЬСЯ поточним (лише минуле, causal);
    center — набір навколо поточного (можна, бо мітка зони і так з майбутнього)."""
    n = len(ts)
    cont = np.zeros(n, np.int32)          # скільки суцільних вікон позаду
    for i in range(1, n):
        cont[i] = cont[i - 1] + 1 if ts[i] - ts[i - 1] == WIN else 0
    fwd = np.zeros(n, np.int32)           # скільки суцільних вікон попереду
    for i in range(n - 2, -1, -1):
        fwd[i] = fwd[i + 1] + 1 if ts[i + 1] - ts[i] == WIN else 0
    out = {}
    for L in range(1, Lmax + 1):
        if mode == "trail":
            ok = np.flatnonzero(cont >= L - 1)
            sh = range(L)
            M = np.concatenate([A[ok - k] for k in sh], axis=1)
        else:
            b = (L - 1) // 2
            f = L - 1 - b
            ok = np.flatnonzero((cont >= b) & (fwd >= f))
            M = np.concatenate([A[ok + k] for k in range(-b, f + 1)], axis=1)
        out[L] = (ok, M)
    return out


def block_folds(ts, folds=FOLDS, seed=SEED):
    rng = np.random.default_rng(seed)
    blk = (ts - ts[0]) // (BLOCK_H * 3600)
    ub = np.unique(blk)
    f = rng.permutation(len(ub)) % folds
    m = dict(zip(ub.tolist(), f.tolist()))
    return np.array([m[b] for b in blk.tolist()])


def _fit(M, y, fold, folds=FOLDS, kind="lr", ts=None):
    """Оцінка out-of-fold по блоках з ембарго: сусідні вікна корелюють, а мітка
    ahead зазирає на годину вперед — без ембарго вона перетнула б межу блока."""
    from sklearn.linear_model import LogisticRegression
    from sklearn.ensemble import HistGradientBoostingClassifier
    p = np.full(len(y), np.nan)
    for f in range(folds):
        va = fold == f
        tr = ~va
        if ts is not None:
            vt = ts[va]
            j = np.searchsorted(vt, ts)
            near = np.zeros(len(ts), bool)
            for sh in (0, -1):
                k = np.clip(j + sh, 0, len(vt) - 1)
                near |= np.abs(ts - vt[k]) <= EMBARGO_S
            tr = tr & ~near
        if y[tr].sum() < 20 or y[va].sum() < 5:
            continue
        if kind == "lr":
            mu, sd = M[tr].mean(0), M[tr].std(0) + 1e-9
            m = LogisticRegression(max_iter=2000, C=1.0,
                                   class_weight="balanced")
            m.fit((M[tr] - mu) / sd, y[tr])
            p[va] = m.predict_proba((M[va] - mu) / sd)[:, 1]
        else:
            m = HistGradientBoostingClassifier(max_iter=150, learning_rate=0.1,
                                               max_bins=64, random_state=SEED)
            m.fit(M[tr], y[tr])
            p[va] = m.predict_proba(M[va])[:, 1]
    return p


def _score(y, p):
    from sklearn.metrics import roc_auc_score, average_precision_score
    m = np.isfinite(p)
    y, p = y[m], p[m]
    if y.sum() < 5 or y.sum() == len(y):
        return None
    base = y.mean()
    thr = np.quantile(p, 1 - base)
    hit = y[p >= thr]
    return {"auc": round(float(roc_auc_score(y, p)), 4),
            "ap": round(float(average_precision_score(y, p)), 4),
            "base": round(float(base), 4),
            "lift": round(float(hit.mean() / base), 3) if len(hit) else None,
            "n": int(len(y)), "pos": int(y.sum())}


def price_feature(ts, px, look=86400):
    """ПОЛОЖЕННЯ ціни: перцентиль у ковзному вікні 24 год (causal) + нахил.
    Це еталон на питання «вершина чи дно»."""
    n = len(ts)
    pc = np.zeros(n); sl = np.zeros(n)
    j = 0
    for i in range(n):
        while ts[j] < ts[i] - look:
            j += 1
        seg = px[j:i + 1]
        pc[i] = (seg <= px[i]).mean()
        sl[i] = px[i] / seg[0] - 1 if len(seg) > 1 else 0.0
    return np.stack([pc, sl], axis=1)


def price_vol(coin, ts):
    """РОЗМАХ ціни у самому вікні і в попередньому — еталон на питання «коли».
    Ордербук мусить бити цю просту ознаку, інакше він нічого не додає."""
    rts, rpx, _, _ = price_series(coin)
    a = np.searchsorted(rts, ts)
    b = np.searchsorted(rts, ts + WIN)
    rg = np.zeros(len(ts))
    for i in range(len(ts)):
        if b[i] > a[i] + 1:
            s = rpx[a[i]:b[i]]
            rg[i] = (s.max() - s.min()) / max(s[0], 1e-9)
    prev = np.concatenate([[0.0], rg[:-1]])
    return np.stack([np.log1p(rg * 1e3), np.log1p(prev * 1e3)], axis=1)


def point_labels(ts, Z, near=1, ahead=3600):
    """Мітки не зони, а САМОГО екстремуму — профіль показав, що моделі бʼють
    у момент, а не в 26-хвилинну зону.
      peak  — екстремум лежить усередині вікна;
      near  — екстремум у вікні ±near сусідів (±3.5 хв при near=1);
      ahead — екстремум настане в наступну годину ПІСЛЯ кінця вікна (прогноз)."""
    tz = np.array(sorted(z["t"] for z in Z["zones"]), np.int64)
    n = len(ts)
    peak = np.zeros(n, np.int8)
    j = np.searchsorted(tz, ts)
    k = np.searchsorted(tz, ts + WIN)
    peak[k > j] = 1
    nr = peak.copy()
    for s in range(1, near + 1):
        nr[s:] |= peak[:-s]
        nr[:-s] |= peak[s:]
    a = np.searchsorted(tz, ts + WIN)
    b = np.searchsorted(tz, ts + WIN + ahead)
    ah = (b > a).astype(np.int8)
    return peak, nr, ah


def stage_ident(coin):
    Ldata = window_labels(coin)
    ts, px, lab, Z = Ldata
    prob, flag, cols, okm = model_answers(coin)
    ts, px, lab = ts[okm], px[okm], lab[okm]
    prob, flag = prob[okm], flag[okm]
    log(f"{coin}: вікон з усіма 4 відповідями {int(okm.sum())} з {len(okm)} "
        f"(прогрів норми pcaquiet відкинуто)")
    log(f"{coin}: вікон {len(ts)} · зонних вершин {(lab==1).sum()} "
        f"({(lab==1).mean()*100:.2f}%) · зонних дна {(lab==-1).sum()} "
        f"({(lab==-1).mean()*100:.2f}%)")

    dmin = Z["dur"]["min"]
    Lfit = max(1, dmin // WIN)              # скільки вікон влазить у НАЙКОРОТШУ зону
    Lmax = max(Lfit, 8)                     # за межею — довідково, з позначкою
    log(f"{coin}: найкоротша зона {dmin} с → у неї влазить {Lfit} вікон; "
        f"скануємо L=1..{Lmax}")

    fold = block_folds(ts)
    SP = {"prob": stack(prob, ts, Lmax), "flag": stack(flag, ts, Lmax),
          "center": stack(prob, ts, Lmax, mode="center")}
    SPX = stack(price_feature(ts, px), ts, Lmax)
    SPV = stack(price_vol(coin, ts), ts, Lmax)

    peak, near, ahead = point_labels(ts, Z)
    tasks = {"top": (lab == 1).astype(np.int8),
             "bot": (lab == -1).astype(np.int8),
             "any": (lab != 0).astype(np.int8),
             "peak": peak, "near": near, "ahead": ahead}
    res_extra = {k: round(float(v.mean()), 4) for k, v in tasks.items()}
    log(f"{coin}: частки міток — " + " · ".join(
        f"{k} {v*100:.2f}%" for k, v in res_extra.items()))
    res = {"coin": coin, "built": int(time.time()), "cols": cols,
           "n_win": int(len(ts)), "cover": COVER, "L_fit": int(Lfit),
           "L_max": int(Lmax), "dur_min": int(dmin), "folds": FOLDS,
           "block_h": BLOCK_H, "sweep": {}, "null": {}, "price": {},
           "combo": {}, "dir": None, "cells": {}}

    rng = np.random.default_rng(SEED)
    for tn, y in tasks.items():
        res["sweep"][tn] = {}; res["null"][tn] = {}; res["price"][tn] = {}
        for L in range(1, Lmax + 1):
            row = {}
            for var in ("prob", "flag", "center"):
                ok, M = SP[var][L]
                s = _fit(M, y[ok], fold[ok], kind="lr", ts=ts[ok])
                r = _score(y[ok], s)
                if r:
                    row[var] = r
                if var == "prob":
                    s2 = _fit(M, y[ok], fold[ok], kind="gbm", ts=ts[ok])
                    r2 = _score(y[ok], s2)
                    if r2:
                        row["gbm"] = r2
            res["sweep"][tn][L] = row
            log(f"  {tn} L={L}: " + " · ".join(
                f"{k} AUC {v['auc']:.3f} PR {v['ap']:.3f} lift {v['lift']}"
                for k, v in row.items()))
        # НУЛЬ: циклічний зсув відповідей (звʼязок зруйновано, автокореляція жива)
        ok, M = SP["prob"][1]
        au, lf = [], []
        for _ in range(5):
            sh = int(rng.integers(len(M) // 10, len(M) - len(M) // 10))
            r = _score(y[ok], _fit(np.roll(M, sh, axis=0), y[ok], fold[ok],
                                   ts=ts[ok]))
            if r:
                au.append(r["auc"]); lf.append(r["lift"])
        res["null"][tn] = {"auc": round(float(np.mean(au)), 4),
                           "auc_sd": round(float(np.std(au)), 4),
                           "lift": round(float(np.mean(lf)), 3)} if au else None
        # ДВА ЦІНОВІ ЕТАЛОНИ: положення (вершина/дно) і розмах (коли)
        res["price"][tn] = {}
        for nm, SS in (("pos", SPX), ("vol", SPV)):
            ok, M = SS[1]
            res["price"][tn][nm] = _score(y[ok], _fit(M, y[ok], fold[ok],
                                                      kind="gbm", ts=ts[ok]))
        pp = res["price"][tn]
        log(f"  {tn}: нуль AUC {res['null'][tn]['auc'] if res['null'][tn] else '—'}"
            f" · ціна-положення AUC {pp['pos']['auc'] if pp['pos'] else '—'}"
            f" · ціна-розмах AUC {pp['vol']['auc'] if pp['vol'] else '—'}")
        # ГОЛОВНЕ ПИТАННЯ: чи ордербук додає ПОНАД ціну (обидві ознаки разом)
        res["combo"][tn] = {}
        for L in (1, min(5, Lmax), Lmax):
            okv, V = SPV[L]; okp, P = SP["prob"][L]
            ok = np.intersect1d(okv, okp)
            iv = np.searchsorted(okv, ok); ip = np.searchsorted(okp, ok)
            M = np.concatenate([V[iv], P[ip]], axis=1)
            a = _score(y[ok], _fit(M, y[ok], fold[ok], kind="gbm", ts=ts[ok]))
            b = _score(y[ok], _fit(V[iv], y[ok], fold[ok], kind="gbm", ts=ts[ok]))
            res["combo"][tn][L] = {"both": a, "price": b,
                                   "d_auc": round(a["auc"] - b["auc"], 4)
                                   if a and b else None}
            log(f"    {tn} L={L}: ціна+ордербук AUC {a['auc']:.4f} проти самої "
                f"ціни {b['auc']:.4f} → +{a['auc']-b['auc']:+.4f} · "
                f"lift {b['lift']}→{a['lift']}")

    # НАПРЯМ: вершина проти дна ЛИШЕ серед зонних вікон
    m = lab != 0
    ok, M = SP["prob"][1]
    sel = m[ok]
    yd = (lab[ok][sel] == 1).astype(np.int8)
    res["dir"] = _score(yd, _fit(M[sel], yd, fold[ok][sel], ts=ts[ok][sel]))
    log(f"  напрям (вершина проти дна серед зонних): "
        f"AUC {res['dir']['auc'] if res['dir'] else '—'}")

    # ІНТЕРПРЕТАЦІЯ: частота зони при кожному спрацюванні
    for j, c in enumerate(cols):
        f = flag[:, j] > 0
        res["cells"][c] = {
            "share": round(float(f.mean()), 4),
            "p_any": round(float((lab != 0)[f].mean()), 4),
            "p_top": round(float((lab == 1)[f].mean()), 4),
            "p_bot": round(float((lab == -1)[f].mean()), 4),
            "p_peak": round(float(peak[f].mean()), 4),
            "lift_any": round(float((lab != 0)[f].mean() /
                                    max((lab != 0).mean(), 1e-9)), 3),
            "lift_peak": round(float(peak[f].mean() /
                                     max(peak.mean(), 1e-9)), 3),
            "rec_peak": round(float(peak[f].sum() / max(peak.sum(), 1)), 4)}
        k = res["cells"][c]
        log(f"  {c}: {f.mean()*100:.1f}% часу → зона {k['p_any']*100:.2f}% "
            f"(×{k['lift_any']}) · момент екстремуму ×{k['lift_peak']} "
            f"(ловить {k['rec_peak']*100:.0f}% усіх)")
    res["base_any"] = round(float((lab != 0).mean()), 4)
    res["shares"] = res_extra

    # ПРОФІЛЬ навколо екстремуму
    res["profile"] = profile(ts, lab, prob, Z)
    json.dump(res, open(os.path.join(OUT_DIR, f"{coin}_ident.json"), "w"),
              ensure_ascii=False)
    charts_ident(coin, res)
    return res


def profile(ts, lab, prob, Z, half=24):
    """Середня відповідь кожної моделі як функція часу до/після екстремуму."""
    tz = np.array([z["t"] for z in Z["zones"]], np.int64)
    idx = np.searchsorted(ts, tz)
    idx = idx[(idx > half) & (idx < len(ts) - half)]
    off = np.arange(-half, half + 1)
    M = np.zeros((len(off), prob.shape[1]))
    for k, o in enumerate(off):
        M[k] = prob[idx + o].mean(0)
    return {"off_min": (off * WIN / 60).round(1).tolist(),
            "curves": {c: M[:, j].round(4).tolist()
                       for j, c in enumerate(TARGETS)},
            "base": prob.mean(0).round(4).tolist(), "n": int(len(idx))}


def charts_ident(coin, R):
    plt = _style()
    CC = {"storm": "#f85149", "mid": "#d29922", "calm": "#58a6ff",
          "pcaquiet": "#a371f7"}
    for lang in ("uk", "en"):
        t = T[lang]
        sfx = "" if lang == "uk" else "_en"
        fig, ax = plt.subplots(1, 2, figsize=(13, 4.6))
        Ls = sorted(int(k) for k in R["sweep"]["any"])
        for tn, c in (("any", "#8b949e"), ("top", C_TOP), ("bot", C_BOT),
                      ("peak", "#d29922"), ("near", "#a371f7"),
                      ("ahead", "#58a6ff")):
            if tn not in R["sweep"]:
                continue
            v = [R["sweep"][tn][str(L) if str(L) in R["sweep"][tn] else L]
                 .get("prob", {}).get("auc", np.nan) for L in Ls]
            ax[0].plot(Ls, v, marker="o", ms=3.5, color=c, lw=1.2, label=tn)
            nl = R["null"][tn]
            if nl:
                ax[0].axhline(nl["auc"], color=c, ls=":", lw=0.8, alpha=0.7)
            pv = (R["price"].get(tn) or {}).get("vol")
            if pv:
                ax[0].axhline(pv["auc"], color=c, ls="--", lw=0.9, alpha=0.8)
        ax[0].plot([], [], color=FG, ls="--", lw=0.9,
                   label="ціна-розмах" if lang == "uk" else "price range")
        ax[0].plot([], [], color=FG, ls=":", lw=0.9,
                   label="нуль" if lang == "uk" else "null")
        ax[0].axvline(R["L_fit"] + 0.5, color=GRID, ls="--", lw=1)
        ax[0].axhline(0.5, color=GRID, lw=0.8)
        ax[0].set_xlabel(t["L"]); ax[0].set_ylabel(t["auc"])
        ax[0].set_title(t["ident"], fontsize=10)
        ax[0].grid(alpha=0.15, color=GRID)
        ax[0].legend(fontsize=8, facecolor="#161b22", edgecolor=GRID)

        P = R["profile"]
        for j, tg in enumerate(TARGETS):
            ax[1].plot(P["off_min"], P["curves"][tg], color=CC[tg], lw=1.3,
                       label=tg)
            ax[1].axhline(P["base"][j], color=CC[tg], ls=":", lw=0.7, alpha=0.6)
        ax[1].axvline(0, color=FG, lw=0.8, alpha=0.5)
        ax[1].set_xlabel(t["dt"]); ax[1].set_ylabel("p")
        ax[1].set_title(t["prof"], fontsize=10)
        ax[1].grid(alpha=0.15, color=GRID)
        ax[1].legend(fontsize=8, facecolor="#161b22", edgecolor=GRID)
        fig.tight_layout()
        p = os.path.join(OUT_DIR, f"{coin}_ident{sfx}.png")
        fig.savefig(p, dpi=110); plt.close(fig)
        log(f"{coin}: {p}")


# ======================= 4. ЗАЧІПКИ =======================
def price_ret(coin, ts, look=(1, 3, 12)):
    """Хід ціни у вікні і за кілька вікон назад — еталон на питання «вершина
    чи дно»: у вершину ціна прийшла знизу, у дно — згори."""
    rts, rpx, _, _ = price_series(coin)
    a = np.searchsorted(rts, ts)
    b = np.clip(np.searchsorted(rts, ts + WIN) - 1, 0, len(rpx) - 1)
    p0, p1 = rpx[np.clip(a, 0, len(rpx) - 1)], rpx[b]
    cols = [np.log(p1 / np.maximum(p0, 1e-9))]
    for k in look:
        j = np.clip(np.searchsorted(rts, ts - k * WIN), 0, len(rpx) - 1)
        cols.append(np.log(p1 / np.maximum(rpx[j], 1e-9)))
    return np.stack(cols, axis=1)


def stage_hooks(coin):
    """Де В ЗОНІ реально є за що зачепитись. Пʼять окремих питань:
      A. форма сигналу ВСЕРЕДИНІ зони (нормований час) — чи це «бутерброд»
         шторм → тиша → шторм, бо зона обмежена двома ногами ≥1%;
      B. явне правило «шторм, потім k тихих вікон» як детектор зони;
      C. серед вікон, де шторм УЖЕ горить, — чи відрізнити екстремум від
         звичайного сплеску всередині ноги;
      D. напрям: чи є асиметрія «вершина проти дна» в 89 описах вікна
         (4 моделі її втратили — вони про силу, не про бік);
      E. покриття на рівні ЗОНИ, а не вікна: скільки з 734 екстремумів
         сигнал узагалі торкнувся, і як це залежить від довжини зони."""
    from sklearn.metrics import roc_auc_score, average_precision_score
    ts, px, lab, Z = window_labels(coin)
    prob, flag, cols, okm = model_answers(coin)
    D = np.load(os.path.join(MYW, f"{coin}_feat.npz"))["D"][okm]
    ts, px, lab = ts[okm], px[okm], lab[okm]
    prob, flag = prob[okm], flag[okm]
    peak, near, ahead = point_labels(ts, Z)
    fold = block_folds(ts)
    zs = Z["zones"]
    out = {"coin": coin, "built": int(time.time()), "n_zones": len(zs)}

    # --- A. профіль по НОРМОВАНОМУ часу зони (0 = вхід, 1 = вихід) ---
    nb = 12
    acc = np.zeros((nb, prob.shape[1])); cnt = np.zeros(nb)
    edge = {"pre": np.zeros((6, prob.shape[1])), "post": np.zeros((6, prob.shape[1])),
            "npre": np.zeros(6), "npost": np.zeros(6)}
    for z in zs:
        if z["dur"] < 2 * WIN:
            continue
        m = (ts + WIN > z["t_a"]) & (ts < z["t_b"])
        i = np.flatnonzero(m)
        if len(i) < 2:
            continue
        u = ((ts[i] + WIN / 2 - z["t_a"]) / max(z["dur"], 1)).clip(0, 0.999)
        k = (u * nb).astype(int)
        for a, b in zip(k, i):
            acc[a] += prob[b]; cnt[a] += 1
        j0, j1 = i[0], i[-1]
        for s in range(6):
            if j0 - 1 - s >= 0:
                edge["pre"][s] += prob[j0 - 1 - s]; edge["npre"][s] += 1
            if j1 + 1 + s < len(ts):
                edge["post"][s] += prob[j1 + 1 + s]; edge["npost"][s] += 1
    out["inside"] = {"bins": nb,
                     "curves": {c: (acc[:, j] / np.maximum(cnt, 1)).round(4).tolist()
                                for j, c in enumerate(TARGETS)},
                     "n": cnt.astype(int).tolist(),
                     "base": prob.mean(0).round(4).tolist()}
    out["edges"] = {"pre": {c: (edge["pre"][:, j] / np.maximum(edge["npre"], 1)).round(4).tolist()
                            for j, c in enumerate(TARGETS)},
                    "post": {c: (edge["post"][:, j] / np.maximum(edge["npost"], 1)).round(4).tolist()
                             for j, c in enumerate(TARGETS)}}
    log(f"{coin}/A: шторм усередині зони по 12 кошиках — " +
        " ".join(f"{v:.2f}" for v in out["inside"]["curves"]["storm"]) +
        f" (база {prob[:,0].mean():.2f})")

    # --- B. правило «шторм → k тихих вікон» ---
    st = flag[:, 0] > 0
    calm = flag[:, list(TARGETS).index("calm")] > 0
    cont = np.zeros(len(ts), bool)
    cont[1:] = ts[1:] - ts[:-1] == WIN
    inz = lab != 0
    out["sandwich"] = []
    for k in (1, 2, 3, 4, 6, 8):
        r = np.zeros(len(ts), bool)
        for i in range(k, len(ts)):
            if not cont[i - k + 1:i + 1].all():
                continue
            r[i] = st[i - k] and calm[i - k + 1:i + 1].all()
        if r.sum() < 30:
            continue
        # чесний рахунок: сусідні спрацювання — це ОДНА подія, не N вікон
        runs = int(((r[1:] & ~r[:-1]).sum()) + (1 if r[0] else 0))
        out["sandwich"].append({
            "k": k, "share": round(float(r.mean()), 4),
            "n": int(r.sum()), "runs": runs,
            "p_zone": round(float(inz[r].mean()), 4),
            "lift": round(float(inz[r].mean() / inz.mean()), 3),
            "rec": round(float((inz & r).sum() / max(inz.sum(), 1)), 4)})
        s = out["sandwich"][-1]
        log(f"{coin}/B: шторм + {k} тихих → {s['share']*100:.1f}% часу, "
            f"у зоні {s['p_zone']*100:.1f}% (×{s['lift']})")
    # КОНТРОЛЬ ЦІНОЮ: те саме правило, але «шторм» і «тиша» визначені розмахом
    # ціни у вікні, з ТИМИ Ж частотами спрацювання. Якщо ціна дає той самий
    # підйом — правило не про ордербук.
    rgw = price_vol(coin, ts)[:, 0]
    st_p = rgw >= np.quantile(rgw, 1 - st.mean())
    calm_p = rgw <= np.quantile(rgw, calm.mean())
    out["sandwich_px"] = []
    for k in (1, 2, 3, 4, 6, 8):
        r = np.zeros(len(ts), bool)
        for i in range(k, len(ts)):
            if not cont[i - k + 1:i + 1].all():
                continue
            r[i] = st_p[i - k] and calm_p[i - k + 1:i + 1].all()
        if r.sum() < 30:
            continue
        out["sandwich_px"].append({
            "k": k, "share": round(float(r.mean()), 4),
            "p_zone": round(float(inz[r].mean()), 4),
            "lift": round(float(inz[r].mean() / inz.mean()), 3)})
        s = out["sandwich_px"][-1]
        log(f"{coin}/B: ЦІНОВИЙ контроль k={k} → {s['share']*100:.1f}% часу, "
            f"у зоні {s['p_zone']*100:.1f}% (×{s['lift']})")
    # контроль: сам штиль без попереднього шторму
    only = calm & ~np.concatenate([[False], st[:-1]])
    out["calm_only"] = {"share": round(float(only.mean()), 4),
                        "p_zone": round(float(inz[only].mean()), 4),
                        "lift": round(float(inz[only].mean() / inz.mean()), 3)}
    log(f"{coin}/B: контроль — просто штиль без шторму перед ним: "
        f"у зоні {out['calm_only']['p_zone']*100:.1f}% "
        f"(×{out['calm_only']['lift']})")

    # --- C. серед штормових вікон: екстремум чи просто сплеск у нозі ---
    sel = np.flatnonzero(st)
    y = peak[sel]
    PVV = price_vol(coin, ts)
    PRR = price_ret(coin, ts)
    PXA = np.concatenate([PVV, PRR], axis=1)
    out["within_storm"] = {"n": int(len(sel)), "pos": int(y.sum()),
                           "base": round(float(y.mean()), 4), "sets": {}}
    for fn, F in (("ордербук (89+4)", np.concatenate([D[sel], prob[sel]], axis=1)),
                  ("ціна (розмах+хід)", PXA[sel]),
                  ("ціна + ордербук",
                   np.concatenate([PXA[sel], D[sel], prob[sel]], axis=1))):
        p = _fit(F, y, fold[sel], kind="gbm", ts=ts[sel])
        mfin = np.isfinite(p)
        q = np.quantile(p[mfin], 0.8)
        hi = y[mfin][p[mfin] >= q]
        out["within_storm"]["sets"][fn] = {
            "auc": round(float(roc_auc_score(y[mfin], p[mfin])), 4),
            "ap": round(float(average_precision_score(y[mfin], p[mfin])), 4),
            "top20_prec": round(float(hi.mean()), 4),
            "top20_lift": round(float(hi.mean() / y.mean()), 3)}
        v = out["within_storm"]["sets"][fn]
        log(f"{coin}/C: серед {len(sel)} штормових вікон екстремум "
            f"{y.mean()*100:.1f}% · {fn}: AUC {v['auc']:.3f}, верхня пʼятина "
            f"{v['top20_prec']*100:.1f}% (×{v['top20_lift']})")

    # --- D. напрям: 89 описів вікна проти 4 ймовірностей ТА проти самої ціни ---
    # Тут еталон обовʼязковий: у вершину ціна прийшла знизу, у дно — згори,
    # і це видно з самої ціни задарма. Питання лише в тому, чи додає ордербук.
    PV = price_ret(coin, ts)
    out["direction"] = {}
    for nm, sub in (("peak", peak > 0), ("near", near > 0), ("zone", lab != 0)):
        s = np.flatnonzero(sub & (lab != 0))
        yd = (lab[s] == 1).astype(np.int8)
        for fn, FF in (("описи 89", D[s]), ("4 моделі", prob[s]),
                       ("ціна: хід у вікні", PV[s]),
                       ("ціна+описи", np.concatenate([PV[s], D[s]], axis=1)),
                       ("описи+моделі", np.concatenate([D[s], prob[s]], axis=1))):
            p = _fit(FF, yd, fold[s], kind="gbm", ts=ts[s])
            m2 = np.isfinite(p)
            if m2.sum() < 50 or yd[m2].sum() < 10:
                continue
            out["direction"].setdefault(nm, {})[fn] = {
                "n": int(m2.sum()), "pos": float(yd[m2].mean()),
                "auc": round(float(roc_auc_score(yd[m2], p[m2])), 4)}
        log(f"{coin}/D: напрям на «{nm}» — " + " · ".join(
            f"{k} AUC {v['auc']:.3f}" for k, v in out["direction"][nm].items()))

    # --- E. покриття на рівні ЗОНИ ---
    out["zone_cov"] = []
    tzs = np.array([z["t"] for z in zs], np.int64)
    dur = np.array([z["dur"] for z in zs], float)
    iz = np.searchsorted(ts, tzs) - 1
    iz = np.clip(iz, 0, len(ts) - 1)
    for tol in (0, 1, 2, 4, 8):
        hit = np.zeros(len(zs), bool)
        for a in range(-tol, tol + 1):
            k = np.clip(iz + a, 0, len(ts) - 1)
            hit |= st[k]
        # порівняння з випадковим сигналом тієї ж частоти
        exp = 1 - (1 - st.mean()) ** (2 * tol + 1)
        out["zone_cov"].append({"tol": tol, "cov": round(float(hit.mean()), 4),
                                "exp": round(float(exp), 4),
                                "lift": round(float(hit.mean() / exp), 3)})
        log(f"{coin}/E: ±{tol} вікон — шторм торкнувся {hit.mean()*100:.0f}% "
            f"зон, випадково було б {exp*100:.0f}% (×{out['zone_cov'][-1]['lift']})")
    ter = np.quantile(dur, [1 / 3, 2 / 3])
    hit = np.zeros(len(zs), bool)
    for a in (-1, 0, 1):
        hit |= st[np.clip(iz + a, 0, len(ts) - 1)]
    exp3 = 1 - (1 - st.mean()) ** 3
    out["by_dur"] = []
    for i, (lo, hi2) in enumerate(zip([0, ter[0], ter[1]], [ter[0], ter[1], 1e18])):
        m3 = (dur >= lo) & (dur < hi2)
        out["by_dur"].append({"terc": i + 1, "n": int(m3.sum()),
                              "dur_med": int(np.median(dur[m3])),
                              "cov": round(float(hit[m3].mean()), 4),
                              "exp": round(float(exp3), 4),
                              "lift": round(float(hit[m3].mean() / exp3), 3)})
        b = out["by_dur"][-1]
        log(f"{coin}/E: зони терцилю {i+1} (медіана {b['dur_med']} с) — "
            f"торкнувся {b['cov']*100:.0f}% при випадкових {exp3*100:.0f}% "
            f"(×{b['lift']})")
    json.dump(out, open(os.path.join(OUT_DIR, f"{coin}_hooks.json"), "w"),
              ensure_ascii=False)
    charts_hooks(coin, out)
    return out


def charts_hooks(coin, H):
    plt = _style()
    CC = {"storm": "#f85149", "mid": "#d29922", "calm": "#58a6ff",
          "pcaquiet": "#a371f7"}
    for lang in ("uk", "en"):
        sfx = "" if lang == "uk" else "_en"
        uk = lang == "uk"
        fig, ax = plt.subplots(1, 2, figsize=(13, 4.4))
        nb = H["inside"]["bins"]
        xin = (np.arange(nb) + 0.5) / nb
        xpre = -np.arange(6, 0, -1) * 0.08 - 0.02
        xpost = 1.02 + np.arange(6) * 0.08
        for j, tg in enumerate(TARGETS):
            y = (H["edges"]["pre"][tg][::-1] + H["inside"]["curves"][tg] +
                 H["edges"]["post"][tg])
            x = np.concatenate([xpre, xin, xpost])
            ax[0].plot(x, y, color=CC[tg], lw=1.4, label=tg)
            ax[0].axhline(H["inside"]["base"][j], color=CC[tg], ls=":", lw=0.7,
                          alpha=0.6)
        for v in (0, 1):
            ax[0].axvline(v, color=FG, lw=0.8, alpha=0.5)
        ax[0].set_xlabel("нормований час у зоні (0 — вхід, 1 — вихід)" if uk else
                         "normalised time in the zone (0 in, 1 out)")
        ax[0].set_ylabel("p")
        ax[0].set_title("Форма сигналу всередині зони" if uk else
                        "Signal shape inside the zone", fontsize=10)
        ax[0].grid(alpha=0.15, color=GRID)
        ax[0].legend(fontsize=8, facecolor="#161b22", edgecolor=GRID)

        S = H["sandwich"]
        k = [s["k"] for s in S]
        ax[1].bar([x - 0.2 for x in k], [s["lift"] for s in S], width=0.4,
                  color=C_TOP, label="шторм + k тихих" if uk else "storm + k calm")
        ax[1].bar([x + 0.2 for x in k], [s["share"] * 10 for s in S], width=0.4,
                  color="#8b949e", label="частка часу ×10" if uk else "share of time x10")
        ax[1].axhline(1, color=FG, lw=0.8, alpha=0.6)
        ax[1].axhline(H["calm_only"]["lift"], color=C_BOT, ls="--", lw=1,
                      label="просто штиль" if uk else "calm alone")
        ax[1].set_xlabel("k тихих вікон після шторму" if uk else
                         "k calm windows after the storm")
        ax[1].set_ylabel("× до бази" if uk else "x over base")
        ax[1].set_title("Правило «шторм → тиша» як детектор зони" if uk else
                        "Rule storm → calm as a zone detector", fontsize=10)
        ax[1].grid(alpha=0.15, color=GRID, axis="y")
        ax[1].legend(fontsize=8, facecolor="#161b22", edgecolor=GRID)
        fig.tight_layout()
        p = os.path.join(OUT_DIR, f"{coin}_hooks{sfx}.png")
        fig.savefig(p, dpi=110); plt.close(fig)
        log(f"{coin}: {p}")


# ======================= 5. ДВА ЛАНЦЮГИ =======================
def raw_series(coin):
    """Усі 8 фіч + ціна на реальних секундах у межах /mywclass."""
    S = ST.load(coin)
    _, first = np.unique(S["ts"], return_index=True)
    first = np.sort(first)
    ts = S["ts"][first].astype(np.int64)
    meta = json.load(open(os.path.join(MYW, f"{coin}_data.json")))
    t0, t1 = int(meta["t0"]), int(meta["t1"])
    m = (ts >= t0) & (ts <= t1)
    ts = ts[m]
    px = S["price"][first][m].astype(np.float64)
    X = np.stack([S[f][first][m].astype(np.float32) for f in ST.FEATS], axis=0)
    return ts, px, X


def stage_chains(coin):
    """Два ланцюги: усі шматки після яких ціна ПАДАЛА (вершини) і всі, після
    яких РОСЛА (дно), зшиті кожен в один. Порівняння по 8 фічах, по сигналах
    4 моделей і по тривалості.

    Головна пастка — епоха історії: рівні фіч їздять по місяцях сильніше, ніж
    між вершиною і дном. Тому крім звичайного порівняння 367 проти 367 тут є
    ПАРНЕ: кожна вершина проти СУСІДНЬОГО дна (вони чергуються за побудовою
    зигзага), і різниця рахується всередині пари. Так епоха скорочується."""
    from scipy.stats import mannwhitneyu, wilcoxon
    Z = json.load(open(os.path.join(OUT_DIR, f"{coin}_zones.json")))
    zs = Z["zones"]
    ts, px, X = raw_series(coin)
    scale = np.load(os.path.join(MYW, f"{coin}_feat.npz"))["scale"]
    wts, wpx, lab, _ = window_labels(coin)
    prob, flag, cols, okm = model_answers(coin)
    F8 = list(ST.FEATS)

    # --- посекундні зрізи по кожному шматку ---
    A = np.searchsorted(ts, [z["t_a"] for z in zs])
    B = np.searchsorted(ts, [z["t_b"] for z in zs], side="right")
    kind = np.array([1 if z["kind"] == "top" else -1 for z in zs])
    dur = np.array([z["dur"] for z in zs], float)
    lev = np.full((len(zs), 8), np.nan)      # середній log1p(x/масштаб)
    raw = np.full((len(zs), 8), np.nan)      # середнє сире
    nsec = np.zeros(len(zs), int)
    for i, (a, b) in enumerate(zip(A, B)):
        if b <= a:
            continue
        seg = X[:, a:b]
        nsec[i] = b - a
        raw[i] = seg.mean(axis=1)
        lev[i] = np.log1p(seg / scale[:, None]).mean(axis=1)
    ok = nsec >= 30                          # шматок має бути хоч пів хвилини
    tp, bt = ok & (kind > 0), ok & (kind < 0)
    log(f"{coin}: ланцюг вершин {tp.sum()} шматків / "
        f"{nsec[tp].sum()/3600:.1f} год · ланцюг дна {bt.sum()} / "
        f"{nsec[bt].sum()/3600:.1f} год")

    # --- сигнали моделей на шматок ---
    ctr = wts + WIN // 2
    zi = np.full(len(wts), -1)
    j = np.searchsorted([z["t_a"] for z in zs], ctr) - 1
    for i in range(len(wts)):
        k = j[i]
        if 0 <= k < len(zs) and zs[k]["t_a"] <= ctr[i] <= zs[k]["t_b"]:
            zi[i] = k
    sig = np.full((len(zs), 4), np.nan)
    fla = np.full((len(zs), 4), np.nan)
    nwin = np.zeros(len(zs), int)
    for k in range(len(zs)):
        m = zi == k
        nwin[k] = int(m.sum())
        if m.any():
            sig[k] = prob[m].mean(0); fla[k] = flag[m].mean(0)

    # --- ПАРИ: кожен екстремум проти сусіднього протилежного ---
    pairs = [(i, i + 1) for i in range(len(zs) - 1)
             if ok[i] and ok[i + 1] and kind[i] != kind[i + 1]]
    log(f"{coin}: пар «сусідні вершина+дно» — {len(pairs)}")

    def cmp_col(v, name, unit=""):
        a, b = v[tp], v[bt]
        a, b = a[np.isfinite(a)], b[np.isfinite(b)]
        if len(a) < 20 or len(b) < 20:
            return None
        u = mannwhitneyu(a, b, alternative="two-sided")
        d = []
        for i, k in pairs:
            x, y = (v[i], v[k]) if kind[i] > 0 else (v[k], v[i])
            if np.isfinite(x) and np.isfinite(y):
                d.append(x - y)
        d = np.array(d)
        w = wilcoxon(d)[1] if len(d) > 20 and np.any(d != 0) else np.nan
        return {"name": name, "unit": unit,
                "top": round(float(np.median(a)), 6),
                "bot": round(float(np.median(b)), 6),
                "ratio": round(float(np.median(a) / np.median(b)), 4)
                if np.median(b) not in (0,) else None,
                "diff": round(float(np.median(a) - np.median(b)), 6),
                "p_mw": float(u.pvalue),
                "paired_med": round(float(np.median(d)), 6) if len(d) else None,
                "p_paired": float(w), "n_pairs": int(len(d)),
                "n_top": int(len(a)), "n_bot": int(len(b))}

    rows = []
    for jf, f in enumerate(F8):
        r = cmp_col(lev[:, jf], f, "рівень log1p")
        if r:
            r["group"] = "фічі"; rows.append(r)
    # похідні асиметрії — саме там мала б жити різниця верх/низ
    with np.errstate(divide="ignore", invalid="ignore"):
        asym = {
            "vol_buy / vol_sell": np.log(raw[:, 6] / np.maximum(raw[:, 7], 1e-12)),
            "стіни: resist / support": np.log(raw[:, 0] / np.maximum(raw[:, 3], 1e-12)),
            "приплив: resist+ / support+": np.log(raw[:, 1] / np.maximum(raw[:, 4], 1e-12)),
            "відтік: resist− / support−": np.log(raw[:, 2] / np.maximum(raw[:, 5], 1e-12))}
    for k, v in asym.items():
        v = np.where(np.isfinite(v), v, np.nan)
        r = cmp_col(v, k, "log-відношення")
        if r:
            r["group"] = "асиметрія"; rows.append(r)
    for jt, t2 in enumerate(TARGETS):
        r = cmp_col(sig[:, jt], f"p({t2})", "ймовірність")
        if r:
            r["group"] = "сигнали"; rows.append(r)
    for nm, v in (("тривалість шматка, с", dur),
                  ("нога ДО, %", np.array([z["leg_prev"] for z in zs])),
                  ("нога ПІСЛЯ, %", np.array([z["leg_next"] for z in zs]))):
        r = cmp_col(np.where(ok, v, np.nan), nm, "")
        if r:
            r["group"] = "геометрія"; rows.append(r)

    # BH-поправка на множинність
    ps = np.array([r["p_paired"] if np.isfinite(r["p_paired"]) else 1.0
                   for r in rows])
    o = np.argsort(ps)
    q = np.empty(len(ps)); n = len(ps)
    prev = 1.0
    for rank, i in enumerate(o[::-1]):
        prev = min(prev, ps[i] * n / (n - rank))
        q[i] = prev
    for r, v in zip(rows, q):
        r["q_paired"] = float(v)
        r["sig"] = bool(v < 0.05)

    # --- КОНТРОЛЬ «ТІЛОМ НОГИ» ---
    # Найважливіша перевірка. Асиметрія книги на вершині може бути просто тінню
    # того, що ціна ТУДИ ЙШЛА вгору. Тому для кожного шматка беремо шматок тієї ж
    # тривалості з СЕРЕДИНИ ноги, яка до нього привела, і питаємо: чи асиметрія
    # на екстремумі ВІДРІЗНЯЄТЬСЯ від асиметрії всередині того ж руху.
    lev_c = np.full((len(zs), 8), np.nan)
    raw_c = np.full((len(zs), 8), np.nan)
    for i in range(1, len(zs)):
        a0 = np.searchsorted(ts, zs[i - 1]["t_b"])
        b0 = np.searchsorted(ts, zs[i]["t_a"])
        if b0 - a0 < 30:
            continue
        w = min(int(nsec[i]) if nsec[i] > 0 else (b0 - a0), b0 - a0)
        c = (a0 + b0) // 2
        a1, b1 = max(a0, c - w // 2), min(b0, c + w // 2)
        if b1 - a1 < 30:
            continue
        seg = X[:, a1:b1]
        raw_c[i] = seg.mean(axis=1)
        lev_c[i] = np.log1p(seg / scale[:, None]).mean(axis=1)
    with np.errstate(divide="ignore", invalid="ignore"):
        asym_c = {
            "vol_buy / vol_sell": np.log(raw_c[:, 6] / np.maximum(raw_c[:, 7], 1e-12)),
            "стіни: resist / support": np.log(raw_c[:, 0] / np.maximum(raw_c[:, 3], 1e-12)),
            "приплив: resist+ / support+": np.log(raw_c[:, 1] / np.maximum(raw_c[:, 4], 1e-12)),
            "відтік: resist− / support−": np.log(raw_c[:, 2] / np.maximum(raw_c[:, 5], 1e-12))}
    ctrl = []
    for k in asym.keys():
        v, vc = asym[k], asym_c[k]
        row = {"name": k}
        for nm, m in (("top", tp), ("bot", bt)):
            d = v[m] - vc[m]
            d = d[np.isfinite(d)]
            if len(d) < 20:
                continue
            row[nm] = {"zone": round(float(np.nanmedian(v[m])), 4),
                       "leg": round(float(np.nanmedian(vc[m])), 4),
                       "diff": round(float(np.median(d)), 4),
                       "p": float(wilcoxon(d)[1]) if np.any(d != 0) else np.nan,
                       "n": int(len(d))}
        ctrl.append(row)
        log(f"  контроль {k:28s} верх: нога {row['top']['leg']:+.3f} → зона "
            f"{row['top']['zone']:+.3f} (Δ{row['top']['diff']:+.3f}, p "
            f"{row['top']['p']:.4f}) · низ: нога {row['bot']['leg']:+.3f} → зона "
            f"{row['bot']['zone']:+.3f} (Δ{row['bot']['diff']:+.3f}, p "
            f"{row['bot']['p']:.4f})")

    # --- ЧИ ВІДРІЗНЯЄ ЦЕ ЕКСТРЕМУМ ВІД ТІЛА НОГИ ---
    # Збалансована задача: шматок-екстремум проти шматка-тіла тієї ж тривалості
    # з тієї ж ноги. Ціновий еталон обовʼязковий — у тілі ноги ціна проходить
    # шлях, у зоні тупцює, і це видно задарма.
    from sklearn.linear_model import LogisticRegression
    from sklearn.metrics import roc_auc_score
    va = np.array([asym[k] for k in asym]).T
    vc2 = np.array([asym_c[k] for k in asym]).T
    okz = ok & np.isfinite(va).all(1) & np.isfinite(vc2).all(1)
    idx = np.flatnonzero(okz)
    # цінові еталони на шматок: |чистий хід| і розмах усередині шматка
    def px_stat(a, b):
        if b - a < 5:
            return [np.nan, np.nan]
        s = px[a:b]
        return [abs(np.log(s[-1] / max(s[0], 1e-9))), (s.max() - s.min()) / s[0]]
    PZ, PL = [], []
    for i in idx:
        PZ.append(px_stat(A[i], B[i]))
        a0 = np.searchsorted(ts, zs[i - 1]["t_b"]) if i > 0 else 0
        b0 = np.searchsorted(ts, zs[i]["t_a"])
        c = (a0 + b0) // 2
        w = min(int(nsec[i]), max(b0 - a0, 1))
        PL.append(px_stat(max(a0, c - w // 2), min(b0, c + w // 2)))
    PZ, PL = np.array(PZ), np.array(PL)
    y = np.r_[np.ones(len(idx)), np.zeros(len(idx))]
    blk = np.r_[ts[np.clip(A[idx], 0, len(ts) - 1)], ts[np.clip(A[idx], 0, len(ts) - 1)]]
    fld = ((blk - blk.min()) // (BLOCK_H * 3600) % FOLDS).astype(int)
    sets = {"асиметрія книги (4)": np.r_[va[idx], vc2[idx]],
            "асиметрія × напрям": np.r_[va[idx], vc2[idx]] * np.r_[kind[idx], kind[idx]][:, None],
            "рівні 8 фіч": np.r_[lev[idx], lev_c[idx]],
            "ціна: хід + розмах": np.r_[PZ, PL],
            "ціна + книга": np.r_[np.c_[PZ, va[idx]], np.c_[PL, vc2[idx]]]}
    out_rev = {}
    for nm, M in sets.items():
        m2 = np.isfinite(M).all(1)
        p = np.full(len(y), np.nan)
        for f in range(FOLDS):
            tr = m2 & (fld != f); te = m2 & (fld == f)
            if tr.sum() < 50 or te.sum() < 10 or len(np.unique(y[tr])) < 2:
                continue
            mu, sd = M[tr].mean(0), M[tr].std(0) + 1e-9
            g = LogisticRegression(max_iter=2000).fit((M[tr] - mu) / sd, y[tr])
            p[te] = g.predict_proba((M[te] - mu) / sd)[:, 1]
        f2 = np.isfinite(p)
        out_rev[nm] = {"all": round(float(roc_auc_score(y[f2], p[f2])), 4)}
        # ОКРЕМО по напрямах: асиметрія має протилежний знак на верху й на низу,
        # тож у спільній купі вони гасять одна одну і AUC валиться до 0.5
        for dn, m3 in (("верх", kind[idx] > 0), ("низ", kind[idx] < 0)):
            mm = np.r_[m3, m3] & f2
            if mm.sum() > 60 and len(np.unique(y[mm])) == 2:
                out_rev[nm][dn] = round(float(roc_auc_score(y[mm], p[mm])), 4)
        log(f"  екстремум проти тіла ноги · {nm}: разом {out_rev[nm]['all']:.3f} · "
            f"верх {out_rev[nm].get('верх')} · низ {out_rev[nm].get('низ')}")

    # --- ланцюг як ціле: середні по всіх секундах шматків ---
    chain = {}
    for nm, m in (("top", tp), ("bot", bt)):
        w = nsec[m].astype(float)
        chain[nm] = {
            "pieces": int(m.sum()), "hours": round(float(w.sum() / 3600), 1),
            "dur_med": int(np.median(dur[m])),
            "lev": {f: round(float(np.average(lev[m, jf], weights=w)), 4)
                    for jf, f in enumerate(F8)},
            "sig": {t2: round(float(np.nanmean(sig[m, jt])), 4)
                    for jt, t2 in enumerate(TARGETS)}}

    out = {"coin": coin, "built": int(time.time()), "n_pairs": len(pairs),
           "chain": chain, "rows": rows, "ctrl": ctrl, "rev": out_rev}
    json.dump(out, open(os.path.join(OUT_DIR, f"{coin}_chains.json"), "w"),
              ensure_ascii=False)
    for r in rows:
        log(f"  {r['group']:9s} {r['name']:28s} верх {r['top']:.4f} низ {r['bot']:.4f}"
            f" · парна різниця {r['paired_med']:+.4f} · q {r['q_paired']:.3f}"
            f"{'  ЗНАЧУЩЕ' if r['sig'] else ''}")
    charts_chains(coin, out, lev, sig, dur, tp, bt, nsec, F8)
    return out


def charts_chains(coin, out, lev, sig, dur, tp, bt, nsec, F8):
    plt = _style()
    for lang in ("uk", "en"):
        uk = lang == "uk"
        sfx = "" if lang == "uk" else "_en"
        fig, axg = plt.subplots(2, 3, figsize=(14, 8.6),
                                gridspec_kw={"width_ratios": [2.1, 1, 1]})
        ax = axg[0]
        # НИЖНІЙ РЯД — головна перевірка: тіло ноги → зона екстремуму
        gs = axg[1, 0].get_gridspec()
        for a in axg[1]:
            a.remove()
        axc = fig.add_subplot(gs[1, :])
        C = out.get("ctrl", [])
        xs = np.arange(len(C)); w = 0.2
        for off, (dn, col, lb) in enumerate(
                ((("top"), C_TOP, "вершини: тіло ноги" if uk else "tops: leg body"),
                 (("bot"), C_BOT, "дно: тіло ноги" if uk else "bottoms: leg body"))):
            axc.bar(xs + (off * 2 - 1.5) * w,
                    [c[dn]["leg"] for c in C], w, color=col, alpha=0.45, label=lb)
            axc.bar(xs + (off * 2 - 0.5) * w,
                    [c[dn]["zone"] for c in C], w, color=col,
                    label=("вершини: зона" if uk else "tops: zone") if off == 0
                    else ("дно: зона" if uk else "bottoms: zone"))
        for i, c in enumerate(C):
            for off, dn in ((0, "top"), (1, "bot")):
                if c[dn]["p"] < 0.05:
                    y = max(c[dn]["leg"], c[dn]["zone"])
                    axc.text(i + (off * 2 - 1.0) * w, y + 0.004, "*",
                             ha="center", color=FG, fontsize=11)
        axc.axhline(0, color=FG, lw=0.8, alpha=0.6)
        axc.set_xticks(xs); axc.set_xticklabels([c["name"] for c in C], fontsize=8)
        axc.set_ylabel("log-відношення" if uk else "log ratio")
        axc.set_title("Головна перевірка: тіло ноги → зона екстремуму "
                      "(* — парний тест p<0.05)" if uk else
                      "Key control: leg body -> extremum zone (* paired p<0.05)",
                      fontsize=10)
        axc.grid(alpha=0.15, color=GRID, axis="y")
        axc.legend(fontsize=8, facecolor="#161b22", edgecolor=GRID, ncol=4)
        # 1) рівні 8 фіч у двох ланцюгах
        x = np.arange(8)
        mt = np.array([np.median(lev[tp, j]) for j in range(8)])
        mb = np.array([np.median(lev[bt, j]) for j in range(8)])
        ax[0].bar(x - 0.2, mt, 0.4, color=C_TOP,
                  label="ланцюг вершин" if uk else "chain of tops")
        ax[0].bar(x + 0.2, mb, 0.4, color=C_BOT,
                  label="ланцюг дна" if uk else "chain of bottoms")
        ax[0].set_xticks(x)
        ax[0].set_xticklabels([f.replace("_", "\n") for f in F8], fontsize=7)
        ax[0].set_ylabel("медіанний рівень log1p" if uk else "median log1p level")
        ax[0].set_title("8 фіч у двох ланцюгах" if uk else
                        "8 features in the two chains", fontsize=10)
        ax[0].grid(alpha=0.15, color=GRID, axis="y")
        ax[0].legend(fontsize=8, facecolor="#161b22", edgecolor=GRID)
        # 2) сигнали моделей
        x2 = np.arange(4)
        st = np.array([np.nanmedian(sig[tp, j]) for j in range(4)])
        sb = np.array([np.nanmedian(sig[bt, j]) for j in range(4)])
        ax[1].bar(x2 - 0.2, st, 0.4, color=C_TOP)
        ax[1].bar(x2 + 0.2, sb, 0.4, color=C_BOT)
        ax[1].set_xticks(x2); ax[1].set_xticklabels(TARGETS, fontsize=7, rotation=20)
        ax[1].set_ylabel("медіанна p" if uk else "median p")
        ax[1].set_title("сигнали 4 моделей" if uk else "the four model signals",
                        fontsize=10)
        ax[1].grid(alpha=0.15, color=GRID, axis="y")
        # 3) тривалість
        b = np.logspace(np.log10(30), np.log10(max(dur.max(), 60)), 26)
        ax[2].hist(dur[tp], bins=b, color=C_TOP, alpha=0.6,
                   label="вершини" if uk else "tops")
        ax[2].hist(dur[bt], bins=b, color=C_BOT, alpha=0.6,
                   label="дно" if uk else "bottoms")
        ax[2].set_xscale("log")
        ax[2].set_xlabel("тривалість шматка, с" if uk else "piece duration, s")
        ax[2].set_title("тривалість" if uk else "duration", fontsize=10)
        ax[2].grid(alpha=0.15, color=GRID)
        ax[2].legend(fontsize=8, facecolor="#161b22", edgecolor=GRID)
        fig.tight_layout()
        p = os.path.join(OUT_DIR, f"{coin}_chains{sfx}.png")
        fig.savefig(p, dpi=110); plt.close(fig)
        log(f"{coin}: {p}")


# ======================= 6. ШТОРМ У ЧОТИРЬОХ СТАНАХ =======================
def stage_storm4(coin, nbin=20):
    """Розподіл сигналу «шторм» у чотирьох станах ринку: зона вершини, зона
    дна, тіло ноги ВГОРУ, тіло ноги ВНИЗ. Стан вікна визначається його
    СЕРЕДИНОЮ. Крім сирих розподілів — те саме після поправки «всередині доби»
    (рівень шторму дрейфує по місяцях сильніше, ніж між станами), і профіль
    уздовж нормованої ноги: чи шторм сидить на кінцях руху, чи рівномірно."""
    from scipy.stats import kruskal, mannwhitneyu
    Z = json.load(open(os.path.join(OUT_DIR, f"{coin}_zones.json")))
    zs = Z["zones"]
    ts, px, lab, _ = window_labels(coin)
    prob, flag, cols, okm = model_answers(coin)
    ts, px, lab = ts[okm], px[okm], lab[okm]
    prob, flag = prob[okm], flag[okm]
    p = prob[:, 0]                      # ймовірність «шторму»
    f = flag[:, 0] > 0                  # прапорець при робочому порозі
    ctr = ts + WIN // 2

    za = np.array([z["t_a"] for z in zs], np.int64)
    zb = np.array([z["t_b"] for z in zs], np.int64)
    ztop = np.array([z["kind"] == "top" for z in zs])
    g = np.zeros(len(ts), np.int8)      # 0 нікуди, 1 верх, 2 низ, 3 нога вгору, 4 нога вниз
    pos = np.full(len(ts), np.nan)      # положення всередині ноги, 0..1
    j = np.searchsorted(za, ctr) - 1
    for i in range(len(ts)):
        k = j[i]
        if 0 <= k < len(zs) and za[k] <= ctr[i] <= zb[k]:
            g[i] = 1 if ztop[k] else 2
        elif 0 <= k < len(zs) - 1 and zb[k] < ctr[i] < za[k + 1]:
            # нога веде В наступний екстремум: у вершину — вгору, у дно — вниз
            g[i] = 3 if ztop[k + 1] else 4
            pos[i] = (ctr[i] - zb[k]) / max(za[k + 1] - zb[k], 1)

    # поправка «всередині доби»: ранг p серед вікон ТІЄЇ Ж доби
    day = (ts - ts[0]) // 86400
    rk = np.zeros(len(ts))
    for d in np.unique(day):
        m = day == d
        v = p[m]
        rk[m] = (v[:, None] > v[None, :]).sum(1) / max(len(v) - 1, 1) if len(v) < 4000 \
            else np.argsort(np.argsort(v)) / max(len(v) - 1, 1)

    NAMES = {1: "зона вершини", 2: "зона дна", 3: "нога вгору", 4: "нога вниз"}
    bins = np.linspace(0, 1, nbin + 1)
    out = {"coin": coin, "built": int(time.time()), "n_win": int(len(ts)),
           "thr": float(np.quantile(p, 1 - f.mean())), "bins": bins.round(3).tolist(),
           "groups": []}
    for k in (1, 2, 3, 4):
        m = g == k
        if m.sum() < 30:
            continue
        v = p[m]
        out["groups"].append({
            "id": int(k), "name": NAMES[k], "n": int(m.sum()),
            "share_time": round(float(m.mean()), 4),
            "mean": round(float(v.mean()), 4), "med": round(float(np.median(v)), 4),
            "q": [round(float(x), 4) for x in np.quantile(v, [.1, .25, .75, .9])],
            "fire": round(float(f[m].mean()), 4),
            "lift_fire": round(float(f[m].mean() / f.mean()), 3),
            "hist": (np.histogram(v, bins=bins)[0] / m.sum()).round(5).tolist(),
            "rank_mean": round(float(rk[m].mean()), 4)})
        s = out["groups"][-1]
        log(f"{coin}: {s['name']:14s} {s['n']:6d} вікон ({s['share_time']*100:5.1f}% часу) · "
            f"p сер {s['mean']:.3f} мед {s['med']:.3f} · горить {s['fire']*100:5.1f}% "
            f"(×{s['lift_fire']}) · ранг у добі {s['rank_mean']:.3f}")

    # тести
    grp = [p[g == k] for k in (1, 2, 3, 4)]
    out["kruskal_p"] = float(kruskal(*grp).pvalue)
    out["pairs"] = []
    for a, b in ((1, 2), (3, 4), (1, 3), (2, 4), (1, 4), (2, 3)):
        u = mannwhitneyu(p[g == a], p[g == b], alternative="two-sided")
        ur = mannwhitneyu(rk[g == a], rk[g == b], alternative="two-sided")
        out["pairs"].append({"a": NAMES[a], "b": NAMES[b],
                             "d_mean": round(float(p[g == a].mean() - p[g == b].mean()), 4),
                             "d_fire": round(float(f[g == a].mean() - f[g == b].mean()), 4),
                             "p": float(u.pvalue), "p_day": float(ur.pvalue)})
        r = out["pairs"][-1]
        log(f"  {r['a']} проти {r['b']}: Δp {r['d_mean']:+.4f} · Δчастота "
            f"{r['d_fire']*100:+.2f} в.п. · p {r['p']:.2e} · усередині доби {r['p_day']:.2e}")

    # ПРАКТИЧНЕ ЧИТАННЯ НАЗАД: що каже спрацювання шторму про стан ринку
    inv = {}
    for nm, m in (("усі вікна", np.ones(len(ts), bool)), ("шторм горить", f),
                  ("шторм мовчить", ~f)):
        d = {NAMES[k]: round(float((g[m] == k).mean()), 4) for k in (1, 2, 3, 4)}
        d["ноги: частка вниз"] = round(float((g[m] == 4).sum() /
                                             max((np.isin(g[m], [3, 4])).sum(), 1)), 4)
        d["у зоні"] = round(float(np.isin(g[m], [1, 2]).mean()), 4)
        inv[nm] = d
        log(f"  {nm}: у зоні {d['у зоні']*100:.1f}% · серед ніг вниз "
            f"{d['ноги: частка вниз']*100:.1f}%")
    out["inverse"] = inv

    # профіль уздовж ноги
    nb2 = 10
    out["leg_profile"] = {}
    for k, nm in ((3, "нога вгору"), (4, "нога вниз")):
        m = (g == k) & np.isfinite(pos)
        b2 = np.clip((pos[m] * nb2).astype(int), 0, nb2 - 1)
        c = np.bincount(b2, minlength=nb2)
        out["leg_profile"][nm] = {
            "x": [round((i + 0.5) / nb2, 2) for i in range(nb2)],
            "p": (np.bincount(b2, weights=p[m], minlength=nb2) /
                  np.maximum(c, 1)).round(4).tolist(),
            "fire": (np.bincount(b2, weights=f[m].astype(float), minlength=nb2) /
                     np.maximum(c, 1)).round(4).tolist(),
            "n": c.astype(int).tolist()}
        log(f"  профіль {nm}: " +
            " ".join(f"{v:.2f}" for v in out["leg_profile"][nm]["p"]))
    json.dump(out, open(os.path.join(OUT_DIR, f"{coin}_storm4.json"), "w"),
              ensure_ascii=False)
    charts_storm4(coin, out)
    return out


def charts_storm4(coin, S):
    plt = _style()
    CG = {"зона вершини": C_TOP, "зона дна": C_BOT,
          "нога вгору": "#238636", "нога вниз": "#8b2c26"}
    EN = {"зона вершини": "top zone", "зона дна": "bottom zone",
          "нога вгору": "up leg", "нога вниз": "down leg"}
    b = np.array(S["bins"]); x = (b[:-1] + b[1:]) / 2
    for lang in ("uk", "en"):
        uk = lang == "uk"
        sfx = "" if uk else "_en"
        fig, ax = plt.subplots(1, 3, figsize=(14, 4.5),
                               gridspec_kw={"width_ratios": [1.5, 1, 1.2]})
        for gr in S["groups"]:
            nm = gr["name"]
            ax[0].plot(x, gr["hist"], color=CG[nm], lw=1.6,
                       label=nm if uk else EN[nm])
        ax[0].axvline(S["thr"], color=FG, ls="--", lw=0.9, alpha=0.7)
        ax[0].set_yscale("log")
        ax[0].set_xlabel("p(шторм) у вікні 210 с" if uk else "p(storm) in a 210 s window")
        ax[0].set_ylabel("частка вікон" if uk else "share of windows")
        ax[0].set_title("Розподіл сигналу «шторм» у чотирьох станах" if uk else
                        "Distribution of the storm signal in four states", fontsize=10)
        ax[0].grid(alpha=0.15, color=GRID)
        ax[0].legend(fontsize=8, facecolor="#161b22", edgecolor=GRID)

        nm = [g["name"] for g in S["groups"]]
        fr = [g["fire"] * 100 for g in S["groups"]]
        ax[1].bar(range(len(nm)), fr, color=[CG[n] for n in nm])
        for i, g in enumerate(S["groups"]):
            ax[1].text(i, fr[i] + 0.4, f"×{g['lift_fire']}", ha="center",
                       fontsize=8, color=FG)
        ax[1].set_xticks(range(len(nm)))
        ax[1].set_xticklabels([n if uk else EN[n] for n in nm], fontsize=7.5,
                              rotation=20)
        ax[1].set_ylabel("% вікон, де горить шторм" if uk else "% storm windows")
        ax[1].set_title("частота спрацювання" if uk else "firing rate", fontsize=10)
        ax[1].grid(alpha=0.15, color=GRID, axis="y")

        for nm2, v in S["leg_profile"].items():
            ax[2].plot(v["x"], v["p"], color=CG[nm2], lw=1.6, marker="o", ms=3.5,
                       label=nm2 if uk else EN[nm2])
        ax[2].set_xlabel("положення в нозі: 0 — старт, 1 — екстремум" if uk else
                         "position in the leg: 0 start, 1 extremum")
        ax[2].set_ylabel("середня p(шторм)" if uk else "mean p(storm)")
        ax[2].set_title("де шторм сидить усередині руху" if uk else
                        "where the storm sits inside the move", fontsize=10)
        ax[2].grid(alpha=0.15, color=GRID)
        ax[2].legend(fontsize=8, facecolor="#161b22", edgecolor=GRID)
        fig.tight_layout()
        pth = os.path.join(OUT_DIR, f"{coin}_storm4{sfx}.png")
        fig.savefig(pth, dpi=110); plt.close(fig)
        log(f"{coin}: {pth}")


# ============ 7. ЩО ВСЕРЕДИНІ ШТОРМІВ: САМІ 1680 ЧИСЕЛ ============
def window_states(ts, zs):
    """Стан вікна за його серединою: 1 вершина · 2 дно · 3 нога вгору ·
    4 нога вниз · 0 поза класифікацією. Плюс положення в нозі 0..1."""
    ctr = ts + WIN // 2
    za = np.array([z["t_a"] for z in zs], np.int64)
    zb = np.array([z["t_b"] for z in zs], np.int64)
    ztop = np.array([z["kind"] == "top" for z in zs])
    g = np.zeros(len(ts), np.int8)
    pos = np.full(len(ts), np.nan)
    j = np.searchsorted(za, ctr) - 1
    for i in range(len(ts)):
        k = j[i]
        if 0 <= k < len(zs) and za[k] <= ctr[i] <= zb[k]:
            g[i] = 1 if ztop[k] else 2
        elif 0 <= k < len(zs) - 1 and zb[k] < ctr[i] < za[k + 1]:
            g[i] = 3 if ztop[k + 1] else 4
            pos[i] = (ctr[i] - zb[k]) / max(za[k + 1] - zb[k], 1)
    return g, pos


def stage_inside(coin, target="storm", use="flag"):
    """Порівняння САМИХ ВХІДНИХ ДАНИХ мережі — матриці 8×210 = 1680 чисел —
    між чотирма станами ринку, але ЛИШЕ серед штормових вікон. Питання: коли
    шторм уже трапився, чи відрізняється він на вершині, на дні, в нозі вгору
    і в нозі вниз — і чи в рівнях, чи у ФОРМІ всередині вікна.

    Дані беруться в тому самому вигляді, в якому їх бачила згортка:
    log1p(x/0.01), стала призначена наперед (prep_fixed у mywclass)."""
    from scipy.stats import mannwhitneyu
    from sklearn.linear_model import LogisticRegression
    from sklearn.ensemble import HistGradientBoostingClassifier
    from sklearn.metrics import roc_auc_score
    Z = json.load(open(os.path.join(OUT_DIR, f"{coin}_zones.json")))
    zs = Z["zones"]
    d = np.load(os.path.join(MYW, f"{coin}_data.npz"), mmap_mode="r")
    ts = np.asarray(d["ts"]).astype(np.int64)
    prob, flag, cols, okm = model_answers(coin)
    ti = list(TARGETS).index(target)
    st = np.where(np.isfinite(flag[:, ti]), flag[:, ti] > 0, False) & okm
    g, _ = window_states(ts, zs)
    sel = np.flatnonzero(st & (g > 0))
    L = np.log1p(np.asarray(d["X"][sel]) / 0.01).astype(np.float32)   # вхід мережі
    gs = g[sel]
    NAMES = {1: "зона вершини", 2: "зона дна", 3: "нога вгору", 4: "нога вниз"}
    log(f"{coin}/{target}: вікон класу в чотирьох станах — " + " · ".join(
        f"{NAMES[k]} {int((gs==k).sum())}" for k in (1, 2, 3, 4)))

    F8 = list(ST.FEATS)
    mean_map = {k: L[gs == k].mean(axis=0) for k in (1, 2, 3, 4)}
    sd_all = L.reshape(len(L), -1).std(axis=0).reshape(8, WIN) + 1e-9
    out = {"coin": coin, "built": int(time.time()), "source": use,
           "target": target, "share": round(float(st.mean()), 4),
           "n": {NAMES[k]: int((gs == k).sum()) for k in (1, 2, 3, 4)},
           "feats": F8,
           "profile": {NAMES[k]: mean_map[k].round(4).tolist() for k in (1, 2, 3, 4)},
           "diff": {}, "levels": [], "shape": [], "clf": {}}

    # карти різниць у частках спільного розкиду (розмір ефекту по клітинці)
    for nm, (a, b) in (("вершина − дно", (1, 2)), ("нога вниз − нога вгору", (4, 3))):
        out["diff"][nm] = ((mean_map[a] - mean_map[b]) / sd_all).round(4).tolist()

    # РІВЕНЬ: середнє по 210 с для кожної фічі
    lev = L.mean(axis=2)
    P1 = np.linspace(-1, 1, WIN); P1 = (P1 - P1.mean()) / np.linalg.norm(P1 - P1.mean())
    trend = L @ P1                        # нахил усередині вікна
    for nm, (a, b) in (("вершина − дно", (1, 2)), ("нога вниз − нога вгору", (4, 3))):
        for kind, V in (("рівень", lev), ("нахил у вікні", trend)):
            rows = []
            for jf, fn in enumerate(F8):
                x, y = V[gs == a, jf], V[gs == b, jf]
                sd = np.sqrt((x.var() + y.var()) / 2) + 1e-12
                rows.append({"feat": fn, "a": round(float(x.mean()), 4),
                             "b": round(float(y.mean()), 4),
                             "d": round(float((x.mean() - y.mean()) / sd), 4),
                             "p": float(mannwhitneyu(x, y).pvalue)})
            ps = np.array([r["p"] for r in rows]); o = np.argsort(ps)
            q = np.empty(len(ps)); prev = 1.0
            for rank, i in enumerate(o[::-1]):
                prev = min(prev, ps[i] * len(ps) / (len(ps) - rank)); q[i] = prev
            for r, v in zip(rows, q):
                r["q"] = float(v); r["sig"] = bool(v < 0.05)
            (out["levels"] if kind == "рівень" else out["shape"]).append(
                {"cmp": nm, "kind": kind, "rows": rows})
            log(f"  {nm} · {kind}: значущих {sum(r['sig'] for r in rows)}/8 — " +
                " ".join(f"{r['feat'][:6]}{'*' if r['sig'] else ''} d={r['d']:+.2f}"
                         for r in rows))

    # ЧИ РОЗДІЛЯЮТЬСЯ ВЗАГАЛІ: класифікатор на всіх 1680 числах
    blk = ((ts[sel] - ts[sel].min()) // (BLOCK_H * 3600) % FOLDS).astype(int)
    rts, rpx, _, _ = price_series(coin)
    ia = np.searchsorted(rts, ts[sel]); ib = np.clip(
        np.searchsorted(rts, ts[sel] + WIN) - 1, 0, len(rpx) - 1)
    PX = np.stack([np.log(rpx[np.clip(ib, 0, len(rpx) - 1)] /
                          np.maximum(rpx[np.clip(ia, 0, len(rpx) - 1)], 1e-9))], axis=1)
    for nm, (a, b) in (("вершина проти дна", (1, 2)),
                       ("нога вниз проти ноги вгору", (4, 3))):
        m = np.isin(gs, [a, b])
        y = (gs[m] == a).astype(np.int8)
        F = L[m].reshape(int(m.sum()), -1)
        res = {}
        for kn, M in (("1680 чисел (вхід мережі)", F),
                      ("8 рівнів", lev[m]),
                      ("8 рівнів + 8 нахилів", np.c_[lev[m], trend[m]]),
                      ("ціна: хід у вікні", PX[m])):
            p = np.full(len(y), np.nan)
            for f in range(FOLDS):
                tr, te = blk[m] != f, blk[m] == f
                if tr.sum() < 50 or te.sum() < 10 or len(np.unique(y[tr])) < 2:
                    continue
                mu, sd = M[tr].mean(0), M[tr].std(0) + 1e-9
                if M.shape[1] > 50:
                    mo = LogisticRegression(max_iter=3000, C=0.02)
                else:
                    mo = HistGradientBoostingClassifier(max_iter=120, random_state=SEED)
                mo.fit((M[tr] - mu) / sd, y[tr])
                p[te] = mo.predict_proba((M[te] - mu) / sd)[:, 1]
            f2 = np.isfinite(p)
            res[kn] = round(float(roc_auc_score(y[f2], p[f2])), 4) if f2.sum() > 50 else None
            log(f"  {nm} · {kn}: AUC {res[kn]}")
        out["clf"][nm] = {"n": int(m.sum()), "pos": int(y.sum()), "auc": res}

    # ПОБІЧНЕ СПОСТЕРЕЖЕННЯ: у середніх профілях видно правильні хвилі. Міряємо
    # період — він спільний для всіх станів, тож на порівняння не впливає, але
    # знати про нього треба (це ритм самих даних, не ринку).
    per = {}
    for jf, fn in enumerate(F8):
        v = mean_map[3][jf] - mean_map[3][jf].mean()
        sp = np.abs(np.fft.rfft(v))
        fr = np.fft.rfftfreq(WIN, 1.0)
        k = np.argmax(sp[2:]) + 2
        per[fn] = round(float(1 / fr[k]), 1)
    out["period_s"] = per
    log(f"{coin}: період хвиль у середньому профілі, с — " +
        " · ".join(f"{k.split('_')[0][:4]} {v}" for k, v in per.items()))

    json.dump(out, open(os.path.join(OUT_DIR, f"{coin}_inside_{target}.json"), "w"),
              ensure_ascii=False)
    charts_inside(coin, out)
    return out


def charts_inside(coin, I):
    plt = _style()
    tg = I.get("target", "storm")
    CG = {"зона вершини": C_TOP, "зона дна": C_BOT,
          "нога вгору": "#238636", "нога вниз": "#8b2c26"}
    EN = {"зона вершини": "top zone", "зона дна": "bottom zone",
          "нога вгору": "up leg", "нога вниз": "down leg"}
    F8 = I["feats"]
    x = np.arange(WIN)
    for lang in ("uk", "en"):
        uk = lang == "uk"
        sfx = "" if uk else "_en"
        # A: 8 фіч × 210 с, по кривій на стан
        fig, ax = plt.subplots(2, 4, figsize=(15, 6.4), sharex=True)
        def sm(v, w=15):
            k = np.ones(w) / w
            return np.convolve(np.pad(v, w // 2, mode="edge"), k, "valid")[:len(v)]
        for jf, fn in enumerate(F8):
            a = ax[jf // 4, jf % 4]
            for nm, prof in I["profile"].items():
                v = np.array(prof[jf])
                a.plot(x, v, color=CG[nm], lw=0.5, alpha=0.22)
                a.plot(x, sm(v), color=CG[nm], lw=1.6,
                       label=(nm if uk else EN[nm]) if jf == 0 else None)
            a.set_title(fn, fontsize=9)
            a.grid(alpha=0.15, color=GRID)
            if jf >= 4:
                a.set_xlabel("секунда вікна" if uk else "second of the window",
                             fontsize=8)
            if jf % 4 == 0:
                a.set_ylabel("log1p(x/0.01)", fontsize=8)
        ax[0, 0].legend(fontsize=7.5, facecolor="#161b22", edgecolor=GRID)
        TN = {"storm": "шторму", "mid": "класу 2", "calm": "штилю",
              "pcaquiet": "тихого ПСА"}
        TE = {"storm": "the storm", "mid": "class 2", "calm": "calm",
              "pcaquiet": "PCA-quiet"}
        fig.suptitle((f"Що всередині {TN.get(tg, tg)}: середні 1680 чисел входу "
                      f"мережі у чотирьох станах ринку") if uk else
                     (f"Inside {TE.get(tg, tg)}: mean of the 1680 network inputs "
                      f"in four market states"), fontsize=11)
        fig.tight_layout(rect=[0, 0, 1, 0.965])
        p = os.path.join(OUT_DIR, f"{coin}_inside_{tg}{sfx}.png")
        fig.savefig(p, dpi=110); plt.close(fig)
        log(f"{coin}: {p}")

        # B: карти різниць у частках розкиду
        fig, ax = plt.subplots(2, 1, figsize=(13, 5.6))
        for i, (nm, M) in enumerate(I["diff"].items()):
            M = np.array(M)
            v = max(abs(M).max(), 1e-6)
            im = ax[i].imshow(M, aspect="auto", cmap="RdBu_r", vmin=-v, vmax=v)
            ax[i].set_yticks(range(8)); ax[i].set_yticklabels(F8, fontsize=7)
            ax[i].set_title(f"{nm} · у частках спільного розкиду" if uk else
                            f"{nm} · in pooled SD units", fontsize=10)
            ax[i].set_xlabel("секунда вікна" if uk else "second of the window",
                             fontsize=8)
            fig.colorbar(im, ax=ax[i], fraction=0.025)
        fig.tight_layout()
        p = os.path.join(OUT_DIR, f"{coin}_inside_{tg}_diff{sfx}.png")
        fig.savefig(p, dpi=110); plt.close(fig)
        log(f"{coin}: {p}")


def stage_inside_sum(coin):
    """Зведення чотирьох класів в одну карту: розмір ефекту по кожній фічі
    в кожному класі, для обох порівнянь. Видно, який клас що саме розрізняє."""
    plt = _style()
    D4 = {}
    for t in TARGETS:
        p = os.path.join(OUT_DIR, f"{coin}_inside_{t}.json")
        if os.path.exists(p):
            D4[t] = json.load(open(p))
    if not D4:
        log("немає жодного класу — спершу inside"); return None
    F8 = D4[list(D4)[0]]["feats"]
    TN = {"storm": "шторм", "mid": "клас 2", "calm": "штиль",
          "pcaquiet": "тихе ПСА"}
    TE = {"storm": "storm", "mid": "class 2", "calm": "calm",
          "pcaquiet": "PCA-quiet"}
    blocks = [("вершина − дно", "рівень"), ("нога вниз − нога вгору", "рівень"),
              ("вершина − дно", "нахил у вікні")]
    S = {"coin": coin, "built": int(time.time()), "feats": F8,
         "classes": list(D4), "maps": {}, "clf": {}, "share": {}}
    for cmp_, kind in blocks:
        M = np.full((len(D4), 8), np.nan); G = np.zeros((len(D4), 8), bool)
        for i, t in enumerate(D4):
            for blk in D4[t]["levels"] + D4[t]["shape"]:
                if blk["cmp"] == cmp_ and blk["kind"] == kind:
                    for j, r in enumerate(blk["rows"]):
                        M[i, j] = r["d"]; G[i, j] = r["sig"]
        S["maps"][f"{cmp_} · {kind}"] = {"d": M.round(3).tolist(),
                                         "sig": G.tolist()}
    for t in D4:
        S["clf"][t] = D4[t]["clf"]
        S["share"][t] = D4[t]["share"]

    for lang in ("uk", "en"):
        uk = lang == "uk"
        sfx = "" if uk else "_en"
        fig, ax = plt.subplots(3, 1, figsize=(12, 9.2))
        for a, (cmp_, kind) in zip(ax, blocks):
            key = f"{cmp_} · {kind}"
            M = np.array(S["maps"][key]["d"]); G = np.array(S["maps"][key]["sig"])
            v = np.nanmax(np.abs(M)) or 1e-6
            im = a.imshow(M, aspect="auto", cmap="RdBu_r", vmin=-v, vmax=v)
            for i in range(M.shape[0]):
                for j in range(M.shape[1]):
                    if np.isfinite(M[i, j]):
                        a.text(j, i, f"{M[i, j]:+.2f}" + ("*" if G[i, j] else ""),
                               ha="center", va="center", fontsize=8,
                               color="#0d1117" if abs(M[i, j]) > v * 0.55 else FG)
            a.set_xticks(range(8))
            a.set_xticklabels([f.replace("_", "\n") for f in F8], fontsize=7.5)
            a.set_yticks(range(len(D4)))
            a.set_yticklabels([(TN if uk else TE).get(t, t) for t in D4], fontsize=9)
            a.set_title(key if uk else key, fontsize=10)
            fig.colorbar(im, ax=a, fraction=0.02)
        fig.suptitle(("Розмір ефекту (d Коена) по кожній фічі у кожному класі · "
                      "* — значуще після BH") if uk else
                     ("Effect size (Cohen d) per feature per class · "
                      "* significant after BH"), fontsize=11)
        fig.tight_layout(rect=[0, 0, 1, 0.965])
        p = os.path.join(OUT_DIR, f"{coin}_inside_sum{sfx}.png")
        fig.savefig(p, dpi=110); plt.close(fig)
        log(f"{coin}: {p}")
    json.dump(S, open(os.path.join(OUT_DIR, f"{coin}_inside_sum.json"), "w"),
              ensure_ascii=False)
    return S


# ======================= 8. ЗВЕДЕННЯ =======================
def stage_tab(coin):
    Z = json.load(open(os.path.join(OUT_DIR, f"{coin}_zones.json")))
    p = os.path.join(OUT_DIR, f"{coin}_ident.json")
    I = json.load(open(p)) if os.path.exists(p) else None
    hp = os.path.join(OUT_DIR, f"{coin}_hooks.json")
    H = json.load(open(hp)) if os.path.exists(hp) else None
    cp = os.path.join(OUT_DIR, f"{coin}_chains.json")
    C = json.load(open(cp)) if os.path.exists(cp) else None
    sp = os.path.join(OUT_DIR, f"{coin}_storm4.json")
    S4 = json.load(open(sp)) if os.path.exists(sp) else None
    IN = {}
    for t in TARGETS:
        ip = os.path.join(OUT_DIR, f"{coin}_inside_{t}.json")
        if os.path.exists(ip):
            IN[t] = {k: v for k, v in json.load(open(ip)).items() if k != "profile"}
    out = {k: Z[k] for k in ("coin", "thr_pct", "band_pct", "days", "n_zones",
                             "n_top", "n_bot", "per_day", "cover_pct", "dur",
                             "leg", "overlaps")}
    out["built"] = int(time.time())
    if I:
        out["ident"] = {k: I[k] for k in ("cols", "L_fit", "L_max", "dur_min",
                                          "sweep", "null", "price", "combo", "dir",
                                          "cells", "base_any", "shares", "n_win",
                                          "folds", "block_h")}
        out["ident"]["profile"] = I["profile"]
    if H:
        out["hooks"] = H
    if C:
        out["chains"] = C
    if S4:
        out["storm4"] = S4
    if IN:
        out["inside"] = IN
    sm = os.path.join(OUT_DIR, f"{coin}_inside_sum.json")
    if os.path.exists(sm):
        out["inside_sum"] = json.load(open(sm))
    json.dump(out, open(os.path.join(OUT_DIR, f"{coin}_tab.json"), "w"),
              ensure_ascii=False)
    log(f"{coin}: зведення для вкладки готове")
    return out


if __name__ == "__main__":
    cmd = sys.argv[1] if len(sys.argv) > 1 else "zones"
    coin = sys.argv[2] if len(sys.argv) > 2 else "BTC"
    if cmd == "zones":
        stage_zones(coin)
    elif cmd == "chart":
        stage_chart(coin)
    elif cmd == "ident":
        stage_ident(coin); stage_tab(coin)
    elif cmd == "hooks":
        stage_hooks(coin); stage_tab(coin)
    elif cmd == "chains":
        stage_chains(coin); stage_tab(coin)
    elif cmd == "storm4":
        stage_storm4(coin); stage_tab(coin)
    elif cmd == "inside":
        tg = sys.argv[3] if len(sys.argv) > 3 else "all"
        for t in (TARGETS if tg == "all" else (tg,)):
            stage_inside(coin, t)
        stage_inside_sum(coin)
        stage_tab(coin)
    elif cmd == "tab":
        stage_tab(coin)
    else:
        print(__doc__)
```

### Маршрути дашборда

`web/dashboard.py`:

```python
@app.route('/pivots')
def pivots(): return render_template('pivots.html')  # ЗОНИ ЛОКАЛЬНИХ ВЕРШИН І ДНА: ZigZag 1% + смуга 0.5% навколо екстремуму; чи ідентифікують зону відповіді 4 моделей /mywclass

@app.route('/api/pivots/<coin>')
def pivots_coin(coin):
    if not re.fullmatch(r'[A-Za-z0-9]{1,10}', coin or ''):
        return jsonify(dict(error='погана монета')), 400
    p = os.path.join(ROOT_DIR, 'media', 'analyst', 'pivots', f'{coin.upper()}_tab.json')
    if not os.path.exists(p):
        return jsonify(dict(error=f'нема зведення по {coin.upper()} — запусти python3 pivot_zones.py ident {coin.upper()}'))
    with open(p) as f:
        return app.response_class(f.read(), mimetype='application/json')
```

### Шаблон вкладки

`web/templates/pivots.html`:

```html
<!doctype html><html lang="uk"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Зони вершин і дна · market_dashboard</title>
<style>
:root{--bg:#0d1117;--fg:#e6edf3;--mut:#8b949e;--grid:#30363d;--grow:#3fb950;--fall:#f85149;--acc:#58a6ff;--warn:#d29922}
body{background:var(--bg);color:var(--fg);font:14px/1.55 system-ui,Segoe UI,Roboto,sans-serif;margin:0;padding:0 18px 70px;max-width:1240px;margin-inline:auto}
a{color:var(--acc)} h1{font-size:23px;margin:16px 0 4px} h2{font-size:15px;margin:28px 0 8px;border-left:3px solid var(--acc);padding-left:9px}
.mut{color:var(--mut)} .small{font-size:12px}
.bar{display:flex;gap:10px;align-items:center;flex-wrap:wrap;margin:10px 0}
table{border-collapse:collapse;font-size:12.5px;margin:6px 0;width:100%}
th,td{border:1px solid var(--grid);padding:3px 8px;text-align:right}
th{background:#161b22} td.l,th.l{text-align:left}
tr:hover td{background:#11161d}
.grow{color:var(--grow)} .fall{color:var(--fall)} .warn{color:var(--warn)} .acc{color:var(--acc)}
.box{border:1px solid var(--grid);border-radius:8px;padding:10px 14px;background:#0f141b;margin:8px 0}
.kpi{display:grid;grid-template-columns:repeat(auto-fill,minmax(170px,1fr));gap:10px;margin:10px 0}
.kpi div{background:#161b22;border:1px solid var(--grid);border-radius:8px;padding:8px 11px}
.kpi b{display:block;font-size:19px;margin-top:2px}
code{background:#161b22;border:1px solid var(--grid);border-radius:4px;padding:1px 5px;font-size:12px}
.gal{display:grid;grid-template-columns:1fr;gap:12px}
.gal img{width:100%;border:1px solid var(--grid);border-radius:8px;background:#0b0e13}
.verdict{border-left:3px solid var(--warn)}
</style></head><body>

<div class="bar"><a href="/">← хаб</a> <a href="/mywclass">/mywclass — ці 4 моделі</a> <a href="/wclass">/wclass</a> <a href="/events">/events</a></div>
<h1>📐 Локальні вершини і дно BTC — і чи видно їх нашим 4 моделям</h1>
<div class="mut" id="sub">завантаження…</div>

<div class="box verdict" id="verdict"></div>

<div class="box small">
  <b>Звідки моделі.</b> Ця лінія нічого не навчає сама — вона перевіряє, чи бачать зони
  чотири детектори станів ринку, навчені в іншій лінії:
  <a href="/mywclass"><b>/mywclass</b></a> (незалежна репліка) і <a href="/wclass">/wclass</a> (оригінал).
  Звідти беруться цілі <code>storm</code> · <code>mid</code> · <code>calm</code> · <code>pcaquiet</code>,
  причому <b>out-of-fold</b> — відповіді моделі, яка цього вікна не бачила.
  <br>
  <b>Щоденники:</b>
  <a href="/media/analyst/pivots/PIVOTS_DIARY.md" download>щоденник цього дослідження (PIVOTS_DIARY.md)</a> ·
  <a href="/media/analyst/wclass/WCLASS_DIARY.md" download>щоденник моделей (WCLASS_DIARY.md)</a>.
  Обидва — виконувані рецепти за <code>DIARY_STANDARD.md</code>: береш свій датасет і отримуєш
  ті самі числа.
</div>

<div class="box small">
  <b>Як побудовано зони.</b> Екстремум береться ZigZag-ом з порогом <b id="p_thr">1%</b>: нова вершина
  (дно) фіксується лише коли ціна відійшла від неї на цей поріг, тож обидві ноги — до попереднього
  екстремуму і до наступного — за побудовою не менші за поріг. Кожен екстремум ще й перевіряється
  явно, тому перший і останній пивот (з незавершеною ногою) відкинуто.
  <b>Зона</b> — це не точка, а найдовший неперервний проміжок навколо екстремуму, поки ціна тримається
  в смузі <b id="p_band">0.5%</b> від нього: для вершини — не нижче, для дна — не вище.
  Ціна береться з РЕАЛЬНИХ спостережень, без інтерполяції (18% секунд у BTC відсутні — інтерполяція
  наробила б фальшивих екстремумів). Зони вершин і дна за побудовою не перетинаються, і це перевірено
  фактично: <b id="p_over">—</b> перетинів.
  Код — <code>pivot_zones.py</code>.
</div>

<h2>① Що знайшлось на всьому датасеті</h2>
<div class="kpi" id="kpi"></div>
<table id="dur"></table>

<h2>② Ціна BTC із зонами</h2>
<div class="mut small"><span class="grow">Зелене — зона вершини</span> (смуга 0.5% вниз від максимуму),
<span class="fall">червоне — зона дна</span> (смуга 0.5% вгору від мінімуму). Шість панелей по ~28 діб,
далі — збільшений відрізок, де зони видно в натуральну ширину.</div>
<div class="gal">
  <img src="/media/analyst/pivots/BTC_zones.png" alt="ціна BTC із зонами вершин і дна">
  <img src="/media/analyst/pivots/BTC_zoom.png" alt="збільшено: зони вершин і дна">
</div>

<h2>③ Чи ідентифікують зону відповіді 4 моделей</h2>
<div class="mut small">Вхід — тільки видача чотирьох моделей лінії <a href="/mywclass">/mywclass</a>
(шторм · клас 2 · штиль · тихе ПСА — чотири продакшен-цілі лінії), взята <b>out-of-fold</b>, тобто моделлю, яка цього вікна
не бачила. Набір відповідей — від одного вікна 210 с до восьми поспіль; вертикальна риска на графіку
позначає межу «влазить у найкоротшу зону» (найкоротша зона <b id="p_dmin">—</b> с, тобто строго за
умовою дозволено рівно <b id="p_lfit">1</b> набір). Оцінка — блоками по 12 год з ембарго 1 год,
5 фолдів. Нуль-контроль — циклічний зсув відповідей: звʼязок зруйновано, автокореляція збережена.</div>
<table id="sweep"></table>

<h2>④ Задачі, нуль і цінові еталони</h2>
<div class="mut small">Два еталони навмисно різні: «положення» — перцентиль ціни в ковзних 24 год
(відповідає на питання «вершина чи дно»), «розмах» — розмах ціни в самому вікні (відповідає на
питання «коли»). Ордербук мусить їх бити, інакше він нічого не додає.</div>
<table id="base"></table>
<div class="gal"><img src="/media/analyst/pivots/BTC_ident.png" alt="AUC за довжиною набору і профіль відповідей навколо екстремуму"></div>

<h2>⑤ Що дає кожен окремий сигнал</h2>
<div class="mut small">Прапорець моделі при її робочому порозі: скільки часу горить і що при цьому
відбувається з ціною.</div>
<table id="cells"></table>

<h2>⑥ Чи додає ордербук ПОНАД ціну</h2>
<div class="mut small">Та сама модель на самому лише розмаху ціни — і вона ж плюс відповіді 4 моделей.
Різниця AUC і є внеском ордербука.</div>
<table id="combo"></table>

<h2>⑦ Де реально є за що зачепитись</h2>
<div class="mut small">Пʼять окремих питань замість одного. Скрізь поруч — ціновий еталон з того ж вікна.</div>
<div class="gal"><img src="/media/analyst/pivots/BTC_hooks.png" alt="форма сигналу всередині зони і правило «шторм → тиша»"></div>

<div class="box small"><b>A · форма всередині зони.</b> Зона обмежена двома ногами ≥1%, тож
сигнал у ній має форму бутерброда: <span id="h_shape">—</span></div>

<div class="mut small" style="margin-top:14px"><b>B · правило «шторм, потім k тихих вікон»</b> —
єдина зачіпка, яку ОДИН набір відповідей дати не може, лише послідовність. Ціновий контроль — те саме
правило, але «шторм» і «тиша» задані розмахом ціни з тими ж частотами спрацювання.</div>
<table id="sand"></table>

<div class="mut small" style="margin-top:14px"><b>C · серед вікон, де шторм уже горить</b> — чи
відрізнити екстремум від звичайного сплеску всередині ноги.</div>
<table id="wstorm"></table>

<div class="mut small" style="margin-top:14px"><b>D · напрям.</b> 89 описів вікна проти 4 моделей
і проти самої ціни.</div>
<table id="dir2"></table>

<div class="mut small" style="margin-top:14px"><b>E · покриття на рівні ЗОНИ, а не вікна</b> —
скільки з 734 екстремумів сигнал «шторм» узагалі торкнувся, і як це залежить від довжини зони.</div>
<table id="cov"></table>

<h2>⑧ Два ланцюги: вершини проти дна</h2>
<div class="mut small">Усі шматки, після яких ціна ПАДАЛА, зшито в один ланцюг; усі, після яких РОСЛА — в
другий. Порівняння по 8 фічах, по сигналах 4 моделей і по тривалості.
<b>Головна пастка тут — епоха історії</b>: рівні фіч їздять по місяцях сильніше, ніж між вершиною і дном.
Тому крім звичайного порівняння 366 проти 362 всюди рахується <b>ПАРНЕ</b>: кожна вершина проти
СУСІДНЬОГО дна (вони чергуються за побудовою зигзага), різниця береться всередині пари, і на всі
порівняння накладено BH-поправку. Друга перевірка ще важливіша: для кожного шматка взято шматок
тієї ж тривалості з <b>СЕРЕДИНИ ноги</b>, яка до нього привела — щоб відділити властивість
розвороту від простої тіні того, куди ціна щойно йшла.</div>
<div class="kpi" id="ch_kpi"></div>
<div class="gal"><img src="/media/analyst/pivots/BTC_chains.png" alt="два ланцюги: фічі, сигнали, тривалість і контроль тілом ноги"></div>
<table id="ch_rows"></table>

<div class="mut small" style="margin-top:14px"><b>Контроль тілом ноги</b> — те саме число всередині
руху, який привів у зону, і в самій зоні. Тут і видно, що є властивістю розвороту, а що лише тінню тренду.</div>
<table id="ch_ctrl"></table>

<div class="mut small" style="margin-top:14px"><b>Чи відрізняє це екстремум від тіла ноги</b> —
збалансована задача, 5 блокових фолдів. Асиметрія має протилежний знак на верху й на низу, тож у
спільній купі гаситься; рядок «× напрям» — та сама асиметрія, помножена на знак екстремуму.</div>
<table id="ch_rev"></table>

<h2>⑨ Розподіл «шторму» у чотирьох станах</h2>
<div class="mut small">Кожне вікно 210 с віднесено за своєю СЕРЕДИНОЮ до одного з чотирьох станів:
зона вершини, зона дна, тіло ноги вгору, тіло ноги вниз. Розподіл p(шторм) украй двогорбий (медіана
майже нуль у всіх станах, маса на краях), тому шкала часток логарифмічна, а порівнювати треба
частоту спрацювання, а не медіану. Поправка «всередині доби» — ранг вікна серед вікон тієї ж доби:
рівень шторму дрейфує по місяцях сильніше, ніж між станами.</div>
<div class="gal"><img src="/media/analyst/pivots/BTC_storm4.png" alt="розподіл шторму в чотирьох станах ринку"></div>
<table id="s4"></table>
<div class="mut small" style="margin-top:12px">Попарні тести (Манна-Вітні на сирій p і на ранзі всередині доби):</div>
<table id="s4p"></table>
<div class="mut small" style="margin-top:12px">Читання назад: що спрацювання шторму каже про стан ринку.</div>
<table id="s4i"></table>

<h2>⑩ Що всередині класів: самі 1680 чисел</h2>
<div class="mut small">Досі порівнювалась ВІДПОВІДЬ мережі. Тут порівнюється те, що вона бачила: матриця
8×210 у тому самому вигляді, в якому йде на вхід — <code>log1p(x/0.01)</code>, стала призначена наперед.
Для КОЖНОЇ з чотирьох цілей лінії (<code>storm</code> · <code>mid</code> · <code>calm</code> ·
<code>pcaquiet</code> — рівно ті, що тренує оригінал) беруться лише її вікна, і питання таке: коли клас уже спрацював, чи
відрізняється він на вершині, на дні, в нозі вгору і в нозі вниз — і чи в РІВНЯХ, чи у ФОРМІ всередині
вікна. Розмір ефекту — d Коена, BH-поправка на 8 фіч у кожному блоці.</div>
<div class="gal"><img src="/media/analyst/pivots/BTC_inside_sum.png" alt="карта розмірів ефекту: 4 класи × 8 фіч"></div>
<table id="in_kpi2"></table>

<div class="mut small" style="margin-top:14px"><b>Чи розділяються стани</b> — 5 блокових фолдів,
всередині кожного класу окремо. Стовпець «ціна» — хід ціни в тому самому вікні.</div>
<table id="in_clf"></table>

<div class="mut small" style="margin-top:14px">Повні профілі по кожному класу: тонка лінія — сире
посекундне середнє, товста — згладжене на 15 с. Далі карта різниць 8×210 у частках спільного розкиду.</div>
<div class="bar" id="in_tabs"></div>
<div class="gal">
  <img id="in_img1" src="/media/analyst/pivots/BTC_inside_storm.png" alt="середні 1680 чисел у чотирьох станах">
  <img id="in_img2" src="/media/analyst/pivots/BTC_inside_storm_diff.png" alt="карта різниць 8×210">
</div>
<div class="box small" id="in_per"></div>

<script>
const F=(v,d)=>(v===null||v===undefined||!isFinite(v))?'—':Number(v).toFixed(d===undefined?2:d);
const PC=(v,d)=>(v===null||v===undefined||!isFinite(v))?'—':(v*100).toFixed(d===undefined?1:d)+'%';
const HMS=s=>{if(s===null||s===undefined)return '—';s=Math.round(s);
  if(s<90)return s+' с';if(s<5400)return (s/60).toFixed(0)+' хв';return (s/3600).toFixed(1)+' год';};
const TT={top:'у зоні вершини',bot:'у зоні дна',any:'у будь-якій зоні',
          peak:'момент екстремуму (вікно з ним)',near:'екстремум ±1 вікно (±3.5 хв)',
          ahead:'екстремум у наступну годину'};
const CN={'storm':'шторм','mid':'клас 2','calm':'штиль','pcaquiet':'тихе ПСА'};
let D=null;

function get(o,k){return o?(o[k]!==undefined?o[k]:o[String(k)]):undefined;}

function kpis(){
  document.getElementById('p_thr').textContent=F(D.thr_pct,1)+'%';
  document.getElementById('p_band').textContent=F(D.band_pct,1)+'%';
  document.getElementById('p_over').textContent=D.overlaps;
  const d=D.dur||{};
  document.getElementById('kpi').innerHTML=
    `<div>зон усього<b>${D.n_zones||'—'}</b><span class="mut small">${D.days||'—'} діб · ${F(D.per_day,2)}/добу</span></div>`+
    `<div>вершин / дна<b class="grow">${D.n_top}</b><span class="mut small">дна <b class="fall">${D.n_bot}</b> — рівно порівну за побудовою</span></div>`+
    `<div>часу в зонах<b>${F(D.cover_pct,1)}%</b><span class="mut small">решта 85% — тіло ноги</span></div>`+
    `<div>медіанна зона<b>${HMS(d.med)}</b><span class="mut small">від ${HMS(d.min)} до ${HMS(d.max)}</span></div>`+
    `<div>медіанна нога<b>${F((D.leg||{}).med,2)}%</b><span class="mut small">мінімум ${F((D.leg||{}).min,2)}% — поріг тримається</span></div>`+
    `<div>вікон 210 с<b>${(D.ident||{}).n_win||'—'}</b><span class="mut small">сітка спільна з /mywclass</span></div>`;
  let h='<tr><th class="l">тривалість зони</th><th>мін</th><th>10%</th><th>медіана</th><th>90%</th><th>макс</th><th>середня</th></tr>'+
        `<tr><td class="l">усі ${D.n_zones} зон</td><td>${HMS(d.min)}</td><td>${HMS(d.q10)}</td>`+
        `<td>${HMS(d.med)}</td><td>${HMS(d.q90)}</td><td>${HMS(d.max)}</td><td>${HMS(d.mean)}</td></tr>`;
  document.getElementById('dur').innerHTML=h;
}

function sweep(){
  const I=D.ident; if(!I) return;
  document.getElementById('p_dmin').textContent=I.dur_min;
  document.getElementById('p_lfit').textContent=I.L_fit;
  const Ls=Object.keys(I.sweep.any).map(Number).sort((a,b)=>a-b);
  let h='<tr><th class="l">задача</th><th class="l">частка вікон</th>';
  for(const L of Ls) h+=`<th>L=${L}</th>`;
  h+='<th>нуль</th><th class="l">найкращий набір</th></tr>';
  for(const t of ['peak','near','ahead','top','bot','any']){
    const S=I.sweep[t]; if(!S) continue;
    let best=null,bl=null;
    let row=`<tr><td class="l">${TT[t]}</td><td class="l mut">${PC((I.shares||{})[t],2)}</td>`;
    for(const L of Ls){
      const r=get(S,L)||{};
      let v=null,w=null;
      for(const k of ['prob','gbm','flag','center']) if(r[k]&&(v===null||r[k].auc>v)){v=r[k].auc;w=k;}
      if(v!==null&&(best===null||v>best)){best=v;bl='L='+L+' · '+w;}
      row+=`<td class="${v>0.6?'warn':''}">${F(v,3)}</td>`;
    }
    const nl=I.null[t];
    row+=`<td class="mut">${F(nl?nl.auc:null,3)}</td><td class="l mut">${bl||'—'} → ${F(best,3)}</td></tr>`;
    h+=row;
  }
  document.getElementById('sweep').innerHTML=h;
}

function base(){
  const I=D.ident; if(!I) return;
  let h='<tr><th class="l">задача</th><th>4 моделі (кращий набір)</th><th>нуль</th>'+
        '<th>ціна: положення</th><th>ціна: розмах</th><th class="l">висновок</th></tr>';
  for(const t of ['peak','near','ahead','top','bot','any']){
    const S=I.sweep[t]; if(!S) continue;
    let best=0;
    for(const L of Object.keys(S)){const r=S[L];
      for(const k of ['prob','gbm','flag','center']) if(r[k]&&r[k].auc>best) best=r[k].auc;}
    const p=I.price[t]||{}, nl=I.null[t]||{};
    const pv=(p.vol||{}).auc, pp=(p.pos||{}).auc;
    let note;
    if(best<0.55) note='<span class="mut">на рівні нуля — зона моделям не видна</span>';
    else if(pv&&pv>best) note='<span class="fall">простий розмах ціни бʼє ордербук</span>';
    else note='<span class="grow">ордербук попереду цінових еталонів</span>';
    h+=`<tr><td class="l">${TT[t]}</td><td class="${best>0.6?'warn':''}">${F(best,3)}</td>`+
       `<td class="mut">${F(nl.auc,3)}</td><td>${F(pp,3)}</td><td>${F(pv,3)}</td>`+
       `<td class="l">${note}</td></tr>`;
  }
  const dr=I.dir;
  h+=`<tr><td class="l">вершина проти дна (серед зонних вікон)</td><td class="fall">${F(dr?dr.auc:null,3)}</td>`+
     `<td class="mut">0.5 за побудовою</td><td colspan="2" class="mut">напрям з ордербука не читається</td>`+
     `<td class="l mut">те саме, що в усій попередній роботі проєкту</td></tr>`;
  document.getElementById('base').innerHTML=h;
}

function cells(){
  const I=D.ident; if(!I) return;
  let h='<tr><th class="l">сигнал</th><th>горить</th><th>шанс бути в зоні</th><th>× до бази</th>'+
        '<th>шанс, що це момент екстремуму</th><th>× до бази</th><th>ловить екстремумів</th></tr>';
  for(const c of Object.keys(I.cells)){
    const v=I.cells[c], nm=CN[c.split(':')[0]]||c;
    h+=`<tr><td class="l">${nm} <span class="mut small">${c.split(':')[1]}</span></td>`+
       `<td>${PC(v.share)}</td><td>${PC(v.p_any,2)}</td>`+
       `<td class="${v.lift_any>1.1?'warn':'mut'}">×${F(v.lift_any)}</td>`+
       `<td>${PC(v.p_peak,2)}</td><td class="${v.lift_peak>1.5?'warn':'mut'}">×${F(v.lift_peak)}</td>`+
       `<td>${PC(v.rec_peak,0)}</td></tr>`;
  }
  h+=`<tr><td class="l mut">база (будь-яке вікно)</td><td class="mut">100%</td>`+
     `<td class="mut">${PC(I.base_any,2)}</td><td class="mut">×1.00</td>`+
     `<td class="mut">${PC((I.shares||{}).peak,2)}</td><td class="mut">×1.00</td><td class="mut">100%</td></tr>`;
  document.getElementById('cells').innerHTML=h;
}

function combo(){
  const I=D.ident; if(!I||!I.combo) return;
  let h='<tr><th class="l">задача</th><th class="l">набір</th><th>сама ціна (розмах)</th>'+
        '<th>ціна + 4 моделі</th><th>внесок ордербука</th></tr>';
  for(const t of ['peak','near','ahead','any']){
    const C=I.combo[t]; if(!C) continue;
    for(const L of Object.keys(C).map(Number).sort((a,b)=>a-b)){
      const r=get(C,L); if(!r||!r.both||!r.price) continue;
      h+=`<tr><td class="l">${TT[t]}</td><td class="l mut">L=${L}</td>`+
         `<td>${F(r.price.auc,4)}</td><td>${F(r.both.auc,4)}</td>`+
         `<td class="${r.d_auc>0.02?'grow':'mut'}">${r.d_auc>=0?'+':''}${F(r.d_auc,4)}</td></tr>`;
    }
  }
  document.getElementById('combo').innerHTML=h;
}

function verdict(){
  const I=D.ident||{}, c=(I.cells||{});
  const st=c['storm:cnn']||c[Object.keys(c)[0]]||{};
  let bp=0,ba=0;
  for(const t of ['peak','ahead']){const S=(I.sweep||{})[t]||{};
    for(const L of Object.keys(S)) for(const k of ['prob','gbm','flag','center'])
      if(S[L][k]){ if(t==='peak'&&S[L][k].auc>bp) bp=S[L][k].auc;
                   if(t==='ahead'&&S[L][k].auc>ba) ba=S[L][k].auc; }}
  const pv=((I.price||{}).peak||{}).vol||{};
  document.getElementById('sub').textContent=
    `${D.coin} · ${D.days} діб · ZigZag ${F(D.thr_pct,1)}% + смуга ${F(D.band_pct,1)}% · `+
    `${D.n_zones} зон · перерахунок ${new Date((D.built||0)*1000).toLocaleString('uk')}`;
  document.getElementById('verdict').innerHTML=
    `<b>Зону — ні, момент — так, напрям — ні.</b> Саму зону (15% часу) відповіді 4 моделей не бачать: `+
    `AUC ${F(0.53,2)} при нулі ${F(((I.null||{}).any||{}).auc,2)} — зона живе довго й більшу її частину `+
    `ринок просто спить. Зате <b>момент екстремуму</b> моделі позначають різко: AUC `+
    `<b class="warn">${F(bp,3)}</b>, а сам сигнал «шторм» горить ${PC(st.share)} часу й накриває `+
    `<b>${PC(st.rec_peak,0)}</b> усіх ${D.n_zones} екстремумів (×${F(st.lift_peak)} до бази). `+
    `Є й прогнозна частина: «екстремум у наступну годину» — AUC <b>${F(ba,3)}</b>. `+
    `АЛЕ чесно: простий розмах ціни в тому ж вікні дає на моменті ${F(pv.auc,3)}, тобто <b class="fall">бʼє `+
    `ордербук</b>, а разом вони додають лише ${F(((I.combo||{}).peak||{})['1']?I.combo.peak['1'].d_auc:null,4)} AUC. `+
    `І головне: вершина від дна не відрізняється взагалі (AUC ${F((I.dir||{}).auc,3)}) — `+
    `ордербук каже «зараз злам», але не каже, в який бік.`;
}

function hooks(){
  const H=D.hooks; if(!H) return;
  const c=H.inside.curves.storm, b=H.inside.base[0];
  document.getElementById('h_shape').innerHTML=
    `на вході в зону p(шторм) = <b class="fall">${F(c[0],2)}</b>, у середині падає до `+
    `<b class="acc">${F(Math.min.apply(null,c),2)}</b>, на виході знову `+
    `<b class="fall">${F(c[c.length-1],2)}</b> — при базі ${F(b,2)}. Тобто ордербук позначає не зону, `+
    `а її КРАЇ: удар, яким ціна прийшла, і удар, яким вона пішла.`;

  let h='<tr><th class="l">правило</th><th>горить</th><th>окремих подій</th><th>шанс бути в зоні</th>'+
        '<th>× до бази</th><th>ловить зон</th></tr>';
  for(const s of (H.sandwich||[]))
    h+=`<tr><td class="l">шторм + ${s.k} тихих вікон</td><td>${PC(s.share)}</td><td>${s.runs}</td>`+
       `<td>${PC(s.p_zone,1)}</td><td class="${s.lift>1.5?'grow':'warn'}">×${F(s.lift)}</td>`+
       `<td class="mut">${PC(s.rec,1)}</td></tr>`;
  for(const s of (H.sandwich_px||[]))
    h+=`<tr><td class="l mut">ціновий контроль, k=${s.k}</td><td class="mut">${PC(s.share)}</td>`+
       `<td class="mut">—</td><td class="mut">${PC(s.p_zone,1)}</td>`+
       `<td class="mut">×${F(s.lift)}</td><td class="mut">—</td></tr>`;
  const co=H.calm_only;
  h+=`<tr><td class="l mut">контроль: просто штиль без шторму перед ним</td><td class="mut">${PC(co.share)}</td>`+
     `<td class="mut">—</td><td class="mut">${PC(co.p_zone,1)}</td><td class="fall">×${F(co.lift)}</td>`+
     `<td class="mut">—</td></tr>`;
  document.getElementById('sand').innerHTML=h;

  const W=H.within_storm;
  h='<tr><th class="l">вхід моделі</th><th>AUC</th><th>PR-AUC</th><th>точність верхньої пʼятини</th><th>× до бази</th></tr>';
  for(const k of Object.keys(W.sets||{})){const v=W.sets[k];
    h+=`<tr><td class="l">${k}</td><td class="${v.auc>0.7?'warn':''}">${F(v.auc,3)}</td>`+
       `<td>${F(v.ap,3)}</td><td>${PC(v.top20_prec,1)}</td>`+
       `<td class="${v.top20_lift>2?'grow':'mut'}">×${F(v.top20_lift)}</td></tr>`;}
  h+=`<tr><td class="l mut">база: ${W.n} штормових вікон, з них екстремумів</td>`+
     `<td class="mut">0.5</td><td class="mut">${F(W.base,3)}</td>`+
     `<td class="mut">${PC(W.base,1)}</td><td class="mut">×1.00</td></tr>`;
  document.getElementById('wstorm').innerHTML=h;

  h='<tr><th class="l">на якій підмножині</th>';
  const kk=Object.keys(H.direction.peak||{});
  for(const k of kk) h+=`<th>${k}</th>`;
  h+='</tr>';
  const NMS={peak:'вікно з екстремумом',near:'екстремум ±1 вікно',zone:'усі зонні вікна'};
  for(const t of ['peak','near','zone']){
    const R=H.direction[t]; if(!R) continue;
    h+=`<tr><td class="l">${NMS[t]}</td>`;
    for(const k of kk) h+=`<td class="${R[k]&&R[k].auc>0.9?'fall':''}">${F(R[k]?R[k].auc:null,3)}</td>`;
    h+='</tr>';
  }
  document.getElementById('dir2').innerHTML=h;

  h='<tr><th class="l">допуск навколо екстремуму</th><th>торкнувся зон</th><th>випадково було б</th><th>×</th></tr>';
  for(const v of (H.zone_cov||[]))
    h+=`<tr><td class="l">±${v.tol} вікон (±${(v.tol*3.5).toFixed(1)} хв)</td><td>${PC(v.cov,0)}</td>`+
       `<td class="mut">${PC(v.exp,0)}</td><td class="${v.lift>1.3?'grow':'mut'}">×${F(v.lift)}</td></tr>`;
  for(const v of (H.by_dur||[]))
    h+=`<tr><td class="l">±1 вікно, зони терцилю ${v.terc} <span class="mut">(медіана ${HMS(v.dur_med)}, ${v.n} шт)</span></td>`+
       `<td class="${v.cov>0.7?'grow':''}">${PC(v.cov,0)}</td><td class="mut">${PC(v.exp,0)}</td>`+
       `<td class="${v.lift>1.3?'grow':(v.lift<1?'fall':'mut')}">×${F(v.lift)}</td></tr>`;
  document.getElementById('cov').innerHTML=h;
}

function chains(){
  const C=D.chains; if(!C) return;
  const t=C.chain.top, b=C.chain.bot;
  document.getElementById('ch_kpi').innerHTML=
    `<div>ланцюг вершин<b class="grow">${t.pieces} шматків</b><span class="mut small">${t.hours} год, медіана ${HMS(t.dur_med)}</span></div>`+
    `<div>ланцюг дна<b class="fall">${b.pieces} шматків</b><span class="mut small">${b.hours} год, медіана ${HMS(b.dur_med)}</span></div>`+
    `<div>пар «сусідні верх+низ»<b>${C.n_pairs}</b><span class="mut small">парний тест знімає епоху</span></div>`+
    `<div>значущих після BH<b>${(C.rows||[]).filter(r=>r.sig).length} з ${(C.rows||[]).length}</b><span class="mut small">q &lt; 0.05</span></div>`;

  let h='<tr><th class="l">що порівнюємо</th><th>ланцюг вершин</th><th>ланцюг дна</th>'+
        '<th>парна різниця</th><th>q (BH)</th><th class="l">вердикт</th></tr>';
  let g='';
  for(const r of (C.rows||[])){
    if(r.group!==g){g=r.group;
      h+=`<tr><td class="l mut" colspan="6"><b>${g}</b></td></tr>`;}
    h+=`<tr><td class="l">${r.name}</td><td>${F(r.top,4)}</td><td>${F(r.bot,4)}</td>`+
       `<td class="${r.sig?'warn':''}">${r.paired_med>=0?'+':''}${F(r.paired_med,4)}</td>`+
       `<td class="${r.sig?'warn':'mut'}">${r.q_paired<0.001?'&lt;0.001':F(r.q_paired,3)}</td>`+
       `<td class="l ${r.sig?'':'mut'}">${r.sig?'різниця є':'різниці немає'}</td></tr>`;
  }
  document.getElementById('ch_rows').innerHTML=h;

  h='<tr><th class="l">асиметрія книги</th><th>вершини: тіло ноги</th><th>вершини: зона</th><th>Δ</th><th>p</th>'+
    '<th>дно: тіло ноги</th><th>дно: зона</th><th>Δ</th><th>p</th><th class="l">що це</th></tr>';
  for(const r of (C.ctrl||[])){
    const a=r.top||{}, z=r.bot||{};
    const both=(a.p<0.05&&z.p<0.05&&a.diff*z.diff<0);
    h+=`<tr><td class="l">${r.name}</td>`+
       `<td class="mut">${F(a.leg,3)}</td><td>${F(a.zone,3)}</td>`+
       `<td class="${a.p<0.05?'warn':'mut'}">${a.diff>=0?'+':''}${F(a.diff,3)}</td>`+
       `<td class="${a.p<0.05?'warn':'mut'}">${a.p<0.001?'&lt;0.001':F(a.p,3)}</td>`+
       `<td class="mut">${F(z.leg,3)}</td><td>${F(z.zone,3)}</td>`+
       `<td class="${z.p<0.05?'warn':'mut'}">${z.diff>=0?'+':''}${F(z.diff,3)}</td>`+
       `<td class="${z.p<0.05?'warn':'mut'}">${z.p<0.001?'&lt;0.001':F(z.p,3)}</td>`+
       `<td class="l ${both?'grow':'mut'}">${both?'властивість РОЗВОРОТУ: у зоні зсув у протилежні боки':'тінь тренду — у зоні те саме, що в нозі'}</td></tr>`;
  }
  document.getElementById('ch_ctrl').innerHTML=h;

  h='<tr><th class="l">вхід моделі</th><th>разом</th><th>лише вершини</th><th>лише дно</th></tr>';
  for(const k of Object.keys(C.rev||{})){const v=C.rev[k];
    h+=`<tr><td class="l">${k}</td><td class="${v.all>0.7?'warn':(v.all<0.55?'mut':'')}">${F(v.all,3)}</td>`+
       `<td>${F(v['верх'],3)}</td><td>${F(v['низ'],3)}</td></tr>`;}
  document.getElementById('ch_rev').innerHTML=h;
}

function storm4(){
  const S=D.storm4; if(!S) return;
  let h='<tr><th class="l">стан ринку</th><th>вікон</th><th>частка часу</th><th>горить шторм</th>'+
        '<th>× до бази</th><th>середня p</th><th>90-й перцентиль p</th><th>ранг у добі</th></tr>';
  for(const g of S.groups)
    h+=`<tr><td class="l">${g.name}</td><td>${g.n}</td><td>${PC(g.share_time)}</td>`+
       `<td class="${g.lift_fire>1.2?'warn':''}">${PC(g.fire)}</td>`+
       `<td class="${g.lift_fire>1.2?'warn':(g.lift_fire<0.95?'mut':'')}">×${F(g.lift_fire)}</td>`+
       `<td>${F(g.mean,3)}</td><td>${F(g.q[3],3)}</td><td>${F(g.rank_mean,3)}</td></tr>`;
  document.getElementById('s4').innerHTML=h;

  h='<tr><th class="l">пара</th><th>Δ середньої p</th><th>Δ частоти, в.п.</th><th>p</th>'+
    '<th>p усередині доби</th><th class="l">вердикт</th></tr>';
  for(const r of S.pairs){
    const ok=r.p_day<0.05;
    h+=`<tr><td class="l">${r.a} проти ${r.b}</td><td>${r.d_mean>=0?'+':''}${F(r.d_mean,4)}</td>`+
       `<td>${r.d_fire>=0?'+':''}${F(r.d_fire*100,2)}</td>`+
       `<td class="${r.p<0.05?'warn':'mut'}">${r.p<1e-4?r.p.toExponential(1):F(r.p,4)}</td>`+
       `<td class="${ok?'warn':'mut'}">${r.p_day<1e-4?r.p_day.toExponential(1):F(r.p_day,4)}</td>`+
       `<td class="l ${ok?'':'mut'}">${ok?'різниця переживає поправку на добу':'різниці немає'}</td></tr>`;
  }
  document.getElementById('s4p').innerHTML=h;

  h='<tr><th class="l">коли…</th><th>у зоні екстремуму</th><th>зона вершини</th><th>зона дна</th>'+
    '<th>нога вгору</th><th>нога вниз</th><th>серед ніг — частка вниз</th></tr>';
  for(const k of Object.keys(S.inverse||{})){const v=S.inverse[k];
    h+=`<tr><td class="l">${k}</td><td class="${k==='шторм горить'?'warn':'mut'}">${PC(v['у зоні'])}</td>`+
       `<td>${PC(v['зона вершини'])}</td><td>${PC(v['зона дна'])}</td>`+
       `<td>${PC(v['нога вгору'])}</td><td>${PC(v['нога вниз'])}</td>`+
       `<td class="${k==='шторм горить'?'warn':'mut'}">${PC(v['ноги: частка вниз'])}</td></tr>`;}
  document.getElementById('s4i').innerHTML=h;
}

function inside(){
  const S=D.inside_sum, I=D.inside; if(!S||!I) return;
  const TN={storm:'шторм',mid:'клас 2',calm:'штиль',pcaquiet:'тихе ПСА'};
  const ST={1:'зона вершини',2:'зона дна',3:'нога вгору',4:'нога вниз'};

  let h='<tr><th class="l">клас</th><th>частка часу</th><th>зона вершини</th><th>зона дна</th>'+
        '<th>нога вгору</th><th>нога вниз</th><th class="l">що саме розрізняє (значуще, |d| найбільші)</th></tr>';
  for(const t of S.classes){
    const n=I[t].n||{};
    const top=[];
    for(const blk of (I[t].levels||[]).concat(I[t].shape||[])){
      const rs=blk.rows.filter(r=>r.sig).sort((x,y)=>Math.abs(y.d)-Math.abs(x.d)).slice(0,2);
      if(rs.length) top.push(`<span class="mut">${blk.cmp==='вершина − дно'?'в/д':'ноги'}·${blk.kind==='рівень'?'рів':'нах'}:</span> `+
        rs.map(r=>`${r.feat} ${r.d>=0?'+':''}${F(r.d,2)}`).join(', '));
    }
    h+=`<tr><td class="l"><b>${TN[t]||t}</b></td><td>${PC(S.share[t])}</td>`+
       `<td>${n['зона вершини']||'—'}</td><td>${n['зона дна']||'—'}</td>`+
       `<td>${n['нога вгору']||'—'}</td><td>${n['нога вниз']||'—'}</td>`+
       `<td class="l small">${top.join('<br>')||'<span class="mut">нічого значущого</span>'}</td></tr>`;
  }
  document.getElementById('in_kpi2').innerHTML=h;

  const tasks=Object.keys(I[S.classes[0]].clf||{});
  const sets=tasks.length?Object.keys(I[S.classes[0]].clf[tasks[0]].auc):[];
  h='<tr><th class="l">клас</th><th class="l">задача</th><th>вікон</th>';
  for(const q of sets) h+=`<th>${q}</th>`;
  h+='</tr>';
  for(const t of S.classes) for(const tk of tasks){
    const c=I[t].clf[tk]; let best=0;
    for(const q of sets) if(c.auc[q]>best) best=c.auc[q];
    h+=`<tr><td class="l">${TN[t]||t}</td><td class="l mut">${tk}</td><td class="mut">${c.n}</td>`;
    for(const q of sets){
      const v=c.auc[q], win=(v===best);
      h+=`<td class="${win?'warn':(v<0.51?'mut':'')}">${F(v,3)}</td>`;}
    h+='</tr>';
  }
  document.getElementById('in_clf').innerHTML=h;

  const tabs=document.getElementById('in_tabs');
  tabs.innerHTML=S.classes.map(t=>`<a href="#" data-t="${t}">${TN[t]||t}</a>`).join(' · ');
  tabs.querySelectorAll('a').forEach(a=>a.onclick=e=>{e.preventDefault();
    document.getElementById('in_img1').src=`/media/analyst/pivots/BTC_inside_${a.dataset.t}.png`;
    document.getElementById('in_img2').src=`/media/analyst/pivots/BTC_inside_${a.dataset.t}_diff.png`;});

  const p=(I.storm||{}).period_s||{};
  document.getElementById('in_per').innerHTML=
    `<b>Побічне спостереження.</b> У середніх профілях видно правильні хвилі з періодом `+
    Object.keys(p).map(k=>`${k} ${p[k]} с`).join(' · ')+
    `. Це ритм самого кадру ордербука, а не ринку: він однаковий у всіх станах, тож на порівняння `+
    `не впливає — але будь-яка модель на формі всередині вікна частину ємності витратить саме на нього.`;
}

fetch('/api/pivots/BTC').then(r=>r.json()).then(j=>{
  if(j.error){document.getElementById('sub').textContent=j.error;return;}
  D=j; verdict(); kpis(); sweep(); base(); cells(); combo(); hooks(); chains(); storm4(); inside();
}).catch(e=>{document.getElementById('sub').textContent='помилка завантаження: '+e;});
</script>
</body></html>
```

### Запис у сторожа вкладок

`test_pages.js`:

```javascript
{file: 'pivots.html', min_draw: 0,
   routes: [['/api/pivots/', 'media/analyst/pivots/{C}_tab.json']]},
```
