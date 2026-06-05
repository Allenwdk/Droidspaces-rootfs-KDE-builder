# 添加自定义用户名选项实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 GitHub Actions workflow 中添加一个可自定义的用户名输入选项，并将该用户名通过构建脚本传递到所有 Dockerfile 中，替换当前硬编码的 `Gold` 用户名。

**Architecture:** 通过 GitHub Actions `workflow_dispatch` 的 `string` 类型输入接收用户名，经由 `build_rootfs-native.sh` 和 `build_rootfs-qemu-aarch64.sh` 作为 Docker build-arg 传递，最后在五个 Dockerfile 中使用该 build-arg 动态创建用户和配置权限。

**Tech Stack:** GitHub Actions, Bash, Docker (Dockerfile)

---

## 当前状态分析

目前代码库中所有发行版的 Dockerfile 都硬编码使用了用户名 `Gold`，具体出现在以下位置：

- **用户创建与密码设置**：`useradd -m -s /bin/bash Gold && echo "Gold:1234" | chpasswd`
- **用户目录配置**：`/home/Gold/.config/...`、`/home/Gold/.bashrc`
- **文件权限**：`chown -R Gold:Gold /home/Gold`
- **用户组添加**：`usermod -a -G ... Gold`

涉及文件：
- `.github/workflows/build-rootfs-releases.yml` — 定义 Action 输入选项
- `build_rootfs-native.sh` — 原生架构构建脚本
- `build_rootfs-qemu-aarch64.sh` — QEMU 跨架构构建脚本
- `Ubuntu-24-KDE.Dockerfile`
- `Ubuntu-25-KDE.Dockerfile`
- `Debian-13-KDE.Dockerfile`
- `Fedroa-43-KDE.Dockerfile`
- `Arch-KDE.Dockerfile`

---

## 任务分解

### Task 1: 修改 GitHub Actions Workflow 添加用户名输入

**Files:**
- Modify: `.github/workflows/build-rootfs-releases.yml`

- [ ] **Step 1: 在 workflow 输入中添加 `custom_username` 字段**

在 `workflow_dispatch.inputs` 区域添加一个新的 `string` 类型输入，放在 `build_target` 之后：

```yaml
      custom_username:
        description: '自定义用户名（默认 Gold）'
        required: false
        type: string
        default: 'Gold'
```

- [ ] **Step 2: 在 build job 中将用户名传递给构建脚本**

在 `build` job 的 `编译 RootFS` 步骤中，添加 `-u` 参数传递用户名：

```bash
          ./build_rootfs-native.sh \
            -i "${{ matrix.template }}.Dockerfile" \
            -v "${{ needs.setup.outputs.build_id }}" \
            -K "${{ inputs.build_KDE }}" \
            -P "${{ inputs.PulseAudio }}" \
            -g "${{ inputs.enable_zh_tz }}" \
            -a "${{ inputs.enable_binfmt }}" \
            -b "${{ inputs.enable_yj }}" \
            -c "${{ inputs.enable_mesa }}" \
            -d "${{ inputs.enable_kfgj }}" \
            -e "${{ inputs.enable_zip }}" \
            -f "${{ inputs.enable_docker }}" \
            -h "${{ inputs.enable_srf }}" \
            -j "${{ inputs.enable_tmoe }}" \
            -u "${{ inputs.custom_username }}"
```

- [ ] **Step 3: 在 Release body 中显示用户名选项**

在 `创建统一 Release` 步骤的 `body` 中添加一行：

```markdown
            - **自定义用户名**: ${{ github.event.inputs.custom_username }}
```

---

### Task 2: 修改原生构建脚本 `build_rootfs-native.sh`

**Files:**
- Modify: `build_rootfs-native.sh`

- [ ] **Step 1: 添加 `-u` 参数解析**

在 `while getopts` 中添加 `u:` 选项：

```bash
while getopts "i:v:K:P:a:b:c:d:e:f:g:h:j:u:" opt; do
```

在 `case` 中添加：

```bash
    u) USERNAME="$OPTARG" ;; # 自定义用户名
```

- [ ] **Step 2: 设置默认用户名并传递 build-arg**

在参数解析后添加默认值：

```bash
: "${USERNAME:=Gold}"
```

在 `docker buildx build` 命令中添加：

```bash
  --build-arg USERNAME="$USERNAME" \
```

---

### Task 3: 修改 QEMU 跨架构构建脚本 `build_rootfs-qemu-aarch64.sh`

**Files:**
- Modify: `build_rootfs-qemu-aarch64.sh`

- [ ] **Step 1: 添加 `-u` 参数解析**

与 Task 2 相同，在 `while getopts` 中添加 `u:`，在 `case` 中处理 `u) USERNAME="$OPTARG" ;;`。

- [ ] **Step 2: 设置默认用户名并传递 build-arg**

与 Task 2 相同，设置默认值为 `Gold`，并在 `docker buildx build` 中添加 `--build-arg USERNAME="$USERNAME"`。

---

### Task 4: 修改 `Ubuntu-24-KDE.Dockerfile`

**Files:**
- Modify: `Ubuntu-24-KDE.Dockerfile`

- [ ] **Step 1: 在 ARG 区域添加 `USERNAME`**

在 `ARG ENABLE_tmoe_ARG` 之后添加：

```dockerfile
ARG USERNAME
```

- [ ] **Step 2: 将所有硬编码的 `Gold` 替换为环境变量引用**

需要修改的位置：

1. 用户创建（第 118 行附近）：
```dockerfile
    useradd -m -s /bin/bash ${USERNAME} && echo "${USERNAME}:1234" | chpasswd && \
```

2. 输入法自启动目录（第 143 行附近）：
```dockerfile
    mkdir -p /home/${USERNAME}/.config/autostart
    cat <<'EOF' > /home/${USERNAME}/.config/autostart/fcitx5.desktop
```

3. `.bashrc` 路径（第 158 行附近）：
```dockerfile
    echo 'export XDG_RUNTIME_DIR=/run/user/$(id -u)' >> /home/${USERNAME}/.bashrc
```

4. KDE 配置目录（第 160 行附近）：
```dockerfile
    mkdir -p /home/${USERNAME}/.config 
    cat <<'EOF' > /home/${USERNAME}/.config/kwinrc
```

5. 权限设置（第 166 行附近）：
```dockerfile
    chown -R ${USERNAME}:${USERNAME} /home/${USERNAME}
```

6. 用户组添加（第 211 行附近）：
```dockerfile
usermod -a -G aid_inet,aid_net_raw,input,video,tty,sudo,droidspaces-gpu ${USERNAME} || true
```

---

### Task 5: 修改 `Ubuntu-25-KDE.Dockerfile`

**Files:**
- Modify: `Ubuntu-25-KDE.Dockerfile`

- [ ] **Step 1-2: 执行与 Task 4 完全相同的修改**

所有 `Gold` 出现的位置与 Ubuntu-24 类似，替换为 `${USERNAME}`。

注意第 123 行用户创建：
```dockerfile
    useradd -m -s /bin/bash -G shadow ${USERNAME} && echo "${USERNAME}:1234" | chpasswd && \
```

---

### Task 6: 修改 `Debian-13-KDE.Dockerfile`

**Files:**
- Modify: `Debian-13-KDE.Dockerfile`

- [ ] **Step 1-2: 执行与 Task 4 完全相同的修改**

所有 `Gold` 替换为 `${USERNAME}`，包括：
- 第 119 行：`useradd -m -s /bin/bash ${USERNAME} && echo "${USERNAME}:1234" | chpasswd`
- 第 143、158、160、166 行的 `/home/Gold` 路径
- 第 211 行的 `usermod -a -G ... ${USERNAME}`

---

### Task 7: 修改 `Fedroa-43-KDE.Dockerfile`

**Files:**
- Modify: `Fedroa-43-KDE.Dockerfile`

- [ ] **Step 1-2: 执行与 Task 4 完全相同的修改**

所有 `Gold` 替换为 `${USERNAME}`，包括：
- 第 102 行：`useradd -m -s /bin/bash ${USERNAME} && echo "${USERNAME}:1234" | chpasswd`
- 第 126、141、143、149 行的 `/home/Gold` 路径
- 第 193 行的 `usermod -a -G ... ${USERNAME}`

---

### Task 8: 修改 `Arch-KDE.Dockerfile`

**Files:**
- Modify: `Arch-KDE.Dockerfile`

- [ ] **Step 1-2: 执行与 Task 4 完全相同的修改**

所有 `Gold` 替换为 `${USERNAME}`，包括：
- 第 105 行：`useradd -m -s /bin/bash ${USERNAME} && echo "${USERNAME}:1234" | chpasswd`
- 第 133、148、150、156 行的 `/home/Gold` 路径
- 第 210 行的 `usermod -a -G ... ${USERNAME}`

---

### Task 9: 验证修改完整性

**Files:**
- All modified files

- [ ] **Step 1: 检查是否还有遗漏的硬编码 `Gold`**

运行以下命令检查所有 Dockerfile 和脚本中是否还有硬编码的 `Gold`：

```bash
grep -rn "Gold" *.Dockerfile build_rootfs-*.sh .github/workflows/*.yml
```

预期输出：应该只有默认值设置 `: "${USERNAME:=Gold}"` 和 workflow 的 `default: 'Gold'`。

- [ ] **Step 2: 检查 Dockerfile 中 `USERNAME` 的 ARG 声明**

```bash
grep -n "ARG USERNAME" *.Dockerfile
```

预期：5 个 Dockerfile 都应该有这一行。

- [ ] **Step 3: 检查 build-arg 传递**

```bash
grep -n "USERNAME" build_rootfs-*.sh
```

预期：两个脚本都应该有 `--build-arg USERNAME=...`。

---

## 假设与决策

1. **默认用户名保持 `Gold`**：用户明确选择保持现有默认用户名。
2. **密码保持默认 `1234`**：用户明确选择不添加自定义密码选项。
3. **使用 Docker `ARG` + shell 变量扩展**：在 Dockerfile 中通过 `ARG USERNAME` 接收，然后在 `RUN` 指令中使用 `${USERNAME}` 引用。这是 Docker 的标准做法。
4. **不修改 `build_rootfs-qemu-aarch64.sh` 的其他参数**：该脚本目前只接收 `-i` 和 `-v`，但我们也需要把其他 build-arg 传递过去（虽然当前脚本没有传递它们，但用户名是新增需求）。为了保持最小改动，只添加 `USERNAME` 的传递。
5. **用户名不包含特殊字符**：假设用户输入的用户名是合法的 Linux 用户名（只包含字母、数字、下划线、连字符，且不以数字开头）。不添加额外的验证逻辑。

---

## 验证步骤

1. **本地验证 Dockerfile 语法**：
   ```bash
   docker build -f Ubuntu-24-KDE.Dockerfile --build-arg USERNAME=testuser .
   ```
   （如果本地有 Docker 环境）

2. **检查 GitHub Actions workflow 语法**：
   在 GitHub 仓库的 Actions 页面测试 `workflow_dispatch` 是否能正确显示新的 `custom_username` 输入框。

3. **检查构建日志**：
   确认 `docker buildx build` 命令中 `--build-arg USERNAME=...` 被正确传递。

4. **检查生成的 RootFS**：
   解压构建产物，检查 `/etc/passwd` 中是否存在用户指定的用户名，以及 `/home/` 下是否有对应的用户目录。
