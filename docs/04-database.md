# 04. Схема базы данных и интеграции с CSV/API (Database Schema)

Документ содержит описание архитектуры БД PostgreSQL для FITZONE: собственные таблицы системы (Команда, Услуги) и 4 конфигурационные таблицы для работы с ценовым API.

---

## 1. Наша локальная БД PostgreSQL (Собственные данные)

### 1.1 Таблица Команды (team_members)
Хранит список тренеров, специалистов и персонала (источник — БД_команда.csv).

from typing import List, Optional
from sqlalchemy import Boolean, ForeignKey, Integer, Numeric, String, Text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class TeamMember(Base):
    __tablename__ = "team_members"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    legacy_wix_id: Mapped[Optional[str]] = mapped_column(
        String(100), index=True
    )  # ID из Wix
    full_name: Mapped[str] = mapped_column(String(255), nullable=False)
    photo_url: Mapped[Optional[str]] = mapped_column(String(500))
    qualification: Mapped[Optional[str]] = mapped_column(
        String(50)
    )  # Classic, PRO, VIP, Авторський
    description: Mapped[Optional[str]] = mapped_column(
        Text
    )  # Биография / Описание
    price_private: Mapped[Optional[float]] = mapped_column(
        Numeric(10, 2)
    )  # Индивидуальная цена
    instagram_url: Mapped[Optional[str]] = mapped_column(String(255))
    is_published: Mapped[bool] = mapped_column(Boolean, default=True, index=True)
    display_order: Mapped[int] = mapped_column(Integer, default=0)

    # Связь M2M с категориями
    categories: Mapped[List["TrainerCategory"]] = relationship(
        back_populates="trainers", secondary="trainer_category_m2m"
    )

### 1.2 Связная таблица категорий тренера (trainer_categories)
Поскольку тренер может принадлежать одновременно к нескольким категориям (например, "фитнес" и "реабилитация").

class TrainerCategory(Base):
    __tablename__ = "trainer_categories"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(
        String(100), unique=True
    )  # тренажерний-зал, масажисти, реабілітація


class TrainerCategoryM2M(Base):
    __tablename__ = "trainer_category_m2m"

    trainer_id: Mapped[int] = mapped_column(
        ForeignKey("team_members.id", ondelete="CASCADE"), primary_key=True
    )
    category_id: Mapped[int] = mapped_column(
        ForeignKey("trainer_categories.id", ondelete="CASCADE"), primary_key=True
    )

### 1.3 Таблица Групповых Услуг (group_training_services)
Хранит каталог всех групповых и индивидуальных программ (источник — БД_ПОСЛУГИ.csv).

class GroupTrainingService(Base):
    __tablename__ = "group_training_services"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    legacy_wix_id: Mapped[Optional[str]] = mapped_column(String(100), index=True)
    title: Mapped[str] = mapped_column(String(255), nullable=False)
    category: Mapped[str] = mapped_column(
        String(100)
    )  # ДИНАМІЧНЕ / СПОКІЙНЕ / ДИТЯЧІ
    description: Mapped[Optional[str]] = mapped_column(Text)
    video_url: Mapped[Optional[str]] = mapped_column(String(500))
    age_label: Mapped[Optional[str]] = mapped_column(
        String(100)
    )  # "18+", "6-10 років"
    subscription_type: Mapped[Optional[str]] = mapped_column(
        String(100)
    )  # ЗАГАЛЬНИЙ / АВТОРСЬКИЙ / РАЗОВА
    is_published: Mapped[bool] = mapped_column(Boolean, default=True)

---

## 2. Конфигурационные таблицы Конструктора Цен (4 таблицы)

Эти таблицы заполняются из CSV-конфигов (PriceGroupsConfig, PricePlansConfig, PriceRulesConfig, PriceNamesConfig) и служат правилами фильтрации и переопределения для внешнего API цен.

### 2.1 Группы цен (price_groups_config)

class PriceGroupConfig(Base):
    __tablename__ = "price_groups_config"

    slug: Mapped[str] = mapped_column(
        String(50), primary_key=True
    )  # gym-no-trainer, massage, etc.
    display_name: Mapped[str] = mapped_column(String(255))
    sort_order: Mapped[int] = mapped_column(Integer, default=0)
    filter_mode: Mapped[str] = mapped_column(
        String(50)
    )  # srd / category / none
    source_idgrs: Mapped[Optional[str]] = mapped_column(
        String(255)
    )  # Список external_id через запятую

### 2.2 Тарифные Планы (price_plans_config)

class PricePlanConfig(Base):
    __tablename__ = "price_plans_config"

    id: Mapped[str] = mapped_column(
        String(100), primary_key=True
    )  # spab.id из внешнего API
    filter_value: Mapped[str] = mapped_column(
        String(100)
    )  # "Жіночий", "CLASSIC", "PRO"
    sort_order: Mapped[int] = mapped_column(Integer, default=0)

### 2.3 Правила и Плашки Инфо (price_rules_config)

class PriceRuleConfig(Base):
    __tablename__ = "price_rules_config"

    id: Mapped[str] = mapped_column(
        String(100), primary_key=True
    )  # "2_30", "massage_default"
    source_idgr: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)
    filter_value: Mapped[str] = mapped_column(String(100))
    rules_html: Mapped[Optional[str]] = mapped_column(
        Text
    )  # Информационный HTML-текст

### 2.4 Переопределение Названий (price_names_config)

class PriceNameConfig(Base):
    __tablename__ = "price_names_config"

    id: Mapped[str] = mapped_column(
        String(100), primary_key=True
    )  # _id из внешнего API
    display_name: Mapped[Optional[str]] = mapped_column(
        String(255), nullable=True
    )
    sort_order: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)

---

## 3. Кэширование Расписания (schedule_cache)

Служебная таблица для хранения расписания групповых занятий, запрашиваемого из API servpost.php.

from datetime import datetime
from sqlalchemy import JSON, Date, DateTime


class ScheduleCache(Base):
    __tablename__ = "schedule_cache"

    cache_key: Mapped[str] = mapped_column(
        String(50), primary_key=True
    )  # 'current' или 'next'
    week_data: Mapped[dict] = mapped_column(JSON)  # JSON-ответ 7 дней
    monday_date: Mapped[datetime] = mapped_column(Date)
    updated_at: Mapped[datetime] = mapped_column(
        DateTime, default=datetime.utcnow
    )