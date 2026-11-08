# Сценарий 4. Write skew (аномалия сериализации)

## Идея

Правило банка: **сумма счетов 1 и 3 не может уйти в минус**.
Начальные балансы: счёт 1 = 1000, счёт 3 = 300, сумма = **1300**.

Две транзакции независимо проверяют правило, обе видят сумму 1300 и решают, что снять 1200 можно.
Они меняют **разные строки** (A — счёт 1, B — счёт 3), поэтому прямого конфликта нет.
По отдельности каждая транзакция права, а вместе они нарушают правило.

**Начальные данные**

```sql
DROP TABLE IF EXISTS shop.accounts;
CREATE TABLE shop.accounts (
  id int PRIMARY KEY, owner text, balance numeric NOT NULL
);
INSERT INTO shop.accounts VALUES
  (1, 'Айбек', 1000), (2, 'Нурия', 500), (3, 'Айбек (сбер.)', 300);
```

---

## 4.1. REPEATABLE READ: правило нарушено

> TODO: выполнить и вставить свой вывод.

### Шаги

| № | Сессия A | Сессия B |
|---|----------|----------|
| 1 | `BEGIN ISOLATION LEVEL REPEATABLE READ;` | `BEGIN ISOLATION LEVEL REPEATABLE READ;` |
| 2 | `SELECT sum(balance) FROM shop.accounts WHERE id IN (1,3);` → 1300 | `SELECT sum(balance) FROM shop.accounts WHERE id IN (1,3);` → 1300 |
| 3 | `UPDATE shop.accounts SET balance = balance - 1200 WHERE id = 1;` | |
| 4 | | `UPDATE shop.accounts SET balance = balance - 1200 WHERE id = 3;` |
| 5 | `COMMIT;` | |
| 6 | | `COMMIT;` |
| 7 | `SELECT sum(balance) FROM shop.accounts WHERE id IN (1,3);` → **-1100** | |

### Вывод psql

```
(вставьте вывод сессии A и сессии B)
```

### Объяснение

- Снимок A и снимок B сделаны на первом `SELECT`; в обоих сумма 1300.
- Транзакции меняют разные строки, поэтому блокировок и конфликтов нет.
- Обе транзакции успешно сделали `COMMIT`. Итог: −200 + (−900) = **−1100**, правило нарушено.

---

## 4.2. SERIALIZABLE: PostgreSQL отменяет одну из транзакций

### Шаги

| № | Сессия A | Сессия B |
|---|----------|----------|
| 1 | `BEGIN ISOLATION LEVEL SERIALIZABLE;` | `BEGIN ISOLATION LEVEL SERIALIZABLE;` |
| 2 | `SELECT sum(balance) FROM shop.accounts WHERE id IN (1,3);` → 1300 | `SELECT sum(balance) FROM shop.accounts WHERE id IN (1,3);` → **1300** |
| 3 | `UPDATE shop.accounts SET balance = balance - 1200 WHERE id = 1;` | |
| 4 | | `UPDATE shop.accounts SET balance = balance - 1200 WHERE id = 3;` → `UPDATE 1` |
| 5 | `COMMIT;` | |
| 6 | | `COMMIT;` → **ERROR: could not serialize access** |
| 7 | | `ROLLBACK;` |
| 8 | `SELECT sum(balance) FROM shop.accounts WHERE id IN (1,3);` → **100** (проверить и вписать) | |

### Вывод psql: сессия B

```
B > BEGIN ISOLATION LEVEL SERIALIZABLE;
BEGIN
B *> SELECT sum(balance) FROM shop.accounts WHERE id IN (1,3);
 sum
------
 1300
(1 row)

B *> UPDATE shop.accounts SET balance = balance - 1200 WHERE id = 3;
UPDATE 1
B *> commit;
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```

### Вывод psql: сессия A

```
(вставьте вывод сессии A)
```

### Что произошло

1. Обе транзакции на первом `SELECT` получили **один снимок на всю транзакцию** и увидели сумму 1300.
2. Каждая прочитала данные, которые потом изменила другая транзакция. Между транзакциями возникли зависимости «чтение → запись» (*read/write dependencies*).
3. PostgreSQL заметил, что эти зависимости образуют цикл: результат нельзя получить, выполняя транзакции по очереди ни в каком порядке.
4. Транзакция B попала в положение «pivot» (это строка `Canceled on identification as a pivot` из вывода): она одновременно зависит от другой транзакции и сама влияет на неё. Такую транзакцию PostgreSQL отменяет.
5. Ошибка возникла именно **на `COMMIT`** (`during commit attempt`), а не на `UPDATE`. Проверка на Serializable выполняется до окончательной фиксации.
6. Подсказка `HINT: The transaction might succeed if retried` означает: приложение должно **повторить транзакцию целиком**. Теперь снимок будет новый, B увидит сумму 100 и не даст снять 1200.

### Итог

| Уровень | Что с правилом «сумма ≥ 0» | Итоговая сумма |
|---|---|---|
| REPEATABLE READ | нарушено, обе транзакции прошли | −1100 |
| SERIALIZABLE | соблюдено, одна транзакция отменена | 100 |

---

## Ответы для защиты

- **Почему на REPEATABLE READ правило нарушилось?**
  Каждая транзакция проверила правило на своём снимке и была права. Менялись разные строки, поэтому PostgreSQL не видел конфликта.
- **Почему на SERIALIZABLE этого не случилось?**
  Кроме одного снимка, уровень отслеживает зависимости между транзакциями. Найдя цикл, он отменяет одну из них.
- **Что должно делать приложение при ошибке `could not serialize access`?**
  Повторить транзакцию с начала. При ошибке откатывается всё, что транзакция успела сделать.
