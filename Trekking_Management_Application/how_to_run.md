## HOW TO RUN THE APP

### Execution Concepts

*   **`pip install -r requirements.txt`**: Reads the requirements definition manifest file and installs the exact libraries listed into your python environment.
*   **`python app.py`**: Executes the entry point Python script, creating the Flask WSGI development server locally. This server listens on port `5000` on the loopback interface (`127.0.0.1` / localhost).

---

### Step-by-Step Run Instructions

```bash
# 1. Navigate to the project root directory containing app.py
cd trekking_app

# 2. Install necessary project packages via requirements file
pip install -r requirements.txt

# 3. Start the Flask local development server
python app.py

# 4. Open your web browser and navigate to the loopback address:
# http://127.0.0.1:5000

# Default administrator login credentials:
# Email: admin@trek.com
# Password: admin123
```