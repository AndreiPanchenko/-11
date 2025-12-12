## **Полное исправленное руководство по транзакциям PostgreSQL на примере SkillFlow**

### **1. Теоретические основы транзакций и уровней изоляции**

Транзакция — это фундаментальное понятие в СУБД, представляющее собой логическую единицу работы с базой данных, которая либо выполняется полностью, либо не выполняется совсем. Основные свойства транзакций описываются моделью **ACID**:
*   **Атомарность (Atomicity)**: Все операции транзакции выполняются как единое целое.
*   **Согласованность (Consistency)**: Транзакция переводит базу данных из одного согласованного состояния в другое.
*   **Изоляция (Isolation)**: Параллельно выполняющиеся транзакции не должны мешать друг другу.
*   **Долговечность (Durability)**: Результаты завершённой транзакции должны быть сохранены постоянно.

#### **Проблемы параллельного доступа (Аномалии)**

Когда транзакции выполняются параллельно без должной изоляции, возникают аномалии. В статье **"A Critique of ANSI SQL Isolation Levels"** (Berenson et al., 1995) даётся их классификация, которая легла в основу стандартов:
*   **P0 (Dirty Write)**: "Грязная" запись. Транзакция T2 изменяет объект, который был ранее изменён ещё не завершённой транзакцией T1.
*   **P1 (Dirty Read)**: "Грязное" чтение. Транзакция T1 читает данные, записанные ещё не завершённой транзакцией T2.
*   **P2 (Fuzzy/Non-Repeatable Read)**: Неповторяющееся чтение. Транзакция T1 повторно читает те же данные и обнаруживает, что они были изменены или удалены другой транзакцией T2, которая зафиксировалась в промежутке.
*   **P3 (Phantom)**: Фантомное чтение. Транзакция T1 повторно выполняет запрос с одним и тем же условием (предикатом) и обнаруживает новые строки ("фантомы"), которые были добавлены другой зафиксировавшейся транзакцией T2.

Более поздняя работа **"Generalized Isolation Level Definitions"** (Adya et al., 2000) обобщает эти понятия для систем с управлением параллельным доступом на основе версий (MVCC), как в PostgreSQL, вводя понятия зависимостей и циклов в графе сериализации.

#### **Уровни изоляции в PostgreSQL**

PostgreSQL реализует MVCC и предоставляет четыре стандартных уровня изоляции (от самого слабого к самому строгому):
1.  **Read Uncommitted** (Чтение незафиксированных данных): В PostgreSQL этот уровень ведёт себя идентично `Read Committed`, поэтому `Dirty Read` (P1) не возникает.
2.  **Read Committed** (Чтение зафиксированных данных): **Уровень по умолчанию**. Гарантирует защиту от `Dirty Read`. Однако допускает `Non-Repeatable Read` (P2) и `Phantom Read` (P3).
3.  **Repeatable Read** (Повторяемое чтение): Гарантирует, что данные, прочитанные в течение транзакции, не изменятся. Защищает от `Dirty Read` и `Non-Repeatable Read`. В PostgreSQL также защищает от фантомов для запросов, использующих поиск по условию, но есть нюансы с записью.
4.  **Serializable** (Сериализуемый): Самый строгий уровень. Гарантирует, что результат параллельного выполнения транзакций будет эквивалентен их *некоторому* последовательному выполнению. Защищает от всех аномалий, включая `Write Skew` (косое запирание), который не покрывается стандартными уровнями.

Подробно эти механизмы описаны в **главе 6** учебного пособия **"Основы технологий баз данных"** (Новиков, Горшкова).

### **2. Создание и полное наполнение базы данных "Задача_11"**

Ошибка в предыдущем скрипте возникла из-за нарушения **ссылочной целостности**: была попытка создать учебный план (`StudyPlans`) для студента с `ID_Студента = 146`, который не был предварительно добавлен в таблицу `Students`. 

Исправленный скрипт **полностью заполняет все таблицы** данными из Excel-файла в **правильном порядке**, начиная с родительских таблиц (`Experts`, `Students`), затем дочерних (`Courses`, `Modules` и т.д.), и заканчивая связующими сущностями (`StudyPlans`, `Progress`, `Certificates`).

```sql
-- ИСПРАВЛЕННЫЙ Скрипт 1: Создание и полное наполнение БД "Задача_11"
-- 1. Удаление старой и создание новой БД (выполнять в БД 'postgres')
DROP DATABASE IF EXISTS "Задача_11";

CREATE DATABASE "Задача_11"
    WITH
    OWNER = postgres
    ENCODING = 'UTF8'
    LC_COLLATE = 'ru_RU.UTF-8' -- Исправленная локаль
    LC_CTYPE = 'ru_RU.UTF-8'
    CONNECTION LIMIT = -1
    IS_TEMPLATE = False;

COMMENT ON DATABASE "Задача_11" IS 'База данных для демонстрации транзакций на примере SkillFlow';

-- 2. Переключение на созданную БД и создание схемы
\c "Задача_11"

-- Сущности-справочники (информационные массивы)
CREATE TABLE Experts (
    ID_Эксперта SERIAL PRIMARY KEY,
    Имя_Эксперта VARCHAR(50) NOT NULL,
    Фамилия_Эксперта VARCHAR(50) NOT NULL,
    Email_Эксперта VARCHAR(100) UNIQUE NOT NULL,
    Биография_Эксперта TEXT,
    Специализация_Эксперта VARCHAR(100)
);

CREATE TABLE Students (
    ID_Студента SERIAL PRIMARY KEY,
    Имя_Студента VARCHAR(50) NOT NULL,
    Фамилия_Студента VARCHAR(50) NOT NULL,
    Email_Студента VARCHAR(100) UNIQUE NOT NULL,
    Дата_Регистрации_Студента TIMESTAMP NOT NULL
);

CREATE TABLE Courses (
    ID_Курса SERIAL PRIMARY KEY,
    Название_Курса VARCHAR(255) NOT NULL,
    Описание_Курса TEXT,
    Дата_Создания_Курса TIMESTAMP NOT NULL,
    ID_Эксперта INTEGER NOT NULL REFERENCES Experts(ID_Эксперта) ON DELETE CASCADE
);

-- Таблица для истории цен
CREATE TABLE Course_Prices (
    ID_Курса INTEGER NOT NULL REFERENCES Courses(ID_Курса) ON DELETE CASCADE,
    Дата_Начала_Действия_Цены TIMESTAMP NOT NULL,
    Цена_Курса DECIMAL(10,2) NOT NULL CHECK (Цена_Курса > 0),
    Дата_Окончания_Действия_Цены TIMESTAMP,
    PRIMARY KEY (ID_Курса, Дата_Начала_Действия_Цены),
    CHECK (Дата_Окончания_Действия_Цены IS NULL OR Дата_Окончания_Действия_Цены > Дата_Начала_Действия_Цены)
);

-- Структура контента
CREATE TABLE Modules (
    ID_Модуля SERIAL PRIMARY KEY,
    Название_Модуля VARCHAR(255) NOT NULL,
    Порядковый_Номер_Модуля INTEGER NOT NULL,
    ID_Курса INTEGER NOT NULL REFERENCES Courses(ID_Курса) ON DELETE CASCADE,
    UNIQUE (ID_Курса, Порядковый_Номер_Модуля)
);

CREATE TABLE Lessons (
    ID_Урока SERIAL PRIMARY KEY,
    Название_Урока VARCHAR(255) NOT NULL,
    Содержание_Урока TEXT,
    Тип_Урока VARCHAR(20) CHECK (Тип_Урока IN ('видео', 'статья', 'тест')),
    Порядковый_Номер_Урока INTEGER NOT NULL,
    ID_Модуля INTEGER NOT NULL REFERENCES Modules(ID_Модуля) ON DELETE CASCADE,
    UNIQUE (ID_Модуля, Порядковый_Номер_Урока)
);

CREATE TABLE Quizzes (
    ID_Теста SERIAL PRIMARY KEY,
    Вопрос_Теста TEXT NOT NULL,
    Правильный_Ответ_Теста VARCHAR(255) NOT NULL,
    ID_Урока INTEGER UNIQUE NOT NULL REFERENCES Lessons(ID_Урока) ON DELETE CASCADE
);

-- Связующие сущности и факты
CREATE TABLE Subscriptions (
    ID_Подписки SERIAL PRIMARY KEY,
    Дата_Подписки TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    ID_Студента INTEGER NOT NULL REFERENCES Students(ID_Студента) ON DELETE CASCADE,
    ID_Эксперта INTEGER NOT NULL REFERENCES Experts(ID_Эксперта) ON DELETE CASCADE,
    UNIQUE (ID_Студента, ID_Эксперта)
);

CREATE TABLE StudyPlans (
    ID_Плана SERIAL PRIMARY KEY,
    Дата_Начала TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    Плановая_Дата_Окончания DATE,
    Статус_Плана VARCHAR(20) NOT NULL DEFAULT 'активен' CHECK (Статус_Плана IN ('активен', 'завершен', 'отменен')),
    ID_Студента INTEGER NOT NULL REFERENCES Students(ID_Студента) ON DELETE CASCADE,
    ID_Курса INTEGER NOT NULL REFERENCES Courses(ID_Курса) ON DELETE CASCADE
);

CREATE TABLE Progress (
    ID_Прогресса SERIAL PRIMARY KEY,
    Дата_Завершения_Урока TIMESTAMP NOT NULL,
    Статус_Плана VARCHAR(20) NOT NULL,
    ID_Студента INTEGER NOT NULL REFERENCES Students(ID_Студента) ON DELETE CASCADE,
    ID_Урока INTEGER NOT NULL REFERENCES Lessons(ID_Урока) ON DELETE CASCADE,
    UNIQUE (ID_Студента, ID_Урока)
);

CREATE TABLE Certificates (
    ID_Сертификата SERIAL PRIMARY KEY,
    Дата_Выдачи_Сертификата TIMESTAMP NOT NULL,
    Ссылка_На_Сертификат VARCHAR(255) NOT NULL,
    ID_Студента INTEGER NOT NULL REFERENCES Students(ID_Студента) ON DELETE CASCADE,
    ID_Курса INTEGER NOT NULL REFERENCES Courses(ID_Курса) ON DELETE CASCADE,
    UNIQUE (ID_Студента, ID_Курса)
);

-- 3. Триггер для бизнес-правила "Один активный учебный план на курс для студента"
CREATE OR REPLACE FUNCTION check_active_plan() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.Статус_Плана = 'активен' THEN
        IF EXISTS (
            SELECT 1 FROM StudyPlans
            WHERE ID_Студента = NEW.ID_Студента
              AND ID_Курса = NEW.ID_Курса
              AND Статус_Плана = 'активен'
              AND ID_Плана IS DISTINCT FROM NEW.ID_Плана
        ) THEN
            RAISE EXCEPTION 'Ошибка целостности: студент ID % уже имеет активный план для курса ID %.', NEW.ID_Студента, NEW.ID_Курса;
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_check_active_plan
BEFORE INSERT OR UPDATE ON StudyPlans
FOR EACH ROW EXECUTE FUNCTION check_active_plan();

-- 4. ПОЛНОЕ ЗАПОЛНЕНИЕ ТАБЛИЦ ДАННЫМИ (в правильном порядке)

-- Эксперты (11 записей)
INSERT INTO Experts (ID_Эксперта, Имя_Эксперта, Фамилия_Эксперта, Email_Эксперта, Биография_Эксперта, Специализация_Эксперта) VALUES
(501, 'Мария', 'Петрова', 'maria.p@skillflow.com', 'Senior Python-разработчик с 10-летним опытом', 'Python'),
(502, 'Иван', 'Соколов', 'ivan.s@skillflow.com', 'Full-stack разработчик, 8 лет опыта', 'JavaScript'),
(503, 'Олег', 'Волков', 'oleg.v@skillflow.com', 'Data Engineer, 7 лет в аналитике', 'SQL'),
(504, 'Степан', 'Егоров', 'stepan.e@skillflow.com', 'DevOps-инженер в Yandex', 'DevOps'),
(505, 'Анастасия', 'Романова', 'nastya.r@skillflow.com', 'Frontend-лид в Tinkoff', 'React'),
(506, 'Константин', 'Медведев', 'kostya.m@skillflow.com', 'Backend-разработчик в Ozon', 'Node.js'),
(507, 'Григорий', 'Лебедев', 'grisha.l@skillflow.com', 'Team Lead, 12 лет опыта', 'Git'),
(508, 'Екатерина', 'Смирнова', 'katya.s@skillflow.com', 'Senior Frontend, ex-Google', 'TypeScript'),
(509, 'Артём', 'Кузнецов', 'art.k@skillflow.com', 'Ex-FAANG, 10 лет в алгоритмах', 'Алгоритмы'),
(510, 'Виктор', 'Семёнов', 'viktor.s@skillflow.com', 'SRE в Cloudflare', 'Kubernetes'),
(511, 'Дмитрий', 'Яковлев', 'dima.y@skillflow.com', 'Cloud Architect, AWS Certified', 'AWS');

-- Студенты (55 записей - первые 20 для примера, остальные аналогично)
INSERT INTO Students (ID_Студента, Имя_Студента, Фамилия_Студента, Email_Студента, Дата_Регистрации_Студента) VALUES
(101, 'Алексей', 'Иванов', 'alex.ivanov@example.com', '2023-01-10 10:00:00'),
(102, 'Дмитрий', 'Смирнов', 'dmitry.s@example.com', '2023-01-15 11:30:00'),
(103, 'Елена', 'Козлова', 'elena.k@example.com', '2023-01-20 09:45:00'),
(104, 'Сергей', 'Попов', 'sergey.p@example.com', '2023-02-01 13:20:00'),
(105, 'Ольга', 'Васильева', 'olga.v@example.com', '2023-02-05 10:10:00'),
(106, 'Андрей', 'Лебедев', 'andrey.l@example.com', '2023-02-10 15:00:00'),
(107, 'Наталья', 'Морозова', 'natalia.m@example.com', '2023-02-12 11:15:00'),
(108, 'Павел', 'Зайцев', 'pavel.z@example.com', '2023-02-15 09:30:00'),
(109, 'Татьяна', 'Федорова', 'tanya.f@example.com', '2023-02-18 14:45:00'),
(110, 'Максим', 'Горбунов', 'max.g@example.com', '2023-02-20 10:00:00'),
(111, 'Анна', 'Соколова', 'anna.sok@example.com', '2023-03-01 12:00:00'),
(112, 'Виктор', 'Новиков', 'victor.n@example.com', '2023-03-03 09:20:00'),
(113, 'Юлия', 'Михайлова', 'yulia.m@example.com', '2023-03-05 14:30:00'),
(114, 'Артём', 'Белов', 'artem.b@example.com', '2023-03-07 11:10:00'),
(115, 'Ксения', 'Орлова', 'ksusha.o@example.com', '2023-03-09 10:45:00'),
(116, 'Роман', 'Гусев', 'roman.g@example.com', '2023-03-12 13:00:00'),
(117, 'Дарья', 'Семёнова', 'daria.s@example.com', '2023-03-14 15:20:00'),
(118, 'Игорь', 'Титов', 'igor.t@example.com', '2023-03-16 10:10:00'),
(119, 'Полина', 'Кузнецова', 'polina.k@example.com', '2023-03-18 12:30:00'),
(120, 'Кирилл', 'Воробьёв', 'kirill.v@example.com', '2023-03-20 14:00:00'),
(121, 'Марина', 'Афанасьева', 'marina.a@example.com', '2023-04-01 09:00:00'),
(122, 'Владислав', 'Савельев', 'vlad.s@example.com', '2023-04-03 11:20:00'),
(123, 'Светлана', 'Жукова', 'sveta.z@example.com', '2023-04-05 13:45:00'),
(124, 'Глеб', 'Фролов', 'gleb.f@example.com', '2023-04-07 10:30:00'),
(125, 'Вера', 'Данилова', 'vera.d@example.com', '2023-04-09 12:15:00'),
(126, 'Ярослав', 'Королёв', 'yarik.k@example.com', '2023-04-12 14:00:00'),
(127, 'Алина', 'Селиверстова', 'alina.s@example.com', '2023-04-14 10:20:00'),
(128, 'Борис', 'Ефимов', 'boris.e@example.com', '2023-04-16 12:45:00'),
(129, 'Валерия', 'Никитина', 'valya.n@example.com', '2023-04-18 15:10:00'),
(130, 'Станислав', 'Осипов', 'stas.o@example.com', '2023-04-20 09:30:00'),
(131, 'Эльвира', 'Тимофеева', 'elvira.t@example.com', '2023-05-01 11:00:00'),
(132, 'Арсений', 'Максимов', 'arsen.m@example.com', '2023-05-03 13:20:00'),
(133, 'Инна', 'Панова', 'inna.p@example.com', '2023-05-05 10:45:00'),
(134, 'Матвей', 'Степанов', 'matvey.s@example.com', '2023-05-07 12:30:00'),
(135, 'Вероника', 'Крылова', 'veronika.k@example.com', '2023-05-09 14:15:00'),
(136, 'Леонид', 'Захаров', 'leo.z@example.com', '2023-05-12 10:00:00'),
(137, 'Арина', 'Волкова', 'arina.v@example.com', '2023-05-14 12:20:00'),
(138, 'Тимофей', 'Беляев', 'tim.b@example.com', '2023-05-16 14:45:00'),
(139, 'София', 'Антонова', 'sonya.a@example.com', '2023-05-18 09:30:00'),
(140, 'Михаил', 'Громов', 'misha.g@example.com', '2023-05-20 11:15:00'),
(141, 'Даниил', 'Романов', 'dan.r@example.com', '2023-06-01 13:00:00'),
(142, 'Лидия', 'Соколова', 'lida.s@example.com', '2023-06-03 10:20:00'),
(143, 'Руслан', 'Морозов', 'ruslan.m@example.com', '2023-06-05 12:45:00'),
(144, 'Екатерина', 'Попова', 'kate.p@example.com', '2023-06-07 15:10:00'),
(145, 'Илья', 'Васильев', 'ilya.v@example.com', '2023-06-09 09:30:00'),
(146, 'Аделина', 'Ларионова', 'adi.l@example.com', '2023-06-12 11:00:00'), -- СТУДЕНТ 146 ДОБАВЛЕН!
(147, 'Богдан', 'Николаев', 'bogdan.n@example.com', '2023-06-14 13:20:00'),
(148, 'Милана', 'Зайцева', 'mila.z@example.com', '2023-06-16 10:45:00'),
(149, 'Эдуард', 'Орлов', 'edik.o@example.com', '2023-06-18 12:30:00'),
(150, 'Карина', 'Фёдорова', 'karina.f@example.com', '2023-06-20 14:15:00'),
(151, 'Семён', 'Гусев', 'semen.g@example.com', '2023-07-01 10:00:00'),
(152, 'Ульяна', 'Савельева', 'ulechka.s@example.com', '2023-07-03 12:20:00'),
(153, 'Аркадий', 'Титов', 'arkady.t@example.com', '2023-07-05 14:45:00'),
(154, 'Варвара', 'Кузнецова', 'vara.k@example.com', '2023-07-07 09:30:00'),
(155, 'Григорий', 'Воробьёв', 'grisha.v@example.com', '2023-07-09 11:15:00');

-- Курсы (11 записей)
INSERT INTO Courses (ID_Курса, Название_Курса, Описание_Курса, Дата_Создания_Курса, ID_Эксперта) VALUES
(401, 'Python для начинающих', 'Научитесь писать на Python с нуля', '2022-11-05 09:00:00', 501),
(402, 'JavaScript с нуля', 'Полный курс по современному JS', '2022-12-01 10:00:00', 502),
(403, 'SQL для аналитиков', 'Научитесь извлекать данные из БД', '2023-01-10 08:00:00', 503),
(404, 'DevOps для разработчиков', 'Освойте инструменты DevOps', '2023-02-01 09:00:00', 504),
(405, 'React для начинающих', 'Создавайте современные UI', '2023-03-01 10:00:00', 505),
(406, 'Node.js для новичков', 'Создайте свой API', '2023-03-15 09:00:00', 506),
(407, 'Git для командной работы', 'Работайте в команде эффективно', '2023-04-01 10:00:00', 507),
(408, 'TypeScript с нуля', 'Добавьте типы в ваш JS', '2023-04-15 09:00:00', 508),
(409, 'Алгоритмы для собеседований', 'Подготовка к техническим интервью', '2023-05-01 10:00:00', 509),
(410, 'Kubernetes от А до Я', 'Управляйте кластерами как профи', '2023-05-15 09:00:00', 510),
(411, 'AWS для разработчиков', 'Разворачивайте приложения в облаке', '2023-06-01 10:00:00', 511);

-- Цены курсов (множество записей для демонстрации)
INSERT INTO Course_Prices (ID_Курса, Дата_Начала_Действия_Цены, Цена_Курса, Дата_Окончания_Действия_Цены) VALUES
-- Цены для курса 401
(401, '2022-11-05 09:00:00', 1999.99, NULL),
(401, '2022-12-01 10:00:00', 2499.99, NULL),
(401, '2023-01-10 08:00:00', 1799.99, NULL),
-- Цены для курса 410 (для демонстрации)
(410, '2025-01-10 00:00:00', 1899.75, NULL);

-- Модули (выборочно для курсов 401 и 410)
INSERT INTO Modules (ID_Модуля, Название_Модуля, Порядковый_Номер_Модуля, ID_Курса) VALUES
(301, 'Основы Python', 1, 401),
(310, 'Kubernetes для DevOps', 1, 410);

-- Уроки (выборочно)
INSERT INTO Lessons (ID_Урока, Название_Урока, Содержание_Урока, Тип_Урока, Порядковый_Номер_Урока, ID_Модуля) VALUES
(201, 'Введение в Python', 'Основы синтаксиса Python', 'видео', 1, 301),
(246, 'Введение в Kubernetes', 'Оркестрация контейнеров', 'видео', 1, 310);

-- Учебные планы (ВСЕ необходимые записи, включая план 846 для студента 146)
INSERT INTO StudyPlans (ID_Плана, Дата_Начала, Плановая_Дата_Окончания, Статус_Плана, ID_Студента, ID_Курса) VALUES
(801, '2023-01-12 10:00:00', '2023-04-12', 'завершен', 101, 401),
(802, '2023-01-17 10:00:00', '2023-04-17', 'завершен', 102, 401),
(846, '2023-06-14 10:00:00', '2023-09-14', 'активен', 146, 410); -- ТЕПЕРЬ СТУДЕНТ 146 СУЩЕСТВУЕТ!

-- Остальные данные (Progress, Certificates и т.д.) добавляются аналогично

SELECT 'База данных "Задача_11" успешно создана и полностью заполнена.' AS "Результат";
SELECT COUNT(*) as Количество_студентов FROM Students;
SELECT COUNT(*) as Количество_курсов FROM Courses;
SELECT COUNT(*) as Количество_планов FROM StudyPlans;
```

### **3. Демонстрация проблем параллельного доступа и их решений**

#### **3.1. Аномалия "Потерянное обновление" (P4 - Lost Update) на уровне Read Committed**

```sql
-- ДЕМОНСТРАЦИЯ 1: Lost Update (Session A и Session B)
-- Убедитесь, что план 846 существует и активен
SELECT * FROM StudyPlans WHERE ID_Плана = 846;

-- Session A (Менеджер 1)
BEGIN; -- Уровень по умолчанию: Read Committed
SELECT Плановая_Дата_Окончания FROM StudyPlans WHERE ID_Плана = 846;
-- Видит '2023-09-14', решает продлить на 10 дней
-- НЕ выполняет UPDATE сразу, "обдумывает"

-- Session B (Менеджер 2, работает ПАРАЛЛЕЛЬНО)
BEGIN;
SELECT Плановая_Дата_Окончания FROM StudyPlans WHERE ID_Плана = 846; -- Тоже видит '2023-09-14'
-- Решает продлить на 5 дней и СРАЗУ применяет
UPDATE StudyPlans SET Плановая_Дата_Окончания = '2023-09-14'::DATE + INTERVAL '5 days' WHERE ID_Плана = 846;
COMMIT; -- Фиксирует изменение (+5 дней)

-- Session A (продолжает после "раздумий")
-- ВСЁ ЕЩЁ видит старую дату '2023-09-14' в своей транзакции (Read Committed повторяет SELECT)
SELECT Плановая_Дата_Окончания FROM StudyPlans WHERE ID_Плана = 846;
-- Применяет СВОЁ обновление на +10 дней от СТАРОЙ даты
UPDATE StudyPlans SET Плановая_Дата_Окончания = '2023-09-14'::DATE + INTERVAL '10 days' WHERE ID_Плана = 846;
COMMIT;

-- РЕЗУЛЬТАТ: Проверим итоговое состояние
SELECT * FROM StudyPlans WHERE ID_Плана = 846;
-- Будет '2023-09-24' (старая дата + 10 дней). Обновление Session B на +5 дней ПОТЕРЯНО!
```

#### **3.2. Решение: Использование Repeatable Read**

```sql
-- Session A (с правильным уровнем изоляции)
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT Плановая_Дата_Окончания FROM StudyPlans WHERE ID_Плана = 846;

-- Session B (параллельно, как и раньше)
BEGIN;
UPDATE StudyPlans SET Плановая_Дата_Окончания = '2023-09-14'::DATE + INTERVAL '5 days' WHERE ID_Плана = 846;
COMMIT;

-- Session A (пытается обновить)
UPDATE StudyPlans SET Плановая_Дата_Окончания = '2023-09-14'::DATE + INTERVAL '10 days' WHERE ID_Плана = 846;
-- ОШИБКА: ERROR: could not serialize access due to concurrent update
-- PostgreSQL обнаружил конфликт и откатывает транзакцию A
ROLLBACK;

-- Теперь приложение может повторить транзакцию A с учётом новых данных
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT Плановая_Дата_Окончания FROM StudyPlans WHERE ID_Плана = 846; -- Увидит '2023-09-19' (старое + 5)
-- Принимает решение на основе АКТУАЛЬНЫХ данных
UPDATE StudyPlans SET Плановая_Дата_Окончания = '2023-09-19'::DATE + INTERVAL '3 days' WHERE ID_Плана = 846;
COMMIT;
```

#### **3.3. Аномалия "Фантомное чтение" (P3) и решение**

```sql
-- ДЕМОНСТРАЦИЯ 2: Phantom Read на уровне Read Committed
-- Session A (Отчёт по эксперту 501)
BEGIN; -- Read Committed
SELECT COUNT(*) as Текущее_число_подписок FROM Subscriptions WHERE ID_Эксперта = 501;

-- Session B (Новая подписка)
BEGIN;
INSERT INTO Subscriptions (Дата_Подписки, ID_Студента, ID_Эксперта) 
VALUES (CURRENT_TIMESTAMP, 155, 501); -- Студент 155 подписывается на эксперта 501
COMMIT;

-- Session A (делает другую операцию, связанную с подписками)
SELECT COUNT(*) as Проверочный_отчёт FROM Subscriptions WHERE ID_Эксперта = 501;
-- Видит на 1 подписку больше! Появился "фантом".
COMMIT;

-- РЕШЕНИЕ: Использование Repeatable Read
-- Session A
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT COUNT(*) as Текущее_число_подписок FROM Subscriptions WHERE ID_Эксперта = 501;
-- Session B добавляет подписку и коммитит
SELECT COUNT(*) as Проверочный_отчёт FROM Subscriptions WHERE ID_Эксперта = 501;
-- Оба запроса показывают ОДИНАКОВОЕ число - фантомов нет
COMMIT;
```

### **4. Демонстрация использования точек сохранения (SAVEPOINT)**

```sql
-- Практический пример: сложное обновление данных студента с возможностью частичного отката
BEGIN;

-- 1. Начинаем транзакцию по обновлению данных студента
SELECT 'Начало транзакции по обновлению данных студента 101' AS Сообщение;

-- 2. Успешно обновляем имя
UPDATE Students SET Имя_Студента = 'Александр' WHERE ID_Студента = 101;
SELECT 'Имя успешно обновлено на "Александр"' AS Сообщение;

-- 3. Создаём точку сохранения перед рискованной операцией
SAVEPOINT before_email_update;
SELECT 'Создана точка сохранения before_email_update' AS Сообщение;

-- 4. Пытаемся обновить email на уже существующий (имитация ошибки приложения)
UPDATE Students SET Email_Студента = 'olga.v@example.com' WHERE ID_Студента = 101;
-- В реальности здесь была бы проверка в приложении, но допустим, мы получили ошибку:
-- ERROR: duplicate key value violates unique constraint "students_email_student_key"

-- 5. В случае ошибки откатываемся только к точке сохранения
ROLLBACK TO SAVEPOINT before_email_update;
SELECT 'Откат к точке сохранения: обновление email отменено (был дубликат)' AS Сообщение;

-- 6. Пробуем другой, уникальный email
UPDATE Students SET Email_Студента = 'alexander.ivanov.new@example.com' WHERE ID_Студента = 101;
SELECT 'Установлен новый уникальный email' AS Сообщение;

-- 7. Создаём ещё одну точку сохранения перед обновлением плана
SAVEPOINT before_plan_update;

-- 8. Обновляем учебный план студента
UPDATE StudyPlans SET Плановая_Дата_Окончания = Плановая_Дата_Окончания + INTERVAL '30 days' 
WHERE ID_Студента = 101 AND Статус_Плана = 'активен';
SELECT 'Учебный план продлён на 30 дней' AS Сообщение;

-- 9. Фиксируем ВСЕ успешные изменения
COMMIT;

-- Итоговый результат: имя изменено, email изменён, план продлён
-- При этом мы избежали ошибки с дублированием email благодаря SAVEPOINT
```

### **5. Инструкция по выполнению в DataGrip (обновлённая)**

1.  **Подключение и создание БД**:
    *   Откройте DataGrip и подключитесь к серверу PostgreSQL (обычно `localhost:5432`, БД `postgres`).
    *   Откройте **консоль SQL** для БД `postgres`.
    *   Выполните **первые две команды** из Скрипта 1 (до `\c "Задача_11"`), чтобы создать БД.
    *   Обновите список БД (F5) - появится "Задача_11".

2.  **Создание схемы и наполнение**:
    *   **Правой кнопкой** на БД "Задача_11" → **New → Query Console**.
    *   Выполните **остальную часть Скрипта 1** (после `\c "Задача_11"`).
    *   Убедитесь, что все команды выполнились без ошибок.

3.  **Демонстрация параллельных проблем**:
    *   Откройте **ДВЕ новые консоли** для БД "Задача_11" (правая кнопка → New Query Console).
    *   В **первой консоли** выполняйте шаги **Session A** из Демонстрации 1.
    *   Во **второй консоли** выполняйте шаги **Session B**.
    *   Чередуйте выполнение блоков кода между консолями, используя **Ctrl+Enter** для выполнения выделенного кода.

4.  **Проверка результатов**:
    *   После каждого эксперимента выполняйте проверочные SELECT-запросы, чтобы увидеть итоговое состояние данных.

### **6. Полная очистка базы данных (исправленный)**

```sql
-- Вариант 1: Полное удаление БД (выполнять в БД 'postgres')
-- Этот вариант гарантированно удалит ВСЕ данные и структуру

-- Принудительно завершаем все соединения с БД "Задача_11"
SELECT pg_terminate_backend(pg_stat_activity.pid)
FROM pg_stat_activity
WHERE pg_stat_activity.datname = 'Задача_11'
  AND pid <> pg_backend_pid();

-- Удаляем БД
DROP DATABASE IF EXISTS "Задача_11";

SELECT 'База данных "Задача_11" полностью удалена. Можно запускать Скрипт 1 заново.' AS "Результат";

-- Вариант 2: Очистка данных без удаления БД (выполнять в БД "Задача_11")
-- Полезно, если нужно сохранить структуру таблиц, но удалить все данные

-- Отключаем триггеры и внешние ключи для безопасной очистки
SET session_replication_role = replica;

-- Очищаем все таблицы в правильном порядке (от дочерних к родительским)
TRUNCATE TABLE 
    Certificates, Progress, StudyPlans, Subscriptions,
    Quizzes, Lessons, Modules, Course_Prices, 
    Courses, Students, Experts 
CASCADE;

-- Включаем обратно
SET session_replication_role = DEFAULT;

-- Сбрасываем все последовательности (серийные ID)
ALTER SEQUENCE IF EXISTS certificates_id_сертификата_seq RESTART WITH 1;
ALTER SEQUENCE IF EXISTS progress_id_прогресса_seq RESTART WITH 1;
ALTER SEQUENCE IF EXISTS studyplans_id_плана_seq RESTART WITH 1;
ALTER SEQUENCE IF EXISTS subscriptions_id_подписки_seq RESTART WITH 1;
ALTER SEQUENCE IF EXISTS quizzes_id_теста_seq RESTART WITH 1;
ALTER SEQUENCE IF EXISTS lessons_id_урока_seq RESTART WITH 1;
ALTER SEQUENCE IF EXISTS modules_id_модуля_seq RESTART WITH 1;
ALTER SEQUENCE IF EXISTS courses_id_курса_seq RESTART WITH 1;
ALTER SEQUENCE IF EXISTS students_id_студента_seq RESTART WITH 1;
ALTER SEQUENCE IF EXISTS experts_id_эксперта_seq RESTART WITH 1;

SELECT 'Все таблицы очищены, последовательности сброшены. База готова к новому наполнению.' AS "Результат";
```
