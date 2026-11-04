import os, json, zipfile, textwrap

base = "/mnt/data/fitai-github"
os.makedirs(base + "/server", exist_ok=True)

# Update package.json with server dependencies and scripts
package = {
  "name": "fitai-fitness-planner",
  "private": True,
  "version": "2.0.0",
  "type": "module",
  "scripts": {
    "dev": "concurrently \"vite\" \"node --watch server/index.js\"",
    "dev:client": "vite",
    "dev:server": "node --watch server/index.js",
    "build": "vite build",
    "start": "node server/index.js",
    "preview": "vite preview"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.4.7",
    "express": "^4.21.2",
    "lucide-react": "^0.468.0",
    "openai": "^4.77.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.4",
    "concurrently": "^9.1.2",
    "vite": "^6.0.5"
  }
}
with open(base + "/package.json", "w", encoding="utf-8") as f:
    json.dump(package, f, ensure_ascii=False, indent=2)

# Patch App.jsx: add loading state and replace createPlan generator
app_path = base + "/src/App.jsx"
with open(app_path, encoding="utf-8") as f:
    app = f.read()

app = app.replace('  const [error, setError] = useState("");',
                  '  const [error, setError] = useState("");\n  const [loading, setLoading] = useState(false);')

start = app.index('  function createPlan(event) {')
end = app.index('\n  function resetPlan()', start)
new_func = r'''  async function createPlan(event) {
    event.preventDefault();
    if (sortedDays.length < 2) {
      setError("Bitte wähle mindestens zwei Trainingstage aus.");
      return;
    }
    setError("");
    setLoading(true);
    setPlan(null);

    const payload = {
      goal,
      days: sortedDays,
      level,
      place,
      duration,
      equipment,
      experience,
      preferences: preferences.slice(0, 500),
      limitations: limitations.slice(0, 500)
    };

    try {
      const response = await fetch("/api/generate-plan", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(payload)
      });
      const data = await response.json();
      if (!response.ok) {
        throw new Error(data.error || "Der Plan konnte gerade nicht erstellt werden.");
      }
      setPlan({ ...data, createdAt: new Date().toLocaleDateString("de-DE") });
      setTimeout(() => document.getElementById("plan")?.scrollIntoView({ behavior: "smooth" }), 50);
    } catch (err) {
      setError(err.message || "Verbindung zum KI-Server fehlgeschlagen. Bitte prüfe, ob das Backend läuft.");
    } finally {
      setLoading(false);
    }
  }
'''
app = app[:start] + new_func + app[end:]

# Change plan display to use AI fields
old = '''<div className="plan-heading"><div><span className="section-kicker">DEIN ERGEBNIS</span><h2>Dein Wochenplan</h2><p>Erstellt am {plan.createdAt} · {goal} · {level}</p></div><span className="generated-badge"><Check size={14}/> Erstellt</span></div>'''
new = '''<div className="plan-heading"><div><span className="section-kicker">DEIN KI-ERGEBNIS</span><h2>Dein Wochenplan</h2><p>Erstellt am {plan.createdAt} · {goal} · {level}</p></div><span className="generated-badge"><Sparkles size={14}/> KI-Plan</span></div>
            {plan.introduction && <p className="ai-introduction">{plan.introduction}</p>}'''
app = app.replace(old, new)

app = app.replace('''{plan.workouts.map((workout, index) => (
                <article className="workout-card" key={workout.day}>
                  <div className="workout-top"><span className="workout-day">{workout.day}</span><span className="workout-index">SESSION 0{index + 1}</span></div>
                  <h3>{workout.title}</h3>
                  <ul>{workout.exercises.map(exercise => <li key={exercise}><span className="exercise-dot"></span>{exercise}</li>)}</ul>
                  <div className="workout-duration"><Timer size={14}/>{duration}</div>
                </article>
              ))}''', '''{plan.workouts.map((workout, index) => (
                <article className="workout-card" key={workout.day}>
                  <div className="workout-top"><span className="workout-day">{workout.day}</span><span className="workout-index">SESSION 0{index + 1}</span></div>
                  <h3>{workout.focus}</h3>
                  {workout.description && <p className="workout-description">{workout.description}</p>}
                  <ul>{workout.exercises.map((exercise, exerciseIndex) => (
                    <li key={`${workout.day}-${exerciseIndex}`}>
                      <span className="exercise-dot"></span>
                      <span><b>{exercise.name}</b>{exercise.details && <small className="exercise-details">{exercise.details}</small>}</span>
                    </li>
                  ))}</ul>
                  <div className="workout-duration"><Timer size={14}/>{workout.duration}</div>
                </article>
              ))}''')

app = app.replace('''<div className="recovery-icon"><Leaf size={22}/></div><h3>Erholungstage</h3><p>An den übrigen Tagen: Pause oder angenehme Alltagsbewegung. Erholung gehört zum Training.</p>''',
                  '''<div className="recovery-icon"><Leaf size={22}/></div><h3>Erholung & Balance</h3><p>{plan.recoveryAdvice || "An den übrigen Tagen: Pause oder angenehme Alltagsbewegung. Erholung gehört zum Training."}</p>''')
app = app.replace('''<div className="plan-disclaimer"><HeartPulse size={19}/><p><b>Gut zu wissen:</b> Dieser Plan ist ein allgemeiner Vorschlag und keine medizinische oder individuell geprüfte Trainingsberatung. Passe Übungen an deine Fähigkeiten an. Pausiere bei Schmerzen oder Unwohlsein.</p></div>''',
                  '''<div className="plan-disclaimer"><HeartPulse size={19}/><p><b>Gut zu wissen:</b> {plan.safetyNote || "Dieser Plan ist ein allgemeiner Vorschlag und keine medizinische Beratung. Passe Übungen an deine Fähigkeiten an und pausiere bei Schmerzen oder Unwohlsein."}</p></div>''')
app = app.replace('''<button className="primary-button" type="submit"><Sparkles size={18}/> Meinen Wochenplan erstellen</button>''',
                  '''<button className="primary-button" type="submit" disabled={loading}><Sparkles size={18}/> {loading ? "KI erstellt deinen Plan ..." : "Meinen KI-Wochenplan erstellen"}</button>''')
with open(app_path, "w", encoding="utf-8") as f:
    f.write(app)

# Add backend server using Responses API and strict JSON schema
server_js = r'''
import "dotenv/config";
import express from "express";
import cors from "cors";
import OpenAI from "openai";
import path from "node:path";
import { fileURLToPath } from "node:url";

const app = express();
const port = Number(process.env.PORT || 3001);
const __dirname = path.dirname(fileURLToPath(import.meta.url));

app.use(cors({ origin: process.env.CLIENT_ORIGIN || "http://localhost:5173" }));
app.use(express.json({ limit: "20kb" }));

const apiKey = process.env.OPENAI_API_KEY;
const openai = apiKey ? new OpenAI({ apiKey }) : null;

const allowedGoals = ["Allgemeine Fitness", "Kraft und Fitness", "Ausdauer", "Beweglichkeit"];
const allowedLevels = ["Anfänger", "Fortgeschritten", "Erfahren"];
const allowedPlaces = ["Fitnessstudio", "Zuhause", "Draußen"];
const allowedDurations = ["20–30 Minuten", "30–45 Minuten", "45–60 Minuten"];
const allowedEquipment = ["Hanteln", "Fitnessgeräte", "Widerstandsbänder", "Keine Geräte"];
const allowedDays = ["Montag", "Dienstag", "Mittwoch", "Donnerstag", "Freitag", "Samstag", "Sonntag"];

const planSchema = {
  type: "object",
  additionalProperties: false,
  properties: {
    introduction: { type: "string" },
    workouts: {
      type: "array",
      items: {
        type: "object",
        additionalProperties: false,
        properties: {
          day: { type: "string" },
          focus: { type: "string" },
          description: { type: "string" },
          duration: { type: "string" },
          exercises: {
            type: "array",
            items: {
              type: "object",
              additionalProperties: false,
              properties: {
                name: { type: "string" },
                details: { type: "string" }
              },
              required: ["name", "details"]
            }
          }
        },
        required: ["day", "focus", "description", "duration", "exercises"]
      }
    },
    recoveryAdvice: { type: "string" },
    safetyNote: { type: "string" }
  },
  required: ["introduction", "workouts", "recoveryAdvice", "safetyNote"]
};

function cleanText(value, max = 500) {
  return typeof value === "string" ? value.trim().slice(0, max) : "";
}

app.get("/api/health", (_req, res) => {
  res.json({ ok: true, aiConfigured: Boolean(openai) });
});

app.post("/api/generate-plan", async (req, res) => {
  if (!openai) {
    return res.status(503).json({
      error: "Der KI-Server ist noch nicht eingerichtet. Bitte OPENAI_API_KEY in der .env-Datei hinterlegen."
    });
  }

  const body = req.body || {};
  const days = Array.isArray(body.days)
    ? [...new Set(body.days)].filter(day => allowedDays.includes(day))
    : [];
  const equipment = Array.isArray(body.equipment)
    ? [...new Set(body.equipment)].filter(item => allowedEquipment.includes(item))
    : [];

  if (!allowedGoals.includes(body.goal) ||
      !allowedLevels.includes(body.level) ||
      !allowedPlaces.includes(body.place) ||
      !allowedDurations.includes(body.duration) ||
      days.length < 2 || days.length > 7) {
    return res.status(400).json({ error: "Bitte prüfe deine Angaben und wähle mindestens zwei gültige Trainingstage." });
  }

  const userProfile = {
    goal: body.goal,
    days,
    level: body.level,
    place: body.place,
    duration: body.duration,
    equipment,
    experience: cleanText(body.experience, 80),
    preferences: cleanText(body.preferences),
    limitations: cleanText(body.limitations)
  };

  try {
    const response = await openai.responses.create({
      model: process.env.OPENAI_MODEL || "gpt-5.5",
      instructions: [
        "Du bist FITAI, ein vorsichtiger und altersgerechter Assistent für allgemeine Bewegung und Fitness.",
        "Erstelle einen sicheren, ausgewogenen Wochenplan ausschließlich anhand der erlaubten Nutzerangaben.",
        "Gehe davon aus, dass die Person minderjährig sein könnte. Gib daher ausschließlich altersgerechte, nicht-extreme Vorschläge.",
        "Keine Gewichtsabnahme-, Kalorien-, Körperform- oder Aussehensziele. Keine Leistungsdruck-Sprache.",
        "Keine maximalen Gewichte, Maximalversuche, riskanten Übungen, gefährlichen Challenges oder Trainingsanweisungen bis zur Erschöpfung.",
        "Bevorzuge einfache, kontrollierte Bewegungen, leicht anpassbare Varianten, Aufwärmen und ausreichende Erholung.",
        "Keine Diagnosen und keine Behandlung oder Rehabilitation von Verletzungen. Wenn Einschränkungen, Schmerzen oder gesundheitliche Probleme genannt werden, weise freundlich darauf hin, vor dem Training eine qualifizierte Fachperson und eine erziehungsberechtigte Person einzubeziehen; gib keine Übungen zur Behandlung dieser Probleme vor.",
        "Nutze nur die ausgewählten Trainingstage. Jeder Trainingstag muss genau einmal vorkommen und in der Reihenfolge der Eingabe erscheinen.",
        "Die Ausstattung und der Trainingsort müssen berücksichtigt werden. Erfinde keine vorhandenen Geräte.",
        "Erkläre Übungen kurz und verständlich. Gib keine starren Gewichtsangaben vor.",
        "Antworte auf Deutsch und liefere ausschließlich Daten passend zum vorgegebenen JSON-Schema."
      ].join(" "),
      input: `Erstelle einen individuellen, ausgewogenen Wochenplan für dieses Profil:\n${JSON.stringify(userProfile)}`,
      text: {
        format: {
          type: "json_schema",
          name: "fitai_weekly_plan",
          strict: true,
          schema: planSchema
        }
      },
      max_output_tokens: 2500
    });

    if (!response.output_text) {
      return res.status(502).json({ error: "Die KI hat keinen Plan zurückgegeben. Bitte versuche es erneut." });
    }

    const plan = JSON.parse(response.output_text);
    const returnedDays = plan.workouts.map(workout => workout.day);
    if (JSON.stringify(returnedDays) !== JSON.stringify(days)) {
      return res.status(502).json({ error: "Der KI-Plan hatte nicht die erwarteten Trainingstage. Bitte versuche es erneut." });
    }

    return res.json(plan);
  } catch (error) {
    console.error("FITAI generation error:", error?.message || error);
    if (error?.status === 401) {
      return res.status(401).json({ error: "Der API-Schlüssel wurde nicht akzeptiert. Bitte prüfe OPENAI_API_KEY." });
    }
    if (error?.status === 429) {
      return res.status(429).json({ error: "Das API-Kontingent ist gerade nicht verfügbar. Bitte prüfe dein API-Konto oder versuche es später erneut." });
    }
    return res.status(500).json({ error: "Beim Erstellen des KI-Plans ist ein Fehler aufgetreten. Bitte versuche es später erneut." });
  }
});

if (process.env.NODE_ENV === "production") {
  const distPath = path.resolve(__dirname, "../dist");
  app.use(express.static(distPath));
  app.get("*", (_req, res) => res.sendFile(path.join(distPath, "index.html")));
}

app.listen(port, () => {
  console.log(`FITAI API läuft auf http://localhost:${port}`);
});
'''
with open(base + "/server/index.js", "w", encoding="utf-8") as f:
    f.write(textwrap.dedent(server_js).lstrip())

# Update Vite config with API proxy
vite_config = r'''
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      "/api": "http://localhost:3001"
    }
  }
});
'''
with open(base + "/vite.config.js", "w", encoding="utf-8") as f:
    f.write(textwrap.dedent(vite_config).lstrip())

# env template
with open(base + "/.env.example", "w", encoding="utf-8") as f:
    f.write("OPENAI_API_KEY=dein_openai_api_key_hier\nOPENAI_MODEL=gpt-5.5\nPORT=3001\nCLIENT_ORIGIN=http://localhost:5173\n")

# Extend CSS
css_path = base + "/src/style.css"
with open(css_path, encoding="utf-8") as f:
    css = f.read()
css += r'''
.primary-button:disabled{opacity:.7;cursor:wait;transform:none}
.ai-introduction{background:#edf5ef;border:1px solid #e0eee4;border-radius:12px;padding:14px 16px;color:#527360;font-size:13px;line-height:1.7;margin:0 0 16px}
.workout-description{font-size:11px;line-height:1.6;color:#819087;margin:-6px 0 10px}
.exercise-details{display:block;color:#829087;font-size:10px;line-height:1.5;margin-top:3px}
'''
with open(css_path, "w", encoding="utf-8") as f:
    f.write(css)

# Update README
readme = r'''
# FITAI – KI-Fitnessplan-Generator

Eine responsive React-/Vite-Webseite mit einem Express-Backend und echter OpenAI-API-Anbindung.

## Voraussetzungen

- Node.js 18 oder neuer
- npm
- Ein OpenAI API-Schlüssel mit verfügbarer API-Abrechnung / Nutzung

## Installation

```bash
npm install
```

## API-Schlüssel einrichten

1. Kopiere `.env.example` und benenne die Kopie in `.env` um.
2. Öffne `.env`.
3. Trage deinen Schlüssel ein:

```env
OPENAI_API_KEY=dein_echter_api_key
OPENAI_MODEL=gpt-5.5
PORT=3001
CLIENT_ORIGIN=http://localhost:5173
```

**Wichtig:** `.env` niemals auf GitHub hochladen oder mit anderen teilen. Die `.gitignore` schließt sie aus. Den Schlüssel nur im Backend verwenden, niemals in React-Code oder mit `VITE_`-Präfix.

## Lokal starten

```bash
npm run dev
```

- Frontend: `http://localhost:5173`
- API-Status: `http://localhost:3001/api/health`

Der Button „Meinen KI-Wochenplan erstellen“ sendet die Angaben an `/api/generate-plan`. Das Backend ruft die OpenAI Responses API auf und gibt strukturierte JSON-Daten für den Wochenplan zurück.

## Produktionsversion

```bash
npm run build
npm start
```

Setze auf dem Hosting-Anbieter `NODE_ENV=production` und hinterlege `OPENAI_API_KEY` als geheime Umgebungsvariable. Das Express-Backend liefert dann auch das gebaute Frontend aus.

## GitHub

```bash
git init
git add .
git commit -m "FITAI mit echter KI-Anbindung"
git branch -M main
git remote add origin https://github.com/DEIN-BENUTZERNAME/fitai-fitness-planner.git
git push -u origin main
```

**Nicht** `.env` hochladen. Auf dem Hosting-Anbieter den API-Schlüssel unter Environment Variables / Secrets eintragen.

## Deployment-Hinweis

Ein reines statisches Hosting wie GitHub Pages kann das Express-Backend nicht ausführen. Für die echte KI-Funktion brauchst du ein Hosting, das Node.js-Server unterstützt, z. B. Render, Railway oder einen vergleichbaren Anbieter. GitHub dient zum Speichern des Codes; der Server muss separat bereitgestellt werden.

## API und strukturierte Antworten

Die Integration nutzt die OpenAI Responses API und Structured Outputs mit JSON Schema, damit der Plan in einem festen Format zurückkommt:
- https://developers.openai.com/api/reference/resources/responses/methods/create
- https://developers.openai.com/api/docs/guides/structured-outputs

## Sicherheit

Die App ist so angelegt, dass sie auch für jüngere Nutzer geeignete, allgemeine Bewegungsvorschläge priorisiert. Sie gibt keine Gewichtsabnahme- oder Körperformziele vor, empfiehlt keine Maximalversuche und ersetzt keine medizinische Beratung. Bei Schmerzen oder gesundheitlichen Einschränkungen sollte vor dem Training fachlicher Rat eingeholt werden.
'''
with open(base + "/README.md", "w", encoding="utf-8") as f:
    f.write(textwrap.dedent(readme).lstrip())

# Ensure .gitignore includes .env
with open(base + "/.gitignore", "w", encoding="utf-8") as f:
    f.write("node_modules\ndist\n.env\n.env.*\n!.env.example\n.DS_Store\n")

# Zip refreshed project
zip_path = "/mnt/data/fitai-github-ki.zip"
with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as z:
    for root, _, filenames in os.walk(base):
        for filename in filenames:
            full = os.path.join(root, filename)
            arc = os.path.relpath(full, base)
            z.write(full, arc)

print("Erweiterte Projekt-ZIP erstellt:", zip_path)
print("Dateien:", sum(len(fs) for _, _, fs in os.walk(base)))

