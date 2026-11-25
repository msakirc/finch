# Finch — Prompt and agent-profile content

> **Standalone.** Split from the [KutAI](https://github.com/msakirc/kutai) monorepo (`packages/finch/`), where it is developed. It does not need the KutAI app; its test suite passes with only its declared dependencies installed.

[English](#english) · [Türkçe](#türkçe)

<a id="english"></a>

## Purpose

Finch owns the **words** KutAI sends to models: every agent profile (system prompt,
tool allow-list, iteration budget, tier) and every overhead rubric (grader, code
review, self-critique, classifier, vision, …), as YAML data, plus the one function
that assembles overhead messages. Keeping content in a dependency-free leaf means a
prompt change is a data change, reviewable on its own, and every caller builds
messages the same way.

**It does NOT** run anything. Finch has no ReAct loop, no LLM calls, no DB access and
no `src.*` imports (`tests/test_purity.py` enforces this). Execution lives in
`coulson` (which drives a `Profile`), dynamic context (mission lessons, tool lists,
calibration) is fetched by the caller and passed in, and versioned prompt overrides
live behind an injected store.

## Install

```bash
pip install "finch @ git+https://github.com/msakirc/finch.git"
```

Requires Python 3.10+. Sibling KutAI packages are installed from their own repositories.

## Quick start

```python
from finch import PROFILE_REGISTRY, build_messages, get_profile

print(len(PROFILE_REGISTRY), "profiles")
reviewer = get_profile("reviewer")
print(reviewer.max_iterations, reviewer.allowed_tools[:3])

messages = build_messages("grading", {
    "title": "Add login",
    "description": "JWT auth for the API",
    "response": "Implemented POST /login returning a signed token.",
})
print([m["role"] for m in messages])
```

## Public API

```python
from finch import (
    get_profile, PROFILE_REGISTRY, Profile,
    build_messages, register_rubric,
    set_store, get_active, record_quality, PromptStore,
    REFLECTION_BLOCKS, STACK_BLOCKS, LAYER_BLOCKS, REFLECT_SYSTEM_BASE,
)

# Agent profiles — parsed once at import from profiles/*.yaml.
p = get_profile("reviewer")              # -> Profile | None
p.allowed_tools, p.max_iterations, p.default_tier
prompt = p.get_system_prompt(task)       # -> str (the YAML seed)

# Overhead prompts — rubrics/*.yaml, keyed by their `key:` field.
messages = build_messages(
    "grading",
    {"title": task["title"], "description": task["description"], "response": out},
    extra_blocks=[lessons_block],        # appended to the system message
)                                        # -> [{"role": "system", ...}, {"role": "user", ...}]

# Versioned overrides: the app injects a DB-backed PromptStore at startup.
set_store(my_store)                      # None -> YAML seeds only
override = await get_active("reviewer")  # -> str | None
await record_quality("reviewer", 0.8)
```

## Key Modules

| module | role |
|---|---|
| `profile.py` | `Profile` dataclass (agent config as data) + `WriterProfile` carve-out |
| `loader.py` | loads `profiles/*.yaml` into `PROFILE_REGISTRY` once at import |
| `build.py` | loads `rubrics/*.yaml`; `build_messages` renders `{field}` tokens |
| `store.py` | `PromptStore` protocol + module-level `get_active` / `record_quality` |
| `reflection_blocks.py` | per-agent / per-stack / per-layer self-reflection text (pure data) |

## Gotchas

- **Adding an agent** = add `profiles/<name>.yaml`. The prompt must pass
  `tests/agents/test_prompt_quality.py`: first line `You are ...`, the body uses
  must/always **and** don't/never, and it contains `final_answer` plus a fenced
  ` ```json ` schema.
- **Per-agent reflection blocks are added here**, in
  `reflection_blocks.REFLECTION_BLOCKS` — `coulson.reflection` only re-exports them.
  The composition logic (which blocks to glue) stays in coulson.
- `build_messages` substitutes only the explicit `{key}` tokens it is given, in one
  pass: a field value containing another token (e.g. a task description with the
  literal text `{response}`) survives verbatim. Other braces are left alone.
- A DB override returned by the store wins over the YAML seed; the seed in
  `system_prompt` is the frozen reference.
- The `Profile` fields starting with `_` and `progress_callback` are per-execution
  state set and restored by coulson, not profile data.

## Tests

```powershell
python -m pytest packages/finch/tests
```

## In KutAI

Coulson reads agent profiles from Finch; Hüsam's grader, the classifier and the reflection hooks build their prompts with `build_messages`. The app injects a DB-backed `PromptStore` for versioned overrides. Source: [`packages/finch/`](https://github.com/msakirc/kutai/tree/main/packages/finch) in the [KutAI monorepo](https://github.com/msakirc/kutai), where this package is developed.

## Status and limitations

- Standalone: it does not import the KutAI app, and its tests pass with only its declared dependencies.
- Version 0.1.0, part of a personal research project. Developed and tested on Windows 11 with Python 3.10; other platforms are untested.
- The profiles and rubrics are written for KutAI's agents and tools; reuse them as examples rather than drop-ins.

## License

MIT. See [LICENSE](https://github.com/msakirc/kutai/blob/main/LICENSE).

---
<a id="türkçe"></a>
## Türkçe

### Amaç

Finch, KutAI'nin modellere gönderdiği **kelimelerin** sahibidir: her ajan profili
(sistem istemi, araç izin listesi, yineleme bütçesi, katman) ve her yardımcı
değerlendirme şablonu (puanlayıcı, kod incelemesi, öz eleştiri, sınıflandırıcı,
görüntü, …) YAML verisi olarak burada durur; yardımcı mesajları kuran tek fonksiyon
da buradadır. İçeriği bağımlılığı olmayan bir yaprak pakette tutmak, istem
değişikliğini kendi başına incelenebilen bir veri değişikliği yapar ve her çağıran
mesajları aynı şekilde kurar.

Finch **hiçbir şey çalıştırmaz.** ReAct döngüsü, LLM çağrısı, veritabanı erişimi ve
`src.*` içe aktarımı yoktur (`tests/test_purity.py` bunu zorlar). Çalıştırma
`coulson`'dadır (bir `Profile`'ı o sürer); dinamik bağlamı (görev dersleri, araç
listeleri, kalibrasyon) çağıran getirip içeri verir; sürümlü istem geçersiz
kılmaları enjekte edilen bir deponun arkasındadır.

### Kurulum

```bash
pip install "finch @ git+https://github.com/msakirc/finch.git"
```

Python 3.10+ gerekir. Kardeş KutAI paketleri kendi depolarından kurulur.

### Hızlı başlangıç

```python
from finch import PROFILE_REGISTRY, build_messages, get_profile

print(len(PROFILE_REGISTRY), "profiles")
reviewer = get_profile("reviewer")
print(reviewer.max_iterations, reviewer.allowed_tools[:3])

messages = build_messages("grading", {
    "title": "Add login",
    "description": "JWT auth for the API",
    "response": "Implemented POST /login returning a signed token.",
})
print([m["role"] for m in messages])
```

### Genel API

```python
from finch import (
    get_profile, PROFILE_REGISTRY, Profile,
    build_messages, register_rubric,
    set_store, get_active, record_quality, PromptStore,
    REFLECTION_BLOCKS, STACK_BLOCKS, LAYER_BLOCKS, REFLECT_SYSTEM_BASE,
)

# Ajan profilleri — içe aktarımda profiles/*.yaml'dan bir kez okunur.
p = get_profile("reviewer")              # -> Profile | None
prompt = p.get_system_prompt(task)       # -> str (YAML tohumu)

# Yardımcı istemler — rubrics/*.yaml, `key:` alanıyla anahtarlanır.
messages = build_messages(
    "grading",
    {"title": task["title"], "description": task["description"], "response": out},
    extra_blocks=[lessons_block],        # sistem mesajının sonuna eklenir
)

# Sürümlü geçersiz kılmalar: uygulama açılışta veritabanı destekli bir PromptStore enjekte eder.
set_store(my_store)                      # None -> yalnızca YAML tohumları
override = await get_active("reviewer")  # -> str | None
```

### Dikkat Edilecekler

- **Yeni ajan** = `profiles/<ad>.yaml` ekle. İstem `tests/agents/test_prompt_quality.py`
  testini geçmeli: ilk satır `You are ...`, gövdede must/always **ve** don't/never,
  ayrıca `final_answer` ve çitli bir ` ```json ` şeması.
- **Ajan başına yansıma blokları buraya**, `reflection_blocks.REFLECTION_BLOCKS`
  içine eklenir — `coulson.reflection` yalnızca yeniden dışa aktarır.
- `build_messages` yalnızca verilen açık `{key}` belirteçlerini tek geçişte değiştirir;
  başka bir belirteç içeren alan değeri aynen korunur.
- Depodan dönen veritabanı geçersiz kılması YAML tohumunu ezer.

### Testler

```powershell
python -m pytest packages/finch/tests
```

### KutAI içindeki yeri

Coulson ajan profillerini Finch'ten okur; Hüsam'ın puanlayıcısı, sınıflandırıcı ve yansıma kancaları istemlerini `build_messages` ile kurar. Uygulama sürümlü geçersiz kılmalar için veritabanı destekli bir `PromptStore` enjekte eder. Kaynak: [KutAI monorepo](https://github.com/msakirc/kutai) içinde [`packages/finch/`](https://github.com/msakirc/kutai/tree/main/packages/finch); paket orada geliştirilir.

### Durum ve sınırlamalar

- Bağımsız: KutAI uygulamasını import etmez; testleri yalnızca bildirilen bağımlılıklarla geçer.
- Sürüm 0.1.0, kişisel bir araştırma projesinin parçası. Windows 11 ve Python 3.10 üzerinde geliştirilip test edildi; diğer platformlar denenmedi.
- Profiller ve rubrikler KutAI'nin ajanları ve araçları için yazıldı; doğrudan değil örnek olarak kullanın.

### Lisans

MIT. [LICENSE](https://github.com/msakirc/kutai/blob/main/LICENSE) dosyasına bakın.
