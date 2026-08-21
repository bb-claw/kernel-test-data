# Kernel Bug Findings

Kernel bugs and regressions found by the kernel-test harness.
Tooling/harness issues are tracked in the [harness repo FINDINGS.md](https://github.com/bb-claw/kernel-test/blob/main/FINDINGS.md).

---

## 2026-07-18 — Kernel Bugs Found by Random-Config Testing

### High — Build Failure

- [x] **PINCTRL_MICROCHIP_SGPIO missing `select REGMAP_MMIO` — build fails without regmap** ✅ patch in pinctrl-fixes tree 2026-08-02, awaiting merge to mainline
  Kernel: v7.2-rc2 and v7.2-rc3. Arch: arm64 (affects all arches). Found by rand500config sampling.

  `pinctrl-microchip-sgpio.c` includes `<linux/mfd/ocelot.h>` which calls
  `ocelot_regmap_from_resource()` → `devm_regmap_init_mmio()`. Both the function and
  `struct regmap_config` are guarded by `#ifdef CONFIG_REGMAP` in `<linux/regmap.h>`, and
  `CONFIG_REGMAP` is only auto-selected when `CONFIG_REGMAP_MMIO` is selected. The Kconfig
  entry for `PINCTRL_MICROCHIP_SGPIO` is missing `select REGMAP_MMIO`, so a random config
  that enables the driver without independently enabling `REGMAP_MMIO` fails to build.

  **Build errors:**
  ```
  include/linux/mfd/ocelot.h:34: error: implicit declaration of function 'devm_regmap_init_mmio'
  drivers/pinctrl/pinctrl-microchip-sgpio.c:910: error: variable 'regmap_config' has initializer but incomplete type
  drivers/pinctrl/pinctrl-microchip-sgpio.c:911: error: 'struct regmap_config' has no member named 'reg_bits'
  ```

  **Note:** `# CONFIG_REGMAP_BUILD is not set` in the trigger config is a red herring —
  `REGMAP_BUILD` exists only for KUnit testing and has no bearing on the actual regmap library.
  A config with both `PINCTRL_MICROCHIP_SGPIO=y` and `REGMAP_MMIO=y` (selected by something
  else) builds successfully. The bug only triggers when `REGMAP_MMIO` is absent.

  **Comparison:** `PINCTRL_OCELOT` uses the same `ocelot.h` header and correctly has
  `select REGMAP_MMIO`. `PINCTRL_INGENIC`, `PINCTRL_K210`, `PINCTRL_K230` also correctly
  `select REGMAP_MMIO`. `PINCTRL_MICROCHIP_SGPIO` is the only one missing it.

  **Trigger config:** `configs/archive_failed/kconfig-rand500config-arm64-v7.2-rc2-edfe557442df5e93de92b3b3cca7c8a36183e28da1169bd8c7112e462b33b42a-BUILD_FAIL.config`

  **Reproduce:** `make checkout TAG=v7.2-rc2 && make replay CONFIG_FILE=<above>`

  **Fix** — `drivers/pinctrl/Kconfig` (tab-indented, same as sibling drivers):
  ```diff
   config PINCTRL_MICROCHIP_SGPIO
  +	select REGMAP_MMIO
  ```

  **Fix confirmed:** replay with original failing config after applying the patch:
  - SHA changed: `edfe557442df5e93...` → `60276c6208800aca...` (olddefconfig auto-added REGMAP_MMIO=y)
  - Build: PASS, Boot: PASS, Tests: 26/26 — v7.2-rc2 arm64

  **Patch — not yet submitted to mailing list (2026-07-18)**

  Subsystem: PIN CONTROL (`drivers/pinctrl/`). Introduced by commit
  `68c873363a78` ("pinctrl: microchip-sgpio: add ability to be used in a
  non-mmio configuration"), present since 2022, affects all stable branches.

  Recipients:
  ```
  To:  Linus Walleij <linusw@kernel.org>
  Cc:  Steen.Hegelund@microchip.com
  Cc:  daniel.machon@microchip.com
  Cc:  UNGLinuxDriver@microchip.com
  Cc:  linux-gpio@vger.kernel.org
  Cc:  linux-arm-kernel@lists.infradead.org
  Cc:  linux-kernel@vger.kernel.org
  Cc:  stable@vger.kernel.org
  ```

  Commit message:
  ```
  Subject: [PATCH] pinctrl: microchip-sgpio: add missing select REGMAP_MMIO

  The driver includes <linux/mfd/ocelot.h>, which calls
  ocelot_regmap_from_resource() -> devm_regmap_init_mmio(). Both the
  function and struct regmap_config are guarded by #ifdef CONFIG_REGMAP
  in <linux/regmap.h>. CONFIG_REGMAP is only auto-selected when something
  selects CONFIG_REGMAP_MMIO.

  Without 'select REGMAP_MMIO' in the Kconfig entry, a config that
  enables PINCTRL_MICROCHIP_SGPIO without any other driver pulling in
  REGMAP_MMIO fails to build:

    include/linux/mfd/ocelot.h:19:51: warning: 'struct regmap_config'
      declared inside parameter list will not be visible outside of
      this definition or declaration
    include/linux/mfd/ocelot.h:34:24: error: implicit declaration of
      function 'devm_regmap_init_mmio'
    drivers/pinctrl/pinctrl-microchip-sgpio.c:910:16: error: variable
      'regmap_config' has initializer but incomplete type
    drivers/pinctrl/pinctrl-microchip-sgpio.c:911:18: error: 'struct
      regmap_config' has no member named 'reg_bits'

  PINCTRL_OCELOT uses the same ocelot.h header and correctly has
  'select REGMAP_MMIO'. Fix PINCTRL_MICROCHIP_SGPIO the same way.

  Fixes: 68c873363a78 ("pinctrl: microchip-sgpio: add ability to be used in a non-mmio configuration")
  Cc: stable@vger.kernel.org
  Signed-off-by: Benjamin Boortz <benjamin.boortz@gmail.com>
  ---
   drivers/pinctrl/Kconfig | 1 +
   1 file changed, 1 insertion(+)

  diff --git a/drivers/pinctrl/Kconfig b/drivers/pinctrl/Kconfig
  --- a/drivers/pinctrl/Kconfig
  +++ b/drivers/pinctrl/Kconfig
  @@ -425,6 +425,7 @@ config PINCTRL_MICROCHIP_SGPIO
   	select GENERIC_PINCONF
   	select GENERIC_PINCTRL_GROUPS
   	select GENERIC_PINMUX_FUNCTIONS
  +	select REGMAP_MMIO
   	help
  ```

  Send with:
  ```sh
  cd ~/git/linux
  git add drivers/pinctrl/Kconfig && git commit
  scripts/checkpatch.pl --strict $(git format-patch -1 --stdout)
  git send-email --to='linusw@kernel.org' \
    --cc='Steen.Hegelund@microchip.com' \
    --cc='daniel.machon@microchip.com' \
    --cc='UNGLinuxDriver@microchip.com' \
    --cc='linux-gpio@vger.kernel.org' \
    --cc='linux-arm-kernel@lists.infradead.org' \
    --cc='linux-kernel@vger.kernel.org' \
    --cc='stable@vger.kernel.org' \
    $(git format-patch -1)
  ```

---

## 2026-07-21 — v7.2-rc4 rand500config Boot Failures (10-run sweep)

### High — Kernel Crash

- [ ] **`CONFIG_RCU_SCALE_TEST=y` triggers NULL pointer dereference in `rcu_scale_writer` on i386**
  Kernel: v7.2-rc4. Arch: i386. Found in 2 of 10 rand500config/i386 runs.

  Both failing configs have `CONFIG_RCU_SCALE_TEST=y`. After the scale test completes
  100 measurements the writer task crashes with a write fault (Oops code 0002) at
  address 0x00000000 — a NULL pointer write:

  ```
  BUG: kernel NULL pointer dereference, address: 00000000
  Oops: Oops: 0002 [#1]
  CPU: 0 PID: 17 Comm: rcu_scale_write  7.2.0-rc4
  EIP: rcu_scale_writer+0x497/0x4b0
  note: rcu_scale_write[17] exited with irqs disabled
  ```

  The second oops config lacks `CONFIG_KALLSYMS` so EIP resolves to a raw address
  (`0xc105fb2b`), but the sequence is identical: 100 measurements → NULL write → irqs
  disabled on exit.

  **Both configs differ significantly** (different CPU targets, different subsystems
  enabled) — the only shared factor relevant to this crash is `CONFIG_RCU_SCALE_TEST=y`.

  **Root cause:** Unknown. Likely a bug in `kernel/rcu/rcuscale.c` on i386 where
  `rcu_scale_writer` writes through a pointer that is NULL after the first measurement
  batch. Could be an i386-specific alignment or pointer arithmetic issue.

  **Trigger configs:**
  ```
  configs/archive_failed/kconfig-rand500config-i386-v7.2-rc4-22799f...BOOT_FAIL-oops.config
  configs/archive_failed/kconfig-rand500config-i386-v7.2-rc4-90748f...BOOT_FAIL-oops.config
  ```

  **Reproduce:**
  ```sh
  make replay CONFIG_FILE=configs/archive_failed/kconfig-rand500config-i386-v7.2-rc4-22799f1390815c3a0bc53894af09fecc99ae7a62245377f3f0938010604dc225-BOOT_FAIL-oops.config CONFIGS=rand500config ARCHS=i386
  ```

  **Next steps:**
  - Replay to confirm consistent reproduction
  - Check if `CONFIG_RCU_EXPERT=y` (present in run 1 config) is required to trigger
  - Check `kernel/rcu/rcuscale.c` around `rcu_scale_writer` offset 0x497 on i386
  - Consider adding `CONFIG_RCU_SCALE_TEST=n` to `configs/randconfig.config` (sibling
    to the already-excluded `CONFIG_RCU_TORTURE_TEST`) to stop this appearing in future
    sampling, pending upstream investigation

  **Subsystem:** RCU (`kernel/rcu/`). Maintainer: Paul McKenney / Joel Fernandes.
  Mailing list: `rcu@vger.kernel.org`, `linux-kernel@vger.kernel.org`.

### Low — Boot Failure (single occurrence)

- [ ] **arm64: init crashes with SIGSEGV (exitcode=0x0000000b) on one rand500config**
  Kernel: v7.2-rc4. Arch: arm64. Found in 1 of 10 runs (config SHA: `d2d7a42d...`).

  ```
  Kernel panic - not syncing: Attempted to kill init! exitcode=0x0000000b
  ```

  `exitcode=0x0000000b` = signal 11 (SIGSEGV). The toybox `/init` process crashed with
  a segfault before any tests ran. No backtrace captured (CONFIG_KALLSYMS status unknown
  for this config).

  **Not yet reproducible:** Only one occurrence. Could be a missing kernel feature that
  toybox requires (BPF, compat32, specific syscall) or a real kernel mm/exec bug.

  **Next step:** Replay the archived config and inspect full dmesg for what init was doing
  at the point of crash:
  ```sh
  make replay CONFIG_FILE=configs/archive_failed/kconfig-rand500config-arm64-v7.2-rc4-d2d7a42d5a261ceb37b934aef3b2f4e98d3a033c0b9d0146ab64d5e5383902fe-BOOT_FAIL-kernel-panic.config CONFIGS=rand500config ARCHS=arm64
  ```

---

## 2026-07-22 — Stable Kernel (7.1.x) KUnit: gpu_buddy 32-bit Bug

### High — Kernel Bug (i386-only, deterministic)

- [ ] **`gpu_test_buddy_alloc_exceeds_max_order` KUnit fails on i386 — `roundup_pow_of_two` silently truncates `u64` to 32-bit**
  Kernel: stable v7.1.3 (first found), v7.1.4 (confirmed). Also present in mainline v7.2 (`drivers/gpu/buddy.c:1356`). Arch: i386 only. Found by kunitrandconfig/i386 sweep.

  **Test failure output:**
  ```
  # KTAP version 1
  # Subtest: gpu_buddy
  not ok 15 gpu_test_buddy_alloc_exceeds_max_order
  # EXPECTATION FAILED at drivers/gpu/tests/gpu_buddy_test.c:1379
  # Expected err == -22, but err == 0 (0x0)
  # gpu_buddy_fini:474: GPU BUG: assertion `gpu_buddy_block_is_free(mm->roots[i])` failed
  # gpu_buddy_fini:482: GPU BUG: assertion `mm->avail == mm->size` failed
  not ok 16 gpu_buddy
  ```
  Result: `KUNIT_FAIL-2-of-1749` on both v7.1.3 and v7.1.4. All other 1747 KUnit tests pass.

  **Root cause — `roundup_pow_of_two` is not `u64`-safe on 32-bit systems:**

  `gpu_buddy_alloc_blocks` (`drivers/gpu/buddy.c:1321`) calls:
  ```c
  u64 size = SZ_8G + SZ_1G;   /* 0x240000000 — a 64-bit value */
  size = roundup_pow_of_two(size);  /* BUG: not u64-safe */
  ```

  `roundup_pow_of_two` dispatches to `__roundup_pow_of_two(unsigned long n)` (`include/linux/log2.h:55`).
  On i386, `unsigned long` is 32 bits. The function argument conversion silently truncates
  `0x240000000` → `0x40000000 = SZ_1G`.

  **Execution trace on i386:**
  | Step | Expected (64-bit) | Actual on i386 |
  |------|-------------------|----------------|
  | `roundup_pow_of_two(0x240000000)` | `0x400000000 = SZ_16G` | `0x40000000 = SZ_1G` (truncated) |
  | `pages = size >> 12` | `0x400000` (order=22) | `0x40000` (order=18) |
  | `order > mm->max_order(21)?` | TRUE → return -EINVAL | FALSE → continues |
  | `size > mm->size(SZ_10G)?` | TRUE → return -EINVAL | FALSE (SZ_1G < SZ_10G) → continues |
  | Return value | -EINVAL | 0 (allocation succeeds) |

  Because the guard at `buddy.c:1335–1342` is never triggered, the function allocates
  `SZ_1G` bytes instead of rejecting the over-limit request. `gpu_buddy_fini` then fires
  internal assertions because the block was never freed (lines 474, 482 are secondary effects).

  **Why x86_64 passes:** `unsigned long` is 64 bits on x86_64, so `roundup_pow_of_two` handles
  `u64` correctly and the overflow guard fires as expected.

  **Evidence — code references:**
  - Test: `drivers/gpu/tests/gpu_buddy_test.c:1350–1382` (`gpu_test_buddy_alloc_exceeds_max_order`)
  - Bug site: `drivers/gpu/buddy.c:1321` (stable), `drivers/gpu/buddy.c:1356` (mainline)
  - Truncation: `include/linux/log2.h:55` (`__roundup_pow_of_two(unsigned long n)`)
  - Guard that should fire: `drivers/gpu/buddy.c:1335–1342`

  **Trigger configs (same logical bug, different random KUnit module sets):**
  ```
  kernel-test-stable/configs/archive_failed/kconfig-kunitrandconfig-i386-v7.1.3-35376dd938df26e368915a706ef26053aec3e6bee302d3825dc0a774007f46fa-KUNIT_FAIL-2-of-1749.config
  ```
  v7.1.4 replay config SHA: `fae25d2690989975c4845d09d204d61a64e7aa9fe4b01891e291bf848aa3652c` (same failure)

  **Reproduce:**
  ```sh
  cd ~/git/kernel-test-stable
  make checkout TAG=v7.1.4
  make replay CONFIG_FILE=configs/archive_failed/kconfig-kunitrandconfig-i386-v7.1.3-35376dd938df26e368915a706ef26053aec3e6bee302d3825dc0a774007f46fa-KUNIT_FAIL-2-of-1749.config NO_FETCH=1
  # Expect: kunit:1747/1749 FAIL — gpu_test_buddy_alloc_exceeds_max_order + gpu_buddy suite
  ```

  **Fix** — `drivers/gpu/buddy.c` (same line number differs by 35 between stable and mainline):
  ```diff
  -		size = roundup_pow_of_two(size);
  +		size = BIT_ULL(fls64(size - 1));
  ```
  `BIT_ULL(fls64(size - 1))` is equivalent to `roundup_pow_of_two` but uses 64-bit
  `fls64` instead of 32-bit `fls_long`, and `1ULL` instead of `1UL`. For the test case:
  `fls64(0x23FFFFFFF) = 34` → `BIT_ULL(34) = 0x400000000 = SZ_16G` ✓

  **Subsystem:** `drivers/gpu/` (GPU memory management). Maintainers: Christian König, Thomas Hellström.
  Mailing lists: `dri-devel@lists.freedesktop.org`, `linux-kernel@vger.kernel.org`.
  Cc `stable@vger.kernel.org` — same bug present in all stable branches that carry the gpu_buddy allocator.

  **Patch recipients:**
  ```
  To:  Christian König <christian.koenig@amd.com>
  To:  Thomas Hellström <thomas.hellstrom@linux.intel.com>
  Cc:  dri-devel@lists.freedesktop.org
  Cc:  linux-kernel@vger.kernel.org
  Cc:  stable@vger.kernel.org
  ```

  **Tinyconfig reproducer:**

  The kunitrandconfig trigger config is large (1749 tests, ~100 MB kernel). A minimal
  reproducer isolates the failure to a single test suite and is better for LKML.

  *Dependency chain* — why GATE_CFGS are needed:
  ```
  GPU_BUDDY_KUNIT_TEST  →  depends on GPU_BUDDY && KUNIT
                                       |
                               GPU_BUDDY (bool, no prompt — selected-only)
                                       |
                               selected by DRM_BUDDY (tristate, no prompt — hidden)
                                       |
                               depends on DRM (menuconfig, user-selectable on i386)
                                       |
                               depends on (AGP || AGP=n) && !EMULATED_CMPXCHG && HAS_DMA
                                        ✓ all satisfied on i386
  ```
  `DRM_BUDDY` and `GPU_BUDDY` have no Kconfig `prompt` — they cannot be enabled by the
  user via `make menuconfig` and are normally only pulled in by heavy GPU drivers (i915,
  amdgpu, xe). However, `make olddefconfig` keeps any option in `.config` whose `depends on`
  chain is satisfied, regardless of whether it has a prompt. Appending `CONFIG_DRM_BUDDY=y`
  to the fragment before `olddefconfig` is enough — it stays because `depends on DRM` is met.

  *Using the harness* — one command, fully automated:
  ```sh
  # In kernel-test or kernel-test-stable, after make checkout TAG=<version>:
  make kconfig-build SUBSYSTEM=gpu ARCHS=i386 GATE_CFGS=CONFIG_DRM,CONFIG_DRM_BUDDY DRY_RUN=1
  # Preview: shows CONFIG_GPU_BUDDY and CONFIG_GPU_BUDDY_KUNIT_TEST will be swept

  make kconfig-build SUBSYSTEM=gpu ARCHS=i386 GATE_CFGS=CONFIG_DRM,CONFIG_DRM_BUDDY
  # Runs: tinyconfig + randkconfigconfig.config + DRM=y + DRM_BUDDY=y + GPU_BUDDY_KUNIT_TEST=y
  # Builds and boots each option; GPU_BUDDY_KUNIT_TEST will show KUNIT_FAIL-1/N
  ```
  The sweep generates one build per Kconfig entry in `drivers/gpu/Kconfig`. Only
  `GPU_BUDDY_KUNIT_TEST` triggers a KUnit run; `GPU_BUDDY` is a plain bool with no test.

  *Manual fragment* — for a standalone reproducer outside the harness (e.g. for LKML):
  ```sh
  # 1. Generate minimal base config for i386
  make -C ~/git/linux-stable tinyconfig ARCH=i386 O=/tmp/repro-gpu-buddy

  # 2. Apply bootability + KUnit + GPU buddy test
  cat >> /tmp/repro-gpu-buddy/.config <<'EOF'
  # Bootability (same as configs/randkconfigconfig.config)
  CONFIG_TTY=y
  CONFIG_SERIAL_8250=y
  CONFIG_SERIAL_8250_CONSOLE=y
  CONFIG_BLK_DEV_INITRD=y
  CONFIG_BINFMT_ELF=y
  CONFIG_BINFMT_SCRIPT=y
  # KUnit framework
  CONFIG_KUNIT=y
  # Gate symbols (DRM + hidden DRM_BUDDY, which selects GPU_BUDDY)
  CONFIG_DRM=y
  CONFIG_DRM_BUDDY=y
  # Target test
  CONFIG_GPU_BUDDY_KUNIT_TEST=y
  EOF

  # 3. Resolve dependencies (olddefconfig keeps DRM_BUDDY=y because depends on DRM is met)
  make -C ~/git/linux-stable ARCH=i386 O=/tmp/repro-gpu-buddy olddefconfig

  # 4. Verify the key options survived
  grep -E "CONFIG_(DRM|GPU_BUDDY|KUNIT)" /tmp/repro-gpu-buddy/.config

  # 5. Build
  make -C ~/git/linux-stable ARCH=i386 O=/tmp/repro-gpu-buddy -j$(nproc) bzImage

  # 6. Boot in QEMU (minimal — no initramfs needed, KUnit runs before /init)
  qemu-system-i386 -kernel /tmp/repro-gpu-buddy/arch/x86/boot/bzImage \
    -append "console=ttyS0 earlycon=uart8250,io,0x3f8 panic=5" \
    -serial stdio -display none -no-reboot -m 512
  ```

  *Expected output in QEMU serial log:*
  ```
  KTAP version 1
  # Subtest: gpu_buddy
  ok 1 gpu_test_buddy_alloc_limit
  ...
  not ok 15 gpu_test_buddy_alloc_exceeds_max_order
  # EXPECTATION FAILED at drivers/gpu/tests/gpu_buddy_test.c:1379
  # Expected err == -22, but err == 0 (0x0)
  not ok 16 gpu_buddy
  ```
  Only 16 test cases run (the full `gpu_buddy` suite), compared to 1749 in kunitrandconfig —
  output is unambiguous and the failure stands alone.

  **Commit message:**
  ```
  Subject: [PATCH] drm/buddy: fix roundup_pow_of_two() truncation on 32-bit arches

  gpu_buddy_alloc_blocks() rounds the requested allocation size up to
  the nearest power of two using roundup_pow_of_two(), which internally
  calls __roundup_pow_of_two(unsigned long). On 32-bit architectures,
  unsigned long is 32 bits, so passing a u64 value larger than UINT32_MAX
  silently truncates it before the rounding.

  With mm_size = SZ_8G + SZ_2G and a CONTIGUOUS|RANGE request for
  SZ_8G + SZ_1G:

    Expected: roundup_pow_of_two(0x240000000) = 0x400000000 (SZ_16G)
    Actual:   roundup_pow_of_two(0x040000000) = 0x040000000 (SZ_1G)
              (0x240000000 truncated to 32-bit → 0x040000000)

  After truncation, order=18 does not exceed max_order=21 and size=SZ_1G
  does not exceed mm->size=SZ_10G, so the -EINVAL guard at line 1335 is
  never reached. The allocation succeeds, and gpu_buddy_fini() subsequently
  fires internal assertions because the block was never freed.

  This is caught by the KUnit test gpu_test_buddy_alloc_exceeds_max_order,
  which fails on i386 with:
    Expected err == -22, but err == 0
    gpu_buddy_fini:474: GPU BUG: assertion failed
    gpu_buddy_fini:482: GPU BUG: assertion failed

  Fix by using BIT_ULL(fls64(size - 1)) instead of roundup_pow_of_two(),
  which performs the equivalent operation in 64 bits on all architectures.

  Fixes: <commit that introduced gpu_buddy_alloc_blocks with CONTIGUOUS path>
  Cc: stable@vger.kernel.org
  Signed-off-by: Benjamin Boortz <benjamin.boortz@gmail.com>
  ---
   drivers/gpu/buddy.c | 2 +-
   1 file changed, 1 insertion(+), 1 deletion(-)

  diff --git a/drivers/gpu/buddy.c b/drivers/gpu/buddy.c
  --- a/drivers/gpu/buddy.c
  +++ b/drivers/gpu/buddy.c
  @@ -1318,7 +1318,7 @@ int gpu_buddy_alloc_blocks(...)
   	/* Roundup the size to power of 2 */
   	if (flags & GPU_BUDDY_CONTIGUOUS_ALLOCATION) {
  -		size = roundup_pow_of_two(size);
  +		size = BIT_ULL(fls64(size - 1));
   		min_block_size = size;
  ```

  **Status:** Not yet submitted. Find the exact `Fixes:` commit with:
  ```sh
  cd ~/git/linux
  git log --oneline drivers/gpu/buddy.c | grep -i "contiguous\|round\|power"
  ```

---

## 2026-07-22 — v7.2-rc4 rand500config Boot Failure: DEBUG_TEST_DRIVER_REMOVE + IIO drivers

### Medium — Boot Failure (complex N-way interaction, not yet actionable for LKML)

- [ ] **`CONFIG_DEBUG_TEST_DRIVER_REMOVE=y` + multiple IIO drivers causes BOOT_FAIL on i386 — minimal reproducer not isolated**
  Kernel: v7.2-rc4. Arch: i386. Found by config bisect (`make bisect`) on
  `kconfig-rand500config-i386-v7.2-rc4-1130034ad8bb931d733ad604ddf6a0eec3d97fa5574b93987a3fa55fa933cb89-BOOT_FAIL-timeout.config`.

  **What `CONFIG_DEBUG_TEST_DRIVER_REMOVE` does:** After each successful `->probe()`, immediately
  calls `->remove()` then re-probes the driver. This stress-tests remove/re-probe paths and can
  expose deadlocks or infinite loops in drivers that handle remove incorrectly.

  **Failure symptom:** `Did not reach init (QEMU exit 0)` — QEMU exits cleanly before the kernel
  reaches `/init`. No oops/panic on the console; the kernel likely stalled or halted during
  a driver's remove+re-probe cycle.

  **Trigger config:**
  ```
  configs/archive_failed/kconfig-rand500config-i386-v7.2-rc4-1130034ad8bb931d733ad604ddf6a0eec3d97fa5574b93987a3fa55fa933cb89-BOOT_FAIL-timeout.config
  ```

  **Multi-pass bisect result (6 passes, ~3 h total):**

  Six rounds of `make bisect` with `PINNED_OPTS=` accumulating each suspect were run.
  Every pass consistently narrowed to the left (alphabetically-first) half, indicating
  all required options reside in the alphabetically-early part of the 150-option candidate space.

  | Pass | Pinned | New suspect found | Verify alone |
  |------|--------|-------------------|--------------|
  | 1 | — | `CONFIG_DEBUG_TEST_DRIVER_REMOVE=y` | PASS (needs more) |
  | 2 | DEBUG_TEST_DRIVER_REMOVE | `CONFIG_AD7405=y` | PASS (needs more) |
  | 3 | + AD7405 | `CONFIG_AD7606_IFACE_PARALLEL=y` | PASS (needs more) |
  | 4 | + AD7606_IFACE_PARALLEL | `CONFIG_AD7606=y` | PASS (needs more) |
  | 5 | + AD7606 | `CONFIG_AUTOFS_FS=y` | PASS (needs more) |
  | 6 | + AUTOFS_FS | `CONFIG_BMC150_ACCEL=y` | PASS (needs more) |

  **Pattern:** Four of the five co-suspects (AD7405, AD7606, AD7606_IFACE_PARALLEL, BMC150_ACCEL)
  are all IIO (Industrial I/O) subsystem drivers from Analog Devices / Bosch. The bisect also found
  `CONFIG_AUTOFS_FS=y`, which is unrelated to IIO and is likely a bisect artifact (it happened to be
  alphabetically adjacent to the IIO drivers in the left half and was not individually required).

  **Root cause hypothesis:** `CONFIG_DEBUG_TEST_DRIVER_REMOVE` exercises driver probe/remove
  at boot. One or more IIO drivers (AD7405, AD7606, BMC150_ACCEL) have a buggy remove path on
  i386 that hangs or panics, causing the guest to shut down before reaching `/init`. The IIO
  infrastructure options (`CONFIG_IIO=y`, `CONFIG_IIO_BUFFER=y`, etc.) are also in the candidate
  set and are likely auto-selected when the IIO drivers are present.

  **Why the bisect did not converge:** The failure requires a combination of options that all
  happen to reside in the same alphabetically-first half of the candidate space. The PINNED_OPTS
  mechanism correctly identifies one option per pass, but when multiple co-required options are
  concentrated in the same half, the bisect must do one pass per required option. With 4+ IIO
  options needed, convergence would require additional passes with diminishing returns and no
  guarantee the final set is minimal.

  **Status:** Not actionable for LKML without a minimal 1–2 option reproducer. The interaction
  is real (full config + pinned suspects reliably fails; baseline passes) but the exact minimal
  set has not been isolated.

  **Next steps if revisiting:**
  - Test `CONFIG_DEBUG_TEST_DRIVER_REMOVE=y` + `CONFIG_IIO=y` (framework only, no specific
    driver) to see if the IIO core is sufficient without the AD7xxx drivers
  - Test `CONFIG_DEBUG_TEST_DRIVER_REMOVE=y` + `CONFIG_AD7606=y` + `CONFIG_AD7606_IFACE_PARALLEL=y`
    (a coherent driver + its interface option) to see if that pair is sufficient
  - If either 2-option test reproduces, file as a driver remove path bug in the IIO subsystem

---

## 2026-07-22 — v7.2-rc4 rand500config/i386: DEBUG_TEST_DRIVER_REMOVE breaks serial console

### High — Kernel Bug (single-option reproducer, actionable)

- [ ] **`CONFIG_DEBUG_TEST_DRIVER_REMOVE=y` alone breaks serial console on i386 — BOOT_FAIL-no-console**
  Kernel: v7.2-rc4. Arch: i386. Found by config bisect (`make bisect`) on
  `kconfig-rand500config-i386-v7.2-rc4-b7e535b388917a55c2c870494459ea80668d073efe99d0760fabcaf5a3ec656d-BOOT_FAIL-no-console.config`.

  **Bisect result:** `CONFIG_DEBUG_TEST_DRIVER_REMOVE=y` confirmed as the sole responsible option.
  Verified alone on a tinyconfig+bootability baseline: `BOOT_FAIL-no-console` reproduces with
  only this one option added. No co-required options.

  **Canary diagnosis:** `CANARY_EARLY=reached` — the raw UART `early_initcall` marker fired,
  confirming the kernel ran past `do_initcalls()`. The kernel is alive; the serial console
  driver was silently broken after the canary fired.

  **Mechanism:** `CONFIG_DEBUG_TEST_DRIVER_REMOVE` calls `->remove()` immediately after each
  successful `->probe()`, then re-probes. The 8250 UART driver (`CONFIG_SERIAL_8250_CONSOLE`)
  is among the drivers probed during boot. When it is removed and re-probed, the console
  registration (`console_tryregister()` / `uart_add_one_port()`) is apparently not re-established
  correctly, leaving the serial console silent for the rest of boot.

  **Relationship to prior finding (IIO interaction):** The previous `BOOT_FAIL-timeout`
  (kernel boots but never reaches init, requires DEBUG_TEST_DRIVER_REMOVE + multiple IIO drivers)
  is a separate failure mode — likely a deadlock or hang in an IIO driver's remove path. This
  is a distinct, simpler bug: the remove+re-probe of the UART console driver itself breaks
  console output, which is observable even without any IIO drivers present.

  **Minimal reproducer archived:**
  ```
  configs/archive_failed/kconfig-rand500config-i386-v7.2-rc4-a036ae3817c6b36ee468e644441358abb6b52765d83ea5217b860bd07d613254-BOOT_FAIL-no-console-bisect-from-b7e535b388917a55c2c870494459ea80668d073efe99d0760fabcaf5a3ec656d.config
  ```

  **Reproduce:**
  ```sh
  make bisect CONFIG_FILE=configs/archive_failed/kconfig-rand500config-i386-v7.2-rc4-b7e535b388917a55c2c870494459ea80668d073efe99d0760fabcaf5a3ec656d-BOOT_FAIL-no-console.config
  # or replay the minimal reproducer directly:
  make replay CONFIG_FILE=configs/archive_failed/kconfig-rand500config-i386-v7.2-rc4-a036ae3817c6b36ee468e644441358abb6b52765d83ea5217b860bd07d613254-BOOT_FAIL-no-console-bisect-from-b7e535b388917a55c2c870494459ea80668d073efe99d0760fabcaf5a3ec656d.config CONFIGS=rand500config ARCHS=i386
  ```

  **Next steps:**
  - Confirm on x86_64 to determine if the bug is i386-specific
  - Reproduce with a manual tinyconfig + `CONFIG_SERIAL_8250_CONSOLE=y` + `CONFIG_DEBUG_TEST_DRIVER_REMOVE=y` to get the minimal kernel config for LKML
  - Inspect `drivers/tty/serial/8250/8250_core.c` remove/probe path for console re-registration
  - File as a `CONFIG_DEBUG_TEST_DRIVER_REMOVE` + 8250 interaction bug

  **Subsystem:** `drivers/tty/serial/8250/` or `drivers/base/` (driver core). Mailing list:
  `linux-serial@vger.kernel.org`, `linux-kernel@vger.kernel.org`.

---

## 2026-07-22 — v7.2-rc3/rc4 rand500config/i386: REED_SOLOMON_DEC16 + SERIAL_8250_16550A_VARIANTS causes boot hang

### Medium — Kernel Bug (4-option reproducer, likely 32-bit arithmetic)

- [ ] **`CONFIG_REED_SOLOMON_DEC16=y` + `CONFIG_SERIAL_8250_16550A_VARIANTS=y` causes BOOT_FAIL on i386 — probable GF(2¹⁶) 32-bit overflow**
  Kernel: v7.2-rc3 (found), v7.2-rc4 (confirmed). Arch: i386 only. Found by 5-pass config bisect.

  **Minimal reproducer (4 options on tinyconfig+bootability base):**
  ```
  CONFIG_SERIAL_8250_16550A_VARIANTS=y
  CONFIG_RUNTIME_TESTING_MENU=y   ← required dep: REED_SOLOMON_TEST depends on this
  CONFIG_REED_SOLOMON_TEST=y
  CONFIG_REED_SOLOMON_DEC16=y
  ```

  **Canary diagnosis:** `CANARY_EARLY=reached` — kernel ran past `early_initcall`. The last
  console message is `printk: legacy bootconsole [uart8250] enabled`; nothing after.
  Either the console is broken at that point (by `SERIAL_8250_16550A_VARIANTS` UART re-probe)
  and the kernel runs silently past 60s, or the kernel hangs in the RS16 self-test.

  **Mechanism hypothesis:** `REED_SOLOMON_DEC16` enables the GF(2¹⁶) Reed-Solomon decoder
  (`lib/reed_solomon/`). `REED_SOLOMON_TEST` exercises it at `module_init()` time. On i386,
  `unsigned long` is 32 bits — GF(2¹⁶) polynomial arithmetic may overflow or produce incorrect
  results, causing the self-test to loop indefinitely or hang. This is the same class of 32-bit
  truncation bug as the `gpu_buddy` `roundup_pow_of_two()` finding (see above). The role of
  `SERIAL_8250_16550A_VARIANTS` is unclear: it may break console output at legacy console
  registration, hiding any subsequent RS16 panic/hang output, or it may add enough boot-time
  latency to push the total past 60s.

  **Why SERIAL_8250_16550A_VARIANTS is needed:** Without it, the 3-option set
  (RUNTIME_TESTING_MENU + REED_SOLOMON_TEST + REED_SOLOMON_DEC16) passes. It is a required
  co-trigger, not a red herring — but its exact role in the interaction is unknown.

  **Bisect chain:** 5 passes of `make bisect` with accumulating `PINNED_OPTS`:
  original 159-option config → 80 → 6 → 6 → 5 → 4 (confirmed alone).
  `CONFIG_SERIAL_SCCNXP=y` appeared in early passes but was confirmed as a red herring
  (dropped when it was pinned and the same minimum set reproduced without it).

  **Minimal reproducer archived:**
  ```
  configs/archive_failed/kconfig-rand500config-i386-v7.2-rc3-0bacb63c3fca296ff2d063a7a5378ab9dd7917b5a0c93f3b60a057482873d7bd-BOOT_FAIL-timeout-bisect-from-7d6c10ba6f8526542dc912aacb8e35453743a0f9d4a36c43b9fd5a4df019d70d.config
  ```

  **Reproduce:**
  ```sh
  make replay CONFIG_FILE=configs/archive_failed/kconfig-rand500config-i386-v7.2-rc3-0bacb63c3fca296ff2d063a7a5378ab9dd7917b5a0c93f3b60a057482873d7bd-BOOT_FAIL-timeout-bisect-from-7d6c10ba6f8526542dc912aacb8e35453743a0f9d4a36c43b9fd5a4df019d70d.config CONFIGS=rand500config ARCHS=i386
  ```

  **Next steps:**
  - Check `lib/reed_solomon/decode_rs.c` for `unsigned long` or `int` used where `u64` is needed
    in GF(2¹⁶) polynomial multiply/divide on 32-bit arches — same pattern as the gpu_buddy fix
  - Confirm whether `SERIAL_8250_16550A_VARIANTS` breaks console (canary replay with this
    4-option config) or merely adds latency
  - If RS16 arithmetic overflow confirmed: fix analogously to gpu_buddy (`BIT_ULL`/`u64` cast)
  - Check if x86_64 is unaffected (expected: `unsigned long` is 64-bit there)

  **Subsystem:** `lib/reed_solomon/`. Maintainer: Thomas Gleixner.
  Mailing list: `linux-kernel@vger.kernel.org`.

---

## 2026-07-23 — v7.1.5-rc2 allmodconfig Build Failure: myri10ge + arm64 + GCC 15

### High — Build Failure (arm64 + GCC 15, actionable)

- [ ] **`myri10ge_dummy_rdma()` sends uninitialized `buf[6..15]` — build error on arm64 with GCC 15 and `CONFIG_WERROR`**
  Kernel: v7.1.5-rc2 (stable-rc) and v7.2-rc4 (mainline). Arch: arm64 cross-compile. Found by
  `make all` full run (`allmodconfig` / `arm64`). Compiler: `aarch64-linux-gnu-gcc` = GCC 15.3.0.

  **Build error:**
  ```
  In function '__const_memcpy_toio_aligned64',
      inlined from '__iowrite64_copy' at arch/arm64/include/asm/io.h:252:3,
      inlined from 'myri10ge_dummy_rdma' at drivers/net/ethernet/myricom/myri10ge/myri10ge.c:535:2:
  arch/arm64/include/asm/io.h:209:17: error: '*(const u64 *)(&buf[6])' is used uninitialized
    [-Werror=uninitialized]
  myri10ge.c:511:16: note: '*(const u64 *)(&buf[6])' was declared here
    __be32 buf[16] __attribute__ ((__aligned__(8)));
  ```

  **Root cause:** `myri10ge_dummy_rdma()` declares a 16-element `__be32` buffer but only
  initializes elements 0–5. Elements 6–15 are intended as padding (device doesn't care about
  their values) but are never zeroed. `myri10ge_pio_copy` maps to `__iowrite64_copy(to, from, size/8)`,
  which on arm64 uses `__const_memcpy_toio_aligned64` — a vectorized inline that reads the buffer
  in 64-bit chunks. GCC 15 traces through the three-level inline chain and detects that
  `*(const u64 *)(&buf[6])` is read uninitialized.

  `CONFIG_WERROR=y` (set by `allmodconfig`) promotes the warning to an error. Without it the
  compiler warns but exits 0.

  **Why only now:** GCC 15 tightened uninitialized-variable analysis to follow deeper inline
  chains. On Arch Linux, `aarch64-linux-gnu-gcc` tracks the system GCC (currently 15). On
  Debian/Ubuntu the cross package is typically GCC 13 or 14 — those systems will not see this failure.

  **Proposed fix:**
  ```diff
  - __be32 buf[16] __attribute__ ((__aligned__(8)));
  + __be32 buf[16] __attribute__ ((__aligned__(8))) = {};
  ```

  **Minimal reproducer** (from `~/git/linux-stable-rc`):
  ```sh
  make O=/tmp/myri-repro ARCH=arm64 tinyconfig
  scripts/config --file /tmp/myri-repro/.config \
      -e CONFIG_NET -e CONFIG_INET -e CONFIG_NETDEVICES \
      -e CONFIG_ETHERNET -e CONFIG_PCI -e CONFIG_MYRI10GE \
      -e CONFIG_WERROR
  make O=/tmp/myri-repro ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
      CC=aarch64-linux-gnu-gcc olddefconfig
  make O=/tmp/myri-repro ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
      CC=aarch64-linux-gnu-gcc \
      drivers/net/ethernet/myricom/myri10ge/myri10ge.o
  ```
  Note: `CC=aarch64-linux-gnu-gcc` must be the full cross-compiler name — `CC=gcc-15` strips
  the cross-compile prefix and the host x86 GCC rejects the arm64 flag `-mlittle-endian`.

  **Subsystem:** `drivers/net/ethernet/myricom/myri10ge/` — legacy 10GbE driver.
  Maintainers: `scripts/get_maintainer.pl drivers/net/ethernet/myricom/myri10ge/myri10ge.c`.
  Mailing list: `netdev@vger.kernel.org`, `linux-kernel@vger.kernel.org`.

---

## 2026-07-24 — riscv64 Integration

### Low — Harness Issue (wasteful, not incorrect)

- [x] **Boot baseline check in `build.sh` is not arch-aware — applies arm64 PL011 options to riscv/defconfig** ✅ resolved 2026-07-24
  Observed during `make full` on the `feat/risc-architecture` branch. When building
  `defconfig/riscv`, `build.sh` detected two options as missing from the boot baseline
  and ran the auto-correct loop twice:

  ```
  WARN  Boot baseline options missing (pass 1) — auto-correcting: CONFIG_SERIAL_AMBA_PL011=y
  INFO  Boot baseline corrected (pass 1): CONFIG_SERIAL_AMBA_PL011=y
  WARN  Boot baseline options missing (pass 2) — auto-correcting: CONFIG_SERIAL_AMBA_PL011_CONSOLE=y
  INFO  Boot baseline corrected (pass 2): CONFIG_SERIAL_AMBA_PL011_CONSOLE=y
  ```

  `CONFIG_SERIAL_AMBA_PL011` and `CONFIG_SERIAL_AMBA_PL011_CONSOLE` are the ARM PL011 UART
  driver options — only meaningful on arm64 (where the QEMU virt machine exposes a PL011).
  On riscv the QEMU virt machine uses NS16550, so `olddefconfig` immediately drops both options
  after each auto-correct pass. The final config is correct; only two redundant
  `make olddefconfig` cycles are wasted.

  **Root cause:** The boot baseline options list in `lib/build.sh` is shared across all
  architectures. It includes both the x86 8250 options and the arm64 PL011 options, and
  the auto-correct loop does not filter by `$ARCH` before attempting to force an option.

  **Impact:** Minor — two extra configuration passes (~5–10 s each) per riscv defconfig build.
  No incorrect behavior: `olddefconfig` correctly ignores options that have no Kconfig entry
  for the target arch.

  **Fix:** `lib/build.sh` now defines `BOOT_BASELINE_OPTS` as an array populated per-arch
  in a `case "$ARCH"` block — common options (PRINTK, TTY, INITRD, BINFMT, TMPFS) plus
  arch-specific serial driver options. The correction loop iterates over the array instead
  of reading from `configs/tinyconfig.config`. arm64 PL011 options are only checked on
  arm64; riscv gets its own set (8250 + OF_PLATFORM + FPU). No file parsing needed.

---

## 2026-07-25 — v7.2-rc4 allmodconfig Build Failures

### High — Build Failure (arm64 + GCC 16, actionable)

- [ ] **`security/landlock/fs.c`: three `struct layer_masks` variables used uninitialized — build error on arm64 with GCC 16 and `CONFIG_WERROR`**
  Kernel: v7.2-rc4 (mainline). Arch: arm64 cross-compile. Found by
  `make all CONFIGS=allmodconfig BUILD_TIMEOUT=3600`. Compiler: `aarch64-linux-gnu-gcc` = GCC 16.1.0.

  **Build error:**
  ```
  security/landlock/fs.c: In function 'is_access_to_paths_allowed':
  security/landlock/fs.c:767:28: error: '_layer_masks_child1' is used uninitialized [-Werror=uninitialized]
    767 |         struct layer_masks _layer_masks_child1, _layer_masks_child2;
        |                            ^~~~~~~~~~~~~~~~~~~
  security/landlock/fs.c:767:49: error: '_layer_masks_child2' is used uninitialized [-Werror=uninitialized]
    767 |         struct layer_masks _layer_masks_child1, _layer_masks_child2;
        |                                                 ^~~~~~~~~~~~~~~~~~~
  security/landlock/fs.c: In function 'hook_unix_find':
  security/landlock/fs.c:1649:28: error: 'layer_masks' is used uninitialized [-Werror=uninitialized]
   1649 |         struct layer_masks layer_masks;
        |                            ^~~~~~~~~~~
  cc1: all warnings being treated as errors
  ```

  **Root cause:** Two functions declare `struct layer_masks` variables without a zero-initializer:
  - `is_access_to_paths_allowed()` at line 767: `_layer_masks_child1` and `_layer_masks_child2` are
    assigned conditionally (via `layer_masks_child1 = &_layer_masks_child1` only inside
    `if (unlikely(dentry_child1))`). GCC 16's uninitialized-variable analysis recognises that on the
    path where neither `dentry_child1` nor `dentry_child2` is set, the structs are read through the
    pointer without ever being written.
  - `hook_unix_find()` at line 1649: `layer_masks` is declared without initializer; GCC 16 flags it
    uninitialized before its first conditional use.

  `CONFIG_WERROR=y` (set by `allmodconfig`) promotes these warnings to errors.

  **Why only now:** The trigger is **GCC 16 + arm64** specifically. Tested combinations:
  - GCC 16 + arm64 allmodconfig → **build error** (confirmed)
  - GCC 16 + x86_64 allmodconfig → no error
  - GCC 15 + x86_64 allmodconfig → no error

  The arm64 allmodconfig build passes `-mgeneral-regs-only` (restricts the compiler to integer
  registers, no FP/SIMD), which changes register allocation and dataflow analysis for large struct
  operations enough that GCC 16 finds the definite uninitialized path. The flag `-Wno-maybe-uninitialized`
  is present in both builds, so the warning must be definite (`-Wuninitialized`), not speculative.
  The pattern was always latently wrong — every other `struct layer_masks` local in the file uses
  `= {}` — but older compilers and x86_64 GCC 16 don't catch it.

  **Proposed fix:**
  ```diff
  - struct layer_masks _layer_masks_child1, _layer_masks_child2;
  + struct layer_masks _layer_masks_child1 = {}, _layer_masks_child2 = {};
  ```
  and at line 1649:
  ```diff
  - struct layer_masks layer_masks;
  + struct layer_masks layer_masks = {};
  ```
  Fix verified: `allmodconfig arm64` compiles clean after applying both hunks.

  **Reproducer** (from `~/git/linux`, requires `aarch64-linux-gnu-gcc` = GCC 16):
  ```sh
  make O=/tmp/landlock-repro ARCH=arm64 allmodconfig
  make O=/tmp/landlock-repro ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
      CC=aarch64-linux-gnu-gcc security/landlock/fs.o
  # Expected: three [-Werror=uninitialized] errors at fs.c:767 and fs.c:1649
  ```
  Note: tinyconfig does not reproduce — lacks enough security config for the hooks to compile.
  allmodconfig is the correct reproducer (also matches the original failure context).

  **Subsystem:** `security/landlock/` — Landlock LSM.
  Maintainer: `Mickaël Salaün <mic@digikod.net>`.
  Mailing list: `linux-security-module@vger.kernel.org`, `linux-kernel@vger.kernel.org`.

---

### High — Build Failure (riscv allmodconfig, patch ready) ✅ resolved 2026-08-03

- [x] **`CONFIG_RISCV_USER_CFI=y` with Binutils 2.44 linker breaks allmodconfig riscv — `__vdso_rt_sigreturn_cfi_offset` undeclared**
  Kernel: v7.2-rc4 + v7.2-rc5 (mainline). Arch: riscv cross-compile. Found by
  `make all CONFIGS=allmodconfig BUILD_TIMEOUT=3600`. Compiler: `riscv64-linux-gnu-gcc` = GCC 15.1.0,
  linker: `riscv64-linux-gnu-ld` = Binutils 2.44.

  ---

  **Background — what the pieces are (beginner-friendly):**

  - **VDSO** (*Virtual Dynamic Shared Object*): A tiny library the kernel builds and maps into every
    process's memory. User programs call it for fast operations like getting the current time — without
    doing a full system call. The kernel builds one VDSO `.so` file per architecture as part of `make`.

  - **vdso_cfi**: When `CONFIG_RISCV_USER_CFI=y`, the kernel builds a *second* VDSO compiled with
    RISC-V CFI (Control Flow Integrity) extensions — hardware instructions that detect when a function
    returns to the wrong place (ROP attacks). Programs without CFI-capable hardware get the regular
    VDSO; programs running on CFI-capable hardware get this one.

  - **GNU property note** (type `0xc0000000`): A metadata tag embedded inside the compiled `.o` files
    that says "this code uses CFI instructions". The linker is supposed to read these tags when it
    combines the `.o` files into the final `.so` file and propagate the information. Think of it as a
    sticky label on each object saying what hardware features it needs.

  - **`CONFIG_WERROR=y`**: Makes the compiler treat all warnings as errors. `allmodconfig` always
    sets this. When `CONFIG_WERROR=y`, the Makefile also passes `--fatal-warnings` to the *linker*,
    which means linker warnings also become fatal errors.

  - **Kconfig `depends on`**: The switch in the feature's configuration that says "only enable this
    feature if condition X is true". Before this fix, `CONFIG_RISCV_USER_CFI` only checked whether
    the *compiler* understood CFI instructions. It did not check whether the *linker* understood the
    resulting metadata tags.

  ---

  **How we found it:**

  1. Ran `make all NO_FETCH=1 CONFIGS=allmodconfig ARCHS=riscv` on v7.2-rc5 — the build failed.
  2. The `build.log` showed `riscv64-linux-gnu-ld: warning: ... unsupported GNU_PROPERTY_TYPE (5)
     type: 0xc0000000` — the linker doesn't know what these tags mean.
  3. Confirmed the `--fatal-warnings` mechanism: found `scripts/Makefile.warn:207` adds
     `KBUILD_LDFLAGS += --fatal-warnings` when `CONFIG_WERROR=y`. So the linker warning became a
     fatal error.
  4. Reproduced the linker failure directly:
     ```sh
     # Extract the exact ld command from the build:
     cat build/allmodconfig-riscv/arch/riscv/kernel/vdso_cfi/.vdso-cfi.so.dbg.cmd
     # Run it — exits 1 with the warning
     ```
  5. Traced the cascade: ld exits 1 → `vdso-cfi.so.dbg` is never written → `nm` complains "No such
     file" → `vdso-cfi-offsets.h` is generated empty → `signal.c` can't find
     `__vdso_rt_sigreturn_cfi_offset` → compile error.
  6. Found the root cause in `arch/riscv/Kconfig:1181`: `CONFIG_RISCV_USER_CFI` checks only the
     compiler (`$(cc-option,...)`), not the linker. Added linker commit author Deepak Gupta as `Cc:`.

  ---

  **Build failure sequence (what actually happens):**

  ```
  Step 1: VDSOLD  arch/riscv/kernel/vdso_cfi/vdso-cfi.so.dbg
    riscv64-linux-gnu-ld: warning: rt_sigreturn.o: unsupported GNU_PROPERTY_TYPE (5) type: 0xc0000000
    riscv64-linux-gnu-ld: warning: vgettimeofday.o: unsupported GNU_PROPERTY_TYPE (5) type: 0xc0000000
    [... same for all 9 vdso_cfi objects ...]
    → ld exits 1 (--fatal-warnings turns warnings into errors)
    → vdso-cfi.so.dbg is NOT created

  Step 2: VDSOSYM include/generated/vdso-cfi-offsets.h
    riscv64-linux-gnu-nm: 'vdso-cfi.so.dbg': No such file or directory
    → vdso-cfi-offsets.h is generated empty (0 bytes)

  Step 3: Compiling arch/riscv/kernel/signal.c
    error: '__vdso_rt_sigreturn_cfi_offset' undeclared
    → The symbol should have come from vdso-cfi-offsets.h — but that file is empty
  ```

  ---

  **Root cause in `arch/riscv/Kconfig`:**

  The feature's Kconfig entry only checks compiler support, not linker support:
  ```kconfig
  config RISCV_USER_CFI
      depends on $(cc-option,-mabi=lp64 -march=rv64ima_zicfiss_zicfilp -fcf-protection=full)
      # ^^^ checks: can riscv64-linux-gnu-gcc compile with CFI flags? YES (GCC 15 can)
      # Missing: can riscv64-linux-gnu-ld link the resulting objects? NO (Binutils 2.44 cannot)
  ```

  GCC 15 + Binutils 2.44 is an inconsistent toolchain: the compiler knows how to emit CFI property
  tags, but the linker doesn't know how to read them. Normally a distro's toolchain would keep these
  in sync, but cross-compilation toolchains (`riscv64-linux-gnu-*`) can lag behind.

  The missing `depends on` line is the bug — introduced by commit
  `22c1e263af2a` ("riscv: create a Kconfig fragment for shadow stack and landing pad support").

  ---

  **Reproducer** (from `~/git/linux`):
  ```sh
  make O=/tmp/riscvcfi-repro ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- allmodconfig
  make O=/tmp/riscvcfi-repro ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- -j$(nproc)
  # Build fails at signal.o with __vdso_rt_sigreturn_cfi_offset undeclared
  ```

  Manual two-step to isolate just the linker failure:
  ```sh
  # Step 1: build just the vdso_cfi (shows the linker warnings + exit 1):
  make O=/tmp/riscvcfi-repro ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- \
      arch/riscv/kernel/vdso_cfi/vdso-cfi.so.dbg
  # Step 2: check the generated header is empty:
  wc -c /tmp/riscvcfi-repro/include/generated/vdso-cfi-offsets.h
  # Shows: 0
  ```

  The linker failure can be reproduced standalone with a tiny assembly file:
  ```sh
  # Confirm riscv64-linux-gnu-ld 2.44 doesn't understand the CFI property type:
  /home/benni/git/linux-dev/scripts/riscv-ld-cfi-prop-test.sh \
      riscv64-linux-gnu-gcc riscv64-linux-gnu-ld
  echo $?  # prints: 1  (non-zero = linker can't handle it)
  ```

  ---

  **Fix — two files changed:**

  The fix adds a Kconfig capability test: when building `allmodconfig` (or any RISC-V config with
  `CONFIG_RISCV_USER_CFI` candidate), Kconfig now runs a small test script that assembles a minimal
  object with the CFI property tag and tries to link it. If the linker exits non-zero (as Binutils
  2.44 does), the `$(success,...)` check returns `n` and `CONFIG_RISCV_USER_CFI` is disabled.

  **`scripts/riscv-ld-cfi-prop-test.sh`** (new file):
  ```bash
  # Assembles a tiny .s with the CFI GNU property, then links with --fatal-warnings.
  # Exit 0 = linker handles it fine; exit 1 = linker doesn't know this property.
  CC="$1"; LD="$2"
  # ... (creates temp dir, assembles, links, returns exit code)
  ```

  **`arch/riscv/Kconfig`** (one line added):
  ```diff
   config RISCV_USER_CFI
       depends on 64BIT && MMU && \
           $(cc-option,-mabi=lp64 -march=rv64ima_zicfiss_zicfilp -fcf-protection=full)
       depends on RISCV_ALTERNATIVE
  +    depends on $(success,$(srctree)/scripts/riscv-ld-cfi-prop-test.sh $(CC) $(LD))
  ```

  When Kconfig runs during `make allmodconfig`, it executes the test script. With Binutils 2.44:
  script returns 1 → `$(success,...)` = `n` → `CONFIG_RISCV_USER_CFI` is not set → the vdso_cfi
  build never runs → no failure.

  **Patch branch:** `~/git/linux-dev` on `fix/20260803_riscv_user_cfi_ld_cfi_prop_test` (commit `592ae4fe`).

  **Verified:** `make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- allmodconfig` on the patched tree
  produces no `CONFIG_RISCV_USER_CFI` entry — feature correctly disabled with Binutils 2.44.

  **Fixes:** `22c1e263af2a` ("riscv: create a Kconfig fragment for shadow stack and landing pad support")
  **Cc:** `stable@vger.kernel.org` (appears in Linux 7.2)

  **Send patch with:**
  ```sh
  # Remove Co-Authored-By line from commit first (git commit --amend), then:
  cd ~/git/linux-dev
  git send-email --to=linux-riscv@lists.infradead.org \
    --cc=pjw@kernel.org --cc=palmer@dabbelt.com --cc=aou@eecs.berkeley.edu \
    --cc=alex@ghiti.fr --cc=debug@rivosinc.com --cc=zong.li@sifive.com \
    --cc=linux-kernel@vger.kernel.org --cc=stable@vger.kernel.org \
    $(git format-patch -1)
  ```

  **Report artifacts** (`reports/mainline-7.2-2026-08-02_21-33-27-v7.2-rc5/`):

  | File | Present | Notes |
  |------|---------|-------|
  | `build-allmodconfig-riscv.log` | ✅ | Primary evidence — linker warnings + `signal.c` error |
  | `kconfig-allmodconfig-riscv.config` | ✅ | The triggering config — shows `CONFIG_RISCV_USER_CFI=y` |
  | `dmesg-allmodconfig-riscv.txt` | ❌ absent | Build failed; QEMU never ran |
  | `qemu-allmodconfig-riscv.log` | ❌ absent | Build failed; QEMU never ran |
  | `vmstatus-allmodconfig-riscv.txt` | ❌ absent | Build failed; QEMU never ran |

  The absence of the three QEMU/boot files is itself a diagnostic signal: any combo missing
  all three while a `build-*.log` exists means the build failed before QEMU was launched.

  **Subsystem:** `arch/riscv/` — RISC-V architecture Kconfig / vDSO CFI.
  Maintainers: `Paul Walmsley <pjw@kernel.org>`, `Palmer Dabbelt <palmer@dabbelt.com>`,
  `Albert Ou <aou@eecs.berkeley.edu>`.
  Mailing list: `linux-riscv@lists.infradead.org`, `linux-kernel@vger.kernel.org`.


- [ ] **CONFIG_ARM64_16K_PAGES=y causes BOOT_FAIL-timeout on arm64**
  Kernel: v7.2-rc4. Found by config bisect from kconfig-rand500config-arm64-v7.2-rc3-7d9b39f86d0e9106b0750c12b024216a1ba1a3297adc7c1dd957daf0525f5485-BOOT_FAIL-timeout.config.
  **Reproduce:**
  ```sh
  make bisect CONFIG_FILE=/home/benni/git/kernel-test/configs/archive_failed/kconfig-rand500config-arm64-v7.2-rc3-207fc28107009142b5b2c6fba12e910d0b4e42462897d911c2f2ffe22332b1a1-BOOT_FAIL-timeout-bisect-from-7d9b39f86d0e9106b0750c12b024216a1ba1a3297adc7c1dd957daf0525f5485.config
  ```


---

## 2026-07-28 — RISC-V VDSO CFI: Linker Warnings in Bootable Configs

### Low — Compiler / Toolchain Warning (warning-only manifestation of known root cause)

- [ ] **`CONFIG_RISCV_USER_CFI=y` generates GNU ELF properties that Binutils 2.44 doesn't understand — linker warns but build succeeds**
  Kernel: v7.2-rc4. Arch: riscv. Found by `lib/warnings.sh` (new warning analysis feature) during `make full`.
  Related to the allmodconfig build *failure* recorded 2026-07-25 — that entry covers the fatal case;
  this entry covers the warning-only case in bootable configs (defconfig, randdefconfig, etc.).

  **Warning output (randdefconfig-riscv):**
  ```
  riscv64-linux-gnu-ld: warning: arch/riscv/kernel/vdso_cfi/flush_icache.o: unsupported GNU_PROPERTY_TYPE (5) type: 0xc0000000
  riscv64-linux-gnu-ld: warning: arch/riscv/kernel/vdso_cfi/getcpu.o: unsupported GNU_PROPERTY_TYPE (5) type: 0xc0000000
  riscv64-linux-gnu-ld: warning: arch/riscv/kernel/vdso_cfi/getrandom.o: unsupported GNU_PROPERTY_TYPE (5) type: 0xc0000000
  [... 9 more — one per vdso_cfi object ...]
  riscv64-linux-gnu-nm: warning: arch/riscv/kernel/vdso_cfi/vdso-cfi.so.dbg: unsupported GNU_PROPERTY_TYPE (5) type: 0xc0000000
  riscv64-linux-gnu-objcopy: warning: arch/riscv/kernel/vdso_cfi/vdso-cfi.so.dbg: unsupported GNU_PROPERTY_TYPE (5) type: 0xc0000000
  ```

  **Root cause:** `CONFIG_RISCV_USER_CFI` (`def_bool y`, gates `arch/riscv/kernel/vdso_cfi/`) has a
  `depends on` clause that checks only the *compiler* capability:
  ```
  depends on $(cc-option,-mabi=lp64 -march=rv64ima_zicfiss_zicfilp -fcf-protection=full)
  ```
  When `riscv64-linux-gnu-gcc` (GCC 15) supports Zicfiss/Zicfilp, `RISCV_USER_CFI` auto-enables and
  the vDSO CFI objects are compiled with RISC-V CFI ELF property notes (`GNU_PROPERTY_RISCV_FEATURE_1_AND`,
  type `0xc0000000`). The *linker* (`riscv64-linux-gnu-ld`, Binutils 2.44 — the latest released version)
  does not yet understand these notes and warns for each object, but exits 0 so the build succeeds.

  **Why warnings but not failure (unlike allmodconfig):** In allmodconfig, `ld` produces the warnings
  AND the `vdso-cfi.so.dbg` output file is not created (the linker exits non-zero or the file is
  truncated), causing the downstream `nm`/`VDSOSYM` step to fail with "No such file" and leaving
  `include/generated/vdso-cfi-offsets.h` incomplete → `signal.c` compile error. In bootable configs
  (defconfig, randdefconfig), `ld` exits 0 and produces a complete `.so.dbg` despite the warnings,
  so all downstream steps succeed. The difference may be config-dependent (allmodconfig enables more
  options that alter the linker invocation or object set).

  **Why only randdefconfig, not defconfig (incremental build limitation):**
  Both `defconfig-riscv` and `randdefconfig-riscv` have `CONFIG_RISCV_USER_CFI=y`. However,
  `defconfig-riscv` was built in a previous incremental run — the `vdso_cfi/` objects already existed
  on disk, so the current `make full` did not recompile or relink them. No linker invocation →
  no warnings in this run's `build.log`. `randdefconfig-riscv` always does a full rebuild (its config
  changes each run), so the linker runs fresh and the warnings are captured.
  Confirmed: `grep -c "vdso_cfi" build/defconfig-riscv/build.log` → `0` (stale log).

  **Proposed fix:** Add a linker capability check to `arch/riscv/Kconfig`:
  ```diff
   config RISCV_USER_CFI
           def_bool y
           depends on 64BIT && MMU && \
                   $(cc-option,-mabi=lp64 -march=rv64ima_zicfiss_zicfilp -fcf-protection=full)
  +        depends on $(ld-option,--march=rv64ima_zicfiss_zicfilp)
           depends on RISCV_ALTERNATIVE
  ```
  Or alternatively, document the minimum Binutils version (> 2.44) required when
  `CONFIG_RISCV_USER_CFI=y` is to build warning-free.

  **Reproduce:**
  ```sh
  # Force a full rebuild of defconfig-riscv to see the warnings directly:
  rm -rf build/defconfig-riscv
  make all NO_FETCH=1 CONFIGS=defconfig ARCHS=riscv
  grep "vdso_cfi" build/defconfig-riscv/build.log | head -5
  # Or: run any randconfig variant and check warnings-summary.txt:
  make all NO_FETCH=1 CONFIGS=randdefconfig ARCHS=riscv
  cat reports/$(ls -t reports/ | grep -v baseline | head -1)/warnings-summary.txt
  ```

  **Subsystem:** `arch/riscv/` — RISC-V architecture Kconfig / vDSO CFI.
  Maintainers: `Paul Walmsley <pjw@kernel.org>`, `Palmer Dabbelt <palmer@dabbelt.com>`,
  `Albert Ou <aou@eecs.berkeley.edu>`.
  Mailing list: `linux-riscv@lists.infradead.org`, `linux-kernel@vger.kernel.org`.

---

## lib/warnings.sh — Known Limitation

### Warning counts are unreliable for deterministic configs on incremental builds

`lib/warnings.sh` extracts warnings from `build/<config>-<arch>/build.log`. The build log only
records what was *compiled in that run*. On an incremental build (config unchanged, kernel source
unchanged since last build), unchanged objects are not recompiled — no compiler/linker invocation →
no warnings for those objects in the log.

**Affected configs:** `defconfig`, `tinyconfig`, `allnoconfig`, `kunitconfig`, `allmodconfig` —
any config that is deterministic and was previously built. The warning count will be 0 or lower
than the true value after the first build.

**Unaffected configs:** `randdefconfig`, `rand500config`, `kunitrandconfig`, `randconfig` —
these always do a full rebuild (config changes each run), so the build log is always complete.

**Practical impact:** The CROSS-ARCH DIVERGENCE and NEW SINCE sections in `warnings-summary.txt`
are reliable for random configs (which are the primary signal source). For stable configs, warning
counts and diffs are only accurate after `make clean` or a kernel version bump (which forces many
recompiles via ccache misses on new object paths).

**Discovery:** `defconfig-riscv` had `CONFIG_RISCV_USER_CFI=y` and the `vdso_cfi/` objects existed
in `build/defconfig-riscv/` from a prior run, but showed 0 `vdso_cfi` occurrences in the current
`build.log` — the incremental build skipped them. The same warnings appeared in `randdefconfig-riscv`
because that config fully rebuilds every run.

**Workaround:** Force a full rebuild of a specific combo when accurate warning counts are needed:
```sh
rm -rf build/<config>-<arch>
make all NO_FETCH=1 CONFIGS=<config> ARCHS=<arch>
```

**Recommendation:** No code change. Rely on `randdefconfig`, `rand500config`, and `kunitrandconfig`
as the primary warning signal sources — they are always accurate. Stable config warning counts are
a lower bound only.

---

## 2026-08-11 — v7.2-rc7 allmodconfig Build Failures

All four were found during two independent `make all NO_FETCH=1 CONFIGS=allmodconfig ARCHS="x86_64 i386 arm64 riscv" BUILD_TIMEOUT=3600` runs on v7.2-rc7 (commit `db2ddb871435`). i386 passes both times. The three failing arches are reproducible, not flakes.

**Reproduce all four with:**
```sh
# In ~/git/kernel-test (mainline clone, after make fetch / make checkout TAG=v7.2-rc7):
make all NO_FETCH=1 CONFIGS=allmodconfig ARCHS="x86_64 arm64 riscv" BUILD_TIMEOUT=3600
# x86_64 fails in ~6s, arm64 in ~50s (ccache warm), riscv in ~25s (ccache warm)
```

### High — Build Failure (x86_64, new finding)

- [ ] **x86_64 allmodconfig: `generated/randstruct_hash.h: No such file` — generated header missing when `scripts/mod` compiles**
  Kernel: v7.2-rc7. Arch: x86_64. Found 2026-08-11. First observation.

  **Build error:**
  ```
  /home/benni/git/linux/include/linux/compiler-version.h:33:10: fatal error: generated/randstruct_hash.h: No such file or directory
     33 | #include <generated/randstruct_hash.h>
  make[5]: *** [.../scripts/Makefile.build:289: scripts/mod/empty.o] Error 1
  make[5]: *** [.../scripts/Makefile.build:184: scripts/mod/devicetable-offsets.s] Error 1
  make[4]: *** [.../Makefile:1407: prepare0] Error 2
  ```

  **Root cause:** `include/linux/compiler-version.h` conditionally includes
  `<generated/randstruct_hash.h>` when `CONFIG_RANDSTRUCT` is set. The file is generated
  during the kernel build's `prepare` phase by the randstruct seed generator. In v7.2-rc7
  allmodconfig, `CONFIG_RANDSTRUCT_FULL=y` (or `CONFIG_RANDSTRUCT_PERFORMANCE=y`) is
  selected. The `prepare0` target compiles `scripts/mod/empty.o` in parallel with or
  before the seed file is generated, causing the include to fail.

  The failure happens in 6–7 seconds on both runs (the `scripts/mod` compile is one of
  the first things the kernel Makefile does). On i386 allmodconfig the failure does not
  occur, suggesting RANDSTRUCT may be disabled or ordered differently for i386.

  **Reproduce (direct):**
  ```sh
  cd ~/git/linux                       # v7.2-rc7
  make O=/tmp/allmod-x86 ARCH=x86_64 allmodconfig
  make O=/tmp/allmod-x86 ARCH=x86_64 -j$(nproc) bzImage
  # Fails in seconds with: generated/randstruct_hash.h: No such file or directory
  ```

  **Reproduce (harness):**
  ```sh
  make all NO_FETCH=1 CONFIGS=allmodconfig ARCHS=x86_64 BUILD_TIMEOUT=3600
  grep "randstruct_hash" build/allmodconfig-x86_64/build.log
  ```

  **Next steps:**
  - Check `include/linux/compiler-version.h:33` for the exact `#ifdef CONFIG_RANDSTRUCT` guard
  - Check whether `prepare0` in `Makefile` depends on the `randstruct_hash.h` generation target
  - Confirm whether the seed generator (likely `kernel/gen_randstruct_seed.pl` or equivalent)
    runs before `scripts/mod/` compilation
  - Check if this is a v7.2-rc7-specific regression (not seen in earlier rc builds — allmodconfig
    x86_64 was not explicitly run before this date)

  **Subsystem:** Kernel build system (`scripts/Makefile.build`, `Makefile`, `kernel/randstruct*`).
  Mailing list: `linux-kbuild@vger.kernel.org`, `linux-kernel@vger.kernel.org`.

### High — Build Failures (arm64 + riscv, existing open issues confirmed in rc7)

These three failures are documented in earlier entries and remain unresolved in v7.2-rc7:

**arm64 — `security/landlock/fs.c` uninitialized variables** (first observed v7.2-rc4):
See 2026-07-25 entry. Same three variables (`_layer_masks_child1`, `_layer_masks_child2` in
`is_access_to_paths_allowed()`, `layer_masks` in `hook_unix_find()`) still uninitialized.
Now observed in rc4, rc5, rc7 — 7 weeks without a fix. Patch fix is a 2-line `= {}` init.
Trigger is GCC 16 + arm64 allmodconfig specifically (GCC 16 x86_64 does not trigger).

```
security/landlock/fs.c:767:28: error: '_layer_masks_child1' is used uninitialized [-Werror=uninitialized]
security/landlock/fs.c:767:49: error: '_layer_masks_child2' is used uninitialized [-Werror=uninitialized]
security/landlock/fs.c:1649:28: error: 'layer_masks' is used uninitialized [-Werror=uninitialized]
```

**Reproduce** (requires `aarch64-linux-gnu-gcc` = GCC 16; tinyconfig does not work — use allmodconfig):
```sh
make O=/tmp/landlock-repro ARCH=arm64 allmodconfig
make O=/tmp/landlock-repro ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
    CC=aarch64-linux-gnu-gcc security/landlock/fs.o
```

---

**arm64 — `myri10ge_dummy_rdma()` uninitialized `buf[6..15]`** (first observed v7.1.5-rc2 + v7.2-rc4):
See 2026-07-23 entry. Same inline chain: `myri10ge_dummy_rdma()` → `myri10ge_pio_copy()`
→ `__iowrite64_copy()` → `__const_memcpy_toio_aligned64()` → reads `buf[6]` uninitialized.

```
arch/arm64/include/asm/io.h:209:17: error: '*(const u64 *)(&buf[6])' is used uninitialized [-Werror=uninitialized]
myri10ge.c:511:16: note: '*(const u64 *)(&buf[6])' was declared here
  __be32 buf[16] __attribute__ ((__aligned__(8)));
```

**Reproduce:**
```sh
make O=/tmp/myri-repro ARCH=arm64 tinyconfig
scripts/config --file /tmp/myri-repro/.config \
    -e CONFIG_NET -e CONFIG_INET -e CONFIG_NETDEVICES \
    -e CONFIG_ETHERNET -e CONFIG_PCI -e CONFIG_MYRI10GE \
    -e CONFIG_WERROR
make O=/tmp/myri-repro ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
    CC=aarch64-linux-gnu-gcc olddefconfig
make O=/tmp/myri-repro ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
    CC=aarch64-linux-gnu-gcc \
    drivers/net/ethernet/myricom/myri10ge/myri10ge.o
```

---

**riscv — `CONFIG_RISCV_USER_CFI` + Binutils 2.44 breaks VDSO CFI build** (patch written 2026-08-03, not yet merged):
See 2026-07-25 entry. Same linker warning cascade → `vdso-cfi.so.dbg` not created → `nm`
fails → `vdso-cfi-offsets.h` empty → `__vdso_rt_sigreturn_cfi_offset` undeclared in `signal.c`.
The patch (`arch/riscv/Kconfig` + `scripts/riscv-ld-cfi-prop-test.sh`) was written on
`linux-dev` branch `fix/20260803_riscv_user_cfi_ld_cfi_prop_test` but has not appeared in
Linus' tree as of v7.2-rc7.

```
riscv64-linux-gnu-ld: warning: vdso_cfi/*.o: unsupported GNU_PROPERTY_TYPE (5) type: 0xc0000000
riscv64-linux-gnu-nm: 'arch/riscv/kernel/vdso_cfi/vdso-cfi.so.dbg': No such file
arch/riscv/include/asm/vdso.h:31:51: error: '__vdso_rt_sigreturn_cfi_offset' undeclared
```

**Reproduce:**
```sh
make O=/tmp/riscvcfi-repro ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- allmodconfig
make O=/tmp/riscvcfi-repro ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- -j$(nproc)
# Or just the linker step:
make O=/tmp/riscvcfi-repro ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- \
    arch/riscv/kernel/vdso_cfi/vdso-cfi.so.dbg
```

---

## Finding Status Summary

| Status | Count |
|--------|-------|
| Open   | 11    |
| Resolved | 19  |
| Won't fix | 0  |
| Reconsider later | 0 |
