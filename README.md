# CCS'25 Walking The Last Mile — PyLingual 디컴파일 복원

ACM CCS 2025 논문 [Walking The Last Mile](https://github.com/syssec-utd/CCS25-WalkingTheLastMile-Supplementary)의
공개 challenge 중 두 샘플을 골라, PyLingual이 복원한 `decomp.py`를 원본 `original.pyc`의
바이트코드를 기준으로 검증하고 고친 결과.

- [`option02_dc_token/fixed.py`](option02_dc_token/fixed.py) · [수정 근거](option02_dc_token/repair_notes.md)
  — 잘린 `paths` 딕셔너리(27개 중 16개만 복원 → `SyntaxError`)를 바이트코드 상수에서 복원
- [`option04_zarnet/fixed.py`](option04_zarnet/fixed.py) · [수정 근거](option04_zarnet/repair_notes.md)
  — 루프 밖 `break`(사실은 `JUMP_FORWARD`)를 단락 평가 가드로 복원
- [전체 보고서](report.md)

두 샘플 모두 실제 자격증명 탈취형 악성코드의 소스이며, 분석 대상은 악성코드의 동작이 아니라
디컴파일러의 복원 오류다. 어떤 샘플도 실행하지 않았다.

검증: `fixed.py`를 재컴파일한 뒤 상수·제어 흐름을 원본 `.pyc`와 대조해 동치임을 확인했다.
