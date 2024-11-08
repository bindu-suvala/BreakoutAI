import openai

import sendgrid

from sendgrid.helpers.mail import Mail, Email, To, Content

from flask import Flask, request, jsonify

import pandas as pd

app = Flask(__name__)

# OpenAI API Key (Replace with your OpenAI key)
openai.api_key = "your_openai_api_key"

# SendGrid API Key (Replace with your SendGrid key)
SENDGRID_API_KEY = "your_sendgrid_api_key"
FROM_EMAIL = "your-email@example.com"

# Function to generate email content using OpenAI GPT
def generate_email_content(prompt, data_row):

    customized_prompt = prompt.format(**data_row)
    
    response = openai.Completion.create(
    
        engine="text-davinci-003",  # or any other engine you prefer
        
        prompt=customized_prompt,
        
        max_tokens=200
    )
    return response.choices[0].text.strip()

# Function to send email via SendGrid

def send_email(to_email, subject, content):

    sg = sendgrid.SendGridAPIClient(api_key=SENDGRID_API_KEY)
    
    from_email = Email(FROM_EMAIL)
    
    to_email = To(to_email)
    
    content = Content("text/plain", content)
    
    mail = Mail(from_email, to_email, subject, content)
    
    try:
        response = sg.send(mail)
        
        return response.status_code, response.body
        
    except Exception as e:
    
        return str(e), None

# Route to handle email sending after CSV processing

@app.route('/send_emails', methods=['POST'])

def send_emails():

    data = request.get_json()
    
    csv_file = data['csv_file']  # Base64 encoded CSV file
    
    prompt = data['prompt']
    
    recipient_column = data['recipient_column']

    # Decode and read CSV file
    
    csv_data = pd.read_csv(pd.compat.StringIO(csv_file.decode('base64')))
    
    results = []

    for _, row in csv_data.iterrows():
    
        # Generate customized email content for each row
        
        email_content = generate_email_content(prompt, row)
        
        # Get recipient email from the row (adjust column name if necessary)
        
        recipient_email = row[recipient_column]
        
        # Send email
        status_code, response = send_email(recipient_email, "Your Customized Email", email_content)
        
        # Store result (Success or failure)
        
        results.append({
        
            "recipient": recipient_email,
            
            "status": "Success" if status_code == 202 else "Failed",
            
            "response": response
        })
    
    return jsonify(results), 200

if __name__ == '__main__':
    app.run(debug=True)
