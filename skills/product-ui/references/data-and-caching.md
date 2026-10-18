# Данные, кэш, оптимистичные обновления, ожидание

Цель: повторные визиты мгновенные, лимиты внешних API не «высушиваются», действия пользователя отражаются сразу.

## Три уровня кэша
1. **Сервер, диск (stale-while-revalidate)** — для любых внешних API, особенно с лимитами.
```js
// server/cache.js
const store = loadJson(FILE) // { key: { t, v } }
const inflight = new Map()
function run (key, fn) {
  if (!inflight.has(key)) inflight.set(key, Promise.resolve().then(fn).then(v => { store[key] = { t: Date.now(), v }; persistSoon(); return v }).finally(() => inflight.delete(key)))
  return inflight.get(key)
}
export async function cached (key, ttl, fn) {
  const e = store[key]
  if (e && Date.now() - e.t < ttl) return e.v            // свежее
  if (e) { run(key, fn).catch(() => {}); return e.v }     // устарело: отдаём сразу, обновляем в фоне
  return run(key, fn)                                      // нет — ждём (одновременные запросы склеиваются)
}
```
   TTL по смыслу: главная/подборки 6 ч, карточка 7 дн., поиск 1 дн., быстро меняющееся 10–30 мин. Ограничь размер
   (LRU ~4000 записей). При ошибке/исчерпании лимита — продолжаем отдавать последнюю версию. «Не найдено» тоже
   кэшируй (негативный кэш). Один «богатый» ответ (append_to_response и т.п.) переиспользуй для нескольких экранов.
   Для API с посекундным лимитом — ограничитель (≤N параллельно, интервал между стартами, ретрай на 429).
2. **Браузер, react-query с сохранением в localStorage** — только тяжёлые и редко меняющиеся ключи:
```ts
const persister = createSyncStoragePersister({ storage: localStorage, key: 'app:query-cache', throttleTime: 30_000, retry: removeOldestQuery })
const PERSIST = new Set(['home', 'title', 'seasons', 'genres', 'collections', 'person'])
<PersistQueryClientProvider client={qc} persistOptions={{ persister, maxAge: 3 * 24 * 3600_000, buster: 'v1',
  dehydrateOptions: { shouldDehydrateQuery: q => q.state.status === 'success' && PERSIST.has(String(q.queryKey[0])) } }}>
```
   Не сохраняй поиск, бесконечные списки, часто опрашиваемые ключи (опрос каждые 2–5 с иначе пересериализует весь
   кэш). `refetchOnWindowFocus: false`, `gcTime` ≥ maxAge. Меняешь формат данных — меняй `buster`.
3. **HTTP-заголовки** (`Cache-Control: private, max-age=…`) на ответы API и `immutable` на статику/картинки.

## Хуки данных
- `staleTime` по типу (часы для каталога, минуты для поиска); `placeholderData` = предыдущие данные **только в пределах
  того же раздела** (`(prev, prevQuery) => sameSection ? prev : undefined`), иначе под заголовком «Сериалы» мелькают
  фильмы; затемняй по `isPlaceholderData`.
- Поиск: debounce ≥ 600 мс, минимум 3 символа, `?q=` в URL с `replace`, недавние запросы в localStorage.
- Предзагрузка по наведению: только `pointerType === 'mouse'`, задержка ~700 мс, бюджет ~15 за сессию, не грузить то,
  что уже в кэше; обязательно `onPointerLeave` отменяет таймер.
- Фильтры каталога — в URL (`?genre=&years=&sort=`), назад/обновление восстанавливают состояние.

## Оптимистичные обновления
- Локальные списки (избранное, «буду смотреть», история): модуль-хранилище + `useSyncExternalStore`, синхронизация
  между вкладками по событию `storage`. Действие сразу меняет UI и показывает тост с «Отменить»/«Вернуть» (возвращает
  ровно удалённый элемент, а не снимок всего списка).
- Серверные действия: `useMutation({ mutationKey: ['entity'], onMutate: snapshot + setQueryData, onError: rollback +
  человеческий тост, onSettled: invalidate })`; общий хук опроса этой сущности приостанавливает `refetchInterval`,
  пока `useIsMutating({ mutationKey }) > 0` — иначе опрос перетирает мгновенное изменение.
- Сохранение настроек: тост сразу, при ошибке он меняется на ошибку, введённое остаётся в форме; инвалидировать
  только то, что реально зависит от изменённого поля.

## Живое ожидание (`lib/wait.ts`)
```ts
export function useElapsed (running: boolean, resetKey?: unknown) {
  const [sec, setSec] = useState(0)
  useEffect(() => { setSec(0); if (!running) return
    const started = Date.now(); const id = setInterval(() => setSec(Math.floor((Date.now() - started) / 1000)), 1000)
    return () => clearInterval(id) }, [running, resetKey])
  return sec
}
// Подсказка становится подробнее со временем
export const stepHint = (sec: number) =>
  sec < 6 ? '' : sec < 20 ? 'Пояснение, что именно происходит' : sec < 45 ? 'Ещё немного, почти готово' : 'Что можно сделать: подождать или выбрать другой вариант'
```
- В многошаговых процессах — список шагов с галочками, активный шаг со спиннером и счётчиком секунд после 3 с,
  `aria-live="polite"` на подсказке.
- Перегрузка (503) — не ошибка: «Сейчас много желающих. Запустим, как только освободится место — окно можно не
  закрывать» + автоповтор каждые ~8 с (с отменой и общим тайм-аутом).
- Долгие операции отменяемы (`AbortController`), уход со страницы отменяет запуск (`useEffect(() => () => abort())`).

## Серверные лимиты ресурсов
Тяжёлые операции (перекодирование, большие загрузки) — через «ворота» `createGate(max)` с ответом 503 и понятным текстом;
клиент превращает 503 в очередь. Буферы/кэши крупных данных — на диске с LRU и потолком от свободного места, не в ОЗУ.
