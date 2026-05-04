# Module 16: Users, Roles, and Permissions

## Table of Contents
1. [Creating Roles/Users](#creating-rolesusers)
2. [GRANT and REVOKE](#grant-and-revoke)
3. [Common Privileges](#common-privileges)
4. [Schema-Level Privileges](#schema-level-privileges)
5. [Role Membership and Groups](#role-membership-and-groups)
6. [Owner vs Grantee](#owner-vs-grantee)
7. [Reading Permission Denied Errors](#reading-permission-denied-errors)
8. [Quick Reference](#quick-reference)

---

## Creating Roles/Users

### What is a Role?

In PostgreSQL, a **role** is an entity that can connect to the database and own database objects. A role can be:
- A **login role** (user): Can connect to the database
- A **group role**: Cannot connect, used to organize permissions

```sql
-- Create a basic login role (user)
CREATE ROLE alice WITH LOGIN PASSWORD 'secure_password';

-- Create a role without login capability (group role)
CREATE ROLE analysts;

-- Create a superuser role (not recommended for most users)
CREATE ROLE admin WITH LOGIN SUPERUSER PASSWORD 'password';

-- Create a role with expiration
CREATE ROLE temporal_user WITH LOGIN PASSWORD 'pass' VALID UNTIL '2026-12-31';

-- Create a role that can create databases
CREATE ROLE dev_user WITH LOGIN CREATEDB PASSWORD 'pass';
```

### Role Attributes

Common attributes when creating roles:

| Attribute | Meaning |
|-----------|---------|
| `LOGIN` | Can connect to database |
| `SUPERUSER` | Has all privileges (bypass permission checks) |
| `CREATEDB` | Can create new databases |
| `CREATEROLE` | Can create other roles |
| `INHERIT` | Inherits privileges of member roles (default: ON) |
| `NOINHERIT` | Doesn't inherit privileges |
| `CONNECTION LIMIT n` | Max simultaneous connections (-1 = unlimited) |
| `PASSWORD` | Set login password |
| `VALID UNTIL` | Expiration timestamp |

```sql
-- Full example with multiple attributes
CREATE ROLE project_manager 
  WITH LOGIN 
  PASSWORD 'secure_pass'
  CREATEDB
  CONNECTION LIMIT 5
  VALID UNTIL '2027-01-01';
```

### Altering and Dropping Roles

```sql
-- Modify a role
ALTER ROLE alice WITH PASSWORD 'new_password';
ALTER ROLE alice CONNECTION LIMIT 10;
ALTER ROLE alice NOINHERIT;

-- Check role details
\du  -- List all roles in psql

-- Drop a role
DROP ROLE alice;  -- Fails if role owns objects

-- Drop role and reassign objects
DROP ROLE alice CASCADE;  -- Dangerous: deletes objects owned by alice
```

---

## GRANT and REVOKE

### Basic GRANT Syntax

```sql
-- Grant privileges on a table
GRANT privilege_type ON table_name TO role_name;

-- Grant multiple privileges
GRANT SELECT, INSERT, UPDATE ON users TO bob;

-- Grant on all tables in schema
GRANT SELECT, INSERT ON ALL TABLES IN SCHEMA public TO analysts;

-- Grant with admin option (grantee can grant to others)
GRANT SELECT ON products TO bob WITH GRANT OPTION;

-- Grant to public (all roles)
GRANT SELECT ON sales TO PUBLIC;
```

### REVOKE Syntax

```sql
-- Revoke a privilege
REVOKE privilege_type ON table_name FROM role_name;

-- Revoke multiple privileges
REVOKE INSERT, UPDATE ON users FROM bob;

-- Revoke all privileges
REVOKE ALL PRIVILEGES ON users FROM bob;

-- Revoke with cascade (revoke from those bob granted to)
REVOKE GRANT OPTION FOR SELECT ON products FROM bob CASCADE;

-- Revoke admin option only
REVOKE ADMIN OPTION FOR role_name FROM user_name;
```

---

## Common Privileges

### Table Privileges

| Privilege | Allows |
|-----------|--------|
| `SELECT` | Read rows using SELECT |
| `INSERT` | Add rows using INSERT |
| `UPDATE` | Modify rows using UPDATE |
| `DELETE` | Remove rows using DELETE |
| `TRUNCATE` | Delete all rows using TRUNCATE |
| `REFERENCES` | Create foreign keys referencing this table |
| `TRIGGER` | Create triggers on this table |
| `ALL` | All privileges (except GRANT OPTION) |

```sql
-- Common patterns

-- Read-only analyst
CREATE ROLE analyst WITH LOGIN PASSWORD 'pass';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO analyst;

-- Data entry operator (read + insert + update)
CREATE ROLE data_entry WITH LOGIN PASSWORD 'pass';
GRANT SELECT, INSERT, UPDATE ON users TO data_entry;
GRANT SELECT, INSERT, UPDATE ON orders TO data_entry;

-- Database administrator for a schema
CREATE ROLE schema_admin WITH LOGIN PASSWORD 'pass';
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO schema_admin;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO schema_admin;

-- No permissions (explicit)
CREATE ROLE restricted WITH LOGIN PASSWORD 'pass';
-- Grant nothing - complete lockdown

-- Audit role (select only for compliance)
CREATE ROLE auditor WITH LOGIN PASSWORD 'pass';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO auditor;
```

---

## Schema-Level Privileges

### Schema Privileges

| Privilege | Allows |
|-----------|---------|
| `USAGE` | Access objects within the schema (SELECT from tables, call functions) |
| `CREATE` | Create new objects in the schema (tables, views, functions, indexes) |

```sql
-- User can access existing tables but not create new ones
GRANT USAGE ON SCHEMA public TO bob;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO bob;

-- User can create tables in schema
GRANT USAGE, CREATE ON SCHEMA public TO dev_team;

-- Create a private schema for a user
CREATE SCHEMA private_alice;
GRANT USAGE, CREATE ON SCHEMA private_alice TO alice;
REVOKE USAGE ON SCHEMA private_alice FROM PUBLIC;

-- User cannot see schema existence without USAGE
-- Without USAGE: SELECT from public.users fails even if user has SELECT privilege

-- Check schema privileges
\dn+  -- List schemas with privileges
```

### Real-World Example

```sql
-- New developer setup
CREATE ROLE junior_dev WITH LOGIN PASSWORD 'pass';

-- Can use public schema (read existing objects)
GRANT USAGE ON SCHEMA public TO junior_dev;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO junior_dev;

-- Can create in development schema
CREATE SCHEMA dev;
GRANT USAGE, CREATE ON SCHEMA dev TO junior_dev;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA dev TO junior_dev;

-- Cannot modify production tables
-- (No INSERT, UPDATE, DELETE on public schema tables)
```

---

## Role Membership and Groups

### Creating Role Hierarchies

In PostgreSQL, roles can be members of other roles to create group hierarchies:

```sql
-- Create a group role (no LOGIN)
CREATE ROLE data_team;

-- Create individual users
CREATE ROLE alice WITH LOGIN PASSWORD 'pass';
CREATE ROLE bob WITH LOGIN PASSWORD 'pass';

-- Add users to group
GRANT data_team TO alice;
GRANT data_team TO bob;

-- Grant privileges to group (inherited by members)
GRANT SELECT, INSERT, UPDATE ON orders TO data_team;

-- Now alice and bob inherit these privileges
```

### Inheritance Behavior

```sql
-- By default, roles inherit privileges (INHERIT is default)
CREATE ROLE alice WITH LOGIN INHERIT PASSWORD 'pass';  -- Default

-- Without inheritance, must explicitly set role
CREATE ROLE bob WITH LOGIN NOINHERIT PASSWORD 'pass';

-- In psql, SET ROLE to explicitly become a role
SET ROLE data_team;  -- Now executing as data_team
RESET ROLE;  -- Return to current user

-- bob doesn't inherit privileges unless SET ROLE data_team used
-- alice inherits automatically
```

### Nested Groups

```sql
-- Create hierarchy: managers contain team leads, contain developers
CREATE ROLE developers;
CREATE ROLE team_leads;
CREATE ROLE managers;

-- team_leads is member of managers
GRANT managers TO team_leads;

-- developers is member of team_leads
GRANT team_leads TO developers;

-- Grant to top-level group
GRANT SELECT, INSERT ON products TO developers;

-- Privilege flows down: managers inherit what developers have
GRANT SELECT, UPDATE ON prices TO managers;

-- team_leads inherit from both
```

### Viewing Role Membership

```sql
-- List roles and their members
\du+

-- Query system catalog
SELECT 
  r.rolname as role_name,
  m.rolname as member_name
FROM pg_roles r
JOIN pg_auth_members am ON r.oid = am.roleid
JOIN pg_roles m ON am.member = m.oid
ORDER BY r.rolname, m.rolname;
```

---

## Owner vs Grantee

### Object Ownership

Every database object has an **owner** (the role that created it):

```sql
-- When alice creates a table, she is the owner
CREATE TABLE alice_table (id INT);
-- Owner: alice

-- alice can grant privileges to others, but only she can DROP or ALTER
ALTER TABLE alice_table ADD COLUMN name TEXT;  -- alice can do this
-- bob cannot, even with ALL PRIVILEGES

-- bob can grant it only if alice granted WITH GRANT OPTION
GRANT ALL PRIVILEGES ON alice_table TO bob WITH GRANT OPTION;
```

### Key Differences

| Aspect | Owner | Grantee |
|--------|-------|---------|
| Can DELETE object | ✓ Yes | ✗ No (even with ALL) |
| Can ALTER object | ✓ Yes | ✗ No (even with ALL) |
| Can DROP object | ✓ Yes | ✗ No (even with ALL) |
| Can GRANT to others | ✓ If GRANT OPTION | ✓ If WITH GRANT OPTION |
| Can REVOKE from others | ✓ Yes | ✓ If WITH GRANT OPTION |
| Can change object privileges | ✓ Yes | ✗ No |

### Changing Ownership

```sql
-- Transfer ownership (requires superuser OR new owner is you)
ALTER TABLE users OWNER TO bob;

-- bob is now owner; original owner loses modification rights
-- bob can alter, drop, and modify privileges

-- Change schema ownership
ALTER SCHEMA my_schema OWNER TO alice;

-- Change role ownership (must be superuser)
ALTER ROLE alice OWNER TO new_superuser;  -- Not standard for roles
```

### Practical Scenario

```sql
-- Application user should own tables (easier permission management)
CREATE ROLE app_user WITH LOGIN PASSWORD 'app_pass';

-- Connect as app_user and create tables
-- (or as superuser with OWNER TO app_user)

CREATE TABLE app_user.products (id INT, name TEXT) 
  WITH (autovacuum_enabled = true);

-- Now app_user owns the table
-- App can modify; admin can SELECT if granted privilege

GRANT SELECT, INSERT, UPDATE, DELETE ON app_user.products TO app_admin;
-- app_admin can query/modify but not alter schema or drop
```

---

## Reading Permission Denied Errors

### Common Permission Errors

```
ERROR: permission denied for schema public
ERROR: permission denied for table users
ERROR: permission denied for sequence id_seq
ERROR: insufficient privilege
```

### Diagnosing the Problem

```sql
-- 1. Check current user
SELECT current_user;  -- Who am I connected as?

-- 2. Check if table exists
SELECT * FROM information_schema.tables 
WHERE table_name = 'users';

-- 3. Check table privileges
SELECT grantee, privilege_type 
FROM information_schema.role_table_grants 
WHERE table_name = 'users';

-- 4. Check schema privileges
SELECT grantee, privilege_type 
FROM information_schema.role_usage_grants 
WHERE object_name = 'public';

-- 5. Check role membership (does user inherit from groups?)
SELECT 
  m.rolname as member,
  r.rolname as role,
  am.admin_option
FROM pg_auth_members am
JOIN pg_roles r ON am.roleid = r.oid
JOIN pg_roles m ON am.member = m.oid
WHERE m.rolname = 'alice';  -- Replace with your user
```

### Troubleshooting Flow

```
ERROR: permission denied for table users
├─ Is user connected? (SELECT current_user;)
│
├─ Does schema exist? (Schema=public usually)
│  └─ If no: CREATE SCHEMA
│
├─ Does user have USAGE on schema?
│  └─ If no: GRANT USAGE ON SCHEMA public TO user;
│
├─ Does user have SELECT on table?
│  └─ If no: GRANT SELECT ON users TO user;
│
├─ Is user inheriting from a role without privilege?
│  └─ Check GRANT memberships and INHERIT flag
│
└─ Does user have enough connection slots?
   └─ Check CONNECTION LIMIT
```

### Example: Debugging Failed INSERT

```sql
-- Error: "permission denied for table orders"

-- Step 1: Who am I?
SELECT current_user;  -- alice

-- Step 2: Can I access the schema?
-- (If error here, missing USAGE on schema)
SELECT * FROM information_schema.schemata WHERE schema_name = 'public';

-- Step 3: Do I have INSERT privilege?
SELECT * FROM information_schema.role_table_grants 
WHERE grantee = 'alice' AND table_name = 'orders';
-- Shows: privilege_type = SELECT only (no INSERT)

-- Step 4: Grant the missing privilege
-- (as superuser or table owner)
GRANT INSERT ON orders TO alice;

-- Step 5: Retry
INSERT INTO orders (product_id, qty) VALUES (1, 5);  -- Now works
```

### Sequence Permission Errors

```
ERROR: permission denied for sequence order_id_seq
```

Sequences need separate permissions from tables:

```sql
-- User has INSERT on table but sequence permission missing
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,  -- Creates sequence order_id_seq
  product_id INT,
  qty INT
);

-- Grant on table but not sequence
GRANT SELECT, INSERT ON orders TO bob;

-- Bob's INSERT fails because sequence doesn't have permission
-- Fix: Grant sequence privileges
GRANT USAGE, SELECT ON order_id_seq TO bob;

-- Or grant all at once
GRANT ALL PRIVILEGES ON orders, order_id_seq TO bob;

-- Grant to all sequences in schema
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO bob;
```

### Default Privileges for New Objects

```sql
-- Future tables created by alice won't be accessible to bob
-- Set default privileges to auto-grant

-- As alice (the table creator)
ALTER DEFAULT PRIVILEGES 
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO bob;

-- Now any table alice creates automatically grants bob permissions
CREATE TABLE new_table (id INT);
-- bob can immediately SELECT/INSERT/UPDATE/DELETE

-- For sequences
ALTER DEFAULT PRIVILEGES 
  GRANT USAGE, SELECT ON SEQUENCES TO bob;

-- Set defaults for a specific schema
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
  GRANT SELECT ON TABLES TO analysts;
```

---

## Quick Reference

### Essential Commands

```sql
-- Role Management
CREATE ROLE name WITH LOGIN PASSWORD 'pass';
ALTER ROLE name WITH PASSWORD 'newpass';
DROP ROLE name;
\du                    -- List roles

-- Grant Privileges
GRANT SELECT ON table TO role;
GRANT privilege ON ALL TABLES IN SCHEMA schema TO role;
GRANT USAGE ON SCHEMA schema TO role;
GRANT role TO user;    -- Add user to group

-- Revoke Privileges
REVOKE privilege ON table FROM role;
REVOKE role FROM user; -- Remove from group

-- Troubleshooting
SELECT current_user;
\dn+                   -- Schemas and privileges
\dp table_name         -- Table privileges
\dx                    -- Extension privileges

-- Default Privileges
ALTER DEFAULT PRIVILEGES GRANT SELECT ON TABLES TO role;
ALTER DEFAULT PRIVILEGES IN SCHEMA s GRANT ... TO role;

-- Ownership
ALTER TABLE name OWNER TO new_owner;
ALTER SCHEMA name OWNER TO new_owner;
```

### Permission Checklist

- [ ] Role created with CREATE ROLE
- [ ] Role has LOGIN if needs database connection
- [ ] Schema has USAGE granted
- [ ] Table has required privileges (SELECT, INSERT, UPDATE, DELETE)
- [ ] Sequence has USAGE granted (if using SERIAL)
- [ ] Role inherits from group roles (if applicable)
- [ ] Default privileges set for future objects
- [ ] Owner vs grantee limitations understood

---

## Key Takeaways

1. **Roles**: Everything is a role (users are roles with LOGIN)
2. **GRANT/REVOKE**: Grants permissions; REVOKE removes them
3. **Privileges**: SELECT, INSERT, UPDATE, DELETE, TRUNCATE, TRIGGER, REFERENCES
4. **Schema access**: Requires USAGE privilege; CREATE allows object creation
5. **Groups**: Use role membership to manage permissions at scale
6. **Ownership**: Owners can alter/drop; grantees can only use if granted
7. **Inheritance**: Default behavior; roles inherit from member groups
8. **Errors**: Check USAGE on schema, privilege on object, role membership
9. **Sequences**: Need separate USAGE grant from table INSERT privilege
10. **Default privileges**: Auto-grant to future objects created by a role
