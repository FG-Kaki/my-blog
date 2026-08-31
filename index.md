---
layout: home
heading: Kaki's Dev Log
subheading: 배운 것을 정리하고 기록하는 공간
# 배너 이미지는 이제 _config.yml의 banner.image 기본값을 사용합니다
# (이 페이지만 다른 배너를 쓰고 싶으면 여기에 banner: /assets/images/다른이미지.jpg 를 추가하세요)
# 사이드바를 카테고리별 대신 월별 아카이브로 표시
sidebar: archive-list
---

안녕하세요. 개발을 배우면서 그날그날 정리한 것을 여기에 쌓습니다.

- 배운 것: Git, GitHub, 마크다운
- 지금 하는 것: 부트캠프 <span id="bootcamp-day">-</span>일차

<!--
  부트캠프 시작일(2026-08-26, KST 기준 1일차) 이후 경과일을 매 방문마다
  브라우저에서 직접 계산합니다. 이 사이트는 push할 때만 다시 빌드되는
  GitHub Pages 기본 빌드라, Liquid로 빌드 시점 날짜를 박아두면 안 밀어붙인
  날은 숫자가 멈춰있게 됩니다. 방문자의 로컬 시간대와 무관하게 항상 한국
  시간(KST, UTC+9) 자정 기준으로 날짜 경계를 계산해서, 어디서 접속하든
  같은 날엔 같은 일차가 보이도록 했습니다.
-->
<script>
(function () {
  var BOOTCAMP_START_UTC = Date.UTC(2026, 7, 26); // 2026-08-26 = 1일차
  var now = new Date();
  var kstNow = new Date(now.getTime() + 9 * 60 * 60 * 1000);
  var todayKstUtc = Date.UTC(kstNow.getUTCFullYear(), kstNow.getUTCMonth(), kstNow.getUTCDate());
  var day = Math.floor((todayKstUtc - BOOTCAMP_START_UTC) / 86400000) + 1;

  var el = document.getElementById('bootcamp-day');
  if (el && day > 0) el.textContent = day;
})();
</script>

### 최근 GitHub 활동

<div id="github-activity">불러오는 중...</div>

<!--
  GitHub REST API(/users/{username}/events/public)를 방문 시점에 브라우저에서
  직접 호출합니다. 별도 토큰이나 서버 없이도 되는 이유: 이 엔드포인트는 공개
  이벤트만 반환해서 인증이 필요 없고, 응답 헤더에 Access-Control-Allow-Origin: *
  이 붙어 있어 다른 도메인(이 블로그)의 JS에서 바로 fetch할 수 있습니다.
  60회/시간 제한이 있지만 이건 요청을 보낸 방문자 개인 IP 기준이라
  블로그 방문자 수가 늘어도 서로 영향을 주지 않습니다.
-->
<script>
(function () {
  var container = document.getElementById('github-activity');
  var typeLabel = {
    PushEvent: '커밋 푸시',
    PullRequestEvent: 'PR',
    IssuesEvent: '이슈',
    IssueCommentEvent: '이슈 댓글',
    CreateEvent: '브랜치/저장소 생성',
    ForkEvent: '포크',
    WatchEvent: 'Star'
  };

  function formatKstDate(iso) {
    var kst = new Date(new Date(iso).getTime() + 9 * 60 * 60 * 1000);
    var mm = String(kst.getUTCMonth() + 1).padStart(2, '0');
    var dd = String(kst.getUTCDate()).padStart(2, '0');
    return kst.getUTCFullYear() + '.' + mm + '.' + dd;
  }

  fetch('https://api.github.com/users/FG-Kaki/events/public')
    .then(function (res) {
      if (!res.ok) throw new Error('GitHub API 응답 오류: ' + res.status);
      return res.json();
    })
    .then(function (events) {
      var items = events.slice(0, 5).map(function (e) {
        var label = typeLabel[e.type] || e.type;
        var repo = e.repo && e.repo.name ? e.repo.name : '';
        return '<li>' + formatKstDate(e.created_at) + ' · ' + label + ' · ' +
          '<a href="https://github.com/' + repo + '" target="_blank" rel="noopener">' + repo + '</a></li>';
      });
      container.innerHTML = items.length
        ? '<ul>' + items.join('') + '</ul>'
        : '최근 공개 활동이 없습니다.';
    })
    .catch(function () {
      container.innerHTML = '<a href="https://github.com/FG-Kaki" target="_blank" rel="noopener">GitHub 프로필에서 바로 보기</a>';
    });
})();
</script>