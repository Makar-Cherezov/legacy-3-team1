# Приложение А к брифу ТЗ
## Диаграмма Маримекко на Kendo UI: простой вариант через canvas

| | |
|---|---|
| **Относится к** | `brief_tz_reports_module.md`, п. 6.3 (диаграмма Маримекко, вторая очередь) |
| **Статус** | Черновик для согласования |
| **Версия** | 0.1 |

---

## 1. Задача

В Kendo UI Chart нет диаграммы Маримекко (Marimekko, Mekko). Нужно получить её в модуле на тех же компонентах Kendo, без сторонних библиотек и с минимальными трудозатратами.

**Что такое Маримекко.** Это нормированная столбчатая диаграмма, у которой **ширина столбца пропорциональна его итогу**, а высота сегментов — доле значения внутри столбца. Площадь прямоугольника пропорциональна значению. Пример: столбец — госпрограмма (ширина — её объём финансирования), сегменты — доли ГРБС в нём.

```
 100% ┌────────────┬───────┬──────┬──┐
      │ ГРБС 3     │       │      │  │
      │────────────│ ГРБС 2│ГРБС 3│  │
      │ ГРБС 2     │       │──────│  │
      │────────────│───────│ГРБС 1│  │
      │ ГРБС 1     │ГРБС 1 │      │  │
   0% └────────────┴───────┴──────┴──┘
        ГП 1 (45%)  ГП 2     ГП 3  ГП 4
        ширина ~ объём ГП
```

## 2. Почему нельзя обойтись стандартным Chart

Kendo Chart строит столбцы одинаковой ширины. Ширину отдельного столбца нельзя привязать к данным, поэтому штатными настройками (`stack: { type: "100%" }`, `gap`, `spacing`) получится только обычная нормированная диаграмма. Приёмы с «шириной через несколько серий» дают приближение, ломаются при изменении данных и в доработке сложнее, чем собственная отрисовка.

## 3. Варианты реализации

| № | Вариант | Плюсы | Минусы |
|---|---|---|---|
| 1 | **Kendo Drawing API, поверхность canvas** (рекомендуется) | Уже входит в Kendo UI; фигуры, текст и события готовы; ~150 строк кода; переход на SVG меняет одну опцию | Нужно самим считать геометрию, легенду и подсказки |
| 2 | Kendo Drawing API, поверхность SVG | Тот же код; чёткая графика при любом масштабе, удобнее отладка в DOM | На сотнях сегментов медленнее canvas |
| 3 | Чистый HTML5 canvas (`<canvas>` и `getContext("2d")`) | Нет зависимостей | Всё вручную: поиск фигуры под курсором, масштабирование под HiDPI, события |
| 4 | Сторонняя библиотека (D3, Highcharts, amCharts) | Готовые решения | Лишняя зависимость, лицензия, чужой стиль |

Выбран **вариант 1**. Из-за него оценка в брифе: «Drawing — для кастомных элементов, в частности диаграммы Маримекко».

## 4. Идея реализации

1. Сервер отдаёт «плоские» строки: `{ column, segment, value }` — например, `{ "ГП 1", "ГРБС 2", 1250.5 }`.
2. Клиент группирует их: по каждому `column` считает итог, по каждому `segment` внутри него — значение.
3. Для каждого столбца рассчитывается:
   - ширина `w = W_полезная × итог_столбца / общий_итог`;
   - высота сегмента `h = H × значение / итог_столбца`.
4. Сегменты рисуются снизу вверх прямоугольниками `kendo.drawing.Rect`, подписи — `kendo.drawing.Text`.
5. Поверхность создаётся как `kendo.drawing.Surface.create(element, { type: "canvas" })`. События `mouseenter`, `mouseleave`, `click` приходят от Kendo и содержат фигуру под курсором: собственный hit-test не нужен.

## 5. Формат данных

```json
{
  "title": "Структура расходов по ГРБС",
  "unit": "млн руб.",
  "rows": [
    { "column": "ГП 11", "segment": "ГРБС 001", "value": 1250.5 },
    { "column": "ГП 11", "segment": "ГРБС 002", "value": 430.0 },
    { "column": "ГП 13", "segment": "ГРБС 001", "value": 980.2 }
  ]
}
```

Правила: значения должны быть **положительными** (нули и отрицательные пропускаются); порядок столбцов — по убыванию итога, порядок сегментов — по первому появлению (или по порядку, заданному специалистом); цвет сегмента определяется его названием, одинаковый во всех столбцах.

## 6. Пример кода (`marimekko.js`)

Стиль ES5, чтобы работало во всех браузерах, поддерживаемых Kendo для ASP.NET MVC 5.

```javascript
(function (global, $, kendo) {
    "use strict";

    var geo = kendo.geometry, draw = kendo.drawing;
    var PALETTE = ["#2e75b6", "#70ad47", "#ffc000", "#a5a5a5",
                   "#c55a11", "#7030a0", "#00b0f0", "#843c0c"];

    function buildModel(rows, maxColumns) {
        var cols = {}, order = [], segs = [], seen = {}, grand = 0;
        rows.forEach(function (r) {
            if (!(r.value > 0)) { return; }
            var c = cols[r.column];
            if (!c) { c = cols[r.column] = { name: r.column, total: 0, cells: {} }; order.push(c); }
            c.total += r.value;
            c.cells[r.segment] = (c.cells[r.segment] || 0) + r.value;
            grand += r.value;
            if (!seen[r.segment]) { seen[r.segment] = true; segs.push(r.segment); }
        });
        order.sort(function (a, b) { return b.total - a.total; });
        if (maxColumns && order.length > maxColumns) {
            var rest = order.splice(maxColumns - 1), other = { name: "Прочие", total: 0, cells: {} };
            rest.forEach(function (c) {
                other.total += c.total;
                Object.keys(c.cells).forEach(function (k) { other.cells[k] = (other.cells[k] || 0) + c.cells[k]; });
            });
            order.push(other);
        }
        return { columns: order, segments: segs, grand: grand };
    }

    // тёмный текст на светлом фоне и наоборот
    function textColor(hex) {
        var n = parseInt(hex.slice(1), 16);
        var lum = 0.299 * (n >> 16) + 0.587 * ((n >> 8) & 255) + 0.114 * (n & 255);
        return lum > 160 ? "#222" : "#fff";
    }

    function MarimekkoChart(element, options) {
        this.element = $(element).css("position", "relative");
        this.options = $.extend(true, {
            margin: { top: 34, right: 16, bottom: 36, left: 44 },
            gap: 2,
            font: "12px Roboto, Arial, sans-serif",
            minLabelWidth: 44,
            minLabelHeight: 16,
            maxColumns: 20,
            emptyText: "Нет данных для отображения",
            colors: PALETTE,
            format: function (v) { return kendo.toString(v, "n1"); },
            click: null
        }, options);

        this.surface = draw.Surface.create(this.element, { type: "canvas" });
        this.tip = $('<div class="mm-tooltip" style="display:none;position:absolute;' +
                     'pointer-events:none;background:#fff;border:1px solid #999;' +
                     'padding:4px 8px;font-size:12px;"></div>').appendTo(this.element);
        this.uid = ++MarimekkoChart.counter;
        this._bind();
    }
    MarimekkoChart.counter = 0;

    MarimekkoChart.prototype = {
        setData: function (rows) {
            this.model = buildModel(rows || [], this.options.maxColumns);
            this.render();
        },

        render: function () {
            var o = this.options, m = this.model, self = this;
            this.surface.resize();
            this.surface.clear();
            if (!m || !m.grand) {
                var empty = new draw.Group();
                empty.append(this._text(o.emptyText, this.element.width() / 2,
                                        this.element.height() / 2, "#666"));
                this.surface.draw(empty);
                return;
            }

            var W = this.element.width(), H = this.element.height();
            var plot = {
                x: o.margin.left, y: o.margin.top,
                w: W - o.margin.left - o.margin.right,
                h: H - o.margin.top - o.margin.bottom
            };
            var usable = plot.w - o.gap * (m.columns.length - 1);
            var root = new draw.Group();
            var x = plot.x;

            m.columns.forEach(function (col) {
                var cw = usable * col.total / m.grand;
                var y = plot.y + plot.h;

                m.segments.forEach(function (seg, i) {
                    var v = col.cells[seg];
                    if (!v) { return; }
                    var ch = plot.h * v / col.total;
                    var fillColor = o.colors[i % o.colors.length];
                    var rect = new draw.Rect(new geo.Rect([x, y - ch], [cw, ch]), {
                        fill: { color: fillColor },
                        stroke: { color: "#fff", width: 1 },
                        cursor: "pointer"
                    });
                    rect.info = {
                        column: col.name, segment: seg, value: v,
                        shareInColumn: v / col.total, shareOfTotal: v / m.grand
                    };
                    root.append(rect);

                    if (cw >= o.minLabelWidth && ch >= o.minLabelHeight) {
                        root.append(self._text(kendo.toString(v / col.total, "p0"),
                                               x + cw / 2, y - ch / 2, textColor(fillColor)));
                    }
                    y -= ch;
                });

                var share = kendo.toString(col.total / m.grand, "p0");
                var title = col.name + " (" + share + ")";
                if (kendo.util.measureText(title, { font: o.font }).width <= cw) {
                    root.append(self._text(title, x + cw / 2, plot.y + plot.h + 14, "#333"));
                }
                x += cw + o.gap;
            });

            [0, 0.25, 0.5, 0.75, 1].forEach(function (t) {
                root.append(self._text(kendo.toString(t, "p0"),
                    o.margin.left - 20, plot.y + plot.h * (1 - t), "#666"));
            });

            this._legend(root, m.segments);
            this.surface.draw(root);
        },

        _text: function (content, cx, cy, color) {
            var size = kendo.util.measureText(content, { font: this.options.font });
            return new draw.Text(content,
                [cx - size.width / 2, cy - size.height / 2],
                { font: this.options.font, fill: { color: color } });
        },

        _legend: function (root, segments) {
            var o = this.options, lx = o.margin.left, ly = 8;
            segments.forEach(function (seg, i) {
                var w = kendo.util.measureText(seg, { font: o.font }).width;
                root.append(new draw.Rect(new geo.Rect([lx, ly + 2], [12, 12]),
                    { fill: { color: o.colors[i % o.colors.length] }, stroke: null }));
                root.append(new draw.Text(seg, [lx + 16, ly], { font: o.font }));
                lx += 16 + w + 16;
            });
        },

        _bind: function () {
            var self = this, last = { x: 0, y: 0 };
            this.element.on("mousemove", function (e) {
                var off = self.element.offset();
                last = { x: e.pageX - off.left, y: e.pageY - off.top };
                if (self.tip.is(":visible")) {
                    self.tip.css({ left: last.x + 12, top: last.y + 12 });
                }
            });
            this.surface.bind("mouseenter", function (e) {
                var i = e.element.info;
                if (!i) { return; }
                self.tip.html("<b>" + kendo.htmlEncode(i.column) + "</b><br>" +
                    kendo.htmlEncode(i.segment) + ": " + self.options.format(i.value) +
                    "<br>Доля в столбце: " + kendo.toString(i.shareInColumn, "p1") +
                    "<br>Доля от общего: " + kendo.toString(i.shareOfTotal, "p1"))
                    .css({ left: last.x + 12, top: last.y + 12 }).show();
            });
            this.surface.bind("mouseleave", function () { self.tip.hide(); });
            this.surface.bind("click", function (e) {
                if (e.element.info && self.options.click) { self.options.click(e.element.info); }
            });
            $(global).on("resize.marimekko" + this.uid, function () { self.render(); });
        },

        destroy: function () {
            $(global).off("resize.marimekko" + this.uid);
            this.element.off("mousemove");
            this.surface.destroy();
            this.tip.remove();
        }
    };

    global.MarimekkoChart = MarimekkoChart;
})(window, jQuery, kendo);
```

> **Проверено запуском.** Расчёт геометрии (ширина столбцов по итогам, высоты сегментов, пропуск нулевых и отрицательных значений, объединение в «Прочие», пустое состояние) прогнан в JavaScriptCore с заглушками jQuery и Kendo; пропорции верны. Вызовы самого Kendo (`Surface.create`, события поверхности, `measureText`, `destroy`) **не проверены**.
>
> Код — рабочая заготовка для старта. Перед использованием его нужно проверить на целевой версии Kendo UI (имена API `kendo.drawing.Surface`, `kendo.util.measureText`, события поверхности стабильны в современных версиях, но версия определяется на старте проекта).

## 7. Подключение в ASP.NET MVC 5

**Что нужно подключить.** `kendo.core`, `kendo.drawing` и `kendo.dataviz.core` (или полный `kendo.all`/`kendo.dataviz`) — то, что уже используется для Chart. Отдельных лицензий не требуется.

**Контроллер** отдаёт данные «плоскими» строками, агрегация выполняется в БД:

```csharp
public class ReportDataController : Controller
{
    private readonly IReportDataService _data;
    public ReportDataController(IReportDataService data) { _data = data; }
    private ICurrentUser CurrentUser { get { return (ICurrentUser)HttpContext.Items["currentUser"]; } }

    [HttpGet]
    [ReportUserAuthorize]   // проверка входа и роли модуля (п. 5 брифа); для AJAX — чистый 401
    public JsonResult Marimekko(Guid reportId, string filter)
    {
        // фильтры разбираются сервисом и принимаются только по полям формы (белый список);
        // Request.QueryString целиком в сервис не передаётся
        var rows = _data.GetMarimekko(reportId, filter, CurrentUser);
        if (rows == null) { Response.StatusCode = 404; return Json(new { code = "NOT_FOUND" }); }
        return Json(new { rows }, JsonRequestBehavior.AllowGet);   // объект, а не массив
    }
}

public class MarimekkoRow
{
    public string Column { get; set; }   // измерение для столбцов
    public string Segment { get; set; }  // измерение для сегментов
    public decimal Value { get; set; }   // мера (>0)
}
```

Запрос сервиса — группировка `GROUP BY column, segment` с `SUM(value)` по источникам из п. 7 брифа.

**Представление (Razor).** В самой АИС ГП представления `.aspx`; для обособленного модуля на MVC 5 допустим Razor, а при необходимости единого стиля разметка переносится в `.aspx` без изменений в скрипте.

```html
<div id="mm" style="height:420px"></div>
<script src="~/Scripts/marimekko.js"></script>
<script>
    var chart = new MarimekkoChart("#mm", { format: function (v) { return kendo.toString(v, "n1") + " млн руб."; } });
    $.getJSON('@Url.Action("Marimekko", "ReportData", new { reportId = Model.ReportId })',
              function (d) { chart.setData(d.rows); });
</script>
```

**Связь с конструктором.** В форме отчёта специалист выбирает тип визуализации «Маримекко» и сопоставляет три поля: **измерение столбцов**, **измерение сегментов**, **мера**. Фильтры пользователя передаются в тот же запрос, а переключатель «таблица ↔ график» использует те же данные: таблица показывает ту же выборку в виде строк.

## 8. Ограничения и поведение в крайних случаях

| Ситуация | Поведение |
|---|---|
| Нулевые и отрицательные значения | Пропускаются (площадь не может быть отрицательной) |
| Очень узкий столбец или сегмент | Подпись скрывается, значение остаётся в подсказке |
| Подпись столбца не помещается | Не выводится, доступна в подсказке; при необходимости добавляется поворот или сокращение |
| Много столбцов (более `maxColumns`, по умолчанию 20) | Самые мелкие объединяются в «Прочие» в клиентском коде (иначе при зазорах шире области ширина становится отрицательной; проверено запуском). Порог можно задать и на сервере |
| Нет данных | Пустая область с сообщением «Нет данных для отображения» (рисуется кодом) |
| Изменение размера окна | Перерисовка по событию `resize` |
| Печать и экспорт | Не требуются (п. 10 брифа) |
| Цвета | Одинаковый цвет сегмента во всех столбцах, палитра — в стиле АИС ГП |

## 9. Сравнение canvas и SVG

Код выше работает с обоими видами поверхности: достаточно заменить `type: "canvas"` на `type: "svg"`. Начинаем с canvas как с самого простого и быстрого варианта. Если потребуется чёткая графика при масштабировании или стилизация через DOM, переход — одна строка.

## 10. Трудоёмкость (ориентировочно)

| Работа | Оценка |
|---|---|
| Модуль отрисовки (геометрия, подписи, легенда, подсказка) | 2–3 дня |
| Интеграция с конструктором (выбор полей, фильтры, переключение «таблица ↔ график») | 1–2 дня |
| Тесты и подгонка внешнего вида | 1 день |
| **Итого** | **4–6 дней** |

Оценка предварительная и подтверждается после прототипа.

## 11. Критерии приёмки

- Ширина столбцов пропорциональна итогам, высоты сегментов — долям внутри столбца; суммы по тестовым данным совпадают с контрольными.
- Подсказка при наведении показывает столбец, сегмент, значение и обе доли.
- Легенда соответствует цветам сегментов.
- Нулевые и отрицательные значения не ломают диаграмму.
- Диаграмма корректно перерисовывается при изменении размера окна и применении фильтров.
- Специалист может подключить диаграмму к форме отчёта, выбрав три поля; пользователь видит её в опубликованной форме.
