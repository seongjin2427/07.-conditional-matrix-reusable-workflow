# 07. Conditional, with Matrix, Reusable Workflows

## Initial Workflow

1. `lint`

2. `test` ➡️ `build` ➡️ `deploy`

<br>

## Control conditional Workflows 1

`if`
> `test` 통과 ➡️ 테스트 레포트 출력 X

> `test` 실패 ➡️ 테스트 레포트 출력 O


### Commits

1. `test`가 실패하도록 고의적으로 에러를 만들어 봅니다. - [`68a841dd`](https://github.com/seongjin2427/07.-conditional-matrix-reusable-workflow/commit/68a841dd3e392af541895984078db04707b1e495)

- Process
  - `src/components/MainContent.test.jsx`의 테스트 중 line 22의 `'help-area'`를 `'help-are'`로 `'a'`를 제거합니다.

- Result
  - `test` Job 실패 후, 그 이후의 `build`, `deploy`Job은 진행되지 않습니다.
  - `lint` Job은 `test`와의 의존관계가 없어 병렬로 진행되어 워크플로우가 정상적으로 동작합니다.

<br>

2. `test` Job에 `if`필드를 통해 조건부로 실행하도록 Step을 제어해 봅니다 - [`081aa3cb`](https://github.com/seongjin2427/07.-conditional-matrix-reusable-workflow/commit/081aa3cbe5c6064835126cca779c68a98a456b43)

- Process
  - `Test code` Step에 id 값을 부여합니다. `id: run-tests`
  - 바로 다음 Step인 `Upload test report` Step에 `if` 필드를 기입합니다.
    - `if: steps.run-tests.outcome == 'failure`
    - 참고 (`steps` 컨텍스트, `==` 연산자)
      - [steps-context](https://docs.github.com/en/actions/learn-github-actions/contexts#steps-context)
      - [operator](https://docs.github.com/en/actions/learn-github-actions/expressions#operators) 

- Result
  - 여전히 `test` Job 실패 후, 그 이후 Job들은 진행되지 않습니다.
  - 특정 Step이 실패하면 그 이후의 Step에 대해서는 진행하지 않는다는 GitHub Actions의 기본 동작이 그대로 유지되는 것을 확인할 수 있습니다.

<br>

3. `if` 필드의 조건에 상태 검사 함수 `failure()`를 `&&`연산자와 함께 작성해야 합니다. - [`c602ecd2`](https://github.com/seongjin2427/07.-conditional-matrix-reusable-workflow/commit/c602ecd258fabb4196e4ba107b5fa8a30681f968)

- Process
  - 상태 검사 함수 `failure()`를 추가합니다.
    - `if: failure() && steps.run-tests.outcome == 'failure'` 
    - 참고 (상태 검사 함수)
      - [status check functions](https://docs.github.com/en/actions/learn-github-actions/expressions#status-check-functions)

> `failure()`는 이전 Step 또는 Job들이 실패했을 때, `true`를 반환합니다.
> 만약, 알 수 없는 이유로 `Install dependencies` Step을 통한 패키지 설치가 실패한다면
> `failure()` 상태 함수는 `true`를 반환하게 되지만
> 그 이후의 `&& steps.run-tests.outcome == 'failure'`는 `false`가 되기 때문에
> `Upload test report` Step은 실행되지 않습니다.

- Result
  - `test` Job의 테스트가 실패해도 테스트 레포트를 출력하는 Step `Upload test report`는 정상적으로 작동됩니다.
  - 이후의 `build`, `deploy` Job은 이어서 진행되지 않습니다.

<br>

4. Job 레벨에서도 `if` 필드를 활용할 수 있습니다. - [efd94e02](https://github.com/seongjin2427/07.-conditional-matrix-reusable-workflow/commit/efd94e025f90a6b0a64e20505602f4cd371807ea)

- Proceess
  - `execution-flow.yml` 파일의 마지막에 `report` Job을 추가합니다.
  - `Output information` Step을 작성하고, `if` 필드로 `failure()` 상태함수를 추가합니다.
    - `if: failure()`

- Result
  - `report` Job은 `test` Job의 실패를 감지하지 못하고 그냥 `skip` 됩니다.

> `test`와는 별개로 `report`가 병렬적으로 진행되기 때문에
> 두 Job 사이의 의존관계가 존재하지 않아 `test`가 실패해도 `report`가 감지하지 못합니다.

<br>

5. Job 레벨에서의 실패를 서로 감지 할 수 있도록 의존 관계를 추가합니다. - [`bef4c911`](https://github.com/seongjin2427/07.-conditional-matrix-reusable-workflow/commit/bef4c911ba2c7f7d8d9ddaeee5d891b3097a41c5)

- Process
  - `report` Job에 `needs: [lint, deploy]` 필드를 추가합니다.
  - `lint` Job의 실패를 감지합니다.
  - `test` Job과 연결된 마지막 Job은 결국 `deploy`이기 때문에, `deploy` Job을 의존관계로 두면, `test`, `build`, `deploy` Job들 중 하나라도 실패하게 되면 감지하게 됩니다.

- Result
  - `test` Job이 실패해도 테스트 레포트는 아티팩트로 업로드 됩니다.
  - `deploy`, `deploy`는 skip 됩니다.
  - `test`가 실패했기 때문에 `report` Job이 동작합니다.

<br>

## Control conditional Workflows 2

`if`
> `actions/cache`를 사용한 캐시를 사용한다면 ➡️ 의존성 설치 X

> `actions/cache`를 사용한 캐시를 사용하지 않았다면 ➡️ 의존성 설치 O

### Commits

1. 캐싱할 의존성 폴더의 경로를 변경하고, `if` 필드로 캐시 사용 여부를 추가합니다. - [`44fc5c31`](https://github.com/seongjin2427/07.-conditional-matrix-reusable-workflow/commit/44fc5c312344a28ac9e7f64200e9c5ffa865075c)

- Process
  - `Cache dependencies` Step의 `path` 필드를 변경합니다.
    - `~/.npm` ➡️ `node_modules`
  - `if` 필드를 추가합니다.
    - `if: steps.cache.outputs.cache-hit != 'true'`
    - 참고 (`cache-hit` outputs)
      - [actions/cache](https://github.com/actions/cache#outputs)

- Result
  - `test` Job에서는 설치되었던 의존성이 `build` Job에서는 캐시가 사용되어 `Install dependencies` Step이 skip 됩니다.

<br>

## `continue-on-error`필드와 `if`필드의 차이점

### Commits

1. 기존의 `execution-flow.yml` 파일을 복사하여 `continue.yml` 파일을 만듭니다. - [`0f7f3119`](https://github.com/seongjin2427/07.-conditional-matrix-reusable-workflow/commit/0f7f31192eee4de8ada42bfc9b1bc2782d2766e8)

- Proccess
  - `MainContent.test.jsx` line 22의 'help-area'를 'help-are'로 변경하여 에러를 발생시킵니다.
  - `test` Job의 'Upload test report' Step `if`필드를 제거합니다.
  - 'Test code' Step에 `continue-on-error` 필드를 `true`로 지정합니다.

- Result
  - `if` 필드를 지정된 워크플로우는 에러가 발생한 이후의 Job들은 skip합니다.
  - `continue-on-error` 필드를 지정한 워크플로우는 이후의 Job들을 그대로 동작시킵니다.

---
## Matrix

> `matrix`는 지정한 키에 대한 값들이 배열이라면, 배열의 요소들에 대한 모든 경우의 수에 대해 Job을 병렬적으로 실행합니다.

1. `matrix.yml`을 생성하여 워크플로우를 정의하고 실행해봅니다. - [`6d870584`](https://github.com/seongjin2427/07.-conditional-matrix-reusable-workflow/commit/6d8705844dcc45c3ffed2620800a58d167b7dc99)

- Process
  - `build` Job 하위에 `strategy` 필드를 지정하고, `matrix`를 지정합니다.
  - `matrix` 하위에 원하는 키를 지정하고, 그에 대한 값을 배열로 지정합니다.
    - `operating-system: [ubuntu-latest, window-latest]`
    - `node-version: [12, 14, 16]`
  - 동적으로 실행할 Step에 `${{ }}` 구문을 통해 해당 `matrix` 및 키를 입력합니다.
    - `runs-on: ${{ matrix.operating-system }}`
    - `node-version: ${{ matrix-node-version }}`

- Result
  - `matrix`에 지정한 6가지 경우의 수에 대해서 Job이 병렬적으로 실행됩니다.
  - 특정 node-version에서 에러가 발생하여 나머지 Job들이 skip 됩니다.

<br>

2. Job들 중 하나라도 에러가 발생하면, 나머지는 skip되기 때문에 `continue-on-error`를 지정해줍니다. - [`bf18d4cc`](https://github.com/seongjin2427/07.-conditional-matrix-reusable-workflow/commit/bf18d4cc227a7e3bbfa9028f898d9135cb2bdf80)

- Process
  - `build` Job 레벨에서 `continue-on-error`를 `true`로 지정해줍니다.

- Result
  - Job들 중에서 에러가 발생하더라도, 그와 관계없이 각각의 Job들이 병렬적으로 실행됩니다.

<br>

3. 특정 조합을 포함하기 위한 `include`와 특정 조합을 제외시키기 위한 `exclude`를 사용해 봅니다. - [`6677bb43`](https://github.com/seongjin2427/07.-conditional-matrix-reusable-workflow/commit/6677bb43244806f017836d853c659e7de77f7911)

- Process
  - `matrix` 필드에서 `include`를 지정하고 그 하위에 포함시킬 조합을 지정합니다.
    - `include:`
    - `  - node-version: 18`
    - `  - operating-system: ubuntu-latest``
    - 기존에 지정하지 않은 키를 사용해도 됩니다.
      - `npm: npm`
  - `matrix` 필드에서 `exclude`를 지정하고, 그 하위에 제외시킬 조합을 지정합니다.
    - `exclude`
    - `  - node-version: 12`
    - `  - operating-system: windows-latest`

- Result
  - `node-version: 12`와 `operating-system: windows-latest`의 조합을 제외하고 실행됩니다.
  - `node-version: 18`과 `operating-system: ubuntu-latest` 조합이 포함되어 실행됩니다.