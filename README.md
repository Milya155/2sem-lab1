# 2sem-lab1
**Store** - это приложение для анализа продаж в интернет-магазине.
Принимает список заказов интернет-магазина и набор параметров фильтрации, а возвращает словарь со статистикой продаж: выручка, средний чек, медиана, топ товаров, распределения по категориям/городам/оплате и тд.

![ozon](https://i.pinimg.com/736x/e1/06/f8/e106f8e3eb97ad57102f2f2decb8b605.jpg)


## Оглавление
- [Подготовка данных для анализа](https://github.com/Milya155/2sem-lab1/edit/main/README.md)
- [Установка](#-установка)
- [Принять список заказов](https://github.com/Milya155/2sem-lab1/edit/main/README.md)
- [Обработка списка](https://github.com/Milya155/2sem-lab1/edit/main/README.md)
- [Сформировать отчёт](https://github.com/Milya155/2sem-lab1/edit/main/README.md)

---

## Возможности
- Расчёт общей выручки
- Расчёт среднего чека
- Расчёт медианы
- Количество возвратов
- топ N-товаров
- Распределение по категориям
- Распределение по городам
- Распределение по способам оплаты
- Доля возвратов

## Установка
1. Установите Python 3.10+
2. Настройте работу с базами данных. Дополнительные библиотеки не нужны - только стандартная
3. Установите зависимости

```bash
git clone https://github.com/user/store.git
cd store
pip install -r requirements.txt
```

>Важно: даты должны быть в формате "YYYY-MM-DD"(например,"2024-03-15")



## Использование
```
python analyze.py --start_date 2024-01-01 --end_date 2024-05-20
python analyze.py --category "Электроника" --top_n 5
```


|Команда|Описание|
|-------|--------|
|'orders'|Заказы|
|'start_date'|Начало периода|
|'end_date'|Конец периода|
|'category_filter'|Сортировать категории|
|'top_n'|Топ N-товаров|
|'include_return'|Количество возвратов|

## Пример заметки
Ниже представлен пример JSON-объекта, который описывает один заказ в интернет-магазине и подается на вход функции `analyze_sales`.

```json
{
  "id": 1,
  "date": "2024-05-20",
  "customer_id": 105,
  "category": "Электроника",
  "amount": 15000,
  "quantity": 1,
  "is_returned": false,
  "city": "Москва",
  "payment_method": "Карта"
}
``` 

## Архитектура
> Приложение разделено на независимые модули: загрузка данных, фильтрация и аналитика. Такое разделение позволяет легко добавлять новые метрики (например, прогнозирование) без переписывания всего основного кода.

```text
[ Входные данные (Список заказов) ]
               |
               v
[ Фильтрация (даты, категории, суммы) ]
               |
               v
[ Модуль аналитики (выручка, топы, медиана) ]
               |
               v
[ Выходной словарь (отчёт) ]
```

## Roadmap

- [x] Фильтрация по датам и категориям
- [x] Подсчёт метрик
- [x] Топ N-товаров
- [x] Распределение по категориям/городам/оплате
- [] Возвраты и их доля
- [] Процентили
- [] Экспорт отчёта JSON
- [] Визуализация графиков

## Код
```
import statistics
import collections
from datetime import datetime
from typing import List, Dict, Any, Optional



def analyze_sales(
    orders: List[Dict[str, Any]],
    start_date: Optional[str] = None,
    end_date: Optional[str] = None,
    category_filter: Optional[str] = None,
    min_amount: Optional[float] = None,
    top_n: int = 5,
    include_returns: bool = True,
    currency_scale: float = 1.0
) -> Dict[str, Any]:
    """
    Анализирует продажи интернет-магазина на основе списка заказов.

    :param orders: Список словарей с данными о заказах.
    :param start_date: Начальная дата периода в формате 'YYYY-MM-DD' (включительно).
    :param end_date: Конечная дата периода в формате 'YYYY-MM-DD' (включительно).
    :param category_filter: Фильтр по категории товара.
    :param min_amount: Минимальная сумма заказа.
    :param top_n: Количество топовых товаров для вывода.
    :param include_returns: Учитывать ли возвраты в выручке.
    :param currency_scale: Множитель для конвертации валюты.
    :return: Словарь со статистикой (не менее 12 ключей).
    """
    
    # Валидация входа. Пустой список — вернуть словарь с нулями/пустыми структурами.
    if not orders:
        return {
            "total_revenue": 0.0,
            "avg_check": 0.0,
            "median_check": 0.0,
            "returns_count": 0,
            "return_rate": 0.0,
            "top_n_products": {},
            "category_distribution": {},
            "city_distribution": {},
            "payment_distribution": {},
            "best_day": None,
            "worst_day": None,
            "percentiles": {}
        }

    # Не мутировать входные данные. Создаем новый список.
    filtered_orders = []
    
    # Подготовка дат для фильтрации
    start_dt = datetime.strptime(start_date, "%Y-%m-%d") if start_date else None
    end_dt = datetime.strptime(end_date, "%Y-%m-%d") if end_date else None

    for order in orders:
        # Обрабатываем None-поля там, где они возможны.
        o_date_str = order.get("date")
        if not o_date_str:
            continue
            
        o_date = datetime.strptime(o_date_str, "%Y-%m-%d")
        o_category = order.get("category") or "Без категории"
        o_amount = order.get("amount")
        if o_amount is None:
            o_amount = 0.0
            
        o_is_returned = order.get("is_returned", False)
        o_city = order.get("city") or "Неизвестно"
        o_payment = order.get("payment_method") or "Неизвестно"
        o_id = order.get("id") or "unknown"

        # Фильтрация по датам
        if start_dt and o_date < start_dt:
            continue
        if end_dt and o_date > end_dt:
            continue
        # Фильтрация по категории
        if category_filter and o_category != category_filter:
            continue
        # Фильтрация по минимальной сумме
        if min_amount is not None and o_amount < min_amount:
            continue
        # Фильтрация возвратов
        if not include_returns and o_is_returned:
            continue

        # Сохранение очищенных данных в новый список
        filtered_orders.append({
            "date": o_date_str,
            "category": o_category,
            "amount": o_amount * currency_scale,
            "is_returned": o_is_returned,
            "city": o_city,
            "payment_method": o_payment,
            "id": o_id
        })

    # Если после фильтрации ничего не осталось
    if not filtered_orders:
        return {
            "total_revenue": 0.0,
            "avg_check": 0.0,
            "median_check": 0.0,
            "returns_count": 0,
            "return_rate": 0.0,
            "top_n_products": {},
            "category_distribution": {},
            "city_distribution": {},
            "payment_distribution": {},
            "best_day": None,
            "worst_day": None,
            "percentiles": {}
        }

    # Сбор базовых метрик
    amounts = [o["amount"] for o in filtered_orders]
    returns = [o for o in filtered_orders if o["is_returned"]]
    
    total_revenue = sum(amounts)
    count_orders = len(filtered_orders)
    returns_count = len(returns)

    # Обработка деления на ноль во всех вычислениях.
    avg_check = total_revenue / count_orders if count_orders > 0 else 0.0
    median_check = statistics.median(amounts) if amounts else 0.0
    return_rate = returns_count / count_orders if count_orders > 0 else 0.0

    # Сортировка должна быть детерминированной.
    # Сортировка по убыванию количества, а при равенстве — по алфавиту (ключу).
    def get_sorted_counts(items):
        counter = collections.Counter(items)
        return dict(sorted(counter.items(), key=lambda x: (-x[1], x[0])))

    # Топ-N товаров 
    product_counter = collections.Counter(o["id"] for o in filtered_orders)
    top_n_products = dict(sorted(product_counter.items(), key=lambda x: (-x[1], x[0]))[:top_n])

    # Распределение
    category_distribution = get_sorted_counts(o["category"] for o in filtered_orders)
    city_distribution = get_sorted_counts(o["city"] for o in filtered_orders)
    payment_distribution = get_sorted_counts(o["payment_method"] for o in filtered_orders)

    # Лучший и худший день по выручке
    day_revenue = collections.defaultdict(float)
    for o in filtered_orders:
        day_revenue[o["date"]] += o["amount"]
    
    if day_revenue:
        # Детерминированная сортировка: по сумме (убывание), потом по дате (возрастание)
        sorted_days = sorted(day_revenue.items(), key=lambda x: (-x[1], x[0]))
        best_day = sorted_days[0][0]
        worst_day = sorted_days[-1][0]
    else:
        best_day = worst_day = None

    # Процентили (25%, 50%, 75%)
    percentiles = {}
    if len(amounts) >= 2:
        quartiles = statistics.quantiles(amounts, n=4)
        percentiles = {
            "25%": round(quartiles[0], 2),
            "50%": round(quartiles[1], 2),
            "75%": round(quartiles[2], 2)
        }
    elif amounts:
        percentiles = {"25%": round(amounts[0], 2), "50%": round(amounts[0], 2), "75%": round(amounts[0], 2)}

    # Округление числовых результатов до 2 знаков.
    # Возвращать не менее 12 ключей в итоговом словаре.
    result = {
        "total_revenue": round(total_revenue, 2),
        "avg_check": round(avg_check, 2),
        "median_check": round(median_check, 2),
        "returns_count": returns_count,
        "return_rate": round(return_rate, 2),
        "top_n_products": top_n_products,
        "category_distribution": category_distribution,
        "city_distribution": city_distribution,
        "payment_distribution": payment_distribution,
        "best_day": best_day,
        "worst_day": worst_day,
        "percentiles": percentiles
    }
```

## Лицензия

[Ссылка на интернет-магазин](https://www.ozon.ru/?__rr=1&abt_att=1&origin_referer=yandex.ru)  


Это учебный проект[^1].

---
[^1]: Выполнено в рамках курса по проектированию баз знаний.
