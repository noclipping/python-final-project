### 🌌 7. Serving Images and Data through Flask

This step refines your Flask application to handle two major functionalities: serving dynamic images created with Matplotlib and serving structured data as JSON. The following outline and detailed explanation cater to these functionalities.

#### Prerequisites

Before proceeding, ensure you have the following packages installed:

- Flask
- psycopg2
- python-dotenv
- pandas
- matplotlib

If not already installed, you can install them using pip:

```bash
pip install Flask psycopg2-binary python-dotenv pandas matplotlib
```

#### Application Setup

**Environment and Imports**: Ensure that your application is set up with all necessary imports and the environment is correctly configured to handle database connections and plot generation.

```python
from flask import Flask, jsonify, send_file
import matplotlib
matplotlib.use('Agg')  # Use non-GUI backend for Matplotlib
import matplotlib.pyplot as plt
import io
import psycopg2
import os
from dotenv import load_dotenv
import pandas as pd

load_dotenv()  # Load environment variables from .env file
app = Flask(__name__)
```

**Database Connection Function**:
A reusable function to handle database connections simplifies your route implementations.

```python
def get_db_connection():
    conn = psycopg2.connect(os.environ['DATABASE_URL'])
    return conn
```

#### Feature 1: Serve Temperature Trends as an Image

**Create a Temperature Plot**:
The function to generate temperature plots fetches data from the database and creates a plot.

```python
def create_temperature_plot():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("""
        SELECT period, temperature
        FROM weather_forecasts
        WHERE scraped_at >= current_date - INTERVAL '7 days'
        ORDER BY scraped_at;
    """)
    data = cur.fetchall()
    cur.close()
    conn.close()

    df = pd.DataFrame(data, columns=['period', 'temperature'])
    df['temperature'] = df['temperature'].str.extract('(\d+)').astype(int)

    fig, ax = plt.subplots()
    ax.plot(df['period'], df['temperature'], marker='o', linestyle='-', color='b')
    ax.set(title='Weekly Temperature Trends', xlabel='Period', ylabel='Temperature (°F)')
    ax.grid(True)
    ax.set_xticklabels(df['period'], rotation=45)
    return fig
```

**Temperature Trends Route**:
A Flask route to serve the generated plot as a PNG image.

```python
@app.route('/temperature_trends')
def temperature_trends():
    fig = create_temperature_plot()
    buf = io.BytesIO()
    fig.savefig(buf, format='png')
    plt.close(fig)
    buf.seek(0)
    return send_file(buf, mimetype='image/png')
```

#### Feature 2: Serve Weather Data as JSON

**Weather Data Route**:
A route to fetch and jsonify weather data, allowing client applications to consume structured weather information.

```python
@app.route('/weather', methods=['GET'])
def get_weather():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("""
        SELECT id, period, short_desc, temperature, scraped_at
        FROM weather_forecasts
        WHERE scraped_at >= current_date - INTERVAL '1 week'
        ORDER BY scraped_at DESC;
    """)
    weather_data = cur.fetchall()
    cur.close()
    conn.close()

    columns = ['id', 'period', 'short_desc', 'temperature', 'scraped_at']
    result = [dict(zip(columns, row)) for row in weather_data]
    return jsonify(result)
```

#### Running the Application

Ensure the Flask application is configured to run with debugging enabled during development:

```python
if __name__ == '__main__':
    app.run(debug=True)
```

### Conclusion

This setup provides a comprehensive approach to serving both images and JSON data through a Flask API, leveraging the capabilities of Matplotlib for visualization and PostgreSQL for data management. It effectively separates concerns, ensuring that each part of the application is focused on a specific functionality while maintaining clean and reusable code structures.
