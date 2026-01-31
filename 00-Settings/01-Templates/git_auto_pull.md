<%*
/**
 * Git Auto Pull (HARD OVERWRITE)
 * 실행 시 현재 볼트(같은 레포)를 origin upstream 기준으로 강제 동기화합니다.
 *
 * ⚠️ 주의:
 * - 로컬 변경사항, 커밋 안 된 파일 전부 삭제됩니다.
 *
 * 전제 조건:
 * 1. Templater: "Allow scripts to access network etc." ON
 * 2. Git 설치 및 vaultPath에서 git 명령 가능
 * 3. GitHub 인증(credential) 설정 완료
 */

// 1. 환경 설정
const vaultPath = app.vault.adapter.basePath;
const timestamp = tp.date.now("YYYY-MM-DD HH:mm:ss");
const { exec } = require("child_process");

// 2. 실행 커맨드 (pullhard)
const cmd = [
  `cd /d "${vaultPath}"`,
  `git rev-parse --is-inside-work-tree`,
  `git fetch origin`,
  `git reset --hard @{u}`,
  // 필요 시 완전 초기화 (untracked 포함) ↓
  // `git clean -fd`,
].join(" && ");

new Notice(`⬇️ Git HARD Pull 시작...\n대상: ${vaultPath}`);

exec(cmd, (err, stdout, stderr) => {
  if (err) {
    console.error("Git HARD Pull Error:", err);
    console.log("Stdout:", stdout);
    console.log("Stderr:", stderr);
    new Notice(`❌ Git HARD Pull 실패\n${err.message}\n콘솔(Ctrl+Shift+I) 확인`);
    tR = "";
    return;
  }

  // 성공 메시지
  if (stdout && stdout.includes("HEAD is now at")) {
    new Notice(`✅ Git HARD Pull 완료!\n시각: ${timestamp}`);
  } else {
    new Notice("✅ Git HARD Pull 완료 (변경사항 없음)");
  }

  console.log("Git HARD Pull Output:", stdout);
  tR = "";
});

tR = "";
%>
