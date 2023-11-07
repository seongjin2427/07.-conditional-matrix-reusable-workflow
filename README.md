# 07. Conditional, with Matrix, Reusable Workflows

## Initial Workflow

1. `lint`

2. `test` ➡️ `build` ➡️ `deploy`

<br>

## First Case

`if`
> `test` 통과 ➡️ 테스트 레포트 출력 X
>
> `test` 실패 ➡️ 테스트 레포트 출력 O


### Commits

1. `test`가 실패하도록 고의적으로 에러를 만들어 봅니다. - `#68a841dd3e`
Process
- `src/components/MainContent.test.jsx`의 테스트 중 line 22의 `'help-area'`를 `'help-are'`로 `'a'`를 제거합니다.

Result
- `test` Job 실패 후, 그 이후의 `build`, `deploy`Job은 진행되지 않습니다.
- `lint` Job은 `test`와의 의존관계가 없어 병렬로 진행되어 워크플로우가 정상적으로 동작합니다.

2. `test` Job에 `if`필드를 통해 조건부로 실행하도록 Step을 제어해 봅니다 - `#081aa3cbe5`
Process
  - `Test code` Step에 id 값을 부여합니다. `id: run-tests`
  - 바로 다음 Step인 `Upload test report` Step에 `if` 필드를 기입합니다.
    - `if: steps.run-tests.outcome == 'failure`
    - 참고 (`steps` 컨텍스트, `==` 연산자)
      - [steps-context](https://docs.github.com/en/actions/learn-github-actions/contexts#steps-context)
      - [operator](https://docs.github.com/en/actions/learn-github-actions/expressions#operators) 