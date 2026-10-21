# 生产级灰度发布完整方案（Argo Rollouts + NGINX Ingress + Prometheus）

一套可直接 apply 的生产可用灰度发布配置。流程：改镜像 tag → 5% 流量灰度 → Prometheus
自动分析错误率与 P99 延迟 → 达标自动推进 / 异常自动回滚，全程无需人工介入。

## 架构与组件

```
用户 → NGINX Ingress（流量按权重拆分 95/5）
            ├── stableService → 稳定版 Pod（v1）
            └── canaryService → 灰度 Pod（v2）
                     ↑
        Argo Rollouts 控制器：steps 驱动灰度节奏
                     ↑
        AnalysisTemplate：PromQL 查询错误率 / P99 → 通过/回滚
```

## 前置条件

```bash
# 1. Argo Rollouts 控制器
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# 2. NGINX Ingress Controller（已部署则跳过）——trafficRouting 依赖其 Canary 注解能力

# 3. Prometheus 已采集 ingress 请求指标（nginx_ingress_controller_requests）
#    kubectl argo rollouts CLI（可选，看灰度进度很方便）
brew install argo-rollouts   # 或 krew install rollouts
```

## 文件清单

| 文件 | 作用 |
|---|---|
| `rollout.yaml` | 核心：Rollout CRD（灰度步骤、流量路由、探针、资源限制、安全上下文） |
| `services.yaml` | stable / canary 双 Service（Rollouts 控制器负责改写 selector） |
| `ingress.yaml` | 稳定版 Ingress（Rollouts 自动生成/管理 canary Ingress） |
| `analysis/success-rate.yaml` | 错误率分析（<99% 连续失败即回滚） |
| `analysis/latency-p99.yaml` | P99 延迟分析（>500ms 判定劣化） |
| `pdb.yaml` | PodDisruptionBudget：节点维护时不破坏可用性 |
| `hpa.yaml` | HPA：target 直接指向 Rollout（自动扩缩与灰度共存） |
| `kustomization.yaml` | 一键部署入口 |

## 快速开始

```bash
# 部署（Kustomize 一键）
kubectl apply -k .

# 实时观察灰度进度（推荐）
kubectl argo rollouts get rollout my-app -n prod --watch

# 手动推进（跳过 pause）
kubectl argo rollouts promote my-app -n prod

# 出问题手动中止（自动回滚到 stable）
kubectl argo rollouts abort my-app -n prod

# 回到上一个稳定版本
kubectl argo rollouts undo my-app -n prod
```

## 生产设计要点（面试可讲）

| 设计 | 说明 |
|---|---|
| `maxSurge: 25% / maxUnavailable: 0` | 灰度期间容量不打折 |
| `matchTrafficWeight: true` | 灰度副本数与流量权重匹配，避免多占资源 |
| `scaleDownDelaySeconds: 300` | 切换后旧版本延迟 5 分钟缩容，防止缓存/长连接断 |
| `dynamicStableScale: true` | 灰度时稳定版不提前扩容，节省资源 |
| `abortScaleDownDelaySeconds: 30s` | 中止后 canary 快速回收 |
| `preStop + sleep 15` | 旧 Pod 下线前等 LB 摘流，避免连接被切断 |
| `securityContext runAsNonRoot` | 容器安全基线 |
| `topologySpreadConstraints` | 副本跨 AZ 均匀分布 |
| 双指标 Analysis（错误率 + P99） | 只看错误率会漏掉"没报错但变慢"的劣化 |
| `failureLimit: 1` + `inconclusiveLimit` | 失败即回滚；无结论（数据不足）也回滚，不赌 |
| HPA target 指向 Rollout | 扩缩容与灰度发布共存不冲突 |

## 与 ArgoCD 集成

ArgoCD 的 Application 同步对象从 Deployment 换成本目录的 Rollout 即可——
Git 里的镜像 tag 是唯一可信源，ArgoCD 负责同步，Rollouts 负责安全上线：

```yaml
# Application spec 摘要
source:
  path: manifests/prod
destination:
  namespace: prod
syncPolicy:
  automated: { prune: true, selfHeal: true }
```
