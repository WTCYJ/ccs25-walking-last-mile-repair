# option02 `dc_token` — 수정 근거

바이트코드는 CPython 3.9(매직 3425), 디컴파일러는 PyLingual이다. 기준은 `original.pyc`를
`xdis` 6.3.0으로 디스어셈블한 결과이고, 고친 `fixed.py`는 다시 컴파일해 상수를 역대조했다.

## 오류 1: paths 딕셔너리가 중간에 잘린다

`decomp.py` 42번째 줄, `extract_tokens.extract` 안이다.

```
paths = {'Discord': self.roaming + '...leveldb\\', ..., 'Chrome': self.appdata + '...leveldb\\', '
```

16번째 키 `Chrome`까지만 나오고 `, '`가 매달린 채 끊긴다. 미묘한 버그가 아니라 파일이 파싱조차
안 되는 상태다.

```
$ python -c "import ast; ast.parse(open('decomp.py').read())"
SyntaxError: unterminated string literal (detected at line 42)
```

바이트코드는 이 딕셔너리를 16개가 아니라 27개 항목으로 `BUILD_CONST_KEY_MAP`으로 만든다.
전체 키 튜플이 `extract` 코드 객체의 상수 #28에 통째로 들어 있다.

```
('Discord','Discord Canary','Lightcord','Discord PTB','Opera','Opera GX','Amigo','Torch',
 'Kometa','Orbitum','CentBrowser','7Star','Sputnik','Vivaldi','Chrome SxS','Chrome',
 'Chrome1','Chrome2','Chrome3','Chrome4','Chrome5','Epic Privacy Browser','Microsoft Edge',
 'Uran','Yandex','Brave','Iridium')
```

27개의 값 문자열은 상수 #1부터 #27이다. 각 값은 `self.<attr> + '<경로>'` 꼴이고, `<attr>`가
`roaming`인지 `appdata`인지는 빌드 시퀀스(오프셋 0–214)에서 `LOAD_CONST` 앞에 오는 `LOAD_ATTR`로
정해진다. 앞 6개가 `self.roaming`, 나머지 21개가 `self.appdata`다.

그래서 수정은 누락된 11개(인덱스 17–27)를 되살리는 것으로 끝난다. 바이트 단위로 같은지도
확인했다. `fixed.py`를 재컴파일해 `leveldb\`로 끝나는 문자열 상수를 원본 `.pyc`와 비교하면
차집합이 양쪽 다 공집합이다(27 == 27, 누락도 초과도 없음).

복원된 키가 사람이 추측할 법한 긴 이름이 아니라 `Chrome1`부터 `Chrome5`, `Microsoft Edge`,
`Uran` 같은 짧은 식별자라는 점도 짚어 둔다. 이게 실제 소스의 이름이라 보기 좋게 다듬지 않고
그대로 뒀다.

## 오류 2: 모듈 최상단의 엉뚱한 global pcuser

`decomp.py` 7번째 줄, 임포트보다도 앞이다. `global pcuser  # inserted`.

`global` 문은 바이트코드를 전혀 만들지 않는다. 컴파일러의 심볼 테이블만 바꾸기 때문에 `.pyc`에서
그 존재 여부를 되읽을 길이 없다. PyLingual이 직접 단 `# inserted` 태그가 이 줄이 디컴파일된 게
아니라 합성된 것임을 알려 준다. 임포트보다 앞에 놓인 모듈 레벨 `global`은 사람이 쓰는 코드가
아니므로 `fixed.py`에서 지웠다.

`def pcuser()` 안의 `global pcuser`는 정반대다. 역시 `# inserted` 태그가 붙어 있지만, 함수 본문이
`STORE_GLOBAL pcuser`(오프셋 8, 22)와 `LOAD_GLOBAL pcuser`(오프셋 30)를 수행한다. 이건 소스에
`global pcuser` 선언이 있어야만 나오는 옵코드라 반드시 있어야 하는 줄이고, 그대로 유지했다. 같은
`# inserted` 태그가 지워야 할 줄과 남겨야 할 줄에 똑같이 붙어 있다는 건, 이 태그가 둘을 구분해
주지 못한다는 뜻이다.

## 확인만 하고 손대지 않은 부분

- `def pcuser()`가 전역 이름을 자기 자신으로 덮어쓰는 동작(`STORE_GLOBAL pcuser` 뒤에
  `return pcuser`)은 바이트코드 그대로다. 모듈 이름 `pcuser`를 함수에서 로그인 문자열로 다시
  바인딩한다.
- `calc_flags.flags_dict`는 바이트코드상 정확히 5개 항목이다. 상수가 5개 키
  `DISCORD_EMPLOYEE … HOUSE_BRAVERY`와 맞아떨어진다. PyLingual이 맞게 복원했다.
- `decrypt_val`, `get_master_key`, 두 정규식은 문자열이든 옵코드든 전부 그대로다.

## AI 도구의 과장 주장 반박

PyLingual은 CCS'25 논문과 자사 사이트에서 높은 세그먼트/정확 일치율을 내세운다. 그런데 이
샘플에서는 애초에 유효한 파이썬이 아닌 출력을 냈다. 끊긴 문자열 리터럴이다. 거기에 데이터 항목
27개 중 11개를 아무 오류나 `# WARNING` 없이 조용히 날렸다. 토큰 단위로 평균 낸 일치율은 바로
이 점을 가린다. 전체 토큰 겹침 점수가 높아도, 컴파일되지도 데이터를 보존하지도 못하는 파일이
나올 수 있다. 분석가에게 이건 0.6이 아니라 0이다. "고충실도"를 말하려면 "컴파일되고
라운드트립된다"가 먼저인데, 이 출력은 거기서 걸린다.
