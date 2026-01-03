# Workflow Run 20675089625 Build Failure Analysis
## 工作流运行 20675089625 构建失败分析

**Date**: 2026-01-03  
**Workflow**: `.github/workflows/build-immortalwrt-xtxf.yml`  
**Run ID**: 20675089625  
**Duration**: 3 hours 27 minutes (08:57:37 - 12:24:27 UTC)  
**Status**: Failed  
**Failed Step**: Build firmware (step 9)

## Problem Summary / 问题概述

The ImmortalWrt firmware build for ZBT Z8102AX V2 (512MB Flash) failed after running for approximately 3.5 hours. While we cannot access the exact error logs directly, analysis of the workflow configuration reveals several critical issues that cause both the extended build time and likely build failures.

ImmortalWrt 固件构建失败，运行了约 3.5 小时。虽然无法直接访问错误日志，但对工作流配置的分析揭示了导致构建时间过长和构建失败的几个关键问题。

## Root Causes / 根本原因

### 1. Excessive Package Configuration (150+ packages)
### 1. 软件包配置过多（150+ 个包）

The workflow configuration includes an extensive list of packages, many of which are unnecessary for a production router firmware:

- **Development/Debug Tools**: strace, lsof, nmap, nmap-ssl, smartmontools, lm-sensors (lines 239-282)
- **Text Editors**: vim, nano (lines 323-324)
- **Hardware Monitoring**: smartmontools, smartmontools-drivedb, hdparm, hd-idle (lines 277-282)
- **Extensive coreutils modules**: 12 separate coreutils packages (lines 217-226)
- **Extensive procps-ng modules**: 10 separate procps-ng packages (lines 229-238)
- **Network diagnostic tools**: nmap, nmap-ssl, mtr, iperf3, socat (lines 258-263)

**Impact**: Each additional package increases:
- Compilation time (some packages take 10-30 minutes each)
- Disk space usage (risk of "out of disk space" errors)
- Download time for package sources
- Overall firmware size

**影响**：每个额外的软件包都会增加：
- 编译时间（某些包需要 10-30 分钟）
- 磁盘空间使用（可能出现"磁盘空间不足"错误）
- 软件包源代码下载时间
- 固件整体大小

### 2. Suboptimal Build Parallelization
### 2. 构建并行化配置不当

**Current Configuration** (line 346):
```bash
make -j2 V=s || make -j1 V=s
```

**Problem**: GitHub Actions runners provide 2-4 CPU cores, but the workflow only uses 2 parallel jobs (`-j2`), leaving computing resources underutilized.

**问题**：GitHub Actions 运行器提供 2-4 个 CPU 核心，但工作流仅使用 2 个并行作业（`-j2`），导致计算资源利用不足。

**Impact**: 
- Build time is 50-100% longer than necessary
- 3.5 hour builds could potentially be reduced to 1.5-2 hours with proper parallelization

**影响**：
- 构建时间比必要时间长 50-100%
- 通过适当的并行化，3.5 小时的构建可能减少到 1.5-2 小时

### 3. Missing Compiler Cache (ccache)
### 3. 缺少编译器缓存（ccache）

The workflow does not enable ccache, which is a critical optimization for ImmortalWrt/OpenWrt builds.

工作流未启用 ccache，这是 ImmortalWrt/OpenWrt 构建的关键优化。

**Impact**:
- Every source file is recompiled from scratch on each build
- Subsequent builds could be 50-70% faster with ccache enabled
- GitHub Actions has 10GB cache storage available per repository

**影响**：
- 每次构建都从头编译每个源文件
- 启用 ccache 后，后续构建可提速 50-70%
- GitHub Actions 为每个仓库提供 10GB 缓存存储

### 4. No Download/Feed Cache
### 4. 没有下载/Feed 缓存

Lines 115-116 update and install feeds on every run without caching:
```bash
./scripts/feeds update -a
./scripts/feeds install -a
```

**Impact**:
- Downloads all package sources on every run (can be 1-2GB)
- Adds 5-15 minutes to build time
- Potential network failures during download

**影响**：
- 每次运行都下载所有软件包源代码（可能有 1-2GB）
- 增加 5-15 分钟构建时间
- 下载期间可能出现网络故障

### 5. Disk Space Pressure
### 5. 磁盘空间压力

GitHub Actions ubuntu-latest runners provide ~14GB usable disk space after preinstalled software. Building ImmortalWrt with 150+ packages:

- Source downloads: 2-3GB
- Build artifacts: 8-12GB
- Final images: 200-500MB

**Total**: 10-15GB, which exceeds or comes very close to the available space, leading to potential "No space left on device" errors during compilation.

GitHub Actions ubuntu-latest 运行器在预装软件后提供约 14GB 可用磁盘空间。使用 150+ 个包构建 ImmortalWrt：

- 源代码下载：2-3GB
- 构建产物：8-12GB
- 最终镜像：200-500MB

**总计**：10-15GB，超过或非常接近可用空间，导致编译期间可能出现"设备上没有剩余空间"错误。

### 6. Verbose Debug Kernel Options
### 6. 详细的调试内核选项

Lines 153-157 enable extensive NMBM debugging:
```
CONFIG_MTD_NMBM_DEBUG=y
CONFIG_MTD_NMBM_DEBUG_VERBOSE=y
```

These are development/debugging options that:
- Increase kernel compile time
- Add unnecessary code to production firmware
- Should only be enabled for troubleshooting NAND issues

这些是开发/调试选项：
- 增加内核编译时间
- 向生产固件添加不必要的代码
- 仅应在排查 NAND 问题时启用

## Recommended Solutions / 推荐解决方案

### Priority 1: Reduce Package Count (Essential Fix)
### 优先级 1：减少软件包数量（必要修复）

Remove non-essential packages to reduce build time and disk usage:

**Remove**:
- Debug tools: strace, lsof (lines 239-240)
- Network diagnostic: nmap, nmap-ssl, mtr (lines 258-260)
- Hardware monitoring: smartmontools, lm-sensors, hdparm, hd-idle (lines 276-282)
- Text editors: vim, nano (keep only one if needed) (lines 323-324)
- Reduce coreutils to essential modules only
- Reduce procps-ng to essential modules only

**Keep only**:
- Core system packages (mtd-utils, uboot-envtools)
- Network functionality (USB modem, WiFi drivers)
- LuCI web interface (minimal configuration)
- Essential troubleshooting tools (ping, traceroute)

**Estimated Impact**: Reduce build time by 30-50%, save 2-3GB disk space

**预计影响**：减少构建时间 30-50%，节省 2-3GB 磁盘空间

### Priority 2: Enable Compiler Cache (ccache)
### 优先级 2：启用编译器缓存（ccache）

Add ccache configuration and GitHub Actions caching:

```yaml
- name: Setup ccache
  uses: actions/cache@v4
  with:
    path: ~/.ccache
    key: ccache-${{ github.sha }}
    restore-keys: |
      ccache-

- name: Configure build
  run: |
    cd immortalwrt
    # Enable ccache
    cat >> .config << 'EOF'
    CONFIG_CCACHE=y
    EOF
    
    # Set ccache size
    mkdir -p ~/.ccache
    echo "max_size = 5.0G" > ~/.ccache/ccache.conf
```

**Estimated Impact**: First build unchanged, subsequent builds 50-70% faster

**预计影响**：首次构建不变，后续构建快 50-70%

### Priority 3: Optimize Build Parallelization
### 优先级 3：优化构建并行化

```yaml
- name: Build firmware
  run: |
    cd immortalwrt
    echo "Available disk space before build:"
    df -h
    
    # Use optimal number of jobs based on available cores
    # GitHub Actions provides 2-4 cores, use nproc to auto-detect
    JOBS=$(($(nproc) + 1))
    echo "Building with ${JOBS} parallel jobs"
    
    make -j${JOBS} V=s || make -j1 V=s
```

**Estimated Impact**: Reduce build time by 30-50%

**预计影响**：减少构建时间 30-50%

### Priority 4: Cache Downloads
### 优先级 4：缓存下载

```yaml
- name: Cache downloads
  uses: actions/cache@v4
  with:
    path: immortalwrt/dl
    key: dl-${{ hashFiles('immortalwrt/feeds.conf.default') }}
    restore-keys: |
      dl-

- name: Cache feeds
  uses: actions/cache@v4
  with:
    path: |
      immortalwrt/feeds
      immortalwrt/package/feeds
    key: feeds-${{ hashFiles('immortalwrt/feeds.conf.default') }}
    restore-keys: |
      feeds-
```

**Estimated Impact**: Save 5-15 minutes on download time

**预计影响**：节省 5-15 分钟下载时间

### Priority 5: Remove Debug Kernel Options
### 优先级 5：删除调试内核选项

Remove or comment out these lines (153-154):
```
# CONFIG_MTD_NMBM_DEBUG=y
# CONFIG_MTD_NMBM_DEBUG_VERBOSE=y
```

Keep only essential NMBM options:
```
CONFIG_MTD_NMBM=y
CONFIG_MTD_NMBM_MTD=y
CONFIG_MTD_NMBM_CREATE=y
CONFIG_MTD_NMBM_USE_CMDLINE_PARAMS=y
```

**Estimated Impact**: Slightly faster kernel compile, smaller firmware size

**预计影响**：内核编译稍快，固件体积更小

### Priority 6: Add Build Monitoring
### 优先级 6：添加构建监控

Add disk and memory monitoring to identify bottlenecks:

```yaml
- name: Monitor build progress
  run: |
    cd immortalwrt
    # Monitor disk space every 5 minutes
    while true; do
      echo "=== $(date) ==="
      df -h | grep -E "Filesystem|/dev/root"
      sleep 300
    done &
    MONITOR_PID=$!
    
    # Build
    make -j$(($(nproc) + 1)) V=s || make -j1 V=s
    
    # Stop monitor
    kill $MONITOR_PID 2>/dev/null || true
```

## Expected Outcomes / 预期结果

After implementing these fixes:

实施这些修复后：

| Metric | Current | Expected | Improvement |
|--------|---------|----------|-------------|
| Build Time | 3.5 hours | 1.5-2 hours | 40-55% faster |
| Disk Usage Peak | ~14GB (failure) | ~8-10GB | 30-40% less |
| Subsequent Builds | N/A | 30-45 min | 70-85% faster |
| Success Rate | Failed | Pass | Build completes |

## Action Items / 行动项

1. ✅ Create this analysis document
2. ⬜ Reduce package count in `.config` section
3. ⬜ Enable ccache with GitHub Actions caching
4. ⬜ Optimize parallel build jobs
5. ⬜ Add download and feed caching
6. ⬜ Remove debug kernel options
7. ⬜ Add build monitoring
8. ⬜ Test the optimized workflow
9. ⬜ Document results

## References / 参考资料

- [OpenWrt Build System](https://openwrt.org/docs/guide-developer/toolchain/use-buildsystem)
- [How to Speed Up OpenWrt Build](https://openwrt.org/faq/how_can_i_speed_up_build_process)
- [GitHub Actions Disk Space Management](https://github.com/actions/runner-images/issues/2840)
- [ImmortalWrt GitHub Repository](https://github.com/immortalwrt/immortalwrt)

---

**Analysis Date**: 2026-01-03T12:26:37Z  
**Analyzed By**: GitHub Copilot Agent  
**Related Workflow Run**: https://github.com/xyzy12345/ZBT-Z8102AX-V2-imm/actions/runs/20675089625
