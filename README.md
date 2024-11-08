from flask import Flask, request, jsonify

from celery import Celery

from apscheduler.schedulers.background import BackgroundScheduler

from apscheduler.triggers.interval import IntervalTrigger

import sendgrid

from sendgrid.helpers.mail import Mail, Email, To, Content

import openai

import redis

from datetime import datetime, timedelta

# Flask setup
app = Flask(__name__)

# Configure Celery with Redis as the broker

app.config['CELERY_BROKER_URL'] = 'redis://localhost:6379/0'

app.config['CELERY_RESULT_BACKEND'] = 'redis://localhost:6379/0'

celery = Celery(app.name, broker=app.config['CELERY_BROKER_URL'])

celery.conf.update(app.config)

# APScheduler setup
scheduler = BackgroundScheduler()

# OpenAI API Key (replace with your actual OpenAI API key)
openai.api_key = "your_openai_api_key"

# SendGrid API Key (replace with your SendGrid API key)
SENDGRID_API_KEY = "your_sendgrid_api_key"
FROM_EMAIL = "your-email@example.com"


# Celery task for sending emails
@celery.task

def send_email(to_email, subject, content):

    sg = sendgrid.SendGridAPIClient(api_key=SENDGRID_API_KEY)
    from_email = Email(FROM_EMAIL)
    to_email = To(to_email)    
    content = Content("text/plain", content) 
    mail = Mail(from_email, to_email, subject, content)    
    try:
        response = sg.send(mail)
        return f"Email sent to {to_email}: {response.status_code}"      
    except Exception as e:
        return str(e)


# Function to generate email content using OpenAI GPT

def generate_email_content(prompt, data_row):

    customized_prompt = prompt.format(**data_row)
    response = openai.Completion.create(
        engine="text-davinci-003",  # or any other engine you prefer
        prompt=customized_prompt,
        max_tokens=200
    )
    return response.choices[0].text.strip()


# Function to schedule email sending
def schedule_email(email_data, schedule_time):

    # Schedule the email to be sent at a specific time
    scheduler.add_job(
        send_scheduled_email, 
        trigger=IntervalTrigger(start_date=schedule_time, seconds=10),  # Adjust interval if necessary
        args=[email_data],
        id="scheduled_email_task"
    )


# Send the email at the scheduled time
def send_scheduled_email(email_data):

    # Generate email content based on row data
    email_content = generate_email_content(email_data['prompt'], email_data['data_row'])
    
    # Send the email
    result = send_email.apply_async(args=[email_data['recipient'], email_data['subject'], email_content])
    return result.get()


@app.route('/send_scheduled_email', methods=['POST'])
def send_scheduled_email_route():

    data = request.get_json()

    # Parse the input data
    email_data = {
        'prompt': data['prompt'],
        'subject': data['subject'],
        'recipient': data['recipient'],
        'data_row': data['data_row']
    }

    # Schedule the email for a specific time
    schedule_time = datetime.now() + timedelta(seconds=10)  # Adjust this to the user-defined schedule time
    schedule_email(email_data, schedule_time)

    return jsonify({"message": "Email scheduled successfully!"}), 200


if __name__ == '__main__':
    # Start the scheduler
    scheduler.start()

    # Run the Flask app
    app.run(debug=True)
