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

## 판정
확인됨(텍스트/댓글/대본/영상) · 의심(수동확인) · 미확인 · 업로드 없음 · 채널 오류. 자세한 기준은 `docs/plan.md` 참고.

## 주의
- `.env`, `config.yaml`, 시딩 리스트, `work/`, `output/` 은 개인정보와 키가 들어 있어 `.gitignore` 로 제외했습니다. 올리지 마세요.
- 영상 다운로드(yt-dlp)는 YouTube 약관상 회색지대입니다. 분석 직후 파일을 지우며, 서비스로 확대하기 전에 약관을 확인해야 합니다.
