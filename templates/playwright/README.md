# Komponent: playwright

Komponent GitLab CI/CD instaluje zależności projektu i uruchamia testy
end-to-end za pomocą [Playwright](https://playwright.dev/). Publikuje wynik w
formacie JUnit, raport HTML oraz artefakty utworzone przez Playwright.

Testy mogą znajdować się w bieżącym projekcie albo w osobnym repozytorium
GitLab klonowanym przed wykonaniem joba.

## Użycie

```yaml
include:
  - component: $CI_SERVER_FQDN/dev.rachuna/pipelines/gitlab/components/bootstrap/_before_script@1.0.0
  - component: $CI_SERVER_FQDN/dev.rachuna/pipelines/gitlab/components/bootstrap/_after_script@1.0.0
  - component: $CI_SERVER_FQDN/dev.rachuna/pipelines/gitlab/components/nodejs/playwright@1.0.0
```

Komponent korzysta z ukrytych jobów `.before_script` i `.after_script`, dlatego
muszą być dostępne w konfiguracji pipeline. W pipeline produkcyjnym należy
wskazywać opublikowany tag SemVer komponentów.

Domyślnie tworzony jest job `playwright-e2e` w etapie `tests`.

## Działanie

Job wykonuje kolejno:

1. Opcjonalnie instaluje globalnie najnowszy pakiet `@playwright/cli`.
2. Opcjonalnie klonuje zewnętrzne repozytorium z testami.
3. Uruchamia `npm ci` w katalogu zawierającym testy.
4. Wykonuje skrypt przekazany przez `job-before-script`.
5. Uruchamia testy poleceniem:

   ```shell
   npx playwright test --reporter=list,junit,html
   ```

6. Wykonuje skrypty sekcji `after_script`.
7. Publikuje raporty i artefakty również wtedy, gdy testy zakończą się błędem.

## Inputy

| Input | Typ | Wartość domyślna | Opis |
|---|---|---|---|
| `job-name` | `string` | `playwright-e2e` | Nazwa joba uruchamiającego testy Playwright. |
| `job-stage` | `string` | `tests` | Etap pipeline, w którym zostanie uruchomiony job. |
| `job-image` | `string` | `registry.gitlab.com/dev.rachuna/artifacts/containers/playwright:1.0.0` | Obraz zawierający Node.js i przeglądarki wymagane przez Playwright. |
| `job-rules` | `array` | `[{ when: on_success }]` | Reguły określające warunki dodania joba do pipeline. |
| `job-needs` | `array` | `[]` | Lista zależności przekazywana do pola `needs`. |
| `job-resource-group` | `string` | `${CI_PIPELINE_ID}-${CI_JOB_NAME}` | Grupa zasobów ograniczająca równoległe wykonywanie joba. |
| `job-before-script` | `string` | `:` | Dodatkowy skrypt wykonywany po `npm ci`, przed testami. |
| `job-after-script` | `string` | `:` | Dodatkowy skrypt wykonywany w sekcji `after_script`. |
| `job-install-dependency` | `boolean` | `false` | Instaluje globalnie najnowszy pakiet `@playwright/cli` przed `npm ci`. |
| `job-external-repository-fullpath` | `string` | `""` | Ścieżka `namespace/projekt` repozytorium GitLab z testami; pusta wartość oznacza bieżący projekt. |
| `job-external-repository-ref` | `string` | `feat/init` | Gałąź, tag lub ref klonowany z zewnętrznego repozytorium. |

Dozwolone wartości `job-stage` to: `.pre`, `prepare`, `dependency`,
`validate`, `build`, `deployment`, `tests`, `publish` i `.post`.

## Konfiguracja

```yaml
include:
  - component: $CI_SERVER_FQDN/dev.rachuna/pipelines/gitlab/components/bootstrap/_before_script@1.0.0
  - component: $CI_SERVER_FQDN/dev.rachuna/pipelines/gitlab/components/bootstrap/_after_script@1.0.0
  - component: $CI_SERVER_FQDN/dev.rachuna/pipelines/gitlab/components/nodejs/playwright@1.0.0
    inputs:
      job-name: e2e
      job-stage: tests
      job-rules:
        - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      job-needs:
        - build
      job-before-script: echo "Przygotowanie testów"
      job-after-script: echo "Testy zakończone"
```

## Zewnętrzne repozytorium

Aby uruchomić testy z innego projektu GitLab, należy podać jego pełną ścieżkę
oraz ref. Repozytorium zostanie sklonowane do katalogu
`.playwright-external-repository`:

```yaml
include:
  - component: $CI_SERVER_FQDN/dev.rachuna/pipelines/gitlab/components/bootstrap/_before_script@1.0.0
  - component: $CI_SERVER_FQDN/dev.rachuna/pipelines/gitlab/components/bootstrap/_after_script@1.0.0
  - component: $CI_SERVER_FQDN/dev.rachuna/pipelines/gitlab/components/nodejs/playwright@1.0.0
    inputs:
      job-external-repository-fullpath: dev.rachuna/example/playwright-tests
      job-external-repository-ref: main
      job-install-dependency: true
```

Klonowanie prywatnego repozytorium wymaga tokenu akceptowanego przez `glab`.
Komponent bootstrap `_before_script` korzysta w tym celu ze zmiennej
`GITLAB_TOKEN`.

## Raporty i artefakty

Raport JUnit jest zapisywany w `test-results/results.xml` i przekazywany do
GitLab jako raport testów. Zmienna `PLAYWRIGHT_JUNIT_OUTPUT_FILE` ustawia tę
ścieżkę niezależnie od tego, czy testy są uruchamiane z bieżącego, czy z
zewnętrznego repozytorium.

Job publikuje z `when: always` następujące katalogi:

- `playwright-report/`
- `test-results/`
- `.playwright-external-repository/playwright-report/`
- `.playwright-external-repository/test-results/`

## Zmienne

| Zmienna | Pochodzenie | Opis |
|---|---|---|
| `PLAYWRIGHT_INSTALL_DEPENDENCY` | input `job-install-dependency` | Steruje instalacją globalnego pakietu `@playwright/cli`. |
| `PLAYWRIGHT_EXTERNAL_REPOSITORY_FULLPATH` | input `job-external-repository-fullpath` | Wskazuje repozytorium GitLab zawierające testy. |
| `PLAYWRIGHT_EXTERNAL_REPOSITORY_REF` | input `job-external-repository-ref` | Wskazuje ref klonowanego repozytorium. |
| `PLAYWRIGHT_JUNIT_OUTPUT_FILE` | komponent | Ustawia ścieżkę raportu JUnit na `${CI_PROJECT_DIR}/test-results/results.xml`. |
| `DOCS_MD_FILE_PATH` | komponent | Ścieżka do dokumentacji komponentu. |
| `GITLAB_CI_COMPONENTS_PATH` | komponent | Ścieżka repozytorium komponentów używana przez skrypty bootstrap. |
| `GITLAB_CI_COMPONENTS_VERSION` | komponent | Tag, gałąź lub SHA użytej wersji komponentu. |
