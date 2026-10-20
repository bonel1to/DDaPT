# Домашнее задание № 4. План запроса и индексы

База `logs`, коллекция `events` (1200 документов). Планы сняты на MongoDB 7.0.14 после загрузки исходного файла `stend/seed/logs.events.json` из [репозитория практикума](https://github.com/MaximBytecamp/mongodb-practice). Каждый составной индекс создавался отдельно и удалялся после замера; перед запросом 2 дополнительных индексов не было. Направление шагов в таблице — снизу вверх, от чтения к выдаче результата.

| План | Индекс | Порядок | `nReturned` | `totalDocsExamined` | `totalKeysExamined` | Шаги |
| --- | --- | --- | ---: | ---: | ---: | --- |
| `plan-1.json` | нет | — | 33 | 1200 | 0 | `COLLSCAN → SORT` |
| `plan-2.json` | `{service: 1, level: 1, ts: -1, duration_ms: 1}` | E–S–R | 33 | 33 | 43 | `IXSCAN → FETCH` |
| `plan-3a.json` | `{duration_ms: 1, service: 1, level: 1, ts: -1}` | R–E–S | 33 | 33 | 378 | `IXSCAN → SORT → FETCH` |
| `plan-3b.json` | `{service: 1, level: 1, duration_ms: 1, ts: -1}` | E–R–S | 33 | 33 | 33 | `IXSCAN → SORT → FETCH` |
| `plan-4-bez.json` | нет | — | 10 | 1200 | 0 | `COLLSCAN → SORT` |
| `plan-4.json` | `{level: 1, duration_ms: -1}` | E–S | 10 | 10 | 10 | `IXSCAN → FETCH → LIMIT` |

## Задача 1. Запрос без индекса

```javascript
db.events.find({
  service: "payments",
  level: "error",
  duration_ms: { $gt: 500 }
}).sort({ ts: -1 })
```

Фильтр находит 33 записи. [Разбор плана 1](https://dfrancour.dev/tools/mongodb-paste-the-plan#v1:zVVZb9swDP4vQh_Twm6ao3lr0mYrlixd3W0YimFQJTpVY0ueJOdYkP8-Ujm7nuiAYU9xSH48PpLinMG0yLjSX8A6ZTRrsZhV2M8S7Owi41qDZa050zwHV3ABqM_M0B3AGLR3aKm0hGlXZR5sAp61Up45qLCCWwfyE7kh_B7XkrWu5yxDYBYk8BN9gbXGssWiMmcO7FhRgLWu4LM8BAlqWVruMcEfuQsmQ4xVi6LF4vtile577m4Rdto8rJ90203MDQvTHS5u4QPM7mtyPj2nxEEObGKykjy7SyBbualha3Si5XNWieDaXZkzZNJI-NNgorRWekhsUubO8yHRmAwurzATZ6y_4B7pC1ost7UfY0k55D2VK6wyjo6atUY9iirMzwqCOpUXGQTyi9InS4dbz51Br5d0Tj6iQRoa8w86IJUF4ZcDlBo74VYSilm4QzlIqh5h12SLjIrAJWbug6-tpBQCHMq8LZE6fQm-tJrIrFZ3gFcqh77KMoWWRIvxPMMeu7Mpz1UwX0tPjdiRxofE4m78IbhHmvJy3DPnVc49hEgTY0eOvFdrFcblmGuxhmoASagQO17-_6YgW6bo-BiIBFITV84buyPAMINu-PqrKVkSwT1P1C9I0BMldxw30W-JS3qq3GgzrK7A8pashk9UExDzQoIIH1T_z9y9tVPUiyc7Fdcbb-7UY4uA0gdjuKDtECbPAz9zZIx-2eZd3TK4oWfLS2XN5orGytMkLWfn3tDsyZvVM85Ije7BnuvUkNGtcWjM-ly0jRntX1izn4Adwt1BZgTP6FEN7uo47FjteHM1GgfRQXyE-qHy21sioHYsUlHnVVGDmojqDSl4JKtR_bgBUtab8iitxc3mNpELbvHWYOmhDqVp5HkWDkkX749vl2mKpwYnsT3ztLw7g__Qus-ng9LjsOI78Dymh-WWRZjpcxLlIBV2F3ElcY6OnsevDRNTWgHvrCkLxPQhN3b2QqZo10Z2R3gkaDuXmM8OM3kBiN25VTfKr9F96tRA940emmS1qPfjnEh5ZfBOv7YONP2Kg2kmXdoE99qKutRD2rWO0d4aGlRvZ8kN3kbnraKDQB03-O7Ei98). Сервер прочитал все 1200 документов, поскольку для условий запроса не было подходящего индекса: `COLLSCAN` проверил каждый документ, затем `SORT` отсортировал найденные 33 записи в памяти.

## Задача 2. Индекс по правилу ESR

- **E (Equality):** `service = "payments"`, `level = "error"`.
- **S (Sort):** `ts: -1`.
- **R (Range):** `duration_ms > 500`.

```javascript
db.events.createIndex({ service: 1, level: 1, ts: -1, duration_ms: 1 })
```

[Разбор плана 2](https://dfrancour.dev/tools/mongodb-paste-the-plan#v1:7VZLc9s2EP4rHkwOOcga6mlZN8uWGk-ryAmVtB3Ho4GBpQSLBBgAlKVq9N-7C-rptnaO7UxPIPeFb3c_YLFmsMxTrvRXsE4ZzbqsxirsewF2dZdyrcGy7pppnoHLuQDUp2bqqrAA7R1aKi1hOVCpBxuDZ92Epw4qLOfWgfxEYcj_HdeSde_XLEXHNEjgO8YCa41lm01lzRzYhaINdrqcr7KwSVDLwnKPACeZCyZT3KsVRZvNw2YL9wN3M3S76dTbV4NeB7FhYvqaixn8DIiCtfqterN5WUdNxpe3BBzkyMYmLSiy-wxkK_c5HIyutHzNKhZcu7HpYyWNhJcGz0prpadUTULuPJ9SGQf98fWHUMC88HEpPGhvf4uvrz6ieg6rO-6xuqXzrki1yq6U-IU16p7jelKj2mbbnI_YO4y4dZ3UJsERV-8m57XJkdOEWq_csEi9CiXbpbgVIJCZO4Fx_7DHQZ8EhNYTIPfUIeW-aIVt2sdULg4cORLccesVTw8SQr_nZR3DKgvClyxNjH3mVu4Y2DOFli-wsftvexJ9Y5Wz478HdoScDAMTS6vtZzAJGbH7IV9i_pWzodK4Bs1pjuw9krFypnRSjVC9Qc4yC08IFyR1flcHZJMIPMKO-4D3ICmEAIcybwvMXn8GX1hNRGo0jhzHKoOhSlOFlhECNJ6niMn1lzxTwbzZ2IpvjDgSn0QJhHN_x8e39-07rzLuIez_bOzclXtyueBa7Bw1gCQf1r0sf35XkMrg4_gCKP8yggXnjT0S4A6jQSC2_EsCPLXAJZ310eNTaf36-fk3pfP_Wf5PnuXQuBfnywEQT2oROedujE2Hkg30e2NNnof_cBEIk2VhAK5Zomhl-_GZhMl5kuth8O2z2s7Jyj9PQURkrCdZSSAUvJOP22nNSI3hwd7qxJDRzDg0ZkMuesbMz--sOY_BTuGpmhqBbcPZGcK1a_UGknGxfxxcVKNqrYn6qfKHJ4OA1qVIRJs3RAtaImpfSMEj2YjalxcgZbsjm0mr1ukcgCA_kMqYeshDaToUPA3vhQE-M3yvSBJ8Uag_oLfyECrd7LQu2lE48S-tsaWjwuM9gDfe6z6_YLpFHq6LWxJlIBUeVPQrqOYY6HX_nWFsCivgJ2uKHH2GkBm7egMp2vWwunN8C8RY3NLni0Mkbzhid2bqUfmd95A6NdJDo6cm3t6Bp_tcSTk2-Bz70TzQ9Fckpnke0KXmfjSjAfWQbsxro701RFRvV_EjPoGct4pmH3XczOkG-xM). План: `IXSCAN → FETCH`, возвращено 33, просмотрено 33 документа и 43 ключа индекса. Чтение документов уменьшилось с 1200 до 33, то есть примерно в **36,4 раза**. `SORT` исчез, потому что после двух полей равенства записи индекса уже идут в нужном порядке по `ts`. Из 43 просмотренных ключей условиям по длительности подошли 33; остальные ключи отсеяны без чтения соответствующих документов.

## Задача 3. Влияние порядка полей

Первые две строки сводной таблицы показывают отсутствие индекса и порядок ESR. Следующие две строки — отдельные замеры с переставленными полями.

- **Вариант А, R–E–S:** первым идёт `duration_ms`. Условие `> 500` охватывает 378 записей индекса из всех сервисов и уровней. Проверка `service` и `level` оставляет 33 документа, но сервер уже просмотрел 378 ключей. Порядок по `ts` при таком обходе не готов, поэтому есть `SORT`.
- **Вариант Б, E–R–S:** по двум равенствам и диапазону сервер просматривает только 33 ключа, но `ts` стоит после диапазона. Записи с разными `duration_ms` не образуют общего порядка по времени, поэтому `SORT` остаётся в памяти. Малое число ключей само по себе не означает лучший план.

В рабочей базе для первого запроса оставил бы индекс **E–S–R** из задачи 2: он читает только нужные 33 документа и обходится без отдельной сортировки. Вариант Б экономит 10 просмотров ключей, но выполняет `SORT`.

## Задача 4. Десять самых долгих ошибок

```javascript
db.events.find({ level: "error" })
  .sort({ duration_ms: -1 })
  .limit(10)
```

Без составного индекса план `COLLSCAN → SORT` читает 1200 документов и возвращает 10. Здесь **E** — `level`, **S** — `duration_ms`, условия диапазона нет. Создан индекс:

```javascript
db.events.createIndex({ level: 1, duration_ms: -1 })
```

[Разбор плана 4](https://dfrancour.dev/tools/mongodb-paste-the-plan#v1:7Vbbcts2EP2VDCaPsoe0rYv9JslWo2kYOaGStON4PDCwlGCRAAOAstSM_r27oG6RWzu9PPShTyQXZ3F29yyw_MZgUeZc6U9gnTKaXbCYNdjXCuzyOudag2UX35jmBbiSC8D13EzcMcxBe4dIpSUsBir3YFPw7CLjuYMGK7l1IN_TNuSfIz6nl9fwFbcAa41lq9Wa6A13U7SedM87vahzibtiSLrPxRR-BvRng8v4pB-1aaXgiyFRghzZ1OSVx6DdByCs3LLvQF0tn0Olgms3NldYAyPhEPCotFZ6QnWg2J3nEyrA22EyHGMouSqU7xam0ph3HFEtysqnNWoHH1yN-2_Yn60Of0n73Xe4PIPlNfdYRr1XsLjBZGU5hX9XOHZxFK_WJX-HipAYhLuL7_ZQd0ekoHJJlXsV6rfJd21AmqnbI7m5PWC5uSUW91ErlGfrrlwaVN0zXHPrFc93Fops20knuK2yIHzdV5mxj9zKTc_0sG7yuzDYzZe6M76wxqvt6y17Eh67SfgCE2m8SpTGJ0JWK-omCw9IB5IU2-SBXSCC_lh6H_h2lkoIcGjztsLo9QfwldXUAKTmFjZWBSQqzxUi0e6N5zmyuqsFL9QWHsyXRhyY9-kn4P6oj17mvXJeFdxD4H80dob7xNgbXM65FhtHDSDJJ6Do41cFuQxfjs-B8q_XLDhv7J4BGUaD0G1_saf_ZujRvx46vskntee5BS7pehndP9To58_gfymd_--Df3QfhAIe9IMDCHqRf-nGWHuoRaHPS2vKMnzXd4kwRcG1pIAyRU-2nXlZGHd7oW4GGlIY62nhqUrhZNVxvJb36zkaXMDOwQ51ZshxahyiWMJFz5jZ0bU1RynYCTwc50ZgaXE2BopWfHKKicy3Y7t9HB3HZ7g-UX43zAU0z0UmWvxUNKEpolZbCh7J06h13gYpWx15ljXjTmcXCGqIrYT5BS2Upg7keZjkA_wB8L0qy3DWq9-gt_QQ2v-s02y3onC8DtEozKjyeOjwZnze5y2mW5XhbA7JVIBUeCrQryqw7rjR8_4bYGoqK-Ana6oSfRIojF2-ECnieljdGc76FItb-3x0GMkLjqjOVN0rv_FOSKmRToyemHR94XzP05VybPBH6UfzQOhn7D7zOKAbxP1oRgPSkK6nvtHeGmpRb5fpPf7iOG8VzUhS3Mxwi9Xv). После индекса план `IXSCAN → FETCH → LIMIT` просматривает 10 ключей и читает 10 документов. В коллекции 180 ошибок, но они уже упорядочены в индексе по убыванию длительности: сервер берёт первые 10 записей и останавливается на `LIMIT`.

## Ответы на вопросы при приёмке

- `totalKeysExamined` считает просмотренные записи индекса, а `totalDocsExamined` - прочитанные документы коллекции. Ключ может быть просмотрен без чтения документа.
- `FETCH` нужен, чтобы получить полный документ по ссылке из индекса. Эти запросы возвращают документы целиком; одних индексных ключей недостаточно.
- Индекс задачи 2 просмотрел 43 ключа, хотя вернул 33 документа: после условий равенства он идёт по `ts`, а проверка диапазона `duration_ms` отсекает 10 ключей.
- При вставке нового события MongoDB обновит и сам документ, и каждый относящийся к коллекции индекс. Индексы ускоряют чтение, но требуют места и работы при записи.
- Индекс из всех полей увеличивает размер и стоимость записи, а его порядок подходит не каждому запросу. Поля выбирают по реальным фильтрам и сортировкам.
