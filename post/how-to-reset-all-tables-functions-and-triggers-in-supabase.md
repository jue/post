---
title: "如何重置supabase里所有表、函数和触发器"
description: "最近在开发过程中，经常会遇到需要重置Supabase数据库的情况。这些情况包括： 1. 开发阶段的数据污染：在开发和测试阶段，我们可能会产生大量的测试数据，这些数据在项目上线或迭代到下一个阶段时就变得无用，甚至会干扰系统的正常运行。 2. 数据库结构变更：当项目需求发生重大变化，数据库的表结构、关系..."
date: 2025-05-05
lastUpdated: 2026-01-13
authors:
  - name: "jue"
    link: "https://github.com/jue"
    avatar: "https://avatars.githubusercontent.com/u/377499?v=4"
categories:
  - "Tech Talk"
wordCount: 1653
readingTime: 6
githubIssue: 1
githubUrl: "https://github.com/jue/post/issues/1"
---
最近在开发过程中，经常会遇到需要重置Supabase数据库的情况。这些情况包括：

1. 开发阶段的数据污染：在开发和测试阶段，我们可能会产生大量的测试数据，这些数据在项目上线或迭代到下一个阶段时就变得无用，甚至会干扰系统的正常运行。
2. 数据库结构变更：当项目需求发生重大变化，数据库的表结构、关系或约束需要大规模调整时，有时完全重置数据库比逐个修改更简单高效。
3. 迁移问题修复：如果在使用Supabase的迁移功能时出现了问题，可能导致数据库状态不一致，这时重置可以帮助我们回到一个干净的状态。
4. 开发环境重建：在团队协作开发中，新成员加入或环境迁移时，需要快速建立一个干净的开发环境。
5. 性能优化：长期运行的数据库可能会因为碎片化等问题而性能下降，完全重置可以解决这些问题。

无论出于什么原因，了解如何安全、有效地重置Supabase环境都是非常重要的技能。接下来，我们将详细介绍如何重置Supabase中的所有表、函数和触发器。

在 Supabase 中完全清空数据库需要谨慎操作，因为这将删除所有数据和结构。以下是逐步操作指南：

## 方法一：使用 SQL 编辑器
Supabase 提供了一个内置的 SQL 编辑器，您可以通过它执行以下操作：

### 1. 删除所有表
```sql
-- 生成删除所有表的脚本
SELECT 'DROP TABLE IF EXISTS "' || tablename || '" CASCADE;' 
FROM pg_tables
WHERE schemaname = 'public';
```
执行上述查询会生成一系列 DROP TABLE 语句。复制这些语句并执行它们。

### 2. 删除所有函数
```sql
-- 生成删除所有函数的脚本
SELECT 'DROP FUNCTION IF EXISTS ' || ns.nspname || '.' || proname || '(' || oidvectortypes(proargtypes) || ') CASCADE;'
FROM pg_proc INNER JOIN pg_namespace ns ON (pg_proc.pronamespace = ns.oid)
WHERE ns.nspname = 'public';
```

### 3. 删除所有触发器
```sql
-- 生成删除所有触发器的脚本
SELECT 'DROP TRIGGER IF EXISTS ' || trigger_name || ' ON ' || event_object_table || ' CASCADE;'
FROM information_schema.triggers
WHERE trigger_schema = 'public';
```

### 4. 删除所有序列
```sql
-- 生成删除所有序列的脚本
SELECT 'DROP SEQUENCE IF EXISTS "' || sequence_name || '" CASCADE;'
FROM information_schema.sequences
WHERE sequence_schema = 'public';
```

### 5. 删除所有视图
```sql
-- 生成删除所有视图的脚本
SELECT 'DROP VIEW IF EXISTS "' || table_name || '" CASCADE;'
FROM information_schema.views
WHERE table_schema = 'public';
```

## 方法二：重置项目
在 Supabase 仪表板中：

1. 转到 Project Settings → General
2. 滚动到底部找到 "Danger Zone"
3. 点击 "Reset Database"

这将重置数据库到初始状态，保留 auth 和 storage 等系统表，但删除所有自定义表、函数和触发器。

### 重要提示

1. **备份数据**：在执行任何删除操作前，确保备份所有重要数据
2. **系统表**：上述操作会删除用户创建的表和函数，但不会删除 Supabase 的系统表 (auth, storage 等)
3. **权限**：对于某些操作可能需要所有者权限
4. **RLS 策略**：删除表也会删除关联的 RLS 策略
5. **外键约束**：使用 CASCADE 选项确保带有外键约束的表能被正确删除
这些方法将帮助您彻底清理 Supabase 数据库并在需要时重新开始。请确保在生产环境中特别谨慎地执行这些操作。

## 一键SQL脚本

以下是一段完整的 SQL 脚本，可以帮助您删除 Supabase 数据库中的所有表、函数、触发器、序列和视图。您可以直接复制粘贴到 Supabase 的 SQL 编辑器中运行：

```sql
-- ========================================
-- Supabase 数据库完整清理脚本
-- ========================================
-- 此脚本会清理 public schema 下的所有自定义对象
-- 保留 auth、storage 和系统表
-- ========================================

-- 第一步：删除所有触发器
DO $$
DECLARE
    trigger_stmt TEXT;
BEGIN
    FOR trigger_stmt IN
        SELECT 'DROP TRIGGER IF EXISTS "' || trigger_name || '" ON "' || event_object_table || '" CASCADE;'
        FROM information_schema.triggers
        WHERE trigger_schema = 'public'
    LOOP
        EXECUTE trigger_stmt;
        RAISE NOTICE '已删除触发器: %', trigger_stmt;
    END LOOP;
END $$;

-- 第二步：删除所有视图
DO $$
DECLARE
    view_stmt TEXT;
BEGIN
    FOR view_stmt IN
        SELECT 'DROP VIEW IF EXISTS "' || table_name || '" CASCADE;'
        FROM information_schema.views
        WHERE table_schema = 'public'
    LOOP
        EXECUTE view_stmt;
        RAISE NOTICE '已删除视图: %', view_stmt;
    END LOOP;
END $$;

-- 第三步：删除所有表（保留 auth、storage 和系统表）
DO $$
DECLARE
    table_stmt TEXT;
BEGIN
    FOR table_stmt IN
        SELECT 'DROP TABLE IF EXISTS "' || tablename || '" CASCADE;'
        FROM pg_tables
        WHERE schemaname = 'public'
        AND tablename NOT LIKE 'pg_%'
        AND tablename NOT LIKE 'auth_%'
        AND tablename NOT LIKE 'storage_%'
    LOOP
        EXECUTE table_stmt;
        RAISE NOTICE '已删除表: %', table_stmt;
    END LOOP;
END $$;

-- 第四步：删除所有序列
DO $$
DECLARE
    seq_stmt TEXT;
BEGIN
    FOR seq_stmt IN
        SELECT 'DROP SEQUENCE IF EXISTS "' || sequence_name || '" CASCADE;'
        FROM information_schema.sequences
        WHERE sequence_schema = 'public'
    LOOP
        EXECUTE seq_stmt;
        RAISE NOTICE '已删除序列: %', seq_stmt;
    END LOOP;
END $$;

-- 第五步：删除所有自定义函数（排除扩展提供的函数）
DO $$
DECLARE
    func_stmt TEXT;
BEGIN
    FOR func_stmt IN
        SELECT 'DROP FUNCTION IF EXISTS ' || ns.nspname || '.' || proname || '(' || oidvectortypes(proargtypes) || ') CASCADE;'
        FROM pg_proc
        INNER JOIN pg_namespace ns ON (pg_proc.pronamespace = ns.oid)
        WHERE ns.nspname = 'public'
        AND NOT EXISTS (
            SELECT 1 FROM pg_depend d
            JOIN pg_extension e ON d.refobjid = e.oid
            WHERE d.objid = pg_proc.oid AND d.deptype = 'e'
        )
    LOOP
        EXECUTE func_stmt;
        RAISE NOTICE '已删除函数: %', func_stmt;
    END LOOP;
END $$;

-- 第六步：删除所有自定义类型（可选）
DO $$
DECLARE
    type_stmt TEXT;
BEGIN
    FOR type_stmt IN
        SELECT 'DROP TYPE IF EXISTS "' || t.typname || '" CASCADE;'
        FROM pg_type t
        JOIN pg_namespace n ON t.typnamespace = n.oid
        WHERE n.nspname = 'public'
        AND t.typtype = 'c'
    LOOP
        EXECUTE type_stmt;
        RAISE NOTICE '已删除类型: %', type_stmt;
    END LOOP;
END $$;

-- 第七步：删除所有扩展（除了Supabase必需的扩展）
-- Supabase必需的扩展：plpgsql, supabase_vault, pg_cron, pg_net, pgjwt, pg_graphql, uuid-ossp, pgcrypto
DO $$
DECLARE
    ext_name TEXT;
BEGIN
    FOR ext_name IN
        SELECT extname FROM pg_extension 
        WHERE extname NOT IN ('plpgsql', 'supabase_vault', 'pg_cron', 'pg_net', 'pgjwt', 'pg_graphql', 'uuid-ossp', 'pgcrypto', 'btree_gist', 'btree_gin')
    LOOP
        EXECUTE 'DROP EXTENSION IF EXISTS "' || ext_name || '" CASCADE;';
        RAISE NOTICE '已删除扩展: %', ext_name;
    END LOOP;
END $$;

-- ========================================
-- 清理完成提示
-- ========================================
DO $$
BEGIN
    RAISE NOTICE '========================================';
    RAISE NOTICE '数据库清理完成！';
    RAISE NOTICE '========================================';
    RAISE NOTICE '已清理的内容：';
    RAISE NOTICE '- 所有触发器';
    RAISE NOTICE '- 所有视图';
    RAISE NOTICE '- 所有自定义表';
    RAISE NOTICE '- 所有序列';
    RAISE NOTICE '- 所有自定义函数';
    RAISE NOTICE '- 所有自定义类型';
    RAISE NOTICE '- 所有非必需扩展';
    RAISE NOTICE '========================================';
    RAISE NOTICE '保留的内容：';
    RAISE NOTICE '- auth_* 表（认证系统）';
    RAISE NOTICE '- storage_* 表（存储系统）';
    RAISE NOTICE '- pg_* 表（系统表）';
    RAISE NOTICE '- Supabase必需扩展';
    RAISE NOTICE '========================================';
END $$;
```

这个脚本使用 PostgreSQL 的 DO 语句块来循环执行每个类型的删除操作。它按照触发器、视图、表、序列和函数的顺序进行删除，这样可以减少依赖关系导致的问题。  

在表的删除部分，我添加了过滤条件，避免删除 PostgreSQL 系统表和 Supabase 的系统表（auth 和 storage 相关）。
请注意，执行此脚本将删除您自定义创建的所有数据库对象，请确保在执行前已备份重要数据。




