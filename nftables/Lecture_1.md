# От iptables к nftables

`nftables` – современный файрвол Linux, замена устаревшему `iptables`. Во всех новых дистрибутивах (RHEL 9, Ubuntu 22.04+, Debian 12+) это стандарт. Для системного администратора и DevOps это значит: один синтаксис для IPv4/IPv6, атомарное применение правил и лучшая производительность.

## Базовые команды

Прежде чем строить правила, научимся смотреть и очищать текущее состояние.

```bash
# Проверить версию
nft --version

# Посмотреть все текущие правила
sudo nft list ruleset

# Сбросить все правила
sudo nft flush ruleset
```

## Ключевые концепции

В `nftables` всё строится из трёх кирпичей:

- **Table** – контейнер для правил. Создаётся под конкретное семейство адресов.

- **Chain** – цепочка правил, привязанная к хуку сетевого стека и имеющая приоритет.

- **Rule** – конкретное правило фильтрации в формате: условие + действие.

```bash
# Создание таблицы для IPv4 и IPv6 одновременно (семейство адресов inet)
nft add table inet firewall

# Цепочка input с политикой DROP по умолчанию
nft add chain inet firewall input { type filter hook input priorirty 0 \; policy drop \; }

# Разрешить SSH-соединение (22 порт)
nft add rule inet firewall input tcp dport 22 accept
```

Дополнительно есть мощные структуры данных:

- `set` – набор значений (IP, порты, интерфейсы). Вместо десяти правил можно прописать одно для нескольких IP-адресов, например.

- `map` – ассоциативный массиа, например "IP → действие".

```bash
# Создание множества доверенных IP-адресов с именем trusted
nft add set inet firewall trusted { type ipv4_addr \; elements = { 10.0.0.5, 10.0.0.6 } \; }

# Добавление правила для множества trusted
nft add rule inet firewall input ip saddr @trusted tcp dport 22 accept
```

**Более подробно семейства адресов, хуки сетевого стека и приоритеты рассмотрим в следующей лекции.**

## Политики по умолчанию и минимальные привилегии

Главный приницип файрвола – запрещать всё, разрешать нужное. Это значит:

- Политика цепочки по умолчанию – `drop` или `reject`.

- Явно разрешаем только то, что необходимо: loopback, установленные соединения, нужные порты.

```bash
# Разрешаем весь трафик на loopback-интерфейсе (127.0.0.1).
nft add rule inet firewall input iif lo accept

# Разрешаем пакеты, которые относятся к уже установленным соединениям.
nft add rule inet firewall input ct state established,related accept

# Разрешаем входящие TCP-соединения на порт 443 с любого IP-адреса.
nft add rule inet firewall input tcp dport 443 accept
```

Разница между `drop` и `reject`:

- `drop` – пакет отбрасывается без отправки сообщения об ошибке. Выбирается в большинстве случаев для внешних интерфейсов.

- `reject` – отправляется ICMP/TCP ошибка.

## Миграция с iptables

Для облегчения перехода можно конвертировать правила iptables в nftables с помощью утилит `iptables-translate`, `iptables-restore-translate`, `iptables-nft-restore` и т.п

```bash
# Перевести одну команду
iptables-translate -A INPUT -p tcp --dport 80 -j ACCEPT
# → nft add rule ip filter INPUT tcp dport 80 accept

# Перевести весь ruleset
iptables-save > old.rules
iptables-restore-translate -f old.rules > new.nft
```

### Почему это важно для администратора

- **Защита серверов**: корректный файрвол – основа периметральной безопасности.
- **Docker и Kubernetes**: обе системы активно используют netfilter. Понимание `nftables` необходимо, чтобы их правила не конфликтовали с вашими.
- **Современные дистрибутивы**: в RHEL 9, Debian 12, Ubuntu 22.04+ `iptables` – это лишь обёртка над `nftables`. Знать оригинал полезнее.
- **Compliance**: требования PCI DSS, ISO 27001 и аналоги прямо говорят о контроле сетевого трафика.

### Takeaways

- `nftables` – современный стандарт файрвола в Linux, замена `iptables`.
- Структура: `table` → `chain` → `rule`, плюс `sets` и `maps` для группировки.
- Политика по умолчанию – drop`, разрешаем только нужное.
- Миграция с `iptables` автоматизируется через `iptables-restore-translate`.
- Всегда проверяйте правила и имейте план отката при работе по SSH.
