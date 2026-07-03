# ec2-auto-attach-role

> 中文说明见 [README-CN.md](./README-CN.md)

A self-contained CloudFormation stack that **automatically attaches an IAM
instance profile to EC2 instances** as they enter the `running` state.

It uses an EventBridge → Lambda → associate-instance-profile pattern with the
following behavior:

- If an instance carries a **tag** (default key `IAMRole`), the tag's value is
  treated as the **role name** to attach.
- If the tag is **absent**, a **default role** (supplied as a stack parameter)
  is attached instead.
- Roles are **only associated, never created** — both the tagged role and the
  default role must already exist.
- If an instance **already has an instance profile at launch, it is left
  untouched** (no overwrite).

## How it works

```
EC2 instance -> running
        │
        ▼
EventBridge rule (state == running)
        │
        ▼
Lambda (AttachRole)
        │
        ├─ instance already has a profile?  -> skip
        ├─ read tag <TagKey>
        │     ├─ present + role exists      -> use tagged role
        │     └─ absent / role missing      -> use DefaultRoleName
        ├─ ensure an instance profile named after the role exists
        │  (create the profile container + add the role if needed)
        └─ associate the instance profile with the instance
```

> **Instance profile vs role:** EC2 can only receive a *role* through an
> *instance profile* container. The Lambda creates/uses an instance profile
> **named identically to the role** and adds the (pre-existing) role to it. It
> never creates the IAM role itself.

## Parameters

| Parameter         | Default   | Description                                                                                 |
|-------------------|-----------|---------------------------------------------------------------------------------------------|
| `TagKey`          | `IAMRole` | EC2 tag key whose value is the role name to attach. Override at deploy time if desired.      |
| `DefaultRoleName` | *(none)*  | Name of an **existing** IAM role used when an instance has no matching tag. Required.        |

## Deploy

```bash
aws cloudformation deploy \
  --template-file ec2-auto-attach-role.yaml \
  --stack-name ec2-auto-attach-role \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
      DefaultRoleName=my-default-ec2-role \
      TagKey=IAMRole
```

`CAPABILITY_IAM` is required because the stack creates the Lambda execution
role.

## Usage

- **Use a specific role for an instance** — tag it:

  ```
  Key:   IAMRole   (or whatever you set TagKey to)
  Value: my-app-role
  ```

  `my-app-role` must already exist in the account.

- **Use the default role** — launch the instance with no `IAMRole` tag. The
  `DefaultRoleName` role is attached.

- **Bring your own profile** — if you launch the instance already specifying an
  instance profile, this stack leaves it alone.

## Notes & caveats

- Only acts on instances that reach `running` **after** the stack is deployed.
- The tag is evaluated at the moment the instance goes `running`. Tags must be
  set at launch (e.g. via RunInstances `TagSpecifications`) to be seen on the
  first pass. The rule fires on every `running` transition (including
  stop/start), so a later-added tag is picked up on the next start *only if the
  instance still has no profile*.
- If the tagged role does not exist, the Lambda falls back to the default role
  and logs an error.
- Idempotent: re-invocations for an instance that already has a profile are
  skipped.

## Files

- `ec2-auto-attach-role.yaml` — the CloudFormation template (Lambda code is
  inlined, so no S3 artifact is needed).
- `README-CN.md` — Chinese documentation.
