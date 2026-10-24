# Приложение Б к брифу ТЗ
## Сущности модели данных АИС ГП, подходящие для выбора источниками данных отчётов

| | |
|---|---|
| **Относится к** | `brief_tz_reports_module.md`, п. 6.2 (выбор источника в конструкторе) и п. 7 |
| **Основа** | Модель данных `Ugf.GovPrograms.DataLayer` (каталог `Model`, около 450 классов) |
| **Статус** | Черновик для согласования |
| **Версия** | 0.1 |

---

## 1. Назначение и принципы отбора

Конструктор форм (п. 6.2 брифа) работает по **белому списку** источников. Здесь предложен такой список: из всех классов модели выбраны те, что содержат осмысленные для аналитики данные.

**Критерии включения.** Сущность подходит, если:
1. она отражает предметную область (программа, план, показатель, финансирование, отчёт о реализации);
2. в ней есть значения, которые можно считать (суммы, даты, статусы, количества) или по которым можно группировать (справочники);
3. её можно привязать к госпрограмме и отчётному периоду.

**Критерии исключения** (в п. 4): технические снимки версий, связочные таблицы, служебные и системные данные (файлы, подписи, импорт, права, логи).

**Что считается источником.** Источник данных — это основная сущность с набором разрешённых связей и справочников. Например, источник «Контрольные события» включает `PlanKeyEvent` и допустимые связи до программы, плана и отчёта. Связочные таблицы `mm*` используются внутри источника как технические связи и в конструкторе отдельно не показываются.

**Обозначения.** «Ядро» — используется в примерах отчётов п. 7 брифа, реализуется в первую очередь. «Доп.» — подходит для источника, реализуется по потребности. «Справочник» — измерение для группировки и фильтрации, а не самостоятельный источник.

## 2. Ключевые особенности модели

- **Версионность.** `GovernmentProgram` — это одна **версия** госпрограммы (поля `GpId`, `VersionName`, `CurrentYear`, `CreationDate`). Данные программы (подпрограммы, показатели, планы) привязаны к версии. Отчёт строится в контексте выбранной версии и отчётного года.
- **Историчность значений.** Значения хранятся по датам (`ActualDate`): `IndicatorValue`, `KBKFinancing`, `PlanKeyEvent` и др. В источнике нужно задавать правило выбора записи на дату.
- **Снимки и `*Version`.** Классы `*Version` и `Snapshot` — копии данных на момент утверждения. Для аналитики берутся рабочие сущности, а снимки — только если нужен срез на прошлую дату (вторая очередь).
- **План и факт разделены.** Плановые данные — `Plan*`, `PlanKeyEvent`; фактические по отчётному периоду — `Report*`, связанные через `RealizationReport` (`RealizationReportId`).
- **Финансирование.** Бюджетные суммы — по КБК (`KBKFinancing`, `ReportKBKFinancing`). Прямой ссылки на программу у `KBKFinancing` нет: связь идёт через `mmFCPKBKFinancing` (ФЦП) и `mmSubprgActKBKFinancing` (основные мероприятия, ВЦП). В `ReportKBKFinancing` есть прямые `FCPId` и `SubprogramActionId`.
- **Данные Минфина** (`MfData*`, `MfProgram`) хранят коды (`GRBSCode`, `VRCode`) строками, без внешних ключей на справочники.

## 3. Рекомендуемые источники

### 3.1. Структура госпрограмм

| Сущность | Что содержит | Меры и измерения | Связи | Статус |
|---|---|---|---|---|
| `GovernmentProgram` | Версия госпрограммы | Измерения: код, название, направление (`ProgramScopeId`), нацпроект (`PNPId`), год (`CurrentYear`), приоритетность, даты. Мера: количество | `ProgramScope`, `PNP`, `Gp` | Ядро |
| `Subprogram` | Подпрограмма | Код, название, ответственный (`DepartmentId`) | `GovernmentProgram` (`GovProgramId`), `Department` | Ядро |
| `FCP` | Федеральная целевая программа | Код, название, ответственный | `GovernmentProgram`, `Department` | Ядро |
| `SubprogramAction` (`VCP`) | Основное мероприятие, ВЦП | Код, название, статус (`StateId`), сроки, ответственный | `Subprogram`, `State`, `Department`, `PNP` | Ядро |
| `ProgramGoal` | Цель программы | Название, порядок | `Program` | Доп. |
| `ProgramTask` | Задача программы | Название, порядок | `Program` | Доп. |
| `Measure` | Мера государственного регулирования | Название, описание | `Subprogram`, `SubprogramAction`, `RealizationReport` | Доп. |

### 3.2. Планы и контрольные события

| Сущность | Что содержит | Меры и измерения | Связи | Статус |
|---|---|---|---|---|
| `Plan` | План реализации (детальный план-график) | Год начала (`StartYear`), дата создания | `GovernmentProgram` | Ядро |
| `PlanSubprogram` | Подпрограмма в плане | Названия, сроки; объёмы по годам (`FirstYearVal`, `SecondYearVal`, `ThirdYearVal`) | `Plan`, `PersonInfo` | Ядро |
| `PlanFcp` | ФЦП в плане | Сроки, объёмы (`FirstYearFin`, `FullFin`) | `Plan`, `PersonInfo` | Ядро |
| `PlanSubprogramAction` | Основное мероприятие в плане | Название, сроки, ожидаемый результат | `PlanSubprogram` | Ядро |
| `PlanSimpleAction` | Мероприятие плана | Код, название, сроки, статус, ожидаемый результат | `PlanSubprogramAction`, `PlanFcp`, `PNP`, `Indicator`, `Territory` | Ядро |
| `PlanKeyEvent` | Контрольное событие плана | Название, срок (`ActualDate`), статус, комментарии | `PlanSimpleAction`, `KeyEvent` | Ядро |
| `PlanSimpleActKbkFin` | Финансирование мероприятия плана по КБК | `FullVal`, суммы по трём годам | `KBK`, `PlanSimpleAction` | Доп. |

### 3.3. Отчёты о реализации (факт)

| Сущность | Что содержит | Меры и измерения | Связи | Статус |
|---|---|---|---|---|
| `RealizationReport` | Отчёт о реализации (период, исполнитель) | Дата, признак завершения, название | `PersonInfo`, версия госпрограммы | Ядро |
| `ReportPlanKeyEvent` | Факт по контрольному событию | Факт срока (`FactEndDate`), ожидаемый срок, проблемы | `RealizationReport`, `PlanKeyEvent` | Ядро |
| `ReportPlanSimpleAction` | Факт по мероприятию плана | Факт сроков, `CashExecution`, `ContractSumm`, `BudgetCost`, результат | `RealizationReport`, `PlanSimpleAction` | Ядро |
| `ReportPlanSubprogramAction` | Факт по основному мероприятию | Сроки, `CashExecution`, `ContractSumm`, `BudgetCost` | `RealizationReport`, `PlanSubprogramAction` | Ядро |
| `ReportPlanFcp` | Факт по ФЦП | `CashExecution`, `ContractSumm`, `BudgetCost` | `RealizationReport`, `PlanFcp` | Доп. |
| `ReportSubprogramAction` | Факт по основному мероприятию или ВЦП (вне плана) | Сроки, `CashExecution`, `ContractSumm` | `RealizationReport`, `SubprogramAction` | Доп. |
| `ReportVCP` | Факт по ВЦП | `CashExecution`, `ContractSumm`, сроки | `RealizationReport`, `VCP` | Доп. |
| `Risk` | Риски отчёта | Название, описание | `RealizationReport` | Доп. |

### 3.4. Показатели

| Сущность | Что содержит | Меры и измерения | Связи | Статус |
|---|---|---|---|---|
| `Indicator` | Показатель (индикатор) | Название, единица, тип, периодичность, ответственный | `Unit`, `IndicatorType`, `Periodicity`, `FOIV`, `TerritoryCategory` | Ядро |
| `IndicatorValue` | Значение показателя | План (`Val`, `ValGoal`), факт (`FactVal`), скорректированное, дата | `Indicator`, `GovernmentProgram` | Ядро |
| `ReportIndicatorValue` | Значение в отчёте | `Val`, значения прошлого года, причины отклонения | `RealizationReport`, `Indicator` | Ядро |
| `TerritoryIndicatorValue` | Значение по регионам | То же + регион | `Territory` | Доп. |
| `ReportTerritoryIndicatorValue` | Значение по регионам в отчёте | То же + регион | `Territory` | Доп. |
| `PNPIndicator`, `PNPIndicatorValue`, `ReportPnpIndicatorValue` | Показатели и значения нацпроектов | Название, единица, значения, дата | `Task`, `Unit`, `RealizationReport` | Доп. |

Привязка показателей к программе: `IndicatorValue.GovernmentProgramId` либо связь `mmProgIndicator` (программа или подпрограмма ↔ показатель).

### 3.5. Финансирование

| Сущность | Что содержит | Меры и измерения | Связи | Статус |
|---|---|---|---|---|
| `KBKFinancing` | План финансирования по КБК | `Val`, `CorrectedVal`, `FactVal`, внебюджет (`OffBudgetVal`, `OffBudgetFactVal`), дата | `KBK` | Ядро |
| `ReportKBKFinancing` | Фактическое финансирование по КБК | `BudgetCost`, `CashExecution` | `RealizationReport`, `KBK`, `FCPId`, `SubprogramActionId` | Ядро |
| `CashExecution` | Кассовое исполнение | `Value`, `SbrValue`, дата, коды ГП и ОМ | `KBK`, `PlanFcp`, `PlanSubprogramAction` | Ядро |
| `SubprogramFinancingValue` | Ресурсное обеспечение подпрограммы по источникам | `Val`, `ValFact`, `ValExt`, дата | `Subprogram`, `FinancingSource` | Ядро |
| `SubprogramActionFinancingValue` | То же для основного мероприятия/ВЦП | `Val`, `ValFact`, `ValExt` | `SubprogramAction`, `FinancingSource` | Ядро |
| `FCPFinancingValue` | То же для ФЦП | `Val`, `ValFact`, `ValExt` | `FCP`, `FinancingSource` | Ядро |
| `MfDataVolumeCashExecution` | Объёмы и кассовое исполнение (данные Минфина) | `Volume`, `CashExecution`, коды ГРБС, ВР, направления расходов | `MfProgramSubProgram`, `MfMainMeasure` | Ядро |
| `MfDataBudgetLimitVolumesProjection` | Проектировки предельных объёмов (Минфин) | Объёмы текущего и двух следующих лет | `MfProgram`, `MfSubject` | Доп. |
| `BudgetLimitVolumesProjections` | Проектировки предельных объёмов по ГП | `Value`, год, ГРБС | `GovernmentProgram`, `KBK` | Доп. |
| `MeasureFinancing`, `TaskFinancing` | Финансовая оценка мер, финансирование задач | `Val`, дата | `Measure`, `Task` | Доп. |

### 3.6. Прочие предметные источники

| Сущность | Что содержит | Статус |
|---|---|---|
| `Npa` | Нормативно-правовой акт: номер, срок, дата утверждения, тип, ответственный | Доп. |
| `Grant`, `GrantValue` | Субсидии и значения по регионам | Доп. |
| `InvestmentProject` | Паспорт инвестиционного проекта: суммы, сроки | Доп. |
| `FaipObject` | Объект ФАИП: регион, ФОИВ, мощность, сроки | Доп. |
| `GovService` | Государственная услуга | Доп. |
| `GovProgramStatistic` | Сводные числа по программе (число подпрограмм, мероприятий, ФОИВ, `AmountOfCharges`) | Доп., с оговоркой: использовать только числовые поля, XML-поля (`...Data`, `OpenDataXML`) не использовать |
| `DocStage`, `CommonDocStage` | Стадия согласования госпрограммы и даты | Доп. |

### 3.7. Справочники (измерения для группировки и фильтрации)

| Сущность | Содержание | Используется для |
|---|---|---|
| `FOIV`, `Department` | Органы власти и департаменты | Ответственные исполнители |
| `GRBS` | Главный распорядитель бюджетных средств | Структура расходов |
| `CSR`, `VR`, `FKR` | Целевая статья, вид расходов, раздел/подраздел | Бюджетная классификация |
| `KBK` | Код бюджетной классификации (ГРБС, ЦСР, ВР, ФКР, департамент) | Связка финансирования |
| `FinancingSource` | Источник финансирования | Структура финансирования |
| `ProgramScope` | Социально значимое направление | Группировка госпрограмм |
| `PNP` | Национальный проект | Группировка |
| `Territory`, `TerritoryCategory`, `Okato` | Регионы и их категории | Региональный срез |
| `Unit`, `Periodicity`, `IndicatorType` | Единицы, периодичность, тип показателя | Подписи и группировка показателей |
| `State`, `KeyEventState`, `PersonInfo` | Статусы и сведения об ответственных | Фильтры, подписи |

## 4. Не рекомендуются в качестве источников

| Группа | Сущности | Причина |
|---|---|---|
| Снимки и версии | Все `*Version` (например, `PlanKeyEventVersion`, `IndicatorValueVersion`), `Snapshot`, `GovernmentProgramVersion`, `MapProgramVer`, `MapProgramVerDesc` | Технические копии на момент утверждения; дублируют рабочие данные |
| Связочные таблицы | Все `mm*` | Только внутренние связи; отдельного смысла для отчёта нет |
| Права и пользователи | `UserGovernmentProgramRole`, `*Right`, `IndicatorInputPermisson`, `BanInput`, `BanTable`, `mmBanTable*` | Безопасность и управление доступом; в отчёты не выводятся |
| Файлы и подписи | `File`, `DocFile`, `FileContent`, `EcpFile`, `EcpSignature`, `EcpFileSignature`, `EcpReportSignature`, `InterestedFoivFile` | Бинарные и служебные данные |
| Обмен и импорт | `ImportFile`, `ImportPacket`, `ExportPacket`, `EmissImportFile*`, `GasuXmlsLog`, `GASUDataTransfer`, `MinecMap`, `MinecLog`, `PublishedGasuTableItem` | Служебные данные интеграций |
| Журналы и настройки | `LoadLog`, `TaskShedulerLogItem`, `SystemSetting`, `ProgramProjectStateStorage` | Системные данные |
| Проверки и экспертизы | `FKCheck*`, `Examination*`, `FoivExaminationXml` | Служебные процессы согласования (при необходимости — отдельным решением) |
| Общественное обсуждение | `Proposal*`, `NewProposal*`, `PublicDiscussions` | Другая подсистема, не относится к отчётности |

## 5. Типовые связи для построения источников

| Задача | Путь |
|---|---|
| Структура программы | `GovernmentProgram` ← `Subprogram` (`GovProgramId`) ← `SubprogramAction` (`SubprogramId`); `GovernmentProgram` ← `FCP` (`GovernmentProgramId`) |
| План → контрольное событие | `GovernmentProgram` ← `Plan` ← `PlanSubprogram` ← `PlanSubprogramAction` ← `PlanSimpleAction` ← `PlanKeyEvent` |
| План по ФЦП | `Plan` ← `PlanFcp` ← `PlanSimpleAction` (`PlanFcpId`) |
| Факт | `RealizationReport` ← `ReportPlanKeyEvent` → `PlanKeyEvent`; аналогично `ReportPlanSimpleAction`, `ReportPlanSubprogramAction`, `ReportPlanFcp` |
| Показатели | `GovernmentProgram` ← `IndicatorValue` → `Indicator` → `Unit`; факт отчёта: `RealizationReport` ← `ReportIndicatorValue` → `Indicator` |
| Бюджет по КБК | `KBKFinancing` → `KBK` → `GRBS`, `CSR`, `VR`, `FKR`, `Department`; к программе через `mmFCPKBKFinancing`, `mmSubprgActKBKFinancing` |
| Факт по КБК | `RealizationReport` ← `ReportKBKFinancing` → `KBK` (`FCPId`, `SubprogramActionId`) |
| Структура финансирования | `Subprogram` / `SubprogramAction` / `FCP` ← `*FinancingValue` → `FinancingSource` |
| Региональный срез | `TerritoryIndicatorValue` → `Territory` → `TerritoryCategory` |

## 6. Соответствие примерам отчётов п. 7 брифа

| Отчёт | Источники |
|---|---|
| 1. КС по планам реализации ГП (прил. 2, 3) | `GovernmentProgram`, `Plan`, `PlanSimpleAction`, `PlanKeyEvent`, `ReportPlanKeyEvent`, `ProgramScope` |
| 2. КС по детальным планам-графикам (прил. 4) | `PlanKeyEvent`, `ReportPlanKeyEvent`, `PlanSimpleAction` |
| 3. Контрольные точки пилотных ГП (прил. 7) | `PlanKeyEvent`, `ReportPlanKeyEvent`, `Subprogram`, `FCP` |
| 4. КТ по структурным элементам (прил. 8) | `PlanKeyEvent`, `ReportPlanKeyEvent`, `Subprogram`, `FCP` |
| 5. Достижение целей (прил. 5) | `Indicator`, `IndicatorValue`, `ReportIndicatorValue`, `Unit` |
| 6. Прогноз достижения целей (прил. 6) | `Indicator`, `IndicatorValue`, `ReportIndicatorValue` |
| 7. Кассовое исполнение (прил. 9) | `KBKFinancing`, `ReportKBKFinancing`, `KBK`, `GRBS`, `CashExecution` |
| 8. Расходные обязательства (прил. 10) | `MfDataVolumeCashExecution` (по кодам ГРБС и ВР), справочники `GRBS`, `VR` |
| 9. Структура финансирования | `SubprogramFinancingValue`, `SubprogramActionFinancingValue`, `FCPFinancingValue`, `FinancingSource` |
| 10. Сводная статистика | `GovProgramStatistic` (числовые поля), `FOIV` |

## 7. Что нужно уточнить

1. **Связь `RealizationReport` с программой.** В классе есть только ссылка на версию программы; уточнить, как выбирается отчёт для версии и периода.
2. **Связь данных Минфина** (`MfProgram`, `MfData*`) с программами АИС ГП: через сопоставление версий или по кодам — зависит от того, как заполнен тестовый набор.
3. **Правило выбора значения на дату** для сущностей с `ActualDate` и `OriginalId` (последнее значение на дату или все записи).
4. **Пилотные и непилотные программы.** По ВКР данные двух типов хранятся в родственных, но разных структурах; нужно решить, показывает ли источник оба типа или каждый отдельно.
5. Состав «Доп.» источников, нужных в первой версии.
