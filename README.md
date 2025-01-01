### 스압) 로컬 백업 이슈와 관련하여

새해 복 많이 받으세요!

요약)
1. 이 이슈는 웹리스, node리스, 로컬리스 등 리스를 사용하는 모든 유저에게 적용됩니다.
2. 리스의 사용 약관( https://sv.risuai.xyz/hub/tos )에는 유저가 AI와 채팅한 내용을 저장한다고 명시합니다. (Article 10.2, 11.6)
  - 또한, 리스에서는 유저의 저장된 채팅 내용은 원할한 서비스 운영에 사용하고, 다른 유저에게 공유할 수 있음을 명시하고 있습니다. (Article 11.7)
3. 우선 로컬 백업 기능에서만 확인한 것입니다. 아직 리스의 소스를 제대로 확인하지는 않아, 다른 부분에서 채팅 로깅이 진행될 지 모릅니다.
4. 해결 방법 (node리스, 로컬리스)
  1. 공식 깃헙에서 저장소를 내려 받습니다.
  2. 'src/ts/drive/backuplocal.ts'에서 다음을 제거하거나 주석 처리합니다.
     ```ts
      //check backup data is corrupted
      const corrupted = await fetch(hubURL + '/backupcheck', {
          method: 'POST',
          headers: {
              'Content-Type': 'application/json'
          },
          body: JSON.stringify(getDatabase()),
      })
      if(corrupted.status === 400){
          alertError('Failed, Backup data is corrupted')
          return
      }
     ```
  3. 저장소 폴더에서 터미널 혹은 명령 프롬프트를 열어 아래 명령어를 실행합니다.
  <details>
    <summary>사용하는 환경이 node리스인 경우</summary>
    - 'pnpm run build'
    - 'pnpm run runserver'
    순서대로 실행하면 됩니다.
  </details>
  <details>
    <summary>사용하는 환경이 로컬리스인 경우</summary>
      ### Node.js를 설치했다고 가정합니다. 설치 방법은 인터넷에서 각자 검색해주세요.
      - Rustup을 이용해 Rust를 설치합니다. 역시나 방법은 각자 인터넷에서 검색 부탁드립니다.
      - 'corepack enable pnpm' // pnpm을 활성화합니다.
      - 'pnpm install'
      - 'pnpm run tauri build --target 타겟_플랫폼'
        - 타겟 플랫폼은 아래를 참고해주세요.
        - x86_64-pc-windows-msvc : x86_64 Windows
        - x86_64-apple-darwin : Intel Mac
        - aarch64-apple-darwin : ARM Mac
        - 리눅스 사용하시는 분들은... 잘 모르겠습니다. 죄송합니다.
      순서대로 실행하면 자동으로 빌드에 필요한 파일을 내려받고, 컴파일합니다.
      - 빌드된 설치 파일은 '저장소폴더/src-tauri/target/타겟 플랫폼 이름/release/bundle'
      에 저장되어 있습니다.
  </details>
  
  ### 주의! 웹리스는 RisuAI에서 호스팅하므로, 해결책을 적용할 수 없습니다.

# 본문

node리스와 로컬리스의 작동 방법을 분석한 결과 **로컬리스 또한 이슈를 회피하지 못한다**는 것을 확인했습니다.
심지어는 로컬 리스는 심지어 코드를 수정하고 재빌드하지 않는 이상, 문제 이슈의 해결이 불가능하다는 것을 파악하였습니다.


<details>
  <summary>리스의 작동 방식</summary>
  리스는 일종의 웹앱 프론트엔드로, TypeScript라는 언어로 이용해 작성되어
  이를 브라우저가 읽을 수 있게 트랜스파일하고 웹 서버를 통해 웹으로 배포합니다.
  우리는 웹 브라우저로 접속하여 리스를 구동합니다.
  (이는 node리스에서 실행할 때, pnpm run build, pnpm run runserver 라는 명령어로 실현합니다.)
</details>

<details>
  <summary>로컬 리스의 작동 방식</summary>
  우선, Tauri라는 Rust 크로스 플랫폼 프레임워크를 사용하여 리스를 임베딩, 트랜스파일하며 자체 웹서버와 임베드 브라우저를 사용해
  리스를 구동합니다. 즉, **웹리스와 구조적인 측면에서 전혀 다르지 않으며, 네이티브 또한 아닙니다.**
  (네이티브로 실행되는 것은 파일 입출력 정도일 뿐, UI의 구동은 여전히 웹리스의 그것과 같습니다.)

  오히려 로컬 리스는 여기서 한 가지 문제가 발생하는데, node로 구동하는 리스는 소스를 직접 수정해 재트랜스파일 후 구동이 가능하지만,
  로컬 리스는 최적화를 명목으로 압축을 하기에, 바로 소스를 수정할 수 없습니다.
  또한 로컬 리스는 CORS라는 일종의 보안 장치도 존재하지 않아 이러한 문제를 파악하기도 힘듭니다.
</details>

<details>
  <summary>문제를 발견하게 된 경위</summary>
  node리스로 즐거운 채팅을 이어가던 중, 백업이 잘 되지 않아 오류를 확인하니 CORS 문제가 발생하여 이를 해결하기 위해, 삽질을 하게 됩니다.
  
  해결이 되질 않자, 공식 소스 홈페이지에서 백업 관련 로직을 확인한 결과, 로컬 백업을 진행할 때, 채팅 내용을 서버에 보내어 백업 파일 파손 체크를 진행하는 것을 발견하게 됩니다.

  이는 굉장히 수상한 것이, RisuAI에 계정을 생성하여 연동한 것이 아님에도, 서버가 백업 파일의 유효성을 확인하는 것이 말이 안될 뿐더러, 백업 파일을 로드하는 과정에서 해당 과정을 수행하지 않아, 굉장히 이상하다고 생각했습니다.
</details>

<details>
  <summary>RisuAI 사용 약관에 관하여</summary>
  저는 이 약관이, RisuAI이 자사가 호스팅하는 웹리스에 적용되는 지, 로컬에서 직접 실행하는 것에도 적용이 되는지 애매하다고 생각합니다.

  우선 사용 약관은 이 홈페이지에서 확인할 수 있습니다.
  https://sv.risuai.xyz/hub/tos

  문제가 되는 부분은 Article 10.2, Article 11.6, 11.7 부분입니다.
  해당 내용은 **Risu가 당신의 정보와 저장 데이터를 서비스 운영을 위해 저장하고 사용할 수 있다**고 나와있습니다.
  또한, 11.7 항목에서는 **원할한 서비스 운영을 위해 다른 유저에게 당신의 정보를 제공할 수 있다고 나와 있습니다.**

  제 해석이 틀리거나, 문제가 있는 경우에는.. 당신이 생각하는 것이 아마도 맞을 것입니다.
</details>

여기까지 읽어주셔서 정말 감사드립니다.
