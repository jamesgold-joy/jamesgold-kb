---
title: "크롬이 꺼져도 1분 안에 복구되는 CDP 자동발행 환경 만들기 [제임스골드 코딩]"
url: https://jamesgold-investment.blogspot.com/2026/07/1-cdp.html
channel: 투자 블로그
date: 2026-07-24
tags: [구글블로그, 네이버블로그, 블로그자동발행, 에르메스에이전트, 윈도우작업스케줄러, 제임스골드코딩, 크론작업, 크롬자동실행, 티스토리, CDP모드]
source: https://jamesgold-investment.blogspot.com
---

# 크롬이 꺼져도 1분 안에 복구되는 CDP 자동발행 환경 만들기 [제임스골드 코딩]

안녕하세요, 제임스골드입니다. 회사 PC를 켜 두었는데도 CDP 크롬 창이 닫혀 크론 작업이 실패한다면, 단순 자동 실행이 아니라 크롬 상태를 계속 감시하고 꺼지면 다시 여는 "감시 시스템"을 만들어야 합니다. 이 글대로 설정하면 윈도우 로그인 뒤 CDP 크롬이 자동으로 실행되고, 실수로 닫히거나 충돌해도 1분 안에 다시 열리게 할 수 있습니다.

## 가장 안정적인 전체 구조

가장 안정적인 방식은 아래 구조입니다.

회사 PC 전원 켜짐 → 윈도우 로그인 → 작업 스케줄러 실행 → CDP 크롬 감시 스크립트 시작 → 9222 포트 확인 → 크롬이 없으면 자동 실행 → 에르메스 크론 작업이 CDP에 연결 → 블로그 글 자동 발행

중요한 것은 크롬을 한 번만 실행하는 것이 아니라, 30초마다 CDP 포트가 살아 있는지 확인하는 것입니다. Chrome 136 이상에서는 기본 개인 프로필에 --remote-debugging-port를 붙여도 기능이 적용되지 않으므로, 자동화 전용 프로필 폴더를 반드시 따로 지정해야 합니다.

## 먼저 폴더 만들기

파일 탐색기를 열고 C: 드라이브에 아래 두 폴더를 만드세요.

C:\ChromeAutomationProfile
C:\Automation

C:\ChromeAutomationProfile에는 자동화 전용 크롬의 로그인 정보, 쿠키, 네이버 블로그·구글 블로그·티스토리 로그인 세션이 보관됩니다. 개인용 크롬의 기본 프로필과 자동화 프로필을 섞지 않는 것이 안전합니다.

이 폴더로 실행한 크롬에서 각 블로그 서비스에 한 번씩 직접 로그인해 두면, 이후 같은 폴더로 실행되는 CDP 크롬은 로그인 상태를 유지합니다.

## CDP 크롬 먼저 열기

명령 프롬프트를 열고 아래 한 줄을 붙여 넣으세요.

"C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-address=127.0.0.1 --remote-debugging-port=9222 --user-data-dir="C:\ChromeAutomationProfile" --new-window "https://www.blogger.com/"

새 크롬 창이 열리면 구글 블로그, 네이버 블로그, 티스토리 등 자동 발행할 서비스에 직접 로그인하세요. 로그인 직후 크롬 주소창에서 아래 주소를 열어 JSON 정보가 표시되는지 확인합니다.

http://127.0.0.1:9222/json/version

JSON이 보이면 CDP 크롬이 정상입니다. 127.0.0.1은 내 회사 PC 안에서만 접속하도록 제한하는 주소이므로, 9222 포트를 인터넷에 공개하지 않게 됩니다.

## 감시 파일 만들기

이제 크롬이 꺼졌을 때 자동으로 다시 띄우는 파일을 만듭니다.

메모장을 실행합니다. 아래 내용을 전부 복사해 붙여 넣습니다.

$chrome = "C:\Program Files\Google\Chrome\Application\chrome.exe"
$profile = "C:\ChromeAutomationProfile"
$port = 9222
$log = "C:\Automation\cdp-chrome.log"

while ($true) {
    $cdp = Get-NetTCPConnection -LocalPort $port -State Listen -ErrorAction SilentlyContinue

    if (-not $cdp) {
        "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') CDP Chrome restart" | Add-Content $log

        Start-Process $chrome -ArgumentList @(
            "--remote-debugging-address=127.0.0.1",
            "--remote-debugging-port=9222",
            "--enable-automation",
            "--user-data-dir=$profile",
            "--new-window",
            "https://www.blogger.com/"
        )

        Start-Sleep -Seconds 25
    }

    Start-Sleep -Seconds 30
}

파일 → 다른 이름으로 저장을 누릅니다. 저장 위치는 C:\Automation으로 선택합니다. 파일 이름은 Start-CDPChrome.ps1로 입력합니다. 파일 형식은 반드시 모든 파일로 고르고, 인코딩은 UTF-8로 선택합니다.

이 파일은 30초마다 9222번 포트를 확인합니다. 크롬이 닫히거나 CDP가 멈추면 자동화용 크롬을 다시 열고, 실행 시각은 C:\Automation\cdp-chrome.log 파일에 기록합니다.

## 먼저 직접 시험하기

PowerShell을 열고 아래 명령을 실행하세요.

powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Automation\Start-CDPChrome.ps1"

크롬이 자동으로 열리면 정상입니다. 그 상태에서 CDP 크롬 창을 직접 닫아 보세요. 30초에서 1분 사이에 크롬이 다시 열리면 성공입니다. 시험이 끝나면 PowerShell 창에서 Ctrl + C를 눌러 감시 프로그램을 종료합니다.

## 작업 스케줄러 등록

윈도우 검색창에서 작업 스케줄러를 열고, 오른쪽의 작업 만들기를 누르세요.

일반 탭

이름: CDP Chrome Watchdog

사용자가 로그온할 때만 실행: 선택

가장 높은 수준의 권한으로 실행: 체크

구성 대상: Windows 11

트리거 탭

새로 만들기 → 작업 시작: 로그온할 때, 지연 작업: 1분, 사용: 체크

동작 탭

프로그램/스크립트: powershell.exe

인수 추가: -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File "C:\Automation\Start-CDPChrome.ps1"

조건 탭

"컴퓨터가 AC 전원에 연결되어 있는 경우에만 작업 시작" 체크 해제

"컴퓨터가 유휴 상태인 경우에만 작업 시작" 체크 해제

설정 탭

요청 시 작업 실행 허용: 체크

예약된 시작 시간을 놓친 경우 가능한 한 빨리 작업 실행: 체크

작업이 실패하면 다시 시작: 체크, 1분 간격, 999회

작업이 다음 시간보다 오래 실행되면 중지: 체크 해제

작업이 이미 실행 중인 경우: 새 인스턴스를 시작하지 않음

## 절전이 가장 큰 적입니다

PC가 켜져 있어도 절전 모드로 들어가면 크롬, 에르메스, WSL2, 크론 작업이 사실상 모두 멈춥니다. 회사에 PC를 두고 무인 자동 발행을 하려면 아래를 반드시 바꾸세요.

설정 → 시스템 → 전원 및 배터리 → 화면 및 절전 → 전원 연결 시 장치를 절전 모드로 전환: 안 함

노트북 덮개를 닫는다면 제어판 → 전원 옵션 → 덮개를 닫을 때의 동작 선택에서 전원 연결 시: 아무 것도 안 함으로 바꿉니다.

## 에르메스 크론 작업 점검

에르메스나 발행 스크립트에는 "크롬을 새로 실행"하도록 하기보다, 아래 주소의 이미 실행 중인 CDP 크롬에 연결하도록 설정하세요.

http://127.0.0.1:9222

각 크론 작업의 첫 단계에 다음 점검을 추가하는 것이 좋습니다. http://127.0.0.1:9222/json/version 확인 → 실패하면 60초 대기 → 다시 확인 → 그래도 실패하면 텔레그램에 오류 알림 → 연결 성공 시에만 글 발행 시작

플랫폼별로 보면 구글 블로그와 티스토리는 가능한 경우 공식 API 방식으로 발행을 옮기면 크롬이 닫혀도 글 발행이 가능해 전체 시스템이 훨씬 안정적입니다. 네이버 블로그처럼 화면 조작이 필요한 서비스만 CDP 크롬에 의존하는 혼합 구조가 가장 좋습니다.

## 매주 3분 점검

http://127.0.0.1:9222/json/version이 열리는지 확인합니다.

C:\Automation\cdp-chrome.log에 재시작 기록이 지나치게 많은지 확인합니다.

자동화용 크롬에서 블로그 로그인 상태가 유지되는지 확인합니다.

PC를 재부팅하고 3분 안에 에르메스와 CDP 크롬이 복구되는지 시험합니다.

네이버 추가 인증이나 CAPTCHA가 나오면 자동 우회를 시도하지 말고 텔레그램 알림을 받은 뒤 직접 인증합니다.

by JamesGold

공감과 댓글은 힘이 됩니다.^^. 이웃추가를 해 두시면 최신 화제와 그 배경에 관한 더 유용한 지식을 받아보실 수 있습니다. 감사합니다.^^

---

## 🔗 연결
- **채널**: [[투자 블로그]]
- **주제**: [[구글블로그]] · [[네이버블로그]] · [[블로그자동발행]] · [[에르메스에이전트]] · [[윈도우작업스케줄러]] · [[제임스골드코딩]] · [[크론작업]] · [[크롬자동실행]] · [[티스토리]] · [[CDP모드]]
- **원문**: https://jamesgold-investment.blogspot.com/2026/07/1-cdp.html
