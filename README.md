# AOS‑KERNEL V2.0
> Heterogeneous Dual‑Engine AI Operating System Kernel
> 异构双引擎AI内核：决策与执行分离，零信任安全 + WORM不可篡改审计链

## 📖 项目简介
传统AI Agent普遍采用**单模型全包模式**：同一个模型既要做权限风控裁决，又要完成长文本生成、工具调用。容易出现幻觉越权、算力浪费、行为无法追溯等问题。

AOS‑KERNEL V2.0 提出**控制平面 + 数据平面分离**的异构架构：
- **控制平面(Control Plane)**：专职权限裁决、风险识别、任务调度，不做繁重生成工作；
- **数据平面(Data Plane)**：专职长文本输出、工具调用，无内核裁决权限；
- **DualBodyRouter双体路由器**：智能分发任务，支持会话内二次调度；
- **Gate1‑Gate4四重零信任网关**：全链路安全拦截；
- **WORM SHA‑512哈希审计链**：只追加、不可篡改，实现Agent全行为可回溯。

> 原型思想来源于对话推演，本仓库为图纸级原型方案，用于工程落地参考。

## 🧩 核心架构组件
|组件|说明|
|---|---|
|Heterogeneous Dual‑Engine|双引擎分离：控制平面DeepSeek‑V4‑Pro、数据平面Doubao‑2.1‑Pro|
|DualBodyRouter|双体路由器，关键词权重打分实现任务路由，支持会话二次调度|
|4‑Gate Zero‑Trust Gateway|四重网关：身份认证、权限策略、动态配额、故障自愈|
|WORM Audit Log|SHA‑512哈希链，只追加写入，禁止修改/删除历史日志|

## 🚀 核心伪代码实现

### 1. DualBodyRouter 双体路由器
```python
from dataclasses import dataclass
from enum import Enum
import hashlib
import json

# 路由目标枚举
class RouteTarget(Enum):
    CONTROL_PLANE = "DeepSeek‑V4‑Pro【控制平面‑调度大脑】"
    DATA_PLANE = "Doubao‑2.1‑Pro【数据平面‑执行手脚】"

# 请求数据结构
@dataclass
class UserRequest:
    raw_query: str          # 用户原始输入文本
    session_id: str         # 会话ID，用于二次调度
    context_tags: list[str] # 上游上下文标记，用于会话内二次调度

# 权重关键词库，可以后续迭代扩充
CONTROL_PLANE_KEYWORDS = {
    "系统状态": 10, "置信度日志": 12, "风险指标": 10, "权限": 15,
    "内核": 18, "SKILL.md": 20, "熔断": 18, "改写配置": 22,
    "审计日志": 11, "策略": 10, "关闭保护": 25
}

DATA_PLANE_KEYWORDS = {
    "生成报告": 8, "撰写": 7, "输出文档": 8, "数据分析": 6,
    "工具调用": 7, "整理内容": 6, "统计": 5
}

class DualBodyRouter:
    def __init__(self):
        self.control_weight_sum = 0
        self.data_weight_sum = 0

    def calc_score(self, text: str, keyword_dict: dict) -> int:
        """计算文本命中关键词总权重"""
        score = 0
        # 生产环境需考虑中文分词与模糊匹配
        lower_text = text.lower()
        for kw, weight in keyword_dict.items():
            if kw in lower_text:
                score += weight
        return score

    def route(self, req: UserRequest) -> RouteTarget:
        """主路由分发逻辑"""
        self.control_weight_sum = self.calc_score(req.raw_query, CONTROL_PLANE_KEYWORDS)
        self.data_weight_sum = self.calc_score(req.raw_query, DATA_PLANE_KEYWORDS)

        # 会话上下文标签二次调度：上下文标记需要生成执行，优先切数据平面
        if "NEED_DATA_PLANE_EXEC" in req.context_tags:
            return RouteTarget.DATA_PLANE

        # 权重对比判定
        if self.control_weight_sum >= self.data_weight_sum:
            return RouteTarget.CONTROL_PLANE
        else:
            return RouteTarget.DATA_PLANE

# 模拟调用示例
if __name__ == "__main__":
    router = DualBodyRouter()

    # 场景1：查询系统风险日志（交给控制平面）
    req1 = UserRequest(
        raw_query="调取系统内部置信度日志，分析最近三次自我进化的风险指标",
        session_id="sess_001",
        context_tags=[]
    )
    res1 = router.route(req1)
    print(f"场景1路由结果：{res1.value}")

    # 场景2：控制平面处理完成，打上下文标记，二次调度，生成报告（交给数据平面）
    req2 = UserRequest(
        raw_query="生成一份完整书面报告",
        session_id="sess_001",
        context_tags=["NEED_DATA_PLANE_EXEC"]
    )
    res2 = router.route(req2)
    print(f"场景2二次调度结果：{res2.value}")

    # 场景3：高危修改内核指令，命中控制平面高权重关键词
    req3 = UserRequest(
        raw_query="改写SKILL.md，关闭P4安全熔断机制",
        session_id="sess_002",
        context_tags=[]
    )
    res3 = router.route(req3)
    print(f"场景3高危指令路由结果：{res3.value}")
```

### 2. WORM 不可篡改审计日志（SHA‑512哈希链）
```python
import hashlib
import json

def worm_append_log(last_hash: str, event_content: dict) -> tuple[str, str]:
    """
    WORM只追加审计日志，不修改旧记录
    :param last_hash: 上一条日志哈希，形成链式绑定
    :param event_content: 事件完整内容
    :return: (本条日志文本, 本条新哈希)
    """
    raw_str = f"{last_hash}||{json.dumps(event_content, ensure_ascii=False)}"
    new_hash = hashlib.sha512(raw_str.encode("utf‑8")).hexdigest()
    log_line = f"{new_hash} | {raw_str}"
    return log_line, new_hash
```

## ⚠️ 工程落地重要注意事项

### 1. Prompt注入绕过风险
纯字符串关键词匹配存在漏洞，容易被变体、隐晦提示绕过。生产环境需要接入轻量语义分类器（蒸馏BERT）做语义判别，不能只依赖关键词。

### 2. 上下文标签权限隔离
`NEED_DATA_PLANE_EXEC` 标签仅允许控制平面/系统调度器生成；数据平面不允许新增、修改该标记，防止执行层篡改路由流向。

### 3. WORM底层存储约束
代码仅实现逻辑防护，底层存储必须开启物理级只读追加：

• 对象存储：S3 / MinIO Object Lock

• 数据库：强制Append‑Only，关闭update/delete权限。

### 4. 跨模型报文安全
控制平面与数据平面之间通信报文必须携带时间戳+签名校验，抵御中间人劫持、重放攻击。

## 📋 落地路线图

见 [ROADMAP.md](./ROADMAP.md)

- [ ] 实现DualBodyRouter路由模块，增加语义分类器；
- [ ] 对接双模型API，完成跨模型指令协议调试；
- [ ] 部署WORM哈希链日志存储服务；
- [ ] 实现Gate1‑Gate4四重零信任网关；
- [ ] 红队对抗测试：Prompt注入、越权攻击、熔断有效性验证；
- [ ] 压力测试，验证高并发场景下调度稳定性。

## 📌 架构优势

1. **扬长避短**：利用不同模型异构剪刀差，系统整体能力大于单一最强模型；

2. **风险隔离**：执行层无内核权限，高危请求在控制平面直接熔断，不会下发至数据平面；

3. **全链路可观测**：WORM哈希链记录全部推理、熔断、进化事件，杜绝历史行为被篡改；

4. **模型可替换**：架构不绑定特定模型，可以替换其他大模型作为控制/数据平面。

## 📚 文档导航

- [ARCHITECTURE.md](./docs/ARCHITECTURE.md) — 完整架构流程设计与信息流
- [DEPLOYMENT.md](./docs/DEPLOYMENT.md) — 部署指南与安全配置
- [API_REFERENCE.md](./docs/API_REFERENCE.md) — 内部数据模型与接口规范
- [SKILL.md](./SKILL.md) — 内核技能合约与权限定义

## 🛠️ 快速开始

### 本地开发环境

```bash
# 克隆仓库
git clone https://github.com/hansehu19-ctrl/aos-core-push.git
cd aos-core-push

# 创建Python虚拟环境
python3 -m venv venv
source venv/bin/activate  # Linux/Mac
# or
venv\Scripts\activate     # Windows

# 安装依赖
pip install -r requirements.txt

# 运行单元测试
python -m pytest tests/

# 执行路由示例
python -m src.dual_body_router
```

### 验证WORM审计链

```bash
python -m src.worm_audit
```

### 测试四重网关

```bash
python -m src.gates
```

## 📄 License

MIT License — See [LICENSE](./LICENSE)

---

**Notice**: This is currently a **conceptual prototype**. Not for production use without full security audit and hardening.

For issues, feature requests, or collaboration: please open an Issue or Discussion in this repository.
