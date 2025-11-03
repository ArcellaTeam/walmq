# `walmq` — Архитектура

**WAL-based embedded message queue for local and edge applications**

---

### 🎯 Цель

Обеспечить **лёгкий, надёжный и встраиваемый** брокер сообщений для приложений на Rust, где:

- Сообщения **не должны теряться** при сбое.
- Система работает **локально** (без внешних зависимостей).
- Нужна **простая интеграция** в runtime’ы (например, Arcella).
- Требуется **чёткое разделение метаданных и данных**.

> `walmq` — это **не распределённый брокер**, а **локальный, durable message queue на WAL**.

---

### 🧩 Ключевые принципы

1. **Durability прежде всего** — сообщения и метаданные переживают падение.
2. **Разделение ответственности**:
   - **Метаданные** → `ministore`
   - **Сообщения** → специализированный `MessageJournal`
3. **Минимализм** — только необходимый функционал: publish, subscribe, ack.
4. **Производительность** — сообщения хранятся в бинарном виде, без serde.
5. **Встраиваемость** — zero-config, single binary.

---

### 🏗️ Архитектура

```
┌───────────────────────────────────────────────┐
│                  walmq                        │
├───────────────────────┬───────────────────────┤
│    Metadata Plane     │      Data Plane       │
│  (управление)         │  (сообщения)          │
├───────────┬───────────┼───────────┬───────────┤
│ MiniStore │           │ Message   │           │
│<BrokerMd> │◄─evts───► │ Journal   │           │
│(ministore)│           │ (WAL)     │           │
└───────────┴───────────┴───────────┴───────────┘
```

- **`ministore`** (внешняя зависимость):  
  Хранит состояние брокера:  
  ```rust
  struct BrokerMd {
      topics: HashMap<String, TopicConfig>,
      consumers: HashMap<String, ConsumerGroup>,
      offsets: HashMap<(String, String), u64>,
  }
  ```

- **`MessageJournal`** (внутренний WAL):  
  - Один файл на топик: `topics/<name>.wal`  
  - Бинарный формат: `[len: u32][payload: bytes]`  
  - Поддержка чтения по смещению  
  - Retention по времени/размеру

---

### 🔁 Поток данных

1. **Публикация**:
   ```rust
   broker.publish("logs", b"hello").await?;
   ```
   → запись в `MessageJournal::logs.wal` → `fsync`

2. **Подписка**:
   ```rust
   let mut stream = broker.subscribe("group1", "logs").await?;
   while let Some(msg) = stream.recv().await? {
       // handle
       msg.ack().await?; // → обновляет offset в ministore
   }
   ```

3. **Создание топика**:
   ```rust
   broker.create_topic("logs", config).await?;
   ```
   → `ministore.apply(CreateTopic { ... })`

---

### 📦 Зависимости

- **Обязательные**:
  - `ministore` — для метаданных
  - `tokio`, `serde`, `thiserror`
- **Опциональные** (фичи):
  - `network` — для TCP/Unix-сокетного API
  - `encryption` — шифрование журналов

---

### 🚫 Что НЕ делает `walmq`

- Не поддерживает кластеризацию (пока).
- Не предоставляет HTTP API по умолчанию (только через фичу `network`).
- Не хранит сообщения в `ministore` (только метаданные).

---

### 🔗 Интеграция с Arcella

`walmq` может быть использован в Arcella для:
- Внутренней шины событий между компонентами.
- Очередей задач для `arcella-worker`.
- Логгирования через публикацию в топик `logs`.

> 💡 Arcella будет **потребителем `walmq`**, а не его частью.

---

### 📅 Дорожная карта

| Версия | Фича |
|-------|------|
| v0.1 | Локальный брокер: publish/subscribe, ack, retention |
| v0.2 | Unix-сокетный API (для CLI и внешних клиентов) |
| v0.3 | Поддержка партиций |
| v0.4 | Replication через WAL (для кластера) |

---

> **`walmq` — это то, что вы получите, если соедините `ministore` и специализированный WAL для сообщений.**  
> Надёжно. Просто. Встраиваемо.