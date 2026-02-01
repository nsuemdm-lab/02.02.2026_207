02.02.2026

Группа: 9-ИС207 Дата: 2 февраля 2026 г. (понедельник) Дисциплина: МДК.09.03 Обеспечение безопасности веб-приложений (2 пары подряд) Аудитория: 5-509 (Очный формат, но работаем с хостингом Beget)

ОБЩАЯ КОНЦЕПЦИЯ ДНЯ: ПЕРСОНАЛИЗАЦИЯ И ЗАЩИТА ДАННЫХ Статус:

    Есть регистрация/вход (22.01).
    Есть каталог и создание заказов (28.01).
    Проблема: Пользователь не видит историю своих заказов.
    Угроза (МДК.09.03): IDOR (Insecure Direct Object References). Если я поменяю ID в адресной строке, увижу ли я чужой заказ?

Цель дня: Реализовать Личный кабинет (Profile) и защитить его от просмотра чужих данных (Anti-IDOR) и подделки запросов (Anti-CSRF).

11:25 | ПАРА №1. IDOR: Уязвимость прямых ссылок Тема: Реализация Личного кабинета и защита приватности. ВВЕДЕНИЕ: На прошлом занятии мы сделали админку, где Администратор видит ВСЕ заказы. Сегодня мы делаем страницу profile.php, где пользователь должен видеть ТОЛЬКО СВОИ заказы. Главная ошибка новичков (IDOR): Делать страницу view_order.php?id=5, которая показывает заказ №5, не проверяя, кому он принадлежит.

ЗАДАНИЕ А. Личный кабинет (Список) Создайте файл profile.php. Задача: Вывести список заказов текущего пользователя.

<?php
session_start();
require 'db.php';

if (!isset($_SESSION['user_id'])) {
    header("Location: login.php");
    exit;
}

$user_id = $_SESSION['user_id'];

// БЕЗОПАСНОСТЬ: В запросе ОБЯЗАТЕЛЬНО должно быть WHERE user_id = ...
// Иначе пользователь увидит всю базу.
$stmt = $pdo->prepare("
    SELECT orders.*, products.title, products.price 
    FROM orders 
    JOIN products ON orders.product_id = products.id 
    WHERE orders.user_id = ? 
    ORDER BY orders.created_at DESC
");
$stmt->execute([$user_id]);
$my_orders = $stmt->fetchAll();
?>
<!-- HTML часть: Таблица с заказами -->
<h1>Мои заказы</h1>
<!-- ... цикл foreach ... -->

Вот полный, готовый к работе код файла profile.php.

Добавлен HTML-каркас (Bootstrap 5), навигацию и обработку ситуации, когда у пользователя еще нет заказов. Также включена защита вывода данных (htmlspecialchars), о которой мы говорили на прошлом семинаре.
Файл: profile.php

<?php
// 1. Начинаем сессию и подключаемся к базе
session_start();
require 'db.php';

// 2. Проверка доступа: Если не вошел — отправляем на вход
if (!isset($_SESSION['user_id'])) {
    header("Location: login.php");
    exit;
}

$user_id = $_SESSION['user_id'];

// 3. БЕЗОПАСНЫЙ ЗАПРОС (Anti-IDOR)
// Мы выбираем только те заказы, где user_id совпадает с текущим пользователем.
// Используем JOIN, чтобы получить название товара и цену из таблицы products.
$sql = "
    SELECT 
        orders.id as order_id, 
        orders.created_at, 
        orders.status, 
        products.title, 
        products.price,
        products.image_url
    FROM orders 
    JOIN products ON orders.product_id = products.id 
    WHERE orders.user_id = ? 
    ORDER BY orders.created_at DESC
";

$stmt = $pdo->prepare($sql);
$stmt->execute([$user_id]);
$my_orders = $stmt->fetchAll();
?>

<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Личный кабинет</title>
    <!-- Подключаем Bootstrap для красоты -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="bg-light">

    <!-- Навигация -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark mb-4">
        <div class="container">
            <a class="navbar-brand" href="index.php">Мой Проект</a>
            <div class="d-flex">
                <span class="navbar-text text-white me-3">
                    Вы вошли как: <b><?= htmlspecialchars($_SESSION['user_role'] ?? 'User') ?></b>
                </span>
                <a href="logout.php" class="btn btn-outline-light btn-sm">Выйти</a>
            </div>
        </div>
    </nav>

    <div class="container">
        <div class="row">
            <div class="col-md-12">
                <div class="card shadow-sm">
                    <div class="card-header bg-white">
                        <h2 class="mb-0">Мои заказы</h2>
                    </div>
                    <div class="card-body">
                        
                        <!-- Проверка: Есть ли заказы вообще? -->
                        <?php if (count($my_orders) > 0): ?>
                            
                            <div class="table-responsive">
                                <table class="table table-hover align-middle">
                                    <thead class="table-light">
                                        <tr>
                                            <th>№ Заказа</th>
                                            <th>Дата</th>
                                            <th>Товар</th>
                                            <th>Цена</th>
                                            <th>Статус</th>
                                        </tr>
                                    </thead>
                                    <tbody>
                                        <?php foreach ($my_orders as $order): ?>
                                            <tr>
                                                <!-- ID заказа -->
                                                <td>#<?= $order['order_id'] ?></td>
                                                
                                                <!-- Дата (форматируем красиво) -->
                                                <td>
                                                    <?= date('d.m.Y H:i', strtotime($order['created_at'])) ?>
                                                </td>
                                                
                                                <!-- Название товара (защита от XSS) -->
                                                <td>
                                                    <strong><?= htmlspecialchars($order['title']) ?></strong>
                                                </td>
                                                
                                                <!-- Цена -->
                                                <td><?= number_format($order['price'], 0, '', ' ') ?> ₽</td>
                                                
                                                <!-- Статус с цветным бейджиком -->
                                                <td>
                                                    <?php 
                                                    // Логика цвета для статуса
                                                    $status_color = 'secondary';
                                                    if ($order['status'] == 'new') $status_color = 'primary';
                                                    if ($order['status'] == 'processing') $status_color = 'warning';
                                                    if ($order['status'] == 'done') $status_color = 'success';
                                                    ?>
                                                    <span class="badge bg-<?= $status_color ?>">
                                                        <?= htmlspecialchars($order['status']) ?>
                                                    </span>
                                                </td>
                                            </tr>
                                        <?php endforeach; ?>
                                    </tbody>
                                </table>
                            </div>

                        <?php else: ?>
                            <!-- Если заказов нет -->
                            <div class="text-center py-5">
                                <h4 class="text-muted">Вы еще ничего не заказывали.</h4>
                                <a href="index.php" class="btn btn-primary mt-3">Перейти в каталог</a>
                            </div>
                        <?php endif; ?>

                    </div>
                </div>
            </div>
        </div>
    </div>

</body>
</html>

Что здесь важно (для сдачи преподавателю):

    WHERE orders.user_id = ?: Это защита от IDOR. Мы жестко фильтруем выдачу.
    htmlspecialchars(...): Это защита от XSS. Если в названии товара будет вирусный скрипт, он не сработает.
    if (count($my_orders) > 0): Улучшение UX (User Experience). Мы не показываем пустую таблицу, а пишем понятное сообщение.

ЗАДАНИЕ Б. Просмотр деталей (Защита от IDOR) Допустим, у вас есть страница order_details.php?id=10. Сценарий атаки: Студент А (id=5) меняет в строке id=10 на id=11 (заказ Студента Б). Если сайт показывает данные — это «двойка» по безопасности.

Правильный код (order_details.php):

$order_id = (int)$_GET['id'];
$user_id = $_SESSION['user_id'];

// ПРОВЕРКА ВЛАДЕЛЬЦА
// Мы ищем заказ с таким ID И (AND) принадлежащий этому юзеру.
$stmt = $pdo->prepare("SELECT * FROM orders WHERE id = ? AND user_id = ?");
$stmt->execute([$order_id, $user_id]);
$order = $stmt->fetch();

if (!$order) {
    // Важно: Не пишите "Это чужой заказ". Пишите "Заказ не найден".
    // Это защита от перебора (User Enumeration).
    die("Заказ не найден или у вас нет прав на его просмотр.");
}

13:20 | ПАРА №2. CSRF: Подделка межсайтовых запросов Тема: Защита форм изменения данных. ТЕОРИЯ: Представьте, что хакер создал сайт с картинкой, у которой src="http://vash-site.ru/delete_profile.php". Если вы залогинены на своем сайте и откроете сайт хакера, браузер попытается загрузить "картинку" и удалит ваш профиль. Это CSRF.

ПРАКТИКУМ: Внедрение CSRF-токенов. Мы должны убедиться, что кнопка нажата именно на НАШЕМ сайте.

ШАГ 1. Генерация токена (в login.php или db.php) При входе пользователя генерируем случайную строку и кладем в сессию.

if (empty($_SESSION['csrf_token'])) {
    $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
}

ШАГ 2. Вставка в форму (Любая форма: Изменение пароля, Редактирование профиля)

<form action="update_profile.php" method="POST">
    <!-- Скрытое поле с секретным кодом -->
    <input type="hidden" name="csrf_token" value="<?= $_SESSION['csrf_token'] ?>">
    
    <input type="text" name="phone" placeholder="Новый телефон">
    <button type="submit">Сохранить</button>
</form>

ШАГ 3. Проверка на сервере (update_profile.php)

session_start();

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    if (!hash_equals($_SESSION['csrf_token'], $_POST['csrf_token'])) {
        die("Ошибка безопасности: Неверный CSRF-токен! Запрос отклонен.");
    }
    
    // Только теперь обновляем данные в БД...
}

ЗАДАНИЕ НА ПАРУ: Реализовать форму смены пароля или редактирования контактов с защитой CSRF.

15:05 | ПАРА №2+. Индивидуальная работа (Logic Security) Тема: Применение принципов безопасности к темам курсовых. ВВЕДЕНИЕ: У каждого из вас своя бизнес-логика. Уязвимости скрываются не только в коде, но и в логике работы. Используем список тем (см. PDF) для конкретных задач.

ИНДИВИДУАЛЬНЫЕ ЗАДАЧИ (ПО СПИСКУ ТЕМ):

    Агапов Артемий (Тема 4: LMS Платформа)
        Задача: Реализовать проверку доступа к уроку.
        Logic Check: В таблице lessons есть course_id. В таблице orders есть покупки. Перед показом видео проверить: есть ли у current_user активный заказ на этот course_id?

    Белозерцева Дарья (Тема 9: Биржа фриланса)
        Задача: Ролевая модель в действиях.
        Logic Check: Кнопку "Взять заказ" видит только freelancer. Кнопку "Принять работу" видит только customer (автор задачи).
        Anti-IDOR: Фрилансер не может отправить решение к задаче, на которую он не назначен.

    Дмитриев Леонид (Тема 1: Автосервис)
        Задача: Отмена записи.
        Logic Check: Пользователь может отменить запись (DELETE) только если:
            Запись принадлежит ему (WHERE user_id = ?).
            Дата записи еще не наступила (нельзя отменить прошлое).

    Вотинцев Александр (Тема 19: Магазин ПК)
        Задача: Корзина и оформление.
        Security: Проверка цены на сервере. Не брать цену из HTML формы (пользователь может поменять <input value="100"> на 1). Цену всегда брать SELECT price FROM products.

    Ибрагимов Роман (Тема 3: Ветклиника)
        Задача: Приватность медицинских данных.
        Access Control: Историю болезни питомца видит только Владелец и Врач. Любой другой авторизованный пользователь (сосед) не должен иметь доступа по ссылке pet_history.php?id=55.

    Торшин Евгений (Тема 14: Личный кабинет ТСЖ)
        Задача: Ввод показаний счетчиков.
        Validation: Нельзя ввести показания меньше, чем были в прошлом месяце. Нельзя ввести отрицательные числа.

ДЛЯ ВСЕХ ОСТАЛЬНЫХ: Реализовать сценарий "Смена пароля".

    Форма требует: Старый пароль, Новый пароль, Повтор нового.
    Проверка: password_verify для старого пароля.
    Валидация: Новый пароль не короче 8 символов.
    Хеширование: password_hash для нового.
    Защита: CSRF-токен обязателен.

КРИТЕРИИ ОЦЕНКИ ЗА ДЕНЬ:

    3 балла: Есть страница profile.php, выводящая заказы.
    4 балла: Реализована защита от IDOR (нельзя посмотреть чужой заказ перебором ID).
    5 баллов: Реализована защита от CSRF на любой форме изменения данных (пароль/профиль) + работающая логика по индивидуальной теме.

ДОМАШНЕЕ ЗАДАНИЕ: Подготовить проект к развертыванию на "чистовик". Проверить, нет ли в коде var_dump, print_r и ошибок PHP, видимых пользователю. Настроить display_errors = Off (мысленно готовимся к этому).
Примеры индивидуализации заданий

Ниже приведены примеры реализации кода (PHP + PDO) для каждой индивидуальной задачи. Эти примеры можно использовать как шаблон ("рыбу") для вставки в свои проекты.

Важное примечание для всех: Во всех примерах подразумевается, что файл db.php (подключение к БД) и session_start() уже подключены.
1. Агапов Артемий (LMS Платформа)

Задача: Проверка доступа к уроку (Купил — смотри, не купил — плати).

Файл: view_lesson.php

<?php
// Получаем ID урока
$lesson_id = (int)$_GET['id'];
$user_id = $_SESSION['user_id'];

// 1. Узнаем, к какому курсу относится урок
$stmt = $pdo->prepare("SELECT * FROM lessons WHERE id = ?");
$stmt->execute([$lesson_id]);
$lesson = $stmt->fetch();

if (!$lesson) die("Урок не найден");

// 2. ПРОВЕРКА ДОСТУПА (Logic Check)
// Ищем в таблице заказов запись, где user_id = текущий юзер
// И product_id = курс этого урока
// И статус заказа = 'paid' (оплачен)
$access_check = $pdo->prepare("
    SELECT id FROM orders 
    WHERE user_id = ? AND product_id = ? AND status = 'paid'
");
$access_check->execute([$user_id, $lesson['course_id']]);
$has_access = $access_check->fetch();

// 3. Логика отображения
if ($has_access) {
    // Показываем видео
    echo "<h1>" . htmlspecialchars($lesson['title']) . "</h1>";
    echo "<video src='" . htmlspecialchars($lesson['video_url']) . "' controls></video>";
} else {
    // Блокируем
    echo "<h1>Доступ закрыт</h1>";
    echo "<p>Чтобы посмотреть этот урок, купите курс.</p>";
    echo "<a href='buy_course.php?id=" . $lesson['course_id'] . "' class='btn btn-primary'>Купить курс</a>";
}
?>

2. Белозерцева Дарья (Биржа фриланса)

Задача: Ролевая модель (Кнопки) и Anti-IDOR (Защита действия).

Часть А: Визуализация (в файле task_view.php)

<?php
// Предположим, мы уже загрузили данные задачи в переменную $task
$current_role = $_SESSION['user_role']; // 'freelancer' или 'customer'
?>

<!-- Кнопка для Фрилансера: Взять заказ -->
<?php if ($current_role === 'freelancer' && $task['status'] === 'open'): ?>
    <form action="take_task.php" method="POST">
        <input type="hidden" name="task_id" value="<?= $task['id'] ?>">
        <button type="submit">Взять в работу</button>
    </form>
<?php endif; ?>

<!-- Кнопка для Заказчика: Принять работу -->
<!-- Показываем только если статус 'review' (на проверке) и это МОЯ задача -->
<?php if ($current_role === 'customer' && $task['status'] === 'review' && $task['customer_id'] == $_SESSION['user_id']): ?>
    <a href="accept_work.php?id=<?= $task['id'] ?>" class="btn btn-success">Принять работу</a>
<?php endif; ?>

Часть Б: Защита действия (в файле submit_solution.php)

<?php
$task_id = (int)$_POST['task_id'];
$freelancer_id = $_SESSION['user_id'];

// ANTI-IDOR ПРОВЕРКА:
// Пытаемся отправить решение. Но сервер должен проверить:
// А правда ли этот юзер назначен исполнителем этой задачи?
$check = $pdo->prepare("SELECT id FROM tasks WHERE id = ? AND executor_id = ?");
$check->execute([$task_id, $freelancer_id]);

if (!$check->fetch()) {
    die("Ошибка: Вы не являетесь исполнителем этой задачи!");
}

// Если проверка прошла — сохраняем решение
// ... UPDATE tasks SET status = 'review' ...
?>

3. Дмитриев Леонид (Автосервис)

Задача: Отмена записи (Временнáя логика).

Файл: cancel_booking.php

<?php
$booking_id = (int)$_POST['id'];
$user_id = $_SESSION['user_id'];

// LOGIC CHECK + SECURITY:
// Удаляем запись ТОЛЬКО если:
// 1. Она принадлежит этому пользователю (user_id = ?)
// 2. Время записи (booking_date) больше текущего времени (NOW())
// Нельзя отменить то, что уже прошло.

$sql = "DELETE FROM bookings 
        WHERE id = ? 
        AND user_id = ? 
        AND booking_date > NOW()";

$stmt = $pdo->prepare($sql);
$stmt->execute([$booking_id, $user_id]);

if ($stmt->rowCount() > 0) {
    echo "Запись успешно отменена.";
} else {
    echo "Ошибка: Нельзя отменить эту запись. Либо она не ваша, либо время уже вышло.";
}
?>

4. Вотинцев Александр (Магазин ПК)

Задача: Защита цены (Backend Validation).

Файл: add_to_cart.php

НЕПРАВИЛЬНО (Как делают новички - Уязвимость): $price = $_POST['price']; // Хакер может подменить HTML и отправить цену 1 рубль

ПРАВИЛЬНО (Secure way):

<?php
$product_id = (int)$_POST['product_id'];
$quantity = (int)$_POST['quantity'];

if ($quantity <= 0) die("Неверное количество");

// 1. Игнорируем цену, пришедшую от формы.
// 2. Спрашиваем цену у Базы Данных.
$stmt = $pdo->prepare("SELECT price, title FROM products WHERE id = ?");
$stmt->execute([$product_id]);
$product = $stmt->fetch();

if (!$product) die("Товар не найден");

$real_price = $product['price']; // Цена из базы (например, 50000)

// 3. Считаем сумму на сервере
$total_cost = $real_price * $quantity;

// 4. Сохраняем в заказ/корзину
$sql = "INSERT INTO cart_items (user_id, product_id, price, quantity) VALUES (?, ?, ?, ?)";
$stmt = $pdo->prepare($sql);
$stmt->execute([$_SESSION['user_id'], $product_id, $real_price, $quantity]);

echo "Товар добавлен. Цена: $real_price руб.";
?>

5. Ибрагимов Роман (Ветклиника)

Задача: Приватность данных (Access Control List).

Файл: pet_history.php

<?php
$pet_id = (int)$_GET['id'];
$current_user = $_SESSION['user_id'];
$current_role = $_SESSION['user_role']; // 'owner' или 'vet'

// 1. Получаем данные о питомце, чтобы узнать, кто хозяин
$stmt = $pdo->prepare("SELECT owner_id, name FROM pets WHERE id = ?");
$stmt->execute([$pet_id]);
$pet = $stmt->fetch();

if (!$pet) die("Питомец не найден");

// 2. ПРОВЕРКА ПРАВ (Сложное условие)
// Доступ разрешен, если:
// (Я - хозяин этого питомца) ИЛИ (Я - врач)
$is_owner = ($pet['owner_id'] == $current_user);
$is_vet   = ($current_role === 'vet');

if (!$is_owner && !$is_vet) {
    // Логируем попытку взлома (опционально)
    die("ДОСТУП ЗАПРЕЩЕН. Это не ваша собака/кошка, и вы не врач.");
}

// 3. Если прошли проверку — показываем историю болезней
$history = $pdo->prepare("SELECT * FROM medical_records WHERE pet_id = ?");
$history->execute([$pet_id]);
// ... вывод таблицы ...
?>

6. Торшин Евгений (ТСЖ)

Задача: Валидация показаний счетчиков.

Файл: submit_meter.php

<?php
$meter_id = (int)$_POST['meter_id'];
$new_value = (float)$_POST['value']; // Новое показание
$user_id = $_SESSION['user_id'];

// 1. Проверка на отрицательные числа
if ($new_value < 0) {
    die("Ошибка: Показания не могут быть отрицательными.");
}

// 2. Получаем ПОСЛЕДНЕЕ показание из базы для этого счетчика
// Сортируем по дате убывания и берем одну запись (LIMIT 1)
$stmt = $pdo->prepare("
    SELECT value FROM meter_readings 
    WHERE meter_id = ? 
    ORDER BY created_at DESC 
    LIMIT 1
");
$stmt->execute([$meter_id]);
$last_reading = $stmt->fetch();

// 3. Сравниваем
if ($last_reading) {
    $old_value = $last_reading['value'];
    
    if ($new_value < $old_value) {
        die("Ошибка: Новые показания ($new_value) не могут быть меньше старых ($old_value). Вы скручиваете счетчик?");
    }
}

// 4. Если всё ок — сохраняем
$insert = $pdo->prepare("INSERT INTO meter_readings (meter_id, value) VALUES (?, ?)");
$insert->execute([$meter_id, $new_value]);

echo "Показания приняты.";
?>

Дополнение в блок «15:05 | ПАРА №2+. Индивидуальная работа»

ИНДИВИДУАЛЬНЫЕ ЗАДАЧИ (ПРОДОЛЖЕНИЕ СПИСКА):

    Барабашов Давид (Тема 5: Агрегатор мероприятий)

        Задача: Ограничение продажи билетов.

        Logic Check: Перед созданием заказа проверить, не превышает ли количество покупаемых билетов остаток в БД (available_tickets). Нельзя продать 11-й билет, если площадка вмещает 10.

    Кудрявцев Дмитрий (Тема 13: Бронирование переговорных)

        Задача: Валидация временных слотов.

        Logic Check: Предотвратить "наложение" броней. Нельзя забронировать комнату на 14:00–15:00, если она уже занята кем-то с 14:30.

    Савинкова Анна (Тема 11: Доставка еды)

        Задача: Конструктор блюд и расчет стоимости.

        Security: Защита от изменения цены топпингов. Цена каждого ингредиента (сыр, соус, бекон) должна браться из БД, а не из скрытых полей формы.

    Сенеджук Глеб (Тема 24: Агентство недвижимости)

        Задача: Скрытие личных данных владельца.

        Access Control: Телефон владельца квартиры видит только авторизованный риелтор. Обычный посетитель сайта видит только общие характеристики объекта.

    Янкулев Владислав (Тема 12: HelpDesk)

        Задача: Жизненный цикл тикета.

        Logic Check: Пользователь (клиент) может перевести тикет только в статус "Отменен". Перевести тикет в статус "Решен" или "Закрыт" может только сотрудник техподдержки.

Примеры реализации (добавление в блок с кодом)
7. Кудрявцев Дмитрий (Бронирование переговорных)

Задача: Проверка пересечения времени (Collision Check).
code PHP

<?php
$room_id = (int)$_POST['room_id'];
$start = $_POST['start_time']; // '2026-01-29 14:00:00'
$end = $_POST['end_time'];     // '2026-01-29 15:00:00'

// LOGIC: Ищем записи, которые пересекаются с выбранным интервалом
$sql = "SELECT id FROM bookings 
        WHERE room_id = ? 
        AND (
            (start_time < ? AND end_time > ?) -- Новое начало внутри старого интервала
            OR (start_time < ? AND end_time > ?) -- Конец внутри старого
            OR (? <= start_time AND ? >= end_time) -- Старый интервал целиком внутри нового
        )";

$stmt = $pdo->prepare($sql);
$stmt->execute([$room_id, $start, $start, $end, $end, $start, $end]);

if ($stmt->fetch()) {
    die("Ошибка: Это время уже занято кем-то другим.");
}

// Если пусто — бронируем...
?>

8. Барабашов Давид (Агрегатор мероприятий)

Задача: Проверка лимита мест (Race Condition Prevention).
code PHP

<?php
$event_id = (int)$_POST['event_id'];
$tickets_to_buy = (int)$_POST['quantity'];

// 1. Проверяем остаток в БД
$stmt = $pdo->prepare("SELECT available_tickets FROM events WHERE id = ?");
$stmt->execute([$event_id]);
$event = $stmt->fetch();

if ($event['available_tickets'] < $tickets_to_buy) {
    die("Ошибка: Извините, осталось всего " . $event['available_tickets'] . " билетов.");
}

// 2. Уменьшаем количество и создаем заказ
$update = $pdo->prepare("UPDATE events SET available_tickets = available_tickets - ? WHERE id = ?");
$update->execute([$tickets_to_buy, $event_id]);

// ... создание записи в таблице orders ...
?>

9. Янкулев Владислав (HelpDesk)

Задача: Разграничение прав на смену статуса.
code PHP

<?php
$ticket_id = (int)$_POST['ticket_id'];
$new_status = $_POST['status']; // 'done', 'canceled', 'in_progress'
$user_id = $_SESSION['user_id'];
$role = $_SESSION['user_role']; // 'client' или 'agent'

// Получаем текущие данные тикета
$stmt = $pdo->prepare("SELECT user_id, status FROM tickets WHERE id = ?");
$stmt->execute([$ticket_id]);
$ticket = $stmt->fetch();

// LOGIC CHECK:
if ($role === 'client') {
    // Клиент может ТОЛЬКО отменить свой тикет
    if ($new_status !== 'canceled' || $ticket['user_id'] != $user_id) {
        die("Ошибка: Вы можете только отменить свою заявку.");
    }
}

if ($role === 'agent') {
    // Агент не может отменить (это делает клиент), но может закрыть
    if ($new_status === 'canceled') {
        die("Ошибка: Только клиент может отменить заявку.");
    }
}

// Если проверки прошли — обновляем
$upd = $pdo->prepare("UPDATE tickets SET status = ? WHERE id = ?");
$upd->execute([$new_status, $ticket_id]);
?>

10. Савинкова Анна (Доставка еды)

Задача: Безопасный расчет стоимости сложного блюда.
code PHP

<?php
$base_pizza_id = (int)$_POST['pizza_id'];
$toppings = $_POST['toppings']; // Массив ID: [5, 12, 18]

// 1. Берем цену основы из БД
$stmt = $pdo->prepare("SELECT price FROM products WHERE id = ?");
$stmt->execute([$base_pizza_id]);
$total_price = $stmt->fetchColumn();

// 2. Берем цены топпингов из БД (Anti-Manipulation)
if (!empty($toppings)) {
    // Создаем строку знаков вопроса для IN (?,?,?)
    $placeholders = str_repeat('?,', count($toppings) - 1) . '?';
    $stmt = $pdo->prepare("SELECT SUM(price) FROM ingredients WHERE id IN ($placeholders)");
    $stmt->execute($toppings);
    $total_price += $stmt->fetchColumn();
}

echo "Итоговая сумма заказа: " . $total_price . " руб.";
// Теперь можно сохранять в БД
?>
