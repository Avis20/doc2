---
title: "Postgres - Создание стандартный полей: ts_create, ts_modify, user_id, modify_user_id"
categories: postgres
tag: [databases, catalyst]
reference:
  -
---

* TOC 
{:toc}


## Поля

<!-- ------------------------------------------------------------- -->

<pre><code class="sql">ALTER TABLE schema.table
  ADD COLUMN ts_create TIMESTAMP(0) WITHOUT TIME ZONE DEFAULT now() NOT NULL;

ALTER TABLE schema.table
  ADD COLUMN ts_modify TIMESTAMP(0) WITHOUT TIME ZONE;

ALTER TABLE schema.table
  ADD COLUMN user_id INTEGER NOT NULL;

ALTER TABLE schema.table
  ADD COLUMN modify_user_id INTEGER;

ALTER TABLE schema.table
  ADD COLUMN _trigger_off BOOLEAN;

COMMENT ON COLUMN schema.table.ts_create
IS 'Дата создания записи';

COMMENT ON COLUMN schema.table.ts_modify
IS 'Дата последнего обновления';

COMMENT ON COLUMN schema.table.user_id
IS 'Пользователь создавший запись';

COMMENT ON COLUMN schema.table.modify_user_id
IS 'Пользователь изменивший запись';
</code></pre>

## Триггеры

### insert
<pre><code class="sql">
CREATE OR REPLACE FUNCTION schema._items_insert_before (
)
RETURNS trigger AS
$body$
DECLARE
    r_user          "public"."users";
BEGIN
    IF NEW._trigger_off IS TRUE THEN
        NEW._trigger_off := NULL;
        RETURN NEW;
    END IF;
 
    NEW.ts_create       := now();
    NEW.ts_modify       := now();
    
-- check user
    SELECT * INTO r_user
        FROM "public".check_user(NEW.user_id, row_to_json(NEW), 'user_id');
    IF r_user.id IS NULL THEN
        RETURN NULL;
    END IF;
    
    NEW.modify_user_id := NEW.user_id;
 
    RETURN NEW;
END;
$body$
LANGUAGE 'plpgsql'
VOLATILE
CALLED ON NULL INPUT
SECURITY INVOKER
COST 100;

CREATE TRIGGER items_insert_before
  BEFORE INSERT 
  ON schema.items
  
FOR EACH ROW 
  EXECUTE PROCEDURE schema._items_insert_before();
</code></pre>

### update
<pre><code class="sql">shema
CREATE OR REPLACE FUNCTION schema._items_update_before (
)
RETURNS trigger AS
$body$
DECLARE
    r_user      "public"."users";
BEGIN
    IF NEW._trigger_off IS TRUE THEN
        NEW._trigger_off := NULL;
        RETURN NEW;
    END IF;
 
-- blocked update fields
    NEW.user_id        := OLD.user_id;
    
    NEW.ts_modify       := now(); 
    NEW._trigger_off    := NULL;

-- check modify user
    SELECT * INTO r_user
        FROM "public".check_user(NEW.modify_user_id, row_to_json(NEW), 'modify_user_id');
    IF r_user.id IS NULL THEN
        RETURN NULL;
    END IF;

    RETURN NEW;
END;
$body$
LANGUAGE 'plpgsql'
VOLATILE
CALLED ON NULL INPUT
SECURITY INVOKER
COST 100;


CREATE TRIGGER items_update_before
  BEFORE UPDATE 
  ON schema.items
  
FOR EACH ROW 
  EXECUTE PROCEDURE schema._items_update_before();
</code></pre>

### delete
<pre><code class="sql">
CREATE OR REPLACE FUNCTION schema._items_delete_before (
)
RETURNS trigger AS
$body$
BEGIN
    RETURN NULL;
END;
$body$
LANGUAGE 'plpgsql'
VOLATILE
CALLED ON NULL INPUT
SECURITY INVOKER
COST 100;

CREATE TRIGGER items_delete_before
  BEFORE DELETE
  ON schema.items
  
FOR EACH ROW 
  EXECUTE PROCEDURE schema._items_delete_before();
</code></pre>

## check функции
<pre><code class="sql">
CREATE OR REPLACE FUNCTION chat.check_item (
  chat_id integer,
  params json = '{}'::json
)
RETURNS chat.items AS
$body$
DECLARE
    r_chat      "chat"."items";
BEGIN
   IF chat_id IS NULL THEN
        RAISE WARNING '{"success":0,"error":428,"error_field":"chat_id","params":%}', params;
        RETURN NULL;
    END IF;

    SELECT * INTO r_chat FROM "chat"."items" 
        WHERE id = chat_id;
    IF r_chat.id IS NULL THEN
        RAISE WARNING '{"success":0,"error":424,"error_field":"chat_id","params":%}', params;
        RETURN NULL;
    END IF;
     
    RETURN r_chat;
END;
$body$
LANGUAGE 'plpgsql'
VOLATILE
CALLED ON NULL INPUT
SECURITY INVOKER
COST 10;
</code></pre>


## DBIx

<!-- ------------------------------------------------------------- -->

<pre><code class="perl">
    'ts_create',
        {
            data_type           => 'datetime',
            timezone            => 'local',
            is_nullable         => 0,
            default_value       => \'now()',
            retrieve_on_insert  => 1
        },
    'ts_modify',
        {
            data_type           => 'datetime',
            timezone            => 'local',
            is_nullable         => 1,
        },
    'user_id',
        {
            data_type           => 'integer',
            is_nullable         => 0
        },
    'modify_user_id',
        {
            data_type           => 'integer',
            is_nullable         => 1
        },
</code></pre>
 
