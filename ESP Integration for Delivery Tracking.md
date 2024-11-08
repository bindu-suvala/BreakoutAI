from flask import Flask, render_template, request

from flask_socketio import SocketIO, emit

from flask_sqlalchemy import SQLAlchemy

from apscheduler.schedulers.background import BackgroundScheduler

import requests

import openai

from datetime import datetime

import threading

import json

from random import randint

from time import sleep

# Initialize Flask app, SQLAlchemy, and SocketIO

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///email_events.db'  # SQLite DB for event logging
db = SQLAlchemy(app)
socketio = SocketIO(app)

# Set up OpenAI API for email content generation
openai.api_key = "your_openai_api_key"  # Replace with your OpenAI API key

# Database model for storing email events

class EmailEvent(db.Model):

    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(120), nullable=False)
    event = db.Column(db.String(50), nullable=False)
    timestamp = db.Column(db.String(50), nullable=False)

    def __repr__(self):
        return f"<EmailEvent {self.email} - {self.event}>"

# Create tables
with app.app_context():

    db.create_all()

# Simulate email sending status (for the demo)

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
        engine="text-davinci-003",  # Or another engine
        prompt=customized_prompt,
        max_tokens=100
    )
    return response.choices[0].text.strip()

@app.route('/')
def dashboard():
    return render_template('dashboard.html')

# Schedule email sending with APScheduler
def send_scheduled_email(email_data):

    print(f"Sending email to: {email_data['to']}")
    # Add logic to send the email here (e.g., via SendGrid, SMTP, etc.)

scheduler = BackgroundScheduler()
scheduler.add_job(send_scheduled_email, 'interval', minutes=30, args=[{"to": "example@example.com"}])
scheduler.start()

# Webhook for SendGrid event tracking (delivered, opened, bounced, etc.)
@app.route('/sendgrid-webhook', methods=['POST'])
def sendgrid_webhook():
    events = request.json
    for event in events:
        # Store event in database
        new_event = EmailEvent(
            email=event['email'],
            event=event['event'],
            timestamp=str(datetime.now())
        )
        db.session.add(new_event)
        db.session.commit()
        
        # Log event for demo
        if event['event'] == 'delivered':
            print(f"Email {event['email']} delivered")
        elif event['event'] == 'opened':
            print(f"Email {event['email']} opened")
        elif event['event'] == 'bounced':
            print(f"Email {event['email']} bounced")
        else:
            print(f"Unknown event: {event['event']} for email {event['email']}")
    
    return '', 200  # Respond with a 200 status code to acknowledge the webhook

# Frontend: Insert dynamic placeholders into the email prompt
@app.route('/insert-placeholder', methods=['POST'])
def insert_placeholder():
    # Get data from CSV/Google Sheets (Simulated for now)
    data = {'{Company Name}': 'ABC Corp', '{Location}': 'New York'}
    prompt = request.json['prompt']
    
    # Insert placeholders dynamically
    for placeholder, value in data.items():
        prompt = prompt.replace(placeholder, value)

    return {'updated_prompt': prompt}, 200

if __name__ == '__main__':
    socketio.run(app, debug=True)
