---
title: 如何在 DDD + CQRS 中处理跨聚合的事务
date: 2026-05-28 17:30:57
tags: [Technique]
---

假设系统里有一个模块，用来管理化合物从 hit、lead 到 candidate 的推进。

药化团队合成了一批化合物。生物团队上传活性、选择性、ADME、早期毒理等实验结果。项目负责人看完某个化合物的数据包后，要做一个决策：

- 推进为候选化合物。
- 退回继续优化。
- 暂停该方向。

提交这个决策时，系统通常要做这些事：

- 写入一条候选化合物推进决策。
- 改变化合物当前研发阶段。
- 如果推进成功，创建候选药物档案。
- 关闭当前评审责任。
- 锁定本次使用的实验数据包版本。
- 写入审计事件，说明阶段变化依据哪一次数据评审。

这些写入必须一起成功。不能出现决策已经提交，但化合物阶段没变, 或是候选药物档案已创建，但数据包没有锁定等问题。

最直接的实现，是在 Command Handler 里注入 Prisma，然后开事务：

```ts
class SubmitCompoundPromotionDecisionHandler {
  constructor(private readonly prisma: PrismaClient) {}

  async execute(command: SubmitCompoundPromotionDecision) {
    return this.prisma.$transaction(async (tx) => {
      const decision = await tx.compoundDecision.create({
        data: {
          compoundId: command.compoundId,
          result: command.result,
          reason: command.reason,
          decidedBy: command.actorUserId,
        },
      });

      await tx.compound.update({
        where: { id: command.compoundId },
        data: { stage: command.result === "promote" ? "candidate" : "lead" },
      });

      if (command.result === "promote") {
        await tx.developmentCandidate.create({
          data: { compoundId: command.compoundId },
        });
      }

      await tx.reviewAssignment.update(...);
      await tx.auditEvent.create(...);

      return decision;
    });
  }
}
```

但它把几个层次混在了一起。

Command Handler 本来应该编排业务流程，但现在它编排了数据库表的更新顺序。

化合物阶段变化本来应该由 `Compound` 聚合判断。比如只有 `lead_optimization` 阶段的化合物才能推进到 `candidate`，已经暂停的化合物不能直接推进。现在这条规则可能散落在 `tx.compound.update` 前后的 if 判断里。

事务边界本来属于用例。现在它由具体 ORM 的 `$transaction` 暴露在 Command Handler 里。换 ORM、拆 outbox、改审计事件写法，Command Handler 都要动。

---

如果把这段逻辑挪到 repository 里：

```ts
compoundRepository.submitPromotionDecision(command);
```

这看起来把 Prisma 从 Command Handler 里拿掉了。但如果 `compoundRepository` 内部仍然创建决策、更新化合物、创建候选药物档案、关闭责任、写审计事件，那只是把混乱换了一个地方，因为 Repository 的核心职责应该只是持久化聚合，例如：

```ts
compoundRepository.findById(id);
compoundRepository.save(compound);
compoundDecisionRepository.save(decision);
developmentCandidateRepository.save(candidate);
```

如果一个 `CompoundRepository` 开始保存 `CompoundDecision`、`DevelopmentCandidate`、`ReviewAssignment`、`AuditEvent`，它就不再只是 Compound 的 repository。它变成了一个跨模块事务服务，只是名字还叫 repository。

---

所以这里我的解决方案之一是在 application 层定义一个 Unit of Work port，它表达的是一个具体业务动作：提交候选化合物推进决策。

```ts
export interface SubmitCompoundPromotionDecisionUnitOfWork {
  submit(input: {
    compoundId: string;
    dataPackageId: string;
    result: "promote" | "optimize" | "pause";
    reason?: string;
    actorUserId: string;
  }): Promise<CompoundPromotionDecisionDto>;
}
```

Command Handler 依赖这个端口，而不是依赖 Prisma。

```ts
class SubmitCompoundPromotionDecisionHandler {
  constructor(
    private readonly unitOfWork: SubmitCompoundPromotionDecisionUnitOfWork,
  ) {}

  async execute(command: SubmitCompoundPromotionDecision) {
    return this.unitOfWork.submit({
      compoundId: command.compoundId,
      dataPackageId: command.dataPackageId,
      result: command.result,
      reason: command.reason,
      actorUserId: command.actorUserId,
    });
  }
}
```

这段代码没有事务细节。它只转交命令意图。

具体事务放在 infrastructure：

```ts
class PrismaSubmitCompoundPromotionDecisionUnitOfWork
  implements SubmitCompoundPromotionDecisionUnitOfWork
{
  constructor(private readonly prisma: PrismaClient) {}

  async submit(input: SubmitCompoundPromotionDecisionInput) {
    return this.prisma.$transaction(async (tx) => {
      // 在这里组合 tx-bound store / repository
    });
  }
}
```

这样就把“业务事务边界”和“命令入口”分开，Command Handler 不需要知道用 Prisma 还是别的 ORM。它也不需要知道事务里要调用哪些表。它只知道这个命令由一个原子用例完成。

但聚合仍然要负责状态规则，Unit of Work 不能变成新的数据库脚本，在事务内部，仍然应该先读取聚合，让聚合执行状态变化，再保存聚合。

例如 `Compound` 聚合可以这样表达规则：

```ts
class Compound {
  promoteToCandidate(input: {
    dataPackageId: string;
    actorUserId: string;
  }) {
    if (this.stage !== "lead_optimization") {
      throw new DomainError("只有先导优化阶段的化合物才能推进为候选化合物");
    }

    if (this.lockedDataPackageId !== null) {
      throw new DomainError("已锁定数据包的化合物不能重复推进");
    }

    this.stage = "candidate";
    this.lockedDataPackageId = input.dataPackageId;
    this.updatedByUserId = input.actorUserId;
  }

  returnToOptimization(input: { reason: string; actorUserId: string }) {
    if (this.stage !== "lead_optimization") {
      throw new DomainError("只有评审中的化合物才能退回优化");
    }

    this.stage = "lead_optimization";
    this.optimizationReason = input.reason;
    this.updatedByUserId = input.actorUserId;
  }
}
```

Unit of Work 在事务里调用这些方法：

```ts
await prisma.$transaction(async (tx) => {
  const compoundStore = new PrismaCompoundStore(tx);
  const compound = await compoundStore.findById(input.compoundId);

  if (input.result === "promote") {
    compound.promoteToCandidate({
      dataPackageId: input.dataPackageId,
      actorUserId: input.actorUserId,
    });
  }

  if (input.result === "optimize") {
    compound.returnToOptimization({
      reason: input.reason ?? "需要补充结构优化",
      actorUserId: input.actorUserId,
    });
  }

  await compoundStore.save(compound);
  await decisionStore.save(decision);
  await candidateStore.createIfPromoted(...);
  await assignmentStore.closeCurrent(...);
  await auditStore.append(...);
});
```

而为了让事务里的代码复用聚合持久化逻辑，我们需要一种 tx-bound repository 或 store。

最粗暴的写法是在领域 repository interface 里加一个可选事务参数：

```ts
interface CompoundRepository {
  findById(id: string, tx?: Prisma.TransactionClient): Promise<Compound | null>;
  save(compound: Compound, tx?: Prisma.TransactionClient): Promise<void>;
}
```

但它又把 Prisma 带进了领域层。就算把 `Prisma.TransactionClient` 包成 `TransactionClient` 类型别名，领域接口仍然在为基础设施妥协。

我更倾向于把 tx-bound 能力留在 infrastructure，领域层只定义干净的 repository port：

```ts
export interface CompoundRepository {
  findById(id: string): Promise<Compound | null>;
  save(compound: Compound): Promise<void>;
}
```

infrastructure 层拆一个 store，接收最小持久化客户端：

```ts
type CompoundStoreClient = {
  compound: {
    findUnique(args: unknown): Promise<unknown>;
    update(args: unknown): Promise<unknown>;
  };
};

class PrismaCompoundStore implements CompoundRepository {
  constructor(private readonly client: CompoundStoreClient) {}

  async findById(id: string) {
    const record = await this.client.compound.findUnique(...);
    return Compound.restore(record);
  }

  async save(compound: Compound) {
    const snapshot = compound.toSnapshot();
    await this.client.compound.update(...);
  }
}
```

普通 repository 可以继承这个 store，传完整 Prisma client：

```ts
class PrismaCompoundRepository extends PrismaCompoundStore {
  constructor(prisma: PrismaClient) {
    super(prisma);
  }
}
```

Unit of Work 在事务里传 tx：

```ts
await prisma.$transaction(async (tx) => {
  const compoundStore = new PrismaCompoundStore(tx);
  const compound = await compoundStore.findById(input.compoundId);
  // ...
  await compoundStore.save(compound);
});
```
