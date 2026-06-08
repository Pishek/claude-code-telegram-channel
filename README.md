# claude-code-telegram-channel

Патченный форк официального плагина-канала **`telegram@claude-plugins-official`**
для Claude Code. Связывает сессию Claude Code в терминале с Telegram: на телефон
приходят уведомления об ответах и запросах разрешений, а одобрить/отклонить
действие можно прямо с телефона — или продиктовать ответ с Apple Watch.

Имя marketplace: `cc-channels` · имя плагина: `tg` (ссылаешься на него как
`tg@cc-channels`).

## Чем отличается от официального плагина

Правки vs официального плагина (полный diff —
[`PATCH-permission-ux.diff`](./PATCH-permission-ux.diff)):

1. **Детали разрешения сразу в сообщении.** Первое сообщение показывает ввод
   тула (`command: …` / `file_path: …`), а не только `🔐 Permission: <tool>` под
   кнопкой «See more».
2. **Ответ одним словом.** Помимо строгой формы `yes <id>`, голое
   `yes` / `ok` / `да` / `ок` (allow) и `no` / `нет` (deny) — регистр не важен
   (включая кириллицу), хвостовая пунктуация диктовки вроде `Да.`/`Ок!`
   допускается — применяется к последнему ожидающему запросу. Именно
   это делает удобным Apple Watch: inline-кнопки с уведомления на часах нажать
   нельзя, но можно ответить (Reply) и продиктовать слово. Что-то длиннее
   (`да, давай`) считается обычным сообщением в чат, а не вердиктом.
3. **Маркер сессии в чате.** Когда новая сессия захватывает бота, в чат уходит
   строка-разделитель `▶️ New session · 📁 <папка> · #<тег> · <дата время>` —
   чтобы в истории было видно, где кончилась прошлая сессия и началась текущая
   (бот один на все сессии). Claude Code не отдаёт плагину session id, поэтому
   тег генерится самим плагином, папка — из рабочего каталога.

## Требования

- **Claude Code ≥ 2.1.81** (Channels + permission relay; research preview).
  Аутентификация через claude.ai (Pro/Max) или ключ Console API — не Bedrock/Vertex.
- **[Bun](https://bun.sh)** — плагин это Bun-скрипт (`grammy` + MCP SDK).
- **Токен Telegram-бота** от [@BotFather](https://t.me/BotFather) (`/newbot`).

## Установка

```bash
git clone https://github.com/Pishek/claude-code-telegram-channel.git
```

Затем в сессии Claude Code:

```
/plugin marketplace add /path/to/claude-code-telegram-channel   # зарегистрировать репо как marketplace
/plugin install tg@cc-channels                                  # установить плагин
/reload-plugins                                                 # применить
/tg:configure <BOT_TOKEN>                                       # сохранит токен в ~/.claude/channels/telegram/.env
```

## Запуск

Канал не входит в allowlist research preview от Anthropic, поэтому грузится
флагом development-channels. **Флаг снимает только проверку allowlist — он НЕ
отключает запросы разрешений** (для этого есть отдельный
`--dangerously-skip-permissions`). При старте попросит подтвердить загрузку.

```bash
claude --dangerously-load-development-channels plugin:tg@cc-channels
```

## Пэйринг и блокировка доступа

Напиши боту любое сообщение — он ответит 6-значным кодом. Затем:

```
/tg:access pair <code>          # добавит твой Telegram-ID в allowlist
/tg:access policy allowlist     # запереть канал: пускать только из allowlist
```

Пэйринг разовый и переживает перезапуски (состояние лежит в
`~/.claude/channels/telegram/`, локально на машине — не в этом репозитории).

## Ответ на permission

- **Кнопки** `✅ Allow` / `❌ Deny` / `See more` — в чате (iPhone).
- **Одним словом** — применяется к последнему ожидающему запросу. Allow:
  `yes` / `ok` / `да` / `ок`; deny: `no` / `нет` (регистр не важен, в т.ч.
  кириллица). На Apple Watch: Reply на уведомлении → продиктовать.
- Первый ответ побеждает — запоздалый ответ на уже решённый запрос игнорируется,
  так что отвечать можно в терминале или в Telegram, в любом порядке.

## Настройки

Делёвери/UX-конфиг лежит в `~/.claude/channels/telegram/access.json` и задаётся
через `/tg:access set <key> <value>` (перечитывается на каждое входящее, без
перезапуска):

| Ключ | Значения | Что делает |
|---|---|---|
| `ackReaction` | эмодзи или `""` | реакция на твоё входящее как подтверждение доставки в сессию (например `👀`); `""` — выключить |
| `replyToMode` | `off` \| `first` \| `all` | ставить ли реплай-цепочку на ответах бота |
| `textChunkLimit` | число | порог разбивки длинного ответа |
| `chunkMode` | `length` \| `newline` | как бить длинный текст |
| `mentionPatterns` | JSON-массив regex | на какие упоминания реагировать в группах |

Доступ — подкоманды `/tg:access`: `pair`, `allow <id>`, `remove <id>`,
`policy <pairing\|allowlist\|disabled>`, `group add/rm`. Без аргументов — статус.

Реакции в чате как индикатор: `👀` — доставлено в сессию, `✅`/`❌` — вердикт
разрешения применён.

## Ограничения

- **Один бот = одна активная сессия.** Telegram держит один `getUpdates`-консьюмер
  на токен (форсится pid-локом). Новая сессия забирает бота себе; старая
  продолжает жить в терминале, но теряет мост в Telegram. Для нескольких сессий
  разом — отдельный токен бота + свой `TELEGRAM_STATE_DIR` на каждую.
- **Канал жив, пока запущен `claude`.** Для «всегда на связи» — `tmux`/`screen`.
- **На Apple Watch inline-кнопки нажать нельзя** — это UI Telegram внутри чата, а
  не действия уведомления watchOS. Вместо этого диктуй `да`/`нет`.
- **Research preview** — флаг/протокол каналов могут поменяться.

## Обновление с upstream

Когда выйдет новая версия официального плагина: скопировать его свежий `server.ts`
поверх `tg/server.ts`, переналожить `PATCH-permission-ux.diff` (`git apply` или
вручную), затем `bun build server.ts` для проверки.

## Благодарности и лицензия

Форк официального Telegram-плагина-канала от Anthropic
(`telegram@claude-plugins-official`). Лицензия Apache-2.0 — см.
[`tg/LICENSE`](./tg/LICENSE).
