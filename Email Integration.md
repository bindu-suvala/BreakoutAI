import os
import pickle
import smtplib
import base64
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request
from googleapiclient.discovery import build
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import Mail

# Define the Gmail API scope for sending emails
SCOPES = ['https://www.googleapis.com/auth/gmail.send']

# Function to authenticate with Gmail using OAuth2
def authenticate_gmail():
    """
    Authenticates the user with Gmail using OAuth2.
    :return: Credentials object for Gmail API
    """
    creds = None
    # Token file stores the user's access and refresh tokens
    if os.path.exists('token.pickle'):
        with open('token.pickle', 'rb') as token:
            creds = pickle.load(token)
    
    # If no valid credentials are available, let the user log in.
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(
                'credentials.json', SCOPES)
            creds = flow.run_local_server(port=0)
        
        # Save the credentials for the next run
        with open('token.pickle', 'wb') as token:
            pickle.dump(creds, token)
    
    return creds

# Function to send email via Gmail using Gmail API
def send_email_gmail(creds, to_email, subject, body):
    """
    Sends an email using the Gmail API.
    :param creds: Credentials object returned by authenticate_gmail
    :param to_email: Recipient email address
    :param subject: Email subject
    :param body: Email body content
    """
    try:
        # Build the Gmail service
        service = build('gmail', 'v1', credentials=creds)

        # Create the email message
        message = MIMEMultipart()
        message['to'] = to_email
        message['subject'] = subject
        message.attach(MIMEText(body, 'plain'))

        # Encode the message and send it
        raw_message = base64.urlsafe_b64encode(message.as_bytes()).decode()
        send_message = service.users().messages().send(userId="me", body={'raw': raw_message}).execute()

        print(f"Message sent to {to_email}. Message ID: {send_message['id']}")
    except Exception as error:
        print(f"An error occurred while sending email: {error}")

# Function to send email via SendGrid API
def send_email_sendgrid(to_email, subject, content):
    """
    Sends an email using the SendGrid API.
    :param to_email: Recipient email address
    :param subject: Email subject
    :param content: Email body content
    :return: SendGrid API response
    """
    sg = SendGridAPIClient(api_key='SENDGRID_API_KEY')  # Replace with your SendGrid API key
    from_email = 'your-email@example.com'  # Replace with your SendGrid verified email
    mail = Mail(from_email, to_email, subject, content)

    response = sg.send(mail)
    print(f"Email sent with status code {response.status_code}")
    return response

# Function to send email via SMTP (for Gmail, Outlook, etc.)
def send_email_smtp(smtp_server, smtp_port, sender_email, sender_password, to_email, subject, body):
    """
    Sends an email using SMTP.
    :param smtp_server: SMTP server address (e.g., 'smtp.gmail.com')
    :param smtp_port: SMTP port (e.g., 587 for TLS)
    :param sender_email: Sender email address
    :param sender_password: Sender email password or app-specific password
    :param to_email: Recipient email address
    :param subject: Email subject
    :param body: Email body content
    """
    try:
        # Create MIME message
        message = MIMEMultipart()
        message['From'] = sender_email
        message['To'] = to_email
        message['Subject'] = subject
        message.attach(MIMEText(body, 'plain'))

        # Connect to the SMTP server and send the email
        with smtplib.SMTP(smtp_server, smtp_port) as server:
            server.starttls()  # Secure the connection
            server.login(sender_email, sender_password)
            server.sendmail(sender_email, to_email, message.as_string())

        print(f"Email sent to {to_email}")
    except Exception as e:
        print(f"Error sending email: {e}")

# Example usage of the functions
if __name__ == "__main__":
    # 1. Gmail Authentication
    print("Authenticating with Gmail...")
    creds = authenticate_gmail()

    # 2. Send email via Gmail API
    send_email_gmail(creds, "recipient@example.com", "Subject from Gmail", "Hello from Gmail API!")

    # 3. Send email via SendGrid API
    send_email_sendgrid("recipient@example.com", "Subject from SendGrid", "Hello from SendGrid API!")

    # 4. Send email via SMTP (example with Gmail)
    send_email_smtp("smtp.gmail.com", 587, "your-email@gmail.com", "your-password", "recipient@example.com", "Subject via SMTP", "Hello from SMTP!")
