[
  {
    "id": 1,
    "question_text": "Retrieve all customers who have placed an order in the last 30 days.",
    "schema": [
      {
        "table_name": "customers",
        "columns": ["id", "name", "email"],
        "primary_key": "id"
      },
      {
        "table_name": "orders",
        "columns": ["id", "customer_id", "order_date"],
        "primary_key": "id",
        "foreign_keys": ["customer_id -> customers.id"]
      }
    ],
    "difficulty": "Intermediate",
    "expected_query": "SELECT c.name, c.email FROM customers c JOIN orders o ON c.id = o.customer_id WHERE o.order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY);",
    "diagram_url": "https://example.com/er_diagram1.png"
  }
  // Add more questions as needed
]
# app/schemas.py
from pydantic import BaseModel
from typing import List, Optional

class TableSchema(BaseModel):
    table_name: str
    columns: List[str]  # List of column names
    primary_key: Optional[str] = None
    foreign_keys: Optional[List[str]] = None  # e.g., ["orders.customer_id -> customers.id"]

class Question(BaseModel):
    id: int
    question_text: str
    schema: List[TableSchema]
    difficulty: str  # e.g., "Beginner", "Intermediate", "Advanced"
    expected_query: Optional[str] = None
    diagram_url: Optional[str] = None  # URL to ER diagram image

@app.get("/api/questions/random", response_model=Question)
async def get_random_question(difficulty: Optional[str] = None):
    """
    Retrieve a random SQL question, optionally filtered by difficulty.
    """
    filtered_questions = questions_db
    if difficulty:
        filtered_questions = [q for q in questions_db if q.difficulty.lower() == difficulty.lower()]
        if not filtered_questions:
            raise HTTPException(status_code=404, detail="No questions found for the specified difficulty")

    return random.choice(filtered_questions)


    # app/main.py
from fastapi import FastAPI, HTTPException, Request
from pydantic import BaseModel
from google.cloud import bigquery
from app.database import client
import re
from fastapi.middleware.cors import CORSMiddleware
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
import logging

app = FastAPI()

# Configure Logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# CORS configuration
origins = [
    "http://localhost",
    "http://localhost:5500",
    "http://127.0.0.1:5500",
    # Add more origins as needed
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,  # List of allowed origins
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Rate Limiting
limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# Define the request schema
class QueryRequest(BaseModel):
    query: str

# Define the response schema
class QueryResponse(BaseModel):
    success: bool
    rows: list = []
    rowCount: int = 0
    fields: list = []
    error: str = ""

# Allowed SQL commands for security
ALLOWED_COMMANDS = {'SELECT', 'INSERT', 'UPDATE', 'DELETE'}

@app.post("/api/execute-sql", response_model=QueryResponse)
@limiter.limit("10/minute")  # Limit to 10 requests per minute per IP
async def execute_sql(query_request: QueryRequest, request: Request):
    query = query_request.query.strip()
    logger.info(f"Received query: {query}")

    # Extract the first word of the query to check allowed commands
    first_word = re.match(r"^\w+", query, re.IGNORECASE)
    if not first_word:
        logger.error("Invalid query format.")
        raise HTTPException(status_code=400, detail="Invalid query format.")

    command = first_word.group(0).upper()
    if command not in ALLOWED_COMMANDS:
        logger.error(f"Disallowed command: {command}")
        raise HTTPException(status_code=400, detail=f"Only {', '.join(ALLOWED_COMMANDS)} statements are allowed.")

    try:
        query_job = client.query(query, job_config=bigquery.QueryJobConfig(
            timeout_ms=60000,  # 60 seconds timeout
        ))  # Make an API request.

        results = query_job.result()  # Wait for the query to finish.

        # Extract field names
        fields = [field.name for field in results.schema]

        # Extract rows as list of dictionaries
        rows = [dict(row) for row in results]

        logger.info(f"Query successful: {len(rows)} rows returned.")

        return QueryResponse(
            success=True,
            rows=rows,
            rowCount=len(rows),
            fields=fields
        )

    except Exception as e:
        logger.error(f"Query failed: {str(e)}")
        return QueryResponse(
            success=False,
            error=str(e)
        )


<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>SQL Assessment Platform</title>
  <link rel="stylesheet" href="https://cdn.datatables.net/1.10.24/css/jquery.dataTables.min.css">
  <style>
    #editor-container {
      width: 100%;
      height: 300px;
      border: 1px solid #ccc;
    }
    #execute-btn {
      margin-top: 10px;
      padding: 10px 20px;
      font-size: 16px;
    }
    #results-container {
      margin-top: 20px;
    }
  </style>
</head>
<body>
  <h1>SQL Assessment</h1>
  <div id="editor-container"></div>
  <button id="execute-btn">Execute Query</button>
  <div id="results-container"></div>

  <!-- Dependencies -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/monaco-editor/0.33.0/min/vs/loader.min.js"></script>
  <script src="https://code.jquery.com/jquery-3.5.1.min.js"></script>
  <script src="https://cdn.datatables.net/1.10.24/js/jquery.dataTables.min.js"></script>

  <script>
    require.config({ paths: { 'vs': 'https://cdnjs.cloudflare.com/ajax/libs/monaco-editor/0.33.0/min/vs' }});
    require(['vs/editor/editor.main'], function() {
      var editor = monaco.editor.create(document.getElementById('editor-container'), {
        value: 'SELECT * FROM users;',
        language: 'sql',
        theme: 'vs-dark',
        automaticLayout: true
      });

      document.getElementById('execute-btn').addEventListener('click', function() {
        var query = editor.getValue();
        // Send the query to the backend via API
        fetch('/api/execute-sql', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({ query: query })
        })
        .then(response => response.json())
        .then(data => {
          var resultsContainer = document.getElementById('results-container');
          resultsContainer.innerHTML = ''; // Clear previous results

          if (data.success) {
            // Create a table to display results
            var table = document.createElement('table');
            table.id = 'results-table';
            table.className = 'display';
            var thead = document.createElement('thead');
            var headerRow = document.createElement('tr');

            data.fields.forEach(field => {
              var th = document.createElement('th');
              th.textContent = field;
              headerRow.appendChild(th);
            });

            thead.appendChild(headerRow);
            table.appendChild(thead);

            var tbody = document.createElement('tbody');
            data.rows.forEach(row => {
              var tr = document.createElement('tr');
              data.fields.forEach(field => {
                var td = document.createElement('td');
                td.textContent = row[field];
                tr.appendChild(td);
              });
              tbody.appendChild(tr);
            });

            table.appendChild(tbody);
            resultsContainer.appendChild(table);

            // Initialize DataTables
            $('#results-table').DataTable();
          } else {
            // Display error message
            var errorDiv = document.createElement('div');
            errorDiv.style.color = 'red';
            errorDiv.textContent = 'Error: ' + data.error;
            resultsContainer.appendChild(errorDiv);
          }
        })
        .catch(error => {
          console.error('Error executing query:', error);
        });
      });
    });
  </script>
</body>
</html>
