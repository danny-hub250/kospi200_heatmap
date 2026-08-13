# KOSPI 200 Heatmap

[market-heatmap](https://github.com/danny-hub250/market-heatmap)을 참고해서 만든
국내 KOSPI 200 종목용 실시간(폴링) 트리맵 히트맵입니다. 빌드 도구나 서버 없이
`index.html` 파일 하나로 동작하며, GitHub Pages에 그대로 올려서 쓸 수 있습니다.

## 실행 방법

- **로컬에서 바로 열기**: `index.html`을 브라우저로 더블클릭해서 열면 됩니다.
- **GitHub Pages로 배포**: 저장소 Settings → Pages → Branch를 이 브랜치(또는 머지 후
  기본 브랜치)로 설정하면 `https://<user>.github.io/kospi200_heatmap/` 에서 바로
  접속할 수 있습니다.

## 데이터 소스 (무료 API)

네이버페이 증권이 자체 페이지에서 실시간 시세를 갱신할 때 사용하는 공개 JSON
엔드포인트를 그대로 사용합니다. 별도의 API 키/가입이 필요 없습니다.

```
https://polling.finance.naver.com/api/realtime/domestic/stock/{종목코드1,종목코드2,...}
```

다만 이 엔드포인트는 `finance.naver.com` 외의 출처(origin)에서 브라우저로 직접
호출하면 CORS 정책에 막힙니다(실제로 확인됨: `No 'Access-Control-Allow-Origin'
header is present`). 그래서 공개 CORS 프록시를 앞단에 두고 호출하며,
`index.html` 상단의 `PROXY_BUILDERS` 배열에 다음 순서로 등록되어 있습니다. 하나가
실패(또는 8초 타임아웃)하면 자동으로 다음 프록시를 시도합니다.

1. 프록시 없이 직접 호출 (위 이유로 항상 실패하지만 비용이 없어 그대로 둠)
2. `https://api.codetabs.com/v1/proxy?quest=...`
3. `https://api.allorigins.win/raw?url=...`
4. `https://corsproxy.io/?url=...`
5. `https://cors.eu.org/...`

200여 종목을 한 번에 요청하면 URL이 너무 길어지므로 40종목씩 끊어서 순차적으로
요청하고(배치 사이 350ms 대기), 5분마다 자동 새로고침합니다.

> **참고**: 처음에 등록했던 `thingproxy.freeboard.io`는 도메인 자체가 죽어서
> (`ERR_NAME_NOT_RESOLVED`) 제거했고, `corsproxy.io`는 요청이 403으로 막히는
> 경우가 관측되어 우선순위를 낮췄습니다. 무료 공개 CORS 프록시는 이렇게 예고 없이
> 죽거나 막히는 일이 흔하므로, 아래 자체 프록시 설정을 강력히 권장합니다.

### 가장 안정적인 방법: 나만의 프록시 만들기 (Cloudflare Workers, 무료)

공개 프록시는 데모 서비스라 트래픽이 몰리거나 운영자가 내리면 바로 죽습니다.
5분 정도만 투자하면 무료로 훨씬 안정적인 나만의 프록시를 만들 수 있습니다.

1. [dash.cloudflare.com](https://dash.cloudflare.com) 가입(무료) 후 로그인
2. 왼쪽 메뉴 **Workers & Pages** → **Create** → **Create Worker**
3. 이름 아무거나 입력(예: `kospi-proxy`) → **Deploy** (일단 기본 코드로 배포)
4. 배포 후 **Edit code** 클릭, 아래 코드로 전체 교체 후 **Deploy**:

   ```js
   export default {
     async fetch(req) {
       const target = new URL(req.url).searchParams.get("url");
       if (!target) return new Response("Missing url param", { status: 400 });
       const res = await fetch(target, {
         headers: { referer: "https://finance.naver.com/" },
       });
       return new Response(res.body, {
         headers: {
           "content-type": "application/json",
           "access-control-allow-origin": "*",
         },
       });
     },
   };
   ```

5. Worker 상세 화면에 나오는 주소(`https://kospi-proxy.<계정>.workers.dev`)를 복사
6. `index.html`의 `PROXY_BUILDERS` 배열 맨 앞에 아래처럼 한 줄 추가:

   ```js
   const PROXY_BUILDERS = [
     u => u,
     u => `https://kospi-proxy.<계정>.workers.dev/?url=${encodeURIComponent(u)}`, // 추가
     u => `https://api.codetabs.com/v1/proxy?quest=${encodeURIComponent(u)}`,
     // ... 기존 목록
   ];
   ```

Cloudflare Workers 무료 플랜은 하루 10만 요청까지 무료라 이 정도 규모에는 충분합니다.

다른 대안으로는 [한국투자증권 Open API](https://apiportal.koreainvestment.com/)
(무료 가입, 앱키/시크릿 발급 후 REST API 제공)가 있는데, 이 경우 앱 시크릿을
브라우저에 노출하지 않도록 별도의 백엔드가 필요합니다.

### 시세가 계속 안 보일 때 (디버깅)

모든 셀이 회색이거나 하단 상태 텍스트가 "불러오는 중…"에서 멈춰 있다면 브라우저
개발자 도구(F12) → **Console** 탭을 열어보세요. `[KOSPI200] 요청 실패: ...` 로그가
찍히는데, 어떤 프록시/엔드포인트가 실패했는지, HTTP 상태 코드나 타임아웃인지가
그대로 남습니다. 공개 CORS 프록시는 언제든 죽거나 막힐 수 있으므로, 계속 실패한다면
`PROXY_BUILDERS` 배열에 다른 프록시를 추가하거나 위에서 설명한 자체 프록시로
교체하는 것을 권장합니다. 각 요청은 8초 타임아웃 후 다음 프록시로 자동 전환됩니다.

## 구성 종목 안내 (중요)

KOSPI 200의 실제 편입 종목·비중은 한국거래소(KRX)가 매년 6월/12월에
정기 변경합니다. `index.html`의 `STOCKS` 배열은 시가총액 상위 종목 위주로
구성한 **135개 대표 종목** 목록이며, 최신 공식 편입 종목 200개와 완전히
일치하지는 않습니다. `w`(weight) 값도 실시간 시가총액이 아니라 트리맵 크기
배분을 위한 대략적인 상대값입니다.

정확한 최신 구성종목이 필요하면:

1. [KRX 정보데이터시스템](http://data.krx.co.kr) → 지수 → 구성종목 상세정보에서
   `KOSPI 200` 구성종목 리스트(종목코드 포함)를 내려받고
2. `index.html`의 `STOCKS` 배열을 해당 200개 종목으로 교체하면 됩니다.

각 항목은 `{c:"종목코드", n:"종목명", sector:"섹터명", w:상대비중}` 형태입니다.

## 화면 구성

- `market-heatmap` 원본과 동일하게 D3 트리맵 기반, 섹터별로 묶어서 표시합니다.
- 국내 증시 관례에 맞춰 **상승은 빨강, 하락은 파랑**으로 표시합니다(원본 미국
  버전과 반대).
- 헤더에 장중/장마감 배지(KST 09:00~15:30 기준, 주말은 항상 장마감)를 표시합니다.
- 종목 클릭 시 현재가·등락률 툴팁이 뜹니다.
- 5분마다 자동 새로고침, 우측 상단 "새로고침" 버튼으로 수동 갱신도 가능합니다.

## 알려진 한계

- 무료 공개 CORS 프록시에 의존하므로 프록시 상태에 따라 일시적으로 시세가
  안 보일 수 있습니다(이 경우 셀이 회색으로 표시됩니다).
- 네이버 시세 API는 비공식(리버스 엔지니어링된) 엔드포인트라 응답 형식이
  예고 없이 바뀔 수 있습니다.
- `STOCKS` 목록은 위에 설명한 대로 완전한 공식 KOSPI 200 리스트가 아닙니다.
