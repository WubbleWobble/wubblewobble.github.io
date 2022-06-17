---
layout: post
title:  "Doctrine migrations and unexpected MySQL revelations"
date:   2022-06-16 01:01:01 +0100
categories: symfony, doctrine, mysql
---

The other day I came across something that was new to me. For many years I've been coding using a PHP, Symfony and Doctrine stack, and as part of this was managing sites' database entities using the [Doctrine Migrations bundle](https://symfony.com/bundles/DoctrineMigrationsBundle/current/index.html).

Generally when a site was going to go live, I'd generate a huge migration to represent the initial state of its database, and any entity/database changes after that would be their own incremental migrations - the idea being that the site's database structure (and data) would be safely migrated to its new form with each site feature update.

Such a migration might look something like this:

```php
final class Version20220414112450 extends AbstractMigration
{
    public function getDescription(): string
    {
        return 'Update the blah blah to support rar-rar';
    }

    public function up(Schema $schema): void
    {
        $this->addSql('CREATE TABLE thing (id INT NOT NULL, code INT)');
        $this->addSql('ALTER TABLE blah DROD obvious_typo');
    }

    public function down(Schema $schema): void
    {
        // ... snip ...
        // (The same but in reverse)
    }
}
```

Whilst developing a migration like this, I had a typo in my SQL, and it migration failed on the second statement. After I fixed the bug, I re-ran the migration and was met by an error:

```js
Base table or view already exists: 1050 Table 'thing' already exists
```

This seemed unintuitive to me, as I had thought that the Doctrine Migration system would run each migration inside a transaction, and so if one of the SQL statements failed, it would be rolled back for me.

My intuition turned out to be right; I looked at the [documentation](https://www.doctrine-project.org/projects/doctrine-migrations/en/3.3/reference/configuration.html) and it said that indeed it wrapped migrations in a transaction by default, so this wasn't the problem.

## Root cause

Eventually after much searching I found the root cause. It wasn't what I expected and it flew in the face of what I expected from databases.

It seems that MySQL (at least up to version 8.0) triggers what it calls an [implicit commit](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html) upon most DDL statements such as CREATE TABLE. It means that it doesn't matter whether you have a transaction open or not; as soon as you call something like CREATE TABLE, your changes get committed **immediately** and **irreversibly**. 

The effects of such DDL statements cannot be rolled back. This means that there's no safe way to do migrations. If for some reason part of a migration fails to be applied, the database could be left in an indeterminate state.

This really, really sucks.

## Solution

So the solution is rather simple; going forward, I plan to ditch MySQL and transition to PostgreSQL. It apparently is one of many databases that [doesn't suffer from this limitation](https://wiki.postgresql.org/wiki/Transactional_DDL_in_PostgreSQL:_A_Competitive_Analysis).