# Нарисовать разбор в Фигме

Как превратить разбор в живые слои рядом с оригиналом: копия фрейма с красными обводками, номерами и подписями от руки, плюс карточка с оценкой. Код проверен на реальном файле, его можно брать почти как есть, меняя только массив `ISS`, оценки и тексты.

## Порядок

1. `get_metadata` по фрейму: размеры, координаты блоков. Координаты замечаний задавай в пикселях фрейма по этим данным, а не на глаз со скриншота. Скриншот 1600 px от фрейма 1920 px масштабирован на 0,833.
2. Один `use_figma` проверяет шрифты (`listAvailableFontsAsync`, нужен Caveat Bold, запасной вариант Inter) и свободное место на странице.
3. Второй `use_figma` рисует всё (код ниже).
4. `get_screenshot` копии. Если подпись закрывает важный текст, подвинь её отдельным маленьким вызовом.

## Подводные камни

- **Фрейм на автолейауте.** Клон тоже на автолейауте, и добавленные слои встанут в поток под контентом и пропадут из вида. Ставь `layoutPositioning = 'ABSOLUTE'` слоям «Пометки» и «Подписи», потом `x = 0`, `y = 0`.
- **Место на странице.** Слайды деки часто стоят в ряд, справа занято. Клади копию под оригиналом и сдвигай вниз, пока не найдёшь свободное место (функция `hits`).
- **Обводки одной картинкой.** Все пути собираются в один SVG через `createNodeFromSvg`. Так быстрее и пометки легко удалить одним слоем.
- **Подписи живыми слоями.** Белая плашка с тенью, номер в круге и текст Caveat Bold 34 px, наклон 1,5°. Ставь подпись рядом с обводкой туда, где пусто: справа от конца строки, в поле слайда или между рядами.
- Вызов `use_figma` транзакционный: при любой ошибке откатывается целиком, поэтому после ошибки просто исправь и запусти заново.

## Код

```js
const src = await figma.getNodeByIdAsync('FRAME_ID');
const page = src.parent;
await Promise.all([
  figma.loadFontAsync({ family: 'Caveat', style: 'Bold' }),
  figma.loadFontAsync({ family: 'Inter', style: 'Regular' }),
  figma.loadFontAsync({ family: 'Inter', style: 'Medium' }),
  figma.loadFontAsync({ family: 'Inter', style: 'Semi Bold' }),
]);
const FW = src.width, FH = src.height;
const W = FW + 80 + 820, H = Math.max(FH, 2400);
let y0 = src.y + FH + 240;
const hits = (y) => page.children.some(c => c !== src && c.x < src.x + W && c.x + c.width > src.x && c.y < y + H && c.y + c.height > y);
let guard = 0; while (hits(y0) && guard++ < 20) y0 += 600;

const RED = { r: 0.89, g: 0.15, b: 0.11 };
const copy = src.clone(); page.appendChild(copy);
copy.name = src.name + ' · Красная ручка'; copy.x = src.x; copy.y = y0;

function rng(seed) { let a = seed >>> 0; return () => { a = (a + 0x6D2B79F5) | 0; let t = Math.imul(a ^ (a >>> 15), 1 | a); t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t; return ((t ^ (t >>> 14)) >>> 0) / 4294967296; }; }
const f = (v) => Math.round(v * 10) / 10;
function curve(p) { let d = `M${f(p[0][0])} ${f(p[0][1])}`; for (let i = 0; i < p.length - 1; i++) { const p0 = p[i - 1] || p[i], p1 = p[i], p2 = p[i + 1], p3 = p[i + 2] || p2; d += ` C${f(p1[0] + (p2[0] - p0[0]) / 6)} ${f(p1[1] + (p2[1] - p0[1]) / 6)} ${f(p2[0] - (p3[0] - p1[0]) / 6)} ${f(p2[1] - (p3[1] - p1[1]) / 6)} ${f(p2[0])} ${f(p2[1])}`; } return d; }
// kind: 'box' для блоков, 'oval' для мелкого элемента, 'line' для строки текста (подчеркнуть)
function mark(n, kind, b) {
  const R = rng(n * 9973 + 7), J = (a) => (R() * 2 - 1) * a * 2.5;
  const [x1, y1, x2, y2] = b, bw = x2 - x1, bh = y2 - y1;
  if (kind === 'line') { const pts = [], N = Math.max(3, Math.round(bw / 180)); for (let i = 0; i <= N; i++) pts.push([x1 - 6 + (bw + 14) * i / N, y2 + 8 + J(1.2) - i / N * 4]); return curve(pts); }
  if (kind === 'oval') { const cx = (x1 + x2) / 2, cy = (y1 + y2) / 2, rx = bw / 2 + 22, ry = bh / 2 + 16, a0 = -2.6 + R() * 0.5, pts = []; for (let i = 0; i <= 32; i++) { const a = a0 + 1.1 * 2 * Math.PI * i / 32, k = 1 + (R() - 0.5) * 0.03 + i / 32 * 0.05; pts.push([cx + Math.cos(a) * rx * k, cy + Math.sin(a) * ry * k]); } return curve(pts); }
  const p = 12, X1 = x1 - p, Y1 = y1 - p, X2 = x2 + p, Y2 = y2 + p;
  const C = [[X1 + J(2), Y1 + J(2)], [X2 + J(2), Y1 + J(2)], [X2 + J(2), Y2 + J(2)], [X1 + J(2), Y2 + J(2)]];
  const st = [C[0][0] + bw * 0.04 + 6, C[0][1] + J(1.5)], en = [C[0][0] + bw * 0.1 + 14, C[0][1] - 5];
  const seg = (a, c) => { const mx = (a[0] + c[0]) / 2, my = (a[1] + c[1]) / 2, dx = c[0] - a[0], dy = c[1] - a[1], L = Math.hypot(dx, dy) || 1, bow = J(3) + L * 0.008 * (R() - 0.5); return `Q${f(mx - dy / L * bow)} ${f(my + dx / L * bow)} ${f(c[0])} ${f(c[1])}`; };
  return `M${f(st[0])} ${f(st[1])} ${seg(st, C[1])} ${seg(C[1], C[2])} ${seg(C[2], C[3])} ${seg(C[3], C[0])} ${seg(C[0], en)}`;
}

// [номер, вид пометки, серьёзность, [x1,y1,x2,y2] во фрейме, подпись ручкой, [x,y] подписи, категория, суть, почему, как поправить]
const ISS = [
  [1, 'box', 'high', [118, 182, 1420, 308], 'дубль цифр', [1450, 196], 'Иерархия', 'Заголовок пересказывает цифры под ним', 'Почему мешает.', 'Что сделать, с цифрами.'],
];

let paths = '';
for (const [n, kind, sev, box] of ISS) paths += `<path d="${mark(n, kind, box)}" stroke="#E3261B" stroke-width="${sev === 'high' ? 5 : sev === 'low' ? 3.5 : 4.2}" stroke-linecap="round" stroke-linejoin="round" fill="none"/>`;
const svg = figma.createNodeFromSvg(`<svg width="${FW}" height="${FH}" viewBox="0 0 ${FW} ${FH}" fill="none" xmlns="http://www.w3.org/2000/svg">${paths}</svg>`);
svg.name = 'Пометки'; copy.appendChild(svg);
const labels = figma.createFrame(); labels.name = 'Подписи'; labels.fills = []; labels.resize(FW, FH); labels.clipsContent = false; copy.appendChild(labels);
for (const n of [svg, labels]) { if (copy.layoutMode && copy.layoutMode !== 'NONE') n.layoutPositioning = 'ABSOLUTE'; n.x = 0; n.y = 0; }

const txt = (s, fam, style, size, color, parent) => { const t = figma.createText(); t.fontName = { family: fam, style }; t.fontSize = size; t.characters = s; t.fills = [{ type: 'SOLID', color }]; if (parent) parent.appendChild(t); return t; };
for (const [n, , , , pen, [lx, ly]] of ISS) {
  const tag = figma.createAutoLayout('HORIZONTAL');
  tag.name = 'Пометка ' + n; tag.itemSpacing = 8; tag.paddingLeft = 6; tag.paddingRight = 12; tag.paddingTop = 4; tag.paddingBottom = 4; tag.counterAxisAlignItems = 'CENTER';
  tag.cornerRadius = 8; tag.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
  tag.effects = [{ type: 'DROP_SHADOW', color: { r: 0, g: 0, b: 0, a: 0.22 }, offset: { x: 0, y: 2 }, radius: 6, spread: 0, visible: true, blendMode: 'NORMAL' }];
  const num = figma.createAutoLayout('HORIZONTAL'); num.resize(38, 38); num.primaryAxisSizingMode = 'FIXED'; num.counterAxisSizingMode = 'FIXED'; num.primaryAxisAlignItems = 'CENTER'; num.counterAxisAlignItems = 'CENTER'; num.cornerRadius = 19; num.fills = []; num.strokes = [{ type: 'SOLID', color: RED }]; num.strokeWeight = 2.5;
  txt(String(n), 'Caveat', 'Bold', 28, RED, num); tag.appendChild(num);
  txt(pen, 'Caveat', 'Bold', 34, RED, tag);
  labels.appendChild(tag); tag.x = lx; tag.y = ly; tag.rotation = 1.5;
}
// Дальше карточка разбора: вертикальный автолейаут 820 px справа от копии
// (copy.x + FW + 80): шапка, оценка Caveat 96 px в красном круге, вердикт,
// шесть строк категорий с полосками из 10 делений, список замечаний
// (номер в круге, суть Inter Semi Bold 22, «серьёзность · категория»,
// почему Inter 18 серым, «Как поправить:» Inter 18), «Что уже хорошо».
// Текстам внутри автолейаута: textAutoResize = 'HEIGHT', layoutSizingHorizontal = 'FILL'.
figma.viewport.scrollAndZoomIntoView([copy]);
return { copy: copy.id };
```
