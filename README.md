# Mem0 Memory Pipeline: 4-Way vs ADD-only

## 1. 핵심 변화

| 항목 | 기존 Mem0 | 2026 V3 Mem0 |
|---|---|---|
| Memory operation | `ADD / UPDATE / DELETE / NONE` | **ADD only** |
| LLM 처리 | Fact extraction → Memory reconciliation | **Single-pass extraction** |
| 기존 memory와 충돌 | overwrite / delete | **둘 다 보존** |
| 중복 처리 | `NONE` 또는 `UPDATE` | 새 memory 생성 + linking / retrieval에서 해결 |
| 시간 변화 | 기존 fact를 UPDATE | 여러 시점의 fact를 저장하고 **Temporal Reasoning** |
| Assistant 발화 | 주로 user fact 중심 | **user + assistant 모두 memory 후보** |
| 관계 표현 | memory 자체를 수정 | `linked_memory_ids`로 연결 |
| 주요 목표 | compact하고 일관된 memory DB | **history preservation + retrieval 품질** |

Mem0는 2026년 새 알고리즘에서 이를 공식적으로 **“Single-pass ADD-only extraction — one LLM call, no UPDATE/DELETE”**라고 설명한다. [GitHub](https://github.com/mem0ai/mem0?utm_source=chatgpt.com)

---

# 2. 기존 4-Way 방식

## Architecture

```text
Conversation
     │
     ▼
┌───────────────────┐
│ Fact Extraction   │
│ LLM               │
└─────────┬─────────┘
          │
          │ newly extracted facts
          ▼
┌───────────────────┐
│ Retrieve Existing │
│ Relevant Memories │
└─────────┬─────────┘
          │
          ▼
┌─────────────────────────┐
│ Memory Manager LLM      │
│                         │
│ ADD / UPDATE / DELETE   │
│ / NONE                  │
└──────────┬──────────────┘
           │
           ▼
       Memory DB
```

즉 보통 **2단계 LLM reasoning** 구조다.

```text
messages
 → extract facts
 → retrieve existing memories
 → compare facts vs memories
 → CRUD decision
```

---

## Memory-manager prompt 요약

공식 prompt의 핵심은 다음과 같다.

```text
You are a smart memory manager.

Compare newly retrieved facts
with the existing memory.

For each fact decide:

ADD
UPDATE
DELETE
NONE
```

operation 의미:

| Operation | 조건 |
|---|---|
| `ADD` | 기존 memory에 없는 새로운 정보 |
| `UPDATE` | 동일한 subject에 대해 더 최신이거나 다른 정보 |
| `DELETE` | 기존 memory가 더 이상 유효하지 않음 |
| `NONE` | 이미 존재하거나 memory로 저장할 필요 없음 |

실제 Mem0 prompt도 이 네 가지 operation을 명시적으로 비교하도록 되어 있다. [GitHub](https://github.com/mem0ai/mem0/blob/main/mem0/configs/prompts.py?utm_source=chatgpt.com)

예를 들어:

```text
Existing:
[id=7] User lives in Seoul

New fact:
User moved to Busan
```

결과:

```json
{
  "id": "7",
  "text": "User lives in Busan",
  "event": "UPDATE"
}
```

반대로:

```text
Existing:
User likes coffee

New:
User likes coffee
```

이면:

```text
NONE
```

---

# 3. 4-Way 방식의 장단점

### 장점

```text
User lives in Seoul
        ↓
User lives in Busan
```

DB에는 최종적으로:

```text
User lives in Busan
```

만 남기 때문에 memory store가 작고 깔끔하다.

또 duplicate를:

```text
NONE
```

으로 제거하기 쉽다.

### 문제점

가장 큰 문제는 **LLM이 DB mutation을 결정한다는 것**이다.

예:

```text
2024: User worked at Samsung
2026: User works at KakaoPay
```

잘못 UPDATE하면:

```text
User worked at Samsung
```

이라는 historical fact가 사라질 수 있다.

DELETE 판단도 동일한 위험이 있다.

```text
fact extraction error
        ↓
DELETE / UPDATE 잘못 선택
        ↓
원본 memory 손실
```

즉 mutation error가 **irreversible information loss**로 연결될 수 있다.

---

# 4. V3 ADD-only 방식

2026년 Mem0가 바꾼 핵심이다.

```text
Conversation
     │
     ▼
┌────────────────────────┐
│ Single-pass Memory     │
│ Extraction LLM        │
└──────────┬─────────────┘
           │
           ├── memory
           ├── attribution
           ├── temporal context
           └── linked_memory_ids
           │
           ▼
       Append Memory
```

**UPDATE / DELETE decision이 없다.**

```text
New information
     ↓
ADD
```

기존 memory는 그대로 유지한다. [GitHub](https://github.com/mem0ai/mem0?utm_source=chatgpt.com)

---

# 5. ADD-only prompt 요약

현재 공개 코드의 `ADDITIVE_EXTRACTION_PROMPT` 핵심은:

```text
You are a Memory Extractor.

Your sole operation is ADD.

Identify every piece of memorable information
and produce self-contained,
contextually rich factual statements.
```

그리고 중요한 변화가 하나 더 있다.

```text
Extract from BOTH:
- user messages
- assistant messages
```

즉:

```text
User:
I'm visiting Tokyo next week.

Assistant:
I recommended staying near Shinjuku.
```

둘 다 memory가 될 수 있다.

공식 prompt는 정확성과 completeness를 강조하고, 관련 기존 memory가 있으면 `linked_memory_ids`를 붙이도록 한다. [GitHub](https://github.com/mem0ai/mem0/blob/main/mem0/configs/prompts.py?utm_source=chatgpt.com)

---

# 6. 실제 ADD-only 입력 context

현재 prompt builder에는 대략 다음 정보가 들어간다.

```text
Summary

Last k Messages

Recently Extracted Memories

Existing Memories

New Messages

Observation Date

Current Date
```

즉 구조적으로:

```text
                  ┌─ Summary
                  ├─ Last-k messages
Conversation ────┼─ Recently extracted
                  ├─ Existing memories
                  └─ New messages
                         │
                         ▼
                 ADD-only Extractor
```

이다. 공개 코드에서도 이 구성 요소들이 확인된다. [GitHub](https://github.com/mem0ai/mem0/blob/main/mem0/configs/prompts.py?utm_source=chatgpt.com)

---

# 7. `linked_memory_ids`

ADD-only에서 중요한 부분이다.

예:

```text
Memory 12
User joined Samsung in 2023.

Memory 41
User left Samsung in 2024.
```

새 memory가 기존 memory와 관련 있다면:

```json
{
  "memory": "User left Samsung in 2024.",
  "linked_memory_ids": ["12"]
}
```

식으로 관계를 남길 수 있다.

따라서 기존:

```text
UPDATE old fact
```

에서

```text
ADD new fact
+
link old/new
```

로 사고방식이 바뀐다.

---

# 8. Temporal information 처리

가장 중요한 차이 중 하나다.

### 기존

```text
User lives in Seoul
        ↓ UPDATE
User lives in Busan
```

결과:

```text
Busan
```

### ADD-only

```text
2024:
User lived in Seoul.

2026:
User lives in Busan.
```

둘 다 저장한다.

검색할 때:

```text
Where does the user live now?
```

이면 최신 fact를,

```text
Where did the user live in 2024?
```

이면 과거 fact를 선택한다.

즉 consistency 해결 위치가:

```text
WRITE time
```

에서

```text
RETRIEVAL time
```

으로 상당 부분 이동했다.

Mem0는 새 알고리즘의 핵심 요소로 **Temporal Reasoning**을 명시하고 있다. [GitHub](https://github.com/mem0ai/mem0?utm_source=chatgpt.com)

---

# 9. Retrieval도 같이 변경됨

ADD-only만 바꾼 것은 아니다.

현재 Mem0 설명상 retrieval은:

```text
                 ┌─ Semantic
Query ───────────┼─ BM25 keyword
                 └─ Entity matching
                         │
                         ▼
                       Fusion
                         │
                         ▼
                Temporal reasoning
                         │
                         ▼
                    Top memories
```

이다.

공식적으로:

- semantic retrieval
- BM25 keyword matching
- entity matching
- entity linking
- temporal reasoning

을 함께 사용한다고 설명한다. [GitHub](https://github.com/mem0ai/mem0?utm_source=chatgpt.com)

그래서 단순히

> “CRUD 제거해서 좋아졌다”

라고 보면 안 되고,

> **ADD-only write + richer retrieval**

의 조합이다.

---

# 10. 성능 변화

Mem0가 공개한 동일 production-representative stack 결과:

| Benchmark | Old | New | 개선 | Retrieved tokens | p50 latency |
|---|---:|---:|---:|---:|---:|
| **LoCoMo** | 71.4 | **92.5** | **+21.1** | 7.0K | 0.88s |
| **LongMemEval** | 67.8 | **94.4** | **+26.6** | 6.8K | 1.09s |
| BEAM 1M | — | 64.1 | — | 6.7K | 1.00s |
| BEAM 10M | — | 48.6 | — | 6.9K | 1.05s | :chatgpt-content-reference{index="7"}


단, 이 숫자는 **Mem0 managed platform 전체 알고리즘** 결과다.

즉:

```text
ADD-only 하나의 효과
```

라고 볼 수 없다.

동시에 들어간 것이:

```text
ADD-only extraction
+ assistant memories
+ entity linking
+ multi-signal retrieval
+ temporal reasoning
```

이기 때문이다.

또 Mem0도 managed platform에 proprietary optimization이 포함되어 있어 OSS SDK 결과가 완전히 동일하지 않을 수 있다고 명시한다. [GitHub](https://github.com/mem0ai/mem0?utm_source=chatgpt.com)

---

# 11. Architecture 비교

### Old

```text
Messages
   │
   ▼
Fact Extraction LLM
   │
   ▼
Extracted Facts
   │
   ├──── Search Existing Memories
   │
   ▼
Memory Manager LLM
   │
   ├── ADD
   ├── UPDATE
   ├── DELETE
   └── NONE
   │
   ▼
Mutable Memory Store
```

### New

```text
Messages
   │
   ├── Summary
   ├── Last-k messages
   ├── Recent memories
   ├── Existing memories
   └── dates
   │
   ▼
Single-pass Memory Extractor
   │
   ├── memory
   ├── attribution
   └── linked_memory_ids
   │
   ▼
Append-only Memory Store
   │
   ▼
Semantic + BM25 + Entity Retrieval
   │
   ▼
Temporal Reasoning
   │
   ▼
Relevant Memories
```

---

# 12. 왜 ADD-only가 합리적인가

결국 trade-off는 이거다.

### 4-way

```text
WRITE 단계에서 consistency 해결
```

장점:

- DB 작음
- duplicate 적음
- current state가 명확

위험:

```text
LLM mistake
→ UPDATE / DELETE
→ information loss
```

### ADD-only

```text
WRITE에서는 evidence 보존
RETRIEVAL에서 consistency 해결
```

장점:

- information loss 감소
- historical / temporal QA에 유리
- extraction logic 단순화
- LLM mutation error 제거

비용:

- memory 증가
- near-duplicate 증가 가능
- retrieval/reranking 난이도 증가

그래서 V3 설계는 사실상:

> **Memory DB를 current-state KV store보다 temporal event/fact log에 가깝게 운영하고, 어떤 memory가 현재 유효한지는 retrieval 단계에서 해결한다.**

라고 이해하면 가장 정확하다.

---

## 한 줄 요약

```text
Old Mem0
Extract → retrieve → LLM CRUD(ADD/UPDATE/DELETE/NONE)
→ compact but destructive memory

New Mem0
Context → single-pass ADD extraction → append + link
→ semantic/BM25/entity retrieval → temporal reasoning
→ non-destructive temporal memory
```

공개 benchmark에서는 이 **전체 V3 architecture**가 LoCoMo `71.4 → 92.5`, LongMemEval `67.8 → 94.4`로 개선됐다. [GitHub](https://github.com/mem0ai/mem0?utm_source=chatgpt.com)
