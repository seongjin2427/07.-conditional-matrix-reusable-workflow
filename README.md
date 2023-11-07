# 07. Conditional, with Matrix, Reusable Workflows

## Initial Workflow

1. `lint`

2. `test` ➡️ `build` ➡️ `deploy`

<br>

## Control conditional Workflows

`if`
> `test` 통과 ➡️ 테스트 레포트 출력 X
>
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

5. Job 레벨에서의 실패를 서로 감지 할 수 있도록 의존 관계를 추가합니다. - 

- Process
  - `report` Job에 `needs: [lint, deploy]` 필드를 추가합니다.
  - `lint` Job의 실패를 감지합니다.
  - `test` Job과 연결된 마지막 Job은 결국 `deploy`이기 때문에, `deploy` Job을 의존관계로 두면, `test`, `build`, `deploy` Job들 중 하나라도 실패하게 되면 감지하게 됩니다.