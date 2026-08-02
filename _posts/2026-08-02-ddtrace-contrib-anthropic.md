---
title: dd-trace-py에 버그 수정 Contribution 하기
date: 2026-08-02
categories: [Open Source, Python]
tags:
  - Datadog
  - dd-trace-py
image: "/assets/2026-01-11-datadog-agent-on-host/datadog-logo.png"
---

> 이 글에서는 Datadog의 오픈소스인 [dd-trace-py](https://github.com/DataDog/dd-trace-py)에 LLMOps 관련 버그를 수정하는 과정에 대해 다룹니다.
> 
> PR: [fix(llmobs): route final response helpers through traced stream #19340](https://github.com/DataDog/dd-trace-py/pull/19340)
{: .prompt-info}

## 1. Overview

Datadog에 입사하여 Technical Support Engineer로 근무한지 이제 4개월이 되었습니다.
Datadog에는 많은 서비스가 있지만, 제 Specialization은 APM, RUM, Synthetics 입니다.
이 중에서도 특히 **APM**에 대한 문의는 항상 많이 들어오며, 공부할 양도 굉장히 많은 서비스 중 하나입니다.

APM은 Application Performance Monitoring을 의미하며, 애플리케이션의 Trace를 추적하고 관리하는 서비스입니다.
이를 통해 사용자는 하나의 요청이 어떤 서비스를 거쳤고, 각 서비스에서 얼마 정도의 시간이 걸렸는지, 얼마 정도의 리소스가 소모되었는지 등을 파악할 수 있습니다.

이 Trace를 추적하기 위해서는 애플리케이션에 저희 Datadog이 제공하는 SDK나 OpenTelemetry SDK가 포함되어야 합니다.
제가 이번에 기여한 [dd-trace-py](https://github.com/DataDog/dd-trace-py)는 Datadog이 제공하는 Python SDK입니다.

따라서 Tracer는 APM 서비스를 사용할 때 가장 중요한 컴포넌트 중 하나이며, APM 서비스의 시작점이라고도 할 수 있습니다.
고객 입장에서 Tracer가 제대로 동작하지 않으면 Trace가 생성되지 않거나 불완전하게 생성되어 APM 서비스로 애플리케이션을 모니터링하는 데에 불편함을 겪을 것이고, 이는 Datadog에 대한 신뢰를 낮추는 문제가 될 수 있습니다.

Technical Support Engineer로서 능동적으로 고객을 도와줄 수 있는 방법이 뭐가 있을까 고민했을 때, 이 Tracer에 기여를 하는 것이 좋은 출발점이 될 수 있겠다고 생각하여 이번 프로젝트를 진행하게 되었습니다.

## 2. Finding Issues

기여를 하기 위해서는 문제를 찾아야 했습니다. 문제를 찾는 데에는 여러 가지 방법이 있을 수 있습니다.
직접 코드를 분석해서 문제를 찾을 수도 있고, 사용자가 리포트한 문제를 해결할 수도 있습니다.
저는 현재 발생하고 있는 문제를 파악하고자 GitHub Issues를 둘러보았고, 아래의 Issue를 발견하게 되었습니다.

[Anthropic AsyncMessageStream.get_final_message()/get_final_text() bypass TracedAsyncStream, so the span never finishes
](https://github.com/DataDog/dd-trace-py/issues/19210)

이 문제에 대해 간략히 요약해보면 아래와 같습니다.

- 환경: `Python 3.11` + `dd-trace-py 4.11.1` + `anthropic >= 0.40.0`
- 문제: Anthropic SDK의 `get_final_message()`와 `get_final_text()`가 Datadog의 Traced Stream을 우회하여, 생성된 Span이 종료되지 않습니다.

앞서 언급했던 Tracer가 정상적으로 동작하지 않아 Span이 누락되어 불편함을 겪고 있는 문제입니다.
제가 Tracer에 기여하고자 했던 목적과 부합하는 Issue이고, 최근 LLM 모델의 Performance를 모니터링하고자 하는 사용자도 증가하고 있기 때문에, 많은 사용자에게 도움이 될 수 있을 거라고 생각했습니다.

## 3. Background Knowledge

이 문제를 해결하기 위해 먼저 dd-trace-py가 Anthropic의 Streaming 응답을 어떻게 계측하는지 확인했습니다.
`ddtrace/contrib/internal/anthropic/patch.py`를 보면 Anthropic SDK의 주요 메서드를 Datadog의 계측 함수로 감싸고 있습니다.

```python
wrap("anthropic", "resources.messages.Messages.create", traced_chat_model_generate)
wrap("anthropic", "resources.messages.Messages.stream", traced_chat_model_generate)
wrap("anthropic", "resources.messages.AsyncMessages.create", traced_async_chat_model_generate)
# AsyncMessages.stream is a sync function
wrap("anthropic", "resources.messages.AsyncMessages.stream", traced_chat_model_generate)
```

`wrap()`은 Anthropic SDK를 직접 수정하지 않고, 기존 메서드 앞에 Datadog의 로직을 추가하는 **Monkey Patching** 방식으로 동작합니다.

```
client.messages.stream()
        ↓
traced_chat_model_generate()
        ↓
Anthropic SDK의 Messages.stream()
```

Non-streaming 요청은 함수가 반환될 때 완성된 응답을 받을 수 있으므로, Output과 Token 사용량을 기록하고 바로 Span을 종료할 수 있습니다.

반면 Streaming 요청은 함수가 반환된 시점에도 응답이 완성되지 않습니다.
따라서 `dd-trace-py`는 Anthropic의 Stream을 `TracedStream` 또는 `TracedAsyncStream`이라는 Proxy로 감싸고, 실제 Stream 소비가 끝날 때까지 Span 종료를 미룹니다.
[`TracedAsyncStream`](https://github.com/DataDog/dd-trace-py/blob/v4.12.2/ddtrace/llmobs/_integrations/base_stream_handler.py#L241-L253)의 주요 동작을 단순화하면 다음과 같습니다.

```python
async def __aiter__(self):
    try:
        async for chunk in self._self_async_stream_iter:
            await self._self_handler.process_chunk(chunk)
            yield chunk
    finally:
        self._self_handler.finalize_stream()
```

사용자가 Stream을 순회하면 각 Chunk가 Datadog Proxy를 통과합니다.
Datadog은 Chunk를 저장하면서도 사용자에게는 원래 Chunk를 그대로 전달합니다. 모든 Chunk가 처리되면 최종 응답을 구성하고 Span을 종료합니다.

```
Anthropic AsyncMessageStream
    ↓
TracedAsyncStream
    ↓
Chunk 저장
    ↓
사용자에게 Chunk 전달
    ↓
Stream 종료
    ↓
최종 응답 구성 및 Span 종료
```

여기서 중요한 점은 Chunk가 반드시 `TracedAsyncStream`을 통해 소비되어야 한다는 것입니다.

Anthropic SDK의 [`get_final_message()`](https://github.com/anthropics/anthropic-sdk-python/blob/v0.120.0/src/anthropic/lib/streaming/_messages.py#L237-L243)는 다음과 같이 구현되어 있습니다.

```python
async def get_final_message(self) -> ParsedMessage[ResponseFormatT]:
    await self.until_done()
    assert self.__final_message_snapshot is not None
    return self.__final_message_snapshot
```

[`until_done()`](https://github.com/anthropics/anthropic-sdk-python/blob/v0.120.0/src/anthropic/lib/streaming/_messages.py#L118-L120)은 아래와 같이 Stream에 남아 있는 모든 Chunk를 소비합니다.

```python
async def until_done(self) -> None:
    """Waits until the stream has been consumed"""
    await consume_async_iterator(self)
```

문제는 `get_final_message()`가 Datadog의 Proxy에 정의된 메서드가 아니라, 원본 `AsyncMessageStream`에 정의된 메서드라는 점입니다.
따라서 아래 코드를 호출하면,
```python
message = await stream.get_final_message()
```

`get_final_message()` 내부의 `self`는 `TracedAsyncStream`이 아닌 Anthropic의 원본 `AsyncMessageStream`입니다.
결과적으로 `until_done()`도 **원본 Stream을 직접 순회**하게 됩니다.

```
stream.get_final_message()
        ↓
AsyncMessageStream.get_final_message()
        ↓
AsyncMessageStream.until_done()
        ↓
원본 Stream 소비
        ↓
TracedAsyncStream을 우회
```

응답 자체는 정상적으로 반환되지만, Chunk가 Datadog Proxy를 통과하지 않기 때문에 Datadog은 Chunk를 저장하거나 Stream의 종료를 감지하지 못합니다.
그 결과 생성된 Span이 종료되지 않고 Datadog으로 Export되지 않습니다.
`get_final_text()`도 내부에서 `get_final_message()`를 호출하기 때문에 동일한 문제가 발생합니다.

## 4. Fix the Issue

문제의 원인을 정리하면, `get_final_message()`가 내부에서 호출하는 `until_done()`이 Datadog의 `TracedAsyncStream`이 아닌 Anthropic의 원본 Stream을 소비한다는 것이었습니다.
이를 해결하려면 `until_done()`이 원본 Stream 대신 Datadog의 Traced Stream을 순회하도록 변경해야 합니다.

저는 먼저 Sync와 Async Stream을 끝까지 소비하는 함수를 작성했습니다.

```python
def _consume_stream(traced_stream):
    for _ in traced_stream:
        pass


async def _consume_async_stream(traced_stream):
    async for _ in traced_stream:
        pass
```

이는 Anthropic SDK에 있는 [`consume_async_iterator(self)`](https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/_utils/_streams.py#L5-L7)와 동일합니다.

```python
def consume_sync_iterator(iterator: Iterator[Any]) -> None:
    for _ in iterator:
        ...
```

이 함수들은 응답 값을 직접 사용하지 않고 Stream에 남아 있는 모든 Chunk를 순회합니다.
순회 대상이 `TracedStream`이므로, 각 Chunk는 Datadog의 Stream Handler에도 전달됩니다.

그리고 실제 Stream이 생성될 때 `until_done()`을 새로운 함수로 교체했습니다.

```python
from functools import partial

def add_text_stream(stream):
    stream.text_stream = _text_stream_generator(stream)
    stream.until_done = partial(_consume_stream, stream)

def add_async_text_stream(stream):
    stream.text_stream = _async_text_stream_generator(stream)
    stream.until_done = partial(_consume_async_stream, stream)
```

여기서 `partial()`은 첫 번째 인자인 stream을 미리 연결한 새로운 함수를 만듭니다.
위 코드는 개념적으로 다음과 같습니다.

```python
async def until_done():
    await _consume_async_stream(stream)
```

Anthropic SDK는 `until_done()`을 인자 없이 호출하므로, `partial()`을 사용해 함수의 형태는 그대로 유지하면서 Datadog의 Traced Stream을 전달할 수 있습니다.
수정 후의 실행 흐름은 다음과 같습니다.

```
stream.get_final_message()
    ↓
Anthropic의 get_final_message()
    ↓
교체된 until_done()
    ↓
_consume_async_stream(TracedAsyncStream)
    ↓
Chunk 저장 및 전달
    ↓
Stream 종료 감지
    ↓
Span 종료
```

이 방법은 Anthropic SDK의 `get_final_message()` 구현을 다시 작성하지 않는다는 장점도 있습니다.
Anthropic SDK는 기존 방식대로 최종 Message를 구성하고, Datadog은 Stream의 소비 경로만 Proxy를 통과하도록 변경합니다.

또한 `get_final_text()`는 내부에서 `get_final_message()`를 호출하므로 별도의 수정 없이 함께 해결됩니다.
Stream을 일부 순회한 뒤 `get_final_message()`를 호출하는 경우에도, 남아 있는 Chunk가 계속 Traced Stream을 통과하므로 전체 응답과 Span을 정상적으로 완성할 수 있습니다.

수정 결과 다음 사용 방식 모두에서 하나의 완성된 Span이 생성되는 것을 확인했습니다.
- Sync `get_final_message()` 및 `get_final_text()`
- Async `get_final_message()` 및 `get_final_text()`
- Stream을 일부 소비한 후 `get_final_message()` 호출

테스트에서는 최종 Output Message뿐만 아니라 Input, Output, Total Token 사용량도 올바르게 기록되는지 함께 검증했습니다.

## 5. Test the fix

수정 사항이 다양한 Stream 사용 방식에서도 정상적으로 동작하는지 확인하기 위해 Unit Test를 추가했습니다.
테스트에서는 실제 API를 매번 호출하지 않고, VCR에 저장된 Anthropic 응답을 사용했습니다. 추가한 테스트 시나리오는 다음과 같습니다.
- Sync Stream에서 `get_final_message()`와 `get_final_text()` 호출
- Async Stream에서 `get_final_message()`와 `get_final_text()` 호출
- Stream을 일부 소비한 뒤 `get_final_message()` 호출

특히 마지막 시나리오는 일부 Chunk가 이미 처리된 상황에서도, `until_done()`이 남아 있는 Chunk를 Traced Stream을 통해 정상적으로 소비하는지 검증하기 위해 추가했습니다.

```python
async with llm.messages.stream(...) as stream:
    first_event = await stream.__anext__()
    assert first_event is not None

    message = await stream.get_final_message()
    assert message is not None
```

각 테스트에서는 단순히 Anthropic의 응답이 반환되는지만 확인하는 것이 아니라, 하나의 Span이 정상적으로 종료되었으며, Output과 Token 사용량도 올바르게 기록되었는지 함께 검증했습니다.

```python
spans = [span for trace in test_spans.pop_traces() for span in trace]

assert len(spans) == 1

assert_llmobs_span_data(
    _get_llmobs_data_metastruct(spans[0]),
    output_messages=[
        {
            "content": "The famous philosophical statement...",
            "role": "assistant",
        }
    ],
    metrics={
        "input_tokens": 27,
        "output_tokens": 15,
        "total_tokens": 42,
    },
)
```

마지막으로 실제 환경에서도 GitHub Issue에 있던 Reproduction 코드를 사용해 결과를 확인했습니다.
`ddtrace.auto`로 Anthropic Integration을 활성화하고, Stream을 직접 순회하는 대신 `get_final_message()`를 호출했습니다.

```python
import asyncio
import anthropic
import ddtrace.auto

async def main():
    client = anthropic.AsyncAnthropic()

    async with client.messages.stream(
        model="claude-haiku-4-5",
        max_tokens=1024,
        messages=[
            {
                "role": "user",
                "content": "What is Datadog APM?",
            }
        ],
    ) as stream:
        message = await stream.get_final_message()
        print(message)

asyncio.run(main())
```

수정 전에는 Anthropic의 응답은 정상적으로 반환되었지만 Span이 종료되지 않아 Datadog에서 확인할 수 없었습니다.
수정 후에는 동일한 코드에서 Span이 정상적으로 전송되었으며, Latency, Input/Output Message 및 Token 사용량도 함께 기록되는 것을 확인했습니다.

![](/assets/2026-08-02-ddtrace-contrib-anthropic/get_final_message-test-result.png)

마지막으로 [PR](https://github.com/DataDog/dd-trace-py/pull/19340)을 생성하여 리뷰를 요청했고, 7월 31일에 머지되었습니다.

## 7. Conclusion

이렇게 해서, [`dd-trace-py`](https://github.com/DataDog/dd-trace-py)에 제 변경사항이 적용되었습니다.
이 변경사항은 `dd-trace-py 4.12` 및 `dd-trace-py 4.13`에 Backporting될 예정입니다.

사실 `dd-trace-py`를 코드 레벨에서 본 적이 없었기 때문에, 내부 동작 구조를 파악하는 데에 꽤 어려움이 있었습니다.
또한 이 라이브러리는 다른 라이브러리를 Monkey Patching 하는 라이브러리이기 때문에, Anthropic 라이브러리의 동작 구조에 대해서도 공부해야했습니다.

무엇보다, 내가 이 PR을 올려도 될까라는 무서움이 가장 컸습니다. `dd-trace-py` 라이브러리는 Datadog에서 사용자에게 제공하고 있는 라이브러리이기 때문에 내 변경사항에 문제가 있다면 사용자에게 영향이 갈 수 있었기 때문입니다.
내가 지금 사용자가 겪고 있는 문제를 해결하기 위해 좋은 마음에서 Fix를 생성했다고 하더라도, 이후 내 Fix때문에 문제가 생기면 내가 사용자에게 불편함을 줄 수 있겠구나라는 부담감이 있었습니다.

그래서 PR을 생성하기 전에 수정 사항을 확인하고 또 확인하면서, 더 나은 방법이 있을지, 코드 스타일은 준수하고 있는지 5번 넘게 확인했습니다.
PR을 생성하고 _"Fix looks great"_ 라는 Comment를 받았을 때, 얼마나 기뻤는지 모릅니다. 

이번 Contribution을 계기로, 더 능동적으로 문제를 찾고 해결하는 것에 대한 자신감을 가지게 되었습니다.
다만, 내 수정 사항이 다른 사용자에게 영향을 준다는 점을 항상 명심하고, Ownership을 가지고 수정해야겠습니다.
앞으로도 이 기억을 바탕으로 능동적으로 문제를 찾고 해결하는 엔지니어가 될 수 있도록 노력해야겠습니다.
