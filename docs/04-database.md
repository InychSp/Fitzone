# 04. Схема бази даних (Database Schema)

Повна схема PostgreSQL для Fitzone: власні таблиці (тренери, послуги),
4 конфігураційні таблиці прайсу, кеш розкладу, адмінка з ролями,
доступ студентів Школи інструктора і єдиний інбокс месенджерів.
ORM — SQLAlchemy 2.0 (стиль `Mapped`/`mapped_column`).

> **Що виправлено відносно попередньої версії файлу:**
> - код обгорнуто в ```` ```python ```` блоки (раніше рендерився як
>   звичайний текст без форматування);
> - додано таблицю `trainer_services` (M2M тренер↔послуга) — без неї
>   не зібрати фільтр "Тренер" на групових тренуваннях;
> - виправлено баг зв'язку `TeamMember.categories` ↔
>   `TrainerCategory.trainers` (`back_populates` вказував на
>   неіснуючий атрибут — застосунок падав би при старті);
> - `age_label`: `String(100)` → `Text` (реальні дані не влазять у
>   100 символів, є багаторядкові приклади);
> - `source_idgrs`: рядок через кому → нативний `ARRAY(Integer)`
>   Postgres;
> - категорійні поля (`qualification`, `category`, `filter_mode`,
>   `subscription_type` та інші) переведені на `enum.Enum` +
>   SQLAlchemy `Enum` — це на рівні БД відсікає розсинхрон регістру
>   на кшталт `"фітнес"` / `"ФІТНЕС"`, знайдений у сирих CSV;
> - додано таблиці, яких не вистачало: `admin_users` (ролі
>   Admin/Reception), `students` (доступ Школи інструктора),
>   `inbox_conversations` + `inbox_messages` (єдиний інбокс
>   месенджерів, нова підтверджена вимога — `01-project.md`,
>   `03-architecture.md`).

---

## Спільний імпорт і базовий клас

```python
import enum
from datetime import date, datetime
from typing import List, Optional

from sqlalchemy import Boolean, Date, DateTime, Enum, ForeignKey, Integer, Numeric, String, Text
from sqlalchemy.dialects.postgresql import ARRAY, JSONB
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass
```

---

## 1. Тренери (`team_members`)

Джерело — `БД_команда.csv`. Один тренер може належати одразу до
кількох категорій (у сирих даних `kategori` — вже JSON-масив), тому
категорії — окрема M2M-таблиця, а не одне поле.

```python
class TrainerQualification(str, enum.Enum):
    CLASSIC = "Classic"
    PRO = "PRO"
    PRO_PLUS = "PRO+"
    VIP = "VIP"
    VIP_PLUS = "VIP+"
    AUTHOR = "Авторський"


class TeamMember(Base):
    __tablename__ = "team_members"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    legacy_wix_id: Mapped[Optional[str]] = mapped_column(String(100), index=True)
    full_name: Mapped[str] = mapped_column(String(255), nullable=False)
    photo_url: Mapped[Optional[str]] = mapped_column(Text)
    qualification: Mapped[Optional[TrainerQualification]] = mapped_column(
        Enum(TrainerQualification, name="trainer_qualification"), nullable=True
    )
    description: Mapped[Optional[str]] = mapped_column(Text)  # багатий HTML-текст (opis)
    price_private: Mapped[Optional[float]] = mapped_column(Numeric(10, 2))
    # ⚠️ у поточному експорті CSV порожнє в усіх 35 рядках — власник
    # ще не заповнив. Без цих даних кнопка "Показати ціну" на сторінці
    # Реабілітація не запрацює на день релізу.
    instagram_url: Mapped[Optional[str]] = mapped_column(String(255))
    link_label: Mapped[Optional[str]] = mapped_column(String(255))  # підпис кнопки-посилання
    is_published: Mapped[bool] = mapped_column(Boolean, default=True, index=True)
    display_order: Mapped[int] = mapped_column(Integer, default=0)

    categories: Mapped[List["TrainerCategory"]] = relationship(
        secondary="trainer_category_m2m", back_populates="trainers"
    )
    services: Mapped[List["GroupTrainingService"]] = relationship(
        secondary="trainer_services", back_populates="trainers"
    )
```

### 1.1. Категорії тренера

```python
class TrainerCategoryName(str, enum.Enum):
    GYM = "тренажерний-зал"
    FITNESS = "фітнес"
    KIDS = "дитячі"
    MASSAGE = "масажисти"
    REHAB = "реабілітація"


class TrainerCategory(Base):
    __tablename__ = "trainer_categories"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    name: Mapped[TrainerCategoryName] = mapped_column(
        Enum(TrainerCategoryName, name="trainer_category_name"), unique=True
    )

    trainers: Mapped[List["TeamMember"]] = relationship(
        secondary="trainer_category_m2m", back_populates="categories"
    )


class TrainerCategoryM2M(Base):
    __tablename__ = "trainer_category_m2m"

    trainer_id: Mapped[int] = mapped_column(
        ForeignKey("team_members.id", ondelete="CASCADE"), primary_key=True
    )
    category_id: Mapped[int] = mapped_column(
        ForeignKey("trainer_categories.id", ondelete="CASCADE"), primary_key=True
    )
```

---

## 2. Групові тренування (`group_training_services`)

Джерело — `БД_ПОСЛУГИ.csv`. Стовпець `Description` з вихідного CSV
**не переносимо** — він порожній у всіх 27 рядках, це невикористовуване
службове поле Wix; реальний опис лежить у стовпці `опис послуги`.

```python
class ServiceCategory(str, enum.Enum):
    DYNAMIC = "ДИНАМІЧНЕ"
    CALM = "СПОКІЙНЕ"
    KIDS = "ДИТЯЧІ"


class SubscriptionType(str, enum.Enum):
    GENERAL = "ЗАГАЛЬНИЙ АБОНЕМЕНТ"
    AUTHOR = "АВТОРСЬКИЙ АБОНЕМЕНТ"
    ONE_TIME = "РАЗОВА ОПЛАТА"


class GroupTrainingService(Base):
    __tablename__ = "group_training_services"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    legacy_wix_id: Mapped[Optional[str]] = mapped_column(String(100), index=True)
    title: Mapped[str] = mapped_column(String(255), nullable=False)
    category: Mapped[ServiceCategory] = mapped_column(Enum(ServiceCategory, name="service_category"))
    description: Mapped[Optional[str]] = mapped_column(Text)
    video_url: Mapped[Optional[str]] = mapped_column(Text)
    age_label: Mapped[Optional[str]] = mapped_column(Text)
    # виправлено: було String(100). Реальні дані — вільний текст,
    # інколи багаторядковий з кількома віковими групами в одній
    # послузі (приклад — POLE SPORT: три групи з різним віком в
    # одному полі), у 100 символів це не влазить.
    subscription_type: Mapped[Optional[SubscriptionType]] = mapped_column(
        Enum(SubscriptionType, name="subscription_type"), nullable=True
    )
    is_published: Mapped[bool] = mapped_column(Boolean, default=True)

    trainers: Mapped[List["TeamMember"]] = relationship(
        secondary="trainer_services", back_populates="services"
    )
```

### 2.1. Зв'язок тренер ↔ послуга (`trainer_services`)

```python
class TrainerService(Base):
    """M2M тренер <-> групове тренування. У попередній версії схеми
    цієї таблиці не було, хоча вона потрібна для фільтра "Тренер" на
    /services/group-training і для карток тренерів на сторінках
    послуг."""

    __tablename__ = "trainer_services"

    trainer_id: Mapped[int] = mapped_column(
        ForeignKey("team_members.id", ondelete="CASCADE"), primary_key=True
    )
    service_id: Mapped[int] = mapped_column(
        ForeignKey("group_training_services.id", ondelete="CASCADE"), primary_key=True
    )
```

> ⚠️ **Важливо для міграції даних (не для коду).** У жодному з трьох
> наявних джерел цього зв'язку в сирих CSV довіряти не можна:
> `БД_команда.Множественная ссылка` заповнена лише в 1 з 35 тренерів;
> `БД_ПОСЛУГИ.послуга тренера` заповнена лише в 3 з 27 рядків і при
> цьому в усіх трьох показує не того тренера, що вказаний у
> текстовому полі `Тренер` поруч. Реальний зв'язок доведеться зібрати
> вручну (зіставити текст `Тренер` з іменами з `БД_команда`, спірні
> випадки — на перевірку власнику), сама ж таблиця `trainer_services`
> у коді від цього не залежить.

---

## 3. Конфігураційні таблиці прайсу (4 таблиці)

Джерело — `PriceGroupsConfig`, `PricePlansConfig`, `PriceRulesConfig`,
`PriceNamesConfig` (експортовані Wix-таблиці власника). Детальна
логіка використання кожної — `05-api.md`, розділ 5.

```python
class PriceFilterMode(str, enum.Enum):
    SRD = "srd"
    CATEGORY = "category"
    NONE = "none"


class PriceGroupConfig(Base):
    __tablename__ = "price_groups_config"

    slug: Mapped[str] = mapped_column(String(50), primary_key=True)  # gym-no-trainer, massage...
    display_name: Mapped[str] = mapped_column(String(255))
    sort_order: Mapped[int] = mapped_column(Integer, default=0)
    filter_mode: Mapped[PriceFilterMode] = mapped_column(Enum(PriceFilterMode, name="price_filter_mode"))
    source_idgrs: Mapped[List[int]] = mapped_column(ARRAY(Integer))
    # виправлено: було String(255) через кому. ARRAY(Integer) — рідний
    # тип Postgres: фільтрація через "WHERE :idgr = ANY(source_idgrs)"
    # замість парсингу рядка на боці Python.


class PricePlanConfig(Base):
    __tablename__ = "price_plans_config"

    id: Mapped[str] = mapped_column(String(100), primary_key=True)  # spab.id із зовнішнього API
    filter_value: Mapped[str] = mapped_column(String(100))  # "Жіночий", "CLASSIC" тощо
    sort_order: Mapped[int] = mapped_column(Integer, default=0)
    # display_name НЕ переносимо: підтверджено фінальним кодом Wix
    # (05-api.md, розд. 7.5), що поле існує в схемі, але ніде не
    # використовується для показу назви картки.


class PriceRuleConfig(Base):
    __tablename__ = "price_rules_config"

    id: Mapped[str] = mapped_column(String(100), primary_key=True)  # "2_30", "massage_default"
    source_idgr: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)  # NULL у personal_default
    filter_value: Mapped[str] = mapped_column(String(100))  # "30"/"90"/"365" або "default"
    rules_html: Mapped[Optional[str]] = mapped_column(Text)


class PriceNameConfig(Base):
    __tablename__ = "price_names_config"

    id: Mapped[str] = mapped_column(String(100), primary_key=True)  # _id = spab.id з API
    original_nameapp: Mapped[Optional[str]] = mapped_column(String(255), nullable=True)
    # довідково/аудит — що саме прийшло з API на момент override
    display_name: Mapped[Optional[str]] = mapped_column(String(255), nullable=True)
    # NULL — можливий і нормальний випадок (17 із 91 рядка в
    # реальному експорті): рядок конфігу існує лише заради
    # sort_order, назву в такому разі беремо з spab.nameapp/name
    # (05-api.md, розд. 5.4)
    sort_order: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)
```

---

## 4. Кеш розкладу (`schedule_cache`)

```python
class ScheduleCache(Base):
    __tablename__ = "schedule_cache"

    cache_key: Mapped[str] = mapped_column(String(50), primary_key=True)  # 'current' | 'next'
    week_data: Mapped[dict] = mapped_column(JSONB)  # усі 7 днів серіалізовано
    monday_date: Mapped[date] = mapped_column(Date)
    updated_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```

Тижні за межами `current`/`next` не кешуються — читаються з API наживо
(рішення зафіксовано в `01-project.md`).

---

## 5. Адмінка: користувачі й ролі (`admin_users`)

Дві ролі: **Admin** (повний доступ) і **Reception** (лише єдиний
інбокс). Повний опис прав — `06-admin.md`.

```python
class AdminRole(str, enum.Enum):
    ADMIN = "admin"
    RECEPTION = "reception"


class AdminUser(Base):
    __tablename__ = "admin_users"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False)
    password_hash: Mapped[str] = mapped_column(String(255), nullable=False)
    role: Mapped[AdminRole] = mapped_column(Enum(AdminRole, name="admin_role"))
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```

---

## 6. Школа інструктора: доступ студентів (`students`)

За описом власника: в адмінці є сторінка "Теорія" зі своїм списком
email, кому відкрито доступ, і окремо сторінка "Практика" з власним
списком email (один і той самий email може бути в обох списках або
лише в одному). Технічно це рівнозначно двом булевим прапорцям на
одному записі — простіше за дві окремі таблиці зв'язків, а в адмінці
однаково реалізується як два екрани-фільтри за цими прапорцями.

```python
class Student(Base):
    __tablename__ = "students"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False)
    has_theory_access: Mapped[bool] = mapped_column(Boolean, default=False)
    has_practice_access: Mapped[bool] = mapped_column(Boolean, default=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    invited_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

    # Поля нижче — під рекомендований, але поки НЕ підтверджений
    # варіант входу "магічним посиланням на email" (без пароля).
    # Якщо власник захоче звичайний логін/пароль — ці два поля не
    # знадобляться, а натомість додасться password_hash.
    login_token: Mapped[Optional[str]] = mapped_column(String(255), nullable=True)
    login_token_expires_at: Mapped[Optional[datetime]] = mapped_column(DateTime, nullable=True)
```

> 🟡 **Відкрите питання (лишається з `pages/10-school-instructor/`):**
> сам механізм входу студента (магічне посилання / логін-пароль / код
> на пошту) остаточно не підтверджено — вище закладено поля під
> магічне посилання як рекомендований дефолт, легко змінити.

---

## 7. Єдиний інбокс месенджерів (`inbox_conversations`, `inbox_messages`)

Нова підтверджена підсистема (`01-project.md`, `03-architecture.md`,
розд. 5). Без прив'язки до внутрішньої бази клієнтів клубу — це
агрегатор чужих месенджерів, не CRM.

```python
class InboxChannel(str, enum.Enum):
    TELEGRAM = "telegram"
    VIBER = "viber"
    INSTAGRAM = "instagram"


class ConversationStatus(str, enum.Enum):
    NEW = "new"
    IN_PROGRESS = "in_progress"
    DONE = "done"


class InboxConversation(Base):
    __tablename__ = "inbox_conversations"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    channel: Mapped[InboxChannel] = mapped_column(Enum(InboxChannel, name="inbox_channel"))
    external_contact_id: Mapped[str] = mapped_column(String(255))
    # ід контакту в самому месенджері (chat_id Telegram, user id Viber/Instagram)
    contact_name: Mapped[Optional[str]] = mapped_column(String(255), nullable=True)
    status: Mapped[ConversationStatus] = mapped_column(
        Enum(ConversationStatus, name="conversation_status"), default=ConversationStatus.NEW
    )
    last_message_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

    messages: Mapped[List["InboxMessage"]] = relationship(back_populates="conversation")


class MessageDirection(str, enum.Enum):
    IN = "in"    # від клієнта
    OUT = "out"  # від нас (адміністратора/ресепшена)


class InboxMessage(Base):
    __tablename__ = "inbox_messages"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    conversation_id: Mapped[int] = mapped_column(ForeignKey("inbox_conversations.id", ondelete="CASCADE"))
    direction: Mapped[MessageDirection] = mapped_column(Enum(MessageDirection, name="message_direction"))
    text: Mapped[str] = mapped_column(Text)
    sent_by_admin_id: Mapped[Optional[int]] = mapped_column(ForeignKey("admin_users.id"), nullable=True)
    # заповнено лише для direction = OUT — хто з персоналу відповів
    sent_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

    conversation: Mapped["InboxConversation"] = relationship(back_populates="messages")
```

> 🟡 Відкрите питання (нове, поза попереднім обговоренням): технічна
> інтеграція з боку кожної платформи — Telegram Bot API, Viber Bot
> API і Instagram Graph API — це три різні протоколи з різними
> вимогами (у Instagram, наприклад, для Graph API потрібен
> підключений Facebook Business-акаунт і проходження модерації
> Meta). Варто закласти час на це окремо в `09-deployment.md`, коли
> дійдемо до нього.

---

## 8. Загальна схема зв'язків (спрощено)

```
team_members ──M2M──> trainer_categories
team_members ──M2M──> group_training_services  (через trainer_services)

price_groups_config.source_idgrs[]  ──> зовнішній grab.id (API, не FK)
price_plans_config.id                ──> зовнішній spab.id (API, не FK)
price_names_config.id                ──> зовнішній spab.id (API, не FK)
price_rules_config.source_idgr       ──> зовнішній grab.id (API, не FK)

admin_users ──1:N──> inbox_messages (sent_by_admin_id, лише direction=OUT)
inbox_conversations ──1:N──> inbox_messages
```

`source_idgr(s)` і `spab.id` — це ідентифікатори з **чужої** бази
(сервер розробника застосунку), а не наші власні PK. Зберігаємо їх як
звичайні числа/рядки з коментарем "зовнішній ідентифікатор" —
референсна цілісність (FK) на рівні Postgres тут неможлива, бо
таблиця, на яку посилаємось, фізично не в нашій БД.
