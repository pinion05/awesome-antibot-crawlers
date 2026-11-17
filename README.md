# Awesome Anti-Bot Crawlers

2026년 기준, 봇탐지(Cloudflare / DataDome / Akamai / PerimeterX / Kasada 등)를 우회하는 오픈소스 크롤링 브라우저·라이브러리 정리. 벤치마크 근거와 실측 함정을 함께 기록한다.

> 최종 갱신: 2026-11-17 · Python 3.12 / Linux 기준 실측

---

## 왜 필요한가 — 탐지는 3개 계층에서 일어난다

| 계층 | 예시 | 패치로 해결 가능? |
|---|---|---|
| **IP** | 데이터센터 ASN vs 거주 IP | ✅ 프록시로 해결 |
| **TLS / HTTP2 지문** | JA4 해시, HTTP/2 SETTINGS 프레임 순서 | ❌ HTTPS 클라이언트 자체가 출처 — 프록시 무용 |
| **자동화 프로토콜 지문** | Playwright의 `Runtime.enable`·`Target.setAutoAttach` 핸드셰이크 | ❌ 파이썬 훅으론 못 건드림 |

핵심: **지문 패치(navigator.webdriver 위장 등)는 JS 계층만 건드린다.** 반면 봇탐지 게이트는 "브라우저를 어떻게 구동했는가"(컨트롤 플레인)를 보고, 여기서 Playwright 계열은 패치 품질과 무관하게 노출된다. 도구 선택은 타겟이 어느 계층을 게이트하는지에 달렸다.

---

## 벤치마크 요약

### techinz/browsers-benchmark (23 엔진 × Cloudflare/DataDome/Akamai/Kasada 등 10 시스템, 회전 프록시)

| 엔진 | 우회율 | 메모리 |
|---|---|---|
| **patchright (헤디드)** | **100%** | 1,314MB |
| cloakbrowser / camoufox_headless | 90% | 936 / 1,318MB |
| nodriver-chrome | 80% | 1,389MB |
| vanilla playwright (헤디드) | 60% | 1,103MB |
| playwright-chrome_headless | 40% | 517MB |

- 헤드리스는 전 엔진에서 일관되게 더 잡힌다.
- reCAPTCHA v3는 패치 없는 Playwright조차 0.9점 — 구글 캡차와 Cloudflare는 다른 축.

### ianlpaterson/anti-detect-browser-bench (7 도구 × 31 타겟 × 3회, 단일 거주 IP)

| 도구 | OK / Blocked |
|---|---|
| **nodriver** | 28 / **0 (유일 무차단)** |
| CloakBrowser | 26 / 2 |
| curl_cffi (브라우저 없음!) | 26 / 2 — CloakBrowser와 동점 |
| patchright (channel=chrome) | 25 / 3 |
| camoufox | 25 / 3 |
| vanilla playwright | 24 / 5 |
| rebrowser-playwright | 24 / 5 — 사실상 vanilla |

- `canadianinsider`(Cloudflare Turnstile)는 **nodriver만 통과** — Playwright 셈이 없어서.
- 두 벤치 순위가 뒤집힌 이유: 회전 프록시는 Camoufox의 식별 가능한 Firefox TLS 지문이 쌓이는 걸 막아준다. **프록시 전략이 브라우저 선택보다 큰 레버.**

---

## 도구 상세

### nodriver
- **방식**: 시스템 Chrome을 DevTools WebSocket으로 직결. Playwright 셈 없음, `Runtime.enable` 시퀀스 없음.
- **GitHub**: <https://github.com/ultrafunkamsterdam/nodriver> (undetected-chromedriver 후속)
- **라이선스**: AGPL-3.0 (상업 서비스 수정 시 소스 공개 의무 — 주의)
- **실측**: 31타겟 블록 0개. canadianinsider/medium/glassdoor 통과.
- **주의**: asyncio 객체 모델(Playwright 드롭인 아님), 기본값으로 쿠키가 타겟 간 공유됨.

### Patchright
- **방식**: Playwright 포크. CDP 누출 시그널 패치 + `channel="chrome"`로 시스템 Chrome TLS 사용.
- **GitHub**: <https://github.com/Kaliiiiiiiiii-Vinyzu/patchright>
- **라이선스**: Apache-2.0
- **실측**: techinz 헤디드 100%. `channel="chrome"`이 패치보다 더 큰 레버.
- **주의**: 번들 Chromium 대신 **반드시 시스템 Chrome** — 번들 156은 Cloudflare 사이트에서 네비게이션 hang 실측.

### Camoufox
- **방식**: Firefox 포크, C 레벨에서 canvas/WebGL/navigator/화면지문 위장. 프로필마다 일관된 가상 하드웨어.
- **GitHub**: <https://github.com/daijro/camoufox>
- **라이선스**: MPL-2.0
- **실측**: 구글 검색 통과(Chromium 계열 3종은 차단됨), headless 90%.
- **주의**: CPU 비싸다(88.6%). Firefox TLS 쉐이프는 dev.to류 CDN에서 한 건 차단 실측.

### curl_cffi
- **방식**: 브라우저 없음. curl-impersonate 래퍼 — TLS 핸드셰이크가 실제 Chrome과 동일한 JA4.
- **GitHub**: <https://github.com/lexiforest/curl_cffi>
- **라이선스**: MIT · 6.4MB 휠
- **실측**: 31타겟 중 26개 OK — 130MB Chromium 포크와 동점. JS 불필요 타겟의 기본 선택.
- **용법**: `requests.get(url, impersonate="chrome")`

### zendriver
- **방식**: nodriver의 유지보수 포크. 같은 CDP 직결 철학.
- **GitHub**: <https://github.com/zenlyd/zendriver>
- **라이선스**: AGPL-3.0
- **실측**: sannysoft 58항목 전부 통과 (headless).

### CloakBrowser
- **방식**: Chromium 포크, 소스 레벨 C++ 패치 49건. Turnstile 자동해소 내장.
- **GitHub**: <https://github.com/CloakHQ/CloakBrowser>
- **라이선스**: MIT(래퍼) + 커스텀 바이너리 라이선스
- **실측**: 벤치마다 편차 — 회전 프록시 환경에선 강함, 단일 IP 벤치에선 curl_cffi와 동점.

### DrissionPage
- **방식**: CDP 직결 + 중국 커뮤니티 표준. 요청/브라우저 하이브리드.
- **GitHub**: <https://github.com/g1879/DrissionPage>
- **라이선스**: BSD-3

### botasaurus
- **방식**: 브라우저+HTTP 프레임워크. Cloudflare 이스케이프 내장, 데코레이터 API.
- **GitHub**: <https://github.com/omkarcloud/botasaurus>
- **라이선스**: MIT

### (비추천) rebrowser-playwright
- 마지막 실질 커밋 2024-09, Chromium 136. 실측에서 vanilla와 동일한 차단 세트. 스킵.

---

## 선택 가이드

| 상황 | 선택 |
|---|---|
| Cloudflare 게이트 강한 프로덕션 타겟 | **nodriver** |
| 기존 Playwright 코드 자산이 클 때 | **patchright + `channel="chrome"`** |
| 구글 검색·Firefox 화이트리스트 타겟 | **camoufox** |
| JS 렌더링 불필요한 API/HTML | **curl_cffi** |
| 대량 병렬 세션 (메모리 제약) | playwright headless + 엄선된 타겟 (517MB) |

## 실측 함정 (2026-11, Linux 서버 기준)

1. **sannysoft 개편** — 기존 `passed/failed` CSS 클래스 행이 사라지고 Intoli 텍스트 테이블 형식이 됨. 구버전 파서는 "0행" 오판. `tr` innerText에서 `(failed)`/`warning` 문자열 검색으로 판정해야.
2. **Patchright 번들 Chromium은 hang** — 시스템 Chrome으로 띄워라: `p.chromium.launch(channel="chrome")`.
3. **헤드풀 시스템 Chrome은 dbus 없는 셸 세션에서 CDP 포트를 안 건다** — 서버 자동화는 headless 또는 `xvfb-run` + dbus 세션.
4. **nodriver 고정 포트 지정 시 접속 실패** — 포트를 지정하지 않는 자동 free-port가 정상 경로.
5. **헤드리스는 어느 엔진이든 더 잡힌다** — 가능하면 xvfb 헤디드.
6. **PEP 668 (Ubuntu 24.04)** — 전역 pip 설치 불가. `python3 -m venv ~/venvs/antibot` 후 설치.

## 재현 가능한 검증 스크립트

```python
# nodriver + bot.sannysoft.com (실측 58/58 통과)
import asyncio, json
import nodriver as uc

async def main():
    browser = await uc.start(headless=True)   # 포트 미지정이 정답
    tab = await browser.get("https://bot.sannysoft.com/")
    await tab.sleep(20)
    raw = await tab.evaluate(
        "JSON.stringify(Array.from(document.querySelectorAll('tr'))"
        ".map(e => e.innerText.replace(/\\n/g,' | ').trim()))"
    )
    rows = json.loads(raw)
    fails = [r for r in rows if "(failed)" in r.lower() or "warning" in r.lower()]
    print(json.dumps({"total": len(rows), "failed": fails}, ensure_ascii=False))
    browser.stop()

asyncio.run(main())
```

```python
# curl_cffi TLS 지문 확인 (browserleaks JA4)
from curl_cffi import requests
d = requests.get("https://tls.browserleaks.com/json", impersonate="chrome").json()
print(d["ja4"])   # 실제 Chrome과 동일한 JA4 해시 형태
```

## 참고 자료

- [techinz/browsers-benchmark](https://github.com/techinz/browsers-benchmark) — 벤치마크 도구 자체가 오픈소스, 직접 돌릴 수 있음
- [ianlpaterson/anti-detect-browser-bench](https://github.com/ianlpaterson/anti-detect-browser-bench) — 31타겟 매트릭스 원본 데이터
- [scrapfly Best Stealth Browsers](https://scrapfly.io/blog/posts/best-stealth-browsers)
- [pim97/anti-detect-browser-tools-tech-comparison](https://github.com/pim97/anti-detect-browser-tools-tech-comparison)

## 라이선스

이 문서: CC0. 각 도구 라이선스는 원본 저장소 참고 (AGPL-3.0 상업 사용 주의: nodriver, zendriver).
