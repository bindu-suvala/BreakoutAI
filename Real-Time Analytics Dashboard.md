from flask import Flask, render_template

from flask_socketio import SocketIO, emit

from random import randint

from time import sleep

import threading

app = Flask(__name__)
socketio = SocketIO(app)

# Initialize email status counters

email_status = {
    'total_sent': 0,
    'pending': 0,
    'failed': 0
}

# Function to simulate real-time email status updates

def update_analytics():

    while True:
        # Simulating changes in the email status
        email_status['total_sent'] += randint(1, 5)
        email_status['pending'] = max(0, email_status['pending'] - randint(0, 2))
        email_status['failed'] = randint(0, 5)
        
        # Emit real-time status update to the client
        socketio.emit('email_status', email_status)
        
        # Wait for 10 seconds before the next update
        sleep(10)

# Start a background thread to update analytics

@app.before_first_request

def before_first_request():
    thread = threading.Thread(target=update_analytics)
    thread.daemon = True
    thread.start()

@app.route('/')
def dashboard():
    return render_template('dashboard.html')

if __name__ == '__main__':
    socketio.run(app, debug=True)
