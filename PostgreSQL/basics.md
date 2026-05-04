## ER Diagram

Entity -> Attribute -> Relationship

### Entity

- Is the table in DB (for example, Student, Course, etc.)

### Attribute

- Is the column in the table (for example, StudentID, StudentName, etc.)

### Relationship

- Is the connection between two entities (for example, Student enrolls in Course)

ENTITIES and ATTRIBUTES

Branch

- branch_id int (primary key)
- name varchar(255)
- state varchar(255)
- city varchar(255) (Texas, New York, etc.)
- phone_number varchar(255)
- created_at timestamp
- updated_at timestamp

Member

- member_id int (primary key)
- branch_id int (foreign key)
- name varchar(255)
- email varchar(255)
- phone_number varchar(255)
- status
- dob date
- joined_at timestamp
- left_at timestamp

Trainer

- trainer_id int (primary key)
- branch_id int (foreign key)
- name varchar(255)
- email varchar(255)
- phone_number varchar(255)
- specialization varchar(255)
- created_at timestamp
- updated_at timestamp

MembershipPlan (the template — no member_id!)

- plan_id (PK)
- name (e.g., "Gold Monthly")
- type ENUM (monthly, yearly)
- price decimal(10,2)
- duration_days int
- created_at timestamp
- updated_at timestamp

Membership (the subscription instance — this is the join table)

- membership_id (PK)
- member_id (FK → Member)
- plan_id (FK → MembershipPlan)
- start_date
- end_date
- status (active, expired, cancelled)
- created_at timestamp
- updated_at timestamp

Class

- class_id int (primary key)
- branch_id int (foreign key)
- name varchar(255)
- schedule varchar(255)
- created_at timestamp
- updated_at timestamp

Equipment

- equipment_id int (primary key)
- branch_id int (foreign key)
- name varchar(255)
- type varchar(255)
- quantity int
- created_at timestamp
- updated_at timestamp

Payment

- payment_id int (primary key)
- member_id int (foreign key)
- branch_id int (foreign key) // <- to track revenues by branch>
- personal_training_session_id int (foreign key, nullable)
- amount decimal(10, 2)
- date date
- paid_at time
- created_at timestamp
- updated_at timestamp

PersonalTrainingSession

- session_id (PK)
- trainer_id (FK)
- member_id (FK)
- mode ENUM ('in_person', 'virtual')
- branch_id (FK, nullable) ← only filled for in_person
- meeting_link varchar(500) (nullable) ← only filled for virtual
- platform varchar(50) (nullable) ← only filled for virtual (zoom, meet, etc.)
- session_date timestamp
- duration int
- status varchar (scheduled, completed, cancelled, no_show)

ClassBooking

- booking_id int (primary key)
- member_id int (foreign key)
- class_id int (foreign key)
- date date
- time time
- status varchar(255)
- created_at timestamp
- updated_at timestamp

RELATIONSHIP

- Branch -> One has many -> Member
- Member -> Many has Many -> Class
- Trainer -> One has Many -> Specialization
- Trainer -> One has Many -> PersionalTrainingSession
- Member -> One has Many -> PersonalTrainingSession
- Member -> Many has Many -> MembershipPlan
- Branch -> One has Many - Equipment
- Member -> One has Many -> Payment
- Class -> Many has Many -> Member (Join Table ClassBooking)


TABLE CREATION QUERIES for above ER DIAGRAM

```sql
-- ============================================================
-- GYM MANAGEMENT SYSTEM — TABLE CREATION
-- PostgreSQL
-- ============================================================

-- ENUMS
CREATE TYPE membership_type     AS ENUM ('monthly', 'yearly');
CREATE TYPE membership_status   AS ENUM ('active', 'expired', 'cancelled');
CREATE TYPE session_mode        AS ENUM ('in_person', 'virtual');
CREATE TYPE session_status      AS ENUM ('scheduled', 'completed', 'cancelled', 'no_show');
CREATE TYPE member_status       AS ENUM ('active', 'inactive', 'suspended');

-- ============================================================
-- 1. BRANCH
-- ============================================================
CREATE TABLE Branch (
    branch_id       SERIAL          PRIMARY KEY,
    name            VARCHAR(255)    NOT NULL,
    state           VARCHAR(255)    NOT NULL,
    city            VARCHAR(255)    NOT NULL,
    phone_number    VARCHAR(20),
    created_at      TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP       NOT NULL DEFAULT NOW()
);

-- ============================================================
-- 2. MEMBER
-- ============================================================
CREATE TABLE Member (
    member_id       SERIAL          PRIMARY KEY,
    branch_id       INT             NOT NULL REFERENCES Branch(branch_id) ON DELETE RESTRICT,
    name            VARCHAR(255)    NOT NULL,
    email           VARCHAR(255)    NOT NULL UNIQUE,
    phone_number    VARCHAR(20),
    status          member_status   NOT NULL DEFAULT 'active',
    dob             DATE,
    joined_at       TIMESTAMP       NOT NULL DEFAULT NOW(),
    left_at         TIMESTAMP
);

-- ============================================================
-- 3. TRAINER
-- ============================================================
CREATE TABLE Trainer (
    trainer_id      SERIAL          PRIMARY KEY,
    branch_id       INT             NOT NULL REFERENCES Branch(branch_id) ON DELETE RESTRICT,
    name            VARCHAR(255)    NOT NULL,
    email           VARCHAR(255)    NOT NULL UNIQUE,
    phone_number    VARCHAR(20),
    specialization  VARCHAR(255),
    created_at      TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP       NOT NULL DEFAULT NOW()
);

-- ============================================================
-- 4. MEMBERSHIP PLAN  (template — no member link here)
-- ============================================================
CREATE TABLE MembershipPlan (
    plan_id         SERIAL              PRIMARY KEY,
    name            VARCHAR(255)        NOT NULL,
    type            membership_type     NOT NULL,
    price           DECIMAL(10, 2)      NOT NULL CHECK (price >= 0),
    duration_days   INT                 NOT NULL CHECK (duration_days > 0),
    created_at      TIMESTAMP           NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP           NOT NULL DEFAULT NOW()
);

-- ============================================================
-- 5. MEMBERSHIP  (subscription instance — Member ↔ Plan join)
-- ============================================================
CREATE TABLE Membership (
    membership_id   SERIAL              PRIMARY KEY,
    member_id       INT                 NOT NULL REFERENCES Member(member_id) ON DELETE CASCADE,
    plan_id         INT                 NOT NULL REFERENCES MembershipPlan(plan_id) ON DELETE RESTRICT,
    start_date      DATE                NOT NULL,
    end_date        DATE                NOT NULL,
    status          membership_status   NOT NULL DEFAULT 'active',
    created_at      TIMESTAMP           NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP           NOT NULL DEFAULT NOW(),

    CONSTRAINT chk_membership_dates CHECK (end_date > start_date)
);

-- ============================================================
-- 6. CLASS
-- ============================================================
CREATE TABLE Class (
    class_id        SERIAL          PRIMARY KEY,
    branch_id       INT             NOT NULL REFERENCES Branch(branch_id) ON DELETE RESTRICT,
    name            VARCHAR(255)    NOT NULL,
    schedule        VARCHAR(255),
    created_at      TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP       NOT NULL DEFAULT NOW()
);

-- ============================================================
-- 7. EQUIPMENT
-- ============================================================
CREATE TABLE Equipment (
    equipment_id    SERIAL          PRIMARY KEY,
    branch_id       INT             NOT NULL REFERENCES Branch(branch_id) ON DELETE RESTRICT,
    name            VARCHAR(255)    NOT NULL,
    type            VARCHAR(255),
    quantity        INT             NOT NULL DEFAULT 1 CHECK (quantity >= 0),
    created_at      TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP       NOT NULL DEFAULT NOW()
);

-- ============================================================
-- 8. PERSONAL TRAINING SESSION
-- (declared before Payment because Payment references it)
-- ============================================================
CREATE TABLE PersonalTrainingSession (
    session_id      SERIAL          PRIMARY KEY,
    trainer_id      INT             NOT NULL REFERENCES Trainer(trainer_id) ON DELETE RESTRICT,
    member_id       INT             NOT NULL REFERENCES Member(member_id) ON DELETE CASCADE,
    mode            session_mode    NOT NULL,
    branch_id       INT             REFERENCES Branch(branch_id) ON DELETE SET NULL,   -- nullable; in_person only
    meeting_link    VARCHAR(500),   -- nullable; virtual only
    platform        VARCHAR(50),    -- nullable; virtual only (zoom, meet, etc.)
    session_date    TIMESTAMP       NOT NULL,
    duration        INT             NOT NULL CHECK (duration > 0),                      -- in minutes
    status          session_status  NOT NULL DEFAULT 'scheduled',

    -- Guard: in_person must have branch_id; virtual must have meeting_link
    CONSTRAINT chk_in_person_branch     CHECK (mode <> 'in_person' OR branch_id IS NOT NULL),
    CONSTRAINT chk_virtual_meeting_link CHECK (mode <> 'virtual'   OR meeting_link IS NOT NULL)
);

-- ============================================================
-- 9. PAYMENT
-- ============================================================
CREATE TABLE Payment (
    payment_id                  SERIAL          PRIMARY KEY,
    member_id                   INT             NOT NULL REFERENCES Member(member_id) ON DELETE RESTRICT,
    branch_id                   INT             NOT NULL REFERENCES Branch(branch_id) ON DELETE RESTRICT,
    personal_training_session_id INT            REFERENCES PersonalTrainingSession(session_id) ON DELETE SET NULL,
    amount                      DECIMAL(10, 2)  NOT NULL CHECK (amount >= 0),
    date                        DATE            NOT NULL,
    paid_at                     TIME,
    created_at                  TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at                  TIMESTAMP       NOT NULL DEFAULT NOW()
);

-- ============================================================
-- 10. CLASS BOOKING  (Member ↔ Class join table)
-- ============================================================
CREATE TABLE ClassBooking (
    booking_id      SERIAL          PRIMARY KEY,
    member_id       INT             NOT NULL REFERENCES Member(member_id) ON DELETE CASCADE,
    class_id        INT             NOT NULL REFERENCES Class(class_id) ON DELETE CASCADE,
    date            DATE            NOT NULL,
    time            TIME            NOT NULL,
    status          VARCHAR(50)     NOT NULL DEFAULT 'confirmed',
    created_at      TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP       NOT NULL DEFAULT NOW(),

    -- Prevent double-booking the same member into the same class slot
    CONSTRAINT uq_member_class_slot UNIQUE (member_id, class_id, date, time)
);

-- ============================================================
-- INDEXES  (high-value lookups)
-- ============================================================

-- Member lookups by branch
CREATE INDEX idx_member_branch       ON Member(branch_id);

-- Active memberships lookup
CREATE INDEX idx_membership_member   ON Membership(member_id);
CREATE INDEX idx_membership_status   ON Membership(status);

-- Sessions by trainer / member
CREATE INDEX idx_pts_trainer         ON PersonalTrainingSession(trainer_id);
CREATE INDEX idx_pts_member          ON PersonalTrainingSession(member_id);
CREATE INDEX idx_pts_date            ON PersonalTrainingSession(session_date);

-- Payments by branch (revenue reports) and member
CREATE INDEX idx_payment_branch      ON Payment(branch_id);
CREATE INDEX idx_payment_member      ON Payment(member_id);
CREATE INDEX idx_payment_date        ON Payment(date);

-- Class bookings
CREATE INDEX idx_booking_class       ON ClassBooking(class_id);
CREATE INDEX idx_booking_member      ON ClassBooking(member_id);

-- Equipment by branch
CREATE INDEX idx_equipment_branch    ON Equipment(branch_id);

```