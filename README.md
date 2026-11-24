# Real-Time Smart Grid Stream Processing & CEP with OpenHAB 4.x

Система моніторингу та керування енергомережею (Smart Grid) у реальному часі на базі **OpenHAB 4.x Rule Engine** та **RRD4J Persistence**.

## 1. Архітектура Smart Grid та потік даних

Потік телеметрії надходить від **6 розподілених вузлів**, згрупованих у 3 зони за критичністю:
* **Зона A (Priority 1 — Critical):** Вузол 1 (Лікарня), Вузол 2 (ЦОД). Не підлягають автоматичному відключенню.
* **Зона B (Priority 2 — Moderate):** Вузол 3 (Житловий сектор), Вузол 4 (Офіси). Відключаються лише при каскадних аваріях (`EMERGENCY_SHUTDOWN`).
* **Зона C (Priority 3 — Sheddable):** Вузол 5 (Зарядні станції EV), Вузол 6 (Промзона). Використовуються для балансування пікових навантажень (`SHEDDING_L1`).

### Концепції Stream Processing у реалізації:
1. **Sliding Window Aggregation:** Використання `averageSince(now - 10s)` у RRD4J для згладжування короткочасних імпульсних шумів частоти та навантаження.
2. **Temporal Correlation:** Зіставлення падіння частоти ($\Delta f$) та просідання напруги ($\Delta U$) в одному часовому вікні для ідентифікації загрози каскадного відключення.
3. **Spatial Correlation:** Крос-зональний аналіз напруги ($Z_1 \cap Z_2 \cap Z_3$) для виявлення переходу в острівний режим (*Islanding*).

## 2. Критерії стабільності мережі (Stability Criteria)

| Подія / Аномалія | Умова детекції (CEP Pattern) | Реакція системи |
| :--- | :--- | :--- |
| **Peak Warning** | $\bar{P}_{10s} > 0.90 \cdot P_{gen}$ | Попередження оператора (`PEAK_WARNING`) |
| **Overload (DR)** | $\bar{P}_{10s} > P_{gen}$ | Поетапне відключення Priority 3 (Вузол 6 $\rightarrow$ Вузол 5) |
| **Voltage Sag / Swell** | $U < 207\text{ В}$ або $U > 242\text{ В}$ ($\pm 5\%$) | Генерація тривоги якості електроенергії |
| **Harmonic Distortion** | $\text{THD} > 5.0\%$ (стандарт IEEE 519) | Фіксація гармонійних спотворень |
| **Cascade Failure Risk** | $\bar{f}_{10s} < 49.4\text{ Гц} \land (U_{Z2} < 207\text{ В} \lor U_{Z3} < 207\text{ В})$ | Перехід у `EMERGENCY_SHUTDOWN`, скидання P3 та P2 |
| **Islanding Detection** | $(U_{Z1}, U_{Z2}, U_{Z3} > 242\text{ В}) \land |f - 50.0| > 0.5\text{ Гц}$ | Перехід у `ISLANDED`, ізоляція промзони |

## 3. Алгоритм відновлення (Recovery Sequencing)

Після усунення аварійного тригера та стабілізації частоти в межах $49.85 - 50.15\text{ Гц}$ система переходить у стан `RECOVERY`. Щоб уникнути повторного просідання через пускові струми (*Cold Load Pickup*), підключення відбувається каскадно з інтервалом **4 секунди**:
`Node 3 (P2)` $\xrightarrow{+4s}$ `Node 4 (P2)` $\xrightarrow{+4s}$ `Node 5 (P3)` $\xrightarrow{+4s}$ `Node 6 (P3)` $\rightarrow$ `NORMAL`.

## 4. Запуск та тестування

```bash
docker-compose up -d
Відкрийте веб-інтерфейс: http://localhost:8080 $\rightarrow$ Basic UI $\rightarrow$ Smart Grid Real-Time CEP Analytics.
