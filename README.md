# System Design социальной сети

## Функциональные требования
#### Заметкы для себя - post, photo, reaction, comment, user, subscription, feed, tag

- Публикация постов из путешествий с фотографиями, небольшим описанием и привязкой к конкретному месту путешествия;
- Оценка и комментарии постов других путешественников;
- Подписка на других путешественников, чтобы следить за их активностью;
- Поиск популярных мест для путешествий и просмотр постов с этих мест;
- Просмотр ленты других путешественников и ленты пользователя, основанной на подписках в обратном хронологическом порядке

## Нефункциональные требования
- DAU 10 000 000
- Пользоватли только из СНГ
- Доступно через браузер, мобилки
- Поведение пользователей - 
  создание публикаций в день (1+1(фотки в s3) раз), 
  оценка(1 раз), 
  коментарии (2 раз), 
  подписка(0.5 раз), 
  поиск популярных мест(фид по тегу - 1 раз),
  просмотр ленты (30 раз), 
  просмотр популярных тегов (0.5 раз, 50 штук), 
  просмотр коментариев публикации (10 раз, умножить на 10 штук)
- Сезонности в приложении - наверное в летний сезон будет больше постов
- Условия хранения данных - Храним всегда (пока пользователь не удалился)
- Лимиты и ограничения - 
  среднее количство фоток в одном посте (3), 
  средный размер 1 фотки (0.5 МБ), 
  средний размер тега (50 символов), 
  средний размер описания (700 символов), 
  средний размер комента (500 символов)
  максимальное количество подписчиков - 1 000 000, 
- Временные ограничения - 
  создание поста (1-2 сек), 
  оценка и комент (1 сек), 
  подписка (1 сек), 
  просмотр ленты, просмотр по тегу (300-500 милисекунд), 
  время попадания в кеш (200 секунд)
- Доступность приложения - Не более, чем 9 часов простоя в год, SLA(~99,90%)


## Нагрузки

### Публикации

```
** RPS(Write) = 10 000 000 * 1 / 86400 = ~120 req/sec
   RPS(Write S3) = 120 req/sec

String unicode
User Token = ~ 200 B (Мне кажется что должен учитывать данные которые 
отправляются через запрос, а не колонки в бд, поэтому во всех запросах 
добавляю примерно 200 байт на токен юзера)
Такж в трафик не добавил респонсы хотя по логике должен

Photo to s3 = 3 x 0.5MB = 1 500 000 B
** Traffic = 1 500 000 B * 120 req/sec = ~0.18 GB/sec

Post (
   tag - string (50) ~ 100 B
   description - string (700) ~ 1400 B
   photo links ~ 1000B
) ~ 2700 B 

** Traffic(Write) = 2700 * 120 = ~ 0.000324 GB/sec
```


### Фиды

Фиды по пользувателю

```
** RPS(Read) = 10 000 000 * 30 / 86400 = ~ 3473 req/sec

Request user token (200B)

Response (
    uid
    description
    created_at
    Photos (x3)
    tag
    reaction (int)
) ~ 1300B x20  = 26000 B

** Traffic(Read) = 26000 * 3473 = ~0.1 GB/sec

```

После получения респонса, браузер делает еще 3x20=60 запросов по картинкам на s3

``` 
** RPS(Read photos) = 10 000 000 * 60 / 86400 = ~6945 req/sec
** Traffic(Read photos) = 500000 * 6945 = ~ 3.48 GB/sec
```

#### Оценка дисков(Публикации(сохранение) + фиды(полуечние))

```
RPS(Write s3) = 120 req/sec
RPS(Read s3) = ~6945 req/sec

Traffic(Write s3) = ~0.18 GB/sec
Traffic(Read s3) =  ~3.48 GB/sec

Capacity(s3) = 0.18 GB/s * 86400 * 365 = 5677 TB

HDD:  
  Disks_for_capacity = 5677 TB / 32TB = 178 Disk
  Disks_for_throughput = 3.66 GB/sec / 100 МБ/с = 36,6 = 40 Disk
  Disks_for_iops = iops / disk_iops = 7065 / 100 =  71 Disk
  Disks = 178
  
SSD(SATA):  
  Disks_for_capacity = 5677 TB / 100TB = 57 Disk
  Disks_for_throughput = 3.66 GB/sec / 500 МБ/с = 8 Disk
  Disks_for_iops = iops / disk_iops = 7065 / 1000 =  8 Disk
  Disks = 57
  
SSD(nVME):  
  Disks_for_capacity = 5677 TB / 30TB = Disk
  Disks_for_throughput = 3.66 GB/sec / 3GB/с = 2 Disk
  Disks_for_iops = iops / disk_iops = 7065 / 10000 =  1 Disk
  Disks = 190

Очевидно выбор SSD(SATA) = 57 x 100 TB, 
хотя есть сомнения и по nVME (из за пропусконой способности и iops)

====================================================================

RPS(Write) - 120 req/sec
RPS(Read) - 3473 req/sec

Traffic(Write) - 0.000324 GB/sec
Traffic(Read) -  0.1 GB/sec

Capacity = 0.000324 GB/sec * 86400 * 365 = 11 TB

HDD:  
  Disks_for_capacity = 11 TB / 10TB = 2 Disk
  Disks_for_throughput = 0.100324 GB/sec / 100 МБ/с = 101/100 = 2 Disk
  Disks_for_iops = iops / disk_iops = 3693 / 100 =  37 Disk
  Disks = 37
  
SSD(SATA):  
  Disks_for_capacity = 11 TB / 20TB = 1 Disk
  Disks_for_throughput = 0.100324 GB/sec / 500 МБ/с = 1 Disk
  Disks_for_iops = iops / disk_iops = 3693 / 1000 =  4 Disk
  Disks = 4
  
SSD(nVME):  
  Disks_for_capacity = 11 TB / 10TB = 2 Disk
  Disks_for_throughput = 0.100324 GB GB/sec / 3GB/с = 1 Disk
  Disks_for_iops = iops / disk_iops = 3693 / 10000 =  1 Disk
  Disks = 1
  
Выбор SSD(nVME) - 2 x 12TB (С запасом, тут желательно получить обратный комнтарий - 
правильно думаю насчет 2х12 тб вместо 1х15, по цене переплта есть но думаю есть смысл
 не использовать всего 1 диск)

```


### Оценки

```
** RPS(Write) = 10 000 000 * 1 / 86400 = 120 req/sec

User Token + post_uid + reaction (int) = 200 + 16 + 4 = 220 B

** Traffic(Write) = 220 B * 120 req/sec = 27 KB/sec
```

#### Оценка дисков(Оценки)

```
Read добавился позже. буду брать примерно те жe данные 
что в комeнтариях(с учетом Write данных)

RPS(Write) = 120 req/sec
RPS(Read) = 1200 req/sec

Traffic(Write) = 27 KB/sec
Traffic(Read) =  0,0013 GB/sec

Capacity = 27 KB/s * 86400 * 365 = ~1 TB

HDD:  
  Disks_for_capacity = 1 TB / 2TB = 1 Disk
  Disks_for_throughput = 1.4 MB/sec / 100 МБ/с = 1 Disk
  Disks_for_iops = iops / disk_iops = 1320 / 100 =  14 Disk
  Disks = 15
  
SSD(SATA):  
  Disks_for_capacity = 1 TB / 2TB = 1 Disk
  Disks_for_throughput = 1.4 MB/sec / 500 МБ/с = 1 Disk
  Disks_for_iops = iops / disk_iops = 1320 / 1000 =  2 Disk
  Disks = 2
  
SSD(nVME):  
  Думаю нту смысла расчитать - SSD(SATA) устраивает

Выберу SSD(SATA) 2x2TB

```


### Коментарии
```
** RPS(Write) = 10 000 000 * 2 / 86400 = 240 req/sec

User Token + post_uid + comment (string 500) = 200 + 16 + 1000 = 1216 B

** Traffic(Write) = 1216 * 240 = 2335 KB/sec
```

Коментарии по 1 публикации, 10 штук за 1 запрос

```
** RPS(Read) = 10 000 000 * 10 / 86400 = ~ 1200 req/sec

User Token + 10 * comment(1000 B) = 200 + 10000 = 10200 B

** Traffic(Read) = 10200 B * 1200 req/sec = ~ 0,013 GB/sec
```

#### Оценка дисков(Коментарии)

```
RPS(Write) = 240 req/sec
RPS(Read) = 1200 req/sec

Traffic(Write) = 2335 KB/sec
Traffic(Read) =  0,013 GB/sec

Capacity = 2335 KB/s * 86400 * 365 = 74 TB

HDD:  
  Disks_for_capacity = 74 TB / 30TB = 3 Disk
  Disks_for_throughput = 15.3 MB/sec / 100 МБ/с = 1 Disk
  Disks_for_iops = iops / disk_iops = 1440 / 100 =  15 Disk
  Disks = 15
  
SSD(SATA):  
  Disks_for_capacity = 74 TB / 100TB = 1 Disk
  Disks_for_throughput = 15.3 MB/sec / 500 МБ/с = 1 Disk
  Disks_for_iops = iops / disk_iops = 1440 / 1000 =  2 Disk
  Disks = 2
  
SSD(nVME):  
  Думаю нту смысла расчитать - SSD(SATA) устраивает

Выберу SSD(SATA) 2x60TB

```


### Подписки
```
** RPS(Write) = 10 000 000 * 0.5 / 86400 = 60 req/sec

User Token + follow_uid  = 200 + 16 = 216 B

** Traffic(Write) = 216 B * 60 req/sec = 13 KB/sec
```

#### Оценка дисков(Подписки)

```
RPS(Write) = 60 req/sec
Traffic(Write) = 13 KB/sec

Capacity = 13 KB/s * 86400 * 365 = 0.5 TB

HDD:  
  Disks_for_capacity = 0.5 TB / 1TB = 1 Disk
  Disks_for_throughput = 13 KB/sec / 100 МБ/с = 0.013 / 100 = 1 Disk
  Disks_for_iops = iops / disk_iops = 60 / 100 =  1 Disk
  Disks = 1
  
SSD: не считаю так как ужe понятно что HDD вполне устраивает
  
Выберу HDD 2x1TB

```


### Теги (популярные места)

Я думаю создание/обновление происходит во время создания публикаций. тоесть по логике нужно рпс публикаций дублировать
```
** RPS(Write) = 10 000 000 * 1 / 86400 = ~120 req/sec

User Token + post_uid + tag = 200 + 16 + 100 = 316 B

** Traffic(Write) = 316 B * 120 req/sec = 304 KB/sec
```

```
** RPS(Read) = 10 000 000 * 0.5 / 86400 = ~ 60 req/sec

User Token + 50 * tag(100 B) = 200 + 5000 = 5200 B

** Traffic(Read) = 5200 B * 60 req/sec = 2496 KB/sec
```


#### Оценка дисков(Теги)

```
RPS(Write) = 120 req/sec
RPS(Read) = 60 req/sec

Traffic(Write) = 304 KB/sec
Traffic(Read) =  2496 KB/sec

Capacity = 304 KB/s * 86400 * 365 = 9.6 TB

HDD:  
  Disks_for_capacity = 9.6 TB / 10TB = 3 Disk
  Disks_for_throughput = 2800 KB/sec / 100 МБ/с = 2.8 / 100 = 1 Disk
  Disks_for_iops = iops / disk_iops = 180 / 100 =  2 Disk
  Disks = 2
  
SSD(SATA):  
  Disks_for_capacity = 9.6 TB / 10TB = 1 Disk
  Disks_for_throughput = 2.8 MB/sec / 500 МБ/с = 1 Disk
  Disks_for_iops = iops / disk_iops = 180 / 1000 =  1 Disk
  Disks = 1
  
SSD(nVME):  
  Думаю нту смысла расчитать

Выберу HDD 2x10TB

```


---

Можно не смотреть - заметки для себя

## Оценка подсистем хранения и нагрузки

| Подсистема | Тип дисков | Кол-во дисков | Capacity (TB) | RPS (Read / Write) | Traffic (GB/sec) | Описние                                                  |
|-------------|-------------|----------------|----------------|---------------------|------------------|----------------------------------------------------------|
| **S3 (Фото)** | SSD (SATA) | 57 | **5677 TB** | 6945 / 120 | 3.48 / 0.18 | Основное хранилище изображений, высокая нагрузка на чтение |
| **Посты (мета)** | NVMe SSD | 2 | **11 TB** | 3473 / 120 | 0.1 / 0.0003 | Основная таблица `posts` + `feed`; критично по latency   |
| **Комментарии** | SSD (SATA) | 2 | **74 TB** | 1200 / 240 | 0.013 / 0.002 | Высокая частота чтения, можно кэшировать в Redis         |
| **Оценки (ratings)** | SSD (SATA) | 2 | **1 TB** | 1200 / 120 | 0.0013 / 0.000027 | Малый объём, часто обновляется      |
| **Подписки (follows)** | HDD | 2 | **0.5 TB** | 60 / 60 | 0.000013 | Низкая нагрузка, cold storage подходит                   |
| **Теги (tags)** | HDD | 2 | **9.6 TB** | 60 / 120 | 0.0028 / 0.0003 | Редкое обновление, можно кэшировать топ-результаты       |

---

## Итоговая сводка

| Показатель                   | Значение                                                                                      |
|------------------------------|-----------------------------------------------------------------------------------------------|
| **Суммарный объём хранения** | ~ **5773 TB**                                                                                 |
| **Общее количество дисков**  | ~ **65** (в основном под S3)                                                                  |
| **Основное узкое место**     | Чтение изображений из S3: **3.48 GB/sec**                                                     |
| **Основная горячая зона**    | Feed (чтение постов + изображений)                                                            |
| **Тёплое хранилище**         | Комментарии, посты                                                                            |
| **Холодное хранилище**       | Подписки, теги, оценки                                                                        |
| **Стек хранения**            | **S3 + CDN** (для фото), **PostgreSQL NVMe** (для posts/comments), **HDD** (для tags/follows) |

---

## Вывод

- 90% общего трафика и емкости — **S3**.
- Критически важна **скорость отдачи фото (CDN)**.
- Остальные таблицы можно хранить в **PostgreSQL + Redis cache**.
- Основная оптимизация — **разделение горячего (NVMe) и холодного (HDD/S3) слоёв данных**.
