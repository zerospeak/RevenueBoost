# RevenueBoost MVP – In-depth Technical Implementation Guide

---

## Table of Contents

1. [Overview & Architecture](#overview--architecture)
2. [Deployment & Setup](#deployment--setup)
3. [Detailed Component Walkthrough](#detailed-component-walkthrough)
    1. [Frontend (HTML/CSS/JS/NGINX)](#frontend-htmlcssjsnginx)
    2. [PHP Backend API](#php-backend-api)
    3. [.NET Compliance Microservice](#net-compliance-microservice)
    4. [Python ML Microservice](#python-ml-microservice)
    5. [Data Stores (MySQL & Redis)](#data-stores-mysql--redis)
4. [Integration Flow (E2E)](#integration-flow-e2e)
5. [Diagrams](#diagrams)
6. [Appendix: Full Source Code](#appendix-full-source-code)

---
# RevenueBoost MVP – In-depth Technical Implementation Guide

---

## Overview & Architecture



 

RevenueBoost MVP is a modular debt relief and compliance platform, built as a set of microservices. The architecture leverages the following components:
- **Frontend:** HTML/CSS/vanilla JavaScript, served via NGINX.
- **Backend API:** PHP, routing user actions, orchestrating business logic, and connecting with other microservices.
- **Compliance Service:** .NET Core (C#) REST API for compliance determination.
- **ML Service:** Python Flask microservice simulating risk scoring.
- **Infra:** All services containerized with Docker, orchestrated with docker-compose.
- **Data Stores:** MySQL and Redis setup for extensibility (persistence/cache optional in MVP).

### High-level Architecture Diagram

graph TD
User[User (Web Browser)]
Frontend[Frontend (HTML/JS/NGINX)]
PHP[PHP Backend API]
DotNet[.NET Compliance Service]
PythonML[Python ML Microservice]
MySQL[(MySQL DB)]
Redis[(Redis Cache)]

User -- HTTP/HTTPS --> Frontend
Frontend -- REST (fetch) --> PHP
PHP -- REST (cURL) --> DotNet
DotNet -- REST (HTTPClient) --> PythonML
PHP -- (optionally SQL) --> MySQL
PHP -- (optionally Cache) --> Redis

---

## Deployment & Setup

### Files

- `docker-compose.yml`
- `frontend/*` (HTML/JS/CSS/NGINX config)
- `php-backend/*` (API, logic)
- `dotnet-compliance/*` (.NET compliance microservice)
- `python-ml/*` (Flask ML service)

### Steps

1. **Clone the repository and descend to the MVP root:**
   ```sh
   git clone <repo-url>
   cd RevenueBoost/revenueboost_mvp
   ```
2. **Build and launch all services:**
   ```sh
   docker-compose up --build
   ```
3. **Access Frontend:**
   - In your browser: `http://localhost:8080`
4. **Component Endpoints:**
   - Frontend: 8080
   - PHP API: 8000
   - .NET Compliance: 5001
   - Python ML: 5002

---

## Detailed Component Walkthrough

### Frontend (HTML/CSS/JS/NGINX)

Described in `frontend/index.html` and `frontend/script.js`.

**Features:**
- User-friendly form for user ID submission, analysis, and recommendations.
- Async fetch requests to PHP backend for analysis and recommendation.
- Displays output and compliance status.

**Key Files:**

#### index.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RevenueBoost MVP</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="container">
        <h1>RevenueBoost MVP</h1>

        <div class="input-section">
            <label for="userId">Enter User ID:</label>
            <input type="text" id="userId" value="user1">
            <button onclick="analyzeUser()">Analyze Behavior</button>
            <button onclick="getRecommendations()">Get Recommendations</button>
        </div>

        <div class="output-section">
            <h2>Analysis Output:</h2>
            <pre id="analysisOutput"></pre>
        </div>

        <div class="output-section">
            <h2>Recommendations:</h2>
            <ul id="recommendationsOutput"></ul>
        </div>
    </div>
    <script src="script.js"></script>
</body>
</html>
```

#### script.js
```javascript
const phpBackendUrl = 'http://localhost:8000';
const analysisOutput = document.getElementById('analysisOutput');
const recommendationsOutput = document.getElementById('recommendationsOutput');

async function analyzeUser() {
    const userId = document.getElementById('userId').value;
    if (!userId) {
        alert('Please enter a User ID.');
        return;
    }

    try {
        const response = await fetch(`${phpBackendUrl}/analyze/${userId}`);
        const data = await response.json();
        if (response.ok) {
            analysisOutput.textContent = JSON.stringify(data, null, 2);
        } else {
            analysisOutput.textContent = `Error: ${data.error || response.statusText}`;
        }
        recommendationsOutput.innerHTML = ''; // Clear previous recommendations
    } catch (error) {
        analysisOutput.textContent = `Error: ${error.message}`;
        recommendationsOutput.innerHTML = '';
    }
}

async function getRecommendations() {
    const userId = document.getElementById('userId').value;
    if (!userId) {
        alert('Please enter a User ID.');
        return;
    }

    try {
        const response = await fetch(`${phpBackendUrl}/recommend/${userId}`);
        const data = await response.json();
        if (response.ok) {
            recommendationsOutput.innerHTML = '';
            data.recommendations.forEach(recommendation => {
                const li = document.createElement('li');
                li.textContent = recommendation;
                recommendationsOutput.appendChild(li);
            });
            if (data.compliance_status) {
                const complianceLi = document.createElement('li');
                complianceLi.textContent = `Compliance Status: ${data.compliance_status}`;
                recommendationsOutput.appendChild(complianceLi);
            }
        } else {
            recommendationsOutput.innerHTML = `Error: ${data.error || response.statusText}`;
        }
    } catch (error) {
        recommendationsOutput.innerHTML = `Error: ${error.message}`;
    }
}
```

---

### PHP Backend API

Serves as the main orchestrator—routing requests, invoking business logic, and communicating with the .NET and Python services.

**Key Files:**

#### index.php
```php
<?php
error_log("index.php started");
header('Content-Type: application/json');
error_log("Content-Type header set");
header('Access-Control-Allow-Origin: *'); // For local development
error_log("Access-Control-Allow-Origin header set");

require 'mock_data.php';
require 'behavior_analyzer.php';
require 'recommendation_engine.php';

$endpoint = $_SERVER['PATH_INFO'] ?? '';
$parts = explode('/', trim($endpoint, '/'));
$resource = $parts[0] ?? '';
$userId = $parts[1] ?? null;

error_log("Endpoint: " . $endpoint);
error_log("Resource: " . $resource);
error_log("User ID: " . $userId);

if (empty($userId)) {
    http_response_code(400);
    echo json_encode(["error" => "User ID is required"]);
    exit;
}

function callService($url) {
    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    return ['code' => $httpCode, 'body' => $response];
}

if ($resource === 'analyze') {
    error_log("Handling analyze endpoint");
    if (isset($customersData[$userId])) {
        $analysis = analyzeBehavior($customersData[$userId]);
        echo json_encode($analysis);
    } else {
        http_response_code(404);
        echo json_encode(["error" => "Customer not found"]);
    }
} elseif ($resource === 'recommend') {
    error_log("Handling recommend endpoint");
    if (isset($customersData[$userId])) {
        $analysis = analyzeBehavior($customersData[$userId]);

        // Simulate calling the .NET Compliance Engine
        $complianceServiceUrl = 'http://dotnet-compliance:5001/check/' . $userId;
        $complianceResponse = callService($complianceServiceUrl);
        $complianceStatus = null;
        if ($complianceResponse['code'] === 200) {
            $complianceData = json_decode($complianceResponse['body'], true);
            $complianceStatus = $complianceData['status'] ?? null;
        }

        $recommendations = getRecommendation($analysis, $complianceStatus);
        echo json_encode(["recommendations" => $recommendations, "compliance_status" => $complianceStatus]);
    } else {
        http_response_code(404);
        echo json_encode(["error" => "Customer not found"]);
    }
} else {
    http_response_code(404);
    echo json_encode(["error" => "Invalid endpoint"]);
}
?>
```

#### recommendation_engine.php
```php
<?php
function getRecommendation($behaviorAnalysis, $complianceStatus = null) {
    $recommendations = [];
    if ($behaviorAnalysis["average_payment"] < 100) {
        $recommendations[] = "Consider exploring debt consolidation options.";
    }
    if (in_array("settlement plans", $behaviorAnalysis["browsing_interests"])) {
        $recommendations[] = "We have personalized settlement plans available.";
    }
    if (in_array("credit counseling", $behaviorAnalysis["browsing_interests"])) {
        $recommendations[] = "Explore our credit counseling services for expert advice.";
    }
    if ($complianceStatus === "compliant") {
        $recommendations[] = "(FTC Compliant Offer)";
    } elseif ($complianceStatus === "not_compliant") {
        $recommendations[] = "(Offer requires further compliance review)";
    }
    if (empty($recommendations)) {
        $recommendations[] = "Based on your profile, we currently don't have specific recommendations.";
    }
    return $recommendations;
}
?>
```

---

### .NET Compliance Microservice

Delivers compliance decisioning. Receives HTTP calls from the PHP backend, sometimes invokes Python ML for risk scoring.

#### ComplianceChecker.cs
```csharp
using System.Text.Json;

public class ComplianceChecker
{
    private readonly HttpClient _httpClient;

    public ComplianceChecker(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<string> CheckCompliance(string userId)
    {
        // In a real system, this would involve complex rule checking.
        // For this MVP, we'll just simulate based on the user ID.
        if (userId == "user1")
        {
            // Simulate calling the Python ML service
            var mlResponse = await _httpClient.GetAsync("http://python-ml:5002/predict/" + userId);
            if (mlResponse.IsSuccessStatusCode)
            {
                var mlResult = await mlResponse.Content.ReadAsStringAsync();
                var predictionData = JsonSerializer.Deserialize<PredictionResult>(mlResult);
                if (predictionData?.RiskScore < 0.7)
                {
                    return "compliant";
                }
                else
                {
                    return "not_compliant";
                }
            }
            else
            {
                return "not_compliant"; // Error calling ML service
            }
        }
        else if (userId == "user2")
        {
            return "compliant";
        }
        else
        {
            return "not_compliant";
        }
    }
}

public class PredictionResult
{
    public double RiskScore { get; set; }
}
```

#### Program.cs
```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using System.Text.Json;

public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.ConfigureServices(services =>
                {
                    services.AddHttpClient();
                    services.AddSingleton<ComplianceChecker>();
                });
                webBuilder.Configure(app =>
                {
                    app.UseRouting();
                    app.UseEndpoints(endpoints =>
                    {
                        endpoints.MapGet("/check/{userId}", async (HttpContext context, string userId, ComplianceChecker checker) =>
                        {
                            var status = await checker.CheckCompliance(userId);
                            await context.Response.WriteAsJsonAsync(new { status });
                        });
                    });
                });
            });
}
```

---

### Python ML Microservice

Simulates a risk score predictor using Flask. Called by the .NET service to obtain a `risk_score` for compliance decisions.

#### app.py
```python
from flask import Flask, jsonify
from flask_cors import CORS
import random

app = Flask(__name__)
CORS(app)

@app.route('/predict/<user_id>', methods=['GET'])
def predict_risk(user_id):
    # Simulate a risk score prediction based on user ID
    if user_id == "user1":
        risk_score = random.uniform(0.6, 0.8)
    elif user_id == "user2":
        risk_score = random.uniform(0.2, 0.5)
    else:
        risk_score = random.uniform(0.4, 0.7)
    return jsonify({"risk_score": risk_score})

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5002)
```

---

### Data Stores (MySQL & Redis)

Provisioned in `docker-compose.yml`. Not directly used for persistence in this MVP, but ready for production integration. Enable/comment volumes for persistence.

#### docker-compose.yml (excerpt)
```yaml
services:
  php-backend:
    build: ./php-backend
    ports:
      - "8000:80"
    volumes:
      - ./php-backend:/var/www/html/
    depends_on:
      - dotnet-compliance

  dotnet-compliance:
    build: ./dotnet-compliance
    ports:
      - "5001:80"
    depends_on:
      - python-ml

  python-ml:
    build: ./python-ml
    ports:
      - "5002:5000"

  frontend:
    build: ./frontend
    ports:
      - "8080:80"
    volumes:
      - ./frontend:/usr/share/nginx/html:ro
    depends_on:
      - php-backend
      - dotnet-compliance
      - python-ml

  mysql-db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: revenueboost
    ports:
      - "3306:3306"
    # volumes:
    #   - ./mysql/data:/var/lib/mysql # Uncomment for persistent data

  redis-cache:
    image: redis:latest
    ports:
      - "6379:6379"
    # volumes:
    #   - ./redis/data:/data # Uncomment for persistent data
```

---

---

## Integration Flow (E2E)




**User Scenario:** _Get Recommendations for User ID "user1"_

1. **Frontend:** User clicks "Get Recommendations".
2. **PHP Backend:** Receives request, processes user behavioral data.
3. **Compliance Engine (.NET):** PHP calls .NET for compliance check.
4. **Python ML:** .NET calls Python to evaluate user risk.
5. **Responses:** Chain of results propagates back, with final customized & compliance-checked recommendations displayed to the user.

#### Sequence Diagram

sequenceDiagram
participant U as User
participant F as Frontend (JS)
participant P as PHP Backend
participant D as .NET Compliance
participant M as Python ML

U->>F: Enters User ID & clicks "Get Recommendations"
F->>P: HTTP GET /recommend/{userId}
P->>D: REST /check/{userId}
D->>M: REST /predict/{userId}
M-->>D: Risk Score
D-->>P: Compliance result
P-->>F: JSON (recommendations + compliance status)
F-->>U: Displays results


---

## Diagrams

### Deployment Diagram:

flowchart LR
subgraph Docker Compose Host
F[Frontend (NGINX)]:::svc
PB[PHP Backend]:::svc
DC[.NET Compliance]:::svc
ML[Python ML]:::svc
MY[MySQL DB]:::db
RC[Redis Cache]:::cache
end

style F fill:#ffe
style PB fill:#e6f7ff
style DC fill:#ffe6ff
style ML fill:#fff1e6
style MY fill:#fffbe6
style RC fill:#e6ffe6

F -- 8000/8080 --> PB
PB -- 5001 --> DC
DC -- 5002 --> ML
PB -- 3306 --> MY
PB -- 6379 --> RC

---

## Appendix: Full Source Code

For brevity, see previous sections or reference the corresponding repo directories for:

- `frontend/*`
- `php-backend/*`
- `dotnet-compliance/*`
- `python-ml/*`
- `docker-compose.yml`

All provided code may be adapted as needed and demonstrates cross-stack orchestration, modular best practices, and developer-ready build/run strategies.

---

_This in-depth technical implementation guide is suitable for engineers, reviewers, or technical leads evaluating the RevenueBoost MVP for code quality, architecture, integration, and real-world deployment._
