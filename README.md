# TIC-VLA on SerBot II — 개발 일지

TIC-VLA 모델을 SerBot II 로봇에 온보드로 올린 과정을 기록하는 Jekyll 사이트입니다.
GitHub Pages에서 별도 빌드 설정 없이 그대로 동작합니다.

`_config.yml`은 `wodu2s/ticvla-devlog` 기준으로 이미 채워져 있습니다.

```yaml
url: "https://wodu2s.github.io"
baseurl: "/ticvla-devlog"
```

저장소 이름을 다르게 만들었다면 `baseurl`을 그 이름으로 바꾸세요. **프로젝트 페이지에서 `baseurl`이 틀리면 CSS가 전혀 적용되지 않습니다.**

---

## 올리기

```bash
cd site
git init
git add .
git commit -m "TIC-VLA 개발 일지 초기 커밋"
git branch -M main
git remote add origin https://github.com/wodu2s/ticvla-devlog.git
git push -u origin main
```

그다음 저장소 → **Settings → Pages** → Source를 **Deploy from a branch**, 브랜치 `main` / `/ (root)`로 지정하고 저장합니다. 1~2분 뒤 주소가 열립니다.

저장소는 **Public**이어야 합니다 (무료 계정에서 Private은 Pages 사용 불가).

---

## 할 일 목록 관리

`todo.md` 하나가 원본입니다. 체크하는 방법은 두 가지입니다.

**GitHub 웹에서 편집** — 저장소에서 `todo.md`를 열고 연필 아이콘 → `- [ ]`를 `- [x]`로 고치고 Commit.
**웹 에디터** — 저장소 화면에서 `.` 키를 누르면 브라우저에 VS Code가 열립니다. 여러 항목을 한 번에 고칠 때 훨씬 빠릅니다.

커밋할 때마다 **언제 무엇을 끝냈는지가 git 히스토리에 자동으로 남습니다.** 이게 이 방식의 가장 큰 장점입니다.

GitHub의 파일 보기에서는 체크박스가 클릭 가능한 형태로 렌더링됩니다. 사이트 페이지(`/todo/`)에서는 읽기 전용으로 표시됩니다.

---

## 글 추가하기

`_posts/`에 `YYYY-MM-DD-영문-슬러그.md` 형식으로 파일을 만들면 자동으로 목록에 올라갑니다.

```markdown
---
title: "글 제목"
date: 2026-08-20 14:00:00 +0900
summary: "목록에 표시될 한 줄 요약."
tags: [태그1, 태그2]
---

본문을 마크다운으로 씁니다.
```

강조 상자를 쓰고 싶으면 본문 안에 그대로 HTML을 넣으면 됩니다.

```html
<div class="callout">
<span class="label">배운 것</span>
내용
</div>
```

## 이미지 · 영상 넣기

`assets/img/` 폴더를 만들어 넣고 본문에서 참조합니다. **이 사이트는 `baseurl`이 있으므로 앞에 반드시 붙이세요.**

```markdown
![캡션]({{ site.baseurl }}/assets/img/파일명.jpg)
```

**데모 영상은 이 사이트의 설득력을 가장 크게 올릴 요소입니다.** 파일이 크면 YouTube에 올리고 임베드하는 편이 낫습니다.

## 로컬에서 미리보기 (선택)

```bash
gem install bundler jekyll
bundle install
bundle exec jekyll serve
```

`http://127.0.0.1:4000/ticvla-devlog/`에서 확인할 수 있습니다. 안 해도 GitHub에 올리면 바로 보입니다.

---

## 공개 시 주의

이 사이트는 내부 정보를 뺀 상태로 작성했습니다. **push한 내용은 지우고 다시 커밋해도 git 히스토리에 영구히 남습니다.** 글을 추가할 때 아래는 넣지 마세요.

- 로봇 IP 주소, VPN 주소, 계정명, SSH 설정
- `/home/사용자명/...` 같은 내부 경로
- 액세스 토큰이 포함된 저장소 URL
- 시연 장소를 특정할 수 있는 사진·표현
- 사람 얼굴이 식별되는 실험 영상 (모자이크 권장)

가장 흔히 새는 경로는 **터미널 로그를 그대로 붙여넣는 것**입니다. 로그에는 경로와 계정명이 거의 항상 들어 있습니다.

커밋에 이메일이 기록되므로, GitHub → Settings → Emails에서 **"Keep my email addresses private"**를 켜두는 편이 좋습니다.
