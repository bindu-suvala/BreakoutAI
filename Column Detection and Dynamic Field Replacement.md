from flask import Flask, render_template, request, jsonify

import pandas as pd

app = Flask(__name__)

# Route to render the HTML page
@app.route('/')
def index():
    return render_template('index.html')

# Route to handle the file upload and auto-detect columns
@app.route('/upload', methods=['POST'])
def upload_file():
    file = request.files['file']
    if file:
        # Read the CSV file into a DataFrame
        data = pd.read_csv(file)
        columns = data.columns.tolist()  # Get the column names
        return jsonify(columns)  # Return columns as JSON
    return jsonify({'error': 'No file uploaded'}), 400

if __name__ == '__main__':
    app.run(debug=True)
