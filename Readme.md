# AI角色扮演机器人内核 — 代码指导

## 一、项目目标

本项目旨在构建一个更接近真人的群聊 AI 角色扮演机器人。  
机器人能够在大群聊中：

- 自动收集并存储消息  
- 维护"聊天欲望""情绪""性格"等动态状态  
- 跟踪各用户好感度，影响互动倾向  
- 保留近期原始消息，并周期性调用 LLM 压缩为长期记忆  
- 通过 cascade prompt 完成两步 LLM 调用：  
  1. 决策是否回复 & 回复类型、语气、简化心情  
  2. 根据上下文、状态、记忆生成最终回复文本  

---

## 二、目录结构

```
ai_chatbot/
├── collector.py    # 消息收集
├── state.py        # 角色状态管理
├── affinity.py     # 好感度管理
├── memory.py       # 长期记忆管理
├── decision.py     # 决策引擎（第一次 LLM 调用）
├── response.py     # 回复生成（第二次 LLM 调用）
├── controller.py   # 主流程控制
└── models.py       # Pydantic 数据模型
```

---

## 三、模块概览

- **collector.py**  
  监听群聊消息，封装为 `Message` 对象并保存历史。  

- **state.py**  
  管理角色动态状态：聊天欲望（desire）、情绪（mood）、性格（personality）。  

- **affinity.py**  
  为每个活跃用户维护好感度分值，提供读取和更新接口。  

- **memory.py**  
  缓存最新 N 条原始消息；超出后调用 LLM 压缩为摘要，形成长期记忆；支持按上下文检索。  

- **decision.py**  
  第一次 LLM 调用。输入消息内容、角色状态、用户好感度，输出 JSON：  
  - should_respond  
  - response_type（share/answer/question）  
  - tone（short/normal/inquiring）  
  - simplified_mood（good/ok/bad）  

- **response.py**  
  第二次 LLM 调用。输入消息上下文、角色状态、决策结果、相关记忆，输出纯文本回复。  

- **controller.py**  
  同步主循环：收消息 → 更新记忆 → 决策 → 生成回复 → 发送 → 状态&好感度更新。  

- **models.py**  
  使用 `pydantic` 定义核心数据结构：`Message`、`CharacterState`、`AffinityEntry`、`MemoryEntry`、`DecisionResult`。

---

## 四、核心数据模型（models.py）

```python
from pydantic import BaseModel
from typing import Literal

class Message(BaseModel):
    sender_id: str
    content: str
    timestamp: float

class CharacterState(BaseModel):
    desire: float
    mood: Literal["happy","neutral","tired"]
    personality: str

class AffinityEntry(BaseModel):
    user_id: str
    score: float

class MemoryEntry(BaseModel):
    summary: str
    timestamp: float

class DecisionResult(BaseModel):
    should_respond: bool
    response_type: Literal["share","answer","question"]
    tone: Literal["short","normal","inquiring"]
    simplified_mood: Literal["good","ok","bad"]
```

---

## 五、主流程示例（controller.py）

```python
import time
from collector import ChatCollector
from state import StateManager
from affinity import AffinityManager
from memory import MemoryManager
from decision import DecisionEngine
from response import ResponseGenerator

def main_loop():
    collector    = ChatCollector()
    state_mgr    = StateManager()
    affinity_mgr = AffinityManager()
    memory_mgr   = MemoryManager()
    decision_eng = DecisionEngine(affinity_mgr)
    responder    = ResponseGenerator(memory_mgr)

    while True:
        sender_id, content = get_next_chat()  
        timestamp = time.time()
        msg = collector.ingest(sender_id, content, timestamp)

        memory_mgr.add(msg)

        decision = decision_eng.decide(msg, state_mgr.state)
        if not decision.should_respond:
            continue

        reply = responder.generate(msg, state_mgr.state, decision)
        send_to_group(reply)

        state_mgr.update_after_reply()
        affinity_mgr.update(msg.sender_id, delta=+1)

if __name__ == "__main__":
    main_loop()
```
