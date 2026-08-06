# Щоденник дослідження /mywclass — чи відтворюється чужий рецепт незалежним конвеєром

**Що це.** Виконуваний рецепт. Читаєш цей файл, береш свій датасет — отримуєш ті самі
стани ринку, ті самі детектори і той самий висновок про те, чого в даних немає. Жодного
захардкодженого списку вікон: клас визначається **процедурою**, і процедура перерахує
його на будь-яких даних — іншій монеті, іншому періоді, більшій історії.

Числа в тексті — це те, що вийшло **на нашому зрізі** (BTC, 171.0 доби,
54,764 вікон по 210 с). Вони наведені як орієнтир «схоже чи ні», а не як
константи. Твій прогін дасть свої.

**Чим ця лінія особлива.** Вона не досліджує ринок — вона перевіряє **заяву іншого
щоденника**. Лінія `/wclass` стверджує, що її рецепт виконуваний. Перевірити це можна
рівно одним способом: написати конвеєр наново **за текстом**, змінити все, що довільне
(сид, блоковий спліт, ініціалізацію мережі, розкладку фолдів), і подивитись, чи випадуть
ті самі стани. Якщо випадуть — структура в даних. Якщо ні — вона була в коді.

Що читати перед цим файлом:

- **`WCLASS_DIARY.md`** — щоденник оригінальної лінії (те, що тут відтворюється);
- вкладки **`/wclass`** (оригінал) і **`/mywclass`** (ця репліка, поточні числа);
- **`DIARY_STANDARD.md`** — стандарт, за яким написано обидва щоденники.

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
Мінімум для повторення — **≥60 діб** посекундних даних однієї монети. Менше — класи
знайдуться (вони знаходяться навіть у чистому шумі, див. пастки), але перевірки на їхнє
існування втратять сенс: стійкість міряється двома повними перебудовами на непересічних
половинах історії, і на 20 добах кожна половина буде однією ринковою епохою.

**Правило, яке визначає всю конструкцію** (успадковане з `/wclass` і не підлягає
послабленню): **вікно класифікується лише за своїми 1680 числами** — 8 фіч × 210 секунд.
Ані ціна, ані час доби, ані сусідні вікна, ані зовнішня норма у вхід не входять. Ззовні
заходять тільки константи масштабу, пораховані **на TRAIN**. Розмах ціни в наступну
годину рахується, але лише як **конверт** — його дозволено відкрити після того, як клас
уже поставлено.

**Друге правило — правило саме цієї лінії.** Спільним з оригіналом лишається рівно два
предмети, і обидва — введення-виведення, а не метод:

1. читач CSV (`standard_signals.load`);
2. **межа даних** `t1` — заморожена на тій самій, що в `/wclass`, щоб сітка вікон збіглася
   вікно-в-вікно. Без цього поточкове порівняння «моя модель проти їхнього класу» просто
   неможливе: вікна поїдуть на кілька секунд.

Усе решта — своє: сид **20260805** (в оригіналі 20260804), а отже інший блоковий спліт,
інші k-середні, інша ініціалізація мережі, інша розкладка фолдів, інші відкладені шматки.
Файли лежать окремо, у `media/analyst/mywclass/`.

---

## 1. Крок перший: вікна і блоковий спліт

`python3 mywclass.py data COIN`

Дедуплікація timestamp → суцільна 1-секундна шкала → лінійна інтерполяція дірок → вікна
**стик у стик** по 210 с (крок дорівнює довжині: перекриття вікон дало б витік між
навчанням і валідацією) → блоковий спліт.

**Чому саме так:**

- **вікно 210 с** — успадковано з оригіналу; це компроміс між «встигає статись
  подія» і «ще не встигає змінитись режим»;
- **дірки інтерполюються, але вікно з часткою реальних секунд < 0.95 викидається.** У нас
  відсутніх секунд 18.415% — це багато, і без порогу повноти частина «вікон»
  була б намальована інтерполяцією;
- **спліт блоками по 2 год з ембарго 30 хв.** Сусідні вікна корелюють; випадковий спліт
  окремими вікнами протягнув би сусіда з навчання у валідацію. Ембарго вирізає вікна, що
  стоять ближче за 30 хв до будь-якого валідаційного;
- **розмах ціни в наступну годину** рахується тут же і кладеться поруч — але у вхід
  жодної моделі не входить.

Контроль етапу: на нашому зрізі 12,325,025 рядків → 12,132,529 після
дедуплікації, 54,764 вікон, навчання 47,183 · валідація 5,394.
Якщо навчання + валідація ≈ усі вікна — **ембарго не спрацювало**, шукай помилку.

---

## 2. Крок другий: 89 описів вікна

`python3 mywclass.py feat COIN`

З кожного вікна (8 × 210 сирих чисел) рахується 89 описів: рівень · розкид · концентрація
(частка 10 найбільших секунд) · тиша (частка секунд нижче 5% масштабу) · нахил · вигин
(поліноми Лежандра 1 і 2 порядку) · автокореляція лаг-1 · «склад» вікна (clr-логвідношення
до спільного розміру) · сам розмір.

**Чому саме так.** Кожне з 89 чисел — функція **лише** 1680 чисел вікна. Єдине, що заходить
ззовні, — вектор масштабу з 8 чисел, порахований **на TRAIN**. Це прямий наслідок головного
правила: якщо в опис заходить ковзна норма, вікно перестає класифікуватись «зсередини»,
і головним сигналом стає епоха історії.

Три подання простору (їх далі порівнює `probe`): `full` (усе), `free` (лише безрозмірні
описи — без рівня), `lev` (лише рівні). Масштабування — вінзоризація 0.1/99.9% → робастний
z (медіана й IQR) → PCA з вибіленням, **усе на TRAIN**.

Контроль: `evr` (частка дисперсії у 16 осях) на нашому зрізі 0.8301 у поданні `full`.

---

## 3. Крок третій: чи є класи взагалі

`python3 mywclass.py probe COIN`

Це головний етап лінії, і його **не можна пропускати**, щоб «швидше дійти до моделей».
Будь-який кластеризатор намалює межі в чистому шумі, тому кожен критерій тут рахується
**разом зі своїм нуль-контролем, прогнаним через увесь конвеєр**: перемішані дані знову
проходять описи → масштабування → PCA → k-середні.

Два нулі, і різниця між ними змістовна:

- **жорсткий**: клітинки (фіча, секунда) перемішано **між** вікнами. Гине все, крім
  маргінальних розподілів;
- **мʼякий**: перемішано **цілі ряди** фіч. Форма кожного ряду ціла, спільність фіч
  усередині вікна зруйнована.

Що виходить на нашому зрізі:

| перевірка | дані | нуль | читання |
|---|---|---|---|
| gap max (`full`) | 0.0733 | 0 за побудовою | розбиття майже не стискає дані краще за шум |
| силует K=4 | 0.0565 | 0.0529 | на рівні шуму |
| Сільверман (мін. p) | 0.22 | — | другого горба немає ніде |
| стійкість K=4 | 0.1628 | 0.1662 | нуль стійкіший або рівний |
| стійкість K=6 | 0.1304 | 0.4160 | **нуль помітно стійкіший** |
| драбина evr | 0.8301 | мʼякий 0.6262 · жорсткий 0.6129 | структура — у СКЛАДІ вікна |

**Головне читання.** Дискретних класів немає — є континуум. Драбина нулів показує, де
сидить те, що PCA все ж стискає: щойно зруйновано **спільність фіч усередині вікна**
(мʼякий нуль), evr падає з 0.8301 до 0.6262, а подальше
руйнування форми (жорсткий нуль) віднімає ще лише 0.0133.
Тобто форма всередині вікна додає ~1%, а все інше — це те, які фічі активні одночасно.

Стійкість рахується **двома повними перебудовами** (кожна половина історії проходить свій
масштаб, свій PCA, свої k-середні), а не переприсвоєнням міток у фіксованому просторі.
Швидка оцінка у фіксованому просторі завищена вдвічі — цей урок лінія `/events` уже
заплатила.

---

## 4. Крок четвертий: два поважні критерії, які тут брешуть

`python3 mywclass.py lrt COIN`

Бутстреп-тест відношення правдоподібності (K=1 проти K=2) і інформаційні критерії BIC/ICL —
стандартний інструмент «скільки компонент у суміші». Обидва рахуються **і на даних, і на
жорсткому нулі**:

| набір | LR (K=1 проти K=2) | 95% нуля | вердикт | ICL мін. | BIC мін. |
|---|---|---|---|---|---|
| дані | 17,274.7 | 168.1 | K=1 відкинуто | K=5 | K=6 |
| жорсткий нуль | 5,080.5 | 180.0 | **K=1 теж відкинуто** | K=2 | K=2 |

Критерій, який спрацьовує на перемішаних клітинках, міряє **негаусовість маргіналій**,
а не наявність класів. Побачити це можна лише прогнавши його по нуль-контролю — тому
етап окремий і обовʼязковий.

---

## 5. Крок пʼятий: розріз континууму і чотири цілі

`python3 mywclass.py labels COIN`

Оскільки дискретних класів немає, розбиття — це **детермінований розріз**: PCA-16 у
поданні `full` → k-середні K=4 на TRAIN → предикт на всі вікна.

**Імена присвоюються з боку ВХОДУ**, а не з боку результату: класи сортуються за медіанним
сумарним рівнем 6 потокових фіч. Найактивніший — «шторм», два найнижчі зливаються в
«штиль», середній лишається «класом 2». Категорично не можна називати клас за тим, що
робила ціна після нього — це замкнене коло, в якому потім знайдеться будь-який «сигнал».

**Чому два найнижчі зливаються.** У жодної пари з 4 класів відстань між центрами не
дотягує до власного радіуса (у нас 0.523–0.673).
Це підтверджує континуум і водночас пояснює, чому чесна структура тут бінарна: шторм
проти решти.

Конверт відкривається тільки тут — і **обовʼязково двічі**: абсолютно і всередині доби
(ранг серед вікон тієї ж доби). На нашому зрізі:

| ціль | частка | розмах наступної години | усередині доби |
|---|---|---|---|
| шторм | 10.3% | ×1.761 | **×1.401** |
| клас 2 | 38.6% | ×1.141 | ×0.977 |
| штиль | 51.1% | ×0.822 | ×0.975 |

Поправка «всередині доби» вбиває майже все: ×1.141 у класу 2
стає ×0.977. Переживає лише шторм.

---

## 6. Крок шостий: детектори, 10 фолдів × 2 движки

`python3 mywclass.py clf COIN storm 10` (далі `mid`, `calm`)

Описи були потрібні, щоб **знайти** розріз. Детектор мусить працювати з **сирого вікна**:
вхід — уся матриця 8 × 210 після `log1p(x / 0.01)`. Стала 0.01 **призначена наперед і з
даних не береться** — інакше вона стає ще одним каналом витоку.

Спліт — 10 випадкових частин, кожне вікно оцінене рівно раз моделлю, яка його не бачила.
**Це навмисно завищена оцінка**, і так її і треба читати: спліт випадковий (як у щоденнику
оригіналу), сусідні вікна корелюють. Консервативні числа дає блоковий спліт з етапу
`probe`.

На нашому зрізі (out-of-fold, середнє по 10 фолдах):

| ціль | частка | Conv1D AUC | Conv1D PR-AUC | бустинг AUC | бустинг PR-AUC | переможець |
|---|---|---|---|---|---|---|
| шторм | 10.3% | 0.9966 | **0.9724** | 0.9934 | 0.9537 | Conv1D |
| клас 2 | 38.6% | 0.9532 | **0.933** | 0.9506 | 0.9281 | Conv1D |
| штиль | 51.1% | 0.959 | 0.9597 | 0.9581 | **0.9598** | бустинг |

Переможець обирається за **PR-AUC**, а не за AUC: цілі незбалансовані (шторм —
10.3% вікон), і AUC на такій частці виглядає красиво в усіх.

---

## 7. Крок сьомий: продакшен-моделі й відкладені 2%

`python3 mywclass.py final COIN` → `python3 mywclass.py scan COIN`

Крос-валідація моделі не лишає — вона прилад. Продакшен-модель довчається окремо, а
чесність міряється на **відкладених 2%, викладених ЧОТИРМА суцільними шматками з
ембарго 200 вікон**. Розсипана по одному вікну вибірка дала б завищене число з тієї ж
причини, що й випадковий спліт.

На нашому зрізі, ціль «шторм»: поріг 0.97343, відкладені 1096 вікон,
AUC 0.9976 · PR-AUC 0.9826 ·
точність 0.8722 · повнота 0.9355.

Етап `scan` проганяє продакшен-моделі по всій історії — і це **не прогноз**: 98% вікон
були в навчанні. Скан потрібен лише для графіків і для поточкового порівняння з іншою
лінією.

---

## 8. Крок восьмий: четверта ціль — «тихе ПСА»

`python3 mywclass.py poles COIN` → `python3 mywclass.py pq COIN 10`

**Тут була наша помилка, і вона лишається в щоденнику.** У першій версії репліки ціль
`pcaquiet` була свідомо пропущена як «мітка чужої лінії», а коли треба було показати
чотири панелі — замість неї підставлено розріз штилю навпіл (`calm_hi`/`calm_lo`). Це
підміна: `calm_hi`/`calm_lo` — це **сирі класи k-середніх до злиття**, а не окрема ціль.
Оригінал тренує рівно чотири: `storm`, `calm`, `mid`, `pcaquiet`. Виправлено 06.08.2026.
Якщо повторюєш — звіряй список цілей з `final()` оригіналу, а не з картинки.

Рецепт полюсів (свій перерахунок, свій сид): 8 × 210 у **разах від causal-норми 28 діб**
→ лог → робастний масштаб (медіана й IQR **лише з train**) → PCA-32 → UMAP-10 (сусідів 30)
→ HDBSCAN (min_cluster_size = 100). Полюси називаються за рівнем активності, а не за
номером, який дав HDBSCAN. Перші 28 діб — прогрів норми, їх у задачі немає.

**Норма саме 28 діб** (рівно 4 тижні), бо коротша норма їздить разом із тижневим циклом:
провал вихідних близько −16% зсуває рівень і підмішується у клас.

Нуль-контроль тут обовʼязковий подвійно: UMAP + HDBSCAN вміють виробляти «стійкі»
розбиття з чистого шуму (урок лінії `/events`: ARI 0.35-0.60 на нулі). На нашому зрізі
мій нуль дав **0 класів при 100.0% шуму** (в оригіналі
2 класи при 99.4%).

Результат: 4 полюси, шум 69.2%, evr 0.314; тихий полюс —
7.11% усіх вікон, розмах наступної години ×0.51
(усередині доби ×0.888).

**Найважливіше з цього етапу:** мітки двох незалежних прогонів UMAP+HDBSCAN сходяться
**гірше** за k-середні — збіг 94.99%, але Жаккар лише 0.5511,
κ 0.6831, ARI 0.6425. Натомість **детектори**, навчені на цих різних
мітках, збігаються майже точно (згортка 0.9711 проти
0.97 AUC). Це і є мораль лінії: нестійка мітка + стійкий детектор
означає, що модель вивчила не мітку, а те спільне, що під нею.

---

## 9. Крок девʼятий: звірка з оригіналом

`python3 mywclass.py compare COIN`

Зіставлення вікно-в-вікно (перетин за timestamp) на трьох рівнях:

1. **класи проти класів.** ARI за іменами 0.7726, збіг 93.0%;
   частки: штиль 51.1% / 51.9%, клас 2
   38.6% / 38.3%, шторм 10.3% /
   9.8%. **Порівнювати треба ІМЕНА, а не номери кластерів**: ARI за
   сирими номерами 0.6702 — нижче, бо k-середні нумерують класи як завгодно;
2. **моделі проти моделей.** Два незалежно навчені Conv1D спрацьовують на однакових
   98.5% вікон, Жаккар 0.856,
   κ 0.9141, ρ ймовірностей 0.914;
3. **перехресно.** Моя модель проти ЇХНЬОГО класу — 98.2%;
   їхня модель проти МОГО класу — 97.8%.

І окремо — **що каже ціна там, де моделі розходяться** (розмах наступної години, поділений
на медіану того ж дня):

| комірка | частка часу | усередині доби |
|---|---|---|
| спрацювали обидві | 8.9% | **×1.432** |
| лише моя | 1.3% | ×1.143 |
| лише їхня | 0.2% | ×1.277 |
| жодна | 89.5% | ×0.976 |

Корисний обʼєкт — не видача одного детектора, а **згода двох незалежних**.

---

## 10. Крок десятий: чотири цілі — алгоритм проти моделі

`python3 mywclass.py four COIN` → `python3 mywclass.py tab COIN`

Чотири панелі = чотири продакшен-цілі лінії. Зелене — що сказав **алгоритм** (мітка),
червоне — що сказала навчена на ній **модель** (переможець крос-валідації за PR-AUC,
поріг виставлений на частоту цілі). Це перевірка «чи модель відтворює розріз», а не
прогноз.

| ціль | алгоритм | модель | збіг | Жаккар | движок | відкладені 2%: AUC / PR-AUC |
|---|---|---|---|---|---|---|
| шторм | 10.3% | 10.3% | 98.2% | 0.841 | cnn | 0.9976 / 0.9826 |
| клас 2 | 38.6% | 38.5% | 88.9% | 0.7489 | cnn | 0.9468 / 0.9589 |
| штиль | 51.1% | 51.0% | 93.5% | 0.8807 | gbm | 0.964 / 0.9234 |
| тихе ПСА | 8.7% | 8.6% | 95.4% | 0.5828 | cnn | 0.9284 / 0.4812 |

Читати цю таблицю треба **парами колонок**. Високий збіг при низькому Жаккарі (тихе ПСА:
95.4% проти 0.5828) означає лише те, що
ціль рідкісна: збіг рахує і «обидва мовчать». У pcaquiet перші 28 діб порожні — там ще
немає норми, на якій визначена мітка.

`tab` зводить усі маленькі звіти лінії в один `{COIN}_tab.json`, який читає вкладка.

---

## Три пастки, які тут неминуче зустрінуться

**1. Критерій, що спрацьовує на шумі.** Бутстреп-LRT і ICL відкидають K=1 і на реальних
даних, і на перемішаних клітинках. Так само UMAP+HDBSCAN знаходять «стійкі» полюси в
чистому шумі. **Правило: жоден критерій наявності структури не читається без свого
нуль-контролю, прогнаного через УВЕСЬ конвеєр** — від описів до кластеризації. Нуль,
порахований у вже навченому просторі, не рахується.

**2. Підміна цілі.** Найлегший спосіб зіпсувати репліку — відтворити не те, що тренує
оригінал. Ми на це наступили: замість `pcaquiet` у четвертій панелі стояв розріз штилю
навпіл. Ознака підміни — ціль, якої немає в списку продакшен-моделей оригіналу.

**3. Епоха історії.** Будь-яке порівняння рівнів фіч між групами вікон насамперед міряє
**місяць**, а не стан ринку. Без зовнішньої норми найсильніший сигнал у 89 описах — саме
епоха. Ліки одні: **поправка «всередині доби»** (ранг серед вікон тієї ж доби). Вона
знижує ×1.761 шторму до ×1.401
і повністю зʼїдає перевагу решти класів. Число без цієї поправки публікувати не можна.

---

## Що перевіряти при повторі

| величина | наше | допуск | що означає вихід за межі |
|---|---|---|---|
| вікон із 171.0 діб | 54,764 | ±2% від `діб × 86400 / 210` | поріг повноти або дедуплікація |
| відсутніх секунд | 18.415% | 5-25% | інший збирач або інша монета |
| evr (`full`, 16 осей) | 0.8301 | 0.78-0.88 | інший набір описів |
| gap max | 0.0733 | < 0.15 | вище — перевір, чи нуль справді проходить конвеєр |
| силует K=4 (дані / нуль) | 0.057 / 0.053 | різниця < 0.02 | більша різниця — з’явились справжні класи, перевір витік |
| стійкість K=4 (дані / нуль) | 0.16 / 0.17 | 0.10-0.40, нуль поруч | дані сильно вище нуля — підозра на витік епохи в описи |
| частка «шторму» | 10.3% | 8-13% | інша монета або інша волатильність періоду |
| шторм: розмах усередині доби | ×1.401 | 1.25-1.55 | нижче 1.1 — розріз не той |
| Conv1D «шторм» PR-AUC (oof) | 0.9724 | 0.94-0.99 | нижче — спліт або вхід не той |
| «шторм» на відкладених 2% | AUC 0.9976 | 0.99-1.00 | нижче 0.97 — шматки не суцільні |
| полюси: нуль-контроль | 0 класів / 100.0% шуму | 0-2 класи, шум ≥99% | більше класів — сид або підвибірка |
| збіг класів з оригіналом | 93.0% | 88-96% | нижче — розійшлась сітка вікон (перевір `t1`) |
| збіг детекторів «шторм» | 98.5% | 96-99% | нижче — інший поріг або інша ціль |

---

## Відкинуті напрямки (щоб не витрачати час повторно)

- **Доводити класи через LRT / BIC / ICL.** Відкидають K=1 на чистому шумі
  (LR 5,080 при 95-му процентилі 180.0).
- **Оцінювати стійкість у фіксованому просторі.** Дає завищене вдвічі число; чесна оцінка —
  дві повні перебудови конвеєра на непересічних половинах історії.
- **Шукати «більше класів».** При K=5-6 **нуль стійкіший за дані**
  (0.42 проти 0.13). Континуум не стає дискретним від
  збільшення K.
- **Читати абсолютний розмах ціни як перевагу класу.** Без поправки «всередині доби» це
  вимір ринкової епохи; після неї лишається тільки шторм.
- **Порівнювати два прогони за сирими номерами кластерів.** ARI 0.6702 проти
  0.7726 за іменами — це артефакт нумерації, а не розбіжність.
- **Довіряти мітці UMAP+HDBSCAN як опорі.** Два незалежні прогони дають Жаккар
  0.5511 на тих самих вікнах; спиратись можна лише на навчений на ній детектор.

---

## Порядок команд

```bash
python3 mywclass.py data    BTC          # ~4 хв
python3 mywclass.py feat    BTC          # ~2 хв
python3 mywclass.py probe   BTC          # ~25 хв (двічі повний конвеєр на нулях)
python3 mywclass.py lrt     BTC          # ~8 хв
python3 mywclass.py labels  BTC          # ~1 хв
python3 mywclass.py clf     BTC storm 10  # ~7 хв на MPS
python3 mywclass.py clf     BTC mid   10  # ~7 хв
python3 mywclass.py clf     BTC calm  10  # ~8 хв
python3 mywclass.py final   BTC          # ~6 хв
python3 mywclass.py scan    BTC          # ~1 хв
python3 mywclass.py poles   BTC          # ~10 хв (UMAP+HDBSCAN двічі: дані + нуль)
python3 mywclass.py pq      BTC 10       # ~10 хв (+ pqcmp + tab автоматично)
python3 mywclass.py compare BTC          # ~3 хв (+ графіки uk/en + tab)
python3 mywclass.py four    BTC          # ~1 хв (+ tab)

node test_pages.js                        # сторож вкладок
```

Порядок обовʼязковий: `feat` читає кеш `data`, `labels` читає `feat`, `clf` читає мітки
`labels`, `four` читає продакшен-моделі `final` і `pq`, `compare` читає скан.

---

## Повний код

### Головний скрипт лінії

`mywclass.py`:

```python
"""
НЕЗАЛЕЖНА РЕПЛІКА ЛІНІЇ /wclass ЗА ЩОДЕННИКОМ WCLASS_DIARY.md.

Що це і навіщо. Щоденник заявлений як «виконуваний рецепт»: читаєш — отримуєш ті
самі класи і навчені моделі. Цей файл — перевірка цієї заяви. Він написаний ЗА
ТЕКСТОМ щоденника (розділи 1-7), а не копіюванням wclass.py, має власні вихідні
файли (media/analyst/mywclass) і власний сид, тож спліт, k-середні та ініціалізація
мережі тут ІНШІ. Збіг результатів з /wclass — це результат, а не наслідок спільного
коду.

Що навмисно спільне (і чому):
  · читач CSV (standard_signals.load) — це введення-виведення, не метод;
  · межа даних заморожена на t1 лінії /wclass, а сітка вікон починається з того
    самого t0 — інакше вікна поїдуть на кілька секунд і поточкове порівняння
    «моя модель проти їхнього класу» стане неможливим.

Ціль pcaquiet («тихе ПСА») спершу була пропущена — і це була помилка: це ЧЕТВЕРТА
продакшен-ціль оригінальної лінії (`final(... "storm","calm","mid","pcaquiet")`),
без неї репліка неповна. З 06.08 вона відтворюється тут повністю: власний
перерахунок полюсів (етап poles: норма 28 діб → PCA-32 → UMAP-10 → HDBSCAN,
мій сид) + власний детектор (етап pq) + порівняння з оригіналом (pqcmp).

Етапи:
    python3 mywclass.py data    BTC        # вікна 210 с + блоковий спліт з ембарго
    python3 mywclass.py feat    BTC        # 89 описів вікна
    python3 mywclass.py probe   BTC        # чи є класи взагалі: gap/силует/Сільверман/
                                           #   стійкість/драбина нулів (розділ 3)
    python3 mywclass.py labels  BTC        # розріз континууму: шторм/клас 2/штиль (розділ 4)
    python3 mywclass.py clf     BTC storm 10   # детектор: 10 фолдів × 2 движки (розділ 6)
    python3 mywclass.py final   BTC        # продакшен-моделі + відкладені 2% (розділ 7)
    python3 mywclass.py scan    BTC        # продакшен-моделі по всій історії
    python3 mywclass.py compare BTC        # МОЇ моделі проти класів /wclass + графіки
    python3 mywclass.py poles   BTC        # полюси ПСА: «тихе ПСА» (pcaquiet) — мій перерахунок
    python3 mywclass.py pq      BTC        # детектор «тихого ПСА» + порівняння з оригіналом
"""
import os, sys, json, time, pickle
import numpy as np

import standard_signals as ST
from standard_signals import FEATS

ROOT = os.path.dirname(os.path.abspath(__file__))
OUT_DIR = os.path.join(ROOT, "media", "analyst", "mywclass")
CACHE = os.path.join(OUT_DIR, "cache")
MODELS = os.path.join(OUT_DIR, "models")
REF_DIR = os.path.join(ROOT, "media", "analyst", "wclass")   # лінія для порівняння

WIN = 210
STEP = WIN
MIN_REAL = 0.95
BLOCK_H = 2
EMBARGO_S = 1800
VAL_FRAC = 0.10
SEED = 20260805            # МІЙ сид — інший, ніж 20260804 у /wclass
D_PCA = 16
K_CLS = 4
WINSOR = (0.001, 0.999)
TOPK = 10
QUIET_U = 0.05
HOLD_FRAC, HOLD_CHUNKS, HOLD_EMB = 0.02, 4, 200


def log(*a):
    print(time.strftime("[%H:%M:%S]"), *a, flush=True)


def _dev():
    import torch
    if torch.backends.mps.is_available():
        return torch.device("mps")
    return torch.device("cpu")


# ============================ 1. ДАНІ ============================
def stage_data(coin):
    """Дедуплікація · суцільна 1-с шкала · вікна стик у стик · блоковий спліт."""
    os.makedirs(CACHE, exist_ok=True)
    S = ST.load(coin)
    if S is None:
        log("немає даних"); return None
    n_raw = len(S["ts"])
    _, first = np.unique(S["ts"], return_index=True)
    first = np.sort(first)
    for k in ["ts", "price"] + FEATS:
        S[k] = S[k][first]
    ts = S["ts"]
    log(f"{coin}: {n_raw} рядків → {len(ts)} після дедуплікації (−{n_raw - len(ts)})")

    t0, t1 = int(ts[0]), int(ts[-1])
    # межа даних: за замовчуванням — та сама, що в /wclass, щоб вікна збігалися
    ref = os.path.join(REF_DIR, "cache", f"{coin}_data.json")
    end = os.environ.get("MYW_END", "").strip()
    if end:
        t1 = min(t1, int(end))
    elif os.path.exists(ref):
        t1 = min(t1, int(json.load(open(ref))["t1"]))
        log(f"{coin}: межу заморожено на t1 лінії /wclass = {t1}")
    N = t1 - t0 + 1
    pos = (ts - t0).astype(np.int64)
    keep_row = pos < N
    pos = pos[keep_row]
    present = np.zeros(N, bool); present[pos] = True
    n_miss = int(N - present.sum())
    log(f"{coin}: шкала {N} с ({N/86400:.1f} діб) · відсутніх {n_miss} "
        f"({n_miss/N*100:.2f}%) — лінійна інтерполяція")

    gi = np.arange(N, dtype=np.float64)
    px_all = np.interp(gi, pos, S["price"][keep_row]).astype(np.float32)

    starts = np.arange(0, N - WIN + 1, STEP, dtype=np.int64)
    cp = np.concatenate([[0], np.cumsum(present, dtype=np.int64)])
    real = (cp[starts + WIN] - cp[starts]) / WIN
    ok = real >= MIN_REAL
    starts, real = starts[ok], real[ok].astype(np.float32)
    n_win = len(starts)
    log(f"{coin}: вікон {n_win} (відсіяно за повнотою {int((~ok).sum())})")

    idx = starts[:, None] + np.arange(WIN, dtype=np.int64)[None, :]
    X = np.empty((n_win, 8, WIN), np.float32)
    for j, f in enumerate(FEATS):
        v = np.interp(gi, pos, S[f][keep_row].astype(np.float64)).astype(np.float32)
        X[:, j, :] = v[idx]
        del v
        log(f"   {f}: нарізано")
    del idx, gi

    fin = np.isfinite(X).all(axis=(1, 2))
    if not fin.all():
        X, starts, real = X[fin], starts[fin], real[fin]
        n_win = len(starts)
    wts = (t0 + starts).astype(np.int64)
    wpx = px_all[starts]

    # КОНВЕРТ: розмах ціни в наступну годину. У вхід моделі не входить.
    rng_fwd = np.zeros(n_win, np.float32)
    for i, s0 in enumerate(starts):
        a = int(s0) + WIN; b = min(a + 3600, N)
        if b > a:
            seg = px_all[a:b]
            rng_fwd[i] = (seg.max() - seg.min()) / max(px_all[a], 1e-9) * 100

    # спліт блоками 2 год з ембарго 30 хв (МІЙ сид → інші блоки, ніж у /wclass)
    rng = np.random.default_rng(SEED)
    blk = ((wts - wts[0]) // (BLOCK_H * 3600)).astype(np.int64)
    ub = np.unique(blk)
    sh = rng.permutation(len(ub))
    val_blocks = set(ub[sh[:max(1, int(round(len(ub) * VAL_FRAC)))]].tolist())
    is_val = np.array([b in val_blocks for b in blk])
    val = np.flatnonzero(is_val).astype(np.int32)
    vts = wts[val]
    near = np.zeros(len(wts), bool)
    j = np.searchsorted(vts, wts)
    for shift in (0, -1):
        k = np.clip(j + shift, 0, len(vts) - 1)
        near |= np.abs(wts - vts[k]) <= EMBARGO_S
    train = np.flatnonzero(~is_val & ~near).astype(np.int32)
    log(f"{coin}: навчання {len(train)} · валідація {len(val)} · "
        f"ембарго {int((~is_val & near).sum())}")

    np.savez(os.path.join(CACHE, f"{coin}_data.npz"), X=X, ts=wts, px=wpx,
             rng_fwd=rng_fwd, real=real, train=train, val=val)
    meta = {"coin": coin, "built": int(time.time()), "seed": SEED,
            "rows_raw": int(n_raw), "rows_dedup": int(len(ts)),
            "t0": t0, "t1": t1, "days": round(N / 86400, 1),
            "missing_pct": round(n_miss / N * 100, 3),
            "win": WIN, "n_win": int(n_win), "n_train": int(len(train)),
            "n_val": int(len(val))}
    json.dump(meta, open(os.path.join(CACHE, f"{coin}_data.json"), "w"),
              ensure_ascii=False)
    log(f"{coin}: дані готові X={X.shape}")
    return meta


# ============================ 2. ОПИС ВІКНА ============================
def _legendre(win=WIN):
    t = np.linspace(-1.0, 1.0, win)
    Q, _ = np.linalg.qr(np.stack([np.ones(win), t, t * t], axis=1))
    return Q[:, 1].astype(np.float32), Q[:, 2].astype(np.float32)


P1, P2 = _legendre()


def descriptors(Xc, s):
    """(m,8,210) сирі → (m,89). Кожне число — функція ЛИШЕ 1680 чисел вікна;
    ззовні заходить тільки масштаб s (константа вибірки, з TRAIN)."""
    u = Xc / s[None, :, None]
    L = np.log1p(u)
    lev = L.mean(axis=2)
    sd = L.std(axis=2)
    tot = u.sum(axis=2)
    top = np.partition(u, -TOPK, axis=2)[:, :, -TOPK:].sum(axis=2)
    conc = top / np.maximum(tot, 1e-12)
    quiet = (u < QUIET_U).mean(axis=2)
    trend = L @ P1
    curv = L @ P2
    a, b = L[:, :, :-1], L[:, :, 1:]
    am = a.mean(axis=2, keepdims=True); bm = b.mean(axis=2, keepdims=True)
    num = ((a - am) * (b - bm)).mean(axis=2)
    den = a.std(axis=2) * b.std(axis=2)
    ac1 = np.where(den > 1e-9, num / np.maximum(den, 1e-12), 0.0)
    lm = np.log(tot + 1e-6)
    size = lm.mean(axis=1, keepdims=True)
    clr = lm - size
    cv = sd / np.maximum(np.abs(lev), 1e-6)
    trend_n = trend / np.maximum(sd, 1e-6)
    curv_n = curv / np.maximum(sd, 1e-6)
    D = np.concatenate([lev, sd, conc, quiet, trend, curv, ac1,
                        cv, trend_n, curv_n, clr, size], axis=1)
    return D.astype(np.float32)


GROUPS = ["lev", "sd", "conc", "quiet", "trend", "curv", "ac1",
          "cv", "trendN", "curvN", "clr"]
NAMES = [f"{g}:{f}" for g in GROUPS for f in FEATS] + ["size"]
REPR = {"full": ["lev", "sd", "conc", "quiet", "trend", "curv", "ac1", "clr", "size"],
        "free": ["conc", "cv", "trendN", "curvN", "ac1", "clr"],
        "lev": ["lev", "size"]}


def repr_cols(name):
    grp = REPR[name]
    return np.array([i for i, n in enumerate(NAMES)
                     if n.split(":")[0] in grp or n in grp])


def stage_feat(coin, chunk=4000):
    d = np.load(os.path.join(CACHE, f"{coin}_data.npz"), mmap_mode="r")
    X = d["X"]; train = np.asarray(d["train"])
    n = X.shape[0]
    acc = np.zeros(8, np.float64); cnt = 0
    for i in range(0, len(train), chunk):
        sl = train[i:i + chunk]
        ch = np.asarray(X[sl])
        acc += ch.mean(axis=(0, 2), dtype=np.float64) * len(sl); cnt += len(sl)
    s = (acc / cnt).astype(np.float32)
    log(f"{coin}: масштаби (TRAIN) " + " ".join(f"{f}={v:.4g}" for f, v in zip(FEATS, s)))
    D = np.empty((n, len(NAMES)), np.float32)
    for i in range(0, n, chunk):
        D[i:i + chunk] = descriptors(np.asarray(X[i:i + chunk]), s)
    D[~np.isfinite(D)] = 0.0
    np.savez(os.path.join(CACHE, f"{coin}_feat.npz"), D=D, scale=s)
    log(f"{coin}: описи {D.shape}")
    return D


class Space:
    """вінзоризація → робастний z → PCA з вибіленням. Усе — тільки на TRAIN."""

    def __init__(self, cols, d):
        self.cols, self.d = cols, d

    def fit(self, D, train):
        from sklearn.decomposition import PCA
        T = D[train][:, self.cols]
        self.lo = np.quantile(T, WINSOR[0], axis=0)
        self.hi = np.quantile(T, WINSOR[1], axis=0)
        T = np.clip(T, self.lo, self.hi)
        self.med = np.median(T, axis=0)
        q75, q25 = np.percentile(T, [75, 25], axis=0)
        self.iqr = np.where(q75 - q25 > 1e-9, q75 - q25, 1.0)
        self.pca = PCA(n_components=min(self.d, T.shape[1]), whiten=True,
                       random_state=SEED).fit((T - self.med) / self.iqr)
        self.evr = float(self.pca.explained_variance_ratio_.sum())
        return self

    def transform(self, D):
        A = np.clip(D[:, self.cols], self.lo, self.hi)
        return self.pca.transform((A - self.med) / self.iqr).astype(np.float32)


def load_all(coin):
    f = np.load(os.path.join(CACHE, f"{coin}_feat.npz"))
    d = np.load(os.path.join(CACHE, f"{coin}_data.npz"), mmap_mode="r")
    return {"D": f["D"], "scale": f["scale"], "train": np.asarray(d["train"]),
            "val": np.asarray(d["val"]), "ts": np.asarray(d["ts"]),
            "px": np.asarray(d["px"]), "rng_fwd": np.asarray(d["rng_fwd"])}


# ============================ НУЛЬ-КОНТРОЛЬ ============================
def _null_D(coin, mode, seed):
    """hard — клітинки (фіча, секунда) перемішані МІЖ вікнами (гине все, крім
    маргіналій); soft — перемішані цілі ряди фіч (форма ціла, спільність фіч ні).
    Перемішування ОКРЕМО в TRAIN і VAL, інакше витік."""
    d = np.load(os.path.join(CACHE, f"{coin}_data.npz"))
    X = d["X"]; train = np.asarray(d["train"]); val = np.asarray(d["val"])
    s = np.load(os.path.join(CACHE, f"{coin}_feat.npz"))["scale"]
    rng = np.random.default_rng(seed)
    for part in (train, val):
        Xp = X[part]
        for j in range(8):
            if mode == "soft":
                Xp[:, j, :] = Xp[rng.permutation(len(part)), j, :]
            else:
                for t in range(WIN):
                    Xp[:, j, t] = Xp[rng.permutation(len(part)), j, t]
        X[part] = Xp
        del Xp
    D = np.empty((X.shape[0], len(NAMES)), np.float32)
    for i in range(0, X.shape[0], 4000):
        D[i:i + 4000] = descriptors(X[i:i + 4000], s)
    D[~np.isfinite(D)] = 0.0
    del X
    return D


# ============================ 3. ЧИ Є КЛАСИ ============================
def _logw(Z, ks, seed=SEED):
    from sklearn.cluster import KMeans
    out = {}
    for K in ks:
        if K == 1:
            out[K] = float(np.log(((Z - Z.mean(0)) ** 2).sum()))
            continue
        km = KMeans(n_clusters=K, n_init=6, random_state=seed).fit(Z)
        out[K] = float(np.log(max(km.inertia_, 1e-12)))
    return out


def _silhouette(Z, lab, n=6000, seed=SEED):
    from sklearn.metrics import silhouette_score
    rng = np.random.default_rng(seed)
    if len(Z) > n:
        i = rng.choice(len(Z), n, replace=False)
        Z, lab = Z[i], lab[i]
    if len(np.unique(lab)) < 2:
        return 0.0
    return float(silhouette_score(Z, lab))


def _n_modes(x, h, grid=512):
    lo, hi = x.min() - 3 * h, x.max() + 3 * h
    g = np.linspace(lo, hi, grid)
    dens = np.exp(-0.5 * ((g[:, None] - x[None, :]) / h) ** 2).sum(1)
    return int(((dens[1:-1] > dens[:-2]) & (dens[1:-1] >= dens[2:])).sum())


def silverman(x, k=1, reps=100, seed=SEED):
    """Критична ширина ядра для k мод + згладжений бутстреп.
    p велике → більше ніж k мод не доведено."""
    rng = np.random.default_rng(seed)
    x = np.asarray(x, np.float64)
    x = x[np.isfinite(x)]
    if len(x) > 2500:
        x = rng.choice(x, 2500, replace=False)
    sd = x.std()
    lo, hi = 1e-3 * sd, 3.0 * sd
    for _ in range(40):
        mid = 0.5 * (lo + hi)
        if _n_modes(x, mid) > k:
            lo = mid
        else:
            hi = mid
    h = hi
    n = len(x)
    cnt = 0
    for _ in range(reps):
        idx = rng.integers(0, n, n)
        xb = x[idx] + h * rng.standard_normal(n)
        xb = x.mean() + (xb - xb.mean()) / np.sqrt(1 + h ** 2 / x.var())
        if _n_modes(xb, h) > k:
            cnt += 1
    return {"h_crit": round(float(h / sd), 4), "p": round(cnt / reps, 3), "k": k}


def stage_probe(coin, ks=tuple(range(1, 13))):
    """Розділ 3 щоденника: gap, силует, Сільверман, стійкість, драбина нулів.
    Кожен критерій — разом зі своїм нуль-контролем, прогнаним через увесь конвеєр."""
    from sklearn.cluster import KMeans
    from sklearn.metrics import adjusted_rand_score
    A = load_all(coin)
    D, train, val = A["D"], A["train"], A["val"]
    out = {"coin": coin, "built": int(time.time()), "seed": SEED, "ks": list(ks)}

    log("нуль-контроль: жорсткий (клітинки)…")
    Dh = _null_D(coin, "hard", SEED + 1)
    log("нуль-контроль: мʼякий (ряди фіч)…")
    Ds = _null_D(coin, "soft", SEED + 2)

    out["spaces"] = {}
    for rp, d in (("full", 16), ("free", 16), ("lev", 8)):
        cols = repr_cols(rp)
        sp = Space(cols, d).fit(D, train)
        Z = sp.transform(D)[val]
        sph = Space(cols, d).fit(Dh, train)
        Zh = sph.transform(Dh)[val]
        lw = _logw(Z, ks); lwh = _logw(Zh, ks)
        gap = [{"K": K, "logw": round(lw[K], 5), "null": round(lwh[K], 5),
                "gap": round(lwh[K] - lw[K] - (lwh[1] - lw[1]), 5)} for K in ks]
        sils = []
        for K in (2, 3, 4, 5, 6):
            l1 = KMeans(n_clusters=K, n_init=6, random_state=SEED).fit_predict(Z)
            l2 = KMeans(n_clusters=K, n_init=6, random_state=SEED).fit_predict(Zh)
            sils.append({"K": K, "data": round(_silhouette(Z, l1), 4),
                         "null": round(_silhouette(Zh, l2), 4)})
        out["spaces"][rp] = {"d": d, "evr": round(sp.evr, 4),
                             "evr_null": round(sph.evr, 4), "gap": gap, "sil": sils}
        log(f"  {rp}/{d}: evr {sp.evr:.3f} (нуль {sph.evr:.3f}) · "
            f"gap max {max(g['gap'] for g in gap):.4f} · "
            f"силует K=4 {sils[2]['data']:.3f} проти {sils[2]['null']:.3f}")

    # драбина нулів: де саме сидить структура
    spf = Space(repr_cols("full"), 16)
    out["ladder"] = {"real": round(spf.fit(D, train).evr, 4),
                     "soft": round(Space(repr_cols("full"), 16).fit(Ds, train).evr, 4),
                     "hard": round(Space(repr_cols("full"), 16).fit(Dh, train).evr, 4)}
    log(f"  драбина evr: реальні {out['ladder']['real']} → мʼякий {out['ladder']['soft']}"
        f" → жорсткий {out['ladder']['hard']}")

    # Сільверман: головні осі + напрямок Фішера, на ВІДКЛАДЕНИХ вікнах
    sp = Space(repr_cols("full"), 16).fit(D, train)
    Ztr, Zva = sp.transform(D)[train], sp.transform(D)[val]
    sil = []
    for a in range(3):
        sil.append({"axis": f"PC{a+1}", **silverman(Zva[:, a])})
    km2 = KMeans(n_clusters=2, n_init=10, random_state=SEED).fit(Ztr)
    w = km2.cluster_centers_[1] - km2.cluster_centers_[0]
    w = w / np.linalg.norm(w)
    sil.append({"axis": "Фішер(2 центри, напрямок з TRAIN)", **silverman(Zva @ w)})
    out["silverman"] = sil
    for s_ in sil:
        log(f"  Сільверман {s_['axis']}: h*={s_['h_crit']} p={s_['p']}")

    # стійкість: ДВІ ПОВНІ перебудови на непересічних часових половинах
    ts = A["ts"]
    half = ts < np.median(ts)
    stab = []
    for K in (2, 3, 4, 5, 6):
        labs = []
        for m in (half, ~half):
            tr_h = train[m[train]]
            s_ = Space(repr_cols("full"), 16).fit(D, tr_h)
            Zh_ = s_.transform(D)
            km = KMeans(n_clusters=K, n_init=10, random_state=SEED).fit(Zh_[tr_h])
            labs.append(km.predict(Zh_[val]))
        # той самий тест на нуль-контролі
        labs_n = []
        for m in (half, ~half):
            tr_h = train[m[train]]
            s_ = Space(repr_cols("full"), 16).fit(Dh, tr_h)
            Zn = s_.transform(Dh)
            km = KMeans(n_clusters=K, n_init=10, random_state=SEED).fit(Zn[tr_h])
            labs_n.append(km.predict(Zn[val]))
        stab.append({"K": K, "ari": round(float(adjusted_rand_score(*labs)), 4),
                     "ari_null": round(float(adjusted_rand_score(*labs_n)), 4)})
        log(f"  стійкість K={K}: ARI {stab[-1]['ari']} проти нуля {stab[-1]['ari_null']}")
    out["stability"] = stab
    json.dump(out, open(os.path.join(CACHE, f"{coin}_probe.json"), "w"), ensure_ascii=False)
    return out


def stage_lrt(coin, rp="full", d=D_PCA, n=6000, reps=20):
    """Пастка щоденника: бутстреп-LRT і ICL НЕ доводять наявність класів.
    Тому обидва рахуються і на даних, і на жорсткому нулі — якщо K=1 відкидається
    в обох випадках, критерій міряє негаусовість маргіналій, а не класи."""
    from sklearn.mixture import GaussianMixture
    A = load_all(coin)
    D, train, val = A["D"], A["train"], A["val"]
    rng = np.random.default_rng(SEED)
    Dh = _null_D(coin, "hard", SEED + 1)
    out = {"coin": coin, "built": int(time.time()), "n": n, "reps": reps, "sets": {}}
    for tag, DD in (("дані", D), ("жорсткий нуль", Dh)):
        sp = Space(repr_cols(rp), d).fit(DD, train)
        Z = sp.transform(DD)[val]
        Z = Z[rng.choice(len(Z), min(n, len(Z)), replace=False)]

        def fit(K, Y):
            return GaussianMixture(K, covariance_type="full", n_init=2,
                                   random_state=SEED, reg_covar=1e-5).fit(Y)
        g1, g2 = fit(1, Z), fit(2, Z)
        lr = float(2 * (g2.score(Z) - g1.score(Z)) * len(Z))
        null_lr = []
        for b in range(reps):
            Yb, _ = g1.sample(len(Z))
            null_lr.append(float(2 * (fit(2, Yb).score(Yb) - fit(1, Yb).score(Yb)) * len(Yb)))
        icl = []
        for K in range(1, 13):
            g = fit(K, Z)
            r = g.predict_proba(Z)
            ent = float(-(r * np.log(np.maximum(r, 1e-12))).sum())
            icl.append({"K": K, "bic": round(float(g.bic(Z)), 1),
                        "icl": round(float(g.bic(Z) + 2 * ent), 1)})
        out["sets"][tag] = {"lr_1vs2": round(lr, 1),
                            "null_med": round(float(np.median(null_lr)), 1),
                            "null_q95": round(float(np.quantile(null_lr, 0.95)), 1),
                            "reject_K1": bool(lr > np.quantile(null_lr, 0.95)),
                            "icl": icl,
                            "icl_argmin": int(min(icl, key=lambda x: x["icl"])["K"]),
                            "bic_argmin": int(min(icl, key=lambda x: x["bic"])["K"])}
        s = out["sets"][tag]
        log(f"  {tag}: LR(1 проти 2) {s['lr_1vs2']} при нулі {s['null_q95']} → "
            f"K=1 {'відкинуто' if s['reject_K1'] else 'не відкинуто'} · "
            f"ICL мінімум K={s['icl_argmin']} · BIC K={s['bic_argmin']}")
    json.dump(out, open(os.path.join(CACHE, f"{coin}_lrt.json"), "w"), ensure_ascii=False)
    return out


# ============================ 4. РОЗРІЗ КОНТИНУУМУ ============================
def build_labels(coin, rp="full", d=D_PCA, K=K_CLS):
    """k-середні K=4 у просторі описів; імена — з боку ВХОДУ (ранг за сумарним
    рівнем 6 потокових фіч), а не за номером кластера."""
    from sklearn.cluster import KMeans
    A = load_all(coin)
    D, train = A["D"], A["train"]
    sp = Space(repr_cols(rp), d).fit(D, train)
    Z = sp.transform(D)
    km = KMeans(n_clusters=K, n_init=10, random_state=SEED).fit(Z[train])
    lab = km.predict(Z)
    flow = [f for f in FEATS if f not in ("const_resist", "const_support")]
    fi = [NAMES.index(f"lev:{f}") for f in flow]
    fl = np.array([float(np.median(D[lab == k][:, fi].sum(1))) for k in range(K)])
    order = list(np.argsort(fl))
    storm = int(order[-1]); calm = [int(order[0]), int(order[1])]; mid = int(order[2])
    return dict(A=A, D=D, Z=Z, lab=lab, sp=sp, fl=fl, storm=storm, calm=calm, mid=mid)


def stage_labels(coin):
    from scipy.stats import mannwhitneyu, kruskal
    B = build_labels(coin)
    D, Z, lab, A = B["D"], B["Z"], B["lab"], B["A"]
    rf, ts = A["rng_fwd"], A["ts"]
    K = K_CLS
    out = {"coin": coin, "built": int(time.time()), "seed": SEED,
           "storm_cls": B["storm"], "calm_cls": B["calm"], "mid_cls": B["mid"],
           "flow_level": [round(float(v), 3) for v in B["fl"]],
           "shares": [round(float((lab == k).mean()), 4) for k in range(K)],
           "evr": round(B["sp"].evr, 4)}

    lev_i = [NAMES.index(f"lev:{f}") for f in FEATS]
    out["portrait"] = [{"cls": k, "n": int((lab == k).sum()),
                        "lev": {f: round(float(np.expm1(np.median(D[lab == k, i]))), 4)
                                for f, i in zip(FEATS, lev_i)}} for k in range(K)]

    # конверт: forward-розмах абсолютно і ВСЕРЕДИНІ ДОБИ
    day = ((ts - ts[0]) // 86400).astype(int)
    medd = np.array([np.median(rf[day == q]) if (day == q).sum() > 10 else np.nan
                     for q in range(day.max() + 1)])
    rel = rf / medd[day]
    base = float(np.median(rf))
    cls = []
    for k in range(K):
        m = lab == k
        cls.append({"cls": k, "n": int(m.sum()),
                    "med": round(float(np.median(rf[m])), 4),
                    "ratio": round(float(np.median(rf[m]) / base), 3),
                    "within_day": round(float(np.nanmedian(rel[m])), 3),
                    "p": float(mannwhitneyu(rf[m], rf[~m]).pvalue)})
    out["fwd"] = {"base_med": round(base, 4), "cls": cls,
                  "kruskal_p": float(kruskal(*[rf[lab == k] for k in range(K)]).pvalue)}

    # чи не є пара класів однією купкою: відстань центрів проти власного радіуса
    C = np.array([Z[lab == k].mean(0) for k in range(K)])
    rad = np.array([np.sqrt(((Z[lab == k] - C[k]) ** 2).sum(1).mean()) for k in range(K)])
    pairs = []
    for i in range(K):
        for j in range(i + 1, K):
            dist = float(np.linalg.norm(C[i] - C[j]))
            r = float((rad[i] + rad[j]) / 2)
            pairs.append({"i": i, "j": j, "dist": round(dist, 3), "rad": round(r, 3),
                          "ratio": round(dist / max(r, 1e-9), 3)})
    out["merge"] = {"pairs": pairs, "radius": [round(float(v), 3) for v in rad]}

    # чотири СИРІ класи k-середніх (до злиття двох найнижчих у «штиль»):
    # calm_lo — найнижчий рівень потоків, calm_hi — другий знизу
    y = {"storm": (lab == B["storm"]).astype(np.int8),
         "calm": np.isin(lab, B["calm"]).astype(np.int8),
         "mid": (lab == B["mid"]).astype(np.int8),
         "calm_lo": (lab == B["calm"][0]).astype(np.int8),
         "calm_hi": (lab == B["calm"][1]).astype(np.int8)}
    out["raw4"] = {"storm": B["storm"], "mid": B["mid"],
                   "calm_hi": B["calm"][1], "calm_lo": B["calm"][0]}
    for tg, v in y.items():
        m = v == 1
        out.setdefault("targets", {})[tg] = {
            "share": round(float(v.mean()), 4),
            "ratio": round(float(np.median(rf[m]) / base), 3),
            "within_day": round(float(np.nanmedian(rel[m])), 3)}
        log(f"  ціль «{tg}»: {v.mean()*100:.1f}% вікон · розмах ×"
            f"{out['targets'][tg]['ratio']} · усередині доби ×{out['targets'][tg]['within_day']}")
    np.savez(os.path.join(CACHE, f"{coin}_labels.npz"), lab=lab, **y)
    json.dump(out, open(os.path.join(CACHE, f"{coin}_labels.json"), "w"), ensure_ascii=False)
    return out


# ============================ 6. ДЕТЕКТОРИ ============================
def prep_fixed(X):
    """Строгий вхід: log1p(x/0.01). Стала призначена наперед, з даних не береться."""
    return np.log1p(X / 0.01).astype(np.float32)


def _cnn():
    import torch.nn as nn
    return nn.Sequential(
        nn.Conv1d(8, 32, 7, padding=3), nn.BatchNorm1d(32), nn.ReLU(), nn.MaxPool1d(2),
        nn.Conv1d(32, 64, 5, padding=2), nn.BatchNorm1d(64), nn.ReLU(), nn.MaxPool1d(2),
        nn.Conv1d(64, 96, 3, padding=1), nn.BatchNorm1d(96), nn.ReLU(),
        nn.AdaptiveAvgPool1d(1), nn.Flatten(), nn.Dropout(0.2), nn.Linear(96, 1))


def fit_cnn(L, tr, y, epochs=8, bs=512, lr=2e-3, seed=SEED, predict_idx=None):
    import torch, torch.nn as nn
    dev = _dev()
    torch.manual_seed(seed)
    net = _cnn().to(dev)
    pos = float(y[tr].mean())
    lossf = nn.BCEWithLogitsLoss(
        pos_weight=torch.tensor([(1 - pos) / max(pos, 1e-6)], device=dev))
    opt = torch.optim.AdamW(net.parameters(), lr=lr, weight_decay=1e-4)
    Lt = torch.from_numpy(L)
    yt = torch.from_numpy(y.astype(np.float32))
    ti = torch.from_numpy(np.asarray(tr, np.int64))
    for ep in range(epochs):
        net.train()
        perm = ti[torch.randperm(len(ti))]
        tot = 0.0
        for i in range(0, len(perm), bs):
            k = perm[i:i + bs]
            xb = Lt[k].to(dev); yb = yt[k].to(dev)
            opt.zero_grad()
            loss = lossf(net(xb).squeeze(1), yb)
            loss.backward(); opt.step()
            tot += float(loss.detach()) * len(k)
        log(f"      епоха {ep+1}/{epochs} loss {tot/len(perm):.4f}")
    net.eval()
    idx = np.arange(len(L)) if predict_idx is None else np.asarray(predict_idx)
    ps = np.empty(len(idx), np.float32)
    with torch.no_grad():
        for i in range(0, len(idx), 4096):
            k = torch.from_numpy(idx[i:i + 4096].astype(np.int64))
            ps[i:i + 4096] = torch.sigmoid(
                net(Lt[k].to(dev)).squeeze(1)).cpu().numpy()
    return net, ps


def fit_gbm(F, tr, y, seed=SEED, predict_idx=None):
    from sklearn.ensemble import HistGradientBoostingClassifier as HGC
    gb = HGC(max_iter=120, learning_rate=0.12, max_bins=32, early_stopping=True,
             validation_fraction=0.1, random_state=seed).fit(F[tr], y[tr])
    idx = np.arange(len(F)) if predict_idx is None else np.asarray(predict_idx)
    ps = np.concatenate([gb.predict_proba(F[idx[i:i + 8192]])[:, 1]
                         for i in range(0, len(idx), 8192)])
    return gb, ps.astype(np.float32)


def stage_clf(coin, target="storm", folds=10):
    """10 випадкових частин, кожне вікно оцінене рівно раз моделлю, яка його не
    бачила. Вхід обох движків — уся матриця 8×210 після log1p(x/0.01)."""
    from sklearn.metrics import roc_auc_score, average_precision_score, f1_score
    y = np.load(os.path.join(CACHE, f"{coin}_labels.npz"))[target].astype(np.int8)
    dd = np.load(os.path.join(CACHE, f"{coin}_data.npz"), mmap_mode="r")
    L = prep_fixed(np.asarray(dd["X"]))
    F = L.reshape(len(L), -1)
    n = len(L)
    rng = np.random.default_rng(SEED)
    part = rng.permutation(n) % folds
    log(f"{coin}/{target}: {n} вікон · ціль {int(y.sum())} ({y.mean()*100:.1f}%) · "
        f"{folds} частин · пристрій {_dev()}")
    oof = {"cnn": np.full(n, np.nan, np.float32), "gbm": np.full(n, np.nan, np.float32)}
    times = {"cnn": 0.0, "gbm": 0.0}
    for f in range(folds):
        va = np.flatnonzero(part == f); tr = np.flatnonzero(part != f)
        t = time.time()
        _, oof["gbm"][va] = fit_gbm(F, tr, y, seed=SEED + f, predict_idx=va)
        times["gbm"] += time.time() - t
        t = time.time()
        _, oof["cnn"][va] = fit_cnn(L, tr, y, seed=SEED + f, predict_idx=va)
        times["cnn"] += time.time() - t
        log(f"   фолд {f+1}/{folds}: " + " · ".join(
            f"{k} AUC {roc_auc_score(y[va], oof[k][va]):.4f}" for k in oof))
    res = {"coin": coin, "target": target, "folds": folds, "n": int(n),
           "pos": float(y.mean()), "prep": "fixed", "seed": SEED, "models": {}}
    for k, p in oof.items():
        per = np.array([[roc_auc_score(y[part == f], p[part == f]),
                         average_precision_score(y[part == f], p[part == f])]
                        for f in range(folds)])
        thr = float(np.quantile(p, 1 - y.mean()))
        pred = (p >= thr).astype(int)
        tp = float(((pred == 1) & (y == 1)).sum())
        res["models"][k] = {"auc": round(float(per[:, 0].mean()), 4),
                            "auc_sd": round(float(per[:, 0].std()), 4),
                            "ap": round(float(per[:, 1].mean()), 4),
                            "ap_sd": round(float(per[:, 1].std()), 4),
                            "f1": round(float(f1_score(y, pred)), 4),
                            "prec": round(tp / max(pred.sum(), 1), 4),
                            "rec": round(tp / max(float(y.sum()), 1), 4),
                            "thr": round(thr, 5), "sec": round(times[k], 1)}
        m = res["models"][k]
        log(f"{k:4s} AUC {m['auc']:.4f}±{m['auc_sd']:.4f} · PR-AUC {m['ap']:.4f} · "
            f"F1 {m['f1']:.3f} · {m['sec']:.0f}с")
    res["best"] = max(res["models"], key=lambda k: res["models"][k]["ap"])
    np.savez(os.path.join(CACHE, f"{coin}_oof_{target}.npz"), y=y, part=part, **oof)
    json.dump(res, open(os.path.join(CACHE, f"{coin}_clf_{target}.json"), "w"),
              ensure_ascii=False)
    log(f"{coin}/{target}: переможець за PR-AUC — {res['best']}")
    return res


# ============================ 7. ПРОДАКШЕН ============================
def holdout_blocks(n, frac=HOLD_FRAC, chunks=HOLD_CHUNKS, emb=HOLD_EMB, seed=SEED):
    """Відкладені 2% ЧОТИРМА суцільними шматками з ембарго — бо сусідні вікна
    корелюють, і розсипана вибірка дала б завищене число."""
    m = max(1, int(round(n * frac / chunks)))
    rng = np.random.default_rng(seed)
    hold = np.zeros(n, bool); ban = np.zeros(n, bool)
    placed = 0
    for _ in range(3000):
        if placed >= chunks:
            break
        a = int(rng.integers(0, n - m)); b = a + m
        if ban[max(0, a - emb):min(n, b + emb)].any():
            continue
        hold[a:b] = True; ban[max(0, a - emb):min(n, b + emb)] = True
        placed += 1
    return np.flatnonzero(hold), np.flatnonzero(~ban), placed


def stage_poles(coin, norm_days=28, mcs=100, sub_n=15000, with_null=True):
    """ПОЛЮСИ ПСА («тихе ПСА» = pcaquiet щоденника) — МІЙ незалежний перерахунок.
    Рецепт зі щоденника: 8×210 у разах від causal-норми 28 діб → лог → робастний
    масштаб (медіана й IQR ЛИШЕ з train) → PCA-32 → UMAP-10 (сусідів 30) →
    HDBSCAN(min_cluster_size=100). Полюси називаються за рівнем активності, а не
    за номером HDBSCAN. Сид мій (20260805), тож підвибірка, UMAP та ініціалізація
    інші, ніж в оригіналі — збіг результату буде результатом, а не спільним кодом.
    Прогрів норми (перші 28 діб) відкидається — там норми ще немає."""
    from sklearn.decomposition import PCA
    from sklearn.cluster import HDBSCAN
    S = ST.load(coin)
    if S is None:
        log("немає даних"); return None
    _, first = np.unique(S["ts"], return_index=True)
    first = np.sort(first)
    for k in ["ts", "price"] + FEATS:
        S[k] = S[k][first]
    meta0 = json.load(open(os.path.join(CACHE, f"{coin}_data.json")))
    t0, t1 = int(meta0["t0"]), int(meta0["t1"])
    N = t1 - t0 + 1
    pos = (S["ts"] - t0).astype(np.int64)
    keep_row = (pos >= 0) & (pos < N)
    pos = pos[keep_row]
    present = np.zeros(N, bool); present[pos] = True

    dd = np.load(os.path.join(CACHE, f"{coin}_data.npz"), mmap_mode="r")
    wts = np.asarray(dd["ts"]).astype(np.int64)
    train = np.asarray(dd["train"])
    NW = norm_days * 86400
    starts = (wts - t0).astype(np.int64)
    keep = starts >= NW
    st2 = starts[keep]
    idx = st2[:, None] + np.arange(WIN, dtype=np.int64)[None, :]
    A = np.empty((len(st2), 8, WIN), np.float32)
    gi = np.arange(N, dtype=np.float64)
    cnt = np.concatenate([[0], np.cumsum(present, dtype=np.int64)])
    nreal = (cnt[NW + 1:] - cnt[1:N - NW + 1]).astype(np.float64)
    for jf, f in enumerate(FEATS):
        v = np.interp(gi, pos, S[f][keep_row].astype(np.float64))
        c = np.concatenate([[0.0], np.cumsum(v * present)])
        nv = np.full(N, np.nan)
        with np.errstate(invalid="ignore", divide="ignore"):
            nv[NW:] = (c[NW + 1:] - c[1:N - NW + 1]) / np.where(nreal > 0, nreal, np.nan)
            r = (v / np.where(nv > 1e-12, nv, np.nan)).astype(np.float32)
        A[:, jf, :] = r[idx]
        del v, c, nv, r
        log(f"   {f}: нормовано на {norm_days} діб")
    del gi, idx
    ok = np.isfinite(A).all(axis=(1, 2))
    A = A[ok]
    wid = np.flatnonzero(keep)[ok]
    log(f"{coin}: вікон з нормою {len(A)} з {len(wts)} (прогрів {norm_days} діб геть)")

    tr_local = np.flatnonzero(np.isin(wid, train))
    F = np.clip(A, 1e-3, 1e4).astype(np.float32)
    act = np.log(F).mean(axis=(1, 2))
    np.log(F, out=F)
    v = F[tr_local].reshape(-1)
    med = float(np.median(v))
    iqr = float(np.subtract(*np.percentile(v, [75, 25]))) or 1.0
    F = ((F - med) / iqr).reshape(len(F), -1)
    del A, v

    rng = np.random.default_rng(SEED)
    sub = np.sort(rng.choice(tr_local, size=min(sub_n, len(tr_local)), replace=False))

    def pipeline(FF):
        import umap
        p = PCA(n_components=32, random_state=SEED).fit(FF[sub])
        Z = p.transform(FF).astype(np.float32)
        u = umap.UMAP(n_components=10, n_neighbors=30, min_dist=0.0,
                      metric="euclidean", random_state=SEED, verbose=False).fit(Z[sub])
        Zu = u.transform(Z).astype(np.float32)
        lb = HDBSCAN(min_cluster_size=int(mcs),
                     cluster_selection_method="eom").fit_predict(Zu)
        return lb, float(p.explained_variance_ratio_.sum())

    tc = time.time()
    lab, evr = pipeline(F)
    ks = sorted({int(c) for c in lab if c >= 0})
    log(f"{coin}: HDBSCAN дав {len(ks)} класів, шуму {(lab<0).mean()*100:.1f}% "
        f"(evr {evr:.3f}, {time.time()-tc:.0f}с)")
    out = {"coin": coin, "built": int(time.time()), "seed": SEED,
           "norm_days": norm_days, "mcs": mcs, "sub_n": int(len(sub)),
           "n_win_all": int(len(wts)), "n_win": int(len(F)),
           "evr": round(evr, 4), "k": len(ks),
           "noise": round(float((lab < 0).mean()), 4), "classes": []}
    if not ks:
        log("класів немає"); return out
    amed = {c: float(np.median(act[lab == c])) for c in ks}
    hi = max(amed, key=amed.get); lo = min(amed, key=amed.get)
    out["hi_id"] = int(hi); out["lo_id"] = int(lo)

    if with_null:
        # НУЛЬ-КОНТРОЛЬ: ті самі клітинки, перемішані МІЖ вікнами
        Fn = F.reshape(len(F), 8, WIN).copy()
        for jf in range(8):
            for tt in range(WIN):
                Fn[:, jf, tt] = Fn[rng.permutation(len(Fn)), jf, tt]
        ln, _ = pipeline(Fn.reshape(len(Fn), -1))
        out["null_k"] = len({int(c) for c in ln if c >= 0})
        out["null_noise"] = round(float((ln < 0).mean()), 4)
        log(f"{coin}: нуль-контроль — {out['null_k']} класів, шуму "
            f"{out['null_noise']*100:.1f}%")
        del Fn, ln

    rf = np.asarray(dd["rng_fwd"])
    day = ((wts - wts[0]) // 86400).astype(int)
    medd = np.array([np.median(rf[day == q]) if (day == q).sum() > 10 else np.nan
                     for q in range(day.max() + 1)])
    rel = rf / medd[day]
    base = float(np.median(rf))
    sc = np.load(os.path.join(CACHE, f"{coin}_feat.npz"))["scale"]
    G = np.expm1(np.log1p(np.asarray(dd["X"])[wid] / sc[None, :, None]).mean(axis=2))
    for c in ks:
        m = lab == c
        nm = "активний" if c == hi else ("тихий" if c == lo else f"проміжний {c}")
        out["classes"].append({
            "id": int(c), "name": nm, "n": int(m.sum()),
            "share": round(float(m.mean()), 4),
            "share_all": round(float(m.sum() / len(wts)), 4),
            "act": round(float(amed[c]), 3),
            "lvl": {f: round(float(np.median(G[m, i])), 4) for i, f in enumerate(FEATS)},
            "ratio": round(float(np.median(rf[wid][m]) / base), 3),
            "within_day": round(float(np.nanmedian(rel[wid][m])), 3)})
    out["classes"].sort(key=lambda x: -x["act"])
    full = np.full(len(wts), -2, np.int16)
    full[wid] = lab
    np.savez(os.path.join(CACHE, f"{coin}_poles.npz"), lab=full, wid=wid, act=act)
    json.dump(out, open(os.path.join(CACHE, f"{coin}_poles.json"), "w"),
              ensure_ascii=False)
    log(f"{coin}: полюси готові · активний {out['classes'][0]['share_all']*100:.1f}% "
        f"· тихий {out['classes'][-1]['share_all']*100:.1f}% від усіх вікон")
    return out


def stage_pq(coin, folds=10):
    """Детектор «тихого ПСА»: та сама схема, що для решти цілей (вхід — уся
    матриця 8×210 після log1p(x/0.01)), але ЛИШЕ на вікнах, де мітка є —
    прогрів 28-денної норми з задачі викинуто, як в оригіналі."""
    import torch
    from sklearn.metrics import roc_auc_score, average_precision_score, f1_score
    P = np.load(os.path.join(CACHE, f"{coin}_poles.npz"))
    meta = json.load(open(os.path.join(CACHE, f"{coin}_poles.json")))
    lab, wid = P["lab"], P["wid"]
    y = (lab[wid] == meta["lo_id"]).astype(np.int8)
    dd = np.load(os.path.join(CACHE, f"{coin}_data.npz"), mmap_mode="r")
    L = prep_fixed(np.asarray(dd["X"])[wid])
    F = L.reshape(len(L), -1)
    n = len(L)
    rng = np.random.default_rng(SEED)
    part = rng.permutation(n) % folds
    log(f"{coin}/pcaquiet: {n} вікон · ціль {int(y.sum())} ({y.mean()*100:.2f}%) · "
        f"{folds} частин · пристрій {_dev()}")
    oof = {"cnn": np.full(n, np.nan, np.float32), "gbm": np.full(n, np.nan, np.float32)}
    times = {"cnn": 0.0, "gbm": 0.0}
    for f in range(folds):
        va = np.flatnonzero(part == f); tr = np.flatnonzero(part != f)
        t = time.time()
        _, oof["gbm"][va] = fit_gbm(F, tr, y, seed=SEED + f, predict_idx=va)
        times["gbm"] += time.time() - t
        t = time.time()
        _, oof["cnn"][va] = fit_cnn(L, tr, y, seed=SEED + f, predict_idx=va)
        times["cnn"] += time.time() - t
        log(f"   фолд {f+1}/{folds}: " + " · ".join(
            f"{k} AUC {roc_auc_score(y[va], oof[k][va]):.4f}" for k in oof))
    res = {"coin": coin, "target": "pcaquiet", "folds": folds, "n": int(n),
           "pos": float(y.mean()), "prep": "fixed", "seed": SEED, "models": {}}
    for k, p in oof.items():
        per = np.array([[roc_auc_score(y[part == f], p[part == f]),
                         average_precision_score(y[part == f], p[part == f])]
                        for f in range(folds)])
        thr = float(np.quantile(p, 1 - y.mean()))
        pred = (p >= thr).astype(int)
        tp = float(((pred == 1) & (y == 1)).sum())
        res["models"][k] = {"auc": round(float(per[:, 0].mean()), 4),
                            "auc_sd": round(float(per[:, 0].std()), 4),
                            "ap": round(float(per[:, 1].mean()), 4),
                            "ap_sd": round(float(per[:, 1].std()), 4),
                            "f1": round(float(f1_score(y, pred)), 4),
                            "prec": round(tp / max(pred.sum(), 1), 4),
                            "rec": round(tp / max(float(y.sum()), 1), 4),
                            "thr": round(thr, 5), "sec": round(times[k], 1)}
        m = res["models"][k]
        log(f"{k:4s} AUC {m['auc']:.4f}±{m['auc_sd']:.4f} · PR-AUC {m['ap']:.4f} · "
            f"F1 {m['f1']:.3f} · {m['sec']:.0f}с")
    res["best"] = max(res["models"], key=lambda k: res["models"][k]["ap"])
    np.savez(os.path.join(CACHE, f"{coin}_oof_pcaquiet.npz"),
             y=y, part=part, wid=wid, **oof)
    json.dump(res, open(os.path.join(CACHE, f"{coin}_clf_pcaquiet.json"), "w"),
              ensure_ascii=False)

    # ПРОДАКШЕН: відкладені 2% чотирма суцільними шматками з ембарго
    os.makedirs(MODELS, exist_ok=True)
    hold, tr, placed = holdout_blocks(n)
    net, ps = fit_cnn(L, tr, y, seed=SEED)
    gb, pg = fit_gbm(F, tr, y, seed=SEED)
    thr = float(np.quantile(ps[tr], 1 - float(y[tr].mean())))
    thr_g = float(np.quantile(pg[tr], 1 - float(y[tr].mean())))
    torch.save(net.state_dict(), os.path.join(MODELS, f"{coin}_pcaquiet.pt"))
    pickle.dump(gb, open(os.path.join(MODELS, f"{coin}_pcaquiet_gbm.pkl"), "wb"))

    def hmet(p, t):
        yy, s = y[hold], p[hold]
        pr = s >= t
        tp = float((pr & (yy == 1)).sum())
        return {"auc": round(float(roc_auc_score(yy, s)), 4),
                "ap": round(float(average_precision_score(yy, s)), 4),
                "prec": round(tp / max(pr.sum(), 1), 4),
                "rec": round(tp / max(float(yy.sum()), 1), 4),
                "fires": round(float(pr.mean()), 4)}
    mt = {"coin": coin, "target": "pcaquiet", "built": int(time.time()), "seed": SEED,
          "prep": "fixed", "n": int(n), "n_train": int(len(tr)),
          "n_hold": int(len(hold)), "hold_chunks": int(placed),
          "pos": round(float(y.mean()), 4), "thr": round(thr, 5),
          "thr_gbm": round(thr_g, 5), "hold_cnn": hmet(ps, thr),
          "hold_gbm": hmet(pg, thr_g), "cv": res["models"]}
    json.dump(mt, open(os.path.join(MODELS, f"{coin}_pcaquiet.json"), "w"),
              ensure_ascii=False)
    np.savez(os.path.join(CACHE, f"{coin}_prod_pcaquiet.npz"),
             cnn=ps, gbm=pg, hold=hold, wid=wid)
    log(f"{coin}/pcaquiet: відкладені 2% — згортка AUC {mt['hold_cnn']['auc']} "
        f"PR {mt['hold_cnn']['ap']} · бустинг AUC {mt['hold_gbm']['auc']} "
        f"PR {mt['hold_gbm']['ap']}")
    return res


def stage_pqcmp(coin):
    """Моє «тихе ПСА» проти оригінального: спершу МІТКИ (два незалежні прогони
    UMAP+HDBSCAN), потім МОДЕЛІ (крос-валідаційні числа), потім розкладка по
    моїх чотирьох класах."""
    from sklearn.metrics import adjusted_rand_score, cohen_kappa_score
    mine = json.load(open(os.path.join(CACHE, f"{coin}_poles.json")))
    Pm = np.load(os.path.join(CACHE, f"{coin}_poles.npz"))
    ym_full = np.full(len(Pm["lab"]), -1, np.int8)
    ym_full[Pm["wid"]] = (Pm["lab"][Pm["wid"]] == mine["lo_id"]).astype(np.int8)
    out = {"coin": coin, "built": int(time.time()), "mine": mine}

    rp = os.path.join(REF_DIR, "cache", f"{coin}_pcapoles.json")
    if os.path.exists(rp):
        ref = json.load(open(rp))
        Pr = np.load(os.path.join(REF_DIR, "cache", f"{coin}_pcapoles.npz"))
        yr_full = np.full(len(Pr["lab"]), -1, np.int8)
        yr_full[Pr["wid"]] = (Pr["lab"][Pr["wid"]] == ref["lo_id"]).astype(np.int8)
        m = (ym_full >= 0) & (yr_full >= 0)
        a, b = ym_full[m] > 0, yr_full[m] > 0
        both = float((a & b).sum()); either = float((a | b).sum())
        out["ref"] = {k: ref.get(k) for k in ("k", "noise", "evr", "null_k",
                                              "null_noise", "norm_days")}
        out["labels"] = {
            "n_common": int(m.sum()),
            "share_mine": round(float(a.mean()), 4),
            "share_ref": round(float(b.mean()), 4),
            "agree": round(float((a == b).mean()), 4),
            "jaccard": round(both / max(either, 1), 4),
            "kappa": round(float(cohen_kappa_score(a, b)), 4),
            "ari": round(float(adjusted_rand_score(a, b)), 4),
            "prec": round(both / max(float(a.sum()), 1), 4),
            "rec": round(both / max(float(b.sum()), 1), 4)}
        log(f"{coin}: мітки — моя частка {a.mean()*100:.2f}% проти "
            f"{b.mean()*100:.2f}% · збіг {(a==b).mean()*100:.2f}% · "
            f"Жаккар {both/max(either,1):.3f} · ARI {out['labels']['ari']:.3f}")
        rc = os.path.join(REF_DIR, "cache", f"{coin}_clf_pcaquiet.json")
        if os.path.exists(rc):
            out["ref_cv"] = json.load(open(rc))["models"]

    mc = os.path.join(CACHE, f"{coin}_clf_pcaquiet.json")
    if os.path.exists(mc):
        out["my_cv"] = json.load(open(mc))["models"]
    mm = os.path.join(MODELS, f"{coin}_pcaquiet.json")
    if os.path.exists(mm):
        j = json.load(open(mm))
        out["hold"] = {"cnn": j["hold_cnn"], "gbm": j["hold_gbm"],
                       "n_hold": j["n_hold"], "thr": j["thr"], "thr_gbm": j["thr_gbm"]}

    # розкладка по моїх чотирьох класах
    Y = np.load(os.path.join(CACHE, f"{coin}_labels.npz"))
    m = ym_full >= 0
    q = ym_full > 0
    out["by_class"] = []
    for tg in ("storm", "mid", "calm_hi", "calm_lo"):
        c = Y[tg].astype(bool) & m
        out["by_class"].append({
            "target": tg, "share": round(float(c.sum() / m.sum()), 4),
            "p_quiet": round(float(q[c].mean()), 4),
            "lift": round(float(q[c].mean() / q[m].mean()), 3),
            "rec": round(float((q & c).sum() / max(q[m].sum(), 1)), 4),
            "jaccard": round(float((q & c).sum() / max((q | c)[m].sum(), 1)), 4)})
        z = out["by_class"][-1]
        log(f"   {tg:9s} частка {z['share']*100:5.1f}% · з них тихе ПСА "
            f"{z['p_quiet']*100:5.2f}% (×{z['lift']}) · ловить {z['rec']*100:4.1f}%")
    json.dump(out, open(os.path.join(OUT_DIR, f"{coin}_pq.json"), "w"),
              ensure_ascii=False)
    return out


def stage_final(coin, targets=("storm", "calm", "mid", "calm_hi", "calm_lo")):
    import torch
    from sklearn.metrics import roc_auc_score, average_precision_score
    os.makedirs(MODELS, exist_ok=True)
    dd = np.load(os.path.join(CACHE, f"{coin}_data.npz"), mmap_mode="r")
    L = prep_fixed(np.asarray(dd["X"]))
    F = L.reshape(len(L), -1)
    Y = np.load(os.path.join(CACHE, f"{coin}_labels.npz"))
    out = {}
    for tg in targets:
        y = Y[tg].astype(np.int8)
        n = len(y)
        hold, tr, placed = holdout_blocks(n)
        log(f"{coin}/{tg}: відкладено {len(hold)} ({len(hold)/n*100:.1f}%) "
            f"{placed} шматками · навчання {len(tr)}")
        net, ps = fit_cnn(L, tr, y, seed=SEED)
        gb, pg = fit_gbm(F, tr, y, seed=SEED)
        thr = float(np.quantile(ps[tr], 1 - float(y[tr].mean())))
        thr_g = float(np.quantile(pg[tr], 1 - float(y[tr].mean())))
        torch.save(net.state_dict(), os.path.join(MODELS, f"{coin}_{tg}.pt"))
        pickle.dump(gb, open(os.path.join(MODELS, f"{coin}_{tg}_gbm.pkl"), "wb"))

        def hmet(p, t):
            yy, sc = y[hold], p[hold]
            pr = sc >= t
            tp = float((pr & (yy == 1)).sum())
            return {"auc": round(float(roc_auc_score(yy, sc)), 4),
                    "ap": round(float(average_precision_score(yy, sc)), 4),
                    "prec": round(tp / max(pr.sum(), 1), 4),
                    "rec": round(tp / max(float(yy.sum()), 1), 4),
                    "fires": round(float(pr.mean()), 4)}
        cvp = os.path.join(CACHE, f"{coin}_clf_{tg}.json")
        cv = json.load(open(cvp)) if os.path.exists(cvp) else {}
        meta = {"coin": coin, "target": tg, "built": int(time.time()), "seed": SEED,
                "prep": "fixed", "prep_desc": "log1p(x/0.01)", "win": WIN,
                "n": int(n), "n_train": int(len(tr)), "n_hold": int(len(hold)),
                "hold_chunks": int(placed), "pos": round(float(y.mean()), 4),
                "thr": round(thr, 5), "thr_gbm": round(thr_g, 5),
                "hold_cnn": hmet(ps, thr), "hold_gbm": hmet(pg, thr_g),
                "cv": cv.get("models", {})}
        json.dump(meta, open(os.path.join(MODELS, f"{coin}_{tg}.json"), "w"),
                  ensure_ascii=False)
        np.savez(os.path.join(CACHE, f"{coin}_prod_{tg}.npz"),
                 cnn=ps, gbm=pg, hold=hold)
        log(f"{coin}/{tg}: відкладені 2% — згортка AUC {meta['hold_cnn']['auc']} "
            f"PR {meta['hold_cnn']['ap']} · бустинг AUC {meta['hold_gbm']['auc']} "
            f"PR {meta['hold_gbm']['ap']}")
        out[tg] = meta
    json.dump(out, open(os.path.join(CACHE, f"{coin}_final.json"), "w"), ensure_ascii=False)
    return out


def stage_scan(coin, targets=("storm", "calm", "mid", "calm_hi", "calm_lo")):
    """Продакшен-моделі по всій історії. Чесно: 98% вікон були в навчанні —
    це памʼять, а не прогноз; чесні числа — в oof (етап clf) і на відкладених 2%."""
    d = np.load(os.path.join(CACHE, f"{coin}_data.npz"), mmap_mode="r")
    ts = np.asarray(d["ts"]); px = np.asarray(d["px"]); rf = np.asarray(d["rng_fwd"])
    ser = {"coin": coin, "built": int(time.time()), "t0": int(ts[0]), "win": WIN,
           "k": [int(v) for v in (ts - ts[0]) // WIN],
           "px": [round(float(v), 1) for v in px],
           "fwd": [round(float(v), 3) for v in rf], "targets": {}}
    lab = np.load(os.path.join(CACHE, f"{coin}_labels.npz"))["lab"]
    ser["lab"] = [int(v) for v in lab]
    for tg in targets:
        p = os.path.join(CACHE, f"{coin}_prod_{tg}.npz")
        if not os.path.exists(p):
            continue
        z = np.load(p)
        meta = json.load(open(os.path.join(MODELS, f"{coin}_{tg}.json")))
        ser["targets"][tg] = {"thr": meta["thr"], "thr_gbm": meta["thr_gbm"],
                              "pos": meta["pos"], "hold_cnn": meta["hold_cnn"],
                              "hold_gbm": meta["hold_gbm"],
                              "cnn": [int(round(v * 100)) for v in z["cnn"]],
                              "gbm": [int(round(v * 100)) for v in z["gbm"]],
                              "hold": [int(v) for v in z["hold"]]}
    os.makedirs(OUT_DIR, exist_ok=True)
    json.dump(ser, open(os.path.join(OUT_DIR, f"{coin}.json"), "w"), ensure_ascii=False)
    log(f"{coin}: скан збережено ({len(ts)} вікон)")
    return True


# ============================ 8. ПОРІВНЯННЯ З ЛІНІЄЮ /wclass ============================
def _ref(coin):
    """Класи і моделі лінії /wclass, зіставлені за часом вікна."""
    rep = json.load(open(os.path.join(REF_DIR, f"{coin}.json")))
    sc = json.load(open(os.path.join(REF_DIR, "scan.json")))
    ser = rep["series"]
    ts = np.array(ser["t0"], np.int64) + np.array(ser["k"], np.int64) * ser["step"]
    lab = np.array(ser["lab"], np.int8)
    storm = int(rep["storm_cls"]); calm = [int(v) for v in rep["merge"]["calm_cls"]]
    mid = [k for k in range(4) if k != storm and k not in calm][0]
    # УВАГА: у scan.json поле step — це стрибок по вікнах при скані, а k — НОМЕР
    # вікна. Час вікна = t0 + k*WIN, а не t0 + k*step.
    ts_sc = np.array(sc["t0"], np.int64) + np.array(sc["k"], np.int64) * WIN
    tg = {}
    for name, key in (("storm", "storm"), ("calm", "calm"), ("mid", "mid")):
        if key in sc["targets"]:
            t = sc["targets"][key]
            tg[name] = {"ts": ts_sc, "cnn": np.array(t["cnn"], np.float32) / 100,
                        "gbm": np.array(t["gbm"], np.float32) / 100,
                        "thr": t["thr"], "thr_gbm": t["thr_gbm"]}
    oof = {}
    for name, sec in (("storm", "clf"), ("calm", "clf_calm"), ("mid", "clf_mid")):
        if sec in rep and rep[sec].get("oof", {}).get("p"):
            oof[name] = np.array(rep[sec]["oof"]["p"], np.float32) / 100
    return {"ts": ts, "lab": lab, "storm": storm, "calm": calm, "mid": mid,
            "tg": tg, "oof": oof, "rep": rep}


def _pair(a, b):
    """Метрики збігу двох бінарних сигналів."""
    from sklearn.metrics import cohen_kappa_score
    a = a.astype(bool); b = b.astype(bool)
    both = float((a & b).sum()); either = float((a | b).sum())
    return {"n": int(len(a)), "share_a": round(float(a.mean()), 4),
            "share_b": round(float(b.mean()), 4),
            "agree": round(float((a == b).mean()), 4),
            "jaccard": round(both / max(either, 1), 4),
            "prec_a_vs_b": round(both / max(float(a.sum()), 1), 4),
            "rec_a_vs_b": round(both / max(float(b.sum()), 1), 4),
            "kappa": round(float(cohen_kappa_score(a, b)), 4)}


def stage_compare(coin):
    from scipy.stats import spearmanr
    mine = json.load(open(os.path.join(OUT_DIR, f"{coin}.json")))
    R = _ref(coin)
    my_ts = np.array(mine["t0"], np.int64) + np.array(mine["k"], np.int64) * mine["win"]
    my_lab = np.array(mine["lab"], np.int8)
    my_lbl = np.load(os.path.join(CACHE, f"{coin}_labels.npz"))
    meta = json.load(open(os.path.join(CACHE, f"{coin}_labels.json")))
    A = load_all(coin)
    rf = A["rng_fwd"]
    day = ((my_ts - my_ts[0]) // 86400).astype(int)
    medd = np.array([np.median(rf[day == q]) if (day == q).sum() > 10 else np.nan
                     for q in range(day.max() + 1)])
    rel = rf / medd[day]

    ts_c, i_my, i_rf = np.intersect1d(my_ts, R["ts"], return_indices=True)
    log(f"{coin}: спільних вікон {len(ts_c)} з {len(my_ts)} моїх і {len(R['ts'])} їхніх")

    out = {"coin": coin, "built": int(time.time()), "n_common": int(len(ts_c)),
           "n_mine": int(len(my_ts)), "n_ref": int(len(R["ts"]))}

    # --- 1. класи проти класів ---
    from sklearn.metrics import adjusted_rand_score
    ml, rl = my_lab[i_my], R["lab"][i_rf]
    my_name = np.full(len(ml), 1, np.int8)      # 0 штиль · 1 клас 2 · 2 шторм
    my_name[np.isin(ml, meta["calm_cls"])] = 0
    my_name[ml == meta["storm_cls"]] = 2
    rf_name = np.full(len(rl), 1, np.int8)
    rf_name[np.isin(rl, R["calm"])] = 0
    rf_name[rl == R["storm"]] = 2
    out["classes"] = {
        "ari_raw": round(float(adjusted_rand_score(ml, rl)), 4),
        "ari_named": round(float(adjusted_rand_score(my_name, rf_name)), 4),
        "agree_named": round(float((my_name == rf_name).mean()), 4),
        "conf4": [[int(((ml == i) & (rl == j)).sum()) for j in range(4)] for i in range(4)],
        "conf3": [[int(((my_name == i) & (rf_name == j)).sum()) for j in range(3)]
                  for i in range(3)],
        "my_storm_cls": meta["storm_cls"], "ref_storm_cls": R["storm"],
        "my_shares": [round(float((my_name == i).mean()), 4) for i in range(3)],
        "ref_shares": [round(float((rf_name == i).mean()), 4) for i in range(3)]}
    log(f"  класи: ARI(сирі номери) {out['classes']['ari_raw']} · "
        f"ARI(за іменами) {out['classes']['ari_named']} · "
        f"збіг {out['classes']['agree_named']*100:.1f}%")

    # --- 2. моделі проти моделей і проти класів ---
    out["targets"] = {}
    for tg in ("storm", "calm", "mid"):
        if tg not in mine["targets"] or tg not in R["tg"]:
            continue
        M = mine["targets"][tg]
        mp = np.array(M["cnn"], np.float32)[i_my] / 100
        mg = np.array(M["gbm"], np.float32)[i_my] / 100
        _, j_ref, j_my = np.intersect1d(R["tg"][tg]["ts"], ts_c, return_indices=True)
        rp = np.zeros(len(ts_c), np.float32); rp[j_my] = R["tg"][tg]["cnn"][j_ref]
        rg = np.zeros(len(ts_c), np.float32); rg[j_my] = R["tg"][tg]["gbm"][j_ref]
        my_fire = mp >= M["thr"]; ref_fire = rp >= R["tg"][tg]["thr"]
        my_y = my_lbl[tg].astype(bool)[i_my]
        ref_y = (rf_name == {"calm": 0, "mid": 1, "storm": 2}[tg])
        d = {"prod_model_vs_model": _pair(my_fire, ref_fire),
             "prod_rho": round(float(spearmanr(mp, rp).statistic), 4),
             "my_model_vs_ref_class": _pair(my_fire, ref_y),
             "ref_model_vs_my_class": _pair(ref_fire, my_y),
             "my_model_vs_my_class": _pair(my_fire, my_y),
             "gbm_vs_gbm": _pair(mg >= M["thr_gbm"], rg >= R["tg"][tg]["thr_gbm"]),
             "hold_cnn": M["hold_cnn"], "hold_gbm": M["hold_gbm"]}
        # чесні out-of-fold прогнози обох ліній
        if tg in R["oof"]:
            oz = np.load(os.path.join(CACHE, f"{coin}_oof_{tg}.npz"))
            mo = oz["cnn"][i_my]
            ro = R["oof"][tg][i_rf]
            # поріг кожної лінії — на її ВЛАСНУ частоту цілі (як у щоденнику)
            tm = float(np.quantile(oz["cnn"], 1 - my_lbl[tg].mean()))
            tr_ = float(np.quantile(R["oof"][tg], 1 - ref_y.mean()))
            d["oof_model_vs_model"] = _pair(mo >= tm, ro >= tr_)
            d["oof_rho"] = round(float(spearmanr(mo, ro).statistic), 4)
        # де сигнали розходяться — що каже ціна
        cells = {"обидві": my_fire & ref_fire, "лише моя": my_fire & ~ref_fire,
                 "лише їхня": ~my_fire & ref_fire, "жодна": ~my_fire & ~ref_fire}
        d["cells"] = {k: {"share": round(float(v.mean()), 4),
                          "n": int(v.sum()),
                          "ratio": round(float(np.median(rf[i_my][v]) /
                                               np.median(rf)), 3) if v.sum() > 30 else None,
                          "within_day": round(float(np.nanmedian(rel[i_my][v])), 3)
                          if v.sum() > 30 else None}
                      for k, v in cells.items()}
        out["targets"][tg] = d
        log(f"  {tg}: моя проти їхньої (продакшен) збіг "
            f"{d['prod_model_vs_model']['agree']*100:.1f}% · Жаккар "
            f"{d['prod_model_vs_model']['jaccard']} · ρ {d['prod_rho']}")

    json.dump(out, open(os.path.join(OUT_DIR, f"{coin}_compare.json"), "w"),
              ensure_ascii=False)
    for lg in ("uk", "en"):
        charts(coin, out, mine, R, my_ts, i_my, i_rf, ts_c, my_name, rf_name, rf, rel, lg)
    return out


LANG = {
    "uk": {"my_cls": "мій клас «шторм»", "ref_cls": "клас «шторм» лінії /wclass",
           "share_day": "% вікон доби", "price": "ціна", "day": "доба датасету",
           "cls_title": "{c} · КЛАСИ: два незалежні прогони конвеєра (ARI {a}, збіг {g}%)",
           "mdl_title": "МОДЕЛІ: спрацювання на ціль «шторм» (збіг {g}%, Жаккар {j})",
           "my_mdl": "моя модель (Conv1D)", "ref_mdl": "модель /wclass (Conv1D)",
           "zoom_title": "{c} · {h} год історії: класи і моделі поруч, вікно 210 с",
           "my_p": "моя модель · P(шторм)", "ref_p": "модель /wclass · P(шторм)",
           "prob": "імовірність", "hours": "годин від початку ділянки",
           "rows": ["мій клас", "клас /wclass", "моя модель", "модель /wclass"],
           "cls_names": ["штиль", "клас 2", "шторм"],
           "ref_axis": "клас лінії /wclass", "my_axis": "мій клас",
           "conf_title": "класи: два незалежні прогони",
           "cells": ["обидві", "лише моя", "лише їхня", "жодна"],
           "share_win": "% вікон",
           "cells_title": "спрацювання «шторм»: збіг і розбіжність",
           "wd_axis": "розмах наступної години, × до медіани доби",
           "wd_title": "що каже ціна там, де моделі розходяться"},
    "en": {"my_cls": "my class \"storm\"", "ref_cls": "\"storm\" class of the /wclass line",
           "share_day": "% of windows in day", "price": "price", "day": "day of dataset",
           "cls_title": "{c} · CLASSES: two independent runs of the pipeline (ARI {a}, {g}% agreement)",
           "mdl_title": "MODELS: firing on the \"storm\" target ({g}% agreement, Jaccard {j})",
           "my_mdl": "my model (Conv1D)", "ref_mdl": "/wclass model (Conv1D)",
           "zoom_title": "{c} · {h} hours of history: classes and models side by side, 210 s window",
           "my_p": "my model · P(storm)", "ref_p": "/wclass model · P(storm)",
           "prob": "probability", "hours": "hours from start of the segment",
           "rows": ["my class", "/wclass class", "my model", "/wclass model"],
           "cls_names": ["calm", "class 2", "storm"],
           "ref_axis": "class of the /wclass line", "my_axis": "my class",
           "conf_title": "classes: two independent runs",
           "cells": ["both", "mine only", "theirs only", "neither"],
           "share_win": "% of windows",
           "cells_title": "\"storm\" firing: agreement and divergence",
           "wd_axis": "next-hour range, × median of the same day",
           "wd_title": "what price says where the models disagree"},
}


def charts(coin, out, mine, R, my_ts, i_my, i_rf, ts_c, my_name, rf_name, rfwd, rel,
           lang="uk"):
    T = LANG[lang]
    SFX = "" if lang == "uk" else "_en"
    import matplotlib
    matplotlib.use("Agg")
    import matplotlib.pyplot as plt
    from matplotlib.colors import ListedColormap
    FG, BG, GRID = "#e6edf3", "#0d1117", "#30363d"
    plt.rcParams.update({"figure.facecolor": BG, "axes.facecolor": BG,
                         "savefig.facecolor": BG, "text.color": FG,
                         "axes.labelcolor": FG, "xtick.color": FG, "ytick.color": FG,
                         "axes.edgecolor": GRID, "font.size": 9})
    C_MY, C_REF, C_PX = "#58a6ff", "#f85149", "#8b949e"
    tg = "storm"
    M = mine["targets"][tg]
    mp = np.array(M["cnn"], np.float32)[i_my] / 100
    _, j_ref, j_my = np.intersect1d(R["tg"][tg]["ts"], ts_c, return_indices=True)
    rp = np.zeros(len(ts_c), np.float32); rp[j_my] = R["tg"][tg]["cnn"][j_ref]
    my_fire = mp >= M["thr"]; ref_fire = rp >= R["tg"][tg]["thr"]
    px = np.array(mine["px"], np.float32)[i_my]
    day = ((ts_c - ts_c[0]) // 86400).astype(int)
    nd = day.max() + 1

    # ---- 1. вся історія: частка «шторму» за добу, чотири джерела ----
    def daily(v):
        return np.array([v[day == q].mean() if (day == q).sum() > 5 else np.nan
                         for q in range(nd)])
    fig, ax = plt.subplots(3, 1, figsize=(13, 8.4), sharex=True,
                           gridspec_kw={"height_ratios": [1.5, 1.5, 1]})
    ax[0].plot(daily(my_name == 2) * 100, color=C_MY, lw=1.4, label=T["my_cls"])
    ax[0].plot(daily(rf_name == 2) * 100, color=C_REF, lw=1.4, ls="--",
               label=T["ref_cls"])
    ax[0].set_ylabel(T["share_day"]); ax[0].legend(loc="upper left", framealpha=0.2)
    ax[0].set_title(T["cls_title"].format(
        c=coin, a=out["classes"]["ari_named"],
        g=f"{out['classes']['agree_named']*100:.1f}"), color=FG)
    ax[1].plot(daily(my_fire) * 100, color=C_MY, lw=1.4, label=T["my_mdl"])
    ax[1].plot(daily(ref_fire) * 100, color=C_REF, lw=1.4, ls="--",
               label=T["ref_mdl"])
    ax[1].set_ylabel(T["share_day"]); ax[1].legend(loc="upper left", framealpha=0.2)
    ax[1].set_title(T["mdl_title"].format(
        g=f"{out['targets']['storm']['prod_model_vs_model']['agree']*100:.1f}",
        j=out["targets"]["storm"]["prod_model_vs_model"]["jaccard"]), color=FG)
    ax[2].plot(np.array([np.nanmedian(px[day == q]) if (day == q).sum() > 5 else np.nan
                         for q in range(nd)]), color=C_PX, lw=1.1)
    ax[2].set_ylabel(T["price"]); ax[2].set_xlabel(T["day"])
    for a in ax:
        a.grid(alpha=0.15, color=GRID)
    fig.tight_layout()
    fig.savefig(os.path.join(OUT_DIR, f"{coin}_compare_history{SFX}.png"), dpi=110)
    plt.close(fig)

    # ---- 2. збільшення: смуги класів і моделей поруч ----
    # беремо ділянку, де є і збіги, і розбіжності
    w = 200                     # ~12 год: вікна по 210 с ще розрізняються оком
    score = np.convolve((my_fire | ref_fire).astype(float), np.ones(w) / w, "same") * \
        np.convolve((my_fire != ref_fire).astype(float), np.ones(w) / w, "same")
    c = int(np.argmax(score))
    a0, a1 = max(0, c - w // 2), min(len(ts_c), c + w // 2)
    sl = slice(a0, a1)
    x = (ts_c[sl] - ts_c[a0]) / 3600.0
    fig, ax = plt.subplots(3, 1, figsize=(13, 7.6), sharex=True,
                           gridspec_kw={"height_ratios": [1.2, 1.6, 1.4]})
    ax[0].plot(x, px[sl], color=C_PX, lw=1.1)
    ax[0].set_ylabel(T["price"]); ax[0].grid(alpha=0.15, color=GRID)
    ax[0].set_title(T["zoom_title"].format(
        c=coin, h=f"{(ts_c[a1-1]-ts_c[a0])/3600:.0f}"), color=FG)
    ax[1].plot(x, mp[sl], color=C_MY, lw=1.0, label=T["my_p"])
    ax[1].plot(x, rp[sl], color=C_REF, lw=1.0, ls="--", label=T["ref_p"])
    ax[1].axhline(M["thr"], color=C_MY, lw=0.7, alpha=0.5)
    ax[1].axhline(R["tg"][tg]["thr"], color=C_REF, lw=0.7, alpha=0.5)
    ax[1].set_ylabel(T["prob"]); ax[1].legend(loc="upper left", framealpha=0.2)
    ax[1].grid(alpha=0.15, color=GRID)
    rows = list(zip(T["rows"], [my_name[sl] == 2, rf_name[sl] == 2,
                                my_fire[sl], ref_fire[sl]],
                    [C_MY, C_REF, C_MY, C_REF]))
    for i, (nm, v, col) in enumerate(rows):
        yy = len(rows) - 1 - i
        ax[2].fill_between(x, yy + 0.12, yy + 0.88, where=v, color=col,
                           step="mid", lw=0)
        ax[2].text(x[0] - (x[-1] - x[0]) * 0.008, yy + 0.5, nm, ha="right",
                   va="center", fontsize=8.5, color=FG)
    ax[2].set_ylim(0, len(rows)); ax[2].set_yticks([])
    ax[2].set_xlabel(T["hours"])
    ax[2].set_xlim(x[0] - (x[-1] - x[0]) * 0.10, x[-1])
    ax[2].grid(alpha=0.15, color=GRID, axis="x")
    fig.tight_layout()
    fig.savefig(os.path.join(OUT_DIR, f"{coin}_compare_zoom{SFX}.png"), dpi=110)
    plt.close(fig)

    # ---- 3. матриця класів + що каже ціна там, де сигнали розходяться ----
    fig, ax = plt.subplots(1, 3, figsize=(13.5, 4.2))
    cm = np.array(out["classes"]["conf3"], float)
    cmn = cm / np.maximum(cm.sum(), 1) * 100
    im = ax[0].imshow(cmn, cmap=ListedColormap(
        plt.cm.magma(np.linspace(0.08, 0.95, 256))))
    nms = T["cls_names"]
    ax[0].set_xticks(range(3), nms); ax[0].set_yticks(range(3), nms)
    ax[0].set_xlabel(T["ref_axis"]); ax[0].set_ylabel(T["my_axis"])
    for i in range(3):
        for j in range(3):
            ax[0].text(j, i, f"{cmn[i,j]:.1f}%", ha="center", va="center",
                       color="#ffffff" if cmn[i, j] < cmn.max() * 0.6 else "#000000",
                       fontsize=9)
    ax[0].set_title(T["conf_title"], color=FG)
    fig.colorbar(im, ax=ax[0], fraction=0.046).ax.tick_params(colors=FG)

    keys = ["обидві", "лише моя", "лише їхня", "жодна"]
    cells = out["targets"]["storm"]["cells"]
    sh = [cells[k]["share"] * 100 for k in keys]
    ax[1].bar(range(4), sh, color=["#a371f7", C_MY, C_REF, "#3fb950"])
    ax[1].set_xticks(range(4), T["cells"], fontsize=8.5)
    ax[1].set_ylabel(T["share_win"])
    for i, v in enumerate(sh):
        ax[1].text(i, v, f"{v:.1f}%", ha="center", va="bottom", fontsize=8.5)
    ax[1].set_title(T["cells_title"], color=FG)
    ax[1].grid(alpha=0.15, color=GRID, axis="y")

    wd = [cells[k]["within_day"] or np.nan for k in keys]
    ax[2].bar(range(4), wd, color=["#a371f7", C_MY, C_REF, "#3fb950"])
    ax[2].axhline(1.0, color=FG, lw=0.8, ls=":")
    ax[2].set_xticks(range(4), T["cells"], fontsize=8.5)
    ax[2].set_ylabel(T["wd_axis"])
    for i, v in enumerate(wd):
        if np.isfinite(v):
            ax[2].text(i, v, f"×{v:.2f}", ha="center", va="bottom", fontsize=8.5)
    ax[2].set_title(T["wd_title"], color=FG)
    ax[2].grid(alpha=0.15, color=GRID, axis="y")
    fig.tight_layout()
    fig.savefig(os.path.join(OUT_DIR, f"{coin}_compare_cells{SFX}.png"), dpi=110)
    plt.close(fig)
    log(f"{coin}: графіки збережено в {OUT_DIR}")


# ====== 9. ЧОТИРИ КЛАСИ: видача моделі проти видачі алгоритму ======
FOUR = ["storm", "mid", "calm", "pcaquiet"]           # ЧОТИРИ продакшен-цілі лінії
FOUR_T = {
    "uk": {"storm": "шторм", "mid": "клас 2", "calm": "штиль",
           "pcaquiet": "тихе ПСА (pcaquiet)",
           "alg": "алгоритм (мітка)", "mdl": "модель",
           "y": "% вікон за 6 год", "x": "доба датасету",
           "title": "{c} · чотири цілі лінії: видача моделі проти видачі алгоритму",
           "pan": "{n} — {s} вікон · модель {e} · збіг {a} · Жаккар {j}",
           "cnn": "Conv1D", "gbm": "бустинг"},
    "en": {"storm": "storm", "mid": "class 2", "calm": "calm",
           "pcaquiet": "PCA-quiet (pcaquiet)",
           "alg": "algorithm (label)", "mdl": "model",
           "y": "% of windows per 6 h", "x": "day of dataset",
           "title": "{c} · the four targets of the line: model vs algorithm",
           "pan": "{n} — {s} of windows · model {e} · agreement {a} · Jaccard {j}",
           "cnn": "Conv1D", "gbm": "boosting"},
}


def stage_four(coin, bucket_h=6):
    """ЧОТИРИ ЦІЛІ ЛІНІЇ — storm · mid · calm · pcaquiet — по одній на панель,
    спільна горизонтальна шкала. Саме ці чотири тренує оригінал
    (final: storm, calm, mid, pcaquiet), тож панелі відповідають моделям, а не
    сирим класам k-середніх. На кожній панелі — зелене: що сказав алгоритм
    (мітка), червоне: що сказала навчена на ньому модель. Продакшен-моделі, тож
    98% вікон були в навчанні — це перевірка «чи модель відтворює розріз», а не
    прогноз. У pcaquiet перші 28 діб порожні: там ще немає норми."""
    import matplotlib
    matplotlib.use("Agg")
    import matplotlib.pyplot as plt
    Y = np.load(os.path.join(CACHE, f"{coin}_labels.npz"))
    meta = json.load(open(os.path.join(CACHE, f"{coin}_labels.json")))
    d = np.load(os.path.join(CACHE, f"{coin}_data.npz"), mmap_mode="r")
    ts = np.asarray(d["ts"])
    out = {"coin": coin, "built": int(time.time()), "bucket_h": bucket_h,
           "raw4": meta.get("raw4", {}), "classes": []}
    S = {}
    for tg in FOUR:
        p = os.path.join(CACHE, f"{coin}_prod_{tg}.npz")
        mp = os.path.join(MODELS, f"{coin}_{tg}.json")
        if not (os.path.exists(p) and os.path.exists(mp)):
            log(f"{coin}/{tg}: немає продакшен-моделі — пропускаю"); continue
        m = json.load(open(mp))
        # показуємо ПЕРЕМОЖЦЯ крос-валідації за PR-AUC, а не завжди згортку
        cvp = os.path.join(CACHE, f"{coin}_clf_{tg}.json")
        eng = json.load(open(cvp))["best"] if os.path.exists(cvp) else "cnn"
        z = np.load(p)
        pr_sub = z[eng] if eng in z else z["cnn"]
        thr = m["thr"] if eng == "cnn" else m["thr_gbm"]
        if tg == "pcaquiet":                 # ціль живе лише на вікнах з нормою
            P = np.load(os.path.join(CACHE, f"{coin}_poles.npz"))
            pm = json.load(open(os.path.join(CACHE, f"{coin}_poles.json")))
            wid = P["wid"]
            y = np.zeros(len(ts), bool); y[wid] = P["lab"][wid] == pm["lo_id"]
            fire = np.zeros(len(ts), bool); fire[wid] = pr_sub >= thr
            have = np.zeros(len(ts), bool); have[wid] = True
        else:
            y = Y[tg].astype(bool)
            fire = pr_sub >= thr
            have = np.ones(len(ts), bool)
        yh, fh = y[have], fire[have]
        both = float((yh & fh).sum()); either = float((yh | fh).sum())
        S[tg] = {"y": y, "fire": fire, "eng": eng, "have": have}
        out["classes"].append({
            "target": tg, "cls": meta.get("raw4", {}).get(tg),
            "n_win": int(have.sum()),
            "share_alg": round(float(yh.mean()), 4),
            "share_mdl": round(float(fh.mean()), 4),
            "agree": round(float((yh == fh).mean()), 4),
            "jaccard": round(both / max(either, 1), 4),
            "prec": round(both / max(float(fh.sum()), 1), 4),
            "rec": round(both / max(float(yh.sum()), 1), 4),
            "engine": eng,
            "hold_auc": m[f"hold_{eng}"]["auc"], "hold_ap": m[f"hold_{eng}"]["ap"]})
        log(f"  {tg}: алгоритм {yh.mean()*100:.1f}% · модель {fh.mean()*100:.1f}% · "
            f"збіг {(yh == fh).mean()*100:.1f}% · Жаккар {both/max(either,1):.3f}")
    if not S:
        return out

    b = ((ts - ts[0]) // (bucket_h * 3600)).astype(int)
    nb = b.max() + 1
    x = np.arange(nb) * bucket_h / 24.0
    cnt = np.bincount(b, minlength=nb).astype(float)
    ok = cnt > 5

    def series(v):
        s = np.bincount(b, weights=v.astype(float), minlength=nb) / np.maximum(cnt, 1) * 100
        s[~ok] = np.nan
        return s

    FG, BG, GRID = "#e6edf3", "#0d1117", "#30363d"
    C_ALG, C_MDL = "#3fb950", "#f85149"
    plt.rcParams.update({"figure.facecolor": BG, "axes.facecolor": BG,
                         "savefig.facecolor": BG, "text.color": FG,
                         "axes.labelcolor": FG, "xtick.color": FG, "ytick.color": FG,
                         "axes.edgecolor": GRID, "font.size": 9})
    for lang in ("uk", "en"):
        T = FOUR_T[lang]
        sfx = "" if lang == "uk" else "_en"
        fig, ax = plt.subplots(len(out["classes"]), 1, figsize=(13, 10.5), sharex=True)
        ax = np.atleast_1d(ax)
        for i, c in enumerate(out["classes"]):
            tg = c["target"]
            a = ax[i]
            hv = S[tg]["have"]
            cnt_h = np.bincount(b[hv], minlength=nb).astype(float)
            okh = cnt_h > 5

            def ser_h(v):
                w = np.bincount(b[hv], weights=v[hv].astype(float),
                                minlength=nb) / np.maximum(cnt_h, 1) * 100
                w[~okh] = np.nan
                return w
            sa, sm = ser_h(S[tg]["y"]), ser_h(S[tg]["fire"])
            a.fill_between(x, 0, sa, color=C_ALG, alpha=0.35, lw=0, label=T["alg"])
            a.plot(x, sa, color=C_ALG, lw=1.0)
            a.plot(x, sm, color=C_MDL, lw=1.0, ls="--",
                   label=f'{T["mdl"]}: {T[c["engine"]]}')
            a.set_ylabel(T["y"], fontsize=8)
            a.set_ylim(0, 100)
            a.grid(alpha=0.15, color=GRID)
            if i == 0:                      # легенда лише на верхній панелі
                a.legend(loc="upper right", framealpha=0.85, facecolor="#161b22",
                         edgecolor=GRID, fontsize=8, ncol=2)
            a.set_title(T["pan"].format(k=c["cls"], n=T[tg], e=T[c["engine"]],
                                        s=f"{c['share_alg']*100:.1f}%",
                                        a=f"{c['agree']*100:.1f}%",
                                        j=f"{c['jaccard']:.3f}"),
                        color=FG, fontsize=10, loc="left")
        ax[-1].set_xlabel(T["x"])
        ax[-1].set_xlim(x[0], x[-1])
        fig.suptitle(T["title"].format(c=coin), color=FG, fontsize=12)
        fig.tight_layout(rect=(0, 0, 1, 0.985))
        fig.savefig(os.path.join(OUT_DIR, f"{coin}_four{sfx}.png"), dpi=110)
        plt.close(fig)
    json.dump(out, open(os.path.join(OUT_DIR, f"{coin}_four.json"), "w"), ensure_ascii=False)
    log(f"{coin}: чотири класи — графік і числа збережено")
    return out


def stage_tab(coin):
    """Зводить усі маленькі звіти лінії в один файл для вкладки дашборда.
    Великий скан {COIN}.json сюди НЕ входить — він на десятки мегабайт."""
    def rd(p, d=None):
        return json.load(open(p)) if os.path.exists(p) else d
    out = {"coin": coin, "built": int(time.time()),
           "data": rd(os.path.join(CACHE, f"{coin}_data.json")),
           "probe": rd(os.path.join(CACHE, f"{coin}_probe.json")),
           "lrt": rd(os.path.join(CACHE, f"{coin}_lrt.json")),
           "labels": rd(os.path.join(CACHE, f"{coin}_labels.json")),
           "compare": rd(os.path.join(OUT_DIR, f"{coin}_compare.json")),
           "four": rd(os.path.join(OUT_DIR, f"{coin}_four.json")),
           "clf": {t: rd(os.path.join(CACHE, f"{coin}_clf_{t}.json"))
                   for t in ("storm", "calm", "mid", "calm_hi", "calm_lo")},
           "models": {t: rd(os.path.join(MODELS, f"{coin}_{t}.json"))
                      for t in ("storm", "calm", "mid", "calm_hi", "calm_lo")},
           "ref": {}}
    # те саме з лінії /wclass — щоб вкладка показувала обидва стовпці
    rp = rd(os.path.join(REF_DIR, f"{coin}.json"))
    if rp:
        out["ref"] = {"shares": rp.get("shares"), "storm_cls": rp.get("storm_cls"),
                      "fwd": rp.get("fwd"), "ladder_evr": rp.get("ladder", {}).get("series"),
                      "n_win": rp.get("n_win")}
        for t in ("storm", "calm", "mid"):
            m = rd(os.path.join(REF_DIR, "models", f"{coin}_{t}.json"))
            if m:
                out["ref"].setdefault("models", {})[t] = {
                    "thr": m["thr"], "pos": m["pos"], "hold_cnn": m["hold_cnn"],
                    "hold_gbm": m["hold_gbm"], "cv_ap": m.get("cv_ap"),
                    "cv_auc": m.get("cv_auc"), "cv_ap_gbm": m.get("cv_ap_gbm"),
                    "cv_auc_gbm": m.get("cv_auc_gbm")}
    pq = os.path.join(OUT_DIR, f"{coin}_pq.json")
    if os.path.exists(pq):
        out["pq"] = json.load(open(pq))
    json.dump(out, open(os.path.join(OUT_DIR, f"{coin}_tab.json"), "w"), ensure_ascii=False)
    log(f"{coin}: зведення для вкладки готове")
    return out


if __name__ == "__main__":
    cmd = sys.argv[1] if len(sys.argv) > 1 else "data"
    coin = sys.argv[2] if len(sys.argv) > 2 else "BTC"
    if cmd == "data":
        stage_data(coin)
    elif cmd == "feat":
        stage_feat(coin)
    elif cmd == "probe":
        stage_probe(coin)
    elif cmd == "lrt":
        stage_lrt(coin)
    elif cmd == "labels":
        stage_labels(coin)
    elif cmd == "clf":
        stage_clf(coin, sys.argv[3] if len(sys.argv) > 3 else "storm",
                  int(sys.argv[4]) if len(sys.argv) > 4 else 10)
    elif cmd == "final":
        stage_final(coin, tuple(sys.argv[3].split(","))) if len(sys.argv) > 3 \
            else stage_final(coin)
    elif cmd == "scan":
        stage_scan(coin)
    elif cmd == "compare":
        stage_compare(coin)
        stage_tab(coin)
    elif cmd == "poles":
        stage_poles(coin)
    elif cmd == "pq":
        stage_pq(coin, int(sys.argv[3]) if len(sys.argv) > 3 else 10)
        stage_pqcmp(coin); stage_tab(coin)
    elif cmd == "pqcmp":
        stage_pqcmp(coin); stage_tab(coin)
    elif cmd == "four":
        stage_four(coin)
        stage_tab(coin)
    elif cmd == "tab":
        stage_tab(coin)
    else:
        print(__doc__)
```

### Маршрути дашборда

`web/dashboard.py`:

```python
@app.route('/mywclass')
def mywclass(): return render_template('mywclass.html')  # НЕЗАЛЕЖНА репліка лінії /wclass за щоденником WCLASS_DIARY.md (свій сид, свій код, свої моделі) + порівняння з класами оригіналу

@app.route('/api/mywclass/<coin>')
def mywclass_coin(coin):
    if not re.fullmatch(r'[A-Za-z0-9]{1,10}', coin or ''):
        return jsonify(dict(error='погана монета')), 400
    p = os.path.join(ROOT_DIR, 'media', 'analyst', 'mywclass', f'{coin.upper()}_tab.json')
    if not os.path.exists(p):
        return jsonify(dict(error=f'нема зведення по {coin.upper()} — запусти python3 mywclass.py tab {coin.upper()}'))
    with open(p) as f:
        return app.response_class(f.read(), mimetype='application/json')
```

### Шаблон вкладки

`web/templates/mywclass.html`:

```html
<!doctype html><html lang="uk"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Незалежна репліка /wclass · market_dashboard</title>
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
.two{display:grid;grid-template-columns:1fr 1fr;gap:14px}
@media(max-width:900px){.two{grid-template-columns:1fr}}
</style></head><body>

<div class="bar"><a href="/">← хаб</a> <a href="/wclass">/wclass — оригінальна лінія</a> <a href="/events">/events</a> <a href="/states">/states</a></div>
<h1>🧪 Незалежна репліка /wclass — власні моделі проти чужих класів</h1>
<div class="mut" id="sub">завантаження…</div>

<div class="box verdict" id="verdict"></div>

<div class="box small">
  <b>Що це.</b> Щоденник <code>WCLASS_DIARY.md</code> заявлений як «виконуваний рецепт»: читаєш —
  отримуєш ті самі класи й моделі. Ця вкладка — перевірка заяви. Конвеєр написано ЗА ТЕКСТОМ щоденника
  (<code>mywclass.py</code>), з власним сидом (20260805 проти 20260804), власним блоковим сплітом,
  власною ініціалізацією мережі й власними файлами <code>media/analyst/mywclass/</code>.
  Спільні з оригіналом лише читач CSV і межа даних — інакше вікна поїхали б і поточкове порівняння
  «моя модель проти їхнього класу» стало б неможливим.
  <a href="/media/analyst/mywclass/MYWCLASS_DIARY.md" download>щоденник цієї лінії</a> ·
  <a href="/media/analyst/wclass/WCLASS_DIARY.md" download>щоденник оригіналу</a>
</div>

<h2>① Дані</h2>
<div class="kpi" id="kpi"></div>

<h2>② Чи відтворюються перевірки існування класів</h2>
<div class="mut small">Кожен критерій — разом зі своїм нуль-контролем, прогнаним через увесь конвеєр.
Жорсткий нуль: клітинки «фіча × секунда» перемішано МІЖ вікнами. Мʼякий: перемішано цілі ряди фіч.</div>
<table id="probe"></table>

<h2>③ Пастка щоденника: LRT та ICL</h2>
<div class="mut small">Обидва критерії поважні й стандартні. Тут вони непридатні — і побачити це можна лише
прогнавши їх по нуль-контролю: K=1 відкидається однаково на даних і на чистому шумі.</div>
<table id="lrt"></table>

<h2>④ Мої детектори: 10 фолдів × 2 движки</h2>
<div class="mut small">Вхід — уся матриця 8×210 сирих значень після <code>log1p(x/0.01)</code>; стала призначена
наперед, з даних не береться. Переможець — за PR-AUC. Спліт випадковий (як у щоденнику), тож числа є
ВЕРХНЬОЮ оцінкою: сусідні вікна корелюють.</div>
<table id="cv"></table>

<h2>⑤ Продакшен-моделі й відкладені 2%</h2>
<div class="mut small">Крос-валідація моделі не лишає — вона прилад. Продакшен-модель довчається окремо,
з чотирма суцільними шматками по 0.5% у відкладі. Стовпці «/wclass» — те саме в оригінальній лінії,
але на ІНШИХ відкладених шматках (сид інший), тож це не пряме порівняння двох чисел.</div>
<table id="hold"></table>

<h2>⑥ Мої класи проти класів /wclass</h2>
<div class="kpi" id="kpi2"></div>
<div class="two">
  <div><div class="small mut">матриця класів: рядки — мої, стовпці — /wclass, % усіх вікон</div>
    <table id="conf"></table></div>
  <div><div class="small mut">частки класів у двох прогонах</div><table id="shares"></table></div>
</div>

<h2>⑦ Мої моделі проти їхніх моделей</h2>
<table id="agree"></table>

<h2>⑧ Коли сигнали збігаються — що каже ціна</h2>
<div class="mut small">Розмах ціни в наступну годину, поділений на медіану ТОГО Ж ДНЯ (без цієї поправки
міряється епоха історії, а не стан ринку).</div>
<table id="cells"></table>
<div class="box small" id="cells_note"></div>

<h2>⑨ Тихе ПСА (pcaquiet) — четверта ціль лінії</h2>
<div class="box small verdict">
  <b>Це було пропущено, і це була помилка.</b> Оригінальна лінія тренує ЧОТИРИ продакшен-цілі —
  <code>storm</code>, <code>calm</code>, <code>mid</code> і <code>pcaquiet</code>. У першій версії репліки
  я свідомо викинув <code>pcaquiet</code> як «мітку чужої лінії», а коли треба було показати чотири класи —
  підставив замість неї розріз штилю навпіл. Це була підміна. З 06.08 «тихе ПСА» відтворено повністю:
  свій перерахунок полюсів (норма 28 діб → PCA-32 → UMAP-10 → HDBSCAN, мій сид), свій детектор,
  свої відкладені 2%.
</div>
<div class="mut small">Полюси шукаються на вікнах, де 28-денна норма вже є (перші 28 діб — прогрів, їх немає
в задачі). «Тихий» і «активний» називаються за рівнем активності вікна, а не за номером, який дав HDBSCAN.</div>
<table id="pq_poles"></table>
<div class="mut small" style="margin-top:12px"><b>Мітки двох незалежних прогонів</b> — тут UMAP+HDBSCAN,
а не k-середні, тож розбіжність очікувано більша, ніж на класах:</div>
<table id="pq_lab"></table>
<div class="mut small" style="margin-top:12px"><b>Детектори</b>: 10 фолдів (вхід — уся матриця 8×210 після
<code>log1p(x/0.01)</code>) і відкладені 2% чотирма суцільними шматками:</div>
<table id="pq_clf"></table>
<div class="mut small" style="margin-top:12px">Як «тихе ПСА» лягає на мої класи k-середніх:</div>
<table id="pq_cls"></table>

<h2>⑩ Чотири цілі лінії: видача моделі проти видачі алгоритму</h2>
<div class="mut small">Чотири панелі — це рівно ті чотири цілі, які тренує оригінальна лінія:
<code>storm</code> · <code>mid</code> · <code>calm</code> · <code>pcaquiet</code>. Спільна горизонтальна шкала. <span class="grow">Зелене — що сказав алгоритм</span> (мітка k-середніх),
<span class="fall">червоне — що сказала навчена на ньому модель</span> (переможець крос-валідації
за PR-AUC — движок підписано на кожній панелі, поріг на частоту цілі).
Точка = частка вікон класу за 6 годин. У «тихого ПСА» перші 28 діб порожні — там ще немає норми, на якій
його мітка визначена. Це перевірка «чи модель відтворює розріз», а не прогноз: 98% вікон були в її навчанні,
чесні числа — у розділах ④-⑤ і ⑨.</div>
<table id="four"></table>
<div class="gal"><img id="four_img" src="/media/analyst/mywclass/BTC_four.png" alt="чотири класи: модель проти алгоритму"></div>

<h2>⑪ Графіки порівняння з /wclass</h2>
<div class="gal">
  <img src="/media/analyst/mywclass/BTC_compare_history.png" alt="класи і моделі двох ліній на всій історії">
  <img src="/media/analyst/mywclass/BTC_compare_zoom.png" alt="12 годин: класи і моделі поруч">
  <img src="/media/analyst/mywclass/BTC_compare_cells.png" alt="матриця класів і поведінка ціни">
</div>

<script>
const F=(v,d)=>(v===null||v===undefined||!isFinite(v))?'—':Number(v).toFixed(d===undefined?2:d);
const PC=(v,d)=>(v===null||v===undefined||!isFinite(v))?'—':(v*100).toFixed(d===undefined?1:d)+'%';
const TN={storm:'шторм',calm:'штиль',mid:'клас 2'};
let D=null;

function kpis(){
  const d=D.data||{}, L=D.labels||{}, C=D.compare||{};
  document.getElementById('kpi').innerHTML=
    `<div>вікон 210 с<b>${d.n_win||'—'}</b><span class="mut small">${d.days||'—'} діб</span></div>`+
    `<div>навчання / валідація<b>${d.n_train||'—'} / ${d.n_val||'—'}</b><span class="mut small">блоки 2 год, ембарго 30 хв</span></div>`+
    `<div>відсутніх секунд<b>${F(d.missing_pct,2)}%</b><span class="mut small">лінійна інтерполяція + маска</span></div>`+
    `<div>мій сид<b>${d.seed||'—'}</b><span class="mut small">у /wclass 20260804</span></div>`+
    `<div>частка «шторму»<b>${PC((L.shares||[])[L.storm_cls])}</b><span class="mut small">у /wclass ${PC((D.ref&&D.ref.shares?D.ref.shares[D.ref.storm_cls]:null))}</span></div>`+
    `<div>вікон у порівнянні<b>${C.n_common||'—'}</b><span class="mut small">збіг сітки вікно-в-вікно</span></div>`;
}

function probe(){
  const p=D.probe; if(!p) return;
  let h='<tr><th class="l">перевірка</th><th>реальні дані</th><th>нуль-контроль</th><th class="l">що це означає</th></tr>';
  for(const rp of ['full','free','lev']){
    const s=p.spaces[rp]; if(!s) continue;
    const g=Math.max.apply(null,s.gap.map(x=>x.gap));
    const s4=s.sil.filter(x=>x.K===4)[0]||{};
    h+=`<tr><td class="l">простір <b>${rp}</b> · gap max</td><td>${F(g,4)}</td><td class="mut">0 за побудовою</td>`+
       `<td class="l mut">розбиття майже не стискає дані краще за шум</td></tr>`;
    h+=`<tr><td class="l">простір <b>${rp}</b> · силует K=4</td><td>${F(s4.data,4)}</td><td>${F(s4.null,4)}</td>`+
       `<td class="l ${(s4.data-s4.null)>0.02?'warn':'mut'}">${(s4.data-s4.null)>0.02?'вище нуля':'на рівні шуму'}</td></tr>`;
  }
  const l=p.ladder||{};
  h+=`<tr><td class="l">частка дисперсії у 16 осях (full)</td><td>${F(l.real,3)}</td><td>мʼякий ${F(l.soft,3)} · жорсткий ${F(l.hard,3)}</td>`+
     `<td class="l mut">структура у СКЛАДІ вікна; форма всередині додає ~1%</td></tr>`;
  for(const s of (p.silverman||[]))
    h+=`<tr><td class="l">Сільверман · ${s.axis}</td><td>p = ${F(s.p,2)}</td><td class="mut">h* = ${F(s.h_crit,3)}</td>`+
       `<td class="l ${s.p<0.05?'warn':'mut'}">${s.p<0.05?'мультимодальність':'другого горба немає'}</td></tr>`;
  for(const s of (p.stability||[]))
    h+=`<tr><td class="l">стійкість (дві повні перебудови) K=${s.K}</td><td>${F(s.ari,3)}</td><td>${F(s.ari_null,3)}</td>`+
       `<td class="l ${s.ari_null>=s.ari?'fall':'mut'}">${s.ari_null>=s.ari?'нуль стійкіший за дані':'дані стійкіші за нуль'}</td></tr>`;
  document.getElementById('probe').innerHTML=h;
}

function lrt(){
  const l=D.lrt; if(!l) return;
  let h='<tr><th class="l">набір</th><th>LR (K=1 проти K=2)</th><th>95% нуля</th><th>висновок</th><th>ICL мінімум</th><th>BIC мінімум</th></tr>';
  for(const k of Object.keys(l.sets)){
    const s=l.sets[k];
    h+=`<tr><td class="l">${k}</td><td>${F(s.lr_1vs2,1)}</td><td class="mut">${F(s.null_q95,1)}</td>`+
       `<td class="${s.reject_K1?'fall':'grow'}">${s.reject_K1?'K=1 відкинуто':'K=1 не відкинуто'}</td>`+
       `<td>K=${s.icl_argmin}</td><td>K=${s.bic_argmin}</td></tr>`;
  }
  document.getElementById('lrt').innerHTML=h;
}

function cv(){
  let h='<tr><th class="l">ціль</th><th>частка</th><th>Conv1D AUC</th><th>Conv1D PR-AUC</th><th>бустинг AUC</th>'+
        '<th>бустинг PR-AUC</th><th>переможець</th><th class="mut">/wclass PR-AUC (згортка)</th></tr>';
  for(const t of ['storm','calm','mid']){
    const c=(D.clf||{})[t]; if(!c) continue;
    const cn=c.models.cnn, gb=c.models.gbm;
    const rf=((D.ref||{}).models||{})[t]||{};
    h+=`<tr><td class="l">${TN[t]}</td><td>${PC(c.pos)}</td>`+
       `<td>${F(cn.auc,4)} <span class="mut">±${F(cn.auc_sd,4)}</span></td><td class="${c.best==='cnn'?'acc':''}">${F(cn.ap,4)}</td>`+
       `<td>${F(gb.auc,4)} <span class="mut">±${F(gb.auc_sd,4)}</span></td><td class="${c.best==='gbm'?'acc':''}">${F(gb.ap,4)}</td>`+
       `<td>${c.best==='cnn'?'Conv1D':'бустинг'}</td><td class="mut">${F(rf.cv_ap,4)}</td></tr>`;
  }
  document.getElementById('cv').innerHTML=h;
}

function hold(){
  let h='<tr><th class="l">ціль</th><th>мій поріг</th><th>мої 2%: AUC</th><th>PR-AUC</th><th>точність</th><th>повнота</th>'+
        '<th class="mut">/wclass поріг</th><th class="mut">їхні 2%: AUC</th><th class="mut">PR-AUC</th></tr>';
  for(const t of ['storm','calm','mid']){
    const m=(D.models||{})[t]; if(!m) continue;
    const hc=m.hold_cnn||{}, rf=((D.ref||{}).models||{})[t]||{}, rh=rf.hold_cnn||{};
    h+=`<tr><td class="l">${TN[t]}</td><td>${F(m.thr,3)}</td><td>${F(hc.auc,4)}</td><td>${F(hc.ap,4)}</td>`+
       `<td>${F(hc.prec,3)}</td><td>${F(hc.rec,3)}</td>`+
       `<td class="mut">${F(rf.thr,3)}</td><td class="mut">${F(rh.auc,4)}</td><td class="mut">${F(rh.ap,4)}</td></tr>`;
  }
  document.getElementById('hold').innerHTML=h;
}

function compare(){
  const C=D.compare; if(!C) return;
  const c=C.classes;
  document.getElementById('kpi2').innerHTML=
    `<div>ARI за іменами класів<b>${F(c.ari_named,3)}</b><span class="mut small">сирі номери ${F(c.ari_raw,3)}</span></div>`+
    `<div>збіг класів<b>${PC(c.agree_named)}</b><span class="mut small">вікно-в-вікно, ${C.n_common} вікон</span></div>`+
    `<div>мій «шторм»<b>${PC(c.my_shares[2])}</b><span class="mut small">у /wclass ${PC(c.ref_shares[2])}</span></div>`+
    `<div>збіг моделей на штормі<b>${PC(C.targets.storm.prod_model_vs_model.agree)}</b><span class="mut small">Жаккар ${F(C.targets.storm.prod_model_vs_model.jaccard,3)}</span></div>`;
  const nm=['штиль','клас 2','шторм'];
  let tot=0; c.conf3.forEach(r=>r.forEach(v=>tot+=v));
  let h='<tr><th class="l"></th>'+nm.map(x=>`<th>${x} /wclass</th>`).join('')+'</tr>';
  c.conf3.forEach((r,i)=>{
    h+=`<tr><td class="l">${nm[i]} (мій)</td>`+r.map((v,j)=>
      `<td class="${i===j?'acc':'mut'}">${PC(v/tot)}</td>`).join('')+'</tr>';
  });
  document.getElementById('conf').innerHTML=h;
  let s='<tr><th class="l">клас</th><th>мій прогін</th><th>/wclass</th></tr>';
  nm.forEach((n,i)=>{s+=`<tr><td class="l">${n}</td><td>${PC(c.my_shares[i])}</td><td>${PC(c.ref_shares[i])}</td></tr>`;});
  document.getElementById('shares').innerHTML=s;

  let a='<tr><th class="l">ціль</th><th>збіг (продакшен)</th><th>Жаккар</th><th>κ</th><th>ρ ймовірностей</th>'+
        '<th>збіг out-of-fold</th><th>моя модель проти ЇХНЬОГО класу</th><th>їхня модель проти МОГО класу</th></tr>';
  for(const t of ['storm','calm','mid']){
    const v=C.targets[t]; if(!v) continue;
    const p=v.prod_model_vs_model, o=v.oof_model_vs_model||{};
    a+=`<tr><td class="l">${TN[t]}</td><td class="acc">${PC(p.agree)}</td><td>${F(p.jaccard,3)}</td><td>${F(p.kappa,3)}</td>`+
       `<td>${F(v.prod_rho,3)}</td><td>${PC(o.agree)}</td>`+
       `<td>${PC(v.my_model_vs_ref_class.agree)} <span class="mut">Жаккар ${F(v.my_model_vs_ref_class.jaccard,3)}</span></td>`+
       `<td>${PC(v.ref_model_vs_my_class.agree)} <span class="mut">Жаккар ${F(v.ref_model_vs_my_class.jaccard,3)}</span></td></tr>`;
  }
  document.getElementById('agree').innerHTML=a;

  const order=['обидві','лише моя','лише їхня','жодна'];
  let z='<tr><th class="l">ціль</th>'+order.map(k=>`<th>${k}</th>`).join('')+'</tr>';
  for(const t of ['storm','calm','mid']){
    const v=C.targets[t]; if(!v) continue;
    z+=`<tr><td class="l">${TN[t]}</td>`+order.map(k=>{
      const x=v.cells[k]||{};
      const cl=(x.within_day>1.1)?'fall':((x.within_day<0.9)?'grow':'');
      return `<td class="${cl}">×${F(x.within_day)} <span class="mut">(${PC(x.share)})</span></td>`;
    }).join('')+'</tr>';
  }
  document.getElementById('cells').innerHTML=z;
  const st=C.targets.storm.cells;
  document.getElementById('cells_note').innerHTML=
    `Там, де на «шторм» спрацьовують <b>обидві незалежні моделі</b> (${PC(st['обидві'].share)} часу), розмах наступної `+
    `години <b class="fall">×${F(st['обидві'].within_day)}</b> до медіани того ж дня. Де спрацювала лише моя — `+
    `×${F(st['лише моя'].within_day)}, лише їхня — ×${F(st['лише їхня'].within_day)}, жодна — ×${F(st['жодна'].within_day)}. `+
    `Тобто <b>перевагу дає згода двох моделей</b>, а розбіжність її майже повністю зʼїдає.`;
}

function four(){
  const f=D.four;
  const el=document.getElementById('four');
  if(!f||!f.classes||!f.classes.length){
    el.innerHTML='<tr><td class="l mut">детекторів на всі 4 цілі ще немає — '+
      '<code>python3 mywclass.py poles BTC</code> → <code>pq BTC</code> → '+
      '<code>four BTC</code></td></tr>';
    document.getElementById('four_img').style.display='none'; return;
  }
  const NM={storm:'шторм',mid:'клас 2',calm:'штиль',pcaquiet:'тихе ПСА (pcaquiet)'};
  let h='<tr><th class="l">клас</th><th>алгоритм</th><th>модель</th><th>збіг</th><th>Жаккар</th>'+
        '<th>движок</th><th>точність</th><th>повнота</th><th>відкладені 2%: AUC</th><th>PR-AUC</th></tr>';
  for(const c of f.classes){
    h+=`<tr><td class="l">${NM[c.target]||c.target}`+
       (c.n_win&&c.n_win<50000?` <span class="mut small">(${c.n_win} вікон)</span>`:'')+`</td>`+
       `<td class="grow">${PC(c.share_alg)}</td><td class="fall">${PC(c.share_mdl)}</td>`+
       `<td class="acc">${PC(c.agree)}</td><td>${F(c.jaccard,3)}</td>`+
       `<td>${c.engine==='gbm'?'бустинг':'Conv1D'}</td>`+
       `<td>${F(c.prec,3)}</td><td>${F(c.rec,3)}</td>`+
       `<td>${F(c.hold_auc,4)}</td><td>${F(c.hold_ap,4)}</td></tr>`;
  }
  el.innerHTML=h;
}


function pq(){
  const Q=D.pq; if(!Q) return;
  const m=Q.mine||{}, r=Q.ref||{};
  let h='<tr><th class="l">пошук полюсів</th><th>моя лінія</th><th>/wclass</th><th class="l">що це</th></tr>';
  const rows=[['класів HDBSCAN','k','k','скільки полюсів знайшлось',0],
              ['шум','noise','noise','частка вікон поза класами',1],
              ['evr PCA-32','evr','evr','частка дисперсії у 32 осях',1],
              ['НУЛЬ: класів','null_k','null_k','те саме на перемішаних клітинках',0],
              ['НУЛЬ: шум','null_noise','null_noise','має бути ≈100%',1]];
  for(const [nm,a,b,d,isPct] of rows){
    const va=m[a], vb=r[b];
    const f=(v)=>v===undefined?'—':(isPct?PC(v,1):v);
    h+=`<tr><td class="l">${nm}</td><td>${f(va)}</td><td>${f(vb)}</td><td class="l mut">${d}</td></tr>`;
  }
  const cl=(m.classes||[]);
  const q=cl.length?cl[cl.length-1]:null;
  if(q) h+=`<tr><td class="l"><b>тихий полюс</b></td><td class="warn">${PC(q.share_all,2)}</td>`+
           `<td class="mut">7.08%</td><td class="l mut">частка від УСІХ вікон · розмах наступної години `+
           `×${F(q.ratio)} (усередині доби ×${F(q.within_day)})</td></tr>`;
  document.getElementById('pq_poles').innerHTML=h;

  const L=Q.labels||{};
  h='<tr><th class="l">мітки</th><th>значення</th><th class="l">для порівняння</th></tr>'+
    `<tr><td class="l">частка «тихого» у мене / в них</td><td>${PC(L.share_mine,2)} / ${PC(L.share_ref,2)}</td>`+
    `<td class="l mut">незалежні прогони на тих самих вікнах</td></tr>`+
    `<tr><td class="l">збіг вікно-в-вікно</td><td class="warn">${PC(L.agree,2)}</td>`+
    `<td class="l mut">на класах k-середніх було 93.0%</td></tr>`+
    `<tr><td class="l">Жаккар</td><td>${F(L.jaccard,3)}</td>`+
    `<td class="l mut">на детекторі «шторм» двох ліній 0.856</td></tr>`+
    `<tr><td class="l">κ Коена · ARI</td><td>${F(L.kappa,3)} · ${F(L.ari,3)}</td>`+
    `<td class="l mut">UMAP+HDBSCAN нестійкіший за k-середні</td></tr>`+
    `<tr><td class="l">точність / повнота проти їхньої мітки</td><td>${F(L.prec,3)} / ${F(L.rec,3)}</td>`+
    `<td class="l mut">${L.n_common} спільних вікон</td></tr>`;
  document.getElementById('pq_lab').innerHTML=h;

  h='<tr><th class="l">движок</th><th>моя AUC</th><th>/wclass AUC</th><th>моя PR-AUC</th>'+
    '<th>/wclass PR-AUC</th><th>мої відкладені 2%: AUC / PR</th></tr>';
  const my=Q.my_cv||{}, rf=Q.ref_cv||{}, hd=Q.hold||{};
  for(const k of ['cnn','gbm']){
    const a=my[k]||{}, b=rf[k]||{}, c=hd[k]||{};
    h+=`<tr><td class="l">${k==='cnn'?'згортка 8×210':'бустинг на 1680'}</td>`+
       `<td class="${a.auc>=b.auc?'grow':''}">${F(a.auc,4)}</td><td class="mut">${F(b.auc,4)}</td>`+
       `<td class="${a.ap>=b.ap?'grow':''}">${F(a.ap,4)}</td><td class="mut">${F(b.ap,4)}</td>`+
       `<td>${F(c.auc,4)} / ${F(c.ap,4)}</td></tr>`;
  }
  document.getElementById('pq_clf').innerHTML=h;

  const TT={storm:'шторм',mid:'клас 2',calm_hi:'штиль-верх',calm_lo:'штиль-низ'};
  h='<tr><th class="l">мій клас</th><th>частка</th><th>з них тихе ПСА</th><th>× до бази</th>'+
    '<th>ловить тихого</th><th>Жаккар</th></tr>';
  for(const z of (Q.by_class||[]))
    h+=`<tr><td class="l">${TT[z.target]||z.target}</td><td>${PC(z.share)}</td>`+
       `<td>${PC(z.p_quiet,2)}</td><td class="${z.lift<0.5?'fall':'mut'}">×${F(z.lift)}</td>`+
       `<td>${PC(z.rec,1)}</td><td class="mut">${F(z.jaccard,3)}</td></tr>`;
  h+='<tr><td class="l mut" colspan="6">Тихе ПСА — окрема мітка, а не котрийсь із класів: '+
     'воно розмазане по трьох тихих класах приблизно рівно і не перетинається зі штормом.</td></tr>';
  document.getElementById('pq_cls').innerHTML=h;
}

function verdict(){
  const C=D.compare, L=D.labels, p=D.probe;
  const st=(L&&L.targets)?L.targets.storm:null;
  const k2=(p&&p.stability)?p.stability.filter(x=>x.K===4)[0]:null;
  document.getElementById('sub').textContent=
    `${D.coin} · незалежний прогін конвеєра ${new Date((D.built||0)*1000).toLocaleString('uk')} · `+
    `${(D.data||{}).n_win||'—'} вікон по 210 с`;
  document.getElementById('verdict').innerHTML=
    `<b>Рецепт відтворюється.</b> Другий незалежний прогін дав ті самі класи (збіг `+
    `<b>${PC(C?C.classes.agree_named:null)}</b>, ARI ${F(C?C.classes.ari_named:null,3)}) і ті самі моделі: `+
    `спрацювання детекторів «шторм» двох ліній збігаються на <b>${PC(C?C.targets.storm.prod_model_vs_model.agree:null)}</b> вікон. `+
    `Вердикт щоденника теж відтворився: дискретних класів немає — при K=4 стійкість `+
    `${F(k2?k2.ari:null,2)} при нулі ${F(k2?k2.ari_null:null,2)}, тобто розбиття не стійкіше за шум. `+
    `Єдине, що пережило поправку «всередині доби», — «шторм»: розмах наступної години `+
    `<b class="fall">×${F(st?st.within_day:null)}</b> (абсолютно ×${F(st?st.ratio:null)}).`;
}

fetch('/api/mywclass/BTC').then(r=>r.json()).then(j=>{
  if(j.error){document.getElementById('sub').textContent=j.error;return;}
  D=j; verdict(); kpis(); probe(); lrt(); cv(); hold(); compare(); pq(); four();
}).catch(e=>{document.getElementById('sub').textContent='помилка завантаження: '+e;});
</script>
</body></html>
```

### Запис у сторожа вкладок

`test_pages.js`:

```javascript
{file: 'mywclass.html', min_draw: 0,
   routes: [['/api/mywclass/', 'media/analyst/mywclass/{C}_tab.json']]},
```
