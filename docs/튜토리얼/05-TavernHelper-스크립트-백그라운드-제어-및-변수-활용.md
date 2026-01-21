# TavernHelper 스크립트: 백그라운드 제어 및 변수 활용

우리는 변수 구조를 통해 변수 업데이트에 대해 이미 많은 제한을 둘 수 있었습니다. 예를 들어, 다음은 우리가 이전에 문지기(Gatekeeper) 캐릭터 작성 도우미에게 요청했던 변수 결과입니다.

```yaml title="변수 생성 프롬프트 예시"
변수를 생성하세요!
- 월드 경로 하위에 현재 시간, 현재 날짜 및 최근 사건을 기록
- 주인공의 여동생 하쿠아(Bai Ya)의 현재 주인공에 대한 의존도, 복장 및 칭호를 기록
  - 의존도는 반드시 0~100 사이여야 함
  - 복장은 상의, 하의, 속옷, 양말, 신발, 장식을 포함
  - 칭호는 칭호의 효과와 그에 대한 하쿠아의 자가 평가를 기록해야 하며, 자가 평가가 비어있을 경우 기본값을 "평가 대기"로 설정
  - 칭호에는 수량 상한이 있으며, 의존도가 높을수록 더 많은 칭호를 가질 수 있음 (의존도가 0일 때는 칭호 보유 불가, 1~10일 때는 1개 보유 가능, 이후 순차적으로 적용). 만약 칭호가 보유 가능 수량을 초과하면 가장 오래된 칭호를 제거해야 함
- 주인공 아이템란에 있는 현재 아이템 기록: 아이템 설명, 수량. 만약 수량을 적지 않으면 기본 수량은 1, 수량이 양수가 아니라면 해당 아이템을 즉시 삭제해야 함
```

보시다시피, 변수 구조 내에서 이미 많은 기능을 구현했습니다.

* 변수값 제한: "의존도는 반드시 0~100 사이여야 함"

* 변수값 오류 시 자동 수정: "만약 수량이 양수가 아니라면 해당 아이템을 즉시 삭제해야 함"

* AI가 변수 삽입 시 필드를 생략할 경우 기본값 설정: "만약 수량을 적지 않으면 기본 수량은 1"

* 변수 수량 상한: "칭호에는 수량 상한이 있으며..."

* ......

!!! info "변수 구조의 한계와 스크립트의 필요성"
    변수 구조는 업데이트된 후의 변수 값을 검증하고 수정하는 데에만 사용할 수 있습니다.

    * Tavern의 다른 정보를 가져올 수 없음
    * 업데이트 전의 변수 값과 비교할 수 없음
    * Tavern과 다른 상호작용을 하거나 Tavern 내용을 수정할 수 없음

    따라서 불가능한 기능이 많으며, 이때 우리는 **Tavern Assistant 스크립트**나 **Tavern Assistant 인터페이스**를 사용해야 합니다.
활용 예시: 쇼핑 기능
예를 들어, 변수를 이용해 상품 쇼핑 등의 기능을 만든다고 가정해 봅시다. 플레이어가 무엇을 살지 일일이 타이핑하고 AI가 구매 가능 여부를 계산하고 금액을 차감하게 하는 것보다는, Tavern Assistant를 사용하여 프론트엔드 인터페이스나 스크립트를 제작하는 것이 훨씬 효율적입니다.

인터페이스에 상품명, 이미지, 가격을 표시하여 플레이어가 버튼을 클릭해 구매하게 하고, 모든 구매가 완료된 후에 구매 과정 로그와 결과만 AI에게 전송하면 됩니다.

<p align="center">
  <img src="https://vwdygpvfycixugqsgwgn.supabase.co/storage/v1/object/public/post-images/1980dc04-3f08-4581-88cb-b949a10d585b/e70d7b01-34a3-42ba-a87a-cdba17caa2d6.jpg" width="400">
</p>

Tavern Assistant 프론트엔드 인터페이스나 스크립트의 구체적인 작성 방법은 **[Aozora Rii(青空莉)의 실시간 프론트엔드 인터페이스 또는 스크립트 작성](https://stagedog.github.io/%E9%9D%92%E7%A9%BA%E8%8E%89/%E5%B7%A5%E5%85%B7%E7%BB%8F%E9%AA%8C/%E5%AE%9E%E6%97%B6%E7%BC%96%E5%86%99%E5%89%8D%E7%AB%AF%E7%95%8C%E9%9D%A2%E6%88%96%E8%84%9A%E6%9C%AC/)**을 참고해 주세요. 여기서는 MVU가 추가로 제공하는 코드 기능과 이에 대응하는 예제를 통해, 프론트엔드 인터페이스나 스크립트가 변수를 어떻게 제어하고 활용할 수 있는지 소개하겠습니다.

(하지만 프론트엔드 인터페이스나 스크립트가 이 기능들만 수행할 수 있다는 뜻은 아닙니다. 예를 들어, MVU 자체도 하나의 Tavern Assistant 스크립트입니다!)

## MVU 이벤트 리스닝

MVU 변수 프레임워크가 어떻게 작동하는지 다시 한번 간단히 상기해 봅시다.

* MVU는 `[initvar]` 및 `<initvar>` 블록을 읽어 변수를 초기화합니다.
* AI가 변수 출력 형식에 맞춰 변수 업데이트 명령을 출력하면, MVU가 이 명령을 파싱 하여 변수를 업데이트합니다.

이 과정에서 다음과 같은 여러 단계를 확인할 수 있습니다.

* (새 채팅 시작 시에만) 변수 초기화 완료 (`VARIABLE_INITIALIZED`)
* 변수 업데이트 시작 (`VARIABLE_UPDATE_STARTED`)
* 변수 업데이트 명령 파싱 완료 (`COMMAND_PARSED`)
* 스크립트가 파싱 된 업데이트 명령을 사용해 순차적으로 변수를 업데이트하며, 매 업데이트마다 변수 구조를 사용해 결과를 검증
* 변수 업데이트 종료 (`VARIABLE_UPDATE_ENDED`)
* 스크립트가 변수 결과를 해당 메시지(로어북/채팅)에 저장하기 직전 (`BEFORE_MESSAGE_UPDATE`)

MVU는 이러한 단계들에 대해 **"이벤트와 해당 정보"**를 전송합니다. 우리는 새로운 Tavern Assistant 스크립트를 만들어 이 이벤트들을 리스닝(감지) 하기만 하면, 해당 단계에서 추가 기능을 실행하거나 MVU의 업데이트 과정을 조정할 수 있습니다.

!!! info "팁"
    아래 내용이 무슨 말인지 이해가 안 가시나요? 괜찮습니다. 지금 중요한 것은 **무엇을 할 수 있는지** 아는 것입니다! 
    
    그다음 **'실시간 프론트엔드 인터페이스 또는 스크립트 작성'** 문서를 읽고, 작성 템플릿에 포함된 `@types/iframe/exported.mvu.d.ts` 파일과 여기 있는 예제를 AI에게 보내서 대신 작성해달라고 시키면 됩니다.

!!! warning "경고"
    아래 기능을 사용하기 전에, 반드시 코드 맨 윗부분에 `await waitGlobalInitialized('Mvu');` 한 줄을 추가하여 MVU 변수 프레임워크의 초기화가 완료될 때까지 기다려야 합니다.
    
    이 점을 강조하기 위해, 이하 모든 예제 코드의 시작 부분에는 `await waitGlobalInitialized('Mvu');`가 추가되어 있습니다. 만약 더 복잡한 코드를 작성한다면, 코드의 최상단에 `waitGlobalInitialized('Mvu');`를 한 번만 추가하면 되며, 여러 번 추가할 필요는 없습니다.

## COMMAND_PARSED: 변수 업데이트 명령 파싱 완료

"변수 업데이트 명령 파싱 완료" 이벤트(`Mvu.events.COMMAND_PARSED`)를 리스닝(감지)하면, 대응하는 변수 업데이트 명령을 가져와 수정(Repair)할 수 있습니다.

### 예시 1: 불필요한 기호 제거
예를 들어, Gemini 모델이 중국어(또는 한글) 사이에 `-`를 집어넣는 문제를 수정하여, `Role.Luo-Luo`를 `Role.LuoLuo`로 고치는 경우입니다.

```javascript title="하이픈(-) 제거 스크립트"
await waitGlobalInitialized('Mvu');

eventOn(Mvu.events.COMMAND_PARSED, commands => {
  commands.forEach(command => {
    // 명령의 첫 번째 인자(변수 경로)에서 모든 '-'를 제거
    command.args[0] = command.args[0].replaceAll('-', '');
  });
});
```

또 다른 예로, 번체자를 간체자로 자동 변환하여 絡絡을 络络으로 수정하는 경우입니다. (한국어 사용자의 경우, 특정 오타 자동 수정 등에 응용할 수 있습니다.)

```
import { toSimplified } from 'chinese-simple2traditional';

await waitGlobalInitialized('Mvu');

eventOn(Mvu.events.COMMAND_PARSED, commands => {
  commands.forEach(command => {
    // 명령의 첫 번째 인자(변수 경로 등)를 간체자로 변환
    command.args[0] = toSimplified(command.args[0]);
  });
});
```

### VARIABLE_UPDATE_ENDED: 변수 업데이트 종료
"변수 업데이트 종료" 이벤트(Mvu.events.VARIABLE_UPDATE_ENDED)를 리스닝하면, 업데이트 **전**과 **후**의 변수 값을 모두 가져올 수 있어, 결과에 대한 추가 처리를 수행할 수 있습니다.

예시: 변수 변경 알림 팝업
예를 들어, 다음과 같이 화면에 업데이트 전후의 변수 값을 팝업으로 띄워줄 수 있습니다.

```javascript
await waitGlobalInitialized('Mvu');
eventOn(Mvu.events.VARIABLE_UPDATE_ENDED, (new_variables, old_variables) => {
  toastr.info(`업데이트 전 하쿠아 의존도는: ${_.get(old_variables, 'stat_data.하쿠아.의존도')}`);
  toastr.info(`업데이트 후 하쿠아 의존도는: ${_.get(new_variables, 'stat_data.하쿠아.의존도')}`);
});
```

또는, 다음과 같이 업데이트된 변수 값을 **강제로 수정**할 수도 있습니다.

```javascript title="변수 값 강제 수정 스크립트" hl_lines="4"
await waitGlobalInitialized('Mvu');

eventOn(Mvu.events.VARIABLE_UPDATE_ENDED, variables => {
  // 업데이트 결과가 무엇이든 관계없이, 하쿠아의 의존도를 강제로 0으로 변경
  _.set(variables, 'stat_data.하쿠아.의존도', 0);
});
```

이것만으로도 우리는 정말 많은 기능을 구현할 수 있습니다.

그중 일부는 **변수 구조**에서도 이미 구현 가능한 기능들입니다:


=== "의존도 범위 제한 (0~100)"
    의존도 수치가 항상 지정된 범위 내에 있도록 강제합니다.

    ```javascript
    await waitGlobalInitialized('Mvu');

    eventOn(Mvu.events.VARIABLE_UPDATE_ENDED, variables => {
      // 하쿠아의 의존도를 0과 100 사이로 고정 (_.clamp 활용)
      _.update(variables, 'stat_data.하쿠아.의존도', value => _.clamp(value, 0, 100));
    });
    ```

=== "아이템 자동 삭제 (수량 0 이하)"
    아이템 수량이 0 이하가 되면 목록에서 즉시 제거합니다.

    ```javascript
    await waitGlobalInitialized('Mvu');

    eventOn(Mvu.events.VARIABLE_UPDATE_ENDED, variables => {
      // 주인공의 아이템란에서 수량이 0보다 큰 항목만 남김
      _.update(variables, 'stat_data.주인공.아이템란', data => _.pickBy(data, ({수량}) => 수량 > 0));
    });
    ```

=== "칭호 수량 상한 제어"
    의존도가 높을수록 더 많은 칭호를 가질 수 있도록 동적으로 제한합니다.

    ```javascript
    await waitGlobalInitialized('Mvu');

    eventOn(Mvu.events.VARIABLE_UPDATE_ENDED, variables => {
      _.update(variables, 'stat_data.하쿠아.칭호', data =>
        _(data)
          .entries()
          // 의존도/10(올림) 값만큼 최신 칭호 유지
          .takeRight(Math.ceil(_.get(variables, 'stat_data.하쿠아.의존도') / 10))
          .value(),
      );
    });
    ```

=== "특정 조건 기록 (플래그)"
    의존도가 처음으로 특정 수치를 넘었을 때 시스템 플래그를 활성화합니다.

    ```javascript
    await waitGlobalInitialized('Mvu');

    eventOn(Mvu.events.VARIABLE_UPDATE_ENDED, variables => {
      // 의존도가 30을 초과하면 '突破30' 플래그를 true로 설정
      if (_.get(variables, 'stat_data.하쿠아.의존도') > 30) {
        _.set(variables, 'stat_data.$flag.하쿠아의존도돌파30', true);
      }
    });
    ```

=== "데이터 일괄 삭제 (사망 등)"
    특정 캐릭터가 사망 처리될 경우 관련 변수 데이터를 모두 제거하여 정리합니다.

    ```javascript
    await waitGlobalInitialized('Mvu');

    eventOn(Mvu.events.VARIABLE_UPDATE_ENDED, variables => {
      // 아오조라 리(青空莉)의 사망 상태가 true이면 관련 데이터 삭제
      if (_.get(variables, 'stat_data.아오조라리.사망') === true) {
        // 아오조라 리와 관련된 모든 변수 제거
        _.unset(variables, 'stat_data.아오조라리');
      }
    });
    ```

하지만 변수 구조 스크립트는 이전의 변수 상태를 가져올 수 없으므로, `old_variables`를 활용해 다음과 같은 작업들을 수행할 수 없습니다. 반면, Tavern Assistant 스크립트를 이용하면 가능합니다.

=== "의존도 변동 폭 제한 (±3)"
    의존도 수치가 한 번에 ±3 이상 변하지 않도록 제한합니다.

    ```javascript
    await waitGlobalInitialized('Mvu');

    eventOn(Mvu.events.VARIABLE_UPDATE_ENDED, (new_variables, old_variables) => {
      const old_value = _.get(old_variables, 'stat_data.하쿠아.의존도');

      // 새로운 호감도는 반드시 (구 호감도 - 3)과 (구 호감도 + 3) 사이여야 함
      _.update(new_variables, 'stat_data.하쿠아.의존도', value => _.clamp(value, old_value - 3, old_value + 3));
    });
    ```

=== "의존도 30 돌파 감지"
    의존도가 30을 돌파하는 순간을 감지하고 알림을 띄웁니다.

    ```javascript
    await waitGlobalInitialized('Mvu');

    eventOn(Mvu.events.VARIABLE_UPDATE_ENDED, (new_variables, old_variables) => {
      const old_value = _.get(old_variables, 'stat_data.하쿠아.의존도');
      const new_value = _.get(new_variables, 'stat_data.하쿠아.의존도');
      if (old_value < 30 && new_value >= 30) {
        toastr.success('하쿠아 의존도가 30을 돌파했습니다!');
      }
    });
    ```

=== "AI 변수 수정 방지"
    AI가 특정 변수를 업데이트하지 못하게 막습니다 (강제 롤백).

    ```javascript
    await waitGlobalInitialized('Mvu');

    eventOn(Mvu.events.VARIABLE_UPDATE_ENDED, (new_variables, old_variables) => {
      // 새로운 하쿠아 의존도를 강제로 구 의존도로 설정하여 AI의 업데이트를 취소함
      _.set(new_variables, 'stat_data.하쿠아.의존도', _.get(old_variables, 'stat_data.하쿠아.의존도'));
    });
    ```

## 스크립트 전용 MVU 변수

스크립트로 이렇게 많은 기능을 구현할 수 있다면, 오직 스크립트에서만 사용할 변수를 설정하고 싶을 수도 있습니다……

변수가 AI에 의해 업데이트되지 않게 하거나 AI에게 보이지 않게 하는 방법을 기억하시나요? 변수 이름 앞에 `_`를 붙여 AI가 업데이트하지 못하게 하거나, `$`를 붙여 AI에게 보이지 않게 만들 수 있습니다!

물론, 방금 `VARIABLE_UPDATE_ENDED`를 사용하여 변수가 AI에 의해 업데이트되지 않도록 하는 방법도 보여드렸습니다:

```javascript
await waitGlobalInitialized('Mvu');

eventOn(Mvu.events.VARIABLE_UPDATE_ENDED, (new_variables, old_variables) => {
  // 새로운 하쿠아 의존도를 강제로 구 의존도로 설정하여 AI의 업데이트를 취소함
  _.set(new_variables, 'stat_data.하쿠아.의존도', _.get(old_variables, 'stat_data.하쿠아.의존도'));
});
```

## 코드에서 직접 MVU 변수 가져오기 및 업데이트

MVU 이벤트를 수신하는 것 외에도, 우리는 직접 MVU 변수를 가져오거나 업데이트할 수 있으며, 텍스트 내의 `<JSONPatch>` 등 업데이트 명령을 주도적으로 구문 분석할 수도 있습니다.

=== "MVU 변수 가져오기"
    ```javascript
    await waitGlobalInitialized('Mvu');

    // 5번째 메시지의 MVU 변수 가져오기
    const variables = Mvu.getMvuData({ type: 'message', message_id: 5 });

    // 마지막 메시지의 MVU 변수 가져오기
    const variables = Mvu.getMvuData({ type: 'message', message_id: -1 });  // 또는 `message_id: 'latest'`

    // 뒤에서 두 번째 메시지의 MVU 변수 가져오기
    const variables = Mvu.getMvuData({ type: 'message', message_id: -2 });

    // 프론트엔드 인터페이스에서, 현재 메시지의 MVU 변수 가져오기
    const variables = Mvu.getMvuData({ type: 'message', message_id: getCurrentMessageId() });
    ```

=== "MVU 변수 업데이트"
    ```javascript
    await waitGlobalInitialized('Mvu');

    // 현재 프론트엔드 인터페이스가 있는 메시지의 MVU 변수 가져오기
    const mvu_data = Mvu.getMvuData({ type: 'message', message_id: getCurrentMessageId() });

    // 하쿠아 의존도 5 증가
    _.update(mvu_data, 'stat_data.하쿠아.의존도', value => value + 5);

    // 업데이트된 결과를 메시지에 다시 쓰기
    await Mvu.replaceMvuData(mvu_data, { type: 'message', message_id: getCurrentMessageId() });
    ```

=== "텍스트 내 업데이트 명령 파싱"
    ```javascript
    await waitGlobalInitialized('Mvu');

    const mvu_data = Mvu.getMvuData({ type: 'message', message_id: -1 });

    // 어딘가에서 얻은 텍스트 내 업데이트 명령을 구문 분석합니다.
    // 여기서는 텍스트를 가정했지만, 'generate' 등에서 가져올 수도 있습니다.
    const content = "<JSONPatch>생략</JSONPatch>";
    const new_data = await Mvu.parseMessage(content, mvu_data);

    await Mvu.replaceMvuData(new_data, { type: 'message', message_id: getCurrentMessageId() });
    ```

## 변수로 녹색 불(Green Light) 활성화하기

우리는 아오조라 리(青空莉)가 활성화 파트에서 언급한 "코드를 직접 작성하여 항목 활성화 제어" 방법 중 하나인 `injectPrompts`를 사용하여, 변수 값을 녹색 불을 활성화할 수 있는 사전 스캔 텍스트로 변환할 수 있습니다.

```javascript
await waitGlobalInitialized('Mvu');
eventOn(Mvu.events.VARIABLE_UPDATE_ENDED, variables => {
  // 현재 하쿠아 의존도 수치 가져오기
  const value = _.get(variables, 'stat_data.하쿠아.의존도');

  injectPrompts([
    {
      id: '활성화-하쿠아의존도',  // 여기서 id는 프롬프트의 고유 식별자입니다.
                             // 만약 나중에 동일한 id로 injectPrompts를 다시 실행하면, 이전 프롬프트가 대체됩니다.

      content: `하쿠아 의존도=${value}`,  // '하쿠아 의존도=xxx' 텍스트 주입, 녹색 불 활성화 용도로만 사용됨;
                                       // 이렇게 하면, 녹색 불 키워드에 '하쿠아 의존도=xxx'를 입력하여 활성화할 수 있습니다.

      position: 'none',
      depth: 0,
      role: 'user',
      should_scan: true,
    },
  ]);
});
```

이렇게 하면 현재 채팅 내에 항상 `하쿠아 의존도=xxx`와 같은 프롬프트가 존재하여 녹색 불 활성화에만 사용되며, 변수가 업데이트될 때마다 스크립트가 `injectPrompts`로 이를 갱신하여 항상 최신 수치를 유지하도록 보장합니다.

물론, 녹색 불 키워드 작성을 더 간단하게 하기 위해 `하쿠아 의존도` 대신 `하쿠아 단계 1`과 같이 직접 주입할 수도 있습니다:

```javascript
await waitGlobalInitialized('Mvu');
eventOn(Mvu.events.VARIABLE_UPDATE_ENDED, variables => {
  // 현재 하쿠아 의존도 수치 가져오기
  const value = _.get(variables, 'stat_data.하쿠아.의존도');

  let content = '하쿠아 단계 ';
  if (value < 20) {
    content += '1';
  } else if (value < 40) {
    content += '2';
  } else if (value < 60) {
    content += '3';
  } else if (value < 80) {
    content += '4';
  } else {
    content += '5';
  }

  injectPrompts([
    {
      id: '활성화-하쿠아의존도',
      content,  // '하쿠아 단계 1', '하쿠아 단계 2', ...
      position: 'none',
      depth: 0,
      role: 'user',
      should_scan: true,
    },
  ]);
});
```

