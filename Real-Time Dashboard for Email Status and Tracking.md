from flask import Flask, render_template, request

from flask_socketio import SocketIO

from apscheduler.schedulers.background import BackgroundScheduler

import openai

import requests

from random import randint

from time import sleep

import threading

from datetime import datetime

# Initialize Flask app, SocketIO, and background scheduler

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///email_events.db'
socketio = SocketIO(app)

# OpenAI GPT-3 API Key (for email content generation)
openai.api_key = "your_openai_api_key"  # Replace with your OpenAI API key

# Simulate email statuses
email_status = {
    'total_sent': 0,
    'pending': 0,
    'failed': 0
}

# Real-time email analytics update
def update_analytics():

    while True:
        email_status['total_sent'] += randint(1, 5)
        email_status['pending'] = max(0, email_status['pending'] - randint(0, 2))
        email_status['failed'] = randint(0, 5)

        # Emit real-time status update to the frontend via Socket.IO
        socketio.emit('email_status', email_status)
        sleep(10)  # Update every 10 seconds

@app.before_first_request
def before_first_request():

    thread = threading.Thread(target=update_analytics)
    thread.daemon = True
    thread.start()

# Email content generation using GPT-3
def generate_email_content(prompt, data_row):

    customized_prompt = prompt.format(**data_row)
    response = openai.Completion.create(
        engine="text-davinci-003",
        prompt=customized_prompt,
        max_tokens=100
    )
    return response.choices[0].text.strip()

# Route to serve the dashboard page
@app.route('/')
def dashboard():

    return render_template('dashboard.html')

# Webhook for ESP (SendGrid) to track email events (Delivered, Opened, Bounced)
@app.route('/sendgrid-webhook', methods=['POST'])
def sendgrid_webhook():

    events = request.json
    for event in events:
        if event['event'] == 'delivered':
            print(f"Email {event['email']} delivered")
        elif event['event'] == 'opened':
            print(f"Email {event['email']} opened")
        elif event['event'] == 'bounced':
            print(f"Email {event['email']} bounced")
        # You can also save these events to a database or log for tracking
    return '', 200

# Function to schedule email sending (Simulated for the demo)
def send_scheduled_email():

    print("Sending scheduled email...")
    # Insert actual email sending logic here (e.g., SendGrid API)
    pass

# Start a background scheduler to simulate sending emails at regular intervals
scheduler = BackgroundScheduler()
scheduler.add_job(send_scheduled_email, 'interval', minutes=30)
scheduler.start()

# Route to insert dynamic placeholders into email content
@app.route('/insert-placeholder', methods=['POST'])
def insert_placeholder():
    data = {'{Company Name}': 'ABC Corp', '{Location}': 'New York'}
    prompt = request.json['prompt']

    # Replace placeholders dynamically with actual values
    for placeholder, value in data.items():
        prompt = prompt.replace(placeholder, value)

    return {'updated_prompt': prompt}, 200

if __name__ == '__main__':
    socketio.run(app, debug=True)
