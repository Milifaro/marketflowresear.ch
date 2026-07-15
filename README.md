# marketflowresear.ch — static landing (GitHub Pages)

Односторінковий статичний сайт бренду **Market Flow Research**.
Хоститься на GitHub Pages, домен `marketflowresear.ch` (GoDaddy → DNS на GitHub).

## Файли
- `index.html` — уся сторінка (стилі inline, без збірки).
- `articles/<slug>/index.html` — статті (чистий URL `/articles/<slug>/`).
- `og.png` — Open Graph зображення 1200×630 (спільне для всіх сторінок).
- `sitemap.xml` — **при додаванні статті дописати її URL сюди** (+ оновити lastmod).
- `robots.txt` — дозволяє все, вказує на sitemap.
- `CNAME` — кастомний домен для GitHub Pages (`marketflowresear.ch`).
- `.nojekyll` — вимикає обробку Jekyll (віддає файли як є).

## SEO-стандарт сторінки (обовʼязково для кожної нової статті)
canonical · title · meta description · og:type=article + og:image (og.png) ·
twitter:card · article:published_time · JSON-LD Article · запис у sitemap.xml ·
alt у всіх зображень · футер: AI-badge + © Market Flow Research.

## Деплой (коротко)
1. Створити публічний репозиторій на GitHub, залити вміст цієї папки в корінь.
2. Settings → Pages → Deploy from branch → `main` / root.
3. Settings → Pages → Custom domain → `marketflowresear.ch` → Enforce HTTPS.
4. GoDaddy DNS (apex-домен) → A-записи на GitHub Pages:
   `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`
   (+ `www` CNAME → `<username>.github.io`).

## TODO
- Підставити фінальні соц-хендли у блоці `nav.links` (`index.html`), коли визначимось.
