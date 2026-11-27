# Playwright E2E Testing mit OpenCode

Java 21 · Quarkus 3 · Jakarta EE 10 · MyFaces · PrimeFaces 15 · Node.js 24 LTS · TypeScript

## Inhaltsverzeichnis

- [1. Architektur](#1-architektur)
- [2. Projektstruktur](#2-projektstruktur)
- [3. Voraussetzungen und Installation](#3-voraussetzungen-und-installation)
- [4. Testumgebung](#4-testumgebung)
- [5. Playwright-Konfiguration](#5-playwright-konfiguration)
- [6. TypeScript](#6-typescript)
- [7. Vollständige package.json](#7-vollständige-packagejson)
- [8. Gitignore](#8-gitignore)
- [9. JSF- und PrimeFaces-Locators](#9-jsf--und-primefaces-locators)
- [10. Login Page Object](#10-login-page-object)
- [11. Login-Tests](#11-login-tests)
- [12. Ausführung](#12-ausführung)
- [13. Playwright MCP](#13-playwright-mcp)
- [14. AGENTS.md ergänzen](#14-agentsmd-ergänzen)
- [15. Spezialisierter Agent](#15-spezialisierter-agent)
- [16. Wiederverwendbarer Skill](#16-wiederverwendbarer-skill)
- [17. Erster Agentenauftrag](#17-erster-agentenauftrag)
- [18. Authentifizierung für weitere Tests](#18-authentifizierung-für-weitere-tests)
- [19. Optionaler automatischer Quarkus-Start](#19-optionaler-automatischer-quarkus-start)
- [20. CI/CD](#20-cicd)
- [21. Checkliste](#21-checkliste)
- [22. Offizielle Dokumentation](#22-offizielle-dokumentation)

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

Node.js 24 LTS verwenden. Bestehendes npm-Projekt und .npmrc beibehalten. Optional `engine-strict=true` in `frontend/.npmrc` ergänzen, um die Node-Version bei npm-Installationen durchzusetzen.

```bash
node --version
npm --version
java --version
opencode --version
cd frontend
export PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1
npm install -D @playwright/test typescript @types/node@24
npx playwright --version
cd ..
mkdir -p frontend/tests/e2e/{pages,specs,fixtures,utils}
mkdir -p .agents/skills/playwright-e2e .opencode/agents
```

### Chromium als Standardbrowser (bereits installiert)

Die von Playwright verwaltete Chromium-Version ist bereits installiert. Keine erneute Installation bei jedem Testlauf. Prüfen:

```bash
cd frontend
npx playwright install --list
```

Nur falls eine zur lokalen Playwright-Version passende Browser-Version fehlt und Downloadzugriff freigegeben ist: `npx playwright install chromium`. Auf einem geeigneten Linux-CI-Runner bei Bedarf `npx playwright install --with-deps chromium`. Offline den passenden Browser-Cache und Systembibliotheken bereitstellen.

Chrome und Edge bleiben optionale Systembrowser. Ihre Standardinstallationspfade werden über die Playwright-Channels aufgelöst; abweichende Pfade können in .env.test gesetzt werden. Sie sind für Chromium-Tests nicht erforderlich.

### Bedeutung der npm-Pakete

| Paket | Aufgabe |
|---|---|
| `@playwright/test` | Test Runner, Browsersteuerung, Assertions und Reporting |
| `typescript` | Statische Typprüfung mit tsc |
| `@types/node@24` | Node.js-24-Typdefinitionen, etwa für process.env |

Node.js 24 unterstützt `.env`-Dateien nativ über `loadEnvFile`. Das Paket `dotenv` ist daher nicht erforderlich. Falls es bereits installiert ist und nicht anderweitig benötigt wird, kann es mit `npm uninstall dotenv` entfernt werden.

`-D` speichert die Pakete als devDependencies in der package.json. Die package-lock.json wird ebenfalls aktualisiert.

### Optionale globale Installation

```bash
npm install -g @playwright/test
playwright --version
```

Die globale CLI ist optional und ersetzt nicht die lokale Projektabhängigkeit. Für Projekttests stets die lokale Version über npm-Skripte verwenden. Browser-Binaries werden unter Linux standardmäßig in `~/.cache/ms-playwright/` gespeichert; kompatible Versionen können diesen Cache gemeinsam nutzen.

## 4. Testumgebung

`frontend/.env.test.example`:

```dotenv
BASE_URL=http://localhost:8080
# Optionaler Application Context Path; leer für Root-Anwendungen
BASE_APP_PATH=/meine-anwendung
E2E_ALLOWED_ORIGINS=http://localhost:8080,http://127.0.0.1:8080
E2E_USERNAME=
E2E_PASSWORD=
# Optional: abweichende absolute Browserpfade; leer = Standardinstallation
# Browserauswahl optional: chromium (Standard), chrome, edge, all
# E2E_BROWSER=chromium
EDGE_EXECUTABLE=
CHROME_EXECUTABLE=
E2E_LOGIN_PATH=/login.xhtml
E2E_SUCCESS_PATH=/dashboard.xhtml
```

```bash
npm --prefix frontend run test:e2e:setup
# Alternativ manuell:
cp -n frontend/.env.test.example frontend/.env.test
```

Das Setup legt `.env.test` nur an, wenn sie noch nicht existiert. npm-Skripte laufen im Verzeichnis der package.json, daher ohne `frontend/` im Kopierbefehl. Echte Werte nur in `.env.test` eintragen; bestehende Dateien werden nicht überschrieben oder mit der Vorlage synchronisiert. Die pre-Skripte führen das Setup automatisch aus. `loadEnvFile` lädt anschließend die Variablen.

`BASE_URL` enthält ausschließlich Schema, Host und optional Port. `BASE_APP_PATH` enthält den Application Context Path (z. B. `/meine-anwendung`); für eine Root-Anwendung leer lassen. Login- und Erfolgspfad sind relativ zur Anwendung. Die Konfiguration normalisiert den Context Path und erzeugt eine baseURL mit abschließendem Slash. Page Objects entfernen führende Slashes vor `page.goto`, damit der Context Path bei der URL-Auflösung erhalten bleibt. Beispiel: `http://localhost:8080/meine-anwendung/login.xhtml`. Die Origin-Allowlist enthält weiterhin nur Origins ohne Context Path.

Nur dedizierte Testkonten verwenden. URLs, Meldungen und Locators sind Beispiele und müssen anhand der echten Anwendung geprüft werden. Keine Produktivsysteme verwenden.

## 5. Playwright-Konfiguration

`frontend/playwright.config.ts` (Beispiel für CommonJS; bei ESM Pfadauflösung anpassen):

```typescript
import { defineConfig, devices } from '@playwright/test';
import { loadEnvFile } from 'node:process';
import { existsSync } from 'node:fs';
import path from 'node:path';

const envFile = path.resolve(__dirname, '.env.test');
if (!existsSync(envFile)) {
  throw new Error(`Missing ${envFile}. Run npm --prefix frontend run test:e2e:setup`);
}
loadEnvFile(envFile);

const origin = new URL(process.env.BASE_URL || 'http://localhost:8080');
const appPath = (process.env.BASE_APP_PATH || '').trim().replace(/^\/+|\/+$/g, '');
const baseURL = new URL(origin.origin);
baseURL.pathname = appPath ? `/${appPath}/` : '/';
const allowedOrigins = (process.env.E2E_ALLOWED_ORIGINS || 'http://localhost:8080,http://127.0.0.1:8080').split(',').map(value => value.trim());
if (!allowedOrigins.includes(baseURL.origin)) {
  throw new Error('E2E target is not an approved test origin');
}

// Chromium ist Standard. Weitere Browser nur bei expliziter Auswahl aktivieren.
const browser = process.env.E2E_BROWSER || 'chromium';
const projects = [
  { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  { name: 'chrome', use: { ...devices['Desktop Chrome'], channel: 'chrome',
    launchOptions: { executablePath: process.env.CHROME_EXECUTABLE || undefined } } },
  { name: 'edge', use: { ...devices['Desktop Edge'], channel: 'msedge',
    launchOptions: { executablePath: process.env.EDGE_EXECUTABLE || undefined } } }
];
if (browser !== 'all' && !projects.some(project => project.name === browser)) {
  throw new Error('Invalid E2E_BROWSER: ' + browser);
}

export default defineConfig({
  tsconfig: './tsconfig.e2e.json',
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
    baseURL: baseURL.href,
    headless: true,
    actionTimeout: 10_000,
    navigationTimeout: 30_000,
    screenshot: 'only-on-failure',
    trace: 'retain-on-failure',
    video: 'off',
    ignoreHTTPSErrors: false
  },
  projects: browser === 'all' ? projects : projects.filter(project => project.name === browser)
});
```

Erst bei isolierten Testdaten parallelisieren. Chromium nutzt die von Playwright verwaltete Binary ohne channel oder executablePath. Chrome und Edge bleiben optional und werden nur über E2E_BROWSER aktiviert. Fehlende optionale Browser blockieren Chromium nicht. Video bleibt offline deaktiviert, da FFmpeg zusätzlich benötigt werden kann. Die Origin-Allowlist ist keine Netzwerksandbox; externe Weiterleitungen bleiben möglich.

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

## 7. Vollständige package.json

Datei: `frontend/package.json`. Beispiel für das gesamte npm-Projekt. Bestehende Build-Skripte, Sass-Pfade und Abhängigkeiten beibehalten. Die Versionsnummern sind Beispiele, keine verifizierten aktuellen Versionen; die tatsächlichen Versionen durch npm install auflösen lassen. Chrome und Edge setzen entsprechende Playwright-Projekte und installierte Browser voraus.

```json
{
  "name": "frontend",
  "version": "1.0.0",
  "private": true,
  "description": "Frontend build and automated end-to-end testing",
  "engines": {
    "node": ">=24.0.0 <25.0.0"
  },
  "scripts": {
    "build:css": "sass --no-source-map scss:css",
    "build": "npm run build:css",
    "test:e2e:setup": "if [ ! -e .env.test ]; then cp .env.test.example .env.test; fi",
    "pretest:e2e": "npm run test:e2e:setup",
    "test:e2e": "playwright test",
    "pretest:e2e:ui": "npm run test:e2e:setup",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:report": "playwright show-report",
    "test:e2e:typecheck": "tsc -p tsconfig.e2e.json",
    "mcp:playwright": "playwright-mcp --browser chromium"
  },
  "devDependencies": {
    "@playwright/test": "^1.56.0",
    "@playwright/mcp": "*",
    "typescript": "^5.9.0",
    "@types/node": "^24.0.0",
    "sass": "^1.93.0",
    "clean-css-cli": "^5.6.3"
  }
}
```

Die fünf zentralen Skripte werden durch das automatische Setup ergänzt. CLI-Argumente werden dynamisch über `--` weitergegeben. Keine Browserinstallation oder Prüfung von Chrome/Edge bei jedem Testlauf.

Reproduzierbare Installation aus dem Root:

```bash
export PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1
npm --prefix frontend ci
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
    const loginPath = process.env.E2E_LOGIN_PATH;
    if (!loginPath) {
      throw new Error('E2E_LOGIN_PATH is not configured');
    }
    await this.page.goto(loginPath.replace(/^\/+/, ''));
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

  test('Gültige Zugangsdaten', async ({ page, baseURL }) => {
    const { E2E_USERNAME, E2E_PASSWORD } = process.env;
    if (!E2E_USERNAME || !E2E_PASSWORD) throw new Error('Testzugangsdaten fehlen');
    await login.login(E2E_USERNAME, E2E_PASSWORD);
    const successPath = (process.env.E2E_SUCCESS_PATH || '/dashboard.xhtml').replace(/^\/+/, '');
    const expectedURL = new URL(successPath, baseURL);
    await expect(page).toHaveURL(url => url.origin === expectedURL.origin && url.pathname === expectedURL.pathname);
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

### Ergänzende Login-Szenarien

Für falsche Passwörter ein dediziertes Testkonto ohne unbeabsichtigte Sperrfolgen verwenden. Geschützte URLs direkt in einem frischen Browserkontext aufrufen und sowohl Weiterleitung als auch fehlenden Zugriff prüfen. Logout anschließend durch erneuten Zugriff auf die geschützte Ressource verifizieren. Die Tabelle oben ist ein Testplan, nicht die Behauptung, dass alle Szenarien bereits implementiert sind.

### Fehlersuche: Umgebungsvariablen

Bei fehlendem `E2E_LOGIN_PATH` die tatsächliche `.env.test` prüfen. Nach `loadEnvFile(envFile)` temporär `console.log(envFile, existsSync(envFile));` und `console.log(process.env.E2E_LOGIN_PATH);` ergänzen. Keine Zugangsdaten protokollieren. Für die Browserauswahl E2E_BROWSER im Shell-Prozess setzen und keinen widersprechenden Wert in .env.test hinterlegen. Nach Änderungen den UI-Modus vollständig neu starten. Page Objects greifen direkt auf `process.env` zu; dotenv ist nicht erforderlich. Der Login-Pfad hat bewusst keinen stillen Fallback.

## 12. Ausführung

Quarkus im Root starten: `./mvnw quarkus:dev` (alternativ `mvn quarkus:dev`). In einem zweiten Terminal:

```bash
npm --prefix frontend run test:e2e:setup
npm --prefix frontend run test:e2e:typecheck
npm --prefix frontend run test:e2e -- login.spec.ts
npm --prefix frontend run test:e2e
npm --prefix frontend run test:e2e:ui
npm --prefix frontend run test:e2e -- --headed
npm --prefix frontend run test:e2e -- --debug
npm --prefix frontend run test:e2e -- --last-failed
npm --prefix frontend run test:e2e -- --list
npm --prefix frontend run test:e2e:report
E2E_BROWSER=edge npm --prefix frontend run test:e2e
E2E_BROWSER=chrome npm --prefix frontend run test:e2e
E2E_BROWSER=all npm --prefix frontend run test:e2e
npm --prefix frontend run test:e2e -- --grep 'Gültige Zugangsdaten'
```

Codegen: `cd frontend && npx playwright codegen --browser chromium http://localhost:8080/meine-anwendung/login.xhtml` (bei leerem BASE_APP_PATH ohne `/meine-anwendung`). Generierten Code überarbeiten; keine aufgezeichneten Geheimnisse übernehmen.

## 13. Playwright MCP

Optional zusätzlich zu vorhandenen Chrome-/Edge-MCP-Servern. Für reproduzierbare Installationen lokal installieren:

```bash
cd frontend
npm install -D --save-exact @playwright/mcp
npm run mcp:playwright -- --help
```

Die Beispielversion `*` in der package.json ist durch die von npm gespeicherte konkrete Version zu ersetzen. In `opencode.json` integrieren:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "playwright": {
      "type": "local",
      "command": ["npm", "--prefix", "frontend", "--silent", "run", "mcp:playwright"],
      "enabled": true
    }
  }
}
```

Mit `opencode mcp list` prüfen und OpenCode neu starten. OpenCode aus dem Projekt-Root starten. Der lokale npm-Skriptaufruf nutzt den Paket-Binäreintrag statt eines fest codierten internen Dateipfads. Die Installation erfolgt im Frontend mit dessen .npmrc. MCP-Browser und Test-Runner-Kontexte sind getrennt. Der MCP verwendet standardmäßig Chromium. Für optionale Chrome-/Edge-MCP-Server separate Konfigurationen mit passendem Browser und gegebenenfalls executable-path verwenden. Die verfügbaren Optionen mit dem lokal installierten MCP-Paket über `npm run mcp:playwright -- --help` prüfen. Keine Browserinstallation durch den Agenten auslösen.

## 14. AGENTS.md ergänzen

```markdown
## End-to-End Testing
- Stack: Java 21, Quarkus 3, Jakarta EE 10, MyFaces, PrimeFaces 15, TypeScript.
- npm project: frontend/
- Tests: frontend/tests/e2e/
- Use the playwright-e2e skill.
- Use installed Playwright-managed Chromium by default for tests and MCP. Chrome and Edge are optional and require explicit selection via E2E_BROWSER. Never install browsers without confirmed download access and explicit approval.
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
description: Implements and analyzes Playwright E2E tests for Quarkus JSF PrimeFaces 15 applications.
mode: subagent
temperature: 0.1
permission:
  edit:
    "*": ask
    "frontend/tests/e2e/**": allow
    "frontend/playwright.config.ts": allow
    "frontend/tsconfig.e2e.json": allow
    "frontend/package.json": ask
    "frontend/package-lock.json": ask
    "src/**": deny
    "**/.env.test": deny
  read:
    "*": allow
    "**/.env.test": deny
    "**/.env.test.local": deny
    "**/playwright/.auth/**": deny
  bash:
    "*": ask
    "npm --prefix frontend run test:e2e": allow
    "npm --prefix frontend run test:e2e -- login.spec.ts": allow
    "npm --prefix frontend run test:e2e:typecheck": allow
---

You are a senior QA automation engineer specializing in Playwright and TypeScript.
Follow AGENTS.md and use the playwright-e2e skill.
Analyze source code and requirements, inspect the running application, implement page objects and tests, execute type checks and tests, classify failures, fix test defects and report application defects separately.
Never modify business logic without authorization. Never expose credentials or access production systems. Stop on missing test data or unavailable environments.
```

Die Regeln müssen mit der eingesetzten OpenCode-Version geprüft werden. Zugangsdaten ausschließlich durch den Runner laden; auch Browser-MCP, Bash und Artefakte können Geheimnisse offenlegen. Berechtigungen sind keine vollständige Sandbox; npm-Skripte können beliebigen Code ausführen. Vor Freigabe prüfen.

## 16. Wiederverwendbarer Skill

`.agents/skills/playwright-e2e/SKILL.md`:

```markdown
---
name: playwright-e2e
description: Implement, execute and analyze Playwright E2E tests for Quarkus JSF PrimeFaces 15 applications.
---
# Workflow
1. Discovery: Read AGENTS.md, existing tests, XHTML, backing beans and authentication logic.
2. Environment: Verify test URL, application readiness, installed Chromium and test data without exposing credentials. Use Chromium unless explicitly requested otherwise.
3. Browser inspection: Inspect accessibility tree, DOM, client IDs, validation and AJAX behavior. Never invent locators.
4. Planning: Define positive, negative, boundary, authorization and regression scenarios with expected outcomes.
5. Implementation: Reuse page objects, use stable locators and web-first assertions; keep tests isolated.
6. Type checking: npm --prefix frontend run test:e2e:typecheck
7. Execution: npm --prefix frontend run test:e2e -- login.spec.ts or npm --prefix frontend run test:e2e
8. Analysis: Inspect assertions, console, network, screenshots, traces, videos and available application logs. Classify test, application, environment, data and flaky failures.
9. Verification: Fix test defects only; rerun up to three correction cycles. Do not weaken assertions or suppress failures.
10. Reporting: List environment, browser, passed/failed/skipped/flaky tests, defects, changed files, artifacts and remaining blockers.
```

### Optionale Agentenaufteilung: Planner, Generator, Healer

Playwright dokumentiert drei spezialisierte Testagenten. Für OpenCode können deren Rollen als eigene Subagenten umgesetzt werden; die offiziellen Definitionen nicht ungeprüft übernehmen.

| Rolle | Aufgabe | Rechte |
|---|---|---|
| Planner | DOM und Anforderungen analysieren, Testplan erstellen | Lesen, Browser-Inspektion |
| Generator | Testplan in Page Objects und Specs umsetzen | Testdateien bearbeiten, Tests ausführen |
| Healer | Fehler klassifizieren und Testdefekte korrigieren | Testdateien bearbeiten, begrenzte Wiederholungen |

Zunächst genügt der e2e-tester als kombinierter Agent. Bei einer Aufteilung bleiben fachliche Assertions verbindlich; der Healer darf keine Anwendungsfehler kaschieren. Siehe [Playwright Test Agents](https://playwright.dev/docs/test-agents).

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
  url: baseURL.href, // aus Abschnitt 5; Anwendung inklusive Context Path
  reuseExistingServer: !process.env.CI,
  timeout: 180_000
}
```

Für CI kontrollierte Testdatenbank, geeignetes Quarkus-Profil und Betriebsbereitschaft sicherstellen.

## 20. CI/CD

```bash
npm --prefix frontend ci
npm --prefix frontend run test:e2e:typecheck
# Testanwendung und Datenbank starten; auf Betriebsbereitschaft warten.
CI=true npm --prefix frontend run test:e2e
```

Zugangsdaten über CI-Secrets bereitstellen. HTML-Bericht, JUnit-XML, Screenshots, Traces und Videos als zugriffsgeschützte Artefakte sichern, auch bei Fehlschlägen.

### GitLab CI – Beispiel

```yaml
e2e:
  stage: test
  script:
    - export PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1
    - npm --prefix frontend ci
    - npm --prefix frontend run test:e2e:typecheck
    - npm --prefix frontend run test:e2e
  artifacts:
    when: always
    expire_in: 7 days
    paths:
      - frontend/playwright-report/
      - frontend/test-results/
    reports:
      junit: frontend/test-results/junit.xml
```

Voraussetzung: Linux-Runner mit Node.js 24 und passendem Playwright-Chromium im Browser-Cache und vorhandenen Browser-Systemabhängigkeiten sowie eine bereits gestartete, erreichbare Testanwendung mit Testdatenbank. BASE_URL, BASE_APP_PATH, E2E_ALLOWED_ORIGINS und Zugangsdaten über geschützte CI-Variablen setzen. Artefaktzugriff einschränken.

### Weitere Testarten

- DataTable: Filter, Sortierung, Pagination und AJAX-Updates über sichtbare Zeilen und fachliche Ergebnisse prüfen; keine festen Wartezeiten.
- Accessibility: optional `@axe-core/playwright` installieren und mit AxeBuilder prüfen; automatisierte Prüfungen ersetzen keine manuelle Barrierefreiheitsprüfung.
- API: Playwrights `request`-Fixture für dedizierte Testendpunkte verwenden; Status und Antwortinhalt prüfen.
- Visual Regression: `expect(page).toHaveScreenshot()` nur mit stabiler Browser-/OS-Umgebung und kontrollierten Testdaten verwenden; Baselines bewusst reviewen.

## 21. Checkliste

- [ ] Playwright-Pakete und vorhandenen Chromium-Cache prüfen
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
- [PrimeFaces 15](https://primefaces.github.io/primefaces/15_0_0/)

Die Konfiguration ist eine Vorlage und wurde nicht gegen das tatsächliche Repository ausgeführt. Nächster Schritt: echte Login-Seite und Authentifizierungslogik prüfen.

- npm-Referenz: https://docs.npmjs.com/cli/commands/npm-install
- Browser-Verwaltung: https://playwright.dev/docs/browsers

- [npm – Pre- und Post-Skripte](https://docs.npmjs.com/cli/v11/using-npm/scripts#pre--post-scripts)
- [GNU Coreutils – cp](https://www.gnu.org/software/coreutils/manual/html_node/cp-invocation.html)
- [Node.js 24 – loadEnvFile](https://nodejs.org/docs/latest-v24.x/api/process.html#processloadenvfilepath)
- [npm – engines](https://docs.npmjs.com/cli/configuring-npm/package-json#engines)
- [npm – engine-strict](https://docs.npmjs.com/cli/using-npm/config#engine-strict)
