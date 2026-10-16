# WeatherApp Infrastructure

DEV ו־PROD מנוהלים באמצעות אותו Helm Chart, עם Release בשם `weatherapp` בכל Namespace.

## מבנה

- `k8s/helm/weatherapp/` — ה־Chart הפעיל לשתי הסביבות.
- `values.yaml` בתוך ה־Chart — הגדרות משותפות, כולל גרסאות Images.
- `values-dev.yaml` / `values-prod.yaml` — הגדרות שונות לכל סביבה.
- `k8s/cluster/` — תשתיות Cluster, בניהול נפרד מהאפליקציה.
- `k8s/namespace/` — הגדרות Namespaces.
- `dev-archive/` — Manifests היסטוריים; אין להחיל אותם על הסביבות הפעילות.

## הבדלים בין הסביבות

| הגדרה | DEV | PROD |
|---|---|---|
| Namespace | `dev` | `prod` |
| דומיין | `weatherapp.dev.local` | `weatherappproject.duckdns.org` |
| `certificate.enabled` | `false` | `true` |
| HTTPS | TLS Secret קיים | Certificate עם `letsencrypt-prod` |

cert-manager מותקן ברמת ה־Cluster. משאב Certificate נוצר רק ב־PROD.
שם TLS Secret בשתי הסביבות הוא `weatherapp-tls`.

## בדיקות לפני עדכון

כל הפקודות מתוך שורש repository זה. יש לבדוק כל סביבה בנפרד:

```bash
helm lint k8s/helm/weatherapp -f k8s/helm/weatherapp/values-dev.yaml
helm lint k8s/helm/weatherapp -f k8s/helm/weatherapp/values-prod.yaml
helm template weatherapp k8s/helm/weatherapp -n dev -f k8s/helm/weatherapp/values-dev.yaml
helm template weatherapp k8s/helm/weatherapp -n prod -f k8s/helm/weatherapp/values-prod.yaml
```

לסימולציית עדכון DEV מול ה־Cluster, ללא שינוי:

```bash
helm upgrade weatherapp k8s/helm/weatherapp -n dev \
  -f k8s/helm/weatherapp/values-dev.yaml --dry-run=server --hide-secret
```

## עדכון בפועל

הפקודה הבאה משנה את DEV. להריץ רק לאחר סקירת הפלט והחלטה לפרוס:

```bash
helm upgrade weatherapp k8s/helm/weatherapp -n dev \
  -f k8s/helm/weatherapp/values-dev.yaml --wait --timeout 5m
```

לעדכון PROD יש להחליף גם Namespace וגם קובץ Values ל־PROD ולבדוק בנפרד.
אין צורך ב־`--take-ownership` בעדכונים רגילים: אימוץ משאבי DEV כבר בוצע.
אין למחוק StatefulSet או PVC כחלק מעדכון רגיל.

## אימות אחרי עדכון

```bash
helm list -n dev
kubectl get pods,pvc -n dev
```

יש לבדוק גם פתיחת האתר וחיפוש מזג אוויר.

## Secrets והשלב הבא

ה־Chart משתמש ב־Secrets קיימים בכל Namespace:

- `postgres-secret`: המפתחות `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`.
- `backend-api-key`: המפתח `OPENWEATHER_API_KEY`.

קובצי Secrets ומפתחות פרטיים מקומיים מוחרגים מ־Git; קובצי הדוגמה נשארים במעקב.
אין להכניס ערכי Secrets ל־Values או ל־Git.

השלב הבא: Vault ו־External Secrets Operator. לשמר את שמות ה־Secrets והמפתחות
כדי שהאפליקציה תמשיך לצרוך אותם ללא שינוי בחוזה ההגדרות שלה.
