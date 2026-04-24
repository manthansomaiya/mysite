from flask import Flask, render_template, request, jsonify, redirect, url_for
import sqlite3
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import os
from datetime import datetime

app = Flask(__name__)
DATABASE = 'tournament.db'
ADMIN_PASSWORD = 'admin123'

# Admin contact
ADMIN_EMAIL = 'manthansomaiya76@gmail.com'
ADMIN_PHONE = '9904701641'

# SMTP config (Gmail)
SMTP_SERVER = 'smtp.gmail.com'
SMTP_PORT = 587
SENDER_EMAIL = 'manthansomaiya76@gmail.com'
# IMPORTANT: Set your Gmail App Password as environment variable or replace below
SENDER_PASSWORD = os.getenv('GMAIL_APP_PASSWORD', 'gdsm qtia ljjv tiuu')

def init_db():
    conn = sqlite3.connect(DATABASE)
    c = conn.cursor()
    c.execute('''
        CREATE TABLE IF NOT EXISTS registrations (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            team_name TEXT NOT NULL,
            game TEXT NOT NULL,
            player1_uid TEXT NOT NULL,
            player2_uid TEXT NOT NULL,
            player3_uid TEXT NOT NULL,
            player4_uid TEXT NOT NULL,
            gmail TEXT NOT NULL,
            phone TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.commit()
    conn.close()

def send_email(subject, body):
    try:
        msg = MIMEMultipart()
        msg['From'] = SENDER_EMAIL
        msg['To'] = ADMIN_EMAIL
        msg['Subject'] = subject
        msg.attach(MIMEText(body, 'plain'))
        server = smtplib.SMTP(SMTP_SERVER, SMTP_PORT)
        server.starttls()
        server.login(SENDER_EMAIL, SENDER_PASSWORD)
        server.send_message(msg)
        server.quit()
        return True
    except Exception as e:
        print(f"[EMAIL ERROR] {e}")
        return False

@app.route('/')
def home():
    return render_template('index.html', admin_email=ADMIN_EMAIL, admin_phone=ADMIN_PHONE)

@app.route('/register', methods=['POST'])
def register():
    data = request.form
    team_name = data.get('teamName', '').strip()
    game = data.get('gameSelect', '').strip()
    p1 = data.get('player1', '').strip()
    p2 = data.get('player2', '').strip()
    p3 = data.get('player3', '').strip()
    p4 = data.get('player4', '').strip()
    gmail = data.get('gmail', '').strip()
    phone = data.get('phone', '').strip()

    if not all([team_name, game, p1, p2, p3, p4, gmail]):
        return jsonify({'status': 'error', 'message': 'Please fill all required fields.'}), 400

    conn = sqlite3.connect(DATABASE)
    c = conn.cursor()
    c.execute('''
        INSERT INTO registrations (team_name, game, player1_uid, player2_uid, player3_uid, player4_uid, gmail, phone)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?)
    ''', (team_name, game, p1, p2, p3, p4, gmail, phone))
    conn.commit()
    reg_id = c.lastrowid
    conn.close()

    # Build email body
    email_body = f"""
New Tournament Registration!

Team Name: {team_name}
Game: {game}
Gmail: {gmail}
Phone: {phone}

Player UIDs:
- Player 1: {p1}
- Player 2: {p2}
- Player 3: {p3}
- Player 4: {p4}

Registered at: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
Database ID: {reg_id}
"""
    email_sent = send_email(f"New Registration: {team_name} | {game}", email_body)

    return jsonify({
        'status': 'success',
        'message': 'Team registered successfully!' + (' Email sent to admin.' if email_sent else ' (Email failed, but data saved.)'),
        'id': reg_id
    })

@app.route('/admin', methods=['GET', 'POST'])
def admin():
    error = None
    records = []
    if request.method == 'POST':
        password = request.form.get('password', '')
        if password == ADMIN_PASSWORD:
            conn = sqlite3.connect(DATABASE)
            conn.row_factory = sqlite3.Row
            c = conn.cursor()
            c.execute('SELECT * FROM registrations ORDER BY created_at DESC')
            records = [dict(row) for row in c.fetchall()]
            conn.close()
            return render_template('admin.html', records=records, logged_in=True, admin_email=ADMIN_EMAIL, admin_phone=ADMIN_PHONE)
        else:
            error = 'Incorrect password.'
    return render_template('admin.html', records=[], logged_in=False, error=error)

@app.route('/api/registrations')
def api_registrations():
    # Optional simple JSON API (no auth for ease; add if needed)
    conn = sqlite3.connect(DATABASE)
    conn.row_factory = sqlite3.Row
    c = conn.cursor()
    c.execute('SELECT * FROM registrations ORDER BY created_at DESC')
    records = [dict(row) for row in c.fetchall()]
    conn.close()
    return jsonify(records)

if __name__ == '__main__':
    init_db()
    print("="*60)
    print("  Battle Arena Esports Server Starting...")
    print("  Website: http://127.0.0.1:5000")
    print("  Admin Panel: http://127.0.0.1:5000/admin")
    print("  Admin Password: admin123")
    print("="*60)
    print("\n  IMPORTANT - Gmail Setup:")
    print("  1. Enable 2-Step Verification: https://myaccount.google.com/security")
    print("  2. Generate App Password: https://myaccount.google.com/apppasswords")
    print("  3. Edit app.py and replace 'YOUR_GMAIL_APP_PASSWORD' with your 16-char app password")
    print("  4. Or set env variable: $env:GMAIL_APP_PASSWORD='yourpassword'")
    print("  Guide: See GMAIL_SETUP_GUIDE.md in this folder\n")
    print("="*60)
    app.run(debug=True, port=5000)

