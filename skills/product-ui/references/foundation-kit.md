# Фундамент: токены и UI-кит (образцы кода)

Адаптируй названия и цвета под продукт; структура и поведение — проверены в продакшене.

## Токены (`src/index.css`)
```css
@import "tailwindcss";
@import "tw-animate-css";
@import "shadcn/tailwind.css";
@import "@fontsource-variable/onest";          /* текст */
@import "@fontsource/unbounded/600.css";        /* заголовки страниц */
@custom-variant dark (&:is(.dark *));

@theme inline {
  --font-heading: 'Unbounded', 'Onest Variable', sans-serif;
  --font-sans: 'Onest Variable', system-ui, sans-serif;
  --color-accent-brand: var(--accent-brand);   /* «золото» — только смысловые акценты */
  --color-surface: var(--surface);
  --color-elevated: var(--elevated);
  --color-success: var(--success);
  /* ...остальные --color-* от shadcn оставить */
}
:root, .dark {
  color-scheme: dark;
  --background: #0e1015;  --foreground: #f2f3f5;
  --surface: #171a21;     --elevated: #20242d;
  --muted: #1d2129;       --muted-foreground: #8b92a1;
  --primary: #f6f7f9;     --primary-foreground: #0e1015;   /* главная кнопка — «экранно-белая» */
  --accent-brand: #f2b84b; --ring: #f2b84b;
  --border: rgb(255 255 255 / 8%); --input: rgb(255 255 255 / 12%);
  --radius: 0.625rem;
}
@layer base {
  :focus-visible { outline: 2px solid var(--ring); outline-offset: 2px; }
  ::selection { background: color-mix(in oklch, var(--ring) 35%, transparent); }
}
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: .01ms !important; transition-duration: .01ms !important; }
}
@layer utilities {
  .no-scrollbar { scrollbar-width: none } .no-scrollbar::-webkit-scrollbar { display: none }
  .pb-safe { padding-bottom: env(safe-area-inset-bottom) }
}
```
Правило: акцентный цвет — только прогресс, активный пункт навигации, фокус, ключевой индикатор. Не для декора.

## Img — плавное появление и зарезервированное место
```tsx
export function Img ({ src, alt = '', className, imgClassName, fallback, eager }: Props) {
  const [state, setState] = useState<'loading' | 'ok' | 'error'>(src ? 'loading' : 'error')
  useEffect(() => setState(src ? 'loading' : 'error'), [src])
  return (
    <div className={cn('relative overflow-hidden bg-muted', className)}>
      {src && state !== 'error' && (
        <img src={src} alt={alt} loading={eager ? 'eager' : 'lazy'} decoding="async"
          onLoad={() => setState('ok')} onError={() => setState('error')}
          className={cn('size-full object-cover transition-opacity duration-500', state === 'ok' ? 'opacity-100' : 'opacity-0', imgClassName)} />
      )}
      {state === 'error' && (fallback !== undefined ? fallback : <Placeholder />)}
    </div>
  )
}
```
Размер задаёт родитель (`aspect-[2/3]`, `aspect-video`) — ноль сдвигов макета. `alt=""`, если рядом видимое название.

## Rail — горизонтальная лента
Требования: нативный скролл + `snap-x snap-mandatory`; стрелки только на устройствах с мышью
(`[@media(hover:hover)]:flex`), листают на 85% ширины; вертикальное колесо НЕ перехватывать; у контейнера
`overflow-y-hidden` (крупные элементы — цифры топ-10, тени — иначе дают вертикальную прокрутку ленты); отрицательные
поля = отступам контента, чтобы лента «вылетала» под край экрана на телефоне; ResizeObserver обновляет края.
```tsx
export function Rail ({ children, className, gap = 'gap-3 sm:gap-4' }) {
  const ref = useRef<HTMLDivElement>(null)
  const [edges, setEdges] = useState({ start: true, end: false })
  const update = useCallback(() => { const el = ref.current; if (!el) return
    setEdges({ start: el.scrollLeft < 8, end: el.scrollLeft + el.clientWidth >= el.scrollWidth - 8 }) }, [])
  useEffect(() => { update(); const ro = new ResizeObserver(update); if (ref.current) ro.observe(ref.current); return () => ro.disconnect() }, [update])
  const page = (dir: 1 | -1) => ref.current?.scrollBy({ left: dir * ref.current.clientWidth * 0.85, behavior: 'smooth' })
  return (
    <div className={cn('group/rail relative', className)}>
      <div ref={ref} onScroll={update} className={cn('no-scrollbar -mx-4 flex snap-x snap-mandatory scroll-px-4 overflow-x-auto overflow-y-hidden overscroll-x-contain px-4 pt-1 pb-2 sm:-mx-6 sm:px-6 lg:-mx-10 lg:px-10', gap)}>{children}</div>
      {!edges.start && <ArrowButton side="left" onClick={() => page(-1)} />}
      {!edges.end && <ArrowButton side="right" onClick={() => page(1)} />}
    </div>
  )
}
```
Крупные декоративные элементы внутри карточек ограничивай рамкой фиксированной высоты с `overflow-hidden`.

## Section / PageTitle / EmptyState / ErrorState
- `Section`: заголовок `text-lg sm:text-xl font-semibold`, ссылка «Все» справа (на телефоне — шеврон у заголовка).
  Секции не делать первыми детьми обёрток с `first:mt-0` (съедает отступы) — используй Fragment.
- `PageTitle`: шрифт заголовков, `text-2xl sm:text-3xl`, подзаголовок `text-sm text-muted-foreground`.
- `EmptyState`: иконка в круге, заголовок, одна строка пояснения, одна кнопка-действие.
- `ErrorState` + `humanError(error)`: сетевые `TypeError`/«Failed to fetch» → «Нет связи с сервером. Проверьте…»;
  сообщения сервера — как есть с точкой; кнопка «Повторить».

## LoadMore — бесконечная подгрузка
IntersectionObserver с `rootMargin: '300px'` (больше — подтягивает лишние страницы на высоких экранах), активен
только при `hasNextPage && !isFetchingNextPage`; во время подгрузки — ряд скелетонов той же формы.

## Навигация (layout)
- ≥1024: боковое меню ~232px с подписями; 768–1023: колонка иконок 72px + тултипы справа; <768: нижний таб-бар (4–5
  пунктов, `pb-safe`), контенту `pb-28`.
- Активный пункт: фон + акцентная иконка; на телефоне один пункт может объединять разделы (Каталог = фильмы/сериалы…).
- Прокрутка: `useNavigationType() !== 'POP'` → `scrollTo(0,0)`; на «Назад» браузер восстановит позицию сам.
- Логотип не обязателен — если пользователь против, навигация начинается с поиска.

## Hero / Billboard (если продукт медийный)
Полноширинный фон с градиентами в цвет фона (снизу и слева), логотип-картинка вместо текста, если API даёт; автосмена
раз в 8 с с паузой при наведении/фокусе, без автосмены при reduced-motion; индикаторы-полоски — только ≥640px;
индекс слайда защищать от уменьшения списка (`items[Math.min(i, len-1)]`).

## Плеер/тяжёлые экраны
Логику держи в отдельном классе-движке (состояние → React через колбэк), UI только рисует. Свои контролы: перемотка
с `role="slider"` и клавишами, жесты на тач (двойной тап по краям ±10 с), автоскрытие панели, полноэкранный режим
через `document.documentElement` (иначе порталы меню не видны), «Следующая серия» с отсчётом и «Отмена».
Safari/iOS: MKV/WebM и бесконечный fMP4-поток без Range не играет — нужен HLS или MP4 с Range.
