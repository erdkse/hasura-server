This repo contains changes that made to database of the hasura project. Migrations can be apply easily to staging and production environments.

#### Hasura Migration Official Documentation

https://docs.hasura.io/1.0/graphql/manual/migrations/manage-migrations.html

#### Creating a Migration Project

This is already performed in this repository, so it does not need to execute.

However, if you need to setup in another repository based on https://hasura.erdkse.xyz, this is the command to execute:

```
$ hasura init --directory hasura-db-migrations --endpoint https://hasura.erdkse.xyz
```

#### Creating a Migration Which Contains Full Metadata and Schema Information

```
$ hasura migrate create "init" --from-server --admin-secret "<some_secret_key>"
```

#### Apply All Migrations to Another Environment

If you are interested in applying all migrations in "Migrations" folder:

```
$ hasura migrate apply --endpoint http://new-hasura.erdkse.xyz --admin-secret "<some_secret_key>"
```

#### Applying a Desired Migration to Another Environment

If you are interested in migrating version named **1574066847808** to **http://new-hasura.erdkse.xyz** endpoint:

```
$ hasura migrate apply --version "1574066847808" --endpoint http://hasura.erdkse.xyz --admin-secret "<some_secret_key>"
```

#### Checking Status of Migrations

You can use:

```
$ hasura migrate status --endpoint http://new-hasura.erdkse.xyz --admin-secret "<some_secret_key>"
```

This will print the differences:

```
VERSION        SOURCE STATUS  DATABASE STATUS
1550925483858  Present        Present
1550931962927  Present        Present
1550931970826  Present        Present
```

#### Making Changes

All changes must be done in a local console. A migration will be created automatically as you make changes to the environment.

##### Starting Local Console

```
$ hasura console --admin-secret "<some_secret_key>"
```

#### Moving Data to New Environment

This section is not about migration, but if moving data is neccesseary below steps must be completed:

1. Create a brand new empty database.
2. Store the new database connection string to new Hasura instance's environment variables as **HASURA_GRAPHQL_DATABASE_URL**.
3. Apply changes to Hasura instance by using command:

```
$ docker-compose up -d
```

This will create Hasura default schemas to the new database.

4. Apply a full metadata and schema migration that mentioned above.
5. Using a pgadmin create a plain, only-data dump file on existing database.
6. You need to make some changes on that file:
   a. Put sql in a transaction by adding

   ```
   BEGIN;
   ```

   to the top of the file and

   ```
   COMMIT;
   ```

   to the end of the file.

   b. You need to drop the all foreign keys and write their create sql commands to a table. So, you need to add this code snippet after transaction started:

   ```
   create table if not exists public.dropped_foreign_keys (
        seq bigserial primary key,
        sql text
   );
   do $$ declare t record;
   begin
   for t in select conrelid::regclass::varchar table_name, conname constraint_name,
   pg_catalog.pg_get_constraintdef(r.oid, true) constraint_definition
   from pg_catalog.pg_constraint r
   where r.contype = 'f'
   -- current schema only:
   and r.connamespace = (select n.oid from pg_namespace n where n.nspname = 'public')
   loop

        insert into public.dropped_foreign_keys (sql) values (
            format('alter table public.%s add constraint %s %s',
                quote_ident(t.table_name), quote_ident(t.constraint_name), t.constraint_definition));

        execute format('alter table public.%s drop constraint %s', t.table_name, quote_ident(t.constraint_name));

    end loop;
    end;
    $$;
   ```

   c. You also need to drop the all triggers and write their create sql commands to a table. So, you need to add this code now:

   ```
   create table public.dropped_triggers (
   seq bigserial primary key,
   sql text
   );
   do $$ declare triggNameRecord record;
      declare triggTableRecord record;
   BEGIN
    FOR triggNameRecord IN select trigger_name, action_timing,event_manipulation,action_statement, trigger_schema
   from information_schema.triggers
   where trigger_schema = 'public'
   OR trigger_schema = 'hdb_views' LOOP
        FOR triggTableRecord IN SELECT distinct(event_object_table)
   from information_schema.triggers
   where trigger_name = triggNameRecord.trigger_name LOOP
            RAISE NOTICE 'Dropping trigger: % on table: %', triggNameRecord.trigger_name, triggTableRecord.event_object_table;
   PERFORM  triggNameRecord ;
            insert into public.dropped_triggers (sql) values(
   	format('create trigger %s %s %s ON %s.%s FOR EACH ROW %s;',
   		   quote_ident(triggNameRecord.trigger_name),
   		   triggNameRecord.action_timing,
   		   triggNameRecord.event_manipulation,
   		   quote_ident(triggNameRecord.trigger_schema),
   		   quote_ident(triggTableRecord.event_object_table),
   		   triggNameRecord.action_statement
   		  ));
    EXECUTE format('DROP TRIGGER %s ON %s.%s;', quote_ident(triggNameRecord.trigger_name),quote_ident(triggNameRecord.trigger_schema), quote_ident(triggTableRecord.event_object_table));
        END LOOP;
    END LOOP;
   END;
   $$;
   ```

   d. If you need to drop some data from database you can do this here. In our case enum tables that have values, and need to delete:

   ```
   DELETE FROM public.decks_status;
   DELETE FROM public.enum_permissions;
   ```

   e. We need to recreate the foreign keys at the end of the script, before commiting the transaction. Put these code snippet at the end of the file before **COMMIT** keyword:

   ```
   set search_path to public;
   do $$ declare t record;
   begin
    -- order by seq for easier troubleshooting when data does not satisfy FKs
    for t in select * from public.dropped_foreign_keys order by seq loop
        execute t.sql;
        delete from public.dropped_foreign_keys where seq = t.seq;
    end loop;
   end $$;
   ```

   f. Also, we need to recreate the triggers at the end of the script, after recreating the foreign keys. Put these code snippet after the above code:

   ```
   do $$ declare trig record;
   begin
    -- order by seq for easier troubleshooting when data does not satisfy FKs
    for trig in select * from public.dropped_triggers order by seq loop
        execute trig.sql;
        delete from public.dropped_triggers where seq = trig.seq;
    end loop;
   end $$;
   ```

That's all, enjoy with new environment. :)
