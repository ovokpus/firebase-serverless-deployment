# firebase-serverless-deployment
Create and deploy a frontend solution using a Rest API and Firestore database.

Here’s a consolidated playbook you can drop into your repo’s **README.md**. It walks through each lab task step-by-step, explains the concepts behind them, and details the commands you need.

---

## Summary

This playbook guides you through building a fully serverless Netflix dataset viewer using Firebase and Google Cloud. You’ll provision a Firestore **Native Mode** database in `us-west1`, bulk-import a Netflix CSV using a Node.js script, stand up a REST API on Cloud Run, iterate on that API to query Firestore, and deploy both staging and production frontends—all with one-click builds via Cloud Build. Along the way you’ll leverage managed services for scalability, pay-as-you-go cost control, and minimal operational overhead ([firebase.google.com][1], [cloud.google.com][2]).

---

## Business Use Case

Users—data analysts, product teams, or curious Netflix subscribers—need a lightweight, scalable way to explore and query the entire Netflix titles dataset without managing servers or databases. By using Firestore for storage, Cloud Run for our API, and a static single-page app for the frontend, we deliver:

* **Real-time querying** of over 5,000 records via HTTP endpoints.
* **Automatic scaling** to handle spikes in traffic without provisioning VMs.
* **Cost efficiency** since you only pay for what you use.
* **Rapid iteration** through containerized revisions and one-command deployments ([cloud.google.com][3], [cloud.google.com][2]).

---

## Prerequisites

* **gcloud CLI** authenticated to your Qwiklabs project with Owner/Editor roles.
* **Google Cloud Build**, **Cloud Run**, and **Firestore APIs** enabled.
* **Node.js (v14+)** for running the import and API scripts.
* **Docker** installed (if you choose to build images locally).
* **Service Account** with Firestore Admin privileges and a JSON key for the import script ([firebase.google.com][4], [cloud.google.com][5]).

---

## Architecture Overview

```
CSV (netflix_titles_original.csv)
        ↓
Node.js Import Script → Firestore (us-west1, Native Mode)
        ↓
Dockerized REST API (v0.1 → v0.2) → Cloud Run (netflix-dataset-service)
        ↓
Containerized Frontend (staging & production) → Cloud Run services
```

All components live in **us-west1** to minimize latency and avoid cross-region costs ([cloud.google.com][6]).

---

## Task 1: Create the Firestore Database

1. **Via gcloud**

   ```bash
   gcloud app create --region=us-west1
   gcloud firestore databases create \
     --location=us-west1 \
     --type=firestore-native
   ```

   * App Engine initialization is required before Firestore creation ([stackoverflow.com][7]).
   * `--type=firestore-native` ensures **Native Mode** (for mobile/web SDKs) rather than Datastore mode ([cloud.google.com][3]).

2. **Via Console**

   * In the Firebase/GCP Console, go to **Firestore** → **Create database**.
   * Choose **Native Mode**, select **us-west1**, and hit **Create** ([cloud.google.com][6]).

> **Concept**: Native Mode provides real-time synchronization, offline support, and mobile/web SDK upgrades. Datastore mode is for legacy App Engine workloads.

---

## Task 2: Populate the Database

1. **Install dependencies**

   ```bash
   cd pet-theory/lab06/firebase-import-csv/solution
   npm install
   ```

   * **firebase-admin**: Firestore Admin SDK for server-side writes.
   * **csvtojson**: Converts CSV rows to JSON objects ([medium.com][8], [npmjs.com][9]).

2. **Run the import**

   ```bash
   node index.js netflix_titles_original.csv
   ```

   * The script batches up to 500 writes per Firestore batch for throughput ([fireship.io][10]).
   * Each row becomes a document in the `data` collection, keyed by `show_id` or auto-ID.

3. **Verify**

   * Open **Firestore** in the console and inspect the `data` collection. You should see documents like `70234439` with fields (`title`, `release_year`, etc.) ([firebase.google.com][1]).

> **Concept**: Bulk imports should batch writes to respect Firestore limits (500 operations/batch) and avoid hot keys.

---

## Task 3: Create the REST API (v0.1)

1. **Build & push**

   ```bash
   cd pet-theory/lab06/firebase-rest-api/solution-01
   gcloud builds submit \
     --tag gcr.io/$PROJECT_ID/rest-api:0.1
   ```

   Cloud Build packages the Dockerfile and pushes `rest-api:0.1` to Container Registry ([cloud.google.com][5]).

2. **Deploy to Cloud Run**

   ```bash
   gcloud run deploy netflix-dataset-service \
     --image gcr.io/$PROJECT_ID/rest-api:0.1 \
     --platform managed \
     --region us-west1 \
     --allow-unauthenticated \
     --max-instances=1
   ```

   * `--allow-unauthenticated` grants public access.
   * `--max-instances=1` prevents autoscaling beyond one container ([cloud.google.com][2], [cloud.google.com][11]).

3. **Test**

   ```bash
   SERVICE_URL=$(gcloud run services describe netflix-dataset-service \
     --platform managed \
     --region us-west1 \
     --format 'value(status.url)')
   curl -X GET $SERVICE_URL
   ```

   Should return:

   ````json
   {"status":"Netflix Dataset! Make a query."}
   ``` :contentReference[oaicite:12]{index=12}.

   ````

> **Concept**: Cloud Run provides fully managed containers with instant scale-to-zero and built-in HTTPS.

---

## Task 4: Firestore API Access (v0.2)

1. **Build the updated API**

   ```bash
   cd pet-theory/lab06/firebase-rest-api/solution-02
   gcloud builds submit \
     --tag gcr.io/$PROJECT_ID/rest-api:0.2
   ```

   The new code initializes Firestore Admin SDK and implements `GET /:year` to query `release_year` ([firebase.google.com][4]).

2. **Deploy revision 0.2**

   ````bash
   gcloud run deploy netflix-dataset-service \
     --image gcr.io/$PROJECT_ID/rest-api:0.2 \
     --platform managed \
     --region us-west1 \
     --allow-unauthenticated \
     --max-instances=1
   ``` :contentReference[oaicite:14]{index=14}.

   ````

3. **Verify filtering**

   ```bash
   curl -X GET $SERVICE_URL/2019
   ```

   Returns an array of all titles released in 2019.

> **Concept**: Cloud Run automatically routes traffic to the new revision; old revisions remain for rollback.

---

## Task 5: Deploy the Staging Frontend

1. **Build & push**

   ````bash
   cd pet-theory/lab06/firebase-frontend
   gcloud builds submit \
     --tag gcr.io/$PROJECT_ID/frontend-staging:0.1
   ``` :contentReference[oaicite:15]{index=15}.

   ````
2. **Deploy to Cloud Run**

   ````bash
   gcloud run deploy frontend-staging-service \
     --image gcr.io/$PROJECT_ID/frontend-staging:0.1 \
     --platform managed \
     --region us-west1 \
     --allow-unauthenticated \
     --max-instances=1
   ``` :contentReference[oaicite:16]{index=16}.

   ````
3. **Verify**

   * Open the staging URL; you’ll see a demo dataset UI pulling from the API.

> **Concept**: Staging allows you to test UI/UX before flipping production.

---

## Task 6: Deploy the Production Frontend

1. **Update API URL in `app.js`**

   ```js
   const API_URL = 'https://<your-production-url>/';
   fetch(`${API_URL}${selectedYear}`)
   ```

   Ensures the year query is appended correctly ([stackoverflow.com][12]).

2. **Build & push**

   ````bash
   cd pet-theory/lab06/firebase-frontend/public
   # edit app.js, then:
   cd ..
   gcloud builds submit \
     --tag gcr.io/$PROJECT_ID/frontend-production:0.1
   ``` :contentReference[oaicite:18]{index=18}.

   ````

3. **Deploy**

   ````bash
   gcloud run deploy frontend-production-service \
     --image gcr.io/$PROJECT_ID/frontend-production:0.1 \
     --platform managed \
     --region us-west1 \
     --allow-unauthenticated \
     --max-instances=1
   ``` :contentReference[oaicite:19]{index=19}.

   ````

4. **Verify**

   * Visit the production URL to see live data from Firestore.

> **Concept**: Keeping staging and production as separate services enables safe rollouts and canary testing.

---

## Next Steps

* **Add Firebase Hosting** with CDN and custom domain for your frontend.
* **Enable IAM policies** on Cloud Run to lock down production.
* **Implement monitoring & alerts** (Stackdriver Logging/Monitoring).
* **Extend API** with additional query parameters (director, country, rating).


[1]: https://firebase.google.com/docs/firestore/quickstart?utm_source=chatgpt.com "Get started with Cloud Firestore - Firebase - Google"
[2]: https://cloud.google.com/run/docs/authenticating/public?utm_source=chatgpt.com "Allowing public (unauthenticated) access | Cloud Run Documentation"
[3]: https://cloud.google.com/firestore/native/docs/manage-databases?utm_source=chatgpt.com "Create and manage databases | Firestore in Native mode"
[4]: https://firebase.google.com/docs/admin/setup?utm_source=chatgpt.com "Add the Firebase Admin SDK to your server"
[5]: https://cloud.google.com/sdk/gcloud/reference/builds/submit?utm_source=chatgpt.com "gcloud builds submit | Google Cloud SDK Documentation"
[6]: https://cloud.google.com/firestore/native/docs/locations?utm_source=chatgpt.com "Locations | Firestore in Native mode - Google Cloud"
[7]: https://stackoverflow.com/questions/55885918/how-to-enable-cloud-firestore-native-mode-using-api-command-line?utm_source=chatgpt.com "How to enable Cloud Firestore Native Mode using API / command ..."
[8]: https://medium.com/%40opeoluborode_9605/import-data-from-a-csv-file-into-cloud-firestore-with-node-js-e4faae454696?utm_source=chatgpt.com "Import data from a CSV file into Cloud Firestore with Node JS"
[9]: https://www.npmjs.com/package/csvtojson?utm_source=chatgpt.com "csvtojson - NPM"
[10]: https://fireship.io/lessons/import-csv-json-or-excel-to-firestore/?utm_source=chatgpt.com "Tutorial: CSV to Firestore | Fireship.io"
[11]: https://cloud.google.com/run/docs/configuring/max-instances?utm_source=chatgpt.com "Set maximum instances for services | Cloud Run Documentation"
[12]: https://stackoverflow.com/questions/52434455/how-to-switch-from-datastore-to-firestore-in-existing-project?utm_source=chatgpt.com "How to switch from Datastore to Firestore in existing project?"

