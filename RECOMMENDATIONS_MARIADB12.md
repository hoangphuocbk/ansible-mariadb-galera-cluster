# MariaDB 12.x Recommendations for Air-Gap Installation

## Summary
Based on the latest MariaDB releases and best practices for 2026, here are recommendations for your ansible-mariadb-galera-cluster role.

## Version Recommendations

### Current State
- Your playbook uses: **MariaDB 12.1.2-GA** ✅ (Good choice!)
- Default in `defaults/main.yml`: **10.11** ⚠️ (Outdated)

### Recommended Versions
1. **Production/LTS**: MariaDB **11.8 LTS** (released June 2025)
   - Long-term support
   - Stable and battle-tested
   - Includes MariaDB Vector support

2. **Latest GA**: MariaDB **12.1.2** or **12.2.x** (Current choice is good)
   - Rolling GA releases
   - Better performance and scalability
   - Native io_uring support
   - Enhanced InnoDB improvements

3. **Future LTS**: MariaDB **12.3 LTS** (Expected Q2 2026)
   - Wait for stable release before production use

### Action Items
- [ ] Update `defaults/main.yml` default version from `10.11` to `12.1` or `11.8`
- [ ] Consider providing version-specific variable files in `vars/`

---

## Configuration Improvements

### 1. Query Cache (DEPRECATED)
**Issue**: Query cache is deprecated and should be disabled in modern MariaDB.

**Current Config** (in playbook example):
```yaml
# defaults/main.yml still has:
query_cache_limit: "1M"
query_cache_size: "16M"
```

**Recommended Action**:
```yaml
# Disable query cache for MariaDB 12.x
mariadb_mysql_settings:
  query_cache_type: 0
  query_cache_size: 0
```

### 2. InnoDB io_uring Support
**Current**: You have `LimitMEMLOCK=infinity` ✅ (Good!)

**New in MariaDB 12**: `innodb_linux_aio` parameter
- Automatically switches between io_uring and libaio
- No manual configuration needed

**Recommended Addition** to `mariadb_config_overrides`:
```yaml
mariadb_config_overrides:
  mariadb:
    # io_uring is auto-detected in MariaDB 12
    # innodb_linux_aio: "auto"  # Default, no need to set
    innodb_use_native_aio: "ON"  # Ensure enabled
```

### 3. InnoDB Buffer Pool
**Current**: `10G` (hardcoded in example)

**Best Practice for 2026**:
```yaml
# For systems with 16GB+ RAM
mariadb_innodb_buffer_pool_size: "{{ (ansible_memtotal_mb * 0.75) | int }}M"

# Additional MariaDB 12 optimizations
mariadb_config_overrides:
  mariadb:
    innodb_buffer_pool_instances: "{{ [ansible_processor_vcpus | int, 8] | min }}"
    innodb_buffer_pool_chunk_size: "128M"
```

### 4. InnoDB Log File Size
**Current**: `innodb_log_file_size: "1G"`

**MariaDB 12 Change**: Deprecated in favor of `innodb_log_file_size` -> consider using new sizing
```yaml
# For write-heavy workloads with MariaDB 12+
innodb_log_file_size: "2G"  # Can go higher for modern systems
innodb_log_buffer_size: "64M"
```

### 5. Thread Handling
**Missing**: Pool of threads for better concurrency

**Recommended Addition**:
```yaml
mariadb_config_overrides:
  mariadb:
    thread_handling: "pool-of-threads"
    thread_pool_size: "{{ ansible_processor_vcpus }}"
    thread_pool_max_threads: 2000
```

### 6. Galera Optimizations for MariaDB 12
**Current**: Basic Galera settings

**Enhanced Configuration**:
```yaml
mariadb_config_overrides:
  mariadb:
    # Galera Performance
    wsrep_slave_threads: "{{ ansible_processor_vcpus }}"  # Not hardcoded 8
    wsrep_provider_options: "gcache.size=2G;gcache.page_size=1G"
    
    # MariaDB 12 specific
    wsrep_trx_fragment_size: 1M
    wsrep_trx_fragment_unit: "bytes"
```

### 7. Character Set
**Current**: Defaults to `latin1` if not set

**Recommended**:
```yaml
mariadb_charset_server: "utf8mb4"
mariadb_collation_server: "utf8mb4_unicode_ci"
mariadb_charset_client: "utf8mb4"
```

---

## Security Enhancements

### 1. Disable Symbolic Links
```yaml
mariadb_config_overrides:
  mariadb:
    symbolic_links: 0
```

### 2. Enable Binary Logging (for PITR)
```yaml
mariadb_config_overrides:
  mariadb:
    log_bin: "/var/log/mysql/mysql-bin"
    expire_logs_days: 7
    sync_binlog: 1
```

### 3. SSL/TLS Enforcement (if using TLS)
Already supported via `mariadb_tls_files` ✅

---

## Air-Gap Specific Recommendations

### 1. Dependency Packages
**Current**: Tries to install via package manager (may fail)

**Improved Approach**: Create a variable for pre-staged dependencies
```yaml
# In defaults/main.yml
mariadb_airgap_dependencies_dir: ""  # Path to pre-staged .deb or .rpm files

# In tasks/install_tarball.yml
- name: install_tarball | install dependencies from local files
  ansible.builtin.apt:
    deb: "{{ mariadb_airgap_dependencies_dir }}/{{ item }}"
  with_items:
    - libaio1_*.deb
    - numactl_*.deb
  when: 
    - mariadb_airgap_dependencies_dir | length > 0
    - ansible_os_family == "Debian"
```

### 2. Memory Allocator
**Current**: Path to jemalloc

**Improvement**: Add TCMalloc as alternative
```yaml
# Auto-detect if jemalloc or tcmalloc is available
mariadb_malloc_lib: "{{ mariadb_malloc_lib_override | default(lookup('first_found', malloc_paths, errors='ignore')) }}"

malloc_paths:
  - /usr/lib/x86_64-linux-gnu/libjemalloc.so.2
  - /usr/lib/x86_64-linux-gnu/libtcmalloc.so.4
```

---

## Monitoring & Observability

### Performance Schema (Already enabled ✅)
**Enhancement**:
```yaml
mariadb_config_overrides:
  mariadb:
    performance_schema: "ON"
    performance_schema_consumer_events_statements_current: "ON"
    performance_schema_consumer_events_statements_history: "ON"
```

### Slow Query Log Improvements
```yaml
mariadb_config_overrides:
  mariadb:
    slow_query_log: 1
    long_query_time: 2
    log_slow_verbosity: "query_plan,explain"  # MariaDB 10.x+
    log_queries_not_using_indexes: 0  # Set to 1 for dev
```

---

## Systemd Service Enhancements

### 1. Add Restart Limits
```jinja2
# In templates/etc/mariadb.service.j2
[Service]
# ... existing config ...

# Prevent restart storms
StartLimitInterval=600
StartLimitBurst=3
```

### 2. Add Watchdog Support (MariaDB 12)
```jinja2
# Enable systemd watchdog
WatchdogSec=60
```

---

## Summary of Priority Actions

### High Priority ✅
1. **Update default version** in `defaults/main.yml` to `12.1` or `11.8`
2. **Disable query cache** for all MariaDB 12.x deployments
3. **Set UTF8MB4** as default character set
4. **Add thread pool** configuration for better concurrency
5. **Dynamic wsrep_slave_threads** based on CPU count

### Medium Priority ⚙️
1. Add pre-staged dependencies support for strict air-gap
2. Auto-detect memory allocator (jemalloc/tcmalloc)
3. Add binary logging for PITR capability
4. Enhanced slow query logging

### Low Priority 📋
1. Add watchdog support in systemd
2. Version-specific vars files
3. Enhanced monitoring configurations

---

## Compatibility Notes

- **MariaDB 12.x** requires:
  - Linux kernel 5.10+ for optimal io_uring support
  - systemd 243+ for best systemd integration
  - Ubuntu 22.04+ or RHEL 8+ recommended

- **Galera 4.x** is bundled with MariaDB 12.x
  - No separate installation needed in tarball

---

## Testing Checklist

Before deploying to production:
- [ ] Test cluster bootstrap with MariaDB 12.x
- [ ] Verify io_uring is active: `SHOW GLOBAL STATUS LIKE 'innodb_use_%';`
- [ ] Check Galera sync: `SHOW STATUS LIKE 'wsrep_%';`
- [ ] Validate thread pool: `SHOW STATUS LIKE 'Threadpool%';`
- [ ] Monitor memory usage with jemalloc/tcmalloc
- [ ] Test SST with mariabackup (if using TLS)
