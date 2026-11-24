# Playwright E2E Testing mit OpenCode

Java · Quarkus · Jakarta EE · MyFaces · PrimeFaces · TypeScript

## 1. Architektur

OpenCode analysiert Java- und XHTML-Code, ein Browser-MCP untersucht die laufende Anwendung, und Playwright Test führt versionierte Tests unabhängig vom Agenten aus. AGENTS.md enthält Regeln, der Subagent definiert Rolle und Berechtigungen, der Skill den wiederverwendbaren Workflow.

## 2. Projektstruktur

```text
project-root/
├── .agents/skills/playwright-e2e/SKILL.md
├── .opencode/agents/e2e-tester.md
├── frontend/
│   ├── tests/e2e/
│   │   ├── pages/login.page.ts
│   │   ├── specs/login.spec.ts
│   │   ├── fixtures/
│   │   └── utils/
│   ├── playwright-report/
│   ├── test-results/
│   ├── .env.test
│   ├── .env.test.example
│   ├── .npmrc
│   ├── package.json
│   ├── package-lock.json
│   ├── tsconfig.e2e.json
│   └── playwright.config.ts
├── src/main/java/
├── src/main/resources/
├── src/main/webapp/
├── src/test/java/
├── .gitignore
├── AGENTS.md
├── opencode.json
└── pom.xml
```

## 3. Voraussetzungen und Installation

Eine von Playwright unterstützte Node.js-LTS-Version verwenden. Bestehendes npm-Projekt und .npmrc beibehalten.

```bash
node --version
npm --version
java --version
opencode --version
cd frontend
npm install -D @playwright/test typescript @types/node dotenv
npx playwright install chromium
npx playwright --version
cd ..
mkdir -p frontend/tests/e2e/{pages,specs,fixtures,utils}
mkdir -p .agents/skills/playwright-e2e .opencode/agents
```

Optional: `npx playwright install firefox webkit` im Frontend-Verzeichnis. Unter Linux/CI: `npx playwright install --with-deps chromium`.

## 4. Testumgebung

`frontend/.env.test.example`:

```dotenv
BASE_URL=http://localhost:8080
E2E_USERNAME=
E2E_PASSWORD=
E2E_LOGIN_PATH=/login.xhtml
E2E_SUCCESS_PATH=/dashboard.xhtml
```

```bash
cp frontend/.env.test.example frontend/.env.test
```

Nur dedizierte Testkonten verwenden. URLs, Meldungen und Locators sind Beispiele und müssen anhand der echten Anwendung geprüft werden. Keine Produktivsysteme verwenden.

## 5. Playwright-Konfiguration

`frontend/playwright.config.ts` (Beispiel für CommonJS; bei ESM Pfadauflösung anpassen):

```typescript
import { defineConfig, devices } from '@playwright/test';
import dotenv from 'dotenv';
import path from 'node:path';

dotenv.config({ path: path.resolve(__dirname, '.env.test') });

export default defineConfig({
  testDir: './tests/e2e/specs',
  outputDir: './test-results',
  fullyParallel: false,
  workers: 1,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 1 : 0,
  timeout: 30_000,
  expect: { timeout: 10_000 },
  reporter: [
    ['list'],
    ['html', { outputFolder: 'playwright-report', open: 'never' }],
    ['junit', { outputFile: 'test-results/junit.xml' }]
  ],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:8080',
    headless: true,
    actionTimeout: 10_000,
    navigationTimeout: 30_000,
    screenshot: 'only-on-failure',
    trace: 'retain-on-failure',
    video: 'retain-on-failure',
    ignoreHTTPSErrors: false
  },
  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }]
});
```

Erst bei isolierten Testdaten parallelisieren. Optional Chrome und Edge als zusätzliche Projekte mit `channel: 'chrome'` bzw. `channel: 'msedge'` und passenden Desktop-Geräteprofilen ergänzen; Browser müssen installiert sein.

## 6. TypeScript

`frontend/tsconfig.e2e.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "types": ["node"]
  },
  "include": ["playwright.config.ts", "tests/e2e/**/*.ts"]
}
```

Playwright transpiliert TypeScript; die separate Typprüfung bleibt sinnvoll. Bei ESM Imports entsprechend anpassen.

## 7. npm-Skripte

In die bestehenden `scripts` der `frontend/package.json` integrieren:

```json
{
  "scripts": {
    "test:e2e": "playwright test",
    "test:e2e:login": "playwright test login.spec.ts",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:headed": "playwright test --headed",
    "test:e2e:debug": "playwright test --debug",
    "test:e2e:report": "playwright show-report",
    "test:e2e:list": "playwright test --list",
    "test:e2e:failed": "playwright test --last-failed",
    "test:e2e:typecheck": "tsc -p tsconfig.e2e.json",
    "test:e2e:codegen": "playwright codegen"
  }
}
```

## 8. Gitignore

```gitignore
/frontend/node_modules/
/frontend/playwright-report/
/frontend/test-results/
/frontend/playwright/.auth/
/frontend/.env.test
/frontend/.env.test.local
!/frontend/.env.test.example
```

Konfigurationen, Tests, Skills und Lockfile versionieren. Traces und Videos können vertrauliche Daten enthalten.

## 9. JSF- und PrimeFaces-Locators

Naming Container erzeugen Client-IDs wie `loginForm:username`. Bevorzugt `getByRole`, `getByLabel` und `getByTestId` verwenden. Fallback: `page.locator('[id="loginForm:username"]')`. Optional JSF-Pass-through-Attribute wie `pt:data-testid="login-username"` mit passendem Namespace verwenden und gerenderten DOM prüfen.

PrimeFaces-AJAX kann DOM-Knoten ersetzen. Locators statt ElementHandles verwenden. Keine festen Wartezeiten (`waitForTimeout`); auf den fachlich erwarteten UI-Zustand mit Web-first Assertions warten. Auto-Waiting garantiert nicht den Abschluss aller AJAX-Requests.

## 10. Login Page Object

`frontend/tests/e2e/pages/login.page.ts`:

```typescript
import { type Page, type Locator } from '@playwright/test';

export class LoginPage {
  readonly username: Locator;
  readonly password: Locator;
  readonly submitButton: Locator;

  constructor(private readonly page: Page) {
    this.username = page.getByLabel('Benutzername', { exact: true });
    this.password = page.getByLabel('Passwort', { exact: true });
    this.submitButton = page.getByRole('button', { name: 'Anmelden', exact: true });
  }

  async goto(): Promise<void> {
    await this.page.goto(process.env.E2E_LOGIN_PATH || '/login.xhtml');
  }

  async login(username: string, password: string): Promise<void> {
    await this.username.fill(username);
    await this.password.fill(password);
    await this.submitButton.click();
  }
}
```

## 11. Login-Tests

`frontend/tests/e2e/specs/login.spec.ts`:

```typescript
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/login.page';

test.describe('Authentifizierung', () => {
  let login: LoginPage;
  test.beforeEach(async ({ page }) => {
    login = new LoginPage(page);
    await login.goto();
  });

  test('Login-Formular sichtbar', async () => {
    await expect(login.username).toBeVisible();
    await expect(login.password).toBeVisible();
    await expect(login.submitButton).toBeEnabled();
  });

  test('Gültige Zugangsdaten', async ({ page }) => {
    const { E2E_USERNAME, E2E_PASSWORD } = process.env;
    if (!E2E_USERNAME || !E2E_PASSWORD) throw new Error('Testzugangsdaten fehlen');
    await login.login(E2E_USERNAME, E2E_PASSWORD);
    await expect(page).toHaveURL(url => url.pathname === (process.env.E2E_SUCCESS_PATH || '/dashboard.xhtml'));
    await expect(page.getByRole('button', { name: 'Abmelden' })).toBeVisible();
  });

  test('Unbekannter Benutzer', async ({ page }) => {
    await login.login('nonexistent-e2e-user', 'invalid-password');
    await expect(page.getByText('Ungültige Zugangsdaten', { exact: true })).toBeVisible();
    await expect(login.username).toBeVisible();
  });

  test('Leere Eingabefelder', async ({ page }) => {
    await login.submitButton.click();
    await expect(page.getByText('Benutzername ist erforderlich', { exact: true })).toBeVisible();
    await expect(page.getByText('Passwort ist erforderlich', { exact: true })).toBeVisible();
  });
});
```

Bei nativer HTML5-Validierung stattdessen gegebenenfalls `validity` prüfen. Erwartete Meldungen und URLs anhand der Anforderungen verifizieren.

| ID | Szenario | Erwartung |
|---|---|---|
| LOGIN-01 | Formular | Sichtbar |
| LOGIN-02 | Gültige Daten | Authentifiziert |
| LOGIN-03 | Falsches Passwort | Abgewiesen |
| LOGIN-04 | Unbekannter Benutzer | Abgewiesen |
| LOGIN-05–07 | Leere Felder | Validierung |
| LOGIN-08 | Gesperrter Benutzer | Abgewiesen |
| LOGIN-09 | Geschützte URL ohne Login | Kein Zugriff |
| LOGIN-10 | Logout | Sitzung beendet |
| LOGIN-11 | Zugriff nach Logout | Kein Zugriff |
| LOGIN-12 | Abgelaufene Sitzung | Erneute Anmeldung |
| LOGIN-13 | Rollen | Berechtigungen eingehalten |

## 12. Ausführung

Quarkus im Root starten: `./mvnw quarkus:dev` (alternativ `mvn quarkus:dev`). In einem zweiten Terminal:

```bash
npm --prefix frontend run test:e2e:typecheck
npm --prefix frontend run test:e2e:login
npm --prefix frontend run test:e2e
npm --prefix frontend run test:e2e:ui
npm --prefix frontend run test:e2e:headed
npm --prefix frontend run test:e2e:debug
npm --prefix frontend run test:e2e:failed
npm --prefix frontend run test:e2e:list
npm --prefix frontend run test:e2e:report
npm --prefix frontend run test:e2e -- --project=chromium
npm --prefix frontend run test:e2e -- --grep 'Gültige Zugangsdaten'
```

Codegen: `npm --prefix frontend run test:e2e:codegen -- http://localhost:8080/login.xhtml`. Generierten Code überarbeiten; keine aufgezeichneten Geheimnisse übernehmen.

## 13. Playwright MCP

Optional zusätzlich zu vorhandenen Chrome-/Edge-MCP-Servern. In `opencode.json` integrieren:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "playwright": {
      "type": "local",
      "command": ["npx", "-y", "@playwright/mcp@latest"],
      "enabled": true
    }
  }
}
```

Mit `opencode mcp list` prüfen und OpenCode neu starten. Nach Erprobung konkrete Paketversion fixieren. Bei interner Registry beachten, dass die frontend/.npmrc im Root nicht automatisch gilt. MCP-Browser und Test-Runner-Kontexte sind getrennt.

## 14. AGENTS.md ergänzen

```markdown
## End-to-End Testing
- npm project: frontend/
- Tests: frontend/tests/e2e/
- Use the playwright-e2e skill.
- Analyze XHTML, backing beans and expected behavior first.
- Inspect the actual rendered DOM via browser MCP.
- Prefer accessible, stable locators; account for JSF naming containers and AJAX.
- Use web-first assertions; never use arbitrary fixed delays.
- Keep tests isolated; reuse page objects.
- Never use production systems or hardcoded credentials.
- Do not weaken assertions, hide failures or modify business logic without approval.
- Run npm --prefix frontend run test:e2e:typecheck and the relevant test suite.
- Report passed, failed, skipped and flaky tests, modified files and blockers.
- Never claim success without confirmed runner results.
```

## 15. Spezialisierter Agent

`.opencode/agents/e2e-tester.md`:

```markdown
---
description: Implements and analyzes Playwright E2E tests for Quarkus JSF PrimeFaces applications.
mode: subagent
temperature: 0.1
permission:
  edit: allow
  bash:
    "*": ask
    "npm --prefix frontend run test:e2e": allow
    "npm --prefix frontend run test:e2e:login": allow
    "npm --prefix frontend run test:e2e:typecheck": allow
---

You are a senior QA automation engineer specializing in Playwright and TypeScript.
Follow AGENTS.md and use the playwright-e2e skill.
Analyze source code and requirements, inspect the running application, implement page objects and tests, execute type checks and tests, classify failures, fix test defects and report application defects separately.
Never modify business logic without authorization. Never expose credentials or access production systems. Stop on missing test data or unavailable environments.
```

Berechtigungen sind keine vollständige Sandbox; npm-Skripte können beliebigen Code ausführen. Vor Freigabe prüfen.

## 16. Wiederverwendbarer Skill

`.agents/skills/playwright-e2e/SKILL.md`:

```markdown
---
name: playwright-e2e
description: Implement, execute and analyze Playwright E2E tests for Quarkus JSF PrimeFaces applications.
---
# Workflow
1. Discovery: Read AGENTS.md, existing tests, XHTML, backing beans and authentication logic.
2. Environment: Verify test URL, application readiness and test data without exposing credentials.
3. Browser inspection: Inspect accessibility tree, DOM, client IDs, validation and AJAX behavior. Never invent locators.
4. Planning: Define positive, negative, boundary, authorization and regression scenarios with expected outcomes.
5. Implementation: Reuse page objects, use stable locators and web-first assertions; keep tests isolated.
6. Type checking: npm --prefix frontend run test:e2e:typecheck
7. Execution: npm --prefix frontend run test:e2e:login or npm --prefix frontend run test:e2e
8. Analysis: Inspect assertions, console, network, screenshots, traces, videos and available application logs. Classify test, application, environment, data and flaky failures.
9. Verification: Fix test defects only; rerun up to three correction cycles. Do not weaken assertions or suppress failures.
10. Reporting: List environment, browser, passed/failed/skipped/flaky tests, defects, changed files, artifacts and remaining blockers.
```

## 17. Erster Agentenauftrag

OpenCode im Root starten und den Subagenten ansprechen:

```text
@e2e-tester
Verwende den playwright-e2e Skill. Analysiere die Login-XHTML-Dateien, Backing Beans und Authentifizierungslogik. Untersuche die laufende Testanwendung über vorhandene Browser-MCP-Tools. Ermittle tatsächliche Locators, Meldungen und Weiterleitungsziele. Vervollständige die Playwright-Infrastruktur im bestehenden frontend-Verzeichnis. Implementiere Formularanzeige, gültige und ungültige Anmeldung, Pflichtfeldvalidierung, Zugriffsschutz und Logout, soweit fachlich unterstützt. Führe Typprüfung und Login-Tests aus. Analysiere Fehler und korrigiere ausschließlich Testimplementierungsfehler. Berichte tatsächliche Ergebnisse und Anwendungsdefekte. Keine Änderungen an fachlicher Anwendungslogik ohne Zustimmung.
```

## 18. Authentifizierung für weitere Tests

Für andere Tests kann ein Setup-Projekt einen authentifizierten Zustand speichern:

```typescript
await page.context().storageState({ path: 'playwright/.auth/user.json' });
```

Ein separates authentifiziertes Projekt kann diese Datei über `use.storageState` laden. Setup als Projektabhängigkeit konfigurieren. Login-Tests bleiben unauthentifiziert. Bei serverseitigen JSF-Sessions parallele Wiederverwendung prüfen; Logout-Tests dürfen keine gemeinsam verwendete Sitzung zerstören. Session-Dateien niemals versionieren.

## 19. Optionaler automatischer Quarkus-Start

In `playwright.config.ts` ergänzen, sofern lokale Testumgebung und Maven Wrapper vorhanden sind:

```typescript
webServer: {
  command: './mvnw quarkus:dev',
  cwd: path.resolve(__dirname, '..'),
  url: process.env.BASE_URL || 'http://localhost:8080',
  reuseExistingServer: !process.env.CI,
  timeout: 180_000
}
```

Für CI kontrollierte Testdatenbank, geeignetes Quarkus-Profil und Betriebsbereitschaft sicherstellen.

## 20. CI/CD

```bash
npm --prefix frontend ci
(cd frontend && npx playwright install --with-deps chromium)
npm --prefix frontend run test:e2e:typecheck
# Testanwendung und Datenbank starten; auf Betriebsbereitschaft warten.
CI=true npm --prefix frontend run test:e2e
```

Zugangsdaten über CI-Secrets bereitstellen. HTML-Bericht, JUnit-XML, Screenshots, Traces und Videos als zugriffsgeschützte Artefakte sichern, auch bei Fehlschlägen.

## 21. Checkliste

- [ ] Playwright und Browser installieren
- [ ] Projektstruktur, Umgebungsvariablen und Gitignore anlegen
- [ ] Playwright- und TypeScript-Konfiguration erstellen
- [ ] npm-Skripte ergänzen
- [ ] MCP-Verbindung prüfen
- [ ] AGENTS.md, Subagent und Skill ergänzen
- [ ] Quarkus-Testumgebung starten
- [ ] Tatsächliche Login-Seite analysieren
- [ ] Page Object und Tests implementieren
- [ ] Typprüfung und Tests ausführen
- [ ] Ergebnisse und Fehler analysieren
- [ ] CI/CD integrieren

## 22. Offizielle Dokumentation

- [Playwright Installation](https://playwright.dev/docs/intro)
- [Konfiguration](https://playwright.dev/docs/test-configuration)
- [Best Practices](https://playwright.dev/docs/best-practices)
- [Locators](https://playwright.dev/docs/locators)
- [Auto-Waiting](https://playwright.dev/docs/actionability)
- [Page Objects](https://playwright.dev/docs/pom)
- [Authentication](https://playwright.dev/docs/auth)
- [Trace Viewer](https://playwright.dev/docs/trace-viewer)
- [UI Mode](https://playwright.dev/docs/test-ui-mode)
- [CI](https://playwright.dev/docs/ci)
- [Playwright MCP](https://github.com/microsoft/playwright-mcp)
- [OpenCode Agents](https://opencode.ai/docs/agents/)
- [OpenCode Skills](https://opencode.ai/docs/skills/)
- [OpenCode Rules](https://opencode.ai/docs/rules/)
- [OpenCode MCP](https://opencode.ai/docs/mcp-servers/)
- [OpenCode Permissions](https://opencode.ai/docs/permissions/)
- [Quarkus Testing](https://quarkus.io/guides/getting-started-testing)
- [Jakarta Faces](https://jakarta.ee/specifications/faces/)
- [PrimeFaces 13](https://primefaces.github.io/primefaces/13_0_0/)

Die Konfiguration ist eine Vorlage und wurde nicht gegen das tatsächliche Repository ausgeführt. Nächster Schritt: echte Login-Seite und Authentifizierungslogik prüfen.
