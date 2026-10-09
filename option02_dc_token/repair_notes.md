# option02 `dc_token` — 수정 근거

바이트코드: CPython 3.9 (매직 3425). 디컴파일러: PyLingual. 기준(ground truth)은
`original.pyc` 디스어셈블(`xdis` 6.3.0)이며, `fixed.py`를 재컴파일해 상수를 역대조해
확인했다.

## 오류 1 — `paths` 딕셔너리가 리터럴 중간에서 잘림 (치명적 `SyntaxError`)

`decomp.py` 42번째 줄 (`extract_tokens.extract`):

```
paths = {'Discord': self.roaming + '...leveldb\\', ..., 'Chrome': self.appdata + '...leveldb\\', '
```

16번째 키(`Chrome`)까지만 나오고 `, '` 가 매달린 채 끊긴다. 그래서 `decomp.py`는
파싱조차 되지 않는다:

```
$ python -c "import ast; ast.parse(open('decomp.py').read())"
SyntaxError: unterminated string literal (detected at line 42)
```

바이트코드는 이 딕셔너리를 16개가 아니라 **27개** 항목으로 `BUILD_CONST_KEY_MAP` 으로
만든다. 전체 키 튜플은 `extract` 코드 객체의 상수 #28에 하나의 상수로 들어 있다:

```
('Discord','Discord Canary','Lightcord','Discord PTB','Opera','Opera GX','Amigo','Torch',
 'Kometa','Orbitum','CentBrowser','7Star','Sputnik','Vivaldi','Chrome SxS','Chrome',
 'Chrome1','Chrome2','Chrome3','Chrome4','Chrome5','Epic Privacy Browser','Microsoft Edge',
 'Uran','Yandex','Brave','Iridium')
```

27개의 값 문자열은 상수 #1..#27 이다. 각 값은 `self.<attr> + '<경로>'` 꼴이고,
`<attr>` 는 빌드 시퀀스(오프셋 0–214)에서 각 `LOAD_CONST` 앞에 오는 `LOAD_ATTR` 로
결정된다. **앞 6개는 `self.roaming`, 나머지 21개는 `self.appdata` 를 쓴다.**

수정: 누락된 11개 항목(인덱스 17–27)을 복원했다. 바이트 단위로 일치함을 검증했다 —
`fixed.py` 를 재컴파일해 `leveldb\` 로 끝나는 문자열 상수를 원본 `.pyc` 와 비교하면
차집합이 공집합(27 == 27, 누락·초과 모두 없음)이다.

키 튜플에서 복원된 키는 사람이 추측할 법한 긴 이름이 아니라 `Chrome1..Chrome5`,
`Microsoft Edge`, `Uran` 이다. 이것이 실제 소스의 식별자이므로 "보기 좋게 다듬지" 않고
그대로 두었다.

## 오류 2 — 모듈 최상단의 엉뚱한 `global pcuser` (PyLingual이 끼워 넣은 뼈대)

`decomp.py` 7번째 줄(임포트 이전): `global pcuser  # inserted`.

`global` 문은 **바이트코드를 전혀 만들지 않는다** — 컴파일러의 심볼 테이블만 바꾼다. 따라서
`.pyc` 에서는 그 존재 여부를 되읽을 수 없다. PyLingual 자신의 `# inserted` 태그가 이 줄이
디컴파일된 게 아니라 합성된(synthesized) 것임을 표시한다. 임포트보다 앞에 놓인 모듈 레벨
`global` 은 사람이 쓰는 코드가 아니므로 `fixed.py` 에서 제거했다.

반면 `def pcuser()` 안의 `global pcuser`(역시 `# inserted` 태그)는 정반대 경우다. 함수
본문이 `STORE_GLOBAL pcuser`(오프셋 8, 22)와 `LOAD_GLOBAL pcuser`(오프셋 30)를 수행하는데,
이는 소스에 `global pcuser` 선언이 있어야만 가능하다 — 따라서 이 줄은 **반드시 필요**하며
유지했다. 두 줄 모두 `# inserted` 가 붙어 있다는 사실은, 이 태그가 "필요한 것"과 "불필요한
것"을 구분해 주지 못함을 보여준다.

## 확인 후 올바르다고 판단해 손대지 않은 부분

- `def pcuser()` 가 전역 이름을 자기 자신으로 덮어쓰는 동작(`STORE_GLOBAL pcuser` 후
  `return pcuser`)은 바이트코드에 충실하다 — 모듈 이름 `pcuser` 를 함수에서 로그인
  문자열로 다시 바인딩한다.
- `calc_flags.flags_dict` 는 바이트코드상 정확히 5개 항목이다(상수가 5개 키
  `DISCORD_EMPLOYEE … HOUSE_BRAVERY` 와 일치). PyLingual이 맞게 복원했다.
- `decrypt_val` / `get_master_key` / 두 정규식은 문자열·옵코드 모두 충실하다.

## AI 도구의 과장 주장 반박

PyLingual은 (CCS'25 논문과 자사 사이트 기준) 높은 *세그먼트/정확 일치율* 을 내세운다.
그러나 이 샘플에서는 **애초에 유효한 파이썬이 아닌** 출력을 냈다 — 끊긴 문자열 리터럴이다.
게다가 **데이터 항목 27개 중 11개를 아무 오류나 `# WARNING` 없이 조용히 날렸다.** 토큰
단위로 평균 낸 일치율 지표는 이 점을 가린다. 디컴파일러는 전체 토큰 겹침 점수가 높아도,
컴파일되지도 않고 데이터를 보존하지도 못하는 파일을 내놓을 수 있다 — 분석가 입장에서 이건
0.6이 아니라 0이다. "고충실도" 주장은 "컴파일되고 라운드트립된다"를 전제로 해야 하는데, 이
출력은 그 전제를 통과하지 못한다.
