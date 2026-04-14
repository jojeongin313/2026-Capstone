# 🎴 Vocab Flashcard Skill — 음성 파일 기반 영단어 플래시카드 생성기

## 개요
음성 파일(mp3, wav, m4a 등)을 업로드하면:
1. AI가 음성 속 영단어를 감지하고 한글 뜻과 함께 추출
2. 추출된 단어 목록을 기반으로 인터랙티브 HTML 플래시카드를 생성
3. 영→한, 한→영 양방향 테스트 모드를 모두 지원

---

## 트리거 조건
다음 중 하나 이상에 해당할 때 이 스킬을 사용한다:
- 사용자가 음성 파일(mp3, wav, m4a, ogg, webm 등)을 업로드하며 단어 학습, 플래시카드, 단어 테스트를 언급할 때
- "음성에서 단어 추출", "단어장 만들어줘", "플래시카드 만들어줘" 등의 요청이 있을 때
- 업로드된 음성 파일과 함께 "영어 공부", "단어 암기", "어휘 테스트" 관련 요청이 있을 때

---

## 실행 절차

### STEP 1 — 음성 파일 수신 및 확인
- 업로드된 파일 경로: `/mnt/user-data/uploads/` 에서 확인
- 지원 포맷: mp3, wav, m4a, ogg, webm, mp4
- 파일이 없으면 사용자에게 음성 파일 업로드를 요청한다

### STEP 2 — AI로 음성에서 영단어+한글 뜻 추출
음성 파일을 base64로 인코딩 후 Anthropic API에 전달한다.

```python
import base64, json, subprocess, sys

audio_path = "/mnt/user-data/uploads/YOUR_AUDIO_FILE"

with open(audio_path, "rb") as f:
    audio_b64 = base64.b64encode(f.read()).decode("utf-8")

# 파일 확장자로 media_type 결정
ext = audio_path.split(".")[-1].lower()
media_type_map = {
    "mp3": "audio/mpeg",
    "wav": "audio/wav",
    "m4a": "audio/mp4",
    "ogg": "audio/ogg",
    "webm": "audio/webm",
    "mp4": "audio/mp4"
}
media_type = media_type_map.get(ext, "audio/mpeg")
```

> **⚠️ 주의:** Anthropic API는 현재 오디오를 직접 처리하지 않는다.
> 음성 파일이 업로드된 경우, Claude는 음성 파일 대신 **사용자에게 음성 내용을 텍스트로 제공해달라고 요청**하거나,
> 음성 파일 내용이 이미 텍스트로 컨텍스트에 있는 경우 해당 텍스트에서 단어를 추출한다.
>
> **현실적인 대안 흐름:**
> 1. 사용자가 음성 파일 업로드 → Claude가 "음성 내용(대본/스크립트)을 텍스트로도 붙여넣어 주시면 바로 추출할 수 있어요" 안내
> 2. 또는 사용자가 단어 목록을 직접 제공하면 바로 STEP 3으로 이동

### STEP 2-B — 텍스트에서 영단어 추출 (대안 흐름)
사용자가 텍스트(스크립트, 단어 목록, 문장 등)를 제공한 경우, 다음 프롬프트로 단어를 추출한다:

```
시스템 프롬프트:
"You are a vocabulary extraction assistant. Given a text in any language, extract ALL English words/vocabulary items. For each word, provide the Korean meaning. Return ONLY a JSON array, no markdown, no explanation.
Format: [{"word": "apple", "meaning": "사과"}, ...]
Rules:
- Include nouns, verbs, adjectives, adverbs
- If a word appears multiple times, include it only once
- Provide the most common/relevant Korean meaning
- For phrases (2-3 words), include them if they form a meaningful unit"
```

### STEP 3 — 플래시카드 HTML 생성
추출된 단어 목록을 바탕으로 인터랙티브 HTML 플래시카드를 생성한다.

**HTML 플래시카드 필수 기능:**
- 🔄 **카드 뒤집기**: 클릭하면 앞면(영어)↔뒷면(한글) 전환, CSS flip 애니메이션
- ↔️ **양방향 모드**: "영→한 모드" / "한→영 모드" 토글 버튼
- ⏭️ **이전/다음 네비게이션**: 카드 순서 탐색
- 🔀 **셔플**: 카드 순서 무작위 섞기
- 📊 **진행률 표시**: "3 / 20" 형태
- ✅❌ **알고 있음/모름 체크**: 학습 완료 추적
- 🎯 **결과 요약**: 전체 학습 후 맞춘 수 / 전체 표시

**HTML 구조 템플릿:**

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>영단어 플래시카드</title>
  <style>
    /* 전체 레이아웃 */
    body { font-family: 'Segoe UI', sans-serif; background: #f0f4ff; display: flex; flex-direction: column; align-items: center; min-height: 100vh; margin: 0; padding: 20px; }
    h1 { color: #2d3a8c; margin-bottom: 8px; }
    .subtitle { color: #666; margin-bottom: 24px; font-size: 14px; }

    /* 모드 토글 */
    .mode-toggle { display: flex; gap: 10px; margin-bottom: 20px; }
    .mode-btn { padding: 8px 20px; border: 2px solid #2d3a8c; border-radius: 20px; cursor: pointer; font-size: 14px; background: white; color: #2d3a8c; transition: all 0.2s; }
    .mode-btn.active { background: #2d3a8c; color: white; }

    /* 플래시카드 컨테이너 */
    .card-container { perspective: 1000px; width: 380px; height: 220px; margin: 20px 0; cursor: pointer; }
    .card { width: 100%; height: 100%; position: relative; transform-style: preserve-3d; transition: transform 0.5s; border-radius: 18px; }
    .card.flipped { transform: rotateY(180deg); }
    .card-face { position: absolute; width: 100%; height: 100%; backface-visibility: hidden; border-radius: 18px; display: flex; flex-direction: column; align-items: center; justify-content: center; box-shadow: 0 8px 32px rgba(45,58,140,0.15); }
    .card-front { background: linear-gradient(135deg, #2d3a8c, #4f5fcf); color: white; }
    .card-back { background: linear-gradient(135deg, #fff, #e8ecff); color: #2d3a8c; transform: rotateY(180deg); }
    .card-word { font-size: 2.2rem; font-weight: 700; margin-bottom: 8px; }
    .card-hint { font-size: 0.85rem; opacity: 0.7; }

    /* 진행률 */
    .progress { color: #555; font-size: 15px; margin: 10px 0; }
    .progress-bar { width: 380px; height: 6px; background: #ddd; border-radius: 3px; margin-bottom: 16px; }
    .progress-fill { height: 100%; background: #2d3a8c; border-radius: 3px; transition: width 0.3s; }

    /* 버튼 영역 */
    .controls { display: flex; gap: 12px; margin: 16px 0; flex-wrap: wrap; justify-content: center; }
    .btn { padding: 10px 22px; border: none; border-radius: 12px; cursor: pointer; font-size: 14px; font-weight: 600; transition: all 0.2s; }
    .btn-know { background: #22c55e; color: white; }
    .btn-dont { background: #ef4444; color: white; }
    .btn-prev, .btn-next { background: #e8ecff; color: #2d3a8c; }
    .btn-shuffle { background: #f59e0b; color: white; }
    .btn:hover { transform: translateY(-2px); opacity: 0.9; }

    /* 결과 화면 */
    .result { display: none; text-align: center; background: white; border-radius: 18px; padding: 40px; box-shadow: 0 8px 32px rgba(0,0,0,0.1); }
    .result h2 { color: #2d3a8c; font-size: 2rem; }
    .result p { font-size: 1.1rem; color: #555; }
  </style>
</head>
<body>
  <h1>🎴 영단어 플래시카드</h1>
  <p class="subtitle" id="subtitle">총 N개 단어 · 클릭하여 뒤집기</p>

  <div class="mode-toggle">
    <button class="mode-btn active" onclick="setMode('en-ko')">영 → 한</button>
    <button class="mode-btn" onclick="setMode('ko-en')">한 → 영</button>
  </div>

  <div class="progress-bar"><div class="progress-fill" id="progressFill"></div></div>
  <div class="progress" id="progressText">1 / N</div>

  <div class="card-container" onclick="flipCard()">
    <div class="card" id="flashcard">
      <div class="card-face card-front">
        <div class="card-word" id="frontWord"></div>
        <div class="card-hint">클릭하여 뜻 확인</div>
      </div>
      <div class="card-face card-back">
        <div class="card-word" id="backWord"></div>
        <div class="card-hint">클릭하여 단어 보기</div>
      </div>
    </div>
  </div>

  <div class="controls">
    <button class="btn btn-prev" onclick="prevCard()">◀ 이전</button>
    <button class="btn btn-dont" onclick="markCard(false)">✗ 모름</button>
    <button class="btn btn-know" onclick="markCard(true)">✓ 알아요</button>
    <button class="btn btn-next" onclick="nextCard()">다음 ▶</button>
  </div>
  <button class="btn btn-shuffle" onclick="shuffleCards()" style="margin-top:4px">🔀 섞기</button>

  <div class="result" id="result">
    <h2>🎉 학습 완료!</h2>
    <p id="resultText"></p>
    <button class="btn" style="background:#2d3a8c;color:white;margin-top:16px" onclick="restart()">다시 시작</button>
  </div>

  <script>
    // ===== 단어 데이터 (STEP 2에서 추출된 결과로 교체) =====
    const words = WORDS_JSON_PLACEHOLDER;
    // 예시: [{"word":"apple","meaning":"사과"},{"word":"brave","meaning":"용감한"}]

    let current = 0;
    let mode = 'en-ko'; // 'en-ko' or 'ko-en'
    let scores = new Array(words.length).fill(null); // true=알아요, false=모름, null=미확인
    let deck = [...Array(words.length).keys()];

    function init() {
      document.getElementById('subtitle').textContent = `총 ${words.length}개 단어 · 클릭하여 뒤집기`;
      showCard();
    }

    function showCard() {
      const idx = deck[current];
      const w = words[idx];
      const card = document.getElementById('flashcard');
      card.classList.remove('flipped');
      if (mode === 'en-ko') {
        document.getElementById('frontWord').textContent = w.word;
        document.getElementById('backWord').textContent = w.meaning;
      } else {
        document.getElementById('frontWord').textContent = w.meaning;
        document.getElementById('backWord').textContent = w.word;
      }
      updateProgress();
    }

    function flipCard() {
      document.getElementById('flashcard').classList.toggle('flipped');
    }

    function nextCard() {
      if (current < deck.length - 1) { current++; showCard(); }
      else showResult();
    }

    function prevCard() {
      if (current > 0) { current--; showCard(); }
    }

    function markCard(known) {
      scores[deck[current]] = known;
      nextCard();
    }

    function setMode(m) {
      mode = m;
      document.querySelectorAll('.mode-btn').forEach((b, i) => {
        b.classList.toggle('active', (i === 0 && m === 'en-ko') || (i === 1 && m === 'ko-en'));
      });
      showCard();
    }

    function shuffleCards() {
      for (let i = deck.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [deck[i], deck[j]] = [deck[j], deck[i]];
      }
      current = 0; scores = new Array(words.length).fill(null); showCard();
    }

    function updateProgress() {
      const pct = ((current + 1) / deck.length * 100).toFixed(0);
      document.getElementById('progressFill').style.width = pct + '%';
      document.getElementById('progressText').textContent = `${current + 1} / ${deck.length}`;
    }

    function showResult() {
      document.querySelector('.card-container').style.display = 'none';
      document.querySelector('.controls').style.display = 'none';
      const known = scores.filter(s => s === true).length;
      document.getElementById('resultText').textContent = `${words.length}개 중 ${known}개를 알고 있어요! (${Math.round(known/words.length*100)}%)`;
      document.getElementById('result').style.display = 'block';
    }

    function restart() {
      current = 0; scores = new Array(words.length).fill(null); deck = [...Array(words.length).keys()];
      document.querySelector('.card-container').style.display = 'block';
      document.querySelector('.controls').style.display = 'flex';
      document.getElementById('result').style.display = 'none';
      showCard();
    }

    init();
  </script>
</body>
</html>
```

---

## 실제 실행 시 Claude의 작업 흐름

```
1. 사용자가 음성 파일 업로드
   ↓
2. Claude: "음성 파일을 받았습니다! 음성의 텍스트 내용(대본)도 함께 붙여넣어 주시면
           즉시 단어를 추출할 수 있어요. 또는 단어 목록을 직접 제공해 주셔도 됩니다."
   ↓
3. 사용자가 텍스트/단어 목록 제공
   ↓
4. Claude가 Anthropic API 또는 자체 능력으로 영단어+한글 뜻 추출
   → JSON 배열: [{"word": "...", "meaning": "..."}, ...]
   ↓
5. HTML 템플릿의 WORDS_JSON_PLACEHOLDER를 추출된 JSON으로 교체
   ↓
6. 완성된 .html 파일을 /mnt/user-data/outputs/ 에 저장 후 present_files로 전달
```

---

## 단어 추출 품질 가이드라인

- **포함할 것:** 명사, 동사, 형용사, 부사, 관용구(2~3단어)
- **제외할 것:** 관사(a, the), 전치사(in, on, at), 접속사(and, but)
- **중복 제거:** 동일 단어는 한 번만 포함
- **한글 뜻:** 문맥상 가장 적합한 의미 1~2개 제공 (예: "run: 달리다, 운영하다")
- **최소 5개 ~ 최대 100개** 단어를 추출 (너무 적으면 테스트 의미 없음)

---

## 출력 파일

- 파일명: `flashcard_[날짜].html` (예: `flashcard_20260414.html`)
- 저장 위치: `/mnt/user-data/outputs/`
- 전달 방법: `present_files` 툴로 사용자에게 제공

---

## 오류 처리

| 상황 | 대응 |
|------|------|
| 음성 파일만 있고 텍스트 없음 | 텍스트/대본 제공 요청 |
| 단어가 5개 미만 추출됨 | 사용자에게 추가 내용 요청 |
| 지원하지 않는 파일 형식 | 지원 형식 안내 후 재업로드 요청 |
| 단어 목록 직접 제공 시 | STEP 2 건너뛰고 바로 HTML 생성 |
