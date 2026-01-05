# Quick Reference: MariaDB 12.x Changes

## Version Update
```yaml
# OLD (defaults/main.yml)
mariadb_version: "10.11"

# NEW - Update to:
mariadb_version: "12.1"  # or "11.8" for LTS
```

## Critical Configuration Changes

### 1. Disable Query Cache (Deprecated)
```yaml
mariadb_config_overrides:
  mariadb:
    query_cache_type: 0
    query_cache_size: 0
```

### 2. Enable Thread Pool
```yaml
mariadb_config_overrides:
  mariadb:
    thread_handling: "pool-of-threads"
    thread_pool_size: "{{ ansible_processor_vcpus }}"
```

### 3. Set UTF8MB4 Default
```yaml
mariadb_charset_server: "utf8mb4"
mariadb_collation_server: "utf8mb4_unicode_ci"
```

### 4. Dynamic Galera Threads
```yaml
# Instead of hardcoded:
wsrep_slave_threads: 8

# Use:
wsrep_slave_threads: "{{ ansible_processor_vcpus }}"
```

### 5. Larger InnoDB Log Files
```yaml
innodb_log_file_size: "2G"  # Up from 1G
```

## Files to Update

1. **defaults/main.yml** - Change default version
2. **examples/playbook_airgap.yml** - Use new optimized version
3. **Review**: `RECOMMENDATIONS_MARIADB12.md`

## Verification Commands

```bash
# Check MariaDB version
mariadb --version

# Verify io_uring is enabled
mariadb -e "SHOW GLOBAL STATUS LIKE 'innodb_use_%';"

# Check Galera status
mariadb -e "SHOW STATUS LIKE 'wsrep_%';"

# Verify thread pool
mariadb -e "SHOW STATUS LIKE 'Threadpool%';"
```

## See Full Details
- **RECOMMENDATIONS_MARIADB12.md** - Complete guide
- **examples/playbook_airgap_mariadb12_optimized.yml** - Reference implementation
