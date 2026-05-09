---
name: interactive-code-dev
description: |
  根据技术方案文档实现后端代码的技能。适用于用户提供了技术设计文档(接口文档、流程图、存储设计等)，
  需要落地为可运行的Java/SpringBoot项目代码的场景。
  当用户提到"按技术方案开发"、"根据文档实现代码"、"技术设计落地"、"开始编码"、
  或者提供了一个技术文档/接口设计/流程图要求实现时，使用此技能。
  本技能记录了互动业务后端开发的所有技术约定和规范，确保生成的代码符合项目架构标准。
---

# 互动代码开发技能

根据技术方案文档，按约定规范实现后端代码。

## 使用时机

当用户提供了一份技术方案文档（包含接口定义、流程图、存储设计等），需要将其落地为代码时启用。
本技能的核心价值是：**将业务开发约定固化，确保每次生成的代码都符合项目架构规范**。

## 第一步：确认开发环境

在理解技术方案之前，必须先确认以下信息：

### 1.1 确认代码仓库

根据技术方案中涉及的模块/服务，确认需要改动的代码仓库路径。如果用户未明确指定：

1. **先尝试从技术方案推断**：根据接口路径、服务名等推断可能涉及的仓库
2. **如果无法确定，主动询问用户**：
   - "这个需求涉及哪些仓库？"
   - "主业务逻辑在哪个项目里？"
3. **常见仓库映射**（供参考，不假设）：
   - `D:/java-project/hd_api` — 协议定义（Proto）
   - `D:/java-project/hd-recommend` — 推荐服务
   - `D:/java-project/hd-room` — 房间服务
   - 其他仓库需用户确认路径

### 1.2 确认开发分支名

询问用户本次开发的分支名。分支命名规范：

- 功能开发：`feat-{简短描述}`，如 `feat-push-room-recommend`
- Bug修复：`fix-{简短描述}`，如 `fix-gift-count-overflow`

如果用户未指定，根据需求名称提议一个分支名供确认。

### 1.3 创建 Git Worktree（如需要）

如果用户要求使用 git worktree 开发，按以下约定创建：

**核心规则：worktree 放在当前需求工作目录下，一个项目一个文件夹。**

```bash
# 1. 到项目主仓库，基于最新 master 创建特性分支的 worktree
cd <项目主仓库路径>
git checkout master && git pull
git worktree add "<当前需求目录>/<项目名>" -b <分支名>

# 示例：
cd D:/java-project/hd-recommend
git checkout master && git pull
git worktree add "D:/features/push_online_room/hd-recommend" -b feat-push-room-recommend
```

**目录结构**：
```
D:/features/push_online_room/     ← 需求工作目录（当前工作目录）
├── docs/                         ← 需求文档、技术方案
├── hd-recommend/                 ← 项目A的worktree
├── hd-room/                      ← 项目B的worktree（如需要）
└── ...
```

**为什么不放在项目的 `.claude/worktrees/` 下**：
- Git 限制同一分支不能被两个 worktree checkout，放在项目内会导致 IDEA 无法切换到该分支
- 放在需求目录下结构清晰，IDEA 可直接打开 worktree 文件夹进行开发

后续所有代码操作都在 worktree 目录下进行，使用绝对路径。

## 第二步：理解技术方案

仔细阅读用户提供的技术方案文档，提取以下关键信息：

1. **接口列表**：HTTP接口路径、方法、请求/响应字段
2. **核心流程**：业务流程图、时序图中的每一步
3. **存储设计**：新建表结构、字段定义、索引
4. **消息/广播**：Kafka事件、单播/广播消息定义
5. **定时任务**：cron表达式、扫描逻辑
6. **外部依赖**：需要调用的RPC/HTTP服务

提取后，生成一份**开发任务清单**供用户确认，按依赖关系排序。

## 第三步：项目探索（必须）

在动手写代码前，**必须先探索项目结构**，理解现有模式：

### 必须探索的内容

1. **模块结构**：Maven多模块划分（common/protocol/persist/service/app）
2. **现有Handler**：找同模块的Handler，理解参数校验+流程编排模式
3. **现有Service**：理解数据访问层的命名和异常处理约定
4. **现有Controller**：理解路由定义、参数绑定、返回值构造
5. **现有Mapper**：理解MyBatis的使用方式（XML vs 注解）
6. **现有Kafka Listener**：理解消费模式（containerFactory、groupId、异常处理）
7. **现有定时任务**：理解调度方式（@Scheduled / @FunJob）
8. **现有YRPC/Thrift Client**：理解服务间调用模式

### 注意事项

- 不要假设任何文件路径或类名——全部通过搜索确认
- 现有代码中可能有工具类、常量、配置可供复用，避免重复造轮子
- 如果探索发现技术方案与现有架构有冲突，先和用户确认

## 第四步：整理开发任务清单

按以下格式整理任务清单供用户Review：

```
### Task N: 任务名称 (模块)
- N.1 具体子任务
- N.2 具体子任务

**开发顺序**: Task 0 → Task 1 → ... → Task N
**依赖说明**: 哪些任务可以并行，哪些有先后依赖
```

**等待用户确认后再开始编码。**

## 开发节奏：逐任务提交

开发过程中，**每完成一个 Task（或有意义的子任务），必须执行一次 git commit**：

- commit 信息格式：`feat(module): 简要描述`，如 `feat(recommend): 新增在线房间推荐Handler`
- 一个 Task 涉及多个子任务时，子任务之间也可以分别 commit，保持每次 commit 改动小且含义清晰
-  commit 前用 `git diff` 快速检查改动是否符合预期，避免提交无关文件
- 所有 Task 完成后，统一 push 并创建 MR

**禁止：** 不允许积累多个 Task 一次性 commit，每个 Task 必须有独立的提交记录。

## 第五步：按约定编码

以下是编码时必须遵守的约定。这些约定来自实际项目经验，违反会导致代码无法编译或不一致。

---

### 5.1 协议定义（Proto）

**位置**: `hd_api/client/hd_client_{module}_api/src/main/proto/`

**三个文件分工**:
- `{module}_common.proto` — 公共结构体（复用的数据模型）
- `{module}_web.proto` — HTTP接口定义（带 `http.url`、`http.method` 注解）
- `{module}_broadcast.proto` — 广播/单播消息（带 `uri.max`、`uri.min` 注解）

**命名约定**:
- 消息名用 PascalCase：`GetNewbieGuideReq`、`HostReturnGiftResp`
- 字段名用 snake_case：`red_heart_count`、`seq_id`
- HTTP路径模式：`/api/{module}/{action}`
- 广播URI范围：先确认已分配的 min 范围，递增分配

**部署流程**:
```bash
cd hd_api
git pull
mvn clean install deploy -am -pl client/hd_client_{module}_api -DskipTests=true
```

### 5.2 MyBatis持久层

**代码生成**:
```bash
cd {module}-persist
mybatis gen -p {base.package} -o . --only mapper,xml "CREATE TABLE ..."
```
- SQL中不要带库名前缀（`hd_free_gift.table_name`），否则解析错误
- 生成后检查类名是否正确

**ExtMapper规范**:
- ExtMapper放在 `mapper/ext/` 子包下，继承基础Mapper
- 优先用注解（`@Select`、`@Insert`、`@Update`）而非XML
- 动态SQL用 `<script>` + `<foreach>`
- SQL中 `<` 符号必须用 `&lt;` 转义

**Model规范**:
- 使用Lombok `@Data`
- 字段类型映射：bigint→Long, int→Integer, varchar→String, datetime→Date

### 5.3 协议客户端

**已有Client优先**:
- `PropsClient` → `TTurnoverService.Iface`（营收道具服务）
- `GiftBagClient` → `TGiftBagService.Iface`（道具背包查询）
- 通过 `@Delegate` 注解自动暴露接口方法

**新增Client模式**:
```java
@Component
public class XxxClient {
    @Delegate
    @Reference(protocol = "attach_nythrift_compact", owner = "${s2sname.xxx.yyy}")
    public TXxxService.Iface xxxService;
}
```

**Service层包装**: 对thrift调用做异常处理、重试、日志记录，返回布尔值或结果对象。

### 5.4 业务层（Service + Handler 分层）

**Handler** — 流程编排，参数校验:
```java
@Component
@Slf4j
@RequiredArgsConstructor
public class XxxHandler {
    private final XxxService xxxService;
    private final BroadcastMobService broadcastMobService;

    public Resp doXxx(long uid, Req req) {
        // 1. 参数校验
        // 2. 调用Service获取数据
        // 3. 组装返回值
    }
}
```

**Service** — 数据处理，DB操作:
```java
@Service
@Slf4j
@RequiredArgsConstructor
public class XxxService {
    private final XxxExtMapper xxxMapper;

    public Xxx findByXxx(long xxx) {
        try {
            return xxxMapper.selectByXxx(xxx);
        } catch (Exception e) {
            log.error("findByXxx failed, xxx:{}", xxx, e);
            return null;
        }
    }
}
```

**关键约定**:
- Handler用 `@Component`，Service用 `@Service`
- 全部用 `@RequiredArgsConstructor` 构造器注入
- Service方法统一 try-catch + log.error + 返回null/false
- 业务参数用 `@Value("${key:default}")` 定义，便于Apollo配置

### 5.5 HTTP接口

**Controller规范**:
```java
@RestController
@RequiredArgsConstructor
public class XxxController extends BaseController {
    private final XxxHandler xxxHandler;

    @GetMapping("/api/{module}/{action}")
    public Resp action(Req req) {
        var uid = getLoginUid();
        AssertUtil.isTrue(uid > 0, "login is invalid");
        AssertUtil.isTrue(req.getSid() > 0, "sid is invalid");
        return xxxHandler.doXxx(uid, req);
    }
}
```

**关键约定**:
- 继承 `BaseController`，用 `getLoginUid()` 获取登录用户
- 参数校验用 `AssertUtil.isTrue()`
- 返回Protobuf生成的Response对象
- `Result` 用 `Common.Result.newBuilder().setCode(0).setMessage("ok").build()`

### 5.6 Kafka消费

**扩展现有Listener，不新建同Topic消费者**:
```java
// 在现有Handler中注入新的处理Handler
private final NewHandler newHandler;

public void onGiftUsedEvent(Event event) {
    // 在过滤之前调用新Handler（避免新propId被过滤掉）
    try {
        newHandler.onEvent(event);
    } catch (Exception e) {
        log.error("newHandler.onEvent failed", e);
    }
    // ... 原有逻辑
}
```

**消费模式**:
- `containerFactory`、`topics`、`groupId` 从现有配置复用
- `@Profile("!dev")` 排除开发环境
- 异常处理：catch后log.warn + throw（让Kafka重试）

### 5.7 广播/单播

```java
// 单播 - 发送给特定用户
broadcastMobService.unicast(uid, protobufMessage);

// 频道广播 - 发送给房间
broadcastMobService.broadcast(sid, ssid, protobufMessage);
```

**Protobuf消息构造**:
```java
var notice = BroadcastPb.XxxNotice.newBuilder()
    .setUid(uid)
    .setSid(sid)
    .build();
broadcastMobService.unicast(uid, notice);
```

### 5.8 定时任务

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class XxxJob {
    private final XxxHandler xxxHandler;

    @Scheduled(fixedRate = 300000) // 5分钟
    public void doXxx() {
        try {
            xxxHandler.doXxx();
        } catch (Exception e) {
            log.error("doXxx failed", e);
        }
    }
}
```

### 5.9 用户信息与在线状态

```java
// 获取用户信息
hdWebdbService.getUserInfo(uid);
hdWebdbService.batchGetUserInfo(uids, hostId, withNickExt);

// 检查用户在线状态
var userChannels = hdCulService.queryUserChannelBo(List.of(uid));
var channel = userChannels.getUserChannel(uid);
boolean inRoom = channel != null
    && channel.getTopsid() == sid
    && channel.getSubsid() == ssid;
```

---

## 第六步：自检清单

编码完成后，逐项检查：

- [ ] Proto文件语法正确，max/min值不冲突，deploy成功
- [ ] mybatis gen生成的Model/Mapper/XML无库名前缀问题
- [ ] ExtMapper在 `mapper/ext/` 子包下，`@Mapper` 注解
- [ ] Handler用 `@Component`，Service用 `@Service`
- [ ] 所有thrift调用有异常处理和日志
- [ ] Kafka消费不新建同Topic同groupId的消费者
- [ ] Controller继承BaseController，参数用AssertUtil校验
- [ ] 广播消息字段与Proto定义一致
- [ ] 定时任务有try-catch保护
- [ ] @Value配置项有合理默认值

## 常见陷阱

1. **mybatis gen库名前缀**：SQL中写 `CREATE TABLE db.table` 会导致生成类名为 `DbTable` 而非 `Table`
2. **Kafka消费同Topic**：同一Topic+GroupId不能有多个消费者，会导致分区分配问题
3. **广播URI冲突**：新增广播消息前必须检查已有的 max/min 范围
4. **TyYInfo构造**：`new TYyInfo(uid, yyId, nickName)`，yyId可传0
5. **SQL中的小于号**：MyBatis XML中 `<` 必须转义为 `&lt;`
6. **Proto字段类型**：int64→long, int32→int, string→String, bool→boolean
7. **Handler调用时机**：新增的Kafka事件处理必须在现有propId过滤之前调用
