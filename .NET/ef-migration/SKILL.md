---
name: ef-core-migrations
description: Professional EF Core migration workflow. Generate migrations through EF CLI, review generated output, and only manually modify migrations when business requirements require it.
---

# EF Core Migration Rules

## Critical Principle

Migration files and ModelSnapshot files are GENERATED ARTIFACTS.

Never manually create, rewrite, or synchronize migration files from scratch.

Do not manually edit:
- *.Designer.cs
- *ModelSnapshot.cs

unless explicitly performing a migration repair operation.

## Standard Workflow

Whenever a schema change is requested:

### Step 1 - Analyze Changes

Determine:

- Which entities changed
- Which EF configurations changed
- Whether data transformation is required
- Whether migration can cause data loss
- Whether indexes or constraints are affected

### Step 2 - Modify Domain Model

Make changes only in:

- Entities
- EntityTypeConfigurations
- DbContext

Do NOT create migration files manually.

### Step 3 - Generate Migration

Always generate migration using EF Core CLI.

**SD-Backend paths (always use these):**

```powershell
dotnet ef migrations add <MigrationName> `
    --project <DB-csproj-path> `
    --startup-project <start-up-csproj-path> `
    --context <DbContext-name>
```

Never manually write migration files before executing the command.

**Before `migrations add`**, confirm no snapshot drift from a prior change:

```powershell
dotnet ef migrations has-pending-model-changes `
    --project <DB-csproj-path> `
    --startup-project <start-up-csproj-path> `
    --context <DbContext-name>
```

If this reports pending changes, fix the model or add a repair migration **before** stacking another feature migration on top.

### Step 4 - Review Generated Migration

After generation inspect:

- Up()
- Down()
- `<MigrationName>.Designer.cs` → `BuildTargetModel`
- `DbContextModelSnapshot.cs`

Verify generated operations are expected.

Pay special attention to:

- DropColumn
- DropTable
- Rename operations
- Foreign keys
- Cascade deletes
- Index changes
- Data loss warnings

#### Snapshot completeness (prevents blocked `MigrateAsync`)

For **new entities** or **new `HasOne` / `HasMany` in configurations**, verify all of the following:

| Check | Pass criteria |
|-------|----------------|
| Entity block | New/changed entity appears in **Designer** and **ModelSnapshot** (not only in `Up()`) |
| Relationships | FK navigations appear in the **relationships section** at the end of Designer/Snapshot (compare with a similar entity, e.g. `HideColumnsSetting` vs `ColumnOrderSetting`) |
| SQL vs model | If `Up()` creates FKs inline in `CreateTable`, snapshot must still record those relationships — SQL alone is not enough |
| No orphan SQL | Never commit a migration whose `Up()` changes schema but whose **Designer omits the entity** |

A migration with correct SQL but an incomplete Designer/Snapshot will pass `dotnet ef database update` once, but **EF Core 10 throws `PendingModelChangesWarning` on the next `MigrateAsync()`** and blocks all later migrations.

### Step 5 - Improve Migration If Needed

Manual modification is allowed ONLY AFTER generation.

Typical valid modifications:

#### Data Migration

```csharp
migrationBuilder.Sql("""
UPDATE Users
SET FullName = FirstName + ' ' + LastName
WHERE FullName IS NULL
""");
```

#### Backfill Data

```csharp
migrationBuilder.Sql("""
UPDATE Products
SET CreatedAt = GETUTCDATE()
WHERE CreatedAt IS NULL
""");
```

#### Reordering Operations

Move:

1. Data copy
2. Constraint creation
3. Column removal

into a safer order.

Example:

Bad:

```csharp
DropColumn("OldName");
CopyData();
```

Good:

```csharp
CopyData();
DropColumn("OldName");
```

#### Renames

Prefer rename operations instead of drop/create.

Replace:

```csharp
DropColumn("Name");
AddColumn("FullName");
```

with:

```csharp
RenameColumn(
    name: "Name",
    table: "Users",
    newName: "FullName");
```

when preserving existing data.

### Step 6 - Validate Snapshot

Verify ModelSnapshot was automatically updated.

Never manually edit snapshot merely to match migration.

If snapshot differs unexpectedly:

1. Investigate model changes.
2. Regenerate migration if needed.
3. Avoid manual snapshot synchronization.

**Required CLI check after every new migration:**

```powershell
dotnet ef migrations has-pending-model-changes `
    --project <DB-csproj-path> `
    --startup-project <start-up-csproj-path> `
    --context <DbContext-name>
```

Must output **no pending changes**. If it fails, do not merge — repair or regenerate first.

### Step 7 - Validate Migration

**Option A — CLI (local DB):**

```powershell
dotnet ef database update `
    --project <DB-csproj-path> `
    --startup-project <start-up-csproj-path> `
    --context <DbContext-name>
```

**Option B — same path as DEV (recommended):**

Stop any running `SD.Host`, then start with Development so `Program.cs` runs `MigrateAsync()`:

```powershell
$env:ASPNETCORE_ENVIRONMENT = 'Development'
dotnet run --project SD/SD.Host/SD.Host.csproj
```

Startup must succeed (no `PendingModelChangesWarning`). This is what DEV uses; `database update` alone can miss that failure mode.

Verify:

- Migration succeeds
- Application starts
- `__EFMigrationsHistory` contains the new `MigrationId`
- Queries execute correctly

### Step 8 - Verify Rollback

Run:

```bash
dotnet ef database update <PreviousMigration>
```

Ensure Down() works.

## Dangerous Operations

Before applying any migration involving:

- DropTable
- DropColumn
- AlterColumn(nullable: false)
- PK changes
- FK changes
- Data type changes

perform additional review.

Assume data preservation is required unless explicitly stated otherwise.

## Migration Naming

Use descriptive names.

Good:

```text
AddUserPreferences
AddOrderAuditFields
RenameCustomerNameToDisplayName
MoveAddressToOwnedEntity
```

Avoid:

```text
UpdateDb
FixStuff
Changes
Migration1
```

## When Existing Migration Is Wrong

If migration has not been committed:

```bash
dotnet ef migrations remove
```

Fix model.

Generate again:

```bash
dotnet ef migrations add <MigrationName>
```

Prefer regeneration over manual repair.

## Snapshot repair (when SQL already exists in DB)

If FKs/columns already exist from an earlier migration but snapshot relationships are missing:

1. Run `dotnet ef migrations add <RepairName>` — EF may generate `AddForeignKey` operations.
2. If those constraints **already exist** in the database, clear `Up()` / `Down()` (snapshot-only migration) and keep the generated Designer/Snapshot update.
3. Re-run `has-pending-model-changes` and SD.Host Development startup test.
4. Never add duplicate FK SQL to environments that already ran the original `CreateTable`.

## SD environment deployment

| Tier | How migrations apply | After merge to `develop` |
|------|----------------------|---------------------------|
| **DEV** | `MigrateAsync()` on startup (`ASPNETCORE_ENVIRONMENT` ≠ Production) | Deploy + **restart app pool** |
| **SDWT** | EF **migration bundle** (Production skips `MigrateAsync`) | Deploy + run latest `efbundle-*.zip` against SD connection string |

**CI:** `azure-pipelines.yml` efbundle must use `--project SD/SD.DataAccess.ApplicationDb` (migrations live there, not `SD.DataAccess`).

## Agent Behavior

When asked to create a migration:

1. NEVER manually write migration files first.
2. ALWAYS generate migration through EF CLI with SD-Backend project paths.
3. Review generated output — **including Designer relationships, not only `Up()`**.
4. Run `has-pending-model-changes` before and after; block merge if pending.
5. Test SD.Host Development startup (`MigrateAsync`), not only `database update`.
6. Explain any suspicious operations.
7. Add custom SQL/data movement only when required.
8. Keep snapshot generated by EF Core.
9. Prefer regeneration over manual synchronization.
10. Treat migrations as database change scripts, not handwritten source code.
11. Preserve production data whenever possible.
12. Warn before any potentially destructive operation.
13. For new tables with navigations, diff snapshot against a similar existing configuration before committing.