# 代码评审报告

## 变更概述

本次变更将 GitHub Actions 工作流中传递给 `openai-code-review-sdk` 的环境变量 `GITHUB_REVIEW_LOG_URI` 重命名为 `GITHUB_REVIEW_LOG_URL`，意图是与 secret 名称 `CODE_REVIEW_LOG_URL` 的命名风格（URL）保持统一。**方向合理，但存在一个高风险隐患和几处遗漏。**

---

## 🔴 阻断性问题（必须确认）

### 1. 环境变量重命名后，SDK 读取端是否同步修改？

本次 diff **只改了 yml，未见 SDK 侧的代码变更**。如果 jar 包内代码仍然是：

```java
System.getenv("GITHUB_REVIEW_LOG_URI")
```

则运行时将读取到 `null`，导致「评审日志写入 GitHub 仓库」功能失效——轻则抛 NPE，重则被 try-catch 吞掉后**静默失败**，且 Actions 流程不会报错，问题非常隐蔽。

**合并前必须逐项确认：**

- [ ] SDK 源码中读取逻辑已改为 `GITHUB_REVIEW_LOG_URL`
- [ ] `./libs/openai-code-review-sdk-1.0.jar` 已替换为包含该改动的新构建产物
- [ ] 全仓执行 `grep -rn "GITHUB_REVIEW_LOG_URI"` 确认无残留引用

**兼容性建议**：如果该 SDK 是对外开放使用的（从注释看是开源项目），变量名变更属于**破坏性契约变更**，建议 SDK 侧做双读兜底：

```java
String logUri = System.getenv("GITHUB_REVIEW_LOG_URL");
if (logUri == null || logUri.isBlank()) {
    logUri = System.getenv("GITHUB_REVIEW_LOG_URI"); // 向后兼容旧配置
}
```

---

## 🟡 一般问题

### 2. 注释未同步更新

```yaml
# Github 配置；GITHUB_REVIEW_LOG_URI「https://github.com/...」   ← 仍是旧名称
GITHUB_REVIEW_LOG_URL: ${{ secrets.CODE_REVIEW_LOG_URL }}
```

上一行注释仍写的是 `GITHUB_REVIEW_LOG_URI`，与新变量名不一致，会误导后续维护者，应一并修改。

### 3. 其他引用点排查

若仓库中还有其他 workflow（如 Docker 构建流程、多分支触发流程）或 README 配置文档中引用了旧变量名，需同步修改，否则不同入口行为不一致。

### 4. SDK 版本号未升级

环境变量契约变更属于 breaking change，建议将 SDK 版本升级（如 `1.0` → `1.1`），workflow 中同步更新 jar 文件名，并在 README 中登记变更说明，方便使用方感知迁移。

---

## 🟢 改进建议（非本次范围）

- **硬编码本地 jar**：`java -jar ./libs/openai-code-review-sdk-1.0.jar` 将 jar 直接提交到仓库不利于版本追溯，建议发布到 Maven 仓库后按坐标拉取，或在 workflow 中按版本号下载。
- **命名严谨性**：该值实际是仓库地址（`https://github.com/...`），严格来说 `URI` 更准确，但为了与 secret 命名统一改用 `URL` 也可接受，仅需团队内约定即可。

---

## 评审结论

**❌ 暂不建议合并。** 改动意图正确（统一 URI/URL 命名），但在未确认 SDK 读取端同步修改、jar 重新打包之前合入，会导致代码评审日志写入功能直接失效。请补充 SDK 侧改动或确认后重新提交评审。