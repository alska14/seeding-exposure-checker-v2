# 시딩 노출 확인 도우미

시딩(협찬)을 받은 유튜버가 영상에서 제품을 실제로 노출했는지 자동으로 1차 선별하는 도구입니다.
**확정과 못 찾은 채널(`미확인`)은 사람이 확인한다**는 전제로 만들었습니다. `미확인`은 "안 올림"이 아닙니다.

## 흐름
시딩 리스트(엑셀) → 채널의 발송일 이후 영상 수집(YouTube API) → 제목·설명란·태그·댓글·자막에서 키워드 검색
→ 단서가 없는 채널만 영상 프레임을 비전 모델(OpenAI)로 분석 → 반박 검수 → 결과 엑셀 리포트

## 준비
1. Python 3.10+ 와 ffmpeg 설치
2. `pip install -r requirements.txt`
3. `.env.example` 을 `.env` 로 복사하고 `YOUTUBE_API_KEY`, `OPENAI_API_KEY` 입력
4. `config.example.yaml` 을 `config.yaml` 로 복사하고 브랜드·상품 정보 입력
5. 엑셀 형식은 `sample_seeding_list.xlsx` 참고 (3행부터 읽음: 발송일·번호·유튜브 아이디·상품명[색상]·옵션·채널 링크)

## 실행
```bash
python seeding_checker.py parse
python seeding_checker.py all --only 1,2,3     # 일부만 먼저 시험
python seeding_checker.py all --skip-watch     # 영상 분석 없이 텍스트 판정만
```
단계: `parse → collect → screen → watch → report`. 중간 결과는 `work/`에 저장되어 중단 후 이어집니다.
영상 분석은 OpenAI 유료 API를 쓰므로 처음에는 `--only` 로 일부만 돌리세요.

## 2차: 미확인 재검증 (`recheck`)
1차에서 `미확인`으로 남은 채널만 대상으로, 1차가 보지 못한 단서를 5가지 방법으로 더 찾습니다.

| 점검 | 하는 일 | 비용 |
| --- | --- | --- |
| `comments` 댓글 재검토 | 댓글을 아직 받지 않은 영상(발송 후 90일 밖 등)의 댓글을 받아 브랜드 언급을 다시 찾음. 착장 제목 영상부터 | YouTube 쿼터(영상당 1) |
| `fuzzy` 표기 변형 | 제목·설명란·태그·댓글에서 브랜드명과 자모 두 개 이내로 다른 표기(예: 브랜드가 '하늘마루'일 때 '하늘마류')를 찾음 | 무료 |
| `links` 설명란 링크 | 설명란 링크를 따라가 최종 주소·페이지에 브랜드가 있는지 봄. 쇼핑·단축 링크 먼저 | 무료(제3자 사이트 요청) |
| `seen` 분석 서술 | 이전 영상 분석이 적은 '본 착장' 글에 상품·브랜드 단어가 있는지 다시 찾음 | 무료 |
| `watch` 안 본 영상 | 아직 분석하지 않은 영상을 `--budget`(USD) 안에서 분석 | OpenAI 비용 |

```bash
python seeding_checker.py recheck --dry-run --budget 5   # 아무것도 호출·저장하지 않고 대상 수와 예상 비용만 출력
python seeding_checker.py recheck --steps fuzzy,links,seen   # 무료 점검만 실행
python seeding_checker.py recheck --budget 5            # 전부 실행(영상 분석 상한 $5)
```
- `--budget` 이 없으면 영상 분석은 하지 않습니다. 영상 분석 비용은 영상당 약 $0.03(1차 실측)으로 추정합니다.
- 링크·표기 변형·분석 서술로 찾은 단서는 확정이 아니라 `의심(수동확인)` 후보입니다. 정확한 표기·댓글·영상으로 확인된 것만 `확인됨`이 됩니다.
- 결과는 `work/recheck.json`, 재검증 전 판정은 `work/recheck_base.json`에 저장되고, 끝나면 "미확인 N → M" 과 바뀐 채널 목록을 출력합니다. 같은 명령을 다시 실행해도 결과가 같습니다.

## 판정
확인됨(텍스트/댓글/대본/영상) · 의심(수동확인) · 미확인 · 업로드 없음 · 채널 오류. 자세한 기준은 `docs/plan.md` 참고.

## 테스트
```bash
python -m pip install pytest
python -m pytest -q
```
외부 API 없이 가짜 데이터로 `recheck`의 각 점검, 판정 규칙, 비용 상한 계산, dry-run이 아무것도 저장하지 않는지를 검사합니다.

## 주의
- `.env`, `config.yaml`, 시딩 리스트, `work/`, `output/` 은 개인정보와 키가 들어 있어 `.gitignore` 로 제외했습니다. 올리지 마세요.
- 영상 다운로드(yt-dlp)는 YouTube 약관상 회색지대입니다. 분석 직후 파일을 지우며, 서비스로 확대하기 전에 약관을 확인해야 합니다.
