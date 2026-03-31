## 基本格式
```
<type>(<scope>): <subject>
<body>
<footer>
```
- **type**：提交类型（必填）
- **scope**：影响的范围（选填，一般是模块、子系统名）
- **subject**：简要说明（必填，一句话，50 字以内）
- **body**：详细说明（选填，用来解释为什么要这么改，而不是“改了什么”）
- **footer**：额外信息（选填，关联 issue、是否有破坏性变更等）
## 提交类型说明
| **type** | **说明**                                |
| -------- | ------------------------------------- |
| feat     | 新功能（feature）                          |
| fix      | 修复 bug                                |
| docs     | 文档修改（比如 README、注释）                    |
| style    | 代码风格修改，不影响逻辑（格式化、空格、分号、缩进）            |
| refactor | 代码重构，不涉及新增功能或修 bug（比如提取公共函数）          |
| perf     | 性能优化（提高速度、减少内存）                       |
| test     | 测试相关（新增、修改、完善测试）                      |
| build    | 构建系统或依赖调整（webpack、npm、maven、docker 等） |
| ci       | 持续集成配置（GitHub Actions、Jenkinsfile 等）  |
| chore    | 其他不影响代码逻辑的修改（脚本、配置、依赖升级）              |
| revert   | 回滚之前的提交                               |
## subject书写规范
- 必须使用 **祈使句**，不要用过去式或完成时。
    - ✅ fix: handle null pointer in parser
    - ❌ fixed null pointer issue
- 首字母小写，不以句号结尾。
- 长度不要超过 **50 个字符**。
- 语言保持简洁明了。
## body详细规则说明
- 用来解释“为什么要改”，而不是“改了什么”。
- 如果提交内容复杂，建议分点描述。
- 每行不超过 **72 个字符**，方便在命令行工具里查看。

实例

```

fix(cache): resolve memory leak in LRU implementation
  

The LRU cache was not properly releasing old keys after exceeding

the capacity. Fixed by adding explicit delete operations.

  

This change ensures better memory usage in long-running services.
```
## footer 规则
- **关联 issue** 时使用关键词：
    - Closes #123 → 自动关闭 issue 123
    - Related to #456 → 只是相关，不会关闭
- **破坏性变更**（Breaking Changes）要单独注明：
	```
	feat(auth): migrate to OAuth 2.0
	BREAKING CHANGE: old API tokens are no longer supported.
	```
## 实例
```
功能新增加
feat(auth): add JWT authentication middleware

```

```
修bug
fix(parser): handle null pointer when input is empty
```

```
写文档
docs(readme): update installation guide
```

```
性能优化
perf(query): improve SQL query speed by adding index
```

```
配置修改
chore(deps): bump lodash from 4.17.15 to 4.17.21
```

```
回滚提交
revert: feat(auth): add JWT authentication middleware
	This reverts commit 9f8c123.
```

