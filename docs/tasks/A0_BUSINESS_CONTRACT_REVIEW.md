# A0：业务权限与公共契约评审提案

任务：[Issue #2](https://github.com/paher-din/XunJie/issues/2)。主责：A。关联：[G0 #1](https://github.com/paher-din/XunJie/issues/1)、[B0 #3](https://github.com/paher-din/XunJie/issues/3)、[C0 #4](https://github.com/paher-din/XunJie/issues/4)。
日期：2026-10-08（Asia/Shanghai）。基线提交：8a0e3325eaa456e903bd31532ce2a5eef8d8736a。
状态：**A 的评审稿已准备，待 B/C 会审与负责人决定；A0 未通过，G0 未冻结，无正式应用实现。**

## 1. 依据、写入边界与提案性质

已读 [AGENTS](../../AGENTS.md)、[README](../../README.md)、[PRD](../product/PRD.md) 第 5/8/9 节、[MVP_SPEC](../product/MVP_SPEC.md) 第 2～8 节、[TECH_DESIGN](../product/TECH_DESIGN.md) 第 2～10/12 节、[TEAM_WORK_PLAN](../planning/TEAM_WORK_PLAN.md) A0 与第 4.1 节，以及两份 [研究](../reference/PRODUCT_RESEARCH.md)/[工程参考](../reference/ENGINEERING_HANDOFF.md)。任务依据为 PRD-02/06/08/09/10，M-01/02/05/06/07/08/09/10，重点 AC-02/06/08/15 与 NFR-04；不改变原 M/AC/NFR 和 G0～G4 条件。

Issue 发布正文要求直接补充 TECH_DESIGN 和 MVP_SPEC 第 10 节；最新 AGENTS 的“授权与冲突处理”明确优先约束历史 Issue 修改要求。因此本次可写文件只有本文与 README 任务导航；产品、规划、规范与参考正文均只读。下文是供汇总的待评审提案，**不成为正式契约或编码授权**；文档批准与基线写入授权需要分别记录。与 B/C 的字段或行为差异收敛后，由获授权维护者合入基线；不平行维护第二份已批准规格。

原工作区的 A 方案和技术草案保持原状，本次不提交这些旧改动。当前工作区从上述最新远程提交独立建立，无旧实验源码、个人目录或本地服务依赖。本文不配置凭据、CI、数据库、执行环境或替代 UI。

| 交付项 | 基线位置 | 本提案位置 | 当前状态 |
| --- | --- | --- | --- |
| 角色与资源授权 | TECH 第 2/5/9 节、M-10 | 第 2/3/7 节 | 已起草，待会审 |
| 四类对象状态、暂停优先级 | TECH 第 4/7 节、MVP 第 4 节 | 第 4 节 | 已起草；差异登记 A0-R01～03 |
| 逐命令身份/版本/幂等/恢复 | TECH 第 5 节 | 第 5/6 节 | 24 项提案；新增接口待确认 |
| 选型/目录/逻辑结构/短事务 | TECH 第 2/3/4/10 节 | 第 2.1/8/9 节 | 推荐方案，未冻结/未建表 |
| D-03 账号/访问/数据/资源 | PRD D-03、TECH 第 9/10 节 | 第 10 节 | 已起草；未与 C 核对 |
| B 候选与 C 作业/失效接入 | TECH 第 6/7 节 | 第 8 节 | 保存归属与事务提案，实际联调未执行 |
| B/C 会审、差异与决定 | TEAM 第 4.1 节 | 第 12 节 | 五处交界待评审；不得勾选完成 |

## 2. 公共身份、范围与版本提案

字段命名与含义由 A 汇总；各领域仍由其负责人实现。浏览器输入是待验证的请求，不是可信身份或记录来源。表中 revision 为非负安全整数，创建时的预期版本 `0` 只表示尚无该对象；已经创建的版本从 `1` 开始。只有从未建立学习状态时允许读取 learnerStateRevision=0；已有状态即使有效主张全为空也保留当前 revision，不重置为 0。自述/反馈尚未创建时读取 null，首次写入的 expectedContextRevision/expectedFeedbackRevision=0。

| 字段/对象 | 含义与核验 | 维护责任 |
| --- | --- | --- |
| userId / courseId / studentId / attemptId | 公共 ID 为不含路径语义的非空字符串；资源关联以服务端记录为准。studentId 指课程中学生账号，不作为真实操作者认证 | A 定义范围；各领域保存关联 |
| ActorContext | 从有效 Session 和课程成员产生的 userId、courseId、课程角色；内部调用额外携带已授权用途/资源范围。客户端或模型不能构造/扩大此上下文 | A；B/C 只能消费 |
| resourceVersionId / activityVersionId | 不可变内容版本 ID；资料段落、量规、政策、规则及运行配置通过活动绑定读取，不解析“最新同名资源”代替旧引用 | A；C 提供运行/检查版本 |
| revision / expectedRevision | 当前蓝图版本与本次修改/检查/确认所预期版本；成功编辑增加一次，拒绝的命令不增加 | A/design |
| assignmentRevision / expectedAssignmentRevision | 分配控制版本；学生集合与活动版本固定，暂停/恢复通过此版本并发检查 | A/design |
| attemptRevision / expectedAttemptRevision | 尝试生命周期、路线/返回位置、个人控制及修订许可的并发版本；文件逐次编辑不增加它 | C/workspace；A 通过事务接口调用 |
| contextRevision / expectedContextRevision | 学生在该课程的目标/约束自述版本；与能力推断、作品版本独立；首次创建 expectedContextRevision=0 | A/review |
| learnerStateRevision / expectedStateRevision | `(courseId, studentId)` 下整个有效学习状态的版本；从未建立状态时读 `0`，不补造历史记录；清空/撤回主张仍保留已有状态版本。候选接纳、异议引起的投影变化和教师决定使用同一 CAS | A/review |
| claimId + claimRevision | 一条主张及其确切修订；主张同时具有课程/学生、目标版本、适用任务范围、证据组和来源。异议/决定引用确切版本，不只引用一句话 | A/review；B 提交候选 |
| feedbackId + feedbackRevision | 教师反馈的追加版本；首次保存用 expectedFeedbackRevision=0，修订引用当前版本；不是能力状态 revision | A/review |
| decisionEpoch | 某 Attempt 的教学决策失效代数；纠正、任务暂停、主动求助替换等按本提案第 8.3 节递增。不等于文件版本或消息展示序号 | C 提供同事务操作；A/B 按原因调用 |
| snapshotId / fileId / documentVersion / contentHash | 确认作品与文件实例的版本；继续使用 TECH_DESIGN 第 4.1 节。求助/运行/提交不能以当前路径或本机未同步内容替代 | C；A/B 核验引用 |
| jobId / commandId / requestId / eventCursor | 持久作业、已接纳业务命令、单次 HTTP 请求追踪与事件补取游标分别标识；重试有新 requestId，但同命令结果/副作用不重做；游标不授予读取权 | C 提供 Job/CommandReceipt/事件；A 生成 requestId |

scope 至少能定位 courseId、studentId、competencyId 及其目标版本，并注明只适用于哪次活动/尝试还是同课程相关目标。新任务读取先按范围过滤，再看 validity/帮助条件；不能仅因 competencyId 相同就认定可迁移。B 的上下文读取必须返回已过滤的主张和反证，不返回另一学生的状态缓存。

### 2.1 A 领域逻辑结构提案

这是建表授权前的逻辑设计，**没有新增数据库、DDL 或迁移**。对象可以按 TECH_DESIGN 第 4 节使用版本化 JSON；实际表、索引和驱动选型在 G0 及数据库操作评审后落实。

| 逻辑对象 | A0 需要保存的关联 | 必须可验证的约束 |
| --- | --- | --- |
| CourseMembership / Session | userId、courseId、角色、成员/会话有效状态 | 教师权限按课程；维护者身份不隐含教师权；新分配对象必须是有效学生成员 |
| ResourceVersion | courseId、材料来源、段落/正文/hash、可见范围、前版本引用 | 内容不可变；材料更新创建新版本；当前活动保留原引用；检索/错误/导出不绕过可见性 |
| BlueprintDraft / Proposal | courseId、蓝图 revision、建议 jobId/baseRevision、差异/引用/影响项 | 提案不直接覆盖草稿；源材料必须获准；人工采用再校验基础版本及所有影响关联 |
| BlueprintCheck / RubricTrial | 蓝图 revision、三类检查、教师疑点处理、样例/试评/歧义引用 | 检查结果不能跨 revision 复用；试评样例只支持设计审阅，不成为学生能力观察 |
| ActivityVersion | 蓝图及其 revision、目标/量规/资源/政策/检查/运行版本与就绪依据 | 确认事务共同核验引用并冻结；不通过原地改内容更新要求 |
| Assignment | courseId、activityVersionId、指定 studentIds、控制状态/revision、教师理由 | 分配集合及活动版本固定；改变范围另建分配；暂停是覆盖层，不篡改旧尝试生命周期 |
| MyContextRevision | courseId、studentId、自述内容、contextRevision 与来源 | 只有本人修改；教师/模型可引用，不把自述改写成观察或正式判断 |
| LearnerStateRevision / Claim | 课程/学生、前 revision、确切主张版本、来源/范围、证据组、支持/反证/帮助/未知 | 同范围只有一个有效头版本；追加历史；撤回/争议不抹去事实；不输出累积掌握概率 |
| Dispute / TeacherDecision | 本人或教师身份、targetRef、理由/引用、所见状态版本、处理关联 | 原异议和决定保留；对历史目标的申诉不直接覆盖当前不同主张；复核不自动复活撤回依据 |
| FeedbackRevision | kind、attemptId、快照或 submissionId、状态依据、教师身份、feedbackRevision | formative 与 submission_review 区分；模型草稿不成为正式反馈；正文与三条能力线/帮助/未知分开 |
| ReopenGrant | attemptId、最新 submissionId、许可教师/理由、许可版本与使用记录 | 教师只许可；学生主动继续才改变学习阶段；许可不能覆盖旧提交或迁入新活动要求 |

ReopenGrant 的教学许可由 A 负责，许可保存/消费与尝试生命周期通过 C 的事务接口完成；A 不直接改 Attempt，C 不改教师理由或教学决定。正式反馈引用后来已撤回/争议的主张时，读取投影标注依据状态，保留原正文/当时版本；旧反馈不能成为恢复已撤回主张的捷径。

所有证据引用同时核验存在、版本、所属课程/学生、用途和当前可用性。引用已撤回/脱敏或本人无权读取时，返回明确的缺口/受限说明；不把 hash、模型摘要或教师赞同补成原始证据。Student 可见状态/反馈也不能借引用反查私有答案。

有效投影保留 self_report/model_estimate/teacher_judgment 等来源区别。主张得到 supported 状态必须有可用原始引用与明确帮助条件；教师无依据赞同只能改变判断/处理状态，不能凭空补观察。教师撤回约束按目标、适用范围及被否定依据保留，B 不能靠换 claimId、措辞或 estimatorVersion 用同一依据恢复该主张；新的证据也不改写旧撤回事实，需按当前版本重新评估。

## 3. 角色与资源授权矩阵

T 表示该课程有效任课教师，S 表示本人且为有效课程学生成员，O 表示获准维护操作。role 与资源归属均由服务端获得。Teacher 不能凭另一课程的教师身份获权；Student 不能凭共享 URL 访问另一学生记录。B/C 的后台调用是受控用途，不新增一个能够自由扮演用户的公开角色。

| 资源 | T | S | O | B/C 受控使用 |
| --- | --- | --- | --- | --- |
| 账号/Session/课程成员 | 登录本人；读取负责课程的必要成员信息 | 登录本人；读取本人课程关系 | 按批准方式预置账号/邀请；不获得教学判断权 | A 提供可信身份/范围，不能从模型参数生成 |
| 课程材料版本 | 创建/读取/选择版本与可见性 | 读取本人获分配活动中可见版本 | 不因维护身份自动读取教学正文 | 教师生成仅选定资料；学生辅导只取 tutor 投影；分析不得借工具获取私有答案 |
| 私有答案/验证资产 | 负责课程范围内的设计/审阅 | 不可读 | 只按批准配置管理，不成为辅导读取入口 | 可信验证器只取指定规则所需资产；不进入学生进程/普通辅导上下文 |
| 蓝图/提案/检查/量规试评 | 负责课程内创建、编辑、预览、采用/确认 | 不读草稿或未采用提案 | 不编辑/确认 | B 仅提交绑定请求和基础版本的建议；C 保存作业与事件 |
| 活动/分配 | 创建固定版本、分配、暂停/恢复 | 读取分配给本人的固定活动，不修改要求 | 不代教师分配 | C 据分配建尝试；B 读本次活动获准投影 |
| 尝试/文件/快照/运行/提交 | 审阅负责课程实际引用；准备样例另走教师用途 | 本人发起编辑/保存/运行/提交与恢复；固定提交不能覆写 | 按批准恢复/备份，不代学生操作学习阶段 | C 执行本人命令；B 只读获准确认对象，无写作品/运行工具 |
| 原始消息/观察/覆盖/回执 | 负责课程的审阅范围 | 本人可见内容及覆盖说明 | 授权运维最少元数据；正文访问另按 D-03 | C 保存真实来源；B 不能补造事实；展示回执不是阅读/理解证明 |
| 自述 | 在审阅上下文读取有关自述，不替学生修改 | 修改自己的课程目标/约束 | 不修改 | B 可引用且注明 self_report；不变成能力证据 |
| 主张/有效状态/异议 | 查看原始依据，确认/纠正/暂缓，处理异议 | 查看本人状态，提出/查看异议；不能写正式判断 | 不作教学决定 | B 提交候选；A 校验/投影；C 提供原始依据和失效能力 |
| 反馈/修订许可 | 保存正式反馈、给出/修订许可 | 查看本人正式反馈；获准后主动继续 | 不代教师反馈、不代学生继续 | B 只起草；A 保存决定；C 负责生命周期与旧提交 |
| Job/事件/导出 | 本人发起 Job；课程审阅/导出按授权投影 | 本人 Job、尝试事件与个人可见记录 | 获准健康/备份/恢复，不开放任意课程导出 | C 实现记录/裁剪；A/B 不另建队列或回执 |

教师预览 student/tutor/teacher-validator 三种载荷时仍以 T 鉴权。Student 或 B 的 student_help 调用即使填写 audience=teacher，也不能取得第三种载荷。授权过滤发生在检索、序列化和事件下发前；不先传全包再隐藏字段。

## 4. 业务状态与暂停优先级

### 4.1 蓝图、活动与分配

“草稿/已确认”是对象性质；阻断/设计疑点/待验证是检查结果，不混成同一生命周期枚举。检查和试评只认所绑定的 revision，编辑后不能拿旧检查冒充新版本通过。

| 对象/当前状态 | 允许命令 | 转换/版本 | 拒绝与保留 |
| --- | --- | --- | --- |
| 尚无蓝图 | CMD-04 manual/agent/copy | manual/copy 创建 revision=1；agent 创建 Job/Proposal，教师选中后创建草稿 | 不要求完整机器配置；生成失败不产生正式活动 |
| BlueprintDraft 草稿 | CMD-05/06/07/08/24，READ-04/05/06 | 编辑增加 revision；检查/试评绑定当前 revision；release 创建独立活动 | 旧提案/CAS/引用不符拒绝；保留人工内容 |
| ActivityVersion 已确认 | CMD-09，CMD-04 copy，READ-07 | 确认内容与资源/政策/量规/运行配置固定；复制创建新蓝图 | 无原地 PATCH；不能替换已开始尝试要求 |
| 无 Assignment | CMD-09 | 固定活动和 studentIds；创建 active 控制 revision=1 | 无有效学生成员或环境未就绪拒绝 |
| Assignment active | CMD-10 pause、CMD-11，所属尝试的学生学习命令 | pause → paused，控制 revision 增加，相关 epoch/作业同事务处理 | 分配不自动开始学生操作 |
| Assignment paused | CMD-10 resume、合法读取/已有草稿保存、教师复核/反馈 | resume → active；不改 Attempt 本人 paused；不重投旧动作 | 学生 start/help/run/submit/resume/resume_revision 拒绝 |
| 分配学生集合或活动要改变 | 新 CMD-09 | 新分配；旧活动/尝试不变 | 不改旧分配的范围或资源引用 |

MVP 第 4 节“已确认活动版本暂停使用”与 TECH 的分配暂停路径需统一：本稿建议通过相关分配暂停覆盖已分配尝试；未分配活动是否需要独立“禁用分配”控制，记录为 A0-R03，不增加未经确认的命令。停用账号/成员只撤销权限，不伪造学生主动暂停，不删除尝试历史。

### 4.2 尝试操作与阶段转换提案

此表细化 MVP 第 4 节已有行为，待 A/B/C 会审、负责人批准并由授权维护者同步基线后作为实现依据。ready 是已创建但尚无学生开始行动；读取/恢复展示不激活它。学生可直接编辑、求助或运行，第一条获准显式学习命令在其事务中开始，不增加必须点击的开始步骤。Attempt 状态由 C 维护；A 的反馈/许可通过 C 的事务接口衔接。

| 操作 | 之前 → 之后 | 发起者/条件 | 保留与拒绝规则 |
| --- | --- | --- | --- |
| 建立当前尝试 | 无 → ready | S 对 active 分配显式进入；CMD-11 | `(assignmentId, studentId)` 只有一个当前尝试；重试返回原对象 |
| 开始或首次有效学习命令 | ready → active | S 的 start，或获准同步/求助/运行/提交；分配 active | 无强制诊断/计划；首次提交可在同事务中再转 submitted；GET 无此副作用 |
| 暂缓个人尝试 | active → paused | S 的 pause | 取消/归档进行中的相关动作；已有作品保存可用；不抹掉结果 |
| 恢复个人尝试 | paused → active | S 的 resume，分配 active | 教师暂停时拒绝；不重投旧提示或悄悄重跑 |
| 固定提交 | active → submitted | S 的 CMD-17，确认快照与课程要求引用成立 | 固定旧快照；课程检查结果与作品评价分别表达；不要求作品一定答对才可完成流程 |
| 学习主张复核 | 生命周期不变 | T 的 CMD-21 | 更新状态/决定和依赖失效；不把“看过一条主张”当作整份提交已审阅 |
| 保存形成性反馈 | 生命周期不变 | T 的 CMD-23 kind=formative | 绑定明确对象/范围；不会自动提交、重开或改量规 |
| 保存提交审阅反馈 | submitted → reviewed；reviewed 保持 | T 的 CMD-23 kind=submission_review，绑定最新固定提交 | 正式反馈/阶段同事务；旧提交、旧反馈和帮助条件不回写 |
| 允许个人修订 | reviewed → reviewed | T 的 CMD-22，绑定最新提交与当前尝试版本 | 保存许可，不代学生主动继续 |
| 学生继续修订 | reviewed → active | S 的 resume_revision，许可仍有效、分配 active | 使用同一固定活动，消费该次许可；保留所有旧提交，新提交另有 ID/版本 |
| 进入相关后续任务 | 原尝试不变；新尝试 ready | T 确认/分配新活动后，S 显式进入 | 新活动/尝试独立；同课程适用有效状态可读取，旧错误不恢复 |
| 暂停/恢复分配 | 各 Attempt 生命周期不变 | T 的 CMD-10 | Assignment 是优先覆盖层；暂停阻止新开始/帮助/运行/提交；恢复不替学生解除个人 paused |
| 暂停采集或关闭提醒 | 生命周期不变 | S 的独立控制 | 功能数据/主动求助继续；按本提案第 8.3 节仅取消相应类别 |

在 ready/active/paused 生命周期内，paused 或分配暂停时 sync 仍可保存已有文件草稿；不能因此恢复 active、发新帮助/运行或提交。submitted/reviewed 不因为分配暂停就获得作品写权。新建/回收文件的暂停处理由 C0 按“查看和保存现有草稿”边界核对，不用 UI 按钮状态代替服务端校验。

设值类控制在版本检查通过后若已经是目标值，返回当前版本/状态，不额外增加 epoch 或补造覆盖区间；同键重放仍按公共回执返回首次结果。改变其他控制不能顺带修改采集、提醒或个人暂停。

### 4.3 控制判断顺序

每次命令先核验当前会话、课程和资源，之后取同一事务的分配与个人状态。教师分配暂停覆盖个人恢复；个人 paused 阻止新帮助/运行/提交，仍可保存现有草稿；采集暂停只停止逐次过程和被动分析；提醒关闭只停止主动提醒。submitted/reviewed 的固定提交不被任何开关解锁。没有“恢复所有控制”的快捷语义。

差异：TECH 第 7.1 节列出 closed，而 MVP 没有关闭状态或命令；本稿不给 closed 发明操作或产品出口，待 A0-R02 裁决。TECH 第 6.1 节只允许 active 求助，而 MVP 第 4 节允许 ready 求助；本稿按 MVP 提议首次有效命令同事务激活，待 A0-R01 会审确认。

## 5. 逐命令业务契约提案

除 CMD-01/02 认证例外，命令都使用本提案第 6 节公共幂等规则。表中“版本”字段为必需的并发/引用检查；纯创建不需要编造已有 revision。assignment active 是个人开始/求助/运行/提交的共同前提，暂停后的保存例外见 CMD-12。T/S 的资源范围按本提案第 3 节逐次核验。

| 编号/接口 | 发起/归属 | 允许状态 | 输入版本与关键内容 | 结果/责任 |
| --- | --- | --- | --- | --- |
| CMD-01 POST /api/sessions | 本人账号 | 账号可登录 | 凭据；不接受 role/studentId 赋权，无业务 revision | Session 与可信本人信息；A |
| CMD-02 POST /api/logout | 本人 Session | 有效或已退出 | 撤销当前会话；不记录凭据摘要 | 重复退出安全结束；A |
| CMD-03 POST /api/courses/:id/resources | T/课程 | 课程允许设计 | TXT/Markdown/粘贴、可见性；更新注明前 resourceVersionId 并校验归属 | 新不可变 ResourceVersion；A |
| CMD-04 POST /api/courses/:id/blueprints | T/课程 | 课程允许设计 | mode=manual/agent/copy；manual 可为少量信息，也可带 sourceProposalId/selectedCandidateId 采用获准根提案；copy 校验源 activityVersionId 或源蓝图 revision；agent 记录获准输入 | manual/copy 返回新草稿；agent 返回 jobId 与候选/草案提案引用，人工选择后再创建草稿；A+B+C |
| CMD-05 PATCH /api/blueprints/:id | T/蓝图 | 草稿 | expectedRevision、局部变化；采用建议另带 proposalId/baseRevision，均须匹配当前草稿 | 新 revision/关联影响；不自动变基旧提案；A |
| CMD-06 POST /api/blueprints/:id/proposals | T/蓝图 | 草稿 | expectedRevision、局部请求/获准引用 | jobId，完成后为提案，不写草稿；A+B+C |
| CMD-07 POST /api/blueprints/:id/checks | T/蓝图 | 草稿 | expectedRevision、设计疑点处理说明 | 绑定该 revision 的阻断/疑点/未验证结果；A |
| CMD-08 POST /api/blueprints/:id/releases | T/蓝图 | 草稿且无阻断 | expectedRevision、相同版本检查与疑点说明；服务端核验关联版本/C1 就绪 | 不可变 ActivityVersion；原草稿仍可编辑出下一版本；A+C |
| CMD-09 POST /api/activities/:id/assignments | T/活动课程 | 已确认活动 | 指定 studentIds，均为课程有效成员；核验固定活动和运行配置 | Assignment 与控制版本；不自动启动尝试；A |
| CMD-10 POST /api/assignments/:id/controls | T/分配 | 已分配 | expectedAssignmentRevision、pause/resume、理由；事务内覆盖相关尝试 | 新分配控制版本、epoch/失效结果；不重新投递旧动作；A+C |
| CMD-11 POST /api/assignments/:id/attempts | S/获分配任务 | 分配 active | 固定 assignmentId；本人 studentId 来自身份；校验唯一当前尝试 | 创建 ready 或返回已有尝试，不能以重试绕过旧提交/政策；C+A |
| CMD-12 POST /api/attempts/:id/sync | S/本人尝试 | ready/active/paused；分配暂停时仍可保存已有作品 | fileId/baseVersion/clientId/clientSeq 与内容变化；不接受任意路径执行 | 落盘 ACK/冲突；ready 首次学习写入仅在分配 active 时转 active，暂停时不恢复；C |
| CMD-13 POST /api/attempts/:id/help | S/本人尝试 | ready/active 且分配 active、政策允许 | expectedAttemptRevision、ObjectRef/问题；核验确认对象并读取当前状态/epoch | ready 同事务激活后接纳，返回 jobId；B+C+A |
| CMD-14 POST /api/attempts/:id/runs | S/本人尝试 | ready/active 且分配 active | expectedAttemptRevision、snapshotId、批准 profile/输入、run 或 course_check；限额与检查政策 | 固定 runId/jobId；ready 同事务激活；C |
| CMD-15 POST /api/attempts/:id/controls | S/本人尝试 | 按第 7.1.1/7.5 节操作分别检查 | expectedAttemptRevision；start/pause/resume/resume_revision 或采集/提醒设值；重开带有效 grant 引用 | 新尝试/控制版本；三个开关独立，不默默恢复；C |
| CMD-16 POST /api/jobs/:id/cancel | 有权发起该 Job 的本人 | 排队/进行中；已终结则返回既有状态 | jobId、所属命令/范围；不授予教师任意代学生发起操作权 | 取消请求/既有结果；不抹去事实；C，B 配合 |
| CMD-17 POST /api/attempts/:id/submissions | S/本人尝试 | ready/active 且分配 active | expectedAttemptRevision、确认 snapshotId、课程要求的说明/检查引用 | 固定 Submission、submitted；不把 stdout/smoke 当通过；C |
| CMD-18 POST /api/actions/:id/receipts | 已认证的目标 S 客户端 | 事实发生过即可追加，迟到事实仍保留 | actionId/contentHash/receiptKind/clientReceiptId，核验目标/内容版本 | 去重回执；迟到/过时冲突标注，不回滚状态；C |
| CMD-19 POST /api/attempts/:id/disputes | S/本人尝试及同课程本人目标 | 已创建尝试 | targetRef={claimId,claimRevision} 或 {feedbackId,feedbackRevision}、理由；影响有效主张时 expectedStateRevision | 异议及所影响新状态；历史目标不覆写不同当前结论；A+C |
| CMD-20 PATCH /api/courses/:id/my-context | S/本人课程 | 本人有效课程成员 | expectedContextRevision、目标/约束自述；不接受能力/教师决定字段 | 自述新版本；相关上下文失效见本提案第 8.3 节；A+C |
| CMD-21 POST /api/attempts/:id/reviews | T/尝试所属课程与学生 | 已创建尝试；不要求已经提交 | expectedStateRevision、确切主张版本、confirm/correct/defer、理由/引用 | TeacherDecision、新有效投影及同事务失效；不自动标 reviewed；A+C |
| CMD-22 POST /api/attempts/:id/reopen | T/本人负责课程 | reviewed | expectedAttemptRevision、最新 submissionId、理由/许可内容 | ReopenGrant；仍是 reviewed，学生再主动继续；A+C |
| CMD-23 POST /api/attempts/:id/feedback | T/本人负责课程 | formative：已创建；submission_review：submitted/reviewed | kind、expectedFeedbackRevision、expectedStateRevision、来源快照/提交；改变阶段另校验 expectedAttemptRevision | 教师正式反馈追加；submission_review 与 reviewed 同事务；模型草稿不能调用；A+C |
| CMD-24 POST /api/blueprints/:id/rubric-trials | T/蓝图课程 | 草稿 | expectedRevision、获准样例引用、教师试评与歧义；不造学生观察 | 试评记录/关联修订提示；修改蓝图另走 CMD-05；A |

读接口同样校验范围，不因 ID 难猜就放开。不存在用于模型直接发布活动、编辑学生文件、运行程序或写正式评价的公共工具。模型能调用的是当前已授权上下文中的少量读取函数，如读取指定快照、读取结果、检索获准资料；调用参数再由服务端校验。

### 5.1 每个命令的幂等摘要、失败与恢复

K(scope) 为可信 userId + 稳定 CMD 编号 + scope + Idempotency-Key。D 为第 6 节规范化合法输入摘要，包含路由关联 ID 和全部合法正文；每行再列必须包含的关键项，不是只摘要这些字段。所有命令先核验身份/归属/用途：缺版本/未知字段 400，越权 403，无权查询的 ID 不泄露存在性。旧版本/旧状态/坏引用/不可用分别用第 6 节错误分类；恢复不扩大读取权。

| 命令 | K 的 scope / D 的关键项 | 拒绝/失败边界 | 恢复方式 |
| --- | --- | --- | --- |
| CMD-01 | 无业务 K/D；凭据不作持久业务摘要 | 无效凭据 401（不区分账号存在）、限流 429 | 本人重新登录/等待限流；不缓存凭据/登录响应 |
| CMD-02 | 无业务 K/D；本人当前会话撤销 | Origin/CSRF 不符拒绝；已退出安全结束 | 重复退出，不恢复已撤销会话 |
| CMD-03 | courseId；正文/可见性/前 resourceVersionId | 材料限额、前版本归属错误 | 调整材料/引用；确认新操作才换键 |
| CMD-04 | courseId；mode/获准输入/源活动或蓝图 revision/提案和候选 ID | 越权来源/旧提案/缺关键约束/模型或预算不可用 | 补条件/重读候选，查询原 Job；人工模式可用 |
| CMD-05 | blueprintId；expectedRevision/patch/proposalId/baseRevision | CAS/提案基础版本/关联冲突 | 保留未采用内容，重读比较，不自动变基 |
| CMD-06 | blueprintId；expectedRevision/局部请求/引用 | 旧 revision/越权资料/限额或预算 | 重读后主动新请求；先查询原 Job |
| CMD-07 | blueprintId；expectedRevision/疑点处理 | 旧 revision/格式或引用错误 | 修改后新版本重检；保留待验证项 |
| CMD-08 | blueprintId；expectedRevision/检查引用/疑点说明 | 旧检查/阻断/坏引用/运行未就绪 | 修正并重检，等 C 真实就绪证据 |
| CMD-09 | activityVersionId；studentIds/固定配置引用 | 非有效课程成员/活动不可用/运行未就绪 | 修正集合或等环境；不扩大名单 |
| CMD-10 | assignmentId；expectedAssignmentRevision/action/reason | 旧控制版本/事务存储失败 | 重读控制，同键查询原结果；不解除个人暂停 |
| CMD-11 | assignmentId；固定 assignmentId（本人来自会话） | 未获分配/分配暂停/当前已有提交不能重建 | 读取本人当前尝试，按反馈和许可继续 |
| CMD-12 | attemptId；完整批次/fileId/baseVersion/clientId/clientSeq/操作内容 | 旧基版本/同序号异内容/路径/阶段/限额 | 保留双端草稿，查询 ACK；确认后合并不强覆盖 |
| CMD-13 | attemptId；expectedAttemptRevision/ObjectRef/问题/意图 | 暂停/限定帮助/未确认或旧对象/预算时限 | 先同步或改引用，查询 Job；取消或人工求助 |
| CMD-14 | attemptId；expectedAttemptRevision/snapshotId/profile/模式/输入 | 暂停/坏快照配置/运行限额/执行器不可用 | 查询原 runId；outcome_unknown 不盲目重跑 |
| CMD-15 | attemptId；expectedAttemptRevision/具体控制/值/grant 引用 | 旧版本/分配覆盖/无效或已消费许可 | 重读控制和最新许可；开关分别处理 |
| CMD-16 | jobId；jobId/取消请求 | 非获准发起者/控制面不可用 | 查询 Job 和取消状态；终结结果保留 |
| CMD-17 | attemptId；expectedAttemptRevision/snapshotId/说明/检查引用 | 暂停/旧状态/坏或他人检查/必需说明缺失 | 修正引用，查询原 Submission；不重复提交或要求必须答对 |
| CMD-18 | actionId；contentHash/receiptKind/clientReceiptId | 非目标本人/不存在的内容版本/伪造种类 | 按原事实重试；迟到追加标冲突，不补造中间回执 |
| CMD-19 | attemptId；targetRef/reason/expectedStateRevision（影响投影时） | 他人或不存在目标版本/状态 CAS | 重读本人目标；历史异议不覆盖新判断 |
| CMD-20 | courseId；expectedContextRevision/目标约束自述 | 旧版本/非本人/能力或教师决定字段 | 重读并保留自述，按新请求修改 |
| CMD-21 | attemptId；expectedStateRevision/targetRef/decision/reason/引用 | 状态 CAS/无权或不可用证据/事务任一步失败 | 重读依据，同键查询；未提交不得发通知 |
| CMD-22 | attemptId；expectedAttemptRevision/最新 submissionId/许可理由 | 非 reviewed/非最新提交/旧尝试版本 | 重读最新提交与许可；教师不代学生继续 |
| CMD-23 | attemptId；kind/feedbackId（修订）/expectedFeedbackRevision/expectedStateRevision/来源正文/阶段变化时 expectedAttemptRevision | 旧反馈/状态/尝试/非最新提交/未经校验来源 | 重读对比保留正文；反馈和阶段整体重试 |
| CMD-24 | blueprintId；expectedRevision/样例引用/教师试评歧义 | 旧草稿/未授权样例/伪造观察 | 重读或换样例；另以 CMD-05 修订量规 |

CMD-12 的 HTTP K/D 与 clientSeq 去重并存：传输重试同键，单独换键也不能重复接纳相同 clientId/clientSeq；同序号不同内容冲突。提交落盘后才 ACK。二者回执与原子性待 C0 核对，A 不另建同步记录。

CMD-23/24 是现有 M-07/M-01 必需行为的接口补全提案，当前 TECH 没有这两条路径；确认前不实现。反馈修订须提供 feedbackId，服务端不按“最后一条”猜对象；首次创建无 feedbackId、expectedFeedbackRevision=0，服务端生成 ID。提交审阅绑定最新 submissionId。教师样例执行已有 TECH 第 8.1 节依据，但公共入口缺失，记为 A0-R05 交 C0 准备；CMD-24 仅保存试评，不代替运行接口。

## 6. 公共响应、错误与幂等提案

成功响应包含 requestId、data 与受影响的版本；异步命令接纳只返回 jobId/排队状态及可查询引用，不冒充已生成/展示。业务命令可附 commandId/replayed；单次网络重试有新 requestId，但原命令的业务结果与对象 ID 保持相同。错误响应包含 requestId、error.code、可恢复 message、retryable 与必要的恢复方式，不回传堆栈/内部路径/密钥/私有正文。

| HTTP/业务错误码 | 触发 | 调用方恢复 |
| --- | --- | --- |
| 400 INVALID_REQUEST | 字段/类型或必需预期版本缺失；试图传入不允许的赋权字段 | 修正输入；不执行副作用 |
| 401 UNAUTHENTICATED | 未登录/会话失效 | 重新登录；不以请求参数补身份 |
| 403 FORBIDDEN | 课程/本人/资源/用途越权 | 拒绝；错误不说明他人内容或私有资产细节 |
| 409 VERSION_CONFLICT | revision/CAS 不匹配 | 在已有读取权下重读版本，保留本次未采用内容 |
| 409 IDEMPOTENCY_CONFLICT | 同键不同规范化请求 | 修正调用；确属新操作才使用新键 |
| 409 STATE_CONFLICT | 当前阶段、分配暂停或许可不允许 | 展示状态和可做操作；不强行改阶段 |
| 413 CONTENT_LIMIT | 已冻结文本/源码/输入限额超出 | 拆分或缩小；不无声截断 |
| 422 INVALID_REFERENCE / INVALID_CONFIGURATION / RUNTIME_NOT_READY | 引用/版本关系、目标关联、配置/就绪条件不成立 | 修正引用/配置或等待真实就绪，不能伪造 ready |
| 429 RATE_LIMITED / BUDGET_EXHAUSTED | 登录/命令配额或模型预算限制 | 明确限额；预算耗尽仍能保存和人工复核 |
| 503 DEPENDENCY_UNAVAILABLE / PERSISTENCE_UNAVAILABLE | 模型/执行器或存储不能确认结果 | 标明不可用/未知；查询原命令，不盲目新建副作用 |

业务幂等由 C 的 CommandReceipt 提供，A/B 不再建第二套。键范围为可信账号 + 稳定命令编号 + 目标资源（创建按课程/分配作用域），保存规范化合法输入的摘要；摘要包含预期版本/目标与实质内容，不把 role 或客户端时间当权限。规范化对象键顺序、不改正文/换行语义；含凭据的认证请求不进入此摘要。

认证与 Origin/CSRF 检查在回执读取之前。事务内先找既有回执：同键同摘要再核验当前读取权后返回原业务结果，不因第一次成功已增加 revision 而冲突；同键异摘要立即 409。新命令再检查当前版本/状态，业务写入、结果回执和事件同事务提交；失败不留半笔成功回执。唯一约束处理并发同键，不能仅靠“先查再写”。

CMD-01 使用认证限流/会话规则，不缓存密码或登录响应作为业务回执；CMD-02 撤销当前会话并安全重复退出。CMD-18 的回执事实还按 TECH_DESIGN 第 7.3 节内容版本去重。回执重放不重发学习动作；已不可见内容不因旧结果缓存泄露。事件补取与业务命令重试是两件事。

## 7. 读取与角色投影提案

读接口不产生学习阶段转换、不触发模型或运行。所有对象 ID 经服务端归属核验，GET 不使用 Idempotency-Key；数据返回确切版本/范围与真实缺口。表内路径为初始对接候选，不要求 UI 使用特定页面组织。

| 编号/接口 | 角色/范围 | 载荷与限制 | 实现责任 |
| --- | --- | --- | --- |
| READ-01 GET /api/session | 当前本人 | 可信本人/有效课程角色与会话状态，不含凭据 | A |
| READ-02 GET /api/courses | 当前成员 | 仅所属课程；维护权限不扩展课程教学读取 | A |
| READ-03 GET /api/resources/:versionId | T；S 需本人 attemptId | T 负责课程；S 只能读固定活动关联且可见段落；正文不按客户端 audience 提权 | A |
| READ-04 GET /api/courses/:id/blueprints；GET /api/blueprints/:id | T/负责课程 | 草稿 revision、关联与检查/疑点；不下发给学生 | A |
| READ-05 GET /api/proposals/:id | T/原请求所属课程 | 已校验候选/草案/局部差异、baseRevision、引用/影响项/过时状态 | A，B 提供输出 |
| READ-06 GET /api/blueprints/:id/preview | T/蓝图课程 | student/tutor/teacher-validator 三种明确投影；预览不分配、不确认 | A |
| READ-07 GET /api/activities/:id | T；S 需所属 assignmentId | 固定活动及绑定版本；S 按所属分配裁剪，不读取别人的分配/草稿 | A |
| READ-08 GET /api/courses/:id/progress | T/负责课程 | 实际尝试/提交/待处理项入口，无日志数量推导的能力分数 | A 聚合，C 提供事实 |
| READ-09 GET /api/attempts/:id | S/本人；T/负责课程 | 活动/生命周期/控制/revision、确认作品与恢复引用；GET 不恢复/重开 | C，A 鉴权 |
| READ-10 GET /api/attempts/:id/submissions | S/本人；T/负责课程 | 固定提交列表与版本；旧提交只读 | C |
| READ-11 GET /api/attempts/:id/review-context | T/负责课程 | 原始引用、候选/决定、作品结果、三条能力线、帮助/反证/未知及覆盖 | A 聚合，B/C 提供受控数据 |
| READ-12 GET /api/courses/:id/learner-states/:studentId | S 仅本人；T/负责课程 | 当前 revision、适用目标/活动投影、本人主张异议/处理与撤回说明；私有引用裁剪；历史不自动全包进入模型 | A |
| READ-13 GET /api/courses/:id/my-context | S/本人 | 自述及 contextRevision；未创建返回 null；T 读取有关部分走审阅上下文 | A |
| READ-14 GET /api/attempts/:id/feedback | S/本人；T/负责课程 | 正式反馈追加版本、固定来源/依据有效性/争议标记与修订许可；不把 B 未采用草稿给学生 | A，C 提供许可状态 |
| READ-15 GET /api/jobs/:id | 有权发起 Job 的本人 | 状态/失败/结果引用；学生教学原文须通过政策与投递校验才可取 | C，B 配合 |
| READ-16 GET /api/attempts/:id/events | S/本人；T/负责课程 | 裁剪 SSE/游标分页；每次连接/补取再鉴权，无未审查答案流 | C |
| READ-17 GET /api/courses/:id/exports | T/负责课程 | 一致范围导出与引用/版本/覆盖，排除凭据；不以导出绕过授权 | C，A 鉴权 |
| READ-18 GET /api/my/exports | S/本人指定课程 | 只导出本人可见记录，私有资产/其他学生内容不进入 | C，A 鉴权 |
| READ-19 GET /api/attempts/:id/snapshots/:snapshotId；GET /api/attempts/:id/runs/:runId | S/本人；T/负责课程 | 确认快照与真实运行/检查结果；完整私有验证细节按角色裁剪 | C |

## 8. B/C 接入与保存归属提案

| 能力 | 输入范围/版本 | 输出与保存归属 | 权限边界 |
| --- | --- | --- | --- |
| B1 教师生成/局部建议 | 教师请求、获准资料、约束、blueprintId/baseRevision（局部建议）；jobId/取消与截止 | 候选/提案、差异/影响项/引用/版本由 B 输出，A/design 保存；用量/job 由 C 保存 | 不编辑主草稿或确认活动；输入不足可说明缺口，不要求完整机器配置 |
| B2 学生上下文读取 | 可信本人/课程/attemptId、ObjectRef、固定活动、当前 epoch/状态 | A 提供获准活动/政策、相关有效主张/反证、自述与正式反馈；C 提供确认作品/结果/实际帮助 | 只按当前用途读取；无历史可读 revision=0/空主张；无私有答案 |
| B3 候选提交 | jobId/输入引用、预期状态/epoch、主张范围/帮助/未知、模型/提示/分析版本 | A/review 校验后接纳候选并投影；C 保留真实输入与分析记录；模型反馈仍是草稿 | 不能伪造观察、改 TeacherDecision 或绕过已撤回依据；旧版本拒绝直接写新头 |
| C2 分配/尝试接入 | 可信本人、assignmentId、固定 activityVersionId、当前分配控制 | A 返回授权/版本/政策；C 创建本人尝试/快照，暴露事务与恢复能力 | 不以旧许可或共享 ID 绕过暂停；不触发模型/自动学生运行 |
| C3 纠正传播接入 | A 的当前事务、课程/学生/相关尝试范围、失效原因与受影响类别 | 同一事务增加 epoch/失效 Job/Action/事件；提交后再通知/取消外部请求 | 不另开独立写事务，不改教师理由/教学判断，不抹掉已展示事实 |
| C4 提交/反馈/重开接入 | 固定 submissionId、尝试版本、正式反馈或教师许可、学生主动命令 | C 保留旧提交/生命周期；A 保存反馈/许可，与阶段变化同事务 | 教师许可不等于学生已继续；新活动只用于新尝试 |

接口实现与字段校验由对应负责人提供；A0 只提交汇总草案。B0/C0 评审应回填输入/输出格式、错误、失效范围及工程样例，不能仅写“能调用”。

### 8.1 短事务入口提案

下面的函数名表达模块契约，可在实现时按 G0 工程调整；**不是已存在 SDK/代码**。tx 为 A 创建的本次短事务上下文，C 的调用加入同一事务，不启动独立提交、不做网络等待。鉴权入口与事务资源读取都必须采用当前服务端记录。

| 提供方/能力 | 输入 | 输出与限制 |
| --- | --- | --- |
| A authorizeResource | 可信 actor、目标资源、稳定操作编号/用途 | 获准课程/本人/版本范围；拒绝不回传私有正文 |
| A withTransaction | 同步本地读写操作 | 原子提交/整体回滚；异步模型/runner/网络不能放入 |
| C resolveCommandReceipt / finishCommand | tx、可信账号/命令/目标/key/请求摘要；最终结果 | 已有同请求结果或新命令槽；同键冲突；结果/审计/事件与业务同提交 |
| C enqueueJob / appendEvent | tx、已获准 scope、版本/epoch、kind/输入引用/截止 | jobId/event 引用；不在事务里调用供应商或执行器 |
| C invalidateDecisionContext | tx、courseId/studentId/相关 attempt 范围、原因 | 原子增加相关 epoch、标旧动作/分析 stale、追加事件；不改原始事实 |
| C cancelByPurpose | tx、获准 attempt 范围、reminder/passive_analysis 等明确用途、原因 | 针对类别取消；支持不增加教学 epoch 的独立开关 |
| C transitionAttempt / grantReopen | tx、已授权操作、expectedAttemptRevision、提交/许可引用 | 新阶段/许可版本；不接受任意目标 state 写入或自动学生继续 |
| C readEvidence / readSubmission | 可信用途/课程/本人、确切 ObjectRef/Submission 引用；事务核验时可用 tx | 当前可用的原始引用/快照/结果/帮助/覆盖；拒绝或缺口明确 |
| C readRuntimeReadiness | 固定 runtimeProfileVersion 与验证配置 | 可信就绪状态、镜像/规则版本、验证证据和检查时间；不能以请求体 ready 代替 |
| C notifyCommitted / requestExternalCancellation | 已提交事件/取消引用 | 提交后推送/取消；失败可重试通知，不重做教学决定或运行 |

A1 先交权限/事务入口；C2 首批交 resolveCommandReceipt、finishCommand、enqueueJob、appendEvent 及可组合基础；A1 再完成命令联调，A3 开始真实教师作业接入。C3 再完成消息/用途取消和完整 epoch 失效；不等 C3 才首次提供 Job，也不把 C 的实现搬到 A 模块中。

教师纠正、分配控制、异议引起的 contested、反馈推动 reviewed 与修订许可分别通过本次 tx 组合业务与 C 的能力。初始逻辑约束包括：回执键唯一、同课程/学生状态头 CAS、蓝图 revision CAS、当前尝试唯一、许可绑定最新提交、外键/引用完整性。物理 SQL 方案仍待授权，不以该列表自动执行建表。

每条新命令的提交顺序：

1. 请求身份/Origin 检查，解析合法输入与摘要；事务内核验当前授权并处理既有回执。
2. 新命令检查当前版本、关联和阶段；涉及快照/证据/许可/就绪时再次核验可信记录。
3. 写本领域对象；调用 C 同事务的生命周期、epoch/用途失效、作业与事件能力。
4. 保存规范化业务结果和成功回执；任一步失败回滚所有数据库写入。
5. 提交后通知/取消外部请求；调用方网络结果未知时查询/重试原命令，不换新键重做。

真实纠正联调必须逐处注入失败并核验决定/状态/epoch/Job/Action/事件/回执的原子性；旧模型完成再检验版本守卫。A/B 的测试替身只证明逻辑，不替代 C3 的同一数据库事务验收。

### 8.2 候选接纳与教师纠正并发

B 输出候选不写 A 的状态头。候选信封至少含 jobId、courseId/studentId/attemptId、activityVersionId、expectedStateRevision、decisionEpoch、已确认输入引用、policy/模型/提示/分析版本、主张/反证/帮助/未知及 collectionPurpose。服务端从获准 Job 的 scope 校验这些字段，不相信模型回传的身份或 evidenceIds。

A 在一个短事务核验当前权限/状态/epoch、用途开关、原始引用存在/版本/范围和撤回约束，保留 B 的原始候选（C 分析记录）与接纳/拒绝原因，成功才追加有效状态 revision；失败不能将旧估计悄悄换上新 revision。被动分析须检查采集许可。B 的反馈草稿另保存为草稿/提案，不调用 CMD-23。模型正文、事实记录、正式教师决定分别读取，不共用一个“结论正文”覆盖字段。

纠正交易依 TECH 第 7.2 节：教师授权与状态 CAS → 追加决定 → 新投影/依赖主张 contested → C 同事务 epoch/Action/Job 失效 → 事件/成功回执 → 提交后通知/外部取消。撤回约束保留被否定的依据与适用范围，候选换 ID/措辞/分析版本也不能用相同依据恢复被撤回主张。

- 候选先提交：状态头增加，旧 expectedStateRevision 的教师表单 409；教师重读，不能静默覆盖。
- 教师先提交：旧候选在最终 CAS/epoch 守卫拒绝，可留分析历史，不能生效。
- 同时提交：只有一个当前头通过 CAS；任一步失败全体回滚。
- 已展示帮助：保留真实内容与回执，标注依据后来修订；未展示旧动作失效。浏览器在实际 UI 接入后仍须检验本地版本，不能以服务端通知宣称撤回已看到内容。

以上是并发验证预期，**未执行真实数据库联调**；B/C 信封、返回值与去重规则待会审。

### 8.3 失效类别与独立控制提案

失效操作由 C 提供，并加入触发业务的同一数据库事务；实际 provider/runner 取消和通知在提交后执行。失效范围不能简单写成“所有 Job”：教师纠正不应取消学生已授权的普通运行事实，关闭提醒也不能使正在等待的主动帮助失效。

| 触发 | 事务内范围/版本 | 要失效/取消的内容 | 保留/允许 |
| --- | --- | --- | --- |
| 教师纠正、确认争议处理改变有效依据 | 同课程同学生相关教学上下文；新状态 revision、相关 Attempt epoch | 未展示教学动作、依赖旧状态的分析/反馈草稿；保守整体失效，不建通用依赖图 | 原始作品/运行/已显示帮助保留；无关学生/课程不受影响 |
| 学生对当前有效主张提出异议 | 异议+新 contested 投影+相关 epoch | 依赖该依据的待投递动作/分析 | 保留原主张与异议；仅历史/反馈正文异议不伪造新能力主张 |
| 新主动求助取代旧求助 | 同 Attempt 新 epoch/新 help Job | 旧有效帮助/待投递内容；不再同时有两个有效帮助 Job | 已显示帮助保留；学生文件/运行不变 |
| 分配暂停/恢复、个人尝试暂停/恢复 | 分配/尝试控制版本及受影响 epoch | 暂停时相关教学动作/分析、待运行；正在执行取消/归档按 C 处理；恢复不重放旧动作 | 保存已有草稿/读旧事实；个人 paused 不被教师恢复覆盖 |
| 进入/退出限定帮助检查 | 可信检查阶段、政策约束与新 epoch | 不符合检查政策的待投递辅导/提醒 | 只限制系统内帮助；不声称阻止外部帮助；既有运行/检查事实可追溯 |
| 关闭主动提醒 | 同 Attempt 控制版本，单独检查提醒许可；不增加通用教学 epoch | 仅待投递/生成的 reminder | 主动求助和被动分析依其各自规则继续；不重写历史帮助 |
| 暂停过程采集 | 同 Attempt 控制版本+覆盖区间；不增加通用教学 epoch | 新逐次过程采集和排队/进行中的被动分析；迟到分析再次检查开关 | 显式保存/求助/运行/提交的必要功能数据继续；不据此恢复自动候选提取/反馈草稿；人工复核继续，恢复不回填空窗 |
| 自述/正式反馈改变相关辅导上下文 | 自述/反馈新版本，相关 Attempt epoch；不自动增加能力状态 revision | 依赖旧上下文的教学动作/分析 | 自述、正式评价、能力主张仍分层；学生主动作业事实不被改写 |

文件内容变化使用 snapshot/documentVersion 检查，不把每次编辑都当作能力状态或 epoch 更新。课程要求/政策变更通过新 ActivityVersion；如果必须停止旧活动先暂停分配，不能原地修改旧尝试规则。

投递/候选写入时在真实事务中重查相关版本、epoch、分配/个人阶段、政策与用途开关；不能仅在模型调用前检查。已生成但未获准内容不能先出现在 SSE/Job 结果中。网络通知与教师纠正存在展示竞态，按 TECH_DESIGN 第 7.3 节由客户端再核验，并保留真实迟到事实，不宣称已发送内容可被“收回”。

## 9. 后台选型与目录责任评审稿

沿用 TECH 第 1～3/10 节的推荐组合：单进程 Node.js 受支持 LTS + TypeScript/Fastify，Zod 校验，better-sqlite3/SQLite 本地 WAL，HTTP/SSE 与 C 的 SQL Job/CommandReceipt。B 负责 AI SDK 与供应商适配；C 负责独立 Linux 执行边界。**本次不新增选型结论、精确依赖版本或安装结果**；这些机制/限制的一手资料入口已在 TECH 第 13 节保留，冻结前需复查实际支持版本、驱动内嵌 SQLite 修复版本及组合兼容性。

选型理由是业务与记录可在同一短事务提交、少量成员共享一个应用进程，避免为 A/B/C 各建服务/数据库/队列。一个并发写者、同步查询阻塞和事件循环延迟是实际限制；是否达 NFR 需实测。多实例写入或实测无法达标再提替代方案，不预建 Redis、向量库、通用工作流或微服务。

| 拟议位置 | 写入责任 | 跨模块约束 |
| --- | --- | --- |
| apps/teaching/package.json、锁文件、tsconfig、启动入口 | A 汇总，B/C 提依赖需求 | 选型/版本确认后创建；新增公共依赖先核对 |
| apps/teaching/contracts/common、access、design、review | A 维护对应定义，B/C 会审共享字段 | 不单方改公共 ID/版本/错误/事务语义 |
| apps/teaching/contracts/tutoring | B | 消费共同字段；候选格式交 A 采用与校验 |
| apps/teaching/contracts/workspace、records、runner | C | 提供 ObjectRef、Job/Receipt/Event 与同事务失效 |
| apps/teaching/server/app、access、design、review、db/transaction | A | A 维护连接/短事务入口；领域与记录写入同数据库 |
| apps/teaching/server/tutoring | B | 不直接写学生文件、TeacherDecision 或发布活动 |
| apps/teaching/server/workspace、records | C | 不重复实现 A 的学习状态/教师权限规则 |
| apps/teaching/runner | C | 独立执行节点；不挂应用数据库/私有答案/凭据 |
| 对应领域 tests | 各领域负责人 | A 权限/设计/纠正；B 语义/候选；C 同步/运行/恢复；跨域联调共同验证 |
| 团队 Web UI 既有工程 | UI 成员 | 最后实际接入；不预建 web 目录或重选框架 |

上述均为拟议代码位置，当前仓库没有这些目录/源码/启动或测试命令。最终“实际目录与共享文件责任”待 G0 会审记录，不将建议当作已经落地。数据库对象见第 2.1 节，约束见第 8.1 节；**不提供可执行建表/迁移脚本，不获得实际建表授权**。

## 10. D-03 账号、访问与数据管理提案

### 10.1 待冻结参数与受阻工作

提案沿用 PRD D-03 与 TECH 第 9/10 节“私有邀请制、最少数据、独立执行和一致性备份”方向。数值/人员范围以下均为建议或待定，不能当作试点同意、已批准安全方案或自动删除指令。A/C 准备，教学/技术负责人决定；B 核对模型数据范围，D-02 与 D-03 分开批准。

| 编号/参数 | 具体提案/待确认内容 | 准备/决定责任 | 未冻结时受阻工作 |
| --- | --- | --- | --- |
| A0-D01 试点参与者与课程关系 | 一位任课教师、有限指定学生；明确课程成员与试点授权，不采集无关身份资料；不重复收集 A/B/C 姓名作为开发前置 | A 整理参与范围；教学负责人确认 | 真实账号预置/真人试点；合成权限验证可准备 |
| A0-D02 账号建立与发放 | 首版推荐维护者预置受控账户，或既有受控邀请方式；无公开注册/SSO；凭据经批准通道分别发放，不进入仓库/聊天/导出 | A 提发放与初始验证/恢复方案；技术负责人选择 | 正式登录流程冻结与真实账户发放 |
| A0-D03 会话与登录策略 | 服务端随机 Session、HttpOnly/Secure/SameSite cookie、Origin/CSRF、成熟密码哈希/限流；会话绝对/空闲期限、失败窗口/上限、退出/改密撤销范围、密码恢复验证方式待定 | A；技术负责人确认 | 正式认证配置；不虚构现成组件或性能 |
| A0-D04 成员变更与账号停用 | 每次命令/读取/事件再核验；停用后禁止新访问/作业，取消受影响未完成作业，保留真实历史；不冒充学生学习操作 | A/C 核对取消与历史读取；技术/教学负责人确认 | 真实停用/权限变更操作；合成验证可准备 |
| A0-D05 网络与入口 | 同源 HTTPS 的私有试点入口；具体内网/VPN/有限外部可达及成员设备访问方式待决定，不默认匿名公网开放 | A/C 清点现有条件；技术负责人确认 | 正式部署/真人网络访问 |
| A0-D06 访问人员与用途 | 任课教师读取负责课程、学生读取本人；维护者健康/备份最小元数据，正文排障需限定对象/理由/时限/审批与审计；不赋予维护者正式评价权 | A；教学/技术负责人确认访问清单 | 真实资料访问、正文排障与导出权限发放 |
| A0-D07 告知与数据用途 | 说明过程记录/功能数据/提醒三个独立控制、模型范围、覆盖缺口、导出/异议/撤回/删除途径；实际 UI 入口由 UI 成员最终接入 | A 提业务说明，B/C 核对；教学负责人确认 | 真人采集与外部模型发送；不先采后告知 |
| A0-D08 保留与处置 | 分类保留期限待定；可沿用“试点结束后 90 天复核去留”的建议，绝非到期自动删除；明确何种引用撤回/脱敏、处理时限和备份到期路径 | A/C 提分类清单；教学/技术负责人确认 | 真人持久化试点；实际删除单独授权 |
| A0-D09 模型处理范围 | 仅当前学生/课程必要获准上下文，不含凭据/私有答案；供应商/区域/保留/训练使用/重试记录等由 B 的 D-02 提案核实，再与告知范围相对照 | B 核对，A/C 配合；产品/技术负责人确认 | 真实数据或付费调用；合成样例可准备 |
| A0-D10 应用与执行资源 | 一个本地持久磁盘应用节点、一个独立 Linux 节点/VM；节点规格/磁盘配额/内网认证/镜像与语言由 C 提证据；资源限额沿用 NFR-03，不擅自下调 | C 清点，A 核对控制面/存储；技术负责人确认 | C1 真实隔离验证/正式部署/真人运行 |
| A0-D11 备份与恢复 | 推荐教学会话前后做 SQLite 一致性备份；目标 RTO 30 分钟、RPO 最近成功备份均待确认；健康重启不丢已 ACK 事务；备份位置/访问/加密/保留/校验/恢复演练待 C 核对 | C 提方案和演练证据；A 核对课程权限与版本；技术负责人确认 | 真实试点恢复承诺；不得只复制活跃 db 漏 WAL |
| A0-D12 处理与支持责任 | 任课教师处理教学异议，A 负责业务授权/数据请求受理规则，C 负责备份/恢复及获授权处置执行，B 负责模型调用范围与故障记录；具体审批通道/处理时限待定 | A/C 整理，B 核对；教学/技术负责人确认 | 真人导出/撤回/删除、事故处置承诺 |

这些参数只阻塞其依赖操作。A0/B0/C0 文档、合成验证方案继续；G0 未确认实施方案前不据本文建立正式服务。也不能因 D-03 真人条件尚未具备而声称通用契约无法起草。

### 10.2 数据类别、投影与处理责任

| 数据类别 | 保存/读取责任 | 用途/导出边界 | 撤回/保留处置 |
| --- | --- | --- | --- |
| 账号、成员、Session、凭据 | A/access；维护者仅获批准账号管理 | Session token、密码哈希/salt、部署凭据不进入课程/个人导出、模型或日志 | 停用/撤销会话不改事实；账号与历史关联去标识方案待审批 |
| 教师材料/私有验证资产 | A/design 版本化；C 验证器只用指定资产 | 按 student/tutor/teacher-validator 投影；导出和引用下载同样过滤 | 删除/脱敏先识别固定活动引用，标缺口/停用影响，不静默换版本 |
| 学生作品、确认快照、提交/运行事实 | C/workspace/records；T 课程审阅，S 本人 | 功能数据即使采集暂停仍按必要范围保存；学生 stdout 不代表验证 | 固定提交可追溯；保留期限/备份周期待定；移除后引用不得伪造 |
| 消息/展示回执、观察、覆盖区间 | C/records 保存实际内容和来源 | 接收/展示/确认分层；无回执和空窗保持未知；S 仅本人可见部分 | 不把恢复后事件回填空窗；处理旧依据时保留真实帮助事实 |
| 模型候选/分析/反馈草稿 | B 产出，C 保存原始分析，A 保存接纳/有效投影 | 未采用反馈不发学生；教师审阅候选与事实分开；私有引用不借摘要泄漏 | 撤回不换 ID 恢复同一依据；保留 rejected/stale 原因，期限待定 |
| 自述、TeacherDecision、正式反馈 | A/review 追加版本；S 本人，T 所负责课程 | 原始引用、帮助/未知、判断版本及争议可追溯；不凭无依据赞同补观察 | 经授权移除证据后有效投影失效/标缺口，旧反馈标依据状态 |
| Job/Receipt/Event/Audit 与诊断元数据 | C 提公共记录；A/B 使用 | 日志只需 requestId/jobId/类型/时延/失败分类/配置版本；正文不默认进运维日志 | 业务幂等回执保留窗口与离线重试范围待 C0/G0 确认，过期不得盲目重做 |
| 备份与导出副本 | C 按获准范围执行；A 鉴权；维护者最小访问 | 导出是授权投影，备份不是任意教学读取入口；副本与原始实验证据分开 | 保留与删除范围必须包含副本；恢复后复核已批准撤回/脱敏决定，防止旧状态复活 |

D-03 处置流程提案：核验申请人/课程/目标和权限 → 列相关事实/候选/决定/导出/备份影响与不可用引用 → 教学/技术负责人决定并取得操作红线授权 → C 执行获准范围、A 使有效引用/投影同步失效 → 留最少处理审计与真实缺口 → 在隔离恢复演练中核验处置不会被旧备份反向恢复。首版不建设通用自助删除后台；本次不执行任何删除/脱敏或备份配置。

## 11. 验证方案与预期结果

以下全部为**计划用例，业务测试未执行**。准备隔离合成 fixture：T1 负责课程 C1，T2 负责 C2，S1/S2 为 C1 不同学生，S3 为 C2 学生，O 仅维护；活动 V1/V2、蓝图 revision 1/2、同名不同 fileId、确认快照 H1/H2，关联 Job/Action/Claim 与被撤回依据。不要创建真实账号/学生记录，不把 fixture 当成真实教学结果。

| 用例 | 操作/触发 | 预期与失败判据 | 对应/负责 |
| --- | --- | --- | --- |
| A0-T01 | S1 访问 S2 的 Attempt、快照、Job、events、export | 每条入口拒绝，无正文/存在性侧漏；ID 难猜不能成为防护 | AC-15/NFR-04；A+C |
| A0-T02 | T1 审阅 C2、O 试写正式反馈、伪造 body role/studentId | 服务端会话/课程决定权限，越权无写入/回执/事件 | AC-15/NFR-04；A |
| A0-T03 | S1/B tutor 请求 teacher audience、私有段落/判定答案 | 检索前/序列化/Job/SSE/导出同样裁剪；私有摘要也不泄漏 | AC-02/15；A+B+C |
| A0-T04 | 草稿 revision=2 使用 revision=1 PATCH/Proposal/Check 发布 | 409 或失效引用；新人工内容与旧固定活动不变 | AC-02/NFR-04；A |
| A0-T05 | 改材料/量规/政策后读取已开始 V1 | V1 原绑定不变；V2 只用于新显式分配/尝试 | AC-02/09；A+C |
| A0-T06 | 相同 K/D 重试及两个并发相同请求 | 原 ID/结果，仅一次副作用/事件；requestId 可不同；不因初次版本增加冲突 | AC-12/15/NFR-04；A+C |
| A0-T07 | 相同 K 不同 D，含不同 expectedRevision/正文/引用 | 409 无第二次写入；规范化不损正文/换行；秘密不进摘要 | AC-15/NFR-04；A+C |
| A0-T08 | 修改归属/撤销权限后重放原成功回执 | 当前读取权再检查，不因缓存泄漏旧结果 | AC-15；A+C |
| A0-T09 | sync 同 seq 同内容与同 seq 异内容，断线恢复 | 去重/冲突正确，落盘才 ACK，双端草稿保留，不假装已同步 | AC-03/11/12；C+A |
| A0-T10 | ready GET 与首次合法帮助/运行/同步/提交 | GET 不激活；学生命令按会审后的阶段规则原子接纳；无强制开工表单 | AC-03/05；A+B+C |
| A0-T11 | 分配 paused、个人 paused、采集/提醒开关组合 | 前两者优先，已有草稿保存允许；独立开关不彼此恢复；submitted 不解锁 | AC-06/11；A+C |
| A0-T12 | 相同控制目标再次设值、关闭提醒但主动帮助运行中 | 不造多余 epoch/空窗；只取消 reminder，主动帮助仍可完成 | AC-06/NFR-04；A+B+C |
| A0-T13 | 采集关闭时帮助/运行/提交、迟到被动分析、恢复采集 | 功能数据保存；无新增过程/被动候选或反馈草稿；迟到守卫拒绝；空窗不补 | AC-06/07/11；A+B+C |
| A0-T14 | 对当前 claim 提异议及对历史 claim/feedback 异议 | 当前有效依据 contested 并失效旧依赖；历史不覆盖新主张/反馈 | AC-07/08；A+B+C |
| A0-T15 | 两个旧教师表单、候选与教师纠正两种提交顺序 | CAS 仅一个新头；旧候选拒绝生效，旧教师表单重读；无丢失更新 | AC-08/NFR-04；A+B+C |
| A0-T16 | 在决定/投影/epoch/Action/Job/Event/Receipt 写入各处注入失败 | 同数据库全回滚，提交前无通知；单模块 mock 不算实际事务通过 | AC-08/15；A+C |
| A0-T17 | 纠正后旧模型完成、SSE 重连、迟到 displayed | 旧结果无效；已展示事实保留并标依据修订；不回滚新状态 | AC-06/08；A+B+C，UI 后验 |
| A0-T18 | 用同被撤回依据换 claimId/措辞/分析版本 | 不能复活主张；相关后续任务只读适用有效状态，缺迁移证据保持未知 | AC-08/09/16；A+B |
| A0-T19 | 教师允许修订、学生继续、重复许可消费与后续新任务 | 许可不代学生继续；CAS/消费一次；旧提交不变，新活动另建尝试 | AC-09/12；A+C |
| A0-T20 | 课程/个人导出、恢复备份、引用被批准撤回后的读取 | 授权投影且引用/帮助/未知/版本/覆盖完整；无凭据/私有答案；恢复不复活失效依据 | AC-12/15；A+C |

实际执行方式待正式工程/依赖及所需操作授权具备：A1 权限/事务 seam 用 Node 测试运行器，C2 实际 Receipt/Job/同步接入后验证去重，A4+C3 真实单数据库失败注入和竞态，B3 候选真实联调，A6 综合验证。浏览器回执/草稿/返回/键盘场景等待实际 UI 的最终接入；语义质量仍需课程审阅，结构检查不替代教学验收。当前没有 npm/test/typecheck 命令可运行，不编造命令或通过统计。

## 12. 会审差异、待决策项与阻塞范围

### 12.1 需要 B/C 或负责人裁决的差异

所有状态均为“待评审/待决定”，没有成员签认或已批准结论。A 负责汇总，B/C 在各自任务文档提交意见和例子；本记录不授权 Agent 给其他成员发消息或替他们签认。

| 编号 | 对应基线/差异与提案 | 需要的核对/决定 | 未解决时阻塞 |
| --- | --- | --- | --- |
| A0-R01 | MVP 第 4 节 ready 可帮助/运行，TECH 第 6.1 只写 active；提议首次合法学习命令同事务开始 | A/B/C 核对触发和 expectedAttemptRevision；负责人确认基线修订 | 首次帮助/运行/同步的真实接入 |
| A0-R02 | TECH 第 7.1 closed 缺 MVP 行为/命令；提议首版不增关闭操作，待明确技术枚举 | 负责人决定；C 核对状态保存 | Attempt 枚举最终冻结 |
| A0-R03 | 活动暂停使用与分配暂停的层次；新分配是否也受活动停用控制 | A/C 提具体控制/权限；负责人决定 | 活动停用/分配控制最终冻结 |
| A0-R04 | paused 保存“现有草稿”与 sync 新建/回收操作边界 | C0 明确 file create/recycle 与同步事务，A 核对权限 | 暂停期间文件管理行为 |
| A0-R05 | M-01 量规试评、TECH 8.1 教师样例运行尚无公共入口 | A 的 CMD-24 仅试评；C0 提样例执行 scope/快照/配置/错误 | 教师样例真实执行接口 |
| A0-R06 | M-07 正式反馈现有路由不足；CMD-23 提 kind/确切 feedbackId/提交引用/CAS | A/C/B 核对草稿/正式保存与 reviewed 同事务；负责人确认 | 正式反馈与阶段接入 |
| A0-R07 | C Receipt/Job/Event、幂等摘要、sync seq、回执保留/重启的实际返回值 | C0 给输入输出/并发与恢复样例；A/B 核对 | A1 最终幂等、A3/B1/C2 真实联调 |
| A0-R08 | 纠正/异议/上下文变化与 C epoch、按用途取消同一短事务 | C0/B0 提失效范围/失败类别/候选与消息守卫；A 汇总 | A4/C3/B3 真实纠正联调 |
| A0-R09 | 候选保存分工：B 输出，C 原始分析，A 接纳/投影/正式决定；撤回约束跨新候选 | B0 给完整信封/反证/帮助/未知样例，C0 给原始引用/持久化接缝 | 有效学习状态/分析最终冻结 |
| A0-R10 | 关闭采集/提醒不使主动帮助整体失效；被动分析迟到不得写新头 | B/C 核对 purpose/collection 守卫与取消分类；负责人确认细化 | 三种控制的消息/候选联调 |
| A0-R11 | 后台版本、物理目录、短事务/连接所有权、逻辑约束 | A/B/C 会审与总负责人确认具体首批实施；物理建表另行授权 | 正式工程创建/依赖安装/大规模实现 |
| A0-R12 | D-03 的账号/网络/正文访问/保留/资源/备份参数及处理责任 | A/C 准备，B 核对 D-02 数据范围，教学/技术负责人决定 | 真实账号/数据/部署/真人试点；通用文档可继续 |

### 12.2 五处交界的会审记录

| 交界 | 本稿输入 | 必须取得的实际记录 | 当前结论 |
| --- | --- | --- | --- |
| 身份与版本 | 第 2/3/4/5/7 节 | B0/C0 的可信身份/资源范围/ObjectRef/CAS 兼容意见和拒绝样例 | 待评审，无签认 |
| 作业、消息与取消 | 第 5/6/8.1/8.3 节 | C0 公共记录格式，B0 的 deadline/取消/过期；每个控制与用途一致 | 待评审，无签认 |
| 事实、候选与教师决定 | 第 2.1/8.2 节 | B0 候选信封、C0 引用/Audit/失效、A 纠正交易和撤回约束一致 | 待评审，无签认 |
| 运行与可信检查 | 第 3/4/5/9/10 节 | C0 教师样例/学生运行/就绪验证、私有资产投影与失败分类 | 待评审，无环境验收 |
| 恢复、覆盖与导出 | 第 6/7/10/11 节 | C0 ACK/结果未知/一致性备份/投影/空窗与实际恢复方案，A/B 数据用途一致 | 待评审，无演练 |

会审意见按“引用成员任务文档/PR → 指出确切差异与工程样例 → 本文修订记录 → 决定者/范围/时间 → 授权维护者同步基线”记录。意见未到达时保持待评审，不以“无反对”当同意。A0-R01～12 的关键冲突未解决不记 A0 已交付；不关闭 #2 或 #1。

## 13. 授权维护者的基线汇总请求

此表是明确的汇总请求，**本次未修改这些文件，均为待授权维护者汇总**。

| 目标文件/章节 | 建议汇总内容 | 前置条件 |
| --- | --- | --- |
| TECH_DESIGN 第 4 节 | 第 2 节公共字段/逻辑约束、证据/候选/决定保存归属 | 字段/候选/ObjectRef 与 B0/C0 一致；获写入授权 |
| TECH_DESIGN 第 5 节 | 第 3/5/6/7 节角色矩阵、24 命令及读取投影、失败/恢复 | 新增接口和版本/幂等会审通过；获写入授权 |
| TECH_DESIGN 第 6/7 节 | 第 4/8 节状态、采集/提醒独立性、纠正/候选/同事务入口 | A0-R01～10 相关决定与 B/C 意见解决；获写入授权 |
| TECH_DESIGN 第 2/3/9/10/12 节 | 第 9/10 节目录/选型责任和 D-03 决定 | 选型与相关 D 决定确认；获写入授权 |
| MVP_SPEC 第 10 节 | A0 评审稿代码位置为“无”；本文与 README 导航；实际文档验证；业务/浏览器/真人未执行；会审/基线汇总未完成 | 记录事实不表示 A0/G0 已通过；获写入授权 |
| TEAM_WORK_PLAN 的 G0/任务登记 | #2 草案已提交评审及实际 PR、差异/会审状态、依赖与受阻操作 | 负责人确认与规划写入授权；不改变职责或里程碑 |
| PRD/MVP_SPEC 业务基线 | 只有裁决涉及用户结果/权利/流程时才提具体修订；本稿不改产品范围 | 负责人明确决定与相应文件写入授权 |

## 14. 交付、验证事实与下一步

- 已完成：远程四个 Issue 的主责/正文核对，定位 A0 #2；最新基线与 Agent 写入规则核对；七项交付要求的评审稿、24 命令/19 组读取/授权与状态矩阵、D-03 12 参数、12 差异及 20 计划用例。
- 写入位置：本文件与 README 导航。没有正式应用源码、DDL、依赖或环境变更。
- 文档验证：PowerShell here-string 的 `node` 内联检查通过：10 份 Markdown、69 处仓库文件/锚点引用、24 个命令及对应 24 行幂等/恢复、19 组读取、12 项 D-03 参数、12 项差异和 20 个计划用例的编号/矩阵完整性通过；Git 差异与未跟踪文件白名单检查只包含本文和 README，受保护基线无差异；两份改动文件的行尾空白、冲突标记和个人目录依赖检查通过。`git diff --check` 通过。首次文档生成命令因脚本字符串中反引号解析失败，未写出文件；改为 JSON 序列化文字后生成成功。首次本地 Git 提交因未配置作者信息失败；后续提交使用本次命令的已登录 GitHub 账号 noreply 署名参数，不修改全局 Git 配置。
- 业务、类型检查、数据库并发/失败注入、模型、执行隔离、浏览器与真人验收：**未执行**；当前仓库没有相应正式工程命令。
- PR/会审：本地分支 codex/a0-contract-review 已准备；向公开仓库推送和创建文档 PR 待公开发布授权；B0/C0 意见未取得，负责人决定与基线写入授权未取得；**待授权维护者汇总**。
- 下一步：B0/C0 以第 12 节五处交界核对；A 汇总差异并修订本文，负责人分别确认契约/参数与基线写入范围；A0 完成条件全部满足后才关闭 Issue。G0 后的正式 A1～A6 不因本文完成自动启动。

### 14.1 文档 PR 说明草稿

拟议标题：docs: submit A0 business contract review proposal (#2)。关联使用 Related #2，不使用自动关闭关键字。

A0 #2 需要将教师/学生/维护者的权限、状态、版本、幂等与事务边界交给 B/C 会审。本次新增独立任务评审稿并补 README 导航，产品/规划/规范/参考文件保持基线；最新 AGENTS 优先于旧 Issue 中的直接基线写入要求。

评审重点为本稿第 2～8 节的授权/四类对象状态/24 命令及恢复/19 读取/候选接纳与纠正事务，第 9～10 节的选型目录与 D-03 参数，第 11～13 节的计划用例、12 差异、五处交界和基线汇总请求。A0-R01～12、A0-D01～12 全部待核对或决定；尤其 ready/active、closed、活动暂停、paused 文件管理、教师样例与正式反馈入口不得默认为冻结。

越权/私有资料拒绝，旧版本和同键异请求冲突，暂停优先，过时模型不能生效，纠正/失效/回执须同事务回滚；保留真实已展示帮助、旧提交、反证与覆盖缺口，不补造证据。文档检查通过；业务、模型、数据库、执行隔离、浏览器与真人验收未执行。B0/C0 会审、负责人决定及授权维护者汇总仍未完成，PR 不关闭 A0/G0。
