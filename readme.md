# Постановка задачи #
Необходимо разработать веб-приложение простой платежной системы.

Требования:

1. В системe клиент соотносится с "кошельком", содержащим денежные средства в некой валюте. Сохраняется информация о имени клиента, стране и городе его регистрации, валюте кошелька и остатке средств на нем. 
2. Клиенты могут делать друг другу денежные переводы в валюте получателя или валюте отправителя.
3. Сохраняется информация о всех операциях на кошельке клиента. 
4. В системе существует информация о курсах валют (для всех зарегистрированных кошельков) к USD на каждый день.
5. Проект представляет из себя 2 основных части - HTTP API и  страница с отчетом. 
6. HTTP API должен представлять следующие интерфейсы: 

+ регистрация клиента с указанием его имени, страны, города регистрации, валюты создаваемого кошелька. 
+ зачисление денежных средств на кошелек клиента 
+ перевод денежных средств с одного кошелька на другой. 
+ загрузка котировки валюты к USD на дату

Отчет должен отображать историю всех операций по кошельку указанного клиента за период. 

+ Параметры: Имя клиента (обязательный параметр), Начало периода (необязательный параметр), конец периода (необязательный параметр). 
+ Необходимо также вывести общую сумму операций по счету за период в USD и валюте счета
+ Должна быть предусмотрена возможность скачивания результатов отчета в файл (например, в CSV или XML формате). 

Примечания: 

1. Авторизация/аутентификация на любой из частей сайта не обязательна. Все данные о пользователях, там, где это нужно, могут приходить в теле запроса одним из параметров. 
2. При решении должна использоваться реляционная СУБД.

# Реализация #
Для реализации задачи была использована следующие средства: БД PostgreSQL 9.3, PHP Framework Laravel 5.3, PHP 7, Apache-PHP-7, BootStrap 3 и jQuery 1.12.

Для успешной работы маршрутизации Laravel требует подключения модуля Apache mod_rewrite.

Вся работа с БД ведется через ORM Laravel, описание моделей в корне папки App.

Для удобства были разработаны репозитории, расположены в папке App\Repositories. Репозитории по сути очень походит на представления БД, но реализованы с применением ORM и могут содержать постобработку полученных коллекций моделей.

Вся бизнес-логика расположена в папке контроллеров App\Http\Controllers.

# Структура БД #
+	Сущность клиента **clients** содержит общую информацию о клиентах (имя, страна, город регистрации).
```
client_id - первичный ключ
name - уникальное имя пользователя
country - страна регистрации
city - город регистрации
reg_date - дата регистрации
```

+	Сущность словаря валют **currency_dicts** содержит названия валют, названия валют ISO и множитель дробной части (10 в степени количества знаков после запятой). 
```
currency_id - первичный ключ
currency_name - название валюты
currency - сокращение валюты по ISO
fractional - дробная часть
```

+	Сущность кошелек **wallets** содержит информацию о кошельке клиента. Ссылается на сущности **clients** и **currency_dicts**, содержит текущий баланс кошелька. Специально отделил от информации о клиенте данную сущность для последующего масштабирования приложения в направлении привязки к одному клиенту нескольких счетов. Баланс хранится в bigint для снижения проблем с хранение чисел в типах данных с плавающей запятой.
```
wallet_id - первичный ключ
client_id - идентификатор клиента
currency_id - идентификатор валюты
balance bigint - баланс
```

+	Сущность обменный курс **exchange_rates** содержит информацию о курсе валют к доллару на конкретную дату. Ссылается на словарь валют **currency_dicts**.
```
rate_id - первичный ключ
currency_id - идентификатор валюты
rate - курс валюты к доллару
rate_date - дата курса
update_date - дата добавления курса
```

+	Сущность транзации **transactions** является входным массивом данных по перемещению денежных средств. Ссылается на кошельки **wallets**, **currency_dicts**.
```
tran_id идентификатор
wallet_id_from кошелек, с которого снимают деньги. Если null, то идет пополнение счета
wallet_id_to кошелек, на который поступают деньги
currency_id идентификатор валюты операции
amount сумма операции
operation название операции. REFILL – пополнение, TRANSFER - перевод
tran_date дата операции
status статус транзакции 
```

+	Сущность последняя транзакия **last_transactions** хранит номер транзакции следующей на очереди обработки.
```
tran_id - идентифиатор транзакции
```

+	Сущность операций **operations** хранит информацию о виде операции над счетами, сумму операции в долларах.
```
oper_id - первичный ключ
currency_id - идентификатор валюты операции
operation - название операции. REFILL – пополнение, TRANSFER - перевод
oper_date - дата операции
oper_amount - сумма операции в валюте операции
usd_amount - сумма операции в долларах 
```

+	Сущность история по кошельку **wallet_hists** хранит информацию о всех переводах кошелка. Ссылается на сущность operations, для четкого установления какие кошелки участвовали в операции.
```
hist_id - первичный ключ
wallet_id - идентификатор кошелька
wallet_partner_id - идентификатор кошелька, с которым проводилась операция
wallet_partner_name - имя клиента, с которым проводилась операция
oper_id - идентификатор операции
hist_date - дата операции
type - тип операции (списание, пополнение)
amount - сумма операции в валюте кошелька
```

Миграции расположены в папке database\migrations. Генерация первичных данных БД описана в database\seeds.

Внешние ключи убрал для снижения нагрузки при добавлении данных. Настроены необходимые индексы.

В ряде репозиториев используется изменение уровня изоляции транзакции для реализации uncommitted read. Этот режим используется, когда нам нужно прочитать информацию с таблиц в которые мы либо не будем писать, либо которые редко обновляются (словарь валют, курсы валют).

# HTTP API #
Для примера буду описывать приложение использующее домен http://wallet/.

Для HTTP API была отключена проверка CSRF-токенов, чтобы исключить необходимость в авторизации.

Все запросы возвращают ответ с кодом 200, если все хорошо, либо с кодом 400, если запрос не удовлетворяет каким-либо требованиям.

+	Регистрация нового клиента осуществляется передачей PUT-запроса по адресу
http://wallet/public/api/client. В запросе необходимо передать параметры name, name, city, currency (сокращенное название валюты ISO).

	Данный запрос вызовет функцию create контроллер ClientController. Будут заполнены данные о клиенте в сущностях client и wallet. Заполняется с нулевым балансом. Имя считаем уникальным, что-то вроде логина.

+	Пополнение счета осуществляется передачей POST-запроса по адресам http://wallet/public/api/client/**{name}**, либо http://wallet/public/api/ wallet/**{wallet_id}**. Пополняем либо по имени клиента (**{name}**), либо во-втором случае по номеру счета (**{wallet_id}**). В теле запроса необходимо передать сумму пополнения (**amount**) в валюте счета.

	Данный запрос вызовет функцию clientTransaction, либо walletTransaction контроллера TransactionController. Будет создана новая транзакция, которая будет ждать момента исполнения. Перед запись производятся проверки на существование счета, вида валюты.

+	Перевод между кошельками осуществляется передачей POST-запроса по адресам http://wallet/public/api/client/**{name_from}**/**{name_to}**, либо http://wallet/public/api/wallet/**{wallet_id_from}**/**{wallet_id_to}**. Как и в случае с пополнением можем работать либо по именам клиентов, либо по номерам счетов. В теле запроса необходимо передать сумму пополнения (**amount**) и в валюте какого счета производится операция (currency_use: ‘FROM’ или ‘TO’).

	Данный запрос вызовет функцию clientTransferTransaction, либо walletTransferTransactionконтроллера TransactionController. Также будет создана новая транзакция, которая будет ждать момента исполнения. 

+	Загрузка котировок осуществляется передачей PUT-запроса по адресу http://wallet/public/api/client/exchange_rate. В теле запроса необходимо передать ISO сокращение названия валюты (currency), котировку (rate) и дату котировки (**rate_date**, формат ‘Y-m-d’).

	Данный запрос вызовет функцию add контроллера ExchangeRateController. Новая котировка будет записана в БД.

# Обработка транзакций #
Транзакции обрабатываются в несколько процессов: первичная синхронизация пакета данных в один поток, построчное выполнение транзакций в несколько процессов и финальная синхронизация в один поток.

Все процессы реализованы через очереди Laravel, их описание располагается в App/Jobs/. Для запуска основного процесса была разработана консольная команда, располагается в App/Console/ Commands.

## Основной процесс ##
Процесс использует контроллер MainJobController.

Берется из таблицы last_transaction номер последней выполненной транзакции и процесс делает первичную синхронизацию: проставляет для следующих 500 строк (размер пакета по-умолчанию) статус 'START'. После этого добавляет в несколько очередей (по количеству обработчиков) события на каждую выделенную транзакцию.

Каждые 10 секунд процесс проверяет выполняются ли дочерние процессы. Если они все выполнены, то проверяется все ли транзакции пакета выполнены (на случай убийства процессов). Если какие-то транзакции не выполнены они опять раздаются обработчикам.

Когда все транзакции пакета имеют статус 'DONE' происходит однопоточная финализация пакета: устанавливается статус 'ENDED' у транзакций. Номер последней транзакции записывается в сущность last_transaction.

## Процессы-обработчики ##
Процессы-обработчики реализованы контроллером TransactionController.

Каждому обработчику передается модель конкретной транзакции. Обработчик создает операцию в таблице operation, определяет курс валют кошельков, участвующих в транзакции, вычисляет суммы в долларах и в валютах кошельков.

Для каждого кошелька, участвующего в операции, создается запись в истории (сущность wallet_hist). В этой же транзакции инкрементально изменяется баланс в сущности wallet.

За счет инкрементального изменения всегда гарантируем, что после обработки всего пакета транзакций будет верный баланс, даже при параллельной правке баланса одного кошелька.

После обработки у транзакции меняется статус на 'DONE', транзакция удаляется из очереди обработки процесса-обработчика.

# Отчет #
Для отчета на этапе обработки транзакций максимально удобно раскиданы данные по таблицам. Отчет работает по индексам и при своем формировании использует контроллер WalletHistController.

Отчет доступен по адресу http://wallet/public/. Отчет реализован с использованием шаблонизатоора Laravel – Blade. В верстке использовался Bootstrap, для обработки действий пользователей используется jQuery. 

Для удобства сделал выпадающий список с именами первых 50 клиентов.

Для выгрузки использовался формат CSV, генерируемый по таблице с отчетым данными jQuery-плагином.
	
# Оценка решения #
Плюсы:

+	Изначально заложены задатки на масштабирование решения
+	Для ускорения работы была проведена работа по выбору необходимых индексов, приняты решения по использованию uncommitted read для ряда ситуаций.
+	За счет использования ORM решение переносимое на другие БД.
+	Быстрый отклик HTTP API за счет переноса обработки транзакций с использованием очереди обработки

Минусом можно отметить некоторую избыточность хранимых данных и короткой фазой однопоточной обработки очереди транзакций.

# Развертывание решения #
```
>>> git clone https://github.com/matskoigor/wallet.git
>>> cd wallet
```

Настроить коннект к БД в файле **.ENV**.

Для развертывания миграций и первоначального наполнения БД необходимо выполнить команду:
```
>>> php artisan migrate:install
>>> php artisan migrate
>>> php artisan db:seed
```
Затем необходимо в отдельных процессах запустить обработчики очередей. Пример для запуска  основного процесса и 5 обработчиков:
```
>>> php artisan queue:work --queue=MainTransaction
>>> php artisan queue:work --queue=tran1
>>> php artisan queue:work --queue=tran2
>>> php artisan queue:work --queue=tran3
>>> php artisan queue:work --queue=tran4
>>> php artisan queue:work --queue=tran5
```

Затем необходимо вызвать команду:
```
>>> php artisan transaction:run
```
Данная команда может получать параметры:
```
--sleep – время между простоя основного процесса. По-умолчанию 10 секунд.
--worker_cnt – количество обработчиков. По-умолчанию 5 процессов.
--pack_cnt – размер пакета обработки (количество обрабатываемых транзакций). По-умолчанию 500 строк.
--logging – признак логирования обработки в файл \storage\logs\laravel.log. По-умолчанию логирование выключено, т.к. генерирует большой объем данных.
```

В очередь MainTransaction через команду TransactionCommand будет добавлено новое событие.

После этого начнет обрабатываться очередь транзакций и можно будет через веб-форму смотреть отчет, либо пользоваться HTTP API.

Для управления обработчиками очередей в UNIX-системах возможно использовать удобное решение [Supervisor](http://supervisord.org).

