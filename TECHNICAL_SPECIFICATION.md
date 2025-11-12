# 🧠 ТЕХНИЧЕСКОЕ ЗАДАНИЕ
## Разработка прототипа многоагентной системы памяти для LLM "АРИАРИХИВАРИУС"

### 1. ОБЩЕЕ ОПИСАНИЕ СИСТЕМЫ

**Цель:** Создание многоагентной системы для решения проблемы потери контекста в длинных диалогах с LLM.

**Ключевые характеристики:**
- Многоагентная архитектура с 6 специализированными модулями
- Бинарный протокол обмена между внутренними модулями
- Система трёх судей для минимизации ошибок
- Низкая латентность (< 2 секунд на полный цикл)
- Горизонтальная масштабируемость

**Архитектура:**
```
Пользователь → РОТ → ШЕРИФ → [СЫЩИК + ЭКСПЕРТ] → [СУДЬЯ_1 + СУДЬЯ_2 + СУДЬЯ_3] → АРХИВАРИУС → Ответ
```

### 2. ТЕХНИЧЕСКИЕ ТРЕБОВАНИЯ

#### 2.1. Базовые требования
- **Язык программирования:** Python 3.9+
- **Протокол обмена:** бинарные структуры данных
- **Время отклика:** < 2000 мс на полный цикл
- **Параллелизм:** асинхронная обработка запросов
- **Память:** циклическая очистка после задач

#### 2.2. Спецификации модулей

##### МОДУЛЬ "РОТ" (интерфейс пользователя)
```python
class RotModule:
    def text_to_binary(self, user_text: str) -> BinaryMessage
    def binary_to_text(self, binary_data: BinaryMessage) -> str
    def maintain_session(self, session_id: str)
```

**Требования:**
- Поддержка русского и английского языков
- Преобразование текст↔бинарный вектор (512-мерный)
- Сессионное управление

##### МОДУЛЬ "ШЕРИФ" (координатор)
```python
class SheriffModule:
    async def coordinate_workflow(self, query: BinaryMessage) -> BinaryMessage
    def manage_timeouts(self) -> bool
    def aggregate_results(self, results: List[BinaryMessage]) -> BinaryMessage
```

**Требования:**
- Параллельный запуск СЫЩИКА и ЭКСПЕРТА
- Таймауты на каждый модуль (< 500 мс)
- Мажоритарное голосование трёх судей

##### МОДУЛЬ "СЫЩИК" (семантический поиск)
```python
class DetectiveModule:
    def semantic_search(self, query_vector: List[float]) -> SearchResults
    def temporal_filter(self, time_range: Tuple[int, int]) -> List[BinaryChunk]
    def relevance_scoring(self, chunks: List[BinaryChunk]) -> List[ScoredChunk]
```

**Требования:**
- Поиск по векторным эмбеддингам (FAISS)
- Фильтрация по временным диапазонам
- Ранжирование по релевантности

##### МОДУЛЬ "ЭКСПЕРТ" (анализ данных)
```python
class ExpertModule:
    def analyze_chunks(self, chunks: List[BinaryChunk]) -> AnalysisResult
    def synthesize_answer(self, analysis: AnalysisResult) -> BinaryMessage
    def calculate_confidence(self, result: AnalysisResult) -> float
```

**Требования:**
- Анализ бинарных структур
- Синтез ответа на основе множества источников
- Оценка уверенности в результате

##### МОДУЛЬ "СУДЬИ" (валидация)
```python
class Judge1:  # Страж точности
    def validate_accuracy(self, data: ValidationRequest) -> Judge1Response

class Judge2:  # Исследователь новизны  
    def validate_novelty(self, data: ValidationRequest) -> Judge2Response

class Judge3:  # Прагматик эффективности
    def validate_efficiency(self, data: ValidationRequest) -> Judge3Response
```

**Требования:**
- Параллельная работа всех трёх судей
- Различные алгоритмы валидации
- Бинарные вердикты (APPROVE/REJECT)

##### МОДУЛЬ "АРХИВАРИУС" (финализация)
```python
class ArchivariusModule:
    def compress_information(self, data: BinaryMessage) -> BinaryMessage
    def update_long_term_memory(self, session_data: BinaryMessage)
    def finalize_response(self, validated_data: BinaryMessage) -> BinaryMessage
```

### 3. БИНАРНЫЕ ПРОТОКОЛЫ - ТЕХНИЧЕСКАЯ СПЕЦИФИКАЦИЯ

#### 3.1. Базовый формат сообщения
```binary
struct BinaryMessage {
    uint32_t header = 0x41524941;  // "ARIA" сигнатура
    uint8_t message_type;          // тип сообщения
    uint8_t version = 0x01;        // версия протокола
    uint16_t payload_length;       // длина полезной нагрузки
    uint8_t session_id[16];        // идентификатор сессии
    uint8_t payload[payload_length]; // полезная нагрузка
    uint32_t checksum;             // контрольная сумма
}
```

#### 3.2. Специфичные структуры данных

**Для поискового запроса:**
```binary
struct SearchRequest {
    uint8_t search_type;           // 0=семантический, 1=временной
    float32_t query_vector[512];   // вектор запроса
    uint64_t time_start;           // начало диапазона
    uint64_t time_end;             // конец диапазона
    uint16_t max_results;          // макс. результатов
    float32_t min_relevance;       // мин. релевантность
}
```

**Для результатов анализа:**
```binary
struct AnalysisResult {
    uint8_t result_type;           // тип результата
    float32_t confidence;          // уверенность (0.0-1.0)
    uint16_t evidence_count;       // кол-во доказательств
    uint64_t evidence_ids[10];     // ID доказательств
    uint16_t data_length;          // длина данных
    uint8_t result_data[];         // данные результата
}
```

### 4. АЛГОРИТМИЧЕСКИЕ ТРЕБОВАНИЯ

#### 4.1. Векторизация текста
```python
# Использование предобученной модели для эмбеддингов
from sentence_transformers import SentenceTransformer

class TextVectorizer:
    def __init__(self):
        self.model = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')
    
    def text_to_vector(self, text: str) -> List[float]:
        return self.model.encode(text).tolist()
```

#### 4.2. Алгоритмы судей

**Судья №1 (пороговый):**
```python
def judge1_algorithm(request: ValidationRequest) -> bool:
    score = 0.0
    if request.expert_confidence > 0.90: score += 0.4
    if request.sources_count >= 2: score += 0.3  
    if not request.has_contradictions: score += 0.3
    return score >= 0.8
```

**Судья №2 (взвешенный):**
```python
def judge2_algorithm(request: ValidationRequest) -> bool:
    score = 0.0
    if request.novelty_flag: score += 0.5
    if request.relevance_score > 0.70: score += 0.3
    if request.importance_level >= 1: score += 0.2
    return score >= 0.65
```

**Судья №3 (композитный):**
```python
def judge3_algorithm(request: ValidationRequest) -> bool:
    score = (request.expert_confidence * 0.6 + 
             request.relevance_score * 0.3 + 
             (1 - request.urgency_level / 255.0) * 0.1)
    return score > 0.75
```

### 5. ИНТЕГРАЦИОННЫЕ ТРЕБОВАНИЯ

#### 5.1. Внешние зависимости
- **Векторная БД:** FAISS для семантического поиска
- **ML модели:** sentence-transformers для эмбеддингов
- **Асинхронность:** asyncio для параллельной обработки
- **Сериализация:** protobuf для бинарных структур

#### 5.2. API endpoints
```python
# Основной endpoint для обработки запросов
@app.post("/process")
async def process_query(user_input: UserQuery) -> ProcessResponse:
    # Полный цикл обработки через все модули
    pass

# Endpoint для мониторинга здоровья системы  
@app.get("/health")
async def health_check() -> HealthStatus:
    pass
```

### 6. КРИТЕРИИ ПРИЕМКИ

#### 6.1. Функциональные тесты
- [ ] Обработка запроса через все модули за < 2000 мс
- [ ] Корректная работа системы трёх судей
- [ ] Преобразование текст ↔ бинарный формат
- [ ] Семантический поиск с релевантностью > 70%
- [ ] Обработка ошибок и таймаутов

#### 6.2. Нагрузочные тесты  
- [ ] Обработка 100+ одновременных запросов
- [ ] Стабильность при 95% загрузке CPU
- [ ] Потребление памяти < 2GB на 1000 сессий

#### 6.3. Интеграционные тесты
- [ ] End-to-end тестирование полного цикла
- [ ] Совместимость бинарных протоколов
- [ ] Восстановление после сбоев модулей

### 7. ПЛАН РАЗРАБОТКИ

**Неделя 1:** Базовая инфраструктура и бинарные протоколы
- [ ] Настройка окружения и зависимостей
- [ ] Реализация базовых бинарных структур
- [ ] Создание скелета всех модулей

**Неделя 2:** Ядро системы - ШЕРИФ и РОТ
- [ ] Модуль РОТ - интерфейс пользователя
- [ ] Модуль ШЕРИФ - координация workflow
- [ ] Интеграция векторной БД (FAISS)

**Неделя 3:** Интеллектуальные модули
- [ ] Модуль СЫЩИК - семантический поиск
- [ ] Модуль ЭКСПЕРТ - анализ данных
- [ ] Система трёх судей

**Неделя 4:** Интеграция и оптимизация
- [ ] Модуль АРХИВАРИУС - финализация
- [ ] Оптимизация производительности
- [ ] Тестирование и отладка

### 8. КРИТИЧЕСКИЕ МОМЕНТЫ

1. **Производительность бинарных операций** - необходим профайлинг
2. **Качество векторных эмбеддингов** - влияет на поисковую релевантность  
3. **Балансировка алгоритмов судей** - требует тонкой настройки
4. **Обработка краевых случаев** - таймауты, ошибки модулей

### 9. КОНТАКТЫ И РЕСУРСЫ

- **Репозиторий проекта:**  https://github.com/ZeroEVO/AI-Memory-Architecture
- **Документация по архитектуре:** PROJECT_SEED.md
- **Контакты для вопросов:** t.me/NEB0ZHITEL

---

**СТАТУС ПРОЕКТА:** Архитектура утверждена, готово к началу разработки POC.

**СЛЕДУЮЩИЙ ШАГ:** Начать реализацию согласно недельному плану разработки.
