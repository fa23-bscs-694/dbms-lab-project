#timebank project 
app.py
"""
TimeBank — a community skill-exchange management system.
Instead of money, members trade in HOURS. Every member starts with a
small balance of free hours. You list skills you can offer, other
members book your time, and when a session is marked complete the
hours move automatically from the requester's balance to the
provider's balance. An admin view lets you audit every transaction.
Only dependency: Flask (Werkzeug ships with it). Uses sqlite3 from the
standard library directly — no ORM, no Flask-Login — so it runs
anywhere with just `pip install flask`.
Run:
 pip install flask
 python app.py
Then open http://127.0.0.1:5000
"""
import sqlite3
from datetime import datetime
from functools import wraps
from pathlib import Path
from flask import (
 Flask, render_template, redirect, url_for, request, flash, abort,
 session, g
)
from werkzeug.security import generate_password_hash, check_password_hash
BASE_DIR = Path(__file__).parent
DB_PATH = BASE_DIR / "timebank.db"
STARTING_BALANCE = 5.0
app = Flask(__name__)
app.config["SECRET_KEY"] = "change-this-secret-key-in-production"
# ---------------------------------------------------------------------------
# Database helpers
# ---------------------------------------------------------------------------
def get_db():
 if "db" not in g:
 g.db = sqlite3.connect(DB_PATH)
 g.db.row_factory = sqlite3.Row
 g.db.execute("PRAGMA foreign_keys = ON")
 return g.db
@app.teardown_appcontext
def close_db(exception=None):
 db = g.pop("db", None)
 if db is not None:
 db.close()
def init_db():
 db = sqlite3.connect(DB_PATH)
 db.executescript(
 """
 CREATE TABLE IF NOT EXISTS user (
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 username TEXT UNIQUE NOT NULL,
 email TEXT UNIQUE NOT NULL,
 password_hash TEXT NOT NULL,
 bio TEXT DEFAULT '',
 balance REAL DEFAULT 5.0,
 is_admin INTEGER DEFAULT 0,
 created_at TEXT NOT NULL );
 CREATE TABLE IF NOT EXISTS skill (
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 title TEXT NOT NULL,
 description TEXT NOT NULL,
 category TEXT NOT NULL,
 hourly_rate REAL NOT NULL,
 owner_id INTEGER NOT NULL REFERENCES user(id) ON DELETE CASCADE,
 created_at TEXT NOT NULL
 );
 CREATE TABLE IF NOT EXISTS booking (
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 skill_id INTEGER NOT NULL REFERENCES skill(id) ON DELETE CASCADE,
 requester_id INTEGER NOT NULL REFERENCES user(id),
 hours REAL NOT NULL,
 message TEXT DEFAULT '',
 status TEXT DEFAULT 'pending',
 created_at TEXT NOT NULL
 );
 CREATE TABLE IF NOT EXISTS "transaction" (
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 booking_id INTEGER NOT NULL REFERENCES booking(id),
 from_user_id INTEGER NOT NULL REFERENCES user(id),
 to_user_id INTEGER NOT NULL REFERENCES user(id),
 hours REAL NOT NULL,
 created_at TEXT NOT NULL
 );
 """
 )
 db.commit()
 db.close()
def now_iso():
 return datetime.now().isoformat(timespec="seconds")
def fmt_dt(iso_str):
 try:
 return datetime.fromisoformat(iso_str).strftime("%b %d, %Y %H:%M")
 except (ValueError, TypeError):
 return iso_str
app.jinja_env.filters["fmt_dt"] = fmt_dt
# ---------------------------------------------------------------------------
# Auth helpers (hand-rolled, session-based)
# ---------------------------------------------------------------------------
def get_current_user():
 if "user" not in g:
 user_id = session.get("user_id")
 g.user = None
 if user_id is not None:
 row = get_db().execute(
 "SELECT * FROM user WHERE id = ?", (user_id,)
 ).fetchone()
 g.user = row
 return g.user
@app.context_processor
def inject_user():
 return {"current_user": get_current_user()}
def login_required(f):
 @wraps(f) def wrapper(*args, **kwargs):
 if get_current_user() is None:
 flash("Please log in to continue.", "warning")
 return redirect(url_for("login", next=request.path))
 return f(*args, **kwargs)
 return wrapper
def admin_required(f):
 @wraps(f)
 def wrapper(*args, **kwargs):
 user = get_current_user()
 if user is None or not user["is_admin"]:
 abort(403)
 return f(*args, **kwargs)
 return wrapper
# ---------------------------------------------------------------------------
# Auth routes
# ---------------------------------------------------------------------------
@app.route("/register", methods=["GET", "POST"])
def register():
 if get_current_user():
 return redirect(url_for("dashboard"))
 if request.method == "POST":
 username = request.form["username"].strip()
 email = request.form["email"].strip().lower()
 password = request.form["password"]
 db = get_db()
 if not username or not email or not password:
 flash("All fields are required.", "danger")
 return redirect(url_for("register"))
 if db.execute("SELECT 1 FROM user WHERE username = ?", (username,)).fetchone():
 flash("That username is already taken.", "danger")
 return redirect(url_for("register"))
 if db.execute("SELECT 1 FROM user WHERE email = ?", (email,)).fetchone():
 flash("That email is already registered.", "danger")
 return redirect(url_for("register"))
 is_admin = 1 if db.execute("SELECT COUNT(*) c FROM user").fetchone()["c"] == 0 else 0
 cur = db.execute(
 "INSERT INTO user (username, email, password_hash, balance, is_admin, created_at) "
 "VALUES (?, ?, ?, ?, ?, ?)",
 (username, email, generate_password_hash(password), STARTING_BALANCE,
 is_admin, now_iso()),
 )
 db.commit()
 session["user_id"] = cur.lastrowid
 flash(f"Welcome to TimeBank, {username}! You start with "
 f"{STARTING_BALANCE:g} free hours.", "success")
 return redirect(url_for("dashboard"))
 return render_template("register.html")
@app.route("/login", methods=["GET", "POST"])
def login():
 if get_current_user():
 return redirect(url_for("dashboard"))
 if request.method == "POST":
 username = request.form["username"].strip()
 password = request.form["password"]
 row = get_db().execute(
 "SELECT * FROM user WHERE username = ?", (username,)
 ).fetchone() if row and check_password_hash(row["password_hash"], password):
 session["user_id"] = row["id"]
 flash("Logged in successfully.", "success")
 return redirect(request.args.get("next") or url_for("dashboard"))
 flash("Invalid username or password.", "danger")
 return render_template("login.html")
@app.route("/logout")
def logout():
 session.clear()
 flash("You have been logged out.", "info")
 return redirect(url_for("index"))
# ---------------------------------------------------------------------------
# Core pages
# ---------------------------------------------------------------------------
@app.route("/")
def index():
 if get_current_user():
 return redirect(url_for("dashboard"))
 db = get_db()
 skill_count = db.execute("SELECT COUNT(*) c FROM skill").fetchone()["c"]
 member_count = db.execute("SELECT COUNT(*) c FROM user").fetchone()["c"]
 return render_template("index.html", skill_count=skill_count, member_count=member_count)
@app.route("/dashboard")
@login_required
def dashboard():
 user = get_current_user()
 db = get_db()
 my_skills = db.execute(
 "SELECT * FROM skill WHERE owner_id = ? ORDER BY created_at DESC", (user["id"],)
 ).fetchall()
 incoming = db.execute(
 """SELECT booking.*, skill.title AS skill_title, user.username AS requester_name
 FROM booking
 JOIN skill ON booking.skill_id = skill.id
 JOIN user ON booking.requester_id = user.id
 WHERE skill.owner_id = ?
 ORDER BY booking.created_at DESC""",
 (user["id"],),
 ).fetchall()
 outgoing = db.execute(
 """SELECT booking.*, skill.title AS skill_title, user.username AS owner_name
 FROM booking
 JOIN skill ON booking.skill_id = skill.id
 JOIN user ON skill.owner_id = user.id
 WHERE booking.requester_id = ?
 ORDER BY booking.created_at DESC""",
 (user["id"],),
 ).fetchall()
 recent_tx = db.execute(
 """SELECT "transaction".*, fu.username AS from_name, tu.username AS to_name
 FROM "transaction"
 JOIN user fu ON "transaction".from_user_id = fu.id
 JOIN user tu ON "transaction".to_user_id = tu.id
 WHERE "transaction".from_user_id = ? OR "transaction".to_user_id = ?
 ORDER BY "transaction".created_at DESC LIMIT 10""",
 (user["id"], user["id"]),
 ).fetchall()
 return render_template("dashboard.html", my_skills=my_skills, incoming=incoming,
 outgoing=outgoing, recent_tx=recent_tx) @app.route("/browse")
@login_required
def browse():
 user = get_current_user()
 category = request.args.get("category", "")
 q = request.args.get("q", "")
 db = get_db()
 sql = """SELECT skill.*, user.username AS owner_name
 FROM skill JOIN user ON skill.owner_id = user.id
 WHERE skill.owner_id != ?"""
 params = [user["id"]]
 if category:
 sql += " AND skill.category = ?"
 params.append(category)
 if q:
 sql += " AND skill.title LIKE ?"
 params.append(f"%{q}%")
 sql += " ORDER BY skill.created_at DESC"
 skills = db.execute(sql, params).fetchall()
 categories = [r["category"] for r in db.execute(
 "SELECT DISTINCT category FROM skill"
 ).fetchall()]
 return render_template("browse.html", skills=skills, categories=categories,
 selected_category=category, q=q)
@app.route("/skill/<int:skill_id>")
@login_required
def skill_detail(skill_id):
 db = get_db()
 skill = db.execute(
 """SELECT skill.*, user.username AS owner_name, user.bio AS owner_bio,
 user.id AS owner_id
 FROM skill JOIN user ON skill.owner_id = user.id
 WHERE skill.id = ?""",
 (skill_id,),
 ).fetchone()
 if skill is None:
 abort(404)
 return render_template("skill_detail.html", skill=skill)
@app.route("/skill/add", methods=["GET", "POST"])
@login_required
def add_skill():
 if request.method == "POST":
 user = get_current_user()
 title = request.form["title"].strip()
 description = request.form["description"].strip()
 category = request.form["category"].strip() or "General"
 try:
 hourly_rate = max(float(request.form["hourly_rate"]), 0.25)
 except ValueError:
 hourly_rate = 1.0
 if not title or not description:
 flash("Title and description are required.", "danger")
 return redirect(url_for("add_skill"))
 db = get_db()
 db.execute(
 "INSERT INTO skill (title, description, category, hourly_rate, owner_id, created_at) "
 "VALUES (?, ?, ?, ?, ?, ?)",
 (title, description, category, hourly_rate, user["id"], now_iso()),
 )
 db.commit()
 flash("Skill listed! Other members can now book your time.", "success")
 return redirect(url_for("dashboard"))
 return render_template("add_skill.html") @app.route("/skill/<int:skill_id>/delete", methods=["POST"])
@login_required
def delete_skill(skill_id):
 user = get_current_user()
 db = get_db()
 skill = db.execute("SELECT * FROM skill WHERE id = ?", (skill_id,)).fetchone()
 if skill is None:
 abort(404)
 if skill["owner_id"] != user["id"]:
 abort(403)
 db.execute("DELETE FROM skill WHERE id = ?", (skill_id,))
 db.commit()
 flash("Skill removed.", "info")
 return redirect(url_for("dashboard"))
# ---------------------------------------------------------------------------
# Booking workflow
# ---------------------------------------------------------------------------
@app.route("/skill/<int:skill_id>/book", methods=["POST"])
@login_required
def book_skill(skill_id):
 user = get_current_user()
 db = get_db()
 skill = db.execute("SELECT * FROM skill WHERE id = ?", (skill_id,)).fetchone()
 if skill is None:
 abort(404)
 if skill["owner_id"] == user["id"]:
 flash("You can't book your own skill.", "danger")
 return redirect(url_for("skill_detail", skill_id=skill_id))
 try:
 hours = float(request.form["hours"])
 except ValueError:
 flash("Enter a valid number of hours.", "danger")
 return redirect(url_for("skill_detail", skill_id=skill_id))
 if hours <= 0:
 flash("Hours must be greater than zero.", "danger")
 return redirect(url_for("skill_detail", skill_id=skill_id))
 cost = round(hours * skill["hourly_rate"], 2)
 if cost > user["balance"]:
 flash(f"Insufficient balance. This booking would cost {cost:g} hours "
 f"but you only have {user['balance']:g}.", "danger")
 return redirect(url_for("skill_detail", skill_id=skill_id))
 message = request.form.get("message", "").strip()
 db.execute(
 "INSERT INTO booking (skill_id, requester_id, hours, message, status, created_at) "
 "VALUES (?, ?, ?, ?, 'pending', ?)",
 (skill_id, user["id"], cost, message, now_iso()),
 )
 db.commit()
 flash("Booking request sent!", "success")
 return redirect(url_for("dashboard"))
@app.route("/booking/<int:booking_id>/<action>", methods=["POST"])
@login_required
def update_booking(booking_id, action):
 user = get_current_user()
 db = get_db()
 booking = db.execute(
 """SELECT booking.*, skill.owner_id AS provider_id
 FROM booking JOIN skill ON booking.skill_id = skill.id
 WHERE booking.id = ?""",
 (booking_id,),
 ).fetchone()
 if booking is None: abort(404)
 is_provider = booking["provider_id"] == user["id"]
 is_requester = booking["requester_id"] == user["id"]
 if not (is_provider or is_requester):
 abort(403)
 if action == "accept" and is_provider and booking["status"] == "pending":
 db.execute("UPDATE booking SET status = 'accepted' WHERE id = ?", (booking_id,))
 flash("Booking accepted.", "success")
 elif action == "decline" and is_provider and booking["status"] == "pending":
 db.execute("UPDATE booking SET status = 'declined' WHERE id = ?", (booking_id,))
 flash("Booking declined.", "info")
 elif action == "cancel" and is_requester and booking["status"] in ("pending", "accepted"):
 db.execute("UPDATE booking SET status = 'cancelled' WHERE id = ?", (booking_id,))
 flash("Booking cancelled.", "info")
 elif action == "complete" and is_provider and booking["status"] == "accepted":
 requester = db.execute(
 "SELECT * FROM user WHERE id = ?", (booking["requester_id"],)
 ).fetchone()
 if requester["balance"] < booking["hours"]:
 flash("Requester no longer has sufficient balance.", "danger")
 return redirect(url_for("dashboard"))
 db.execute("UPDATE user SET balance = balance - ? WHERE id = ?",
 (booking["hours"], requester["id"]))
 db.execute("UPDATE user SET balance = balance + ? WHERE id = ?",
 (booking["hours"], user["id"]))
 db.execute("UPDATE booking SET status = 'completed' WHERE id = ?", (booking_id,))
 db.execute(
 'INSERT INTO "transaction" (booking_id, from_user_id, to_user_id, hours, created_at) '
 "VALUES (?, ?, ?, ?, ?)",
 (booking_id, requester["id"], user["id"], booking["hours"], now_iso()),
 )
 flash(f"Session marked complete. {booking['hours']:g} hours transferred.", "success")
 else:
 flash("That action isn't allowed right now.", "danger")
 db.commit()
 return redirect(url_for("dashboard"))
# ---------------------------------------------------------------------------
# Profile
# ---------------------------------------------------------------------------
@app.route("/profile", methods=["GET", "POST"])
@login_required
def profile():
 user = get_current_user()
 if request.method == "POST":
 bio = request.form.get("bio", "").strip()[:300]
 db = get_db()
 db.execute("UPDATE user SET bio = ? WHERE id = ?", (bio, user["id"]))
 db.commit()
 flash("Profile updated.", "success")
 return redirect(url_for("profile"))
 return render_template("profile.html")
# ---------------------------------------------------------------------------
# Admin
# ---------------------------------------------------------------------------
@app.route("/admin")
@admin_required
def admin_panel():
 db = get_db()
 users = db.execute("SELECT * FROM user ORDER BY created_at DESC").fetchall() transactions = db.execute(
 """SELECT "transaction".*, fu.username AS from_name, tu.username AS to_name
 FROM "transaction"
 JOIN user fu ON "transaction".from_user_id = fu.id
 JOIN user tu ON "transaction".to_user_id = tu.id
 ORDER BY "transaction".created_at DESC LIMIT 50"""
 ).fetchall()
 total_hours = sum(u["balance"] for u in users)
 return render_template("admin.html", users=users, transactions=transactions,
 total_hours=total_hours)
# ---------------------------------------------------------------------------
# Entry point
# ---------------------------------------------------------------------------
if __name__ == "__main__":
 init_db()
 app.run(debug=True)
 requirements.txt
Flask>=3.0
ask>=3.0 templates/base.html
<!doctype html>
<html lang="en">
<head>
 <meta charset="utf-8">
 <meta name="viewport" content="width=device-width, initial-scale=1">
 <title>{% block title %}TimeBank{% endblock %}</title>
 <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
 <nav class="navbar">
 <a href="{{ url_for('index') }}" class="brand">■ TimeBank</a>
 <div class="nav-links">
 {% if current_user.is_authenticated %}
 <span class="balance-pill">{{ '%.2f'|format(current_user.balance) }} hrs</span>
 <a href="{{ url_for('dashboard') }}">Dashboard</a>
 <a href="{{ url_for('browse') }}">Browse Skills</a>
 <a href="{{ url_for('profile') }}">Profile</a>
 {% if current_user.is_admin %}<a href="{{ url_for('admin_panel') }}">Admin</a>{% endif %}
 <a href="{{ url_for('logout') }}">Logout</a>
 {% else %}
 <a href="{{ url_for('login') }}">Login</a>
 <a href="{{ url_for('register') }}" class="cta">Join</a>
 {% endif %}
 </div>
 </nav>
 <main class="container">
 {% with messages = get_flashed_messages(with_categories=true) %}
 {% if messages %}
 <div class="flash-wrap">
 {% for category, message in messages %}
 <div class="flash flash-{{ category }}">{{ message }}</div>
 {% endfor %}
 </div>
 {% endif %}
 {% endwith %}
 {% block content %}{% endblock %}
 </main>
</body>
</html>
templates/index.html
{% extends "base.html" %}
{% block title %}TimeBank — Trade skills, not money{% endblock %}
{% block content %}
<div class="hero">
 <h1>Your time is the currency.</h1>
 <p>List a skill, help a neighbor, earn hours. Spend those hours booking
 help from someone else. No money ever changes hands.</p>
 <div class="hero-stats">
 <div><strong>{{ member_count }}</strong><span>members</span></div>
 <div><strong>{{ skill_count }}</strong><span>skills listed</span></div>
 </div>
 <a class="cta big" href="{{ url_for('register') }}">Join TimeBank — get 5 free hours</a>
 <a class="secondary" href="{{ url_for('login') }}">I already have an account</a>
</div>
<div class="how-it-works">
 <div class="step"><h3>1. List a skill</h3><p>Tutoring, repairs, design, cooking — anything you 
can teach or do for someone.</p></div>
 <div class="step"><h3>2. Get booked</h3><p>Members spend their hour-balance to book time with you.
</p></div>
 <div class="step"><h3>3. Earn hours</h3><p>Mark the session complete and the hours land straight 
in your balance.</p></div>
</div>
{% endblock %}
templates/register.html
{% extends "base.html" %}
{% block title %}Join — TimeBank{% endblock %}
{% block content %}
<div class="form-card">
 <h2>Create your account</h2>
 <form method="POST">
 <label>Username</label>
 <input type="text" name="username" required autofocus>
 <label>Email</label>
 <input type="email" name="email" required>
 <label>Password</label>
 <input type="password" name="password" required>
 <button type="submit" class="cta">Create account — get 5 free hours</button>
 </form>
 <p class="muted">Already a member? <a href="{{ url_for('login') }}">Log in</a></p>
</div>
{% endblock %}
templates/dashboard.html
{% extends "base.html" %}
{% block title %}Dashboard — TimeBank{% endblock %}
{% block content %}
<h1>Welcome back, {{ current_user.username }}</h1>
<p class="muted">Balance: <strong>{{ '%.2f'|format(current_user.balance) }} hours</strong></p>
<div class="grid-2">
 <section class="card">
 <div class="card-header">
 <h2>My skills</h2>
 <a class="cta small" href="{{ url_for('add_skill') }}">+ List a skill</a>
 </div>
 {% if my_skills %}
 <ul class="list">
 {% for s in my_skills %}
 <li>
 <div>
 <strong>{{ s.title }}</strong>
 <span class="tag">{{ s.category }}</span>
 <div class="muted small">{{ s.hourly_rate }} hrs / hour</div>
 </div>
 <form method="POST" action="{{ url_for('delete_skill', skill_id=s.id) }}"
 onsubmit="return confirm('Remove this skill?');">
 <button class="link-danger" type="submit">Remove</button>
 </form>
 </li>
 {% endfor %}
 </ul>
 {% else %}
 <p class="muted">You haven't listed any skills yet.</p>
 {% endif %}
 </section>
 <section class="card">
 <h2>Booking requests for me</h2>
 {% if incoming %}
 <ul class="list">
 {% for b in incoming %}
 <li class="booking-row">
 <div>
 <strong>{{ b.skill_title }}</strong> requested by {{ b.requester_name }}
 <div class="muted small">{{ b.hours }} hrs &middot; status:
 <span class="status status-{{ b.status }}">{{ b.status }}</span></div>
 {% if b.message %}<div class="muted small">"{{ b.message }}"</div>{% endif %}
 </div>
 <div class="row-actions">
 {% if b.status == 'pending' %}
 <form method="POST" action="{{ url_for('update_booking', booking_id=b.id, 
action='accept') }}"><button class="cta small">Accept</button></form>
 <form method="POST" action="{{ url_for('update_booking', booking_id=b.id, 
action='decline') }}"><button class="secondary small">Decline</button></form>
 {% elif b.status == 'accepted' %}
 <form method="POST" action="{{ url_for('update_booking', booking_id=b.id, 
action='complete') }}"><button class="cta small">Mark complete</button></form>
 {% endif %}
 </div>
 </li>
 {% endfor %}
 </ul>
 {% else %}
 <p class="muted">No booking requests yet.</p>
 {% endif %}
 </section>
</div>
<div class="grid-2">
 <section class="card">
 <h2>My booking requests</h2>
 {% if outgoing %}
 <ul class="list"> {% for b in outgoing %}
 <li class="booking-row">
 <div>
 <strong>{{ b.skill_title }}</strong> with {{ b.owner_name }}
 <div class="muted small">{{ b.hours }} hrs &middot; status:
 <span class="status status-{{ b.status }}">{{ b.status }}</span></div>
 </div>
 {% if b.status in ['pending', 'accepted'] %}
 <form method="POST" action="{{ url_for('update_booking', booking_id=b.id, 
action='cancel') }}">
 <button class="link-danger" type="submit">Cancel</button>
 </form>
 {% endif %}
 </li>
 {% endfor %}
 </ul>
 {% else %}
 <p class="muted">You haven't booked anything yet. <a href="{{ url_for('browse') }}">Browse 
skills</a>.</p>
 {% endif %}
 </section>
 <section class="card">
 <h2>Recent transactions</h2>
 {% if recent_tx %}
 <ul class="list">
 {% for t in recent_tx %}
 <li>
 {% if t.from_user_id == current_user.id %}
 Paid <strong>{{ t.hours }} hrs</strong> to {{ t.to_name }}
 {% else %}
 Earned <strong>{{ t.hours }} hrs</strong> from {{ t.from_name }}
 {% endif %}
 <div class="muted small">{{ t.created_at|fmt_dt }}</div>
 </li>
 {% endfor %}
 </ul>
 {% else %}
 <p class="muted">No transactions yet.</p>
 {% endif %}
 </section>
</div>
{% endblock %} templates/browse.html
{% extends "base.html" %}
{% block title %}Browse Skills — TimeBank{% endblock %}
{% block content %}
<h1>Browse skills</h1>
<form method="GET" class="filter-bar">
 <input type="text" name="q" placeholder="Search skills..." value="{{ q }}">
 <select name="category">
 <option value="">All categories</option>
 {% for c in categories %}
 <option value="{{ c }}" {% if c == selected_category %}selected{% endif %}>{{ c }}</option>
 {% endfor %}
 </select>
 <button class="cta small" type="submit">Filter</button>
</form>
{% if skills %}
 <div class="skill-grid">
 {% for s in skills %}
 <a class="skill-card" href="{{ url_for('skill_detail', skill_id=s.id) }}">
 <span class="tag">{{ s.category }}</span>
 <h3>{{ s.title }}</h3>
 <p class="muted">{{ s.description[:100] }}{% if s.description|length > 100 %}…{% endif %}</
p>
 <div class="skill-card-footer">
 <span>{{ s.owner_name }}</span>
 <strong>{{ s.hourly_rate }} hrs/hr</strong>
 </div>
 </a>
 {% endfor %}
 </div>
{% else %}
 <p class="muted">No skills match your search.</p>
{% endif %}
{% endblock %} templates/skill_detail.html
{% extends "base.html" %}
{% block title %}{{ skill.title }} — TimeBank{% endblock %}
{% block content %}
<div class="form-card wide">
 <span class="tag">{{ skill.category }}</span>
 <h1>{{ skill.title }}</h1>
 <p class="muted">Offered by <strong>{{ skill.owner_name }}</strong> &middot; {{ skill.hourly_rate 
}} hrs per hour</p>
 <p>{{ skill.description }}</p>
 {% if skill.owner_bio %}
 <div class="bio-box"><strong>About {{ skill.owner_name }}:</strong> {{ skill.owner_bio }}</div>
 {% endif %}
 {% if skill.owner_id != current_user.id %}
 <hr>
 <h3>Request a booking</h3>
 <form method="POST" action="{{ url_for('book_skill', skill_id=skill.id) }}">
 <label>How many hours of their time?</label>
 <input type="number" name="hours" step="0.25" min="0.25" required>
 <label>Message (optional)</label>
 <textarea name="message" rows="3" placeholder="What do you need help with?"></textarea>
 <button type="submit" class="cta">Request booking</button>
 </form>
 <p class="muted small">Your balance: {{ '%.2f'|format(current_user.balance) }} hrs</p>
 {% else %}
 <p class="muted">This is your own listing.</p>
 {% endif %}
</div>
{% endblock %} templates/add_skill.html
{% extends "base.html" %}
{% block title %}List a Skill — TimeBank{% endblock %}
{% block content %}
<div class="form-card">
 <h2>List a skill</h2>
 <form method="POST">
 <label>Title</label>
 <input type="text" name="title" placeholder="e.g. Guitar lessons for beginners" required>
 <label>Category</label>
 <input type="text" name="category" placeholder="e.g. Music, Tech, Home Repair" required>
 <label>Description</label>
 <textarea name="description" rows="4" required placeholder="What will you teach or do?"></
textarea>
 <label>Rate (hours charged per hour of your time)</label>
 <input type="number" name="hourly_rate" step="0.25" min="0.25" value="1" required>
 <button type="submit" class="cta">List skill</button>
 </form>
</div>
{% endblock %} templates/profile.html
{% extends "base.html" %}
{% block title %}My Profile — TimeBank{% endblock %}
{% block content %}
<div class="form-card">
 <h2>My profile</h2>
 <p><strong>Username:</strong> {{ current_user.username }}</p>
 <p><strong>Email:</strong> {{ current_user.email }}</p>
 <p><strong>Balance:</strong> {{ '%.2f'|format(current_user.balance) }} hours</p>
 <p><strong>Member since:</strong> {{ current_user.created_at|fmt_dt }}</p>
 <form method="POST">
 <label>Bio</label>
 <textarea name="bio" rows="4" maxlength="300" placeholder="Tell other members about yourself...
">{{ current_user.bio }}</textarea>
 <button type="submit" class="cta">Save</button>
 </form>
</div>
{% endblock %} templates/admin.html
{% extends "base.html" %}
{% block title %}Admin — TimeBank{% endblock %}
{% block content %}
<h1>Admin panel</h1>
<p class="muted">Total hours in circulation: <strong>{{ '%.2f'|format(total_hours) }}</strong></p>
<div class="grid-2">
 <section class="card">
 <h2>Members ({{ users|length }})</h2>
 <table class="table">
 <thead><tr><th>Username</th><th>Email</th><th>Balance</th><th>Joined</th></tr></thead>
 <tbody>
 {% for u in users %}
 <tr>
 <td>{{ u.username }}{% if u.is_admin %} <span class="tag">admin</span>{% endif %}</td>
 <td>{{ u.email }}</td>
 <td>{{ '%.2f'|format(u.balance) }}</td>
 <td>{{ u.created_at|fmt_dt }}</td>
 </tr>
 {% endfor %}
 </tbody>
 </table>
 </section>
 <section class="card">
 <h2>Recent transactions</h2>
 <table class="table">
 <thead><tr><th>From</th><th>To</th><th>Hours</th><th>When</th></tr></thead>
 <tbody>
 {% for t in transactions %}
 <tr>
 <td>{{ t.from_name }}</td>
 <td>{{ t.to_name }}</td>
 <td>{{ t.hours }}</td>
 <td>{{ t.created_at|fmt_dt }}</td>
 </tr>
 {% endfor %}
 </tbody>
 </table>
 </section>
</div>
{% endblock %} static/style.css
:root {
 --ink: #1c2027;
 --muted: #6b7280;
 --accent: #0f766e;
 --accent-dark: #0b5e58;
 --bg: #f7f7f5;
 --card: #ffffff;
 --border: #e5e7eb;
 --danger: #b91c1c;
}
* { box-sizing: border-box; }
body {
 margin: 0;
 font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
 background: var(--bg);
 color: var(--ink);
 line-height: 1.5;
}
.navbar {
 display: flex;
 justify-content: space-between;
 align-items: center;
 padding: 14px 28px;
 background: var(--card);
 border-bottom: 1px solid var(--border);
 position: sticky;
 top: 0;
 z-index: 10;
}
.brand { font-weight: 700; font-size: 1.15rem; color: var(--ink); text-decoration: none; }
.nav-links { display: flex; align-items: center; gap: 18px; }
.nav-links a { color: var(--ink); text-decoration: none; font-size: 0.95rem; }
.nav-links a:hover { color: var(--accent); }
.balance-pill {
 background: #ecfdf5;
 color: var(--accent-dark);
 border: 1px solid #a7f3d0;
 padding: 4px 10px;
 border-radius: 999px;
 font-size: 0.85rem;
 font-weight: 600;
}
.container { max-width: 1000px; margin: 0 auto; padding: 32px 20px 60px; }
.cta {
 display: inline-block;
 background: var(--accent);
 color: #fff !important;
 padding: 10px 18px;
 border-radius: 8px;
 text-decoration: none;
 border: none;
 cursor: pointer;
 font-size: 0.95rem;
 font-weight: 600;
}
.cta:hover { background: var(--accent-dark); }
.cta.big { padding: 14px 26px; font-size: 1.05rem; margin-top: 20px; }
.cta.small { padding: 6px 12px; font-size: 0.85rem; }
.secondary {
 display: inline-block;
 margin-left: 12px; color: var(--muted);
 text-decoration: none;
 font-size: 0.9rem;
}
.secondary.small { background: #f3f4f6; color: var(--ink) !important; padding: 6px 12px; border-
radius: 8px; border: 1px solid var(--border); font-size: 0.85rem; cursor: pointer; }
.link-danger { background: none; border: none; color: var(--danger); cursor: pointer; font-size: 0.
85rem; text-decoration: underline; padding: 0; }
.flash-wrap { margin-bottom: 20px; }
.flash { padding: 10px 14px; border-radius: 8px; margin-bottom: 8px; font-size: 0.92rem; }
.flash-success { background: #ecfdf5; color: #065f46; border: 1px solid #a7f3d0; }
.flash-danger { background: #fef2f2; color: #991b1b; border: 1px solid #fecaca; }
.flash-info { background: #eff6ff; color: #1e40af; border: 1px solid #bfdbfe; }
.flash-warning { background: #fffbeb; color: #92400e; border: 1px solid #fde68a; }
.hero { text-align: center; padding: 60px 0 40px; }
.hero h1 { font-size: 2.4rem; margin-bottom: 10px; }
.hero p { color: var(--muted); max-width: 560px; margin: 0 auto; }
.hero-stats { display: flex; justify-content: center; gap: 40px; margin: 30px 0; }
.hero-stats div { display: flex; flex-direction: column; }
.hero-stats strong { font-size: 1.6rem; }
.hero-stats span { color: var(--muted); font-size: 0.85rem; }
.how-it-works { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; margin-top: 50px; }
.step { background: var(--card); border: 1px solid var(--border); border-radius: 12px; padding: 
20px; }
.step h3 { margin-top: 0; color: var(--accent-dark); }
.step p { color: var(--muted); font-size: 0.92rem; }
.form-card {
 background: var(--card);
 border: 1px solid var(--border);
 border-radius: 12px;
 padding: 28px;
 max-width: 420px;
 margin: 0 auto;
}
.form-card.wide { max-width: 640px; }
form label { display: block; font-size: 0.85rem; font-weight: 600; margin: 14px 0 6px; }
form input, form select, form textarea {
 width: 100%;
 padding: 9px 12px;
 border: 1px solid var(--border);
 border-radius: 8px;
 font-size: 0.95rem;
 font-family: inherit;
}
form button.cta { margin-top: 18px; width: 100%; }
.muted { color: var(--muted); }
.muted.small { font-size: 0.82rem; }
.grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-top: 24px; }
@media (max-width: 760px) { .grid-2, .how-it-works { grid-template-columns: 1fr; } }
.card { background: var(--card); border: 1px solid var(--border); border-radius: 12px; padding: 
22px; }
.card-header { display: flex; justify-content: space-between; align-items: center; }
.list { list-style: none; margin: 12px 0 0; padding: 0; }
.list li { padding: 12px 0; border-bottom: 1px solid var(--border); display: flex; justify-content: 
space-between; align-items: center; gap: 10px; }
.list li:last-child { border-bottom: none; }
.booking-row { flex-direction: row; }
.row-actions { display: flex; gap: 8px; }
.row-actions form { display: inline; }
.tag {
 display: inline-block; background: #f1f5f4;
 color: var(--accent-dark);
 font-size: 0.75rem;
 padding: 2px 8px;
 border-radius: 999px;
 margin-left: 6px;
}
.status { font-weight: 600; text-transform: capitalize; }
.status-pending { color: #b45309; }
.status-accepted { color: #0f766e; }
.status-completed { color: #15803d; }
.status-declined, .status-cancelled { color: #b91c1c; }
.filter-bar { display: flex; gap: 10px; margin-bottom: 20px; }
.filter-bar input, .filter-bar select { padding: 9px 12px; border: 1px solid var(--border); border-
radius: 8px; }
.filter-bar input { flex: 1; }
.skill-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(240px, 1fr)); gap: 
16px; }
.skill-card {
 background: var(--card);
 border: 1px solid var(--border);
 border-radius: 12px;
 padding: 18px;
 text-decoration: none;
 color: var(--ink);
 display: flex;
 flex-direction: column;
 gap: 6px;
}
.skill-card:hover { border-color: var(--accent); }
.skill-card h3 { margin: 4px 0; }
.skill-card-footer { display: flex; justify-content: space-between; font-size: 0.85rem; margin-top: 
8px; color: var(--muted); }
.bio-box { background: #f7f7f5; border-radius: 8px; padding: 12px; margin: 16px 0; font-size: 0.
92rem; }
.table { width: 100%; border-collapse: collapse; margin-top: 12px; font-size: 0.9rem; }
.table th, .table td { text-align: left; padding: 8px 6px; border-bottom: 1px solid var(--border); }
