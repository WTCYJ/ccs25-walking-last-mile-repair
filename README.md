# CCS'25 Walking The Last Mile — PyLingual 디컴파일 복원

ACM CCS 2025 논문 [Walking The Last Mile](https://github.com/syssec-utd/CCS25-WalkingTheLastMile-Supplementary)의
공개 challenge에서 두 샘플을 골랐다. PyLingual이 복원한 `decomp.py`가 어디서 틀렸는지
원본 `original.pyc` 바이트코드로 되짚어 확인하고, 고친 결과를 여기 모았다.

- [`option02_dc_token/fixed.py`](option02_dc_token/fixed.py) · [수정 근거](option02_dc_token/repair_notes.md)
  잘린 `paths` 딕셔너리(27개 중 16개만 나오고 끊겨 `SyntaxError`)를 바이트코드 상수에서 되살렸다.
- [`option04_zarnet/fixed.py`](option04_zarnet/fixed.py) · [수정 근거](option04_zarnet/repair_notes.md)
  루프도 없는데 박힌 `break`(실체는 `JUMP_FORWARD`)를 단락 평가 가드로 복원했다.
