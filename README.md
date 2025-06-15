# Vitapro project/
├── app.py
├── requirements.txt
├── templates/
│   ├── register.html
│   ├── login.html
│   ├── dashboard.html
│   ├── chat.html
│   ├── exchange.html
│   └── payment.html
from flask import Flask, render_template, request, redirect, url_for, session, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_socketio import SocketIO, emit
from werkzeug.security import generate_password_hash, check_password_hash
import requests

app = Flask(__name__)
app.secret_key = 'your_secret_key'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///users.db'
db = SQLAlchemy(app)
socketio = SocketIO(app, cors_allowed_origins="*")

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(150), unique=True, nullable=False)
    password = db.Column(db.String(150), nullable=False)

@app.before_first_request
def create_tables():
    db.create_all()

@app.route('/')
def index():
    if 'user_id' in session:
        return render_template('dashboard.html', username=session.get('username'))
    return redirect(url_for('login'))

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = generate_password_hash(request.form['password'])
        if User.query.filter_by(username=username).first():
            return 'User exists!'
        db.session.add(User(username=username, password=password))
        db.session.commit()
        return redirect(url_for('login'))
    return render_template('register.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        user = User.query.filter_by(username=request.form['username']).first()
        if user and check_password_hash(user.password, request.form['password']):
            session['user_id'] = user.id
            session['username'] = user.username
            return redirect(url_for('index'))
        return 'Invalid credentials!'
    return render_template('login.html')

@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('login'))

# ======== CHAT ========
@app.route('/chat')
def chat():
    if 'user_id' not in session:
        return redirect(url_for('login'))
    return render_template('chat.html', username=session.get('username'))

@socketio.on('send_message')
def handle_send(data):
    emit('receive_message', {'username': session['username'], 'message': data['message']}, broadcast=True)

# ======== EXCHANGE ========
@app.route('/exchange')
def exchange():
    return render_template('exchange.html')

@app.route('/exchange-rate')
def exchange_rate():
    res = requests.get("https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum&vs_currencies=usd").json()
    return jsonify(res)

# ======== PAYMENTS ========
@app.route('/payment')
def payment():
    return render_template('payment.html')

@app.route('/create-payment')
def create_payment():
    api_key = 'your_nowpayments_api_key'
    payload = {
        "price_amount": 10,
        "price_currency": "usd",
        "pay_currency": "btc",
        "ipn_callback_url": "https://yourdomain.com/callback",
    }
    headers = {
        "x-api-key": api_key,
        "Content-Type": "application/json"
    }
    r = requests.post("https://api.nowpayments.io/v1/payment", json=payload, headers=headers)
    return r.json()

if __name__ == "__main__":
    socketio.run(app, debug=True)
    
