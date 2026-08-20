# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

管理员删除一个已被历史订单选用的护理服务时，数据库正确拒绝了操作，但接口只返回 500，页面看起来像服务故障。此次只排查而不改代码，生产代码、测试和配置都不作调整；请追踪外键错误经过删除逻辑和统一映射时为何失去冲突身份，并验证失败后服务项目仍然存在。

## 含 Bug 版本

- 仓库：zhanglei10281852-gif/pet-foster-task-20
- 仓库地址：https://github.com/zhanglei10281852-gif/pet-foster-task-20.git
- parent SHA：74ccfd7557846cb59ed99940aa552b9b035993a1

## 复现步骤

```bash
git clone -- https://github.com/zhanglei10281852-gif/pet-foster-task-20.git bug-repro
cd bug-repro
git checkout --detach 74ccfd7557846cb59ed99940aa552b9b035993a1
go test ./internal/pet -run ^TestAnnotationUsedServiceDeleteReturnsConflict$ -count=1
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/pet -run ^TestAnnotationUsedServiceDeleteReturnsConflict$ -count=1
--- FAIL: TestAnnotationUsedServiceDeleteReturnsConflict (0.30s)
    annotation_pet_behavior_test.go:390: delete service error=delete catalog resource: constraint failed: FOREIGN KEY constraint failed (787)
FAIL
FAIL	github.com/zhanglei10281852-gif/pet-foster-go/internal/pet	0.304s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/pet -run ^TestAnnotationUsedServiceDeleteReturnsConflict$ -count=1
--- FAIL: TestAnnotationUsedServiceDeleteReturnsConflict (1.07s)
    annotation_pet_behavior_test.go:390: delete service error=delete catalog resource: constraint failed: FOREIGN KEY constraint failed (787)
FAIL
FAIL	github.com/zhanglei10281852-gif/pet-foster-go/internal/pet	1.301s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

诊断结论须定位 internal/pet/domain.go 的 catalogDeleteError，说明其使用 %v 丢失底层错误链后，Service.DeleteServiceItem 无法让 writeAPIError 识别 ErrConflict，最终把外键拒绝映射为 500；证据需包含删除响应及数据库中服务项目仍存在的状态。调查结束时不得有生产代码、测试或配置改动。
