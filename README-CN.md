# ec2-auto-attach-role（中文说明）

一个自包含的 CloudFormation 堆栈，在 EC2 实例进入 `running` 状态时**自动为其挂载 IAM 实例配置文件（instance profile）**。

它采用 EventBridge → Lambda → 关联 instance profile 的模式，核心特性：

- 如果实例带有指定 **tag**（默认 key 为 `IAMRole`），则用该 tag 的**值**作为要挂载的 **role 名**。
- 如果实例**没有**该 tag，则挂载堆栈参数指定的**默认 role**。
- Role **只关联，绝不创建** —— 无论 tag 指定的 role 还是默认 role，都必须已经存在。
- 如果实例在启动时**已经带有 instance profile，则保持不动**（不覆盖）。

## 工作原理

```
EC2 实例 -> running
        │
        ▼
EventBridge 规则（state == running）
        │
        ▼
Lambda（AttachRole）
        │
        ├─ 实例是否已有 profile？        -> 有则跳过
        ├─ 读取 tag <TagKey>
        │     ├─ 存在且 role 存在        -> 使用 tag 指定的 role
        │     └─ 不存在 / role 不存在    -> 使用 DefaultRoleName
        ├─ 确保存在一个以该 role 命名的 instance profile
        │  （必要时创建 profile 容器并把 role 加进去）
        └─ 将 instance profile 关联到实例
```

> **Instance profile 与 role 的区别：** EC2 只能通过 *instance profile*（容器）来接收一个 *role*。本 Lambda 会创建/复用一个**与 role 同名**的 instance profile，并把（已存在的）role 加进去。它**不会创建 IAM role 本身**。

## 参数

| 参数              | 默认值    | 说明                                                                       |
|-------------------|-----------|----------------------------------------------------------------------------|
| `TagKey`          | `IAMRole` | 用于查找 role 名的 EC2 tag key。tag 的值即要挂载的 role 名。部署时可覆盖。   |
| `DefaultRoleName` | *(无)*    | 实例无匹配 tag 时使用的**已存在** IAM role 名。必填。                        |

## 部署

```bash
aws cloudformation deploy \
  --template-file ec2-auto-attach-role.yaml \
  --stack-name ec2-auto-attach-role \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
      DefaultRoleName=my-default-ec2-role \
      TagKey=IAMRole \
  --region <目标region>
```

> 由于堆栈会创建 Lambda 执行角色，必须带 `CAPABILITY_IAM`。

## 使用方式

- **为某台实例指定专属 role** —— 给它打 tag：

  ```
  Key:   IAMRole   （或你设置的 TagKey）
  Value: my-app-role
  ```

  `my-app-role` 必须已存在于账号中。

- **使用默认 role** —— 启动实例时不带 `IAMRole` tag，即自动挂载 `DefaultRoleName` 指定的 role。

- **自带 profile** —— 如果你启动实例时已经指定了 instance profile，本堆栈会保持不动。

## 注意事项与限制

- 仅对堆栈部署**之后**进入 `running` 的实例生效。
- tag 在实例进入 `running` 的那一刻被读取。tag 需要在启动时设置（例如通过 RunInstances 的 `TagSpecifications`）才能在第一次就被看到。规则在**每次** `running` 状态切换时触发（含 stop/start），因此后加的 tag 只有在实例下次启动、**且此时仍无 profile** 的情况下才会被采用。
- 如果 tag 指定的 role 不存在，Lambda 会回退到默认 role 并记录错误日志。
- 幂等：对已有 profile 的实例重复触发会被跳过。
- **同名 profile 提醒**：Lambda 假设 instance profile 与 role 同名。控制台/CFN 创建的 role 会自动带同名 profile，因此通常没问题。仅当你手工用 CLI 制造出"profile 名与 role 名故意错开且互相撞名"的非常规配置时，才可能因"一个 profile 最多只能装 1 个 role"的限制而报错。
- **IAM 最终一致性**：新建的 instance profile 不会立刻对 EC2 可见。Lambda 会先用 `instance_profile_exists` waiter 等 IAM 确认，再对 `associate_iam_instance_profile` 做指数退避重试（1/2/4/8s，共 5 次），以消除 `InvalidParameterValue: Invalid IAM Instance Profile ARN` 竞态。即使首次关联偶发失败，EventBridge 的异步重投递也会兜底。

## 文件

- `ec2-auto-attach-role.yaml` —— CloudFormation 模板（Lambda 代码已内联，无需 S3 制品）。
- `README.md` —— 英文说明。
- `README-CN.md` —— 本文件。
