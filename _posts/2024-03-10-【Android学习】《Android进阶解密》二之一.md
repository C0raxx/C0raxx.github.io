---
layout:     post
title:      【Android学习】《Android进阶解密》二之一
subtitle:   工具学习
date:       2024-03-10
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android进阶解密】
    - 【逆向】
    - 【Android】
---

# 第二章 Android系统启动

Android系统启动与很多内容都有关联，比如应用进程启动流程、四大组件原理、AMS、ClassLoader等，而ClassLoader又是热修复和插件化的基础，可见Android系统启动是十分重要并且需要首先学习的知识点。

![image-20240311081147537](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403110913167.png)

## init进程启动过程

init是Android用户空间第一个进程，进程号1，创建zygote（孵化器）和属性服务等。

init进程是由多个源文件共同组成的，这些文件位于源码目录system/core/init中。

一部分

![image-20240311080555630](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403110913169.png)

### 引入init进程

1. 当电源按下时引导芯片代码从预定义的地方（固化在ROM）开始执行。加载**引导程序BootLoader**到RAM中，然后执行。
2. BootLoader是在Android操作系统开始运行前的一个小程序，它的主要作用是把系统OS拉起来并运行。
3. 当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。在内核完成系统设置后，它首先在系统文件中寻找init.rc文件，并启动init进程。
4. init进程做的工作比较多，主要用来初始化和启动属性服务，也用来启动Zygote进程。



```
总结 当我们按下启动电源时，系统启动后会加载引导程序，引导程序又启动Linux 内核，在Linux 内核加载完成后，第一件事就是要启动init 进程。
```



### init进程的入口函数

来到上面的第三步

linux内核加载后，在系统文件中找init.rc，并启动init，（看init的main函数）。

```c++
/*
 * Copyright (C) 2008 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#include <ctype.h>
#include <dirent.h>
#include <errno.h>
#include <fcntl.h>
#include <inttypes.h>
#include <keyutils.h>
#include <libgen.h>
#include <paths.h>
#include <signal.h>
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/epoll.h>
#include <sys/mount.h>
#include <sys/socket.h>
#include <sys/stat.h>
#include <sys/sysmacros.h>
#include <sys/types.h>
#include <sys/un.h>
#include <sys/wait.h>
#include <unistd.h>

#include <selinux/selinux.h>
#include <selinux/label.h>
#include <selinux/android.h>

#include <android-base/file.h>
#include <android-base/properties.h>
#include <android-base/stringprintf.h>
#include <android-base/strings.h>
#include <android-base/unique_fd.h>
#include <libavb/libavb.h>
#include <private/android_filesystem_config.h>

#include <fstream>
#include <memory>
#include <vector>

#include "action.h"
#include "bootchart.h"
#include "devices.h"
#include "import_parser.h"
#include "init.h"
#include "init_first_stage.h"
#include "init_parser.h"
#include "keychords.h"
#include "log.h"
#include "property_service.h"
#include "reboot.h"
#include "service.h"
#include "signal_handler.h"
#include "ueventd.h"
#include "util.h"
#include "watchdogd.h"

using android::base::GetProperty;
using android::base::StringPrintf;

struct selabel_handle *sehandle;
struct selabel_handle *sehandle_prop;

static int property_triggers_enabled = 0;

static char qemu[32];

std::string default_console = "/dev/console";
static time_t process_needs_restart_at;

const char *ENV[32];

static int epoll_fd = -1;

static std::unique_ptr<Timer> waiting_for_prop(nullptr);
static std::string wait_prop_name;
static std::string wait_prop_value;

void register_epoll_handler(int fd, void (*fn)()) {
    epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.ptr = reinterpret_cast<void*>(fn);
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &ev) == -1) {
        PLOG(ERROR) << "epoll_ctl failed";
    }
}

/* add_environment - add "key=value" to the current environment */
int add_environment(const char *key, const char *val)
{
    size_t n;
    size_t key_len = strlen(key);

    /* The last environment entry is reserved to terminate the list */
    for (n = 0; n < (arraysize(ENV) - 1); n++) {

        /* Delete any existing entry for this key */
        if (ENV[n] != NULL) {
            size_t entry_key_len = strcspn(ENV[n], "=");
            if ((entry_key_len == key_len) && (strncmp(ENV[n], key, entry_key_len) == 0)) {
                free((char*)ENV[n]);
                ENV[n] = NULL;
            }
        }

        /* Add entry if a free slot is available */
        if (ENV[n] == NULL) {
            char* entry;
            asprintf(&entry, "%s=%s", key, val);
            ENV[n] = entry;
            return 0;
        }
    }

    LOG(ERROR) << "No env. room to store: '" << key << "':'" << val << "'";

    return -1;
}

bool start_waiting_for_property(const char *name, const char *value)
{
    if (waiting_for_prop) {
        return false;
    }
    if (GetProperty(name, "") != value) {
        // Current property value is not equal to expected value
        wait_prop_name = name;
        wait_prop_value = value;
        waiting_for_prop.reset(new Timer());
    } else {
        LOG(INFO) << "start_waiting_for_property(\""
                  << name << "\", \"" << value << "\"): already set";
    }
    return true;
}

void property_changed(const std::string& name, const std::string& value) {
    // If the property is sys.powerctl, we bypass the event queue and immediately handle it.
    // This is to ensure that init will always and immediately shutdown/reboot, regardless of
    // if there are other pending events to process or if init is waiting on an exec service or
    // waiting on a property.
    if (name == "sys.powerctl") HandlePowerctlMessage(value);

    if (property_triggers_enabled)
        ActionManager::GetInstance().QueuePropertyTrigger(name, value);
    if (waiting_for_prop) {
        if (wait_prop_name == name && wait_prop_value == value) {
            wait_prop_name.clear();
            wait_prop_value.clear();
            LOG(INFO) << "Wait for property took " << *waiting_for_prop;
            waiting_for_prop.reset();
        }
    }
}

static void restart_processes()
{
    process_needs_restart_at = 0;
    ServiceManager::GetInstance().ForEachServiceWithFlags(SVC_RESTARTING, [](Service* s) {
        s->RestartIfNeeded(&process_needs_restart_at);
    });
}

void handle_control_message(const std::string& msg, const std::string& name) {
    Service* svc = ServiceManager::GetInstance().FindServiceByName(name);
    if (svc == nullptr) {
        LOG(ERROR) << "no such service '" << name << "'";
        return;
    }

    if (msg == "start") {
        svc->Start();
    } else if (msg == "stop") {
        svc->Stop();
    } else if (msg == "restart") {
        svc->Restart();
    } else {
        LOG(ERROR) << "unknown control msg '" << msg << "'";
    }
}

static int wait_for_coldboot_done_action(const std::vector<std::string>& args) {
    Timer t;

    LOG(VERBOSE) << "Waiting for " COLDBOOT_DONE "...";

    // Historically we had a 1s timeout here because we weren't otherwise
    // tracking boot time, and many OEMs made their sepolicy regular
    // expressions too expensive (http://b/19899875).

    // Now we're tracking boot time, just log the time taken to a system
    // property. We still panic if it takes more than a minute though,
    // because any build that slow isn't likely to boot at all, and we'd
    // rather any test lab devices fail back to the bootloader.
    if (wait_for_file(COLDBOOT_DONE, 60s) < 0) {
        LOG(ERROR) << "Timed out waiting for " COLDBOOT_DONE;
        panic();
    }

    property_set("ro.boottime.init.cold_boot_wait", std::to_string(t.duration_ms()).c_str());
    return 0;
}

/*
 * Writes 512 bytes of output from Hardware RNG (/dev/hw_random, backed
 * by Linux kernel's hw_random framework) into Linux RNG's via /dev/urandom.
 * Does nothing if Hardware RNG is not present.
 *
 * Since we don't yet trust the quality of Hardware RNG, these bytes are not
 * mixed into the primary pool of Linux RNG and the entropy estimate is left
 * unmodified.
 *
 * If the HW RNG device /dev/hw_random is present, we require that at least
 * 512 bytes read from it are written into Linux RNG. QA is expected to catch
 * devices/configurations where these I/O operations are blocking for a long
 * time. We do not reboot or halt on failures, as this is a best-effort
 * attempt.
 */
static int mix_hwrng_into_linux_rng_action(const std::vector<std::string>& args)
{
    int result = -1;
    int hwrandom_fd = -1;
    int urandom_fd = -1;
    char buf[512];
    ssize_t chunk_size;
    size_t total_bytes_written = 0;

    hwrandom_fd = TEMP_FAILURE_RETRY(
            open("/dev/hw_random", O_RDONLY | O_NOFOLLOW | O_CLOEXEC));
    if (hwrandom_fd == -1) {
        if (errno == ENOENT) {
            LOG(ERROR) << "/dev/hw_random not found";
            // It's not an error to not have a Hardware RNG.
            result = 0;
        } else {
            PLOG(ERROR) << "Failed to open /dev/hw_random";
        }
        goto ret;
    }

    urandom_fd = TEMP_FAILURE_RETRY(
            open("/dev/urandom", O_WRONLY | O_NOFOLLOW | O_CLOEXEC));
    if (urandom_fd == -1) {
        PLOG(ERROR) << "Failed to open /dev/urandom";
        goto ret;
    }

    while (total_bytes_written < sizeof(buf)) {
        chunk_size = TEMP_FAILURE_RETRY(
                read(hwrandom_fd, buf, sizeof(buf) - total_bytes_written));
        if (chunk_size == -1) {
            PLOG(ERROR) << "Failed to read from /dev/hw_random";
            goto ret;
        } else if (chunk_size == 0) {
            LOG(ERROR) << "Failed to read from /dev/hw_random: EOF";
            goto ret;
        }

        chunk_size = TEMP_FAILURE_RETRY(write(urandom_fd, buf, chunk_size));
        if (chunk_size == -1) {
            PLOG(ERROR) << "Failed to write to /dev/urandom";
            goto ret;
        }
        total_bytes_written += chunk_size;
    }

    LOG(INFO) << "Mixed " << total_bytes_written << " bytes from /dev/hw_random into /dev/urandom";
    result = 0;

ret:
    if (hwrandom_fd != -1) {
        close(hwrandom_fd);
    }
    if (urandom_fd != -1) {
        close(urandom_fd);
    }
    return result;
}

static void security_failure() {
    LOG(ERROR) << "Security failure...";
    panic();
}

static bool set_highest_available_option_value(std::string path, int min, int max)
{
    std::ifstream inf(path, std::fstream::in);
    if (!inf) {
        LOG(ERROR) << "Cannot open for reading: " << path;
        return false;
    }

    int current = max;
    while (current >= min) {
        // try to write out new value
        std::string str_val = std::to_string(current);
        std::ofstream of(path, std::fstream::out);
        if (!of) {
            LOG(ERROR) << "Cannot open for writing: " << path;
            return false;
        }
        of << str_val << std::endl;
        of.close();

        // check to make sure it was recorded
        inf.seekg(0);
        std::string str_rec;
        inf >> str_rec;
        if (str_val.compare(str_rec) == 0) {
            break;
        }
        current--;
    }
    inf.close();

    if (current < min) {
        LOG(ERROR) << "Unable to set minimum option value " << min << " in " << path;
        return false;
    }
    return true;
}

#define MMAP_RND_PATH "/proc/sys/vm/mmap_rnd_bits"
#define MMAP_RND_COMPAT_PATH "/proc/sys/vm/mmap_rnd_compat_bits"

/* __attribute__((unused)) due to lack of mips support: see mips block
 * in set_mmap_rnd_bits_action */
static bool __attribute__((unused)) set_mmap_rnd_bits_min(int start, int min, bool compat) {
    std::string path;
    if (compat) {
        path = MMAP_RND_COMPAT_PATH;
    } else {
        path = MMAP_RND_PATH;
    }

    return set_highest_available_option_value(path, min, start);
}

/*
 * Set /proc/sys/vm/mmap_rnd_bits and potentially
 * /proc/sys/vm/mmap_rnd_compat_bits to the maximum supported values.
 * Returns -1 if unable to set these to an acceptable value.
 *
 * To support this sysctl, the following upstream commits are needed:
 *
 * d07e22597d1d mm: mmap: add new /proc tunable for mmap_base ASLR
 * e0c25d958f78 arm: mm: support ARCH_MMAP_RND_BITS
 * 8f0d3aa9de57 arm64: mm: support ARCH_MMAP_RND_BITS
 * 9e08f57d684a x86: mm: support ARCH_MMAP_RND_BITS
 * ec9ee4acd97c drivers: char: random: add get_random_long()
 * 5ef11c35ce86 mm: ASLR: use get_random_long()
 */
static int set_mmap_rnd_bits_action(const std::vector<std::string>& args)
{
    int ret = -1;

    /* values are arch-dependent */
#if defined(__aarch64__)
    /* arm64 supports 18 - 33 bits depending on pagesize and VA_SIZE */
    if (set_mmap_rnd_bits_min(33, 24, false)
            && set_mmap_rnd_bits_min(16, 16, true)) {
        ret = 0;
    }
#elif defined(__x86_64__)
    /* x86_64 supports 28 - 32 bits */
    if (set_mmap_rnd_bits_min(32, 32, false)
            && set_mmap_rnd_bits_min(16, 16, true)) {
        ret = 0;
    }
#elif defined(__arm__) || defined(__i386__)
    /* check to see if we're running on 64-bit kernel */
    bool h64 = !access(MMAP_RND_COMPAT_PATH, F_OK);
    /* supported 32-bit architecture must have 16 bits set */
    if (set_mmap_rnd_bits_min(16, 16, h64)) {
        ret = 0;
    }
#elif defined(__mips__) || defined(__mips64__)
    // TODO: add mips support b/27788820
    ret = 0;
#else
    LOG(ERROR) << "Unknown architecture";
#endif

    if (ret == -1) {
        LOG(ERROR) << "Unable to set adequate mmap entropy value!";
        security_failure();
    }
    return ret;
}

#define KPTR_RESTRICT_PATH "/proc/sys/kernel/kptr_restrict"
#define KPTR_RESTRICT_MINVALUE 2
#define KPTR_RESTRICT_MAXVALUE 4

/* Set kptr_restrict to the highest available level.
 *
 * Aborts if unable to set this to an acceptable value.
 */
static int set_kptr_restrict_action(const std::vector<std::string>& args)
{
    std::string path = KPTR_RESTRICT_PATH;

    if (!set_highest_available_option_value(path, KPTR_RESTRICT_MINVALUE, KPTR_RESTRICT_MAXVALUE)) {
        LOG(ERROR) << "Unable to set adequate kptr_restrict value!";
        security_failure();
    }
    return 0;
}

static int keychord_init_action(const std::vector<std::string>& args)
{
    keychord_init();
    return 0;
}

static int console_init_action(const std::vector<std::string>& args)
{
    std::string console = GetProperty("ro.boot.console", "");
    if (!console.empty()) {
        default_console = "/dev/" + console;
    }
    return 0;
}

static void import_kernel_nv(const std::string& key, const std::string& value, bool for_emulator) {
    if (key.empty()) return;

    if (for_emulator) {
        // In the emulator, export any kernel option with the "ro.kernel." prefix.
        property_set(StringPrintf("ro.kernel.%s", key.c_str()).c_str(), value.c_str());
        return;
    }

    if (key == "qemu") {
        strlcpy(qemu, value.c_str(), sizeof(qemu));
    } else if (android::base::StartsWith(key, "androidboot.")) {
        property_set(StringPrintf("ro.boot.%s", key.c_str() + 12).c_str(), value.c_str());
    }
}

static void export_oem_lock_status() {
    if (!android::base::GetBoolProperty("ro.oem_unlock_supported", false)) {
        return;
    }

    std::string value = GetProperty("ro.boot.verifiedbootstate", "");

    if (!value.empty()) {
        property_set("ro.boot.flash.locked", value == "orange" ? "0" : "1");
    }
}

static void export_kernel_boot_props() {
    struct {
        const char *src_prop;
        const char *dst_prop;
        const char *default_value;
    } prop_map[] = {
        { "ro.boot.serialno",   "ro.serialno",   "", },
        { "ro.boot.mode",       "ro.bootmode",   "unknown", },
        { "ro.boot.baseband",   "ro.baseband",   "unknown", },
        { "ro.boot.bootloader", "ro.bootloader", "unknown", },
        { "ro.boot.hardware",   "ro.hardware",   "unknown", },
        { "ro.boot.revision",   "ro.revision",   "0", },
    };
    for (size_t i = 0; i < arraysize(prop_map); i++) {
        std::string value = GetProperty(prop_map[i].src_prop, "");
        property_set(prop_map[i].dst_prop, (!value.empty()) ? value.c_str() : prop_map[i].default_value);
    }
}

static void process_kernel_dt() {
    if (!is_android_dt_value_expected("compatible", "android,firmware")) {
        return;
    }

    std::unique_ptr<DIR, int (*)(DIR*)> dir(opendir(kAndroidDtDir.c_str()), closedir);
    if (!dir) return;

    std::string dt_file;
    struct dirent *dp;
    while ((dp = readdir(dir.get())) != NULL) {
        if (dp->d_type != DT_REG || !strcmp(dp->d_name, "compatible") || !strcmp(dp->d_name, "name")) {
            continue;
        }

        std::string file_name = kAndroidDtDir + dp->d_name;

        android::base::ReadFileToString(file_name, &dt_file);
        std::replace(dt_file.begin(), dt_file.end(), ',', '.');

        std::string property_name = StringPrintf("ro.boot.%s", dp->d_name);
        property_set(property_name.c_str(), dt_file.c_str());
    }
}

static void process_kernel_cmdline() {
    // The first pass does the common stuff, and finds if we are in qemu.
    // The second pass is only necessary for qemu to export all kernel params
    // as properties.
    import_kernel_cmdline(false, import_kernel_nv);
    if (qemu[0]) import_kernel_cmdline(true, import_kernel_nv);
}

static int property_enable_triggers_action(const std::vector<std::string>& args)
{
    /* Enable property triggers. */
    property_triggers_enabled = 1;
    return 0;
}

static int queue_property_triggers_action(const std::vector<std::string>& args)
{
    ActionManager::GetInstance().QueueBuiltinAction(property_enable_triggers_action, "enable_property_trigger");
    ActionManager::GetInstance().QueueAllPropertyTriggers();
    return 0;
}

static void selinux_init_all_handles(void)
{
    sehandle = selinux_android_file_context_handle();
    selinux_android_set_sehandle(sehandle);
    sehandle_prop = selinux_android_prop_context_handle();
}

enum selinux_enforcing_status { SELINUX_PERMISSIVE, SELINUX_ENFORCING };

static selinux_enforcing_status selinux_status_from_cmdline() {
    selinux_enforcing_status status = SELINUX_ENFORCING;

    import_kernel_cmdline(false, [&](const std::string& key, const std::string& value, bool in_qemu) {
        if (key == "androidboot.selinux" && value == "permissive") {
            status = SELINUX_PERMISSIVE;
        }
    });

    return status;
}

static bool selinux_is_enforcing(void)
{
    if (ALLOW_PERMISSIVE_SELINUX) {
        return selinux_status_from_cmdline() == SELINUX_ENFORCING;
    }
    return true;
}

static int audit_callback(void *data, security_class_t /*cls*/, char *buf, size_t len) {

    property_audit_data *d = reinterpret_cast<property_audit_data*>(data);

    if (!d || !d->name || !d->cr) {
        LOG(ERROR) << "audit_callback invoked with null data arguments!";
        return 0;
    }

    snprintf(buf, len, "property=%s pid=%d uid=%d gid=%d", d->name,
            d->cr->pid, d->cr->uid, d->cr->gid);
    return 0;
}

/*
 * Forks, executes the provided program in the child, and waits for the completion in the parent.
 * Child's stderr is captured and logged using LOG(ERROR).
 *
 * Returns true if the child exited with status code 0, returns false otherwise.
 */
static bool fork_execve_and_wait_for_completion(const char* filename, char* const argv[],
                                                char* const envp[]) {
    // Create a pipe used for redirecting child process's output.
    // * pipe_fds[0] is the FD the parent will use for reading.
    // * pipe_fds[1] is the FD the child will use for writing.
    int pipe_fds[2];
    if (pipe(pipe_fds) == -1) {
        PLOG(ERROR) << "Failed to create pipe";
        return false;
    }

    pid_t child_pid = fork();
    if (child_pid == -1) {
        PLOG(ERROR) << "Failed to fork for " << filename;
        return false;
    }

    if (child_pid == 0) {
        // fork succeeded -- this is executing in the child process

        // Close the pipe FD not used by this process
        TEMP_FAILURE_RETRY(close(pipe_fds[0]));

        // Redirect stderr to the pipe FD provided by the parent
        if (TEMP_FAILURE_RETRY(dup2(pipe_fds[1], STDERR_FILENO)) == -1) {
            PLOG(ERROR) << "Failed to redirect stderr of " << filename;
            _exit(127);
            return false;
        }
        TEMP_FAILURE_RETRY(close(pipe_fds[1]));

        if (execve(filename, argv, envp) == -1) {
            PLOG(ERROR) << "Failed to execve " << filename;
            return false;
        }
        // Unreachable because execve will have succeeded and replaced this code
        // with child process's code.
        _exit(127);
        return false;
    } else {
        // fork succeeded -- this is executing in the original/parent process

        // Close the pipe FD not used by this process
        TEMP_FAILURE_RETRY(close(pipe_fds[1]));

        // Log the redirected output of the child process.
        // It's unfortunate that there's no standard way to obtain an istream for a file descriptor.
        // As a result, we're buffering all output and logging it in one go at the end of the
        // invocation, instead of logging it as it comes in.
        const int child_out_fd = pipe_fds[0];
        std::string child_output;
        if (!android::base::ReadFdToString(child_out_fd, &child_output)) {
            PLOG(ERROR) << "Failed to capture full output of " << filename;
        }
        TEMP_FAILURE_RETRY(close(child_out_fd));
        if (!child_output.empty()) {
            // Log captured output, line by line, because LOG expects to be invoked for each line
            std::istringstream in(child_output);
            std::string line;
            while (std::getline(in, line)) {
                LOG(ERROR) << filename << ": " << line;
            }
        }

        // Wait for child to terminate
        int status;
        if (TEMP_FAILURE_RETRY(waitpid(child_pid, &status, 0)) != child_pid) {
            PLOG(ERROR) << "Failed to wait for " << filename;
            return false;
        }

        if (WIFEXITED(status)) {
            int status_code = WEXITSTATUS(status);
            if (status_code == 0) {
                return true;
            } else {
                LOG(ERROR) << filename << " exited with status " << status_code;
            }
        } else if (WIFSIGNALED(status)) {
            LOG(ERROR) << filename << " killed by signal " << WTERMSIG(status);
        } else if (WIFSTOPPED(status)) {
            LOG(ERROR) << filename << " stopped by signal " << WSTOPSIG(status);
        } else {
            LOG(ERROR) << "waitpid for " << filename << " returned unexpected status: " << status;
        }

        return false;
    }
}

static bool read_first_line(const char* file, std::string* line) {
    line->clear();

    std::string contents;
    if (!android::base::ReadFileToString(file, &contents, true /* follow symlinks */)) {
        return false;
    }
    std::istringstream in(contents);
    std::getline(in, *line);
    return true;
}

static bool selinux_find_precompiled_split_policy(std::string* file) {
    file->clear();

    static constexpr const char precompiled_sepolicy[] = "/vendor/etc/selinux/precompiled_sepolicy";
    if (access(precompiled_sepolicy, R_OK) == -1) {
        return false;
    }
    std::string actual_plat_id;
    if (!read_first_line("/system/etc/selinux/plat_and_mapping_sepolicy.cil.sha256",
                         &actual_plat_id)) {
        PLOG(INFO) << "Failed to read "
                      "/system/etc/selinux/plat_and_mapping_sepolicy.cil.sha256";
        return false;
    }
    std::string precompiled_plat_id;
    if (!read_first_line("/vendor/etc/selinux/precompiled_sepolicy.plat_and_mapping.sha256",
                         &precompiled_plat_id)) {
        PLOG(INFO) << "Failed to read "
                      "/vendor/etc/selinux/"
                      "precompiled_sepolicy.plat_and_mapping.sha256";
        return false;
    }
    if ((actual_plat_id.empty()) || (actual_plat_id != precompiled_plat_id)) {
        return false;
    }

    *file = precompiled_sepolicy;
    return true;
}

static bool selinux_get_vendor_mapping_version(std::string* plat_vers) {
    if (!read_first_line("/vendor/etc/selinux/plat_sepolicy_vers.txt", plat_vers)) {
        PLOG(ERROR) << "Failed to read /vendor/etc/selinux/plat_sepolicy_vers.txt";
        return false;
    }
    if (plat_vers->empty()) {
        LOG(ERROR) << "No version present in plat_sepolicy_vers.txt";
        return false;
    }
    return true;
}

static constexpr const char plat_policy_cil_file[] = "/system/etc/selinux/plat_sepolicy.cil";

static bool selinux_is_split_policy_device() { return access(plat_policy_cil_file, R_OK) != -1; }

/*
 * Loads SELinux policy split across platform/system and non-platform/vendor files.
 *
 * Returns true upon success, false otherwise (failure cause is logged).
 */
static bool selinux_load_split_policy() {
    // IMPLEMENTATION NOTE: Split policy consists of three CIL files:
    // * platform -- policy needed due to logic contained in the system image,
    // * non-platform -- policy needed due to logic contained in the vendor image,
    // * mapping -- mapping policy which helps preserve forward-compatibility of non-platform policy
    //   with newer versions of platform policy.
    //
    // secilc is invoked to compile the above three policy files into a single monolithic policy
    // file. This file is then loaded into the kernel.

    // Load precompiled policy from vendor image, if a matching policy is found there. The policy
    // must match the platform policy on the system image.
    std::string precompiled_sepolicy_file;
    if (selinux_find_precompiled_split_policy(&precompiled_sepolicy_file)) {
        android::base::unique_fd fd(
            open(precompiled_sepolicy_file.c_str(), O_RDONLY | O_CLOEXEC | O_BINARY));
        if (fd != -1) {
            if (selinux_android_load_policy_from_fd(fd, precompiled_sepolicy_file.c_str()) < 0) {
                LOG(ERROR) << "Failed to load SELinux policy from " << precompiled_sepolicy_file;
                return false;
            }
            return true;
        }
    }
    // No suitable precompiled policy could be loaded

    LOG(INFO) << "Compiling SELinux policy";

    // Determine the highest policy language version supported by the kernel
    set_selinuxmnt("/sys/fs/selinux");
    int max_policy_version = security_policyvers();
    if (max_policy_version == -1) {
        PLOG(ERROR) << "Failed to determine highest policy version supported by kernel";
        return false;
    }

    // We store the output of the compilation on /dev because this is the most convenient tmpfs
    // storage mount available this early in the boot sequence.
    char compiled_sepolicy[] = "/dev/sepolicy.XXXXXX";
    android::base::unique_fd compiled_sepolicy_fd(mkostemp(compiled_sepolicy, O_CLOEXEC));
    if (compiled_sepolicy_fd < 0) {
        PLOG(ERROR) << "Failed to create temporary file " << compiled_sepolicy;
        return false;
    }

    // Determine which mapping file to include
    std::string vend_plat_vers;
    if (!selinux_get_vendor_mapping_version(&vend_plat_vers)) {
        return false;
    }
    std::string mapping_file("/system/etc/selinux/mapping/" + vend_plat_vers + ".cil");
    // clang-format off
    const char* compile_args[] = {
        "/system/bin/secilc",
        plat_policy_cil_file,
        "-M", "true", "-G", "-N",
        // Target the highest policy language version supported by the kernel
        "-c", std::to_string(max_policy_version).c_str(),
        mapping_file.c_str(),
        "/vendor/etc/selinux/nonplat_sepolicy.cil",
        "-o", compiled_sepolicy,
        // We don't care about file_contexts output by the compiler
        "-f", "/sys/fs/selinux/null",  // /dev/null is not yet available
        nullptr};
    // clang-format on

    if (!fork_execve_and_wait_for_completion(compile_args[0], (char**)compile_args, (char**)ENV)) {
        unlink(compiled_sepolicy);
        return false;
    }
    unlink(compiled_sepolicy);

    LOG(INFO) << "Loading compiled SELinux policy";
    if (selinux_android_load_policy_from_fd(compiled_sepolicy_fd, compiled_sepolicy) < 0) {
        LOG(ERROR) << "Failed to load SELinux policy from " << compiled_sepolicy;
        return false;
    }

    return true;
}

/*
 * Loads SELinux policy from a monolithic file.
 *
 * Returns true upon success, false otherwise (failure cause is logged).
 */
static bool selinux_load_monolithic_policy() {
    LOG(VERBOSE) << "Loading SELinux policy from monolithic file";
    if (selinux_android_load_policy() < 0) {
        PLOG(ERROR) << "Failed to load monolithic SELinux policy";
        return false;
    }
    return true;
}

/*
 * Loads SELinux policy into the kernel.
 *
 * Returns true upon success, false otherwise (failure cause is logged).
 */
static bool selinux_load_policy() {
    return selinux_is_split_policy_device() ? selinux_load_split_policy()
                                            : selinux_load_monolithic_policy();
}

static void selinux_initialize(bool in_kernel_domain) {
    Timer t;

    selinux_callback cb;
    cb.func_log = selinux_klog_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);
    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);

    if (in_kernel_domain) {
        LOG(INFO) << "Loading SELinux policy";
        if (!selinux_load_policy()) {
            panic();
        }

        bool kernel_enforcing = (security_getenforce() == 1);
        bool is_enforcing = selinux_is_enforcing();
        if (kernel_enforcing != is_enforcing) {
            if (security_setenforce(is_enforcing)) {
                PLOG(ERROR) << "security_setenforce(%s) failed" << (is_enforcing ? "true" : "false");
                security_failure();
            }
        }

        if (!write_file("/sys/fs/selinux/checkreqprot", "0")) {
            security_failure();
        }

        // init's first stage can't set properties, so pass the time to the second stage.
        setenv("INIT_SELINUX_TOOK", std::to_string(t.duration_ms()).c_str(), 1);
    } else {
        selinux_init_all_handles();
    }
}

// The files and directories that were created before initial sepolicy load
// need to have their security context restored to the proper value.
// This must happen before /dev is populated by ueventd.
static void selinux_restore_context() {
    LOG(INFO) << "Running restorecon...";
    restorecon("/dev");
    restorecon("/dev/kmsg");
    restorecon("/dev/socket");
    restorecon("/dev/random");
    restorecon("/dev/urandom");
    restorecon("/dev/__properties__");

    restorecon("/file_contexts.bin");
    restorecon("/plat_file_contexts");
    restorecon("/nonplat_file_contexts");
    restorecon("/plat_property_contexts");
    restorecon("/nonplat_property_contexts");
    restorecon("/plat_seapp_contexts");
    restorecon("/nonplat_seapp_contexts");
    restorecon("/plat_service_contexts");
    restorecon("/nonplat_service_contexts");
    restorecon("/plat_hwservice_contexts");
    restorecon("/nonplat_hwservice_contexts");
    restorecon("/sepolicy");
    restorecon("/vndservice_contexts");

    restorecon("/sys", SELINUX_ANDROID_RESTORECON_RECURSE);
    restorecon("/dev/block", SELINUX_ANDROID_RESTORECON_RECURSE);
    restorecon("/dev/device-mapper");
}

// Set the UDC controller for the ConfigFS USB Gadgets.
// Read the UDC controller in use from "/sys/class/udc".
// In case of multiple UDC controllers select the first one.
static void set_usb_controller() {
    std::unique_ptr<DIR, decltype(&closedir)>dir(opendir("/sys/class/udc"), closedir);
    if (!dir) return;

    dirent* dp;
    while ((dp = readdir(dir.get())) != nullptr) {
        if (dp->d_name[0] == '.') continue;

        property_set("sys.usb.controller", dp->d_name);
        break;
    }
}

static void install_reboot_signal_handlers() {
    // Instead of panic'ing the kernel as is the default behavior when init crashes,
    // we prefer to reboot to bootloader on development builds, as this will prevent
    // boot looping bad configurations and allow both developers and test farms to easily
    // recover.
    struct sigaction action;
    memset(&action, 0, sizeof(action));
    sigfillset(&action.sa_mask);
    action.sa_handler = [](int) {
        // panic() reboots to bootloader
        panic();
    };
    action.sa_flags = SA_RESTART;
    sigaction(SIGABRT, &action, nullptr);
    sigaction(SIGBUS, &action, nullptr);
    sigaction(SIGFPE, &action, nullptr);
    sigaction(SIGILL, &action, nullptr);
    sigaction(SIGSEGV, &action, nullptr);
#if defined(SIGSTKFLT)
    sigaction(SIGSTKFLT, &action, nullptr);
#endif
    sigaction(SIGSYS, &action, nullptr);
    sigaction(SIGTRAP, &action, nullptr);
}

int main(int argc, char** argv) {
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    if (!strcmp(basename(argv[0]), "watchdogd")) {
        return watchdogd_main(argc, argv);
    }

    if (REBOOT_BOOTLOADER_ON_PANIC) { //用于在系统发生严重错误（如 panic）时重新启动引导加载程序（bootloader）
        install_reboot_signal_handlers();
    }

    add_environment("PATH", _PATH_DEFPATH);

    bool is_first_stage = (getenv("INIT_SECOND_STAGE") == nullptr);

    if (is_first_stage) {
        boot_clock::time_point start_time = boot_clock::now();

        // Clear the umask.
        umask(0);

        // Get the basic filesystem setup we need put together in the initramdisk
        // on / and then we'll let the rc file figure out the rest.
		//创建和挂在启动所需的文件目录

		mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        #define MAKE_STR(x) __STRING(x)
        mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
        // Don't expose the raw commandline to unprivileged processes.
        chmod("/proc/cmdline", 0440);
        gid_t groups[] = { AID_READPROC };
        setgroups(arraysize(groups), groups);
        mount("sysfs", "/sys", "sysfs", 0, NULL);
        mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL);
        mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11));
        mknod("/dev/random", S_IFCHR | 0666, makedev(1, 8));
        mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9));

        // Now that tmpfs is mounted on /dev and we have /dev/kmsg, we can actually
        // talk to the outside world...
        //初始化kernel的log 这样可以从外界获取kernellog了
        InitKernelLogging(argv);

        LOG(INFO) << "init first stage started!";

        if (!DoFirstStageMount()) {
            LOG(ERROR) << "Failed to mount required partitions early ...";
            panic();
        }

        SetInitAvbVersionInRecovery();

        // Set up SELinux, loading the SELinux policy.
        selinux_initialize(true);

        // We're in the kernel domain, so re-exec init to transition to the init domain now
        // that the SELinux policy has been loaded.
        if (restorecon("/init") == -1) {
            PLOG(ERROR) << "restorecon failed";
            security_failure();
        }

        setenv("INIT_SECOND_STAGE", "true", 1);

        static constexpr uint32_t kNanosecondsPerMillisecond = 1e6;
        uint64_t start_ms = start_time.time_since_epoch().count() / kNanosecondsPerMillisecond;
        setenv("INIT_STARTED_AT", StringPrintf("%" PRIu64, start_ms).c_str(), 1);

        char* path = argv[0];
        char* args[] = { path, nullptr };
        execv(path, args);

        // execv() only returns if an error happened, in which case we
        // panic and never fall through this conditional.
        PLOG(ERROR) << "execv(\"" << path << "\") failed";
        security_failure();
    }

    // At this point we're in the second stage of init.
    InitKernelLogging(argv);
    LOG(INFO) << "init second stage started!";

    // Set up a session keyring that all processes will have access to. It
    // will hold things like FBE encryption keys. No process should override
    // its session keyring.
    keyctl(KEYCTL_GET_KEYRING_ID, KEY_SPEC_SESSION_KEYRING, 1);

    // Indicate that booting is in progress to background fw loaders, etc.
    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
	//属性初始化
    property_init();

    // If arguments are passed both on the command line and in DT,
    // properties set in DT always have priority over the command-line ones.
    process_kernel_dt();
    process_kernel_cmdline();

    // Propagate the kernel variables to internal variables
    // used by init as well as the current required properties.
    export_kernel_boot_props();

    // Make the time that init started available for bootstat to log.
    property_set("ro.boottime.init", getenv("INIT_STARTED_AT"));
    property_set("ro.boottime.init.selinux", getenv("INIT_SELINUX_TOOK"));

    // Set libavb version for Framework-only OTA match in Treble build.
    const char* avb_version = getenv("INIT_AVB_VERSION");
    if (avb_version) property_set("ro.boot.avb_version", avb_version);

    // Clean up our environment.
    unsetenv("INIT_SECOND_STAGE");
    unsetenv("INIT_STARTED_AT");
    unsetenv("INIT_SELINUX_TOOK");
    unsetenv("INIT_AVB_VERSION");

    // Now set up SELinux for second stage.
    selinux_initialize(false);
    selinux_restore_context();
    //epoll句柄的创建
    epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    if (epoll_fd == -1) {
        PLOG(ERROR) << "epoll_create1 failed";
        exit(1);
    }
    //设置子进程处理信号 如果子进程zygote异常退出 会调用设定的信号处理函数处理
    signal_handler_init();

    property_load_boot_defaults();
    export_oem_lock_status();
	//启动属性服务
    start_property_service();
    set_usb_controller();

    const BuiltinFunctionMap function_map;
    Action::set_function_map(&function_map);

    Parser& parser = Parser::GetInstance();
    parser.AddSectionParser("service",std::make_unique<ServiceParser>());
    parser.AddSectionParser("on", std::make_unique<ActionParser>());
    parser.AddSectionParser("import", std::make_unique<ImportParser>());
    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
		//解析init.rc 就是上面讲到的
        parser.ParseConfig("/init.rc");
        parser.set_is_system_etc_init_loaded(
                parser.ParseConfig("/system/etc/init"));
        parser.set_is_vendor_etc_init_loaded(
                parser.ParseConfig("/vendor/etc/init"));
        parser.set_is_odm_etc_init_loaded(parser.ParseConfig("/odm/etc/init"));
    } else {
        parser.ParseConfig(bootscript);
        parser.set_is_system_etc_init_loaded(true);
        parser.set_is_vendor_etc_init_loaded(true);
        parser.set_is_odm_etc_init_loaded(true);
    }

    // Turning this on and letting the INFO logging be discarded adds 0.2s to
    // Nexus 9 boot time, so it's disabled by default.
    if (false) parser.DumpState();

    ActionManager& am = ActionManager::GetInstance();

    am.QueueEventTrigger("early-init");

    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    am.QueueBuiltinAction(set_mmap_rnd_bits_action, "set_mmap_rnd_bits");
    am.QueueBuiltinAction(set_kptr_restrict_action, "set_kptr_restrict");
    am.QueueBuiltinAction(keychord_init_action, "keychord_init");
    am.QueueBuiltinAction(console_init_action, "console_init");

    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger("init");

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn't ready immediately after wait_for_coldboot_done
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");

    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }

    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

    while (true) {
        // By default, sleep until something happens.
        int epoll_timeout_ms = -1;

        if (!(waiting_for_prop || ServiceManager::GetInstance().IsWaitingForExec())) {
            am.ExecuteOneCommand();
        }
        if (!(waiting_for_prop || ServiceManager::GetInstance().IsWaitingForExec())) {
            restart_processes();

            // If there's a process that needs restarting, wake up in time for that.
            if (process_needs_restart_at != 0) {
                epoll_timeout_ms = (process_needs_restart_at - time(nullptr)) * 1000;
                if (epoll_timeout_ms < 0) epoll_timeout_ms = 0;
            }

            // If there's more work to do, wake up again immediately.
            if (am.HasMoreCommands()) epoll_timeout_ms = 0;
        }

        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, epoll_timeout_ms));
        if (nr == -1) {
            PLOG(ERROR) << "epoll_wait failed";
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }

    return 0;
}

```

我们只需关注主要的几点就可以了。

1. 在开始的时候创建和挂载启动所需的文件目录，其中挂载了tmpfs、devpts、proc、sysfs和selinuxfs共5种文件系统，这些都是系统运行时目录，顾名思义，只在系统运行时才会存在，系统停止时会消失。
2. property_init函数来对属性进行初始化，并在注释3处调用start_property_service函数启动属性服务
3. signal_handler_init 函数用于设置子进程信号处理函数，它被定义在system/core/init/signal_handler.cpp中，主要用于防止init进程的子进程成为僵尸进程，为了防止僵尸进程的出现，系统会在子进程暂停和终止的时候发出SIGCHLD信号，而signal_handler_init函数就是用来接收SIGCHLD信号的（其内部只处理进程终止的SIGCHLD信号）。

### 解析init.rc

代码很长

![image-20240311082647087](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403110913170.png)



on init是action类型语句

![image-20240311082705981](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403110913171.png)

创建zygote涉及service语句

![image-20240311082734440](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403110913172.png)

这里Android 8.0中对init.rc文件进行了拆分，每个服务对应一个rc文件。我们要分析的Zygote启动脚本则在init.zygoteXX.rc中定义 我们找到zygote64对应的

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
    group root readproc
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks
```

Service 用于通知init进程创建名为zygote的进程，这个进程执行程序的路径为/system/bin/app_process64①，其后面的代码是要传给app_process64的参数。

class main是zygote的classname为main

### 解析service类型语句

Action类型语句采用ActionParser来进行解析，Service 类型语句采用ServiceParser来进行解析

会用到两个函数：一个是ParseSection，它会解析Service的rc文件，比如上文讲到的init.zygote64.rc，**ParseSection**函数主要用来搭建Service的架子；另一个是**ParseLineSection**，用于解析子项。

ParseSection

```
bool ServiceParser::ParseSection(const std::vector<std::string>& args,
                                 std::string* err) {
    if (args.size() < 3) { 判断service是否有name与可执行程序
        *err = "services must have a name and a program";
        return false;
    }

    const std::string& name = args[1];
    if (!IsValidName(name)) { 看name是覅否有效
        *err = StringPrintf("invalid service name '%s'", name.c_str());
        return false;
    }

    std::vector<std::string> str_args(args.begin() + 2, args.end());
    service_ = std::make_unique<Service>(name, str_args);都行的化构造一个service对象 classname为default
    return true;
}

```

ParseLineSection 解析子项

```
bool ServiceParser::ParseLineSection(const std::vector<std::string>& args,
                                     const std::string& filename, int line,
                                     std::string* err) const {
    return service_ ? service_->ParseLine(args, err) : false;
}
```

解析完数据之后调用endsection 这里面调用了一个AddService 感觉像是添加了一个什么服务 看看

```
void ServiceParser::EndSection() {
    if (service_) {
        ServiceManager::GetInstance().AddService(std::move(service_));
    }
}
```

addservice

```
void ServiceManager::AddService(std::unique_ptr<Service> service) {
    Service* old_service = FindServiceByName(service->name());
    if (old_service) {
        LOG(ERROR) << "ignored duplicate definition of service '" << service->name() << "'";
        return;
    }
    services_.emplace_back(std::move(service)); 将Service对象加入Service链表中 然后根据选项域的内容填充Service对象
}
```

### init启动Zygote

讲完service 下面就是具体怎么启动zygote了

Zygote 的classname 为main  Zygote 的classname 为main

![image-20240311083757031](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403110913173.png)

class_start是一个COMMAND，对应的函数为do_class_start。注释1处启动那些classname为main的Service，而zygote的classname就是main

do_class_start 具体就是同个这个函数取拉起main 也就是zygote的

```
static int do_class_start(const std::vector<std::string>& args) {
        /* Starting a class does not start services
         * which are explicitly disabled.  They must
         * be started individually.
         */
    ServiceManager::GetInstance().
        ForEachServiceInClass(args[1], [] (Service* s) { s->StartIfNotDisabled(); }); 遍历Service链表，找到classname为main的Zygote，并执行StartIfNotDisabled函数
    return 0;
}
```

而StartIfNotDisabled

```
bool Service::StartIfNotDisabled() {
    if (!(flags_ & SVC_DISABLED)) { 这玩意是对应rc文件会设置好的 如果没有设置，就会调用Start来启动这个service
        return Start();
    } else {
        flags_ |= SVC_DISABLED_START;
    }
    return true;
}
```

start 很长一个 

```
{
    // Starting a service removes it from the disabled or reset state and
    // immediately takes it out of the restarting state if it was in there.
    flags_ &= (~(SVC_DISABLED|SVC_RESTARTING|SVC_RESET|SVC_RESTART|SVC_DISABLED_START));

    // Running processes require no additional work --- if they're in the
    // process of exiting, we've ensured that they will immediately restart
    // on exit, unless they are ONESHOT.
    看看是否已经启动了
    if (flags_ & SVC_RUNNING) {
        return false;
    }

    bool needs_console = (flags_ & SVC_CONSOLE);
    if (needs_console) {
        if (console_.empty()) {
            console_ = default_console;
        }

        // Make sure that open call succeeds to ensure a console driver is
        // properly registered for the device node
        int console_fd = open(console_.c_str(), O_RDWR | O_CLOEXEC);
        if (console_fd < 0) {
            PLOG(ERROR) << "service '" << name_ << "' couldn't open console '" << console_ << "'";
            flags_ |= SVC_DISABLED;
            return false;
        }
        close(console_fd);
    }
判断启动的service的执行文件是否存在 如果不存在就不启动了

    struct stat sb;
    if (stat(args_[0].c_str(), &sb) == -1) {
        PLOG(ERROR) << "cannot find '" << args_[0] << "', disabling '" << name_ << "'";
        flags_ |= SVC_DISABLED;
        return false;
    }

    std::string scon;
    if (!seclabel_.empty()) {
        scon = seclabel_;
    } else {
        LOG(INFO) << "computing context for service '" << name_ << "'";
        scon = ComputeContextFromExecutable(name_, args_[0]);
        if (scon == "") {
            return false;
        }
    }

    LOG(INFO) << "starting service '" << name_ << "'...";

	判断子进程有没有启动 没有启动就fork创建子进程
    pid_t pid = -1;
    if (namespace_flags_) {
        pid = clone(nullptr, nullptr, namespace_flags_ | SIGCHLD, nullptr);
    } else {
        pid = fork();
    }
	代码逻辑在子进程当中运行了
    if (pid == 0) {
        umask(077);

        if (namespace_flags_ & CLONE_NEWPID) {
            // This will fork again to run an init process inside the PID
            // namespace.
            SetUpPidNamespace(name_);
        }

        for (const auto& ei : envvars_) {
            add_environment(ei.name.c_str(), ei.value.c_str());
        }

        std::for_each(descriptors_.begin(), descriptors_.end(),
                      std::bind(&DescriptorInfo::CreateAndPublish, std::placeholders::_1, scon));

        // See if there were "writepid" instructions to write to files under /dev/cpuset/.
        auto cpuset_predicate = [](const std::string& path) {
            return android::base::StartsWith(path, "/dev/cpuset/");
        };
        auto iter = std::find_if(writepid_files_.begin(), writepid_files_.end(), cpuset_predicate);
        if (iter == writepid_files_.end()) {
            // There were no "writepid" instructions for cpusets, check if the system default
            // cpuset is specified to be used for the process.
            std::string default_cpuset = android::base::GetProperty("ro.cpuset.default", "");
            if (!default_cpuset.empty()) {
                // Make sure the cpuset name starts and ends with '/'.
                // A single '/' means the 'root' cpuset.
                if (default_cpuset.front() != '/') {
                    default_cpuset.insert(0, 1, '/');
                }
                if (default_cpuset.back() != '/') {
                    default_cpuset.push_back('/');
                }
                writepid_files_.push_back(
                    StringPrintf("/dev/cpuset%stasks", default_cpuset.c_str()));
            }
        }
        std::string pid_str = StringPrintf("%d", getpid());
        for (const auto& file : writepid_files_) {
            if (!WriteStringToFile(pid_str, file)) {
                PLOG(ERROR) << "couldn't write " << pid_str << " to " << file;
            }
        }

        if (ioprio_class_ != IoSchedClass_NONE) {
            if (android_set_ioprio(getpid(), ioprio_class_, ioprio_pri_)) {
                PLOG(ERROR) << "failed to set pid " << getpid()
                            << " ioprio=" << ioprio_class_ << "," << ioprio_pri_;
            }
        }

        if (needs_console) {
            setsid();
            OpenConsole();
        } else {
            ZapStdio();
        }

        // As requested, set our gid, supplemental gids, uid, context, and
        // priority. Aborts on failure.
        SetProcessAttributes();

        std::vector<char*> strs;
        ExpandArgs(args_, &strs);
        调用execve service就会启动了
        if (execve(strs[0], (char**) &strs[0], (char**) ENV) < 0) {
            PLOG(ERROR) << "cannot execve('" << strs[0] << "')";
        }

        _exit(127);
    }

    if (pid < 0) {
        PLOG(ERROR) << "failed to fork for '" << name_ << "'";
        pid_ = 0;
        return false;
    }

    if (oom_score_adjust_ != -1000) {
        std::string oom_str = StringPrintf("%d", oom_score_adjust_);
        std::string oom_file = StringPrintf("/proc/%d/oom_score_adj", pid);
        if (!WriteStringToFile(oom_str, oom_file)) {
            PLOG(ERROR) << "couldn't write oom_score_adj: " << strerror(errno);
        }
    }

    time_started_ = boot_clock::now();
    pid_ = pid;
    flags_ |= SVC_RUNNING;

    errno = -createProcessGroup(uid_, pid_);
    if (errno != 0) {
        PLOG(ERROR) << "createProcessGroup(" << uid_ << ", " << pid_ << ") failed for service '"
                    << name_ << "'";
    }

    if ((flags_ & SVC_EXEC) != 0) {
        LOG(INFO) << android::base::StringPrintf(
            "SVC_EXEC pid %d (uid %d gid %d+%zu context %s) started; waiting...", pid_, uid_, gid_,
            supp_gids_.size(), !seclabel_.empty() ? seclabel_.c_str() : "default");
    }

    NotifyStateChange("running");
    return true;
}
```

首先判断Service是否已经运行，如果运行则不再启动，直接返回false。如果程序走到注释1处，说明子进程还没有被启动，就调用fork函数创建子进程，并返回pid值，注释2处如果pid值为0，则说明当前代码逻辑在子进程中运行。注释3处在子进程中调用execve函数，Service子进程就会被启动，并进入该Service的main函数中，如果该Service是Zygote，从2.1.3节末段的标注①处我们可知Zygote执行程序的路径为/system/bin/app_process64，对应的文件为app_main.cpp，这样就会进入app_main.cpp的main函数中

这部分代码也很长

关键部分如下 注释1处的代码可以得知调用runtime的start函数启动Zygote，至此Zygote就启动了。

![image-20240311085033373](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403110913174.png)



至此我们梳理一下

就是init.rc里面的class_start通过调用各种函数启动的zygote

### 属性服务

这玩意类似于一个注册表，注册表的内容采用键值对的形式来记录用户、软件的一些使用信息。

init进程启动时会启动属性服务，并为其分配内存，用来存储这些属性，如果需要这些属性直接读取就可以了

就是这两行代码

![image-20240311085740590](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403110913175.png)

#### 属性服务初始化与启动

来看看第一个

```
void property_init() {
    if (__system_property_area_init()) {
        LOG(ERROR) << "Failed to initialize property area";
        exit(1);
    }
}
```

就调用了__system_property_area_init 不需要太关注 主要是初始化用

![image-20240311090246194](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403110913176.png)

接下来看看start poperty service

```
void start_property_service() {
    property_set("ro.property_service.version", "2");

    property_set_fd = create_socket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK,
                                    0666, 0, 0, NULL);创建非阻塞socket
    if (property_set_fd == -1) {
        PLOG(ERROR) << "start_property_service socket creation failed";
        exit(1);
    }

    listen(property_set_fd, 8);调用listen函数对property_set_fd进行监听，这样创建的Socket就成为server，也就是属性服务

    register_epoll_handler(property_set_fd, handle_property_set_fd);将property_set_fd放入了epoll中，用epoll来监听property_set_fd：当property_set_fd中有数据到来时，init进程将调用handle_property_set_fd函数进行处理
}
```

`epoll是Linux内核为处理大批量文件描述符而做了改进的poll，是Linux下多路复用I/O接口select/poll的增强版本`

#### 服务处理客户端请求

![image-20240311090755880](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403110913177.png)

里面关键是handle_property_set 涉及进一步封装

handle_property_set进行属性的检查

一种是普通属性；还有一种是控制属性，控制属性用来执行一些命令，比如开机的动画就使用了这种属性。



### 总结

init进程启动做了很多的工作，总的来说主要做了以下三件事：（1）创建和挂载启动所需的文件目录。（2）初始化和启动属性服务。（3）解析init.rc配置文件并启动Zygote进程。























































































































































































































































