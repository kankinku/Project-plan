<%*
/**
 * Git Auto Update Template
 * 이 템플릿은 실행 시 백그라운드에서 현재 볼트의 변경 사항을 GitHub에 푸시합니다.
 * 
 * 전제 조건:
 * 1. Obsidian Templater 설정에서 "Allow scripts to access network etc."가 켜져 있어야 합니다.
 * 2. 시스템에 Git이 설치되어 있어야 합니다. (C:\Users\hanji\Documents\Obsidian Vault 경로에서 git 명령 가능해야 함)
 * 3. GitHub 인증(Credential Manager 등)이 로컬에 설정되어 있어야 비밀번호 입력 없이 푸시됩니다.
 */

// 1. 환경 설정
const vaultPath = app.vault.adapter.basePath; // C:\Users\hanji\Documents\Obsidian Vault
const timestamp = tp.date.now("YYYY-MM-DD HH:mm:ss");
const commitMsg = `Auto Update: ${timestamp}`;

// 2. 실행할 Git 명령어 조합 (Windows 호환)
// Change Directory -> Add -> Commit -> Push
const cmd = `cd /d "${vaultPath}" && git add . && git commit -m "${commitMsg}" && git push origin main`;

// 3. child_process를 통한 실행
const { exec } = require('child_process');

new Notice(`🚀 Git Auto Update 시작...\n대상: ${vaultPath}`);

exec(cmd, (err, stdout, stderr) => {
    if (err) {
        // 에러 발생 (Exit Code != 0)
        // 변경 사항이 없는 경우도 여기서 걸릴 수 있음
        if (stdout && stdout.includes("nothing to commit")) {
            new Notice("✅ 변경 사항이 없어 업데이트를 건너뜁니다.");
            return;
        }
        
        console.error("Git Update Error:", err);
        console.log("Stdout:", stdout);
        console.log("Stderr:", stderr);
        
        new Notice(`❌ Git Update 실패!\n${err.message}\n(Console 로그(Ctrl+Shift+I)를 확인하세요)`);
        return;
    }

    // 성공
    console.log("Git Update Success:", stdout);
    new Notice(`✅ Git Update 성공!\nCommited: ${timestamp}`);
});

// 문서에 내용을 남기지 않음
tR = "";
%>