---
title: "Connecting Metabase to Supabase Securely with Read-Only Roles and RLS Policies"
date: 2025-10-29
tags: ["supabase", "metabase", "postgres", "rls", "data-visualization", "data-engineering", "security", "dashboard"]
categories: ["Data Engineering"]
---

## 🧭 Situation

I'm setting a **Metabase dashboard** to view the status of my data stored in **Supabase**.  

## Options for Secure Connection to Metabase 

### **Option 1. IP Allowlist via EC2 Security Group**

* Allow inbound access **only** from your home or office IP.
* Effect: Only you can reach the login page.
* Tradeoff: You’ll need to update the rule if you travel or use a new network.

### **Option 2. Put Metabase Behind Reverse Proxy + Basic Auth**

* Run an Nginx reverse proxy on the same EC2 instance.
* Nginx handles:

  * HTTPS (TLS certificate)
  * An extra password gate before users even hit Metabase
* Expose **only port 443** (HTTPS) to the world, not 3000.
* This is the common “small team, serious setup” pattern.


### **Option 3. Don’t Expose It Publicly at All (What I chose)**

Keep port 3000 **closed** to the world.

Instead, use SSH local port forwarding:

```bash
ssh -i your-key.pem \
    -L 3000:localhost:3000 \
    your-ec2@ip-address
```

Metabase will load through the secure SSH tunnel — no public exposure.

* You don’t need HTTPS, Nginx, or any firewall rules.
* It’s the simplest and safest method for a single user setup.
* Perfect if you’re still in early development or using Metabase as a personal “founder dashboard.”

Later, when you onboard teammates or investors, you can layer on Nginx + HTTPS and make it multi-user.


## 🧩 On Supabase side 

#### Make a Read-Only DB User Just for Metabase

Inside Supabase (Postgres), never use your `service_role` for BI tools.
Instead, create a minimal **read-only role** that can view but not modify data.

Even if Metabase is ever compromised, an attacker only gets read access — no inserts, updates, or deletions.
Your production data stays safe.

```sql
create role metabase_reader login password 'STRONG_PASSWORD';
grant usage on schema public to metabase_reader;
grant select on all tables in schema public to metabase_reader;
alter default privileges in schema public
grant select on tables to metabase_reader;
````

💡 *Principle of least privilege:* Metabase can only run SELECT queries and introspect table metadata — nothing else.

## 🧩 Handling Row-Level Security (RLS)

Supabase enforces **Row Level Security (RLS)** by default on many tables.
Even with `SELECT` grants, a role sees **nothing** unless a policy explicitly allows it.

To let Metabase read all rows safely:

```sql
-- Create read policy for Metabase
create policy metabase_can_read_all
on public.table
for select
to metabase_reader
using (true);
```

Repeat for each table you want Metabase to see — or automate it with a DO-block:

```sql
do $$
declare r record;
begin
  for r in
    select table_schema, table_name
    from information_schema.tables
    where table_schema in ('public', 'schema2', 'schema3')
      and table_type = 'BASE TABLE'
  loop
    execute format('alter table %I.%I enable row level security;', r.table_schema, r.table_name);
    begin
      execute format('create policy metabase_can_read_all on %I.%I for select to metabase_reader using (true);',
                     r.table_schema, r.table_name);
    exception when duplicate_object then null;
    end;
  end loop;
end$$;
```

This script:

* Enables RLS for every table in the target schemas.
* Adds a universal read-only policy for `metabase_reader`.
* Skips tables that already have the same policy.

## ✅ Wrap-Up

A secure BI connection is not an afterthought — it’s the foundation.

By combining:

* **read-only Postgres role**, and
* **restricted network path (SSH tunnel)**

You get a dashboard that’s private, auditable, and production-safe.

No public ports.
No privileged credentials.
No accidental data leaks.
