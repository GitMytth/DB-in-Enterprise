# Лабораторная работа 2
### Выполнил студент группы 6133-010402D Калашников Дмитрий
Цель данной лабораторной работы: построить модель классификации, которая будет определять подозрительных клиентов.
Сначала сделаем выгрузку информации о клиентах из базы данных. Для этого был написан следующий SQL запрос:
```sql
  with card_freq as (select user_id, 
  count(*) filter (where payment_type = 'card') as card_usage
  from public."User_actions"
  group by user_id
  order by user_id),
  
  cash_freq as (
  select user_id, 
  count(*) filter (where payment_type = 'cash') as cash_usage
  from public."User_actions"
  group by user_id
  order by user_id
  ),
  
  payment_type_stats as (
  select t1.user_id, t1.card_usage, t2.cash_usage
  from card_freq t1
  join 
  cash_freq t2
  on t1.user_id = t2.user_id
  order by t1.user_id
  ),
  
  count_actions as (
  select user_id,
  count(*) filter (where action = 'cancel_order') as count_cancelations,
  count(*) filter (where action = 'create_order') as count_order_creations
  from public."User_actions"
  group by user_id
  order by user_id
  ),
  
  first_last_order_date as (
  select t1.user_id,
  min(t2.creation_date) as first_order_date,
  max(t2.creation_date) as last_order_date
  from (select * 
    from public."User_actions"
    where action = 'create_order') t1
  join 
  public."Orders" t2
  on t1.order_id = t2.order_id
  group by t1.user_id
  order by t1.user_id
  )
  
  select t1.user_id, t1.fname, t1.sname, t1.gender,
  t1.age, t1.address, t1.phone, t1.email,
  t1.is_suspicious, t2.card_usage, t2.cash_usage,
  t3.count_cancelations, t3.count_order_creations,
  t4.first_order_date, t4.last_order_date
  from public."Users" t1
  left join payment_type_stats t2
  on t1.user_id = t2.user_id
  left join count_actions t3
  on t1.user_id = t3.user_id
  left join first_last_order_date t4
  on t1.user_id = t4.user_id
  order by t1.user_id

```

В данном запросе мы объединяем данные о клиентах с некоторыми полезными статистиками такими как:
- количество оплат картой и наличными;
- количество совершенных заказов и отмененных заказов;
- дата первого и последнего заказов.

Выбранный классификатор - RandomForestClassifier.
После загрузки данных добавим в датасет еще признаки - количество дней между первым и последним заказами, количество цифр в email клиента, а также закодируем категориальную переменную gender.
```py
def count_digits(x):
    x = list(x)
    counter = 0
    for ch in x:
        if ch.isdigit():
            counter += 1
    return counter

df["number_digits_in_email"] = df["email"].apply(lambda w: count_digits(w))

df["first_order_date"] = pd.to_datetime(df[df["first_order_date"] != 0]["first_order_date"], format='%Y-%m-%d')
df["last_order_date"] = pd.to_datetime(df[df["last_order_date"] != 0]["last_order_date"], format='%Y-%m-%d')
df["days_between_first_and_last_orders"] = (df["last_order_date"] - df["first_order_date"]).dt.days

df["DecodeGender"] = df["gender"].map({"M": 1, "F": 0})
```

Выделим целевую переменную и уберем ненужные признаки.


```py
data = df.drop(["user_id", "fname", "sname", "address", "phone", "email", "is_suspicious", "gender", "first_order_date", "last_order_date"], axis=1)
target = df.is_suspicious
```

Далее делим данные на обучающие и тестовые
```py
X_train, X_test, y_train, y_test = train_test_split(data, target, test_size = 0.4, random_state = 42)
```

Создаем baseline-модель, чтобы было с чем сравнивать
```py
baseline = RandomForestClassifier()
baseline.fit(X_train, y_train)
```

Получаем следующий `classification report`:

||precision|recall|f1-score|support|
|-|---------|------|--------|-------|
|False|0.76|0.85|0.81|661|
|True|0.92|0.87|0.90|1339|
|accuracy|||0.86|2000|

Метрики качества показывают, что модель нередко определяет клиента, не являющегося подозрительным, за подозрительного. Возможно в тренировочной выборке существует дизбаланс классов, и из-за этого модель ошибается в классификации клиентов, не являющихся подозрительными.
Попробуем улучшить метрики путем подбора гиперпараметров для модели классификации с помощью RandomSearch.

```py
clf = RandomizedSearchCV(baseline, grid, random_state=42)
search = clf.fit(X_new, target)
```

Получаем следующие значения
- n_estimators - 80
- max_features - 12
- min_samples_leaf - 7

Протестируем модель с полученными гиперпараметрами.

||precision|recall|f1-score|support|
|-|---------|------|--------|-------|
|False|0.80|0.92|0.85|661|
|True|0.95|0.89|0.92|1339|
|accuracy|||0.9|2000|

Метрики немного улучшились: accuracy подросла на 0,04, но нас больше интересует метрика f1 - она выросла также на 0,04 для класса 0 и на 0,02 для класса 1.
