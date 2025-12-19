# Body Type Contract Design

(JSON Schema / Pydantic Models)

---

## 왜 이걸 다루는가

AI Agent 시스템에서 가장 먼저 고정해야 하는 것은 "모델"이나 "프롬프트"가
아니라\
**시스템에 들어오는 요청의 형태**라고 판단했다.

이 프로젝트는 `text`, `vision`, `rag`, `metadata` 등 다양한 입력 유형이
생길 가능성이 높다.\
요청 형태가 불안정하면 이후 Dispatcher, Engine, Agent의 전체 구조도
흔들린다.

그래서 초기 단계에서\
**4가지 Body Type으로 요청 구조를 계약(Contract)처럼 고정**하기로
결정했다.

---

## 이 문제가 왜 중요한가

요청 스키마가 명확하지 않으면 시스템 내부에 "추측"이 생긴다.

- API는 payload를 직접 뜯어보고 판단한다
- Engine/Agent는 특정 필드의 존재 유무를 가정하게 된다
- 타입이 늘어날수록 `if/else` 분기가 분산되고 복잡해진다

이 방식은 초기 개발 속도는 빠를 수 있지만,\
기능이 늘어나면 변경 비용이 폭발하고 구조가 무너진다.

Body Type은 이런 문제를 **입구에서 막아주는 구조적 장치**다.

---

## 방치할 경우 생기는 문제

- 필드 추가/삭제가 자연스럽게 이뤄지며 입력 형태가 암묵적으로 바뀐다
- 로직 분기가 시스템 곳곳에 흩어진다
- 요청 흐름이 어디로 가는지 추적이 어려워진다

결과적으로:

- 디버깅에 시간 낭비
- 테스트는 조합 폭발로 어려워짐
- 미래의 내가 코드를 손댈 때 유지보수 비용이 커짐

---

## 처음 생각

초기에는 이렇게 생각했다:

- "그냥 dict로 받고 필요할 때만 키 확인하면 되잖아?"
- "스키마는 나중에 정리해도 되지 않을까?"
- "혼자 하는데 왜 엄격하게 해야 하지?"

빠르게 시작하려면 스키마 정의가 쓸모없어 보였고,\
**"일단 만들어보고 나중에 정리"**라는 함정에 빠질 뻔했다.

---

## 시도 / 비교

### 1) `dict` 기반 요청 (스키마 없음)

**장점** - 구현이 빠르고 유연하다 - 실험, 프로토타이핑에 좋다

**단점** - 검증 로직이 분산되어 혼란스럽다 - 구조가 흐려지고 추론에
의존한다 - 타입 추가 시 분기가 복잡해지고 테스트도 힘들어진다

---

### 2) 타입별로 API 엔드포인트 분리

예: `/request/text`, `/request/rag`, `/request/vision`

**장점** - 입력 형태가 명확하게 구분됨 - 각 엔드포인트마다 스키마 고정
가능

**단점** - API 라우팅이 과하게 분산될 수 있음 - 같은 리소스를 다르게
처리할 때 애매해짐 - 클라이언트 입장에서 어떤 엔드포인트를 써야 할지
판단 부담이 생김

---

### 3) 단일 엔드포인트 + Body Type 계약 (최종 선택)

**장점** - API는 한 곳으로 통일되고, 분기는 내부에서 처리됨 -
Dispatcher/Engine이 Body Type 기준으로 명확하게 설계됨 - 검증이 API
입구에서 끝나서 구조가 단순해짐 - Swagger 문서 자동화가 깔끔하게 됨

**단점** - 초기에 과해 보일 수 있음 - Pydantic의 Union, Discriminated
Union 같은 개념을 알아야 함

---

## 왜 이 방식을 선택했는가

빠르게 만드는 것보다\
**변경 비용을 줄이는 것**이 더 중요하다고 판단했다.

- Vision, RAG 등 확장 방향이 명확하고
- 타입이 더 늘어날 가능성이 크며
- 구조를 일찍 고정해야 이후가 편하다

또한 전 프로젝트에서 "빨리 만들고 나중에 정리"해봤기에\
이번에는 **초기에 구조를 잡고 가는 게 훨씬 효율적**이라 판단했다.

---

## 실패/위험 요소에 대한 판단

스키마 없이 진행하면 이런 문제가 생길 수 있다:

- 필드가 슬금슬금 늘어나고
- 여러 레이어에서 입력을 직접 해석하게 되고
- 나중에 스키마를 적용하려면 손댈 곳이 너무 많아진다

결국 스키마는 정리가 아니라\
**초기 설계의 핵심**이라는 것을 체감했다.

---

## 현재 판단

Body Type 4개를 다음과 같이 **요청 계약(Contract)** 으로 확정한다:

- API에서 입력 검증을 끝낸다
- 내부 구조는 **검증된 입력만** 받는다
- 타입이 늘어나도 변경 지점은 명확하게 고정된다

새로운 Body Type은\
**비동기, 병렬, 워크플로우** 같은 차원이 달라지는 요청이 생길 때만
고려한다.

현재 구조는 확장성 + 안정성 모두 확보한 합리적 선택이다.

---

## Body Type 4개 정의 (JSON Schema / Pydantic)

```python
from typing import Dict, Literal, Optional, Union
from pydantic import BaseModel, Field, HttpUrl

class TextBody(BaseModel):
    type: Literal["text"] = "text"
    query: str = Field(..., min_length=1, description="User text prompt")

class TextMetaBody(BaseModel):
    type: Literal["text_meta"] = "text_meta"
    query: str = Field(..., min_length=1)
    metadata: Dict[str, str] = Field(
        default_factory=dict,
        description="Key-value metadata to condition generation"
    )

class VisionBody(BaseModel):
    type: Literal["vision"] = "vision"
    query: str = Field(..., min_length=1)
    image_url: HttpUrl = Field(..., description="Public image URL")
    image_id: Optional[str] = Field(default=None, description="Internal image reference")

class RagBody(BaseModel):
    type: Literal["rag"] = "rag"
    query: str = Field(..., min_length=1)
    top_k: int = Field(default=5, ge=1, le=20, description="Top-K for retrieval")
    namespace: Optional[str] = Field(default=None, description="Optional index namespace")

RequestBody = Union[TextBody, TextMetaBody, VisionBody, RagBody]
```
