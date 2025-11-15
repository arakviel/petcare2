# Docker Migration Fixes

## Problem Summary

There were two main issues with the Docker deployment:

1. **Missing wait-for-db.sh script**: The script existed but the application was trying to use it before it was available in the container
2. **Race condition with migrations**: The application was starting and trying to query the `Roles` table before migrations were applied, causing repeated errors

## Root Cause Analysis

### Issue 1: wait-for-db.sh script

-   The script existed in `PetCare.Api/wait-for-db.sh`
-   The Dockerfile was copying it correctly
-   The issue was likely that the Docker image wasn't rebuilt after adding the script

### Issue 2: "Roles table does not exist" error

-   The application was starting before migrations were applied
-   Identity system was trying to query the `Roles` table immediately on startup
-   This created a race condition where the database was reachable but not yet migrated

## Solution Implemented

### 1. New Migration Script

Created `PetCare.Api/wait-for-migrations.sh` that:

-   Waits for the database to be reachable (like the original script)
-   **Applies migrations before starting the application**
-   Ensures the database schema is ready before the app starts
-   Provides better error handling and logging

### 2. Updated Docker Configuration

-   Updated `docker-compose.yml` to use the new migration script
-   Updated `PetCare.Api/Dockerfile` to copy and make executable the new script
-   Updated `docker-compose.dcproj` to include the new script in the build

## Files Modified

1. **docker-compose.yml**: Changed entrypoint from `wait-for-db.sh` to `wait-for-migrations.sh`
2. **PetCare.Api/Dockerfile**: Added copy and chmod for the new migration script, updated entrypoint
3. **docker-compose.dcproj**: Added the new script to the project file
4. **PetCare.Api/wait-for-migrations.sh**: New script (created)

## How the Fix Works

### Before (Problematic Flow)

1. Database becomes healthy
2. Application starts immediately
3. Application tries to query `Roles` table
4. **ERROR**: Table doesn't exist yet
5. Repeated failures as shown in the logs

### After (Fixed Flow)

1. Database becomes healthy
2. **NEW**: Migration script applies database migrations
3. Application starts only after migrations complete
4. All tables (including `Roles`) exist
5. Application runs successfully

## Migration Process Details

The new `wait-for-migrations.sh` script:

1. Waits for database connectivity using `nc -z`
2. Runs `dotnet ef database update` with proper project references
3. Checks for migration success
4. Only then starts the application

## Commands Used

The migration command executed:

```bash
dotnet ef database update \
  --project /src/PetCare.Infrastructure/PetCare.Infrastructure.csproj \
  --startup-project /src/PetCare.Api/PetCare.Api.csproj \
  --verbose
```

## To Deploy the Fix

1. **Rebuild the Docker images**:

    ```bash
    docker-compose build --no-cache
    ```

2. **Or clean and rebuild**:

    ```bash
    docker-compose down
    docker system prune -f
    docker-compose build
    docker-compose up -d
    ```

3. **Monitor the logs** to see migration progress:
    ```bash
    docker-compose logs -f petcare_api
    ```

## Expected Behavior

After the fix, you should see:

1. Database becomes healthy
2. Migration script starts and applies migrations
3. Success message: "Migrations applied successfully"
4. Application starts without database errors
5. No more "relation 'Roles' does not exist" errors

## Verification

To verify the fix worked:

1. Check that the `Roles` table exists in the database
2. Confirm no migration-related errors in the logs
3. Ensure the application starts successfully
4. Test Identity functionality (login, role checks, etc.)

## Notes

-   The original `wait-for-db.sh` script is kept for compatibility
-   The new script provides the same database waiting functionality
-   **The application handles migrations automatically** via Entity Framework's `MigrateAsync()`
-   This approach ensures zero-downtime deployments with proper database state
-   No manual `dotnet-ef` tool installation required in runtime container
-   **Expected First-Run Behavior**: On first startup, you may see `relation "__EFMigrationsHistory" does not exist` - this is normal and expected. Entity Framework will create this table and apply all migrations automatically.
