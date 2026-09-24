# Backroom Lab — зафиксированное направление разработки

**Статус:** принятое архитектурное решение  
**Дата:** 2026-09-24  
**Репозиторий:** `SUKUNA-AI/Backroom`

## 1. Что такое Backroom

Backroom — экспериментальный Windows-first C++/CUDA-проект по исследованию инференса очень больших sparse/MoE-моделей на обычном desktop-железе с ограниченным объёмом VRAM.

Основной target — **Qwen3.8-Flash-Next** на машине с RTX 5070 Ti 16 GiB, 64 GiB RAM и быстрым NVMe. Qwen3.6-35B-A3B используется как первая лабораторная модель; Qwen3-Next-80B-A3B — как промежуточная проверка масштабирования перед Flash-Next.

Главный вопрос проекта:

> Как заставить физически очень большую sparse-модель интерактивно работать на consumer hardware, когда модель существенно больше доступной VRAM, и при этом понимать, где именно тратятся память, пропускная способность и время каждого токена?

Backroom не должен становиться ещё одним универсальным inference framework. Он строится как узкий исследовательский runtime и набор экспериментальных компонентов под конкретный класс моделей и конкретное железо.

---

## 2. Главное принятое решение: QwFNfer не портируем

Ранее рассматривался вариант взять QwFNfer как основную кодовую базу, форкнуть его и перенести Linux-first реализацию на Windows.

От этого решения отказались.

Причина не в том, что QwFNfer плохой. Наоборот, он остаётся одним из важнейших референсов проекта. Но портирование чужого специализированного inference engine означает необходимость сначала глубоко изучить чужие архитектурные решения, а затем постепенно переделывать их под нашу модель исполнения, Windows, наше железо и будущие эксперименты.

Это создаёт слишком большой объём работы, который плохо соответствует учебной и исследовательской цели Backroom.

Поэтому:

- **не форкаем QwFNfer как фундамент проекта;**
- **не ставим цель сделать “QwFNfer for Windows”;**
- **не обязуемся сохранять его архитектуру;**
- **не переносим весь engine слой за слоем.**

QwFNfer становится:

- референсом;
- донором идей;
- источником benchmark-методик;
- источником конкретных решений;
- местом, куда можно прийти с одним точным вопросом и посмотреть, как автор решил похожую проблему.

Например, если понадобится понять, как организовать индекс expert weights, eviction, whole-slice reads, prefetch или layout cache, можно изучить соответствующий кусок QwFNfer. Но нет необходимости понимать и переносить весь проект целиком.

---

## 3. Backroom пишется под Windows изначально

Основная платформа разработки и исполнения:

```text
Windows 11
Intel Core i7-14700KF
RTX 5070 Ti 16 GiB
64 GiB DDR5
NVMe SSD
MSVC + C++20
CUDA
CMake + Ninja
```

Linux остаётся полезным как внешний reference environment: на нём можно запускать существующие реализации, сравнивать результаты и получать routing/performance traces. Но Backroom не строится по схеме “сначала Linux, потом порт”.

Windows-specific компоненты проектируются сразу вокруг Win32 и CUDA на Windows.

Для storage path это означает, что нас интересуют в первую очередь:

- `CreateFileW`;
- `OVERLAPPED` I/O;
- IOCP;
- `FILE_FLAG_NO_BUFFERING` там, где это оправдано;
- требования к alignment;
- собственная очередь запросов;
- управление completion/cancellation;
- page-locked/pinned host memory;
- `cudaMemcpyAsync`;
- CUDA streams/events;
- измерение overlap между I/O, H2D и compute.

Windows здесь не временное ограничение и не проблема, которую надо обойти. Это часть исследовательского предмета проекта.

---

## 4. Мы НЕ пишем весь inference stack с нуля

Отказ от форка QwFNfer не означает, что Backroom должен заново изобретать tokenizer, GGUF, все quant formats, SIMD, GEMM и весь остальной ML runtime stack.

Проект делит код на три категории.

### 4.1. Пишем сами

Сами реализуем то, ради чего Backroom существует и чему хотим научиться:

- Windows asynchronous storage layer;
- aligned/unbuffered I/O experiments;
- pinned-memory management;
- device-memory management;
- асинхронный путь NVMe → RAM → VRAM;
- storage scheduler;
- ExpertStore;
- ExpertCache;
- VRAM/RAM/NVMe residency management;
- eviction policies;
- prefetch;
- routing statistics и routing-aware policies;
- tracing и profiling;
- benchmark infrastructure;
- instrumentation;
- позднее — отдельные CUDA kernels;
- позднее — собственные эксперименты с quantization/correction.

Именно эти компоненты являются основной инженерной и исследовательской ценностью проекта.

### 4.2. Используем готовое

Не тратим месяцы на повторное написание инфраструктурных компонентов, которые уже качественно реализованы:

- GGUF infrastructure;
- tokenizer;
- стандартные quant formats;
- базовая tensor math;
- CPU SIMD kernels;
- зрелые CUDA primitives;
- reference implementations отдельных операций.

В качестве базы/доноров могут использоваться:

- `ggml`;
- `llama.cpp`;
- CUTLASS;
- FlashInfer;
- другие подходящие библиотеки.

### 4.3. Адаптируем и заимствуем

Есть промежуточная категория: код, который не является нашей исследовательской целью, но нужен для корректного исполнения конкретной модели.

Сюда могут относиться:

- реализация конкретного блока Qwen;
- tensor mapping;
- GGUF indexing;
- router code;
- model-specific layout;
- отдельные utility-функции;
- хорошо реализованные алгоритмы cache/prefetch.

Если такой код уже есть в QwFNfer, llama.cpp или другом проекте, нормальный инженерный путь — изучить его, понять, адаптировать или перенести нужную часть вместо того, чтобы принципиально писать всё заново.

При прямом использовании или адаптации чужого кода соблюдаются лицензии и attribution.

Основное правило проекта:

> **Исследовательские компоненты пишем сами. Инфраструктурную сантехнику переиспользуем. Уже решённые model-specific части адаптируем, если это экономит время и не уничтожает смысл исследования.**

---

## 5. Почему разработка идёт по слоям

Backroom не должен начинаться с попытки сразу реализовать полный `QwenModel::forward()`.

Если одновременно добавить model graph, GGUF, MoE, async I/O, cache, CUDA, квантование и Windows-specific storage, то любая ошибка превращается в поиск среди десяти неизвестных.

Поэтому каждый следующий слой строится только после того, как предыдущий можно отдельно проверить и измерить.

Важно: первые этапы не являются одноразовыми “игрушечными тестами”. Они должны превращаться в реальные компоненты будущего runtime.

---

## 6. Этапы разработки

### Этап 0. Скелет Backroom

Создаётся минимальный C++20/CUDA проект.

Ориентировочная структура:

```text
Backroom/
├── CMakeLists.txt
├── cmake/
├── include/
├── src/
│   ├── platform/
│   │   └── windows/
│   ├── storage/
│   ├── memory/
│   ├── cuda/
│   ├── cache/
│   ├── model/
│   └── tracing/
├── bench/
├── tests/
├── third_party/
└── docs/
```

На этом этапе не нужны:

- сервер;
- GUI;
- Qt;
- tokenizer;
- agent layer;
- полный model runtime.

Цель этапа:

- проект собирается MSVC/NVCC;
- есть тестовый каркас;
- есть benchmark harness;
- есть базовое логирование;
- CUDA device корректно определяется;
- границы модулей понятны.

---

### Этап 1. Windows storage path

Первая серьёзная собственная подсистема.

Берётся большой обычный файл, разбитый на блоки, похожие по размеру на будущие expert slices.

Backroom должен уметь:

```text
NVMe
  ↓
asynchronous read
  ↓
aligned host buffer
```

Изучаются и реализуются:

- RAII для Win32 handles;
- `CreateFileW`;
- `OVERLAPPED`;
- IOCP;
- submission/completion;
- queue depth;
- cancellation;
- alignment;
- buffered vs unbuffered reads;
- throughput и latency distributions.

На этом этапе модель не нужна.

---

### Этап 2. Память CPU/GPU

Добавляется реальный путь:

```text
NVMe
  ↓
Pinned RAM
  ↓
cudaMemcpyAsync
  ↓
VRAM
```

Появляются реальные reusable компоненты:

- aligned host buffer;
- pinned buffer;
- pinned-memory pool;
- device buffer;
- device arena;
- CUDA stream wrapper;
- CUDA event wrapper;
- тайминги и трассировка.

Основной вопрос этапа:

> Насколько хорошо можно перекрывать чтение с NVMe, передачу H2D и GPU compute на Windows?

---

### Этап 3. Block Store / Block Cache

До подключения LLM вводится абстрактный блок:

```text
BlockId
offset
size
location
```

Появляется трёхуровневое размещение:

```text
VRAM
RAM
NVMe
```

Реализуются базовые политики:

- LRU;
- LFU;
- простой hybrid.

Для тестов используются искусственные access traces.

Цель — получить рабочую и измеряемую систему residency/eviction без зависимости от корректности нейросети.

---

### Этап 4. Реальные routing traces Qwen3.6

Qwen3.6-35B-A3B запускается через существующий reference runtime, например llama.cpp.

Снимаются реальные данные вида:

```text
token
layer
selected expert ids
router weights
```

Эти traces подаются в наш cache/storage simulator/runtime.

После этого можно исследовать уже реальные свойства MoE routing:

- распределение популярности экспертов;
- горячих/холодных экспертов;
- корреляцию между соседними токенами;
- переходы между экспертами;
- hit rate при разных VRAM budgets;
- потенциальную предсказуемость prefetch;
- различия между типами prompt/workload.

Backroom всё ещё может не генерировать собственные токены, но уже работает с реальным паттерном доступа настоящей модели.

---

### Этап 5. ExpertStore и реальные веса GGUF

Подключается готовая GGUF infrastructure.

Мы не пишем формат GGUF заново.

Строится собственный индекс:

```text
(layer, expert, tensor)
        ↓
file / shard
offset
size
quant type
```

После этого storage/cache subsystem начинает перемещать реальные expert tensors Qwen3.6 вместо искусственных блоков.

---

### Этап 6. Один настоящий MoE layer

Это первый этап, где Backroom действительно выполняет часть нейросети.

Путь:

```text
hidden state
    ↓
router
    ↓
top-k experts
    ↓
ExpertStore / ExpertCache
    ↓
expert compute
    ↓
combine
```

Для базовой quantized math используются существующие ggml/CUDA primitives.

Результат слоя сравнивается с reference implementation.

Это важный milestone: настоящий tensor Qwen проходит через наш storage/cache path и даёт корректный численный результат.

---

### Этап 7. Полный блок Qwen3.6

К MoE добавляются остальные необходимые операции конкретного блока модели:

- normalization;
- Gated DeltaNet / attention;
- residual paths;
- state handling;
- прочие model-specific операции.

Model-specific математика может быть адаптирована из reference implementations. Нет цели заново изобретать формулы только ради количества собственного кода.

Главное требование — понимать execution path и иметь возможность сравнивать intermediate tensors с reference.

---

### Этап 8. Полный decode Qwen3.6

Только после предыдущих этапов появляется собственный узкий inference runtime:

```text
token
  ↓
layers
  ↓
logits
  ↓
next token
```

Первая полноценная версия намеренно ограничена:

```text
Windows only
NVIDIA only
Qwen3.6 only
batch = 1
text only
decode first
без универсальной model abstraction
```

После этого принимается отдельное решение: есть ли смысл продолжать до более универсального runtime или полезнее сосредоточиться на cache/prefetch/quantization/performance research.

---

## 7. Что исследуем после появления рабочего пути

Только после стабильного baseline имеет смысл начинать более серьёзные эксперименты:

- разные VRAM budgets;
- GPU expert hit rate;
- RAM hit rate;
- NVMe miss rate;
- разные eviction policies;
- routing-aware placement;
- predictive prefetch;
- co-occurrence/transition statistics;
- read amplification;
- request size / queue depth;
- H2D overlap;
- kernel launch/synchronization overhead;
- quantization trade-offs;
- mixed precision для hot/cold experts;
- low-rank/sparse residual correction;
- собственные CUDA kernels там, где profiler показывает смысл.

Обязательный цикл каждого performance-изменения:

```text
гипотеза
→ реализация
→ проверка корректности
→ микробенчмарк
→ end-to-end benchmark
→ profiler
→ проверка качества
→ решение оставить/выкинуть
```

---

## 8. Лестница моделей

### Qwen3.6-35B-A3B

Первая лабораторная модель.

Задачи:

- понять MoE routing;
- научиться работать с expert tensors;
- исследовать residency/cache;
- построить первый работающий decode path;
- намеренно ограничивать VRAM и наблюдать деградацию.

Qwen3.6 рассматривается прежде всего как лабораторный организм, а не как конечный продукт.

### Qwen3-Next-80B-A3B

Промежуточный масштабный тест.

Нужен, чтобы понять, сохраняются ли найденные закономерности при значительно большем физическом размере модели и более жёстком storage pressure.

### Qwen3.8-Flash-Next

Конечная исследовательская цель.

Здесь к уже изученным MoE/storage/cache задачам добавляются более сложные архитектурные механизмы, включая QSA/Gated DeltaNet, Gated Residual, большую n-gram/lookup memory и MTP.

Цель не формулируется как гарантированные “20 tok/s”. Интерактивная скорость — target, который должен подтверждаться измерениями.

---

## 9. Роль QwFNfer после изменения стратегии

QwFNfer остаётся важным проектом для Backroom, но используется точечно.

Нормальный сценарий работы:

1. Возник конкретный инженерный вопрос.
2. Сначала формулируем, что именно хотим решить.
3. Смотрим paper/docs/reference implementations.
4. Если QwFNfer уже решает похожую задачу — изучаем только релевантный код.
5. Решаем, что лучше:
   - использовать идею;
   - адаптировать код;
   - перенести отдельный фрагмент;
   - написать собственную реализацию.
6. Проверяем решение своим benchmark/profiler.

Ненормальный сценарий:

> читать весь QwFNfer целиком только потому, что когда-то собирались его портировать.

Так больше не делаем.

---

## 10. Что пока намеренно НЕ входит в Backroom Lab

Чтобы не раздувать проект до бесконечности, следующие вещи не являются ближайшей задачей:

- агентная orchestration;
- LangGraph;
- cloud routing;
- домашний AI/Jarvis;
- Qt GUI;
- HTTP/OpenAI server;
- multi-user serving;
- distributed inference;
- multi-GPU server;
- vision;
- MTP на первых этапах;
- универсальный inference framework;
- статья как обязательный deliverable.

Они могут появиться позже, но не определяют первые этапы разработки.

---

## 11. Первый практический milestone

Первая версия Backroom не обязана генерировать ни одного токена.

### Backroom v0.1

Цель:

> Надёжно и измеримо перемещать model-sized blocks по пути `NVMe → pinned RAM → VRAM` на native Windows.

Необходимые свойства:

- C++20;
- MSVC;
- CUDA;
- CMake/Ninja;
- Win32 async I/O;
- reusable buffers;
- pinned-memory pool;
- CUDA stream/event wrappers;
- корректные тайминги;
- latency/throughput benchmark;
- тесты;
- никаких LLM-зависимостей в core storage benchmark.

Следующие версии постепенно добавляют:

```text
v0.1  storage + pinned RAM + H2D
v0.2  block cache / residency
v0.3  real Qwen routing traces
v0.4  real expert weights
v0.5  one MoE layer
...
```

Версии здесь — ориентиры, а не обязательная публичная схема релизов.

---

## 12. Критерий успеха проекта

Успех Backroom — не количество самостоятельно переписанных строк и не сам факт запуска большой модели.

Главный результат — способность объяснить полный hot path и количественно показать, где находится bottleneck.

Например, вместо ответа:

> “Модель большая, поэтому токен медленный.”

Backroom должен приводить к ответу уровня:

> “При текущем VRAM budget основная часть tail latency приходит из cold expert misses; read amplification и недостаточное перекрытие H2D с compute дают такую-то долю времени; после изменения cache/prefetch bottleneck смещается в конкретный kernel.”

Именно это является основной образовательной и инженерной целью Lab.

---

## 13. Зафиксированное решение в одном абзаце

**Backroom — собственный Windows-first C++/CUDA runtime и исследовательская лаборатория для sparse LLM inference. QwFNfer целиком не портируется и не используется как основная кодовая база. Ключевые системные части — Windows I/O, память, кэш экспертов, residency, prefetch, tracing и performance-исследования — пишутся самостоятельно. Инфраструктурные и уже качественно решённые части — GGUF, tokenizer, quantized math, model-specific фрагменты — переиспользуются или точечно адаптируются из ggml, llama.cpp, QwFNfer, CUTLASS, FlashInfer и других подходящих проектов с соблюдением лицензий. Разработка идёт слоями от storage/memory microbenchmarks к реальным routing traces, expert weights, одному MoE layer, затем к Qwen3.6 decode, после чего масштабируется через Qwen3-Next к Qwen3.8-Flash-Next.**
