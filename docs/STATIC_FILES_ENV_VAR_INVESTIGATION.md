# Investigation: Django STATIC_ROOT/STATICFILES_DIRS for NixOS Deployment

**Task ID:** 39dbd8f0-ae71-4bce-b7c3-f8a18ee38853  
**Date:** 2026-01-20

## Executive Summary

This investigation examined how to make Django's static file configuration dynamic for NixOS deployment where the Python project is mounted on a read-only filesystem. The key finding is that only `STATIC_ROOT` needs to be configurable via environment variable - `STATICFILES_DIRS` can remain hardcoded as it points to read-only source files.

## Current Configuration

**Location:** `config/django_settings.py` (Lines 188-193)

```python
STATIC_ROOT = "staticfiles"
STATIC_URL = "/static/"
STATICFILES_DIRS = ["frontend/static/"]
```

**Usage in `urls.py` (Line 284):**
```python
urlpatterns.extend(static(settings.STATIC_URL, document_root=settings.STATIC_ROOT))
```

## Problem Statement

On NixOS:
- Python packages are installed in `/nix/store/...` which is **immutable/read-only**
- Django's `collectstatic` command needs to **write** to `STATIC_ROOT`
- Current hardcoded `STATIC_ROOT = "staticfiles"` resolves inside the read-only package directory
- This causes `collectstatic` to fail with permission errors

## Analysis

### What Needs to Change

| Setting | Current | Needs Change? | Reason |
|---------|---------|---------------|--------|
| `STATIC_ROOT` | `"staticfiles"` | **Yes** | collectstatic writes here |
| `STATICFILES_DIRS` | `["frontend/static/"]` | No | Only read from, ships with package |
| `STATIC_URL` | `"/static/"` | No | URL path, not filesystem |

### Existing Environment Variable Pattern

From `config/settings.py`:
```python
from os import getenv

# Pattern for optional string with default:
S3_REGION_NAME: str = getenv("S3_REGION_NAME", "us-east-1")

# Pattern for required string:
DOMAIN_NAME: str = getenv("DOMAIN_NAME")
```

## Recommended Implementation

### 1. Add to `config/settings.py`

Add after line 46 (after `DOMAIN_NAME`):

```python
#
# Static files configuration
#

# The directory where Django's collectstatic command will place collected static files.
# On read-only deployments (like NixOS), set this to a writable path.
#   Example: STATIC_ROOT=/var/lib/beiwe/staticfiles
# For development or standard deployments, the default "staticfiles" works fine.
STATIC_ROOT_PATH: str = getenv("STATIC_ROOT", "staticfiles")
```

### 2. Modify `config/django_settings.py`

Update the import (line 11):
```python
from config.settings import DOMAIN_NAME, FLASK_SECRET_KEY, SENTRY_ELASTIC_BEANSTALK_DSN, STATIC_ROOT_PATH
```

Update static files section (lines 191-193):
```python
####################################################################################################
################################### Django Static Files ############################################
####################################################################################################

STATIC_ROOT = STATIC_ROOT_PATH  # Configurable via STATIC_ROOT env var
STATIC_URL = "/static/"
STATICFILES_DIRS = ["frontend/static/"]
```

## NixOS Deployment Example

### Environment Variable
```bash
export STATIC_ROOT=/var/lib/beiwe/staticfiles
```

### NixOS Module Configuration (pseudocode)
```nix
{
  services.beiwe = {
    environment = {
      STATIC_ROOT = "/var/lib/beiwe/staticfiles";
      # ... other env vars
    };
  };
  
  # Create writable directory
  systemd.tmpfiles.rules = [
    "d /var/lib/beiwe/staticfiles 0755 beiwe beiwe -"
  ];
}
```

### Deployment Flow
1. NixOS module sets `STATIC_ROOT=/var/lib/beiwe/staticfiles`
2. Service starts, Django reads environment variable
3. Run `manage.py collectstatic --noinput` (as part of deployment)
4. Static files are written to `/var/lib/beiwe/staticfiles/`
5. Nginx serves `/static/` from that directory

### Nginx Configuration
```nginx
location /static/ {
    alias /var/lib/beiwe/staticfiles/;
}
```

## Backward Compatibility

- **No changes required for existing deployments**
- Default value `"staticfiles"` preserves current behavior
- Only deployments needing a different path need to set the env var

## Migration Considerations

1. **No database migrations needed** - this is configuration only
2. **No code changes in application logic** - Django handles the paths internally
3. **Existing Elastic Beanstalk deployments** - continue to work with default value
4. **New NixOS deployments** - set `STATIC_ROOT` env var and run `collectstatic`

## Files to Modify

1. `config/settings.py` - Add `STATIC_ROOT_PATH` variable
2. `config/django_settings.py` - Import and use `STATIC_ROOT_PATH`

## Testing Recommendations

1. Verify default behavior unchanged (don't set env var, confirm `staticfiles` directory used)
2. Set `STATIC_ROOT=/tmp/test-static` and run `collectstatic`, verify files appear there
3. Test that Django serves static files correctly from custom path
