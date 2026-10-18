# Стек и каркас

## Структура
```
project/
  server/            # Node (Express 5, ESM) — API, кэш внешних сервисов, раздача web/dist
  web/               # фронтенд (Vite + React + TS)
    DESIGN.md        # дизайн-система — источник правды для агентов
    src/
      components/ui/ # shadcn
      components/    # UI-кит продукта (media.tsx, layout.tsx, ...)
      lib/           # api, хуки данных, хранилища, форматирование, wait.ts
      pages/         # страницы (lazy)
  shared/            # правила, общие для сервера и клиента (если есть)
  tests/             # node:test
  CLAUDE.md
```
Если сервера нет (только фронтенд + публичные API) — всё равно заведи прослойку кэша (edge-функция/прокси), когда
у API есть лимиты.

## Команды создания фронтенда
`shadcn create` в папке, где уже есть package.json, падает («не найден фреймворк») — создавай Vite вручную:
```bash
npm create -y vite@latest web -- --template react-ts --no-interactive
cd web && npm install && npm install tailwindcss @tailwindcss/vite @types/node
```
`vite.config.ts`:
```ts
import path from 'node:path'
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'
const API = 'http://localhost:3210'
export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: { alias: { '@': path.resolve(__dirname, './src') } },
  server: { proxy: { '/api': API } }
})
```
В `tsconfig.json` и `tsconfig.app.json`: `compilerOptions.paths = { "@/*": ["./src/*"] }`. В `src/index.css` —
`@import "tailwindcss";`. Затем (неинтерактивно, иначе CLI ждёт выбора пресета):
```bash
npx -y shadcn@latest init -b radix -p nova --pointer -y < /dev/null
npx -y shadcn@latest add button badge card dialog drawer dropdown-menu input select tabs toggle-group tooltip \
  skeleton slider scroll-area separator sonner switch progress label radio-group checkbox sheet popover command -y < /dev/null
npm install react-router-dom @tanstack/react-query @tanstack/react-query-persist-client @tanstack/query-sync-storage-persister
npm install @fontsource-variable/<шрифт-текста> @fontsource/<шрифт-заголовков>
```
После `init` замени Geist (дефолт) на выбранные шрифты и перепиши токены — дефолтная серая тема shadcn = «шаблон».

## Обязательные настройки
- `main.tsx`: `PersistQueryClientProvider` (см. data-and-caching.md), `TooltipProvider`, `Toaster`
  (на телефоне `mobileOffset` над таб-баром), `ThemeProvider` (если одна тема — `forcedTheme`).
- `App.tsx`: страницы через `lazy()` + `Suspense`; плеер/полноэкранные режимы — вне `Layout`.
- Диалоги/шторки: русские `sr-only` подписи кнопок закрытия (в shadcn по умолчанию «Close»).
- Сборка: `tsc -b && vite build`; агенты проверяют `npx tsc --noEmit -p tsconfig.app.json`, сборку делает лид.

## Деплой (если нужен)
- systemd-сервис от отдельного пользователя, nginx (для видео/стримов: `proxy_buffering off`, длинные таймауты),
  certbot, ufw (только 22/80/443 + нужное), доступ к приватному репо — read-only deploy key.
- Скрипт `deploy/setup.sh` идемпотентный: права на папку ДО `npm install`, `git safe.directory`, сборка, сервис,
  nginx, сертификат, файрвол. Всё, что правилось на сервере руками, переноси в скрипт — иначе потеряется.
- До публикации — вход по паролю/авторизация.
- Россия: блокировки зарубежных API и трекеров, фильтрация TLS у региональных провайдеров; проверяй доступность
  через check-host.net и предусматривай запасной вход (другой порт/прокси/российская «входная дверь»).
