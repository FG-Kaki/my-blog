---
layout: home
heading: Kaki's Dev Log
subheading: 배운 것을 정리하고 기록하는 공간
# 배너 이미지는 이제 _config.yml의 banner.image 기본값을 사용합니다
# (이 페이지만 다른 배너를 쓰고 싶으면 여기에 banner: /assets/images/다른이미지.jpg 를 추가하세요)
# 사이드바를 카테고리별 대신 월별 아카이브로 표시
sidebar: archive-list
---

<div class="dev-terminal">
  <div class="dev-terminal-bar">
    <span class="dev-terminal-dot dev-terminal-dot-red"></span>
    <span class="dev-terminal-dot dev-terminal-dot-yellow"></span>
    <span class="dev-terminal-dot dev-terminal-dot-green"></span>
    <span class="dev-terminal-title">kaki@bootcamp: ~/my-blog</span>
  </div>
  <pre class="dev-terminal-body" id="dev-terminal-body">$ git log -1 --oneline
006일차  버그는 하루 만에 안 없어지지만, 기록은 쌓입니다.

$ cat progress.txt
6 / 176일차 진행 중</pre>
</div>

<div class="dev-status-card">
  <div class="dev-status-tags">
    <span class="dev-tag">Git</span>
    <span class="dev-tag">GitHub</span>
    <span class="dev-tag">마크다운</span>
  </div>

  <div class="dev-progress">
    <div class="dev-progress-head">
      <span class="dev-progress-label">전체 진행률</span>
      <span class="dev-progress-numbers"><b id="bootcamp-day">-</b> / 176일 · <span id="bootcamp-percent">-</span>%</span>
    </div>
    <div class="dev-progress-track">
      <div class="dev-progress-fill" id="bootcamp-progress-fill"></div>
    </div>
  </div>

  <div class="dev-divider"></div>

  <div class="dev-github">
    <div class="dev-github-head">
      <span>GitHub 활동</span>
      <a href="https://github.com/FG-Kaki" target="_blank" rel="noopener">github.com/FG-Kaki ↗</a>
    </div>
    <!--
      ghchart.rshah.org / streak-stats.demolab.com은 GitHub 공식 서비스가
      아니라 커뮤니티가 운영하는 무료 위젯입니다. 계정명만으로 그때그때
      이미지를 그려주는 방식이라 토큰이나 별도 코드 없이 <img> 한 줄로
      끝나지만, 두 서비스가 죽으면 위젯도 같이 안 보이게 됩니다.
      색상은 브랜드 컬러(#8C8863)에 맞췄습니다.
    -->
    <div class="dev-widgets">
      <div class="dev-widget-card">
        <img src="https://ghchart.rshah.org/8C8863/FG-Kaki" alt="GitHub Contribution Chart" loading="lazy">
      </div>
      <div class="dev-widget-card">
        <img src="https://streak-stats.demolab.com/?user=FG-Kaki&theme=dark&ring=8C8863&fire=8C8863&currStreakLabel=8C8863" alt="GitHub Streak Stats" loading="lazy">
      </div>
    </div>
  </div>
</div>

<!--
  부트캠프 기간: 2026-08-26(1일차) ~ 2027-02-17(176일차, 종료일 포함).
  경과일/진행률을 매 방문마다 브라우저에서 직접 계산합니다. 이 사이트는
  push할 때만 다시 빌드되는 GitHub Pages 기본 빌드라, Liquid로 빌드
  시점 날짜를 박아두면 안 밀어붙인 날은 숫자가 멈춰있게 됩니다.
  방문자의 로컬 시간대와 무관하게 항상 한국 시간(KST, UTC+9) 자정
  기준으로 날짜 경계를 계산해서, 어디서 접속하든 같은 날엔 같은
  일차가 보이도록 했습니다.
-->
<script>
(function () {
  var BOOTCAMP_START_UTC = Date.UTC(2026, 7, 26); // 2026-08-26 = 1일차
  var BOOTCAMP_TOTAL_DAYS = 176; // 2027-02-17 종료일 포함
  var now = new Date();
  var kstNow = new Date(now.getTime() + 9 * 60 * 60 * 1000);
  var todayKstUtc = Date.UTC(kstNow.getUTCFullYear(), kstNow.getUTCMonth(), kstNow.getUTCDate());
  var day = Math.floor((todayKstUtc - BOOTCAMP_START_UTC) / 86400000) + 1;
  if (day < 1) day = 1;
  if (day > BOOTCAMP_TOTAL_DAYS) day = BOOTCAMP_TOTAL_DAYS;
  var percent = Math.round((day / BOOTCAMP_TOTAL_DAYS) * 100);
  var dayStr = ('000' + day).slice(-3);

  var prefersReduced = window.matchMedia &&
    window.matchMedia('(prefers-reduced-motion: reduce)').matches;

  // 진행률 카드 숫자 채우기
  var dayEl = document.getElementById('bootcamp-day');
  var percentEl = document.getElementById('bootcamp-percent');
  var fillEl = document.getElementById('bootcamp-progress-fill');
  if (dayEl) dayEl.textContent = day;
  if (percentEl) percentEl.textContent = percent;

  // 진행률 바가 0%로 이미 렌더링된 프레임을 한 번 그린 뒤에 목표 값으로
  // 바꿔야 CSS transition이 실제로 애니메이션으로 보입니다. 이 스크립트가
  // 실행되자마자 바로 폭을 바꾸면 브라우저가 중간 프레임을 그릴 새 없이
  // 최종값으로 바로 그려버려서 애니메이션이 생략된 것처럼 보였습니다.
  if (fillEl) {
    requestAnimationFrame(function () {
      requestAnimationFrame(function () {
        fillEl.style.width = percent + '%';
      });
    });
  }

  // 터미널 부팅 연출: HTML의 <pre>엔 이미 완성된 텍스트가 하드코딩되어
  // 있어서(자바스크립트가 꺼져 있어도, 크롤러가 봐도 정상적인 문장) 여기선
  // 그 자리를 오늘 계산한 실제 일차/진행률로 다시 채운 다음, 한 글자씩
  // 타이핑해서 보여줍니다. reduced-motion이면 타이핑 없이 값만 갱신합니다.
  var terminal = document.getElementById('dev-terminal-body');
  if (terminal) {
    var script = [
      '$ git log -1 --oneline',
      dayStr + '일차  버그는 하루 만에 안 없어지지만, 기록은 쌓입니다.',
      '',
      '$ cat progress.txt',
      day + ' / ' + BOOTCAMP_TOTAL_DAYS + '일차 진행 중'
    ].join('\n');

    if (prefersReduced) {
      terminal.textContent = script;
    } else {
      terminal.textContent = '';
      terminal.classList.add('dev-terminal-typing');
      var i = 0;
      (function typeNext() {
        if (i <= script.length) {
          terminal.textContent = script.slice(0, i);
          i++;
          setTimeout(typeNext, 18);
        } else {
          terminal.classList.remove('dev-terminal-typing');
        }
      })();
    }
  }
})();
</script>
