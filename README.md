# 云原生DevSecOps安全流水线系统 🛡️🤖💥
> 代码提交全程安检，漏洞无处可逃 🕵️‍♂️😼
# 流水线总流程图

```
代码提交(Push/PR)
    ↓
Gitleaks Secret密钥扫描
    ↓
Bandit Python SAST扫描
    ↓
Semgrep 通用SAST扫描（自定义规则）
    ↓
Trivy SCA 依赖包漏洞扫描
    ↓
构建Docker镜像 → Trivy 容器镜像扫描
    ↓
推送镜像至Harbor仓库
    ↓
K8s资源提交 → Gatekeeper准入策略校验
    ↓
Pod启动运行
    ↓
OWASP ZAP DAST动态扫描(测试环境)
    ↓
Falco 容器运行时行为监控(生产环境持续告警)
```

1. Gitleaks Secret 密钥扫描 🔑👀（抓硬编码密钥）
2. Bandit Python SAST 扫描 🐍🤕（揪 Python 漏洞）
3. Semgrep 通用 SAST 扫描 🧐⚡自定义规则
4. Trivy SCA 依赖扫描 📦💣（第三方包炸弹）
5. Trivy 容器镜像扫描 🐳🔍
6. Gatekeeper K8s 准入控制 ☸️🚧（拦住非法资源）
7. OWASP ZAP DAST 动态扫描 🕸️👾（黑盒爆破）
8. Falco 容器运行时监控 🚨👁️（实时盯容器）

##  1. Secret 密钥扫描 (Gitleaks)

### 部署

```
# 二进制安装（Linux）
curl -s https://api.github.com/repos/gitleaks/gitleaks/releases/latest | grep "linux_amd64.tar.gz"
# 或者使用docker
docker pull gitleaks/gitleaks
```

### 使用命令

- 本地扫描当前仓库

```
gitleaks detect -s .
```

- JSON 格式输出报告

```
gitleaks detect -s . --report-format json --report-path gitleaks-report.json
```

## GitHub Action 自动触发（放到仓库 `.github/workflows/gitleaks.yml`）

```
name: Gitleaks Secret扫描
on: [push, pull_request]
jobs:
  gitleaks-scan:
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v4
      - name: Gitleaks密钥扫描
        uses: gitleaks/gitleaks-action@v2
        env:
          GITLEAKS_CONFIG: .gitleaks.toml
      - name: 上传扫描报告
        uses: actions/upload-artifact@v4
        with:
          name: gitleaks-report
          path: gitleaks-report.json
```
# 2.静态代码扫描(SAST)
## -首推Semgrep(多语言支持+自定义规则）
### 部署
```
 pipx install semgrep
```
### 使用命令
- 使用官方Python规则集扫描当前目录
```
 semgrep scan --config=p/python .
```
- 使用本地自定义规则扫描
```
semgrep scan --config=rules/no-os-system.yml src/
```
- JSON格式输出报告
```
semgrep scan --config=p/python --json > semgrep-report.json
```
## 自定义规则示例（`rules/no-os-system.yml`）
```yaml
> rules:  
> - id: forbid-os-system  
>   pattern: os.system(...)  
>   message: 禁止使用os.system，存在命令注入漏洞风险  
>   languages: [python]  
>   severity: ERROR
```
## GitHub Action 自动触发（放到仓库 `.github/workflows/semgrep.yml`）
```yaml
name: Semgrep静态代码扫描
on: [push, pull_request]
jobs:
  semgrep-scan:
    runs-on: ubuntu-latest
    container:
      image: semgrep/semgrep
    steps:
      - name: 检出代码
        uses: actions/checkout@v4
      - name: 执行Semgrep扫描
        run: semgrep scan --config=p/python --json > semgrep-report.json
      - name: 上传扫描报告
        uses: actions/upload-artifact@v4
        with:
          name: semgrep-report
          path: semgrep-report.json
```
# 3. 软件成分分析 SCA (Trivy)

### 部署

```
# Linux安装Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
```

### 使用命令

- 扫描项目依赖（SCA）

```
trivy fs .
```

- JSON 格式输出报告

```
trivy fs -f json -o trivy-sca-report.json .
```

## GitHub Action 自动触发（放到仓库 `.github/workflows/trivy-sca.yml`）

```
name: Trivy SCA依赖漏洞扫描
on: [push, pull_request]
jobs:
  trivy-sca:
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v4
      - name: 安装Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: .
          format: json
          output: trivy-sca-report.json
      - name: 上传扫描报告
        uses: actions/upload-artifact@v4
        with:
          name: trivy-sca-report
          path: trivy-sca-report.json
```
# 4. 容器镜像漏洞扫描 (Trivy)

### 部署

```
# Linux安装Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
```

### 使用命令

- 扫描本地镜像

```
trivy image myapp:v1
```

- JSON 格式输出报告

```
trivy image -f json -o trivy-image-report.json myapp:v1
```

## GitHub Action 自动触发（放到仓库 `.github/workflows/trivy-image.yml`）

```
name: Trivy容器镜像扫描
on: [push, pull_request]
jobs:
  trivy-image:
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v4
      - name: 构建镜像
        run: docker build -t myapp:v1 .
      - name: 镜像漏洞扫描
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:v1
          format: json
          output: trivy-image-report.json
      - name: 上传扫描报告
        uses: actions/upload-artifact@v4
        with:
          name: trivy-image-report
          path: trivy-image-report.json
```
# 5. K8s 准入控制 (Gatekeeper)

### 部署

```
# 安装Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.15/deploy/gatekeeper.yaml
```

### 使用命令

- 查看约束模板

```
kubectl get constrainttemplates
```

- 查看约束策略执行状态

```
kubectl get constraints
```

## 约束策略示例 (`constraints/require-labels.yaml`)

```
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: deployment-must-have-labels
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    labels: ["app"]
```

## GitHub Action 自动触发（Gatekeeper 策略校验，`.github/workflows/gatekeeper.yml`）

```
name: Gatekeeper OPA策略校验
on: [push, pull_request]
jobs:
  gatekeeper-validate:
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v4
      - name: 安装conftest校验OPA策略
        run: |
          curl -Lo conftest https://github.com/open-policy-agent/conftest/releases/download/v0.50.0/conftest_0.50.0_Linux_x86_64
          chmod +x conftest
      - name: 校验Gatekeeper策略
        run: ./conftest test constraints/
```


# 6. 动态应用安全测试 DAST (OWASP ZAP)

### 部署

```
docker pull owasp/zap2docker-stable
```

### 使用命令

- 基线扫描目标 Web 服务

```
docker run --rm owasp/zap2docker-stable zap-baseline.py -t http://127.0.0.1:8000
```

- JSON 格式输出报告

```
docker run --rm -v $(pwd):/zap/wrk owasp/zap2docker-stable zap-baseline.py -t http://127.0.0.1:8000 -f json -o zap-report.json
```

## GitHub Action 自动触发（放到仓库 `.github/workflows/zap-dast.yml`）

```
name: OWASP ZAP DAST动态扫描
on: [push, pull_request]
jobs:
  zap-scan:
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v4
      - name: 启动应用
        run: nohup python app.py &
      - name: ZAP基线扫描
        uses: zaproxy/action-baseline@v0.13.0
        with:
          target: "http://localhost:8000"
          report_format: json
          report_name: zap-report.json
      - name: 上传扫描报告
        uses: actions/upload-artifact@v4
        with:
          name: zap-report
          path: zap-report.json
```
# 7. 容器运行时安全 (Falco)

### 部署

```
# Helm安装Falco
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
helm install falco falcosecurity/falco
```

### 使用命令

- 查看 Falco 告警日志

```
kubectl logs -l app.kubernetes.io/name=falco
```

- 测试触发告警规则

```
kubectl exec -it <pod-name> -- touch /etc/shadow
```

## 自定义规则示例 (`falco/rules.yaml`)

```
- rule: 容器内读取敏感shadow文件
  desc: 检测容器内读取/etc/shadow行为
  condition: open_read and filename=/etc/shadow
  output: "敏感文件被读取，容器ID=%container_id，进程=%proc.name"
  priority: ERROR
```

## GitHub Action 自动触发（Falco 规则校验，`.github/workflows/falco.yml`）

```
name: Falco规则校验
on: [push, pull_request]
jobs:
  falco-validate:
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v4
      - name: 安装falcoctl
        run: curl -fsSL https://download.falco.org/packages/bin/x86_64/falcoctl/latest/falcoctl -o /usr/local/bin/falcoctl && chmod +x /usr/local/bin/falcoctl
      - name: 校验Falco规则
        run: falcoctl verify rules falco/rules.yaml
```

